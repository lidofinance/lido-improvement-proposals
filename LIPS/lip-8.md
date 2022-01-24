---
lip: LIP-8
title: Increase keysOpIndex in assignNextSigningKeys
status: Proposed
author: George Avsetsin, Artyom Veremeenko
discussions-to: https://research.lido.fi/t/lip-8-increase-keysopindex-in-assignnextsigningkeys/1608
created: 2022-01-24
updated: 2022-01-24

---

# Increase keysOpIndex in assignNextSigningKeys

## Simple summary
We propose to call `_increaseKeysOpIndex` if the number of used keys has changed during the call of `assignNextSigningKeys` (contract `NodeOperatorsRegistry`).

## Abstract

Calling `assignNextSigningKeys` method changes the "usedSigningKeys" properties of operators, which is responsible for the number of used and unsigned keys. By explicitly calling `_increaseKeysOpIndex` we can add an intrusive indicator for the off-chain listeners.

## Motivation

With current protocol implementation `keysOpIndex` counter is not incremented when a deposit happens. Thus it's difficult to track used keys with off-chain monitoring services. We need an ad-hoc loop to manually refresh used keys which looks like a suboptimal workaround.

## Specification

### Update `assignNextSigningKeys` function

Add calls to `_increaseKeysOpIndex()` at the two execution branches if returned assigned keys number is non-zero.

## Gas price effects

The proposed change will slightly increase gas cost of `depositBufferedEther` call for Depositor Bot. An additional call of `_increaseKeysOpIndex()` costs ~7000 gas and does not depend on number of deposits. Relative cost increment depends on amount of transaction. For example for a typical [transaction with 38 deposits](https://etherscan.io/tx/0x1722113d54960b7cda789e6f9561a56ad3d1c33c491dcb9ee6825e151cdf18c7) the gas cost will increase by ~0.3% (total cost depends on the amount of 32ETH deposits done).

The chain of calls is:
- `DepositSecurityModule::depositBufferedEther` 
- `Lido::depositBufferedEther() auth(DEPOSIT_ROLE)`
- `NodeOperatorsRegistry::assignNextSigningKeys()`

Depositor Bot is run by Lido dev team ([see LIP-5](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-5.md#depositor-bot)) thus the additional price would be paid by Lido DAO.


## Links

The proposed change in `NodeOperatorsRegistry` contract can be found in PR linked to [this issues](https://github.com/lidofinance/lido-dao/issues/371).
