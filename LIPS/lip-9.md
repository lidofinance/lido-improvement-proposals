---
lip: lip-9
title: Add an explicit log for the stETH burn events
status: Proposed
author: Eugine Mamin, Artyom Veremeenko
discussions-to: https://research.lido.fi/t/lip-9-add-an-explicit-log-for-the-steth-burn-events/1609?u=arwer13
created: 2022-01-24
updated: 2022-01-24
---

# Add an explicit log for the stETH burn events

## Simple summary

Add a dedicated event to log the `burnShares` function invocation backing the on-chain storage state change which incurs stETH token rebase (all token holders get their balances increased).

## Abstract

We propose to include `StETHBurnt` event into the `StETH` contract and emit the proposed event on every `burnShares` invocation.
It would allow to complete off-chain logs and hold a new invariant for Lido: every `stETH` rebase must be covered with the some explicitly emitted event.

## Motivation

Burning of the `stETH` underlying shares is one of the on-chain effects causing stETH token rebase. We'd like to have all token rebases logged. It's a good thing to have in general. In particular, it would allow not to miss a rebase for the off-chain interactive UIs and to sync on-chain storage change with 3rd-party off-chain indexers and monitoring.

## Specification

Every token rebase caused by shares burning must emit `StETHBurnt` event indicating the caller account, burnt shares amount and the corresponding amount of tokens burnt.

### New `StETHBurnt` event

We propose to include `StETHBurnt` event into the `StETH` contract having the following signature:
```solidity
    /**
     * @notice An executed stETH burn event
     *
     * @dev Reports simultaneously stETH amount and shares amount.
     * The stETH amount is calculated before the burning incurred rebase.
     *
     * @param account holder of the burnt stETH
     * @param amount amount of the burnt stETH
     * @param sharesAmount amount of the burnt shares
     */
    event StETHBurnt(
        address indexed account,
        uint256 amount,
        uint256 sharesAmount
    );
```

### New version of the `burnShares` function

Two lines to add into the function body:

    uint256 amount = getPooledEthByShares(_sharesAmount);
    emit StETHBurnt(_account, amount, _sharesAmount);

## Gas price effects

Negligible: stETH burning is expected to happen no more than a few times per year, extra event costs are negligible. 

## Links

The proposed change for the `StETH` contract can be found in PR linked to [this issue](https://github.com/lidofinance/lido-dao/issues/372).
