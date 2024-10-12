---
lip: 21
title: Simple On-chain Delegation
status: Implemented
author: Eugene Pshenichnyy, Ekaterina Zueva, Victor Suzdalev, Alexander Belokon
discussions-to: https://research.lido.fi/t/lip-21-simple-on-chain-delegation/6840
created: 2024-03-06
updated: 2024-03-08
---

# LIP-21. Simple On-chain Delegation

## Simple Summary

The Simple On-chain Delegation allows LDO token holders to delegate their voting power to other addresses and delegates to participate in on-chain voting on behalf of their delegated voters.

## Abstract

This proposal is intended to allow LDO holders to designate addresses as their delegates. Delegates will be able to take part in on-chain voting using the voting power delegated to them. Each delegate will be able to use the voting power of multiple delegated voters.
Alongside that, it's proposed to add [TRP (Token Rewards Plan)](https://research.lido.fi/t/lidodao-token-rewards-plan-trp/3364) participants the ability to delegate their LDO rewards.

## Motivation

In the current conditions, reaching a quorum in on-chain voting has become highly challenging, and several consecutive votes failed to reach a quorum. The absence of an on-chain delegation is a blocker for stable and sufficient participation in on-chain voting and protocol development.

To respond to the growing inconvenience as quickly as possible, the DAO-ops workstream contributors propose to implement a straightforward solution: add a delegation feature directly to the `Voting` contract. This feature will allow the DAO to gain additional voting power and reduce the operational burden on workstream members.

In addition, it's proposed to update the `VotingAdapter` contract by implementing new features from the `Voting` contract so that TRP participants can also delegate their LDO rewards.

## Specification

### Overview

The Simple Delegation represents an updated implementation of `Voting` and `VotingAdapter` contracts. The main idea is to develop a simple and secure solution for on-chain delegation, considering the current Lido DAO voting architecture.

Using new mechanics introduced in the `Voting` contract, each token holder can assign themself a delegate address and thus become a delegated voter. That address could be any except for a zero address, a holder's address, or the holder's previous delegate address. They also can unassign a delegate at any moment. Assigning a delegate doesn't affect the holder's LDO balance in any way.

A delegated voter:

- retains the right to participate in votes;
- have a right to overwrite the delegate's vote made on their behalf;
- can assign only one delegate address at a time;
- can obtain information about which address is listed as their delegate.

A delegate:

- can use delegated voting power to participate in votes;
- can change their decision and re-vote with a different option, using the voting power of one or more delegated voters;
- can participate in votes on behalf of themself.

### Voting contract

Below, we'll take a closer look at the implementation details of Simple Delegation. In order to have a complete understanding of the implementation of mechanics described above, it's necessary to understand how it answers the following questions:

