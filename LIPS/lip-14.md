---
tags: LIP
lip: 14
blockchain: ethereum
title: Protocol safeguards. Staking rate limiting
status: Proposed
author: Eugene Mamin, Sam Kozin, Eugene Pshenichnyy, Aleksei Potapkin
discussions-to: TBA
created: 2022-05-05
updated: 2022-05-05
---

# Protocol safeguards. Staking rate limiting

Add staking rate limiting feature to have the soft moving cap for stake amount per desired period.

## Abstract
We propose to implement rate limiting mechanics by using the following conventions:

- Staking can be paused completely by `pauseStaking()` method
- Staking can be resumed then while having a rate limit set or without rate limit at all
- Rate limit is managed by two parameters:
    - `maxStakingLimit` - amount of ETH that should be staked in the protocol to meet the limit
    - `stakeLimitIncreasePerBlock` - speed of replenishing the staking capacity of a protocol in ETH per block 
    
Thus if the protocol reaches the limit at the beginning of the period, then the staking capacity starts gradually increasing to the `maxStakingLimit` as time passes. The approach prevents discrete pauses in protocol's staking ability.

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

After [The Merge](https://ethereum.org/en/upgrades/merge/) we're expecting APR increase and a corresponding boom of staking. According to the [analysis](https://blog.lido.fi/modelling-the-entry-queue-post-merge-an-analysis-of-impacts-on-lidos-socialized-model/), it may induce additional demand on validation entry queue. That can lead to APR decrease, as more ETH will be locked in a queue to be actually staked. So, we want to have a lever to control the incoming stream of ETH.

## Specification

We propose to accomplish the `Lido.sol` contract with additional functions to control staking rate limit and view its state, and slightly edit its existent functions.

### `Lido.sol` changes

```solidity
function pauseStaking() external
```

- method that sets `maxStakingLimit` to 0. All subsequent staking calls to the protocol will be reverted
- secured by `STAKING_PAUSE_ROLE`

``` solidity
function resumeStaking(
    uint96 _maxStakeLimit,
    uint96 _stakeLimitIncreasePerBlock
) external
```

- method that allows to resume the staking after it was paused while setting the rate limits. It sets stake limit equal to `_maxStakeLimit`. After that, each staking call will reduce the limit by the staked amount. And stake limit will restore with `_stakeLimitIncreasePerBlock` speed until it will be `_maxStakeLimit` again.
- secured by `STAKING_RESUME_ROLE`

``` solidity
function isStakingPaused() external view returns (bool)
```
- returns `true` if `maxStakeLimit` is set to 0

``` solidity
function calculateCurrentStakeLimit() external view returns (
    uint256 currentStakeLimit,
    uint256 maxStakeLimit,
    uint256 stakeLimitIncreasePerBlock
)
```

- returns actual staking parameters with `currentStakeLimit` calculated for current block

Also, there are changes in `_submit()` implementation, that:
- checks if the staked amount is less or equal to `currentStakeLimit` and reverts if it's not
- updates `prevStakeLimit`, reducing it by amount staked

### New `StakeLimitUtils` library

To save a gas, we use only one additional storage slot containing all rate limiting parameters and state. Thus, to preserve level of abstractions in the `Lido.sol` contract and hide rate limiting implementation complexity, we have extracted low-level bit-twiddling and calculations into the separate `StakeLimitUtils` library. 

## Considerations

### Access control
We are adding the set of roles to control the rate limit:
`STAKING_PAUSE_ROLE` and `STAKING_RESUME_ROLE`.

### Contract size
We've got to contract size limit and have to introduce some additional changes that allow us to fit in the limit
- Replacing `auth()` modifier with `_auth()` internal function appeared to reduce contract size substantially.
- Combining three managing methods `setOracle()`, `setTreasury()` and `setInsuranceFund()` into the single one `setProtocolContracts()`
- Inlining some internal convenience methods like `_getFee()`  or `_submitted()`

## Backward compatibility

Stake rate limiting maybe disabled at all by passing special constant. Thus, the behavior will stay the same (except just one extra SLOAD operation to check the rate limiting slot state).
While enabled, it can affect the 3rd party integrations as all the transactions that will hit the limit will be reverted. It can be mitigated by having a fallback route to get stETH on market if the limit is reached. 

## Gas price effects

Stake rate limiting has a tolerable impact on the gas.

#### Scenario 1: rate limiting is enabled

If staking limit is set, we expect that overall spending increase will be lower than 6k gas (1 SLOAD + 1 SSTORE (~5k gas) + some more code for memory). While the current limit is ~83k gas, there is about extra 7% gas per single staking request.

#### Scenario 2: rate limiting is disabled

There is only one extra SLOAD needed to check the limit's slot value. Extra gas spending will be about 2k gas (2.5% gas increase per single staking request)

## Pull request

The proposed changes can be found in the following [`PR #410`](https://github.com/lidofinance/lido-dao/pull/410/files).