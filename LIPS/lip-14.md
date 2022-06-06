---
tags: LIP
lip: 14
blockchain: ethereum
title: Protocol safeguards. Staking rate limiting
status: Implemented
author: Eugene Mamin, Sam Kozin, Eugene Pshenichnyy, Aleksei Potapkin
discussions-to: https://research.lido.fi/t/announcement-merge-ready-protocol-service-pack/2184
created: 2022-05-05
updated: 2022-06-06
---

# Protocol safeguards. Staking rate limiting

Add a staking rate limiting feature to have the soft moving cap for the stake amount per desired period.

## Abstract
We propose to implement a rate limiting mechanics by using the following conventions:

- Staking resume/pause
- Staking limit set/remove
- Staking limit parameters

The rate limit is managed by two parameters:
- `maxStakingLimit` - max amount of ETH that could be staked into the protocol to meet the limit;
- `stakeLimitIncreasePerBlock` - speed of replenishing the staking capacity of the protocol in ETH per block.

Thus, if the protocol reaches the limit at the beginning of the period, then the staking capacity starts gradually increasing to the `maxStakingLimit` as time passes. The approach prevents discrete pauses in protocol's staking ability.

```
    ▲ Stake limit
    │.....  .....   ........ ...            ....     ... Stake limit = max
    │      .       .        .   .   .      .    . . .
    │     .       .              . .  . . .      . .
    │            .                .  . . .
    │──────────────────────────────────────────────────> Time
    │     ^      ^          ^   ^^^  ^ ^ ^     ^^^ ^     Stake events
```

## Motivation

After [The Merge](https://ethereum.org/en/upgrades/merge/) we're expecting APR increase and a corresponding boom of staking. According to the [analysis](https://blog.lido.fi/modelling-the-entry-queue-post-merge-an-analysis-of-impacts-on-lidos-socialized-model/), it may induce additional demand on the consensus layer validation entry queue. That can lead to an APR decrease, as more ETH will be locked in the queue to be actually staked. So, we want to have a lever to control the incoming stream of ETH.

## Specification

We propose to accomplish the `Lido.sol` contract with additional functions to control staking rate limit and view its state, and slightly edit its existent functions.

### `Lido.sol` changes


#### Function: pauseStaking
``` solidity
function pauseStaking() external
```

- a method that puts staking on pause: all subsequent staking calls to the protocol will be reverted
- the method is secured by `STAKING_PAUSE_ROLE`
- Emits `StakingPaused` event.

#### Function: resumeStaking

``` solidity
function resumeStaking() external
```
- the method is secured by `STAKING_CONTROL_ROLE`
- Emits `StakingResumed` event

#### Function: setStakingLimit
```solidity
function setStakingLimit(uint256 _maxStakeLimit, uint256 _stakeLimitIncreasePerBlock) external
```

The method sets stake limit equal to `_maxStakeLimit`. After that, each staking call will reduce the limit by the staked amount. And stake limit will restore with `_stakeLimitIncreasePerBlock` speed until it will be `_maxStakeLimit` again.
- the method is secured by `STAKING_CONTROL_ROLE`
- to disable stake limit one need to pass both zero args values
- Emits `StakingLimitSet` event

#### Function: removeStakingLimit
```solidity
function removeStakingLimit() external
```
The methods sets `maxStakeLimit` to zero
- Emits `StakingLimitRemoved` event

#### Function: isStakingPaused
``` solidity
function isStakingPaused() external view returns (bool)
```
- returns `true` if staking is on pause

#### Function: getCurrentStakeLimit
``` solidity
function getCurrentStakeLimit() public view returns (uint256)
```
Returns how much Ether can be staked in the current block
- 2^256 - 1 if staking is unlimited;
- 0 if staking is paused or if limit is exhausted.

#### Function: getStakeLimitFullInfo
```solidity
function getStakeLimitFullInfo() external view returns (
    bool isStakingPaused,
    bool isStakingLimitSet,
    uint256 currentStakeLimit,
    uint256 maxStakeLimit,
    uint256 maxStakeLimitGrowthBlocks,
    uint256 prevStakeLimit,
    uint256 prevStakeBlockNumber
)
```
Might be used for the advanced integration requests. Returns actual staking parameters for current block.

#### Event: StakingPaused
```solidity
event StakingPaused()
```

#### Event: StakingResumed
```solidity
event StakingResumed()
```

#### Event: StakingLimitSet
```solidity
event StakingLimitSet(uint256 maxStakeLimit, uint256 stakeLimitIncreasePerBlock)
```

#### Event: StakingLimitRemoved
```solidity
event StakingLimitRemoved()
```

Also, there are changes in `_submit()` implementation, that:
- checks whether the limit is set, if 'yes', checks the staked amount is less or equal to `currentStakeLimit` and reverts if it's not
- updates `prevStakeLimit`, reducing it by the amount just staked

### New `StakeLimitUtils` library

To save a gas, the implementation uses only one additional storage slot containing all rate limiting parameters and state. Thus, to preserve level of abstractions in the `Lido.sol` contract and hide rate limiting implementation complexity, low-level bit-twiddling and calculations are extracted into the separate `StakeLimitUtils` library.

## Considerations

### Access control
There is a set of roles to control the rate limit:
`STAKING_PAUSE_ROLE` and `STAKING_CONTROL_ROLE`.

### Contract size
The `Lido.sol` contract' size has almost reached the mainnet limit. To fit in the limit, there are implemented following size optimizations:
- `auth()` modifier is replaced with `_auth()` internal function
- three managing methods `setOracle()`, `setTreasury()` and `setInsuranceFund()` are packed into the single one `setProtocolContracts()`
- some internal convenience methods are inlined (`_getFee()`  or `_submitted()`)

## Backward compatibility

Stake rate limiting maybe disabled at all by passing special constant. Thus, the behavior will stay the same (except just one extra `SLOAD` operation to check the rate limiting slot state).

While enabled, it can affect the 3rd party integrations as all the transactions that will hit the limit will be reverted. It can be mitigated by having a fallback route to buy `stETH` on market if the limit is reached.

## Gas price effects

Stake rate limiting has a tolerable impact on the gas.

#### Scenario 1: rate limiting is enabled

If staking limit is set, we expect that overall spending increase will be lower than 6k gas (1 SLOAD + 1 SSTORE + some more code for memory). While the current limit is ~83k gas, there is about extra 7% gas per single staking request.

#### Scenario 2: rate limiting is disabled

There is only one extra SLOAD needed to check the limit's slot value. Extra gas spending will be about 2k gas (2.5% gas increase per single staking request)

## Pull request

The proposed changes can be found in the following [`PR #410`](https://github.com/lidofinance/lido-dao/pull/410/files).
