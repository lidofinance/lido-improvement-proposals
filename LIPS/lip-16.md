---
tags: LIP
lip: 16
blockchain: ethereum
title: Protocol safeguards. Two-phase voting.
status: Proposed
author: Alexey Potapkin, Sam Kozin
discussions-to: https://research.lido.fi/t/proposal-last-minute-vote-mitigation/2162/14
created: 2022-06-01
updated: 2022-06-07
---

# Protocol safeguards. Two-phase voting

## Simple summary

Split the voting timeframe into two phases, one — for voting 'for' and 'against'
and the other — for voting only 'against'

## Abstract

We propose to split each vote timeframe into two phases while maintaining 
the overall vote time to be [72h](
https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-4.md):

- Main phase, where one can vote both for and against — 48h
- Objection phase, where one is only allowed to vote against — 24h

```
                         Vote time (72h)
|-------------Yes or No--------------|---------No only----------|
           Main phase (48h)              Objection phase (24h)
```

Proposed timeframes can be changed later using the added methods, using newly 
added methods of the `Voting` contract.
Furthermore, as an additional change, we propose to disable the mechanism that 
allowed the premature execution of a vote if more than a half of the total 
issued tokens voted supporting the outcome.  

## Motivation

Firstly, it allows us to avoid a governance takeover by pushing 
the untelegraphed changes via ['last moment vote'](https://hackmd.io/BlfaMs30TVeK8OyU6OI-1A). 

Secondly, it introduces the timelock-like period when voting is decided,  
3rd-party integrations can prepare its code for the changes, and stETH-holders
can exit if they don't like the outcome. That's the reason for disabling the 
`executeIfdecided` mechanism.

As a side effect, we're getting a governance process easier to cancel the vote 
than to make it pass and making Lido DAO governance more conservative and 
resistant to changes. 

As for the timings, we'd like to retain the overall duration to be 72h to have
the whole process, including preparing the vote and enacting it to fit in 
a working week.

## Specification

We propose to add the following field to the `Voting.sol` contract to store 
the duration of the Objection phase. 

```solidity
uint64 public objectionPhaseTime;
```
The duration of the Main phase is calculated as `voteTime - objectionPhaseTime`.

The respective method is added to manage the mentioned field:

```solidity
function unsafelyChangeObjectionPhaseTime(uint64 _objectionPhaseTime)
        external
        auth(UNSAFELY_MODIFY_VOTE_TIME_ROLE);
```
- protected by `UNSAFELY_MODIFY_VOTE_TIME_ROLE` as well as
`unsafelyChangeVoteTime` was. We decided to avoid adding a new role, because we 
don't expect these methods to be used separately.
- new event `event ChangeObjectionPhaseTime(uint64 objectionPhaseTime)`
is emitted in this method
- `require` with `VOTING_OBJ_TIME_TOO_BIG` error is introduced here to maintain
`voteTime > objectionPhaseTime` invariant. The similar `require` is added to 
the `initialize()` and `unsafelyChangeVoteTime()` methods.

__NB!__ The method should be used with caution as it changes all the votes 
retroactively, so it's marked as unsafe in its name.

New `require` in `vote()` checks that only 'no' votes are accepted during the 
objection phase with 

```solidity
function vote(
    uint256 _voteId, 
    bool _supports, 
    bool _executesIfDecided_deprecated)
external voteExists(_voteId) {
    //...
    require(!_supports || _getVotePhase(votes[_voteId]) == VotePhase.Main, 
        ERROR_CAN_NOT_VOTE);
    //...
}
```
To disable immediate execution on decide (when more than 50% of tokens is 
supporting the vote) we're deprecating `executeIfDecided` argument of 
`newVote()` and `vote()` methods and removing all related logic. The event
```solidity
event CastObjection(uint256 indexed voteId, address indexed voter, uint256 stake);
```
is added to be emitted along the `CastVote` if someone voted during 
the Objection phase.

`initialize()` is updated accordingly:
```solidity
function initialize(
    MiniMeToken _token, 
    uint64 _supportRequiredPct, 
    uint64 _minAcceptQuorumPct, 
    uint64 _voteTime, 
    uint64 _objectionPhaseTime //+
) external onlyInit;
```

Special enum was added to differ various phases through the voting process:
```solidity
enum VotePhase { Main, Objection, Closed }
```
and a getter to obtain the current phase: 
```solidity
function getVotePhase(uint256 _voteId) external view voteExists(_voteId) 
    returns (VotePhase);
```
also, current `phase` is appended to the getVote return values list

```solidity
function getVote(uint256 _voteId)
    public
    view
    voteExists(_voteId)
    returns (
        bool open,
        bool executed,
        uint64 startDate,
        uint64 snapshotBlock,
        uint64 supportRequired,
        uint64 minAcceptQuorum,
        uint256 yea,
        uint256 nay,
        uint256 votingPower,
        bytes script
        bytes script,
        VotePhase phase //+
    );
```

## Consideration

### Access control

The role `UNSAFELY_MODIFY_VOTE_TIME_ROLE` was used to protect 
`unsafelyChangeObjectionPhaseTime` as well as `unsafelyChangeVoteTime` as we 
don't expect them to be used separately. 

And to maintain consistency, `authP` modifier in both methods is replaced by 
`auth`, because one cannot distinguish parameters of one method from the other 
one.
 
### Backward compatibility

All changes are designed with backward compatibility in mind and UI that 
supported the previous version should remain functional. So the only discrepancy
are going to be the inability to vote positively during the Objection phase

The overall vote duration remains 72h, but the main phase ends earlier, so DAO
should correct their communications to voters.

The `CastVote` event is emitted for both the Main and Objection phases as usual,
but there is an additional `CastObjection` event emitted during the Objection 
phase only.

### Last moment vote mitigation

The mitigation does not resolve the 'last moment cancel' part of the problem and
even introduces an objection period which can be used to unexpectedly block any
decision. But it's an expected behaviour and considered relatively harmless 
as soon as the vote can be relaunched

## Pull request

The proposed changes can be found in the following [`PR #8`](
https://github.com/lidofinance/aragon-apps/pull/8)


