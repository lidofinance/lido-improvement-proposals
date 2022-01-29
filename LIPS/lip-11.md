---
lip: 11
title: Add a transfer shares function for stETH
status: Proposed
author: Eugine Mamin, Sam Kozin, Artyom Veremeenko
discussions-to: https://research.lido.fi/t/lip-11-add-a-transfer-shares-function-for-steth/1622
created: 2022-01-27
updated: 2022-01-27
---

# Add a transfer shares function for stETH

## Simple summary
Add the `transferShares` function which allows to transfer `stETH` in a "rebase-agnostic" manner: transfer in terms of shares amount (not the token amount).

## Abstract

We propose to add `transferShares` function into the `stETH` contract which accepts shares amount as input and performs shares movement from one account to another. We also propose to add event  `TransferShares` which must be emitted along with `Transfer` event. We propose that calling `transferShares(recipient, sharesAmt)` should lead to exactly the same outcome as calling `transfer(recipient, sharesAmt * sharePrice)`. Including that both `transferShares` and `transfer` should emit `TransferShares` and `Transfer` events. The only difference being the measurement unit of the input amount.

## Motivation

`stETH` is a [rebasing token](https://docs.lido.fi/contracts/lido#rebasing). Usually, we transfer `stETH` using ERC-20 `transfer` and `transferFrom` functions which accept as input amount of `stETH`, not the amount of the underlying shares.

Sometimes we'd better operate with shares directly to avoid possible rounding issues (the first clear example is [LIP-6 proposal](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-6.md), see `recoverExcessStETH`). Rounding issues usually could appear after a token rebase.

## Specification

### New `transferShares` function

    /**
     * @notice Moves `_sharesAmount` token shares from the caller's account to the `_recipient` account.
     *
     * @return amount of transferred tokens.
     * Emits a `TransferShares` event.
     * Emits a `Transfer` event.
     *
     * Requirements:
     *
     * - `_recipient` cannot be the zero address.
     * - the caller must have at least `_sharesAmount` shares.
     * - the contract must not be paused.
     *
     * @dev The `_sharesAmount` argument is the amount of shares, not tokens.
     */
    function transferShares(address _recipient, uint256 _sharesAmount) public returns (uint256)

### `TransferShares` event
Add event with this signature:

    /**
      * @notice An executed shares transfer from `sender` to `recipient`
      */
    event TransferShares(
        address indexed from,
        address indexed to,
        uint256 sharesValue
    );

### Update `transfer` and `transferFrom` functions

In the `_transfer` function emit the `TransferShares` event in addition to emitting the `Transfer` event.
Add no changes to `transferFrom` as it calls `_transfer` internally.


### Update minting function

In the `_emitTransferAfterMintingShares` function of the `Lido` contract emit the `TransferShares` event in addition to emitting the `Tranfer` event.

## Backward compatibility

We preserve the existing `Transfer` event signature cause it's defined by ERC-20. That's why we introduce the new `TransferShares` event instead of adding another arg for the existing `Transfer` event.

## Gas price effects

Emitting the `TransferShares` event costs approximately 1900 additional gas, depending on the execution context.

### Transfer of stETH

An addition of emitting the `TransferShares` event increases the cost of every stETH transfer (call to Lido contract's `transfer(...)`) by ~3.6% (1967 gas).

### Submitting ETH to Lido contract

Gas price of submitting ETH for minting stETH is also affected because on minting we need to emit `TransferShares(address(0), ...)` as well as `Transfer(address(0), ...)` event.

Slight 1891 gas (~2.2%) increase in costs for every call to the Lido contract's `submit(...)` function.

### Handling LidoOracle report

Call to LidoOracle's `reportBeacon` which leads to consensus will emit `2 + N` `TransferShares` events, where `N` is the amount of Node Operators stored in the registry (14 at the moment). Thus, cost of calling `reportBeacon` will increase by ~30000 gas which is ~6%. The cost is paid approximately every day on a beacon chain state report by the Oracle who reports by calling `reportBeacon` "the last". It happens to be [0x007DE4a5F7bc37E2F26c0cb2E8A95006EE9B89b5](https://etherscan.io/address/0x007de4a5f7bc37e2f26c0cb2e8a95006ee9b89b5) these days.

## Pull request

The proposed change of the `StETH` contract can be found in PR linked to [this issues](https://github.com/lidofinance/lido-dao/issues/376). It also refactors `ISTETH` contract interface to remove obsolete code.