1. [How can one delegate voting power?](#Delegating-voting-power)
2. [How can a delegate assigned by token holders participate in the vote?](#Voting-as-a-delegate)
3. [How do tradeoffs affect technical implementation?](#Mitigating-design-tradeoffs)
4. [What changes to the existing code were required to make the implementation complete?](#Changes-made-to-existing-code)
5. [What other changes are decided to be made as part of this update?](#Deprecations)

The code examples presume the Solidity **v0.4.24** syntax.

#### Delegating voting power

The delegation allows token holders to delegate their voting power to other addresses. To assign a delegate, one can use the `assignDelegate` method.

```solidity
function assignDelegate(address _delegate) public
```

To successfully assign a delegate, the following requirements must be met:

- `_delegate` must not be a zero address;
- `_delegate` must not match with a caller's address;
- `_delegate` must not match with a current caller's delegate address;
- caller must have a non-zero LDO amount on their account;

If all requirements are met, the state will be updated, and the `AssignDelegate` event will be emitted.

```solidity
event AssignDelegate(
    address indexed voter,
    address indexed assignedDelegate
);
```

Where:

- `voter` is the caller's address;
- `assignedDelegate` is the address passed from the `_delegate` argument.

A token holder can unassign a delegate at any time. To do that, they can use the `unassignDelegate` method.

```solidity
function unassignDelegate() public
```

If the delegate wasn't assigned before, the call will be reverted. Otherwise, the state will be updated, and the `UnassignDelegate` event will be emitted.

```solidity
event UnassignDelegate(
    address indexed voter,
    address indexed unassignedDelegate
);
```

Where:

- `voter` is the caller's address;
- `unassignedDelegate` is the address of the current delegate.

Both `assignDelegate` and `unassignDelegate` are calling two internal methods to make the changes in the state:

```solidity
function _addDelegatedAddressFor(
    address _delegate,
    address _voter
) internal

function _removeDelegatedAddressFor(
    address _delegate,
    address _voter
) internal
```

The state itself consists of two new variables:

```solidity
struct DelegatedVoters {
        address[] addresses;
}
// delegate -> [delegated voter address]
mapping(address /* delegate */ => DelegatedVoters) private delegatedVoters;

struct Delegate {
        address delegate;
        uint96 voterIndex;
}
// voter -> delegate
mapping(address /* voter */ => Delegate) private delegates;
```

- `delegatedVoters` is a mapping containing all delegated voters' addresses for a given delegate.

  We chose to use an array to store delegated voters' addresses because we want to allow delegates to vote for multiple delegated voters at once. This structure, combined with new getters described below, will enable delegates to retrieve a list of addresses and then cast a vote for them.

- `delegates` is a mapping containing a current delegate address and its index in the delegate's `DelegatedVoters` array for a delegated voter's address.

  The choice of `uint96` type for `voterIndex` was made due to storage optimization: delegate and `voterIndex` take up only one 32-byte slot together.

  However, this optimization imposes a certain restriction on the maximum size of the `DelegatedVoters` array; the corresponding condition check was added to the `_addDelegatedAddressFor` method to avoid an overflow:

```solidity
...
uint256 delegatedVotersCount = votersList.length;
require(delegatedVotersCount <= UINT_96_MAX, ERROR_MAX_DELEGATED_VOTERS_REACHED);
...
```

After a holder assigns themself a delegate, the delegate can participate in voting on behalf of this holder. Next, we will inspect this process in detail.

#### Voting as a delegate

To allow delegates to use delegated voting power in votes, the following method has been added:

`attemptVoteForMultiple`

- **Arguments**:
  - `uint256 _voteId`;
  - `bool _supports` - whether the delegate's vote is "yea" or "nay";
  - `address[] _voters` - an array of addresses that the delegate is going to vote on behalf of.
- **Visibility**: `external`
- **Modifiers**: `voteExists(_voteId)`

Specific requirements must be met for a delegate's vote to be accepted; those can be divided into two categories:

1. Requirements inherited from the `vote` method:

   - the vote must be **open**: not expired and not yet executed;
   - a voter (delegate) must not cast a "yea" vote during the objection phase;
   - a voter must have a non-zero amount of LDO by the time the vote has started.

2. Requirements related to the delegation:
   - **each** voter from the `_voters` array must have the caller's address as their assigned delegate;
   - **each** voter from the `_voters` array must not have a vote cast by themself before the call.

If one of the requirements from (1) isn't met, the call will be reverted. However, requirements from (2) would not cause a revert. Voters who haven't met the requirements will be skipped. But if all of voters from the `_voters` were skipped, the call will be reverted.

Otherwise, the voting state will be updated, and in addition to existing `CastVote` and `CastObjection` events, one new event will be emitted.

```solidity
event AttemptCastVoteAsDelegate(
    uint256 indexed voteId,
    address indexed delegate,
    address[] voters,
)
```

- `voteId` is the `_voteId`;
- `delegate` is the caller's address;
- `voters` is the `_voters` list no matter how many voters were skipped;
- `supports` is the flag that indicates whether a caller supports voting decisions.

If a delegate wants to vote on behalf of a single delegated voter, they can use the `attemptVoteFor` method, which is basically a wrapper over the `attemptVoteForMultiple`.

```solidity
function attemptVoteFor(
    uint256 _voteId,
    bool _supports,
    address _voter
) external {
    address[] memory voters = new address[](1);
    voters[0] = _voter;
    attemptVoteForMultiple(_voteId, _supports, voters);
}
```

Using an array of addresses as an argument adds flexibility, allowing delegates to select the voters they want to vote for themselves. However, how delegates can vote using all their available delegated voting power, given all design tradeoffs, still needs to be determined. Below, we will look over what will allow them to do this.

#### Mitigating design tradeoffs

For delegates to obtain an up-to-date list of delegated voters, the corresponding getters have been added. However, some restrictions do not allow this to be done straightforwardly:

- an array is used to store a list of delegated voters. Choosing this data structure allows delegates to obtain the list and vote on behalf of its members. However, this choice creates a potential situation in which an attacker can fill this array, making it impossible for a delegate to obtain the complete list of delegated voters;
- to successfully vote for a list of voters, each member must meet specific requirements. So then, we need to ensure that the delegate can prepare a valid delegated voters list.

The following getters have been added to the contract, the use of which will mitigate these restrictions. First, let's look at the new internal getter.

`getDelegatedVoters`

- **Arguments**:
  - `address _delegate`;
  - `uint256 _offset` - number of addresses in a delegated voters array to skip;
  - `uint256 _limit` - maximum amount of addresses to get;
- **Visibility**: `external`
- **Mutability**: `view`
- **Returns**: `address[] memory voters`

The `getDelegatedVoters` returns a sliced array of addresses of voters who have assigned the `_delegate` as their delegate.
To avoid a potential attack that could end by flooding delegate's voters list, it was decided to implement this getter with `offset` and `limit` arguments, which will return a list in a range specified by users.

Knowing the LDO balances of voters is necessary to prepare a list of candidates for a delegate's vote. That is why the following getter was introduced.

`_getVotingPowerMultipleAt`

- **Arguments**:
  - `address[] _voters` - list of voters whose balances need to be obtained;
  - `uint256 _blockNumber` - block number for which LDO balances will be requested.
- **Visibility**: `internal`
- **Mutability**: `view`
- **Returns**: `uint256[] memory balances`

This method returns the list of LDO balances for the requested list of voters.

To make this data accessible to users, two external getters were introduced. Those methods are just wrappers over `_getVotingPowerMultipleAt`.

`getVotingPowerMultiple`

- **Arguments**:
  - `address[] _voters` - list of voters whose balances need to be obtained.
- **Visibility**: `external`
- **Mutability**: `view`
- **Returns**: `uint256[] memory balances`

This method returns the list of LDO balances at the current block.

`getVotingPowerMultipleAtVote`

- **Arguments**:
  - `uint256 _voteId`;
  - `address[] _voters` - list of voters whose balances need to be obtained.
- **Visibility**: `external`
- **Mutability**: `view`
- **Modifiers**: `voteExists(_voteId)`
- **Returns**: `uint256[] memory balances`

This method works similarly to the `getVotingPowerMultiple`, with the difference that the users' balances will be returned at the block number specified in the vote. Using this getter, a delegate can easily compose a list of those eligible to participate in a specified vote.

But there is one more thing to consider. As mentioned above, a delegate can't vote on behalf of someone who voted for themself earlier. To consider this fact when preparing a list of delegated voters, the delegate must know the `VoterState` of each voter for the specified voting.

`getVoterStateMultipleAtVote`

- **Arguments**:
  - `uint256 _voteId`;
  - `address[] _voters`.
- **Visibility**: `external`
- **Mutability**: `view`
- **Modifiers**: `voteExists(_voteId)`
- **Returns**: `(VoterState[] memory voterStatesList)`

This method returns an array of `VoterState` values for specified voting and for each specified voter address.

To display the fact of delegates participating in voting, the `VoterState` enum was supplemented by new values:

```solidity
enum VoterState { Absent, Yea, Nay, DelegateYea, DelegateNay }
```

This and other changes to the contract code will be examined in the next section

### Changes made to existing code

Integrating the delegation feature into the `Voting` contract is inevitably connected with a need to change existing code.

#### Changes in the `_vote`

First of all, the `bool _isDelegate` argument was added to the internal `_vote` method. Considering the updated enum, this addition allows us to update `VoterState` properly.

```solidity
...
if (_supports) {
    vote_.yea = vote_.yea.add(voterStake);
    vote_.voters[_voter] = _isDelegate ? VoterState.DelegateYea : VoterState.Yea;
} else {
    vote_.nay = vote_.nay.add(voterStake);
    vote_.voters[_voter] = _isDelegate ? VoterState.DelegateNay : VoterState.Nay;
}
...
```

To optimize delegates' gas spending, it was decided to introduce two more arguments to the `_vote` method:

- `uint256 _votingPower` - since the call of `token.balanceOfAt(blockNumber)` leads to the increase of gas amount needed for the transaction execution, it was decided to pass the `votingPower` value to the `_vote` directly from an external method.
- `VotePhase _votePhase` - for the same reason, it is proposed to calculate the `votePhase` value from an external method.

Another small optimization is the additional `if` statement that wraps the change in the number of tokens voted for one or another option.

```solidity
if (state != VoterState.Absent) {
    if (state == VoterState.Yea || state == VoterState.DelegateYea) {
        vote_.yea = vote_.yea.sub(_votingPower);
    } else { // Remaining states are Nay and DelegateNay
        vote_.nay = vote_.nay.sub(_votingPower);
    }
}
```

#### Changes in the invariants

As described above, some of the requirements for delegatd voting are the same as those for direct voting. However, due to the fact that in the case of delegated voting these requirements are checked inside of a loop, the decision was made to optimize t hese checks by rearranging them in such a way as to minimize the number of calls within the loop. This led to:

- removal of `_canVote` method;
- introduction of `_isValidPhaseToVote` method;
- removal of `_isVoteOpen` method;
- changes in functions `vote` and `canVote`.

It was also decided to remove `_isVoteOpen` and use updated `getVotePhase` instead.

`_isValidPhaseToVote`

- **Arguments**:
  - `VotePhase`;
  - `bool _supports`.
- **Visibility**: `internal`
- **Mutability**: `view`
- **Returns**: `(bool)`

This function returns true if:

- the vote is open, meaning not expired and not yet executed;
- `_supports` can be applied at the current vote phase.

Since `_canVote` was removed, new functions replaced it in the existing code.

### Deprecations

During the development, it was decided to remove the `bool _castVote` argument from the internal `_newVote` method. Since that there's no more "execute before timelock" option, it seems that creating new vote and casting vote in the same transaction is not useful anymore, so in the sake of code consistency, it was decided to remove the following lines:

```solidity
...
if (_castVote && canVote(voteId, msg.sender)) {
    _vote(voteId, true, msg.sender, false);
}
...
```

This change was followed by changes in `newVote` and `forward` functions.

## Links

- [Voting app](https://github.com/lidofinance/aragon-apps/tree/master/apps/voting)
- [Simple Delegation PR](https://github.com/lidofinance/aragon-apps/pull/34)
