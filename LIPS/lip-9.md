---
lip: 9
title: Add an explicit log for the stETH burn events
status: Implemented
author: Eugene Mamin, Artyom Veremeenko
discussions-to: https://research.lido.fi/t/lip-9-add-an-explicit-log-for-the-steth-burn-events/1609
created: 2022-01-24
updated: 2022-06-06
---

# Add an explicit log for the stETH burn events

## Simple summary

Add a dedicated event to log the `burnShares` function invocation backing the on-chain storage state change which incurs stETH token rebase (all token holders get their balances increased).

## Abstract

We propose to include `SharesBurnt` event into the `StETH` contract and emit the proposed event on every `burnShares` invocation.
It would allow to complete off-chain logs and hold a new invariant for Lido: every `stETH` rebase must be covered with the some explicitly emitted event.

## Motivation

Burning of the `stETH` underlying shares is one of the on-chain effects causing stETH token rebase. We'd like to have all token rebases logged. It's a good thing to have in general. In particular, it would allow not to miss a rebase for the off-chain interactive UIs and to sync on-chain storage change with 3rd-party off-chain indexers and monitoring.

## Specification

Every token rebase caused by shares burning must emit `SharesBurnt` event indicating the caller account, burnt shares amount and the corresponding amount of tokens burnt.

### New `SharesBurnt` event

We propose to include `SharesBurnt` event into the `StETH` contract having the following signature:
```solidity
    /**
     * @notice An executed `burnShares` request
     *
     * @dev Reports simultaneously burnt shares amount
     * and corresponding stETH amount.
     * The stETH amount is calculated twice: before and after the burning incurred rebase.
     *
     * @param account holder of the burnt shares
     * @param preRebaseTokenAmount amount of stETH the burnt shares corresponded to before the burn
     * @param postRebaseTokenAmount amount of stETH the burnt shares corresponded to after the burn
     * @param sharesAmount amount of burnt shares
     */
    event SharesBurnt(
        address indexed account,
        uint256 preRebaseTokenAmount,
        uint256 postRebaseTokenAmount,
        uint256 sharesAmount
    );
```

### New version of the `burnShares` function

Cause the burning also decreases the total amount of minted shares, the balance of the account will be decreased by a different amount, so we need to calculate a pre/post token amount. So there are 3 new lines to add:

```solidity
uint256 preRebaseTokenAmount = getPooledEthByShares(_sharesAmount);
...
uint256 postRebaseTokenAmount = getPooledEthByShares(_sharesAmount);

emit SharesBurnt(_account, preRebaseTokenAmount, postRebaseTokenAmount, _sharesAmount);
```

## Gas price effects

Negligible: stETH burning is expected to happen no more than a few times per year, extra event costs are negligible.

## Links

The proposed change for the `StETH` contract can be found in PR linked to [this issue](https://github.com/lidofinance/lido-dao/issues/372).
