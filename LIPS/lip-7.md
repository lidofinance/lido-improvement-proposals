---
title: Introduce a composite oracle beacon report receiver
tags: LIPs
blockchain: ethereum
lip: 7
status: WIP
author: Eugine Mamin, Sam Kozin, Eugene Pshenichniy
discussions-to: https://research.lido.fi/t/lip-6-in-protocol-coverage-proposal/1468/10
created: 2022-01-14
updated: 2022-01-24
---

# Introduce a composite oracle beacon report receiver

## Simple summary

To have a possibility of using two or even more Lido Oracle beacon report intercepting callbacks, we introduce a composite oracle beacon report receiver. The last one implements the `IBeaconReceiver` interface and internally holds an array of nested `IBeaconReceiver` instances and supplies iterative execution by calling `processLidoOracleReport` in a loop.

## Abstract

The `OrderedCallbacksArray` contract implements the `IOrderedCallbacksArray` interface (to add/insert/remove/view the callbacks) adding the access modifiers to allow storage changing calls (add/insert/remove) only originated by the `Voting` contract (having `require(msg.sender == Voting.address)` ). The second proposed contract (`CompositePostRebaseBeaconReceiver`) inherits `OrderedCallbacksArray` simultaneously implementing the `IBeaconReportReceiver` interface. It is allowed to call the `processLidoOracleReport` function only by the `LidoOracle` contract (having `require(msg.sender == LidoOracle.address)`).

In the end, the `CompositePostRebaseBeaconReceiver` contract is a top-level entity implementing the current proposal.

## Motivation

Currently, the `LidoOracle` [contract](https://docs.lido.fi/contracts/lido-oracle) provides only one slot for a quorum report intercepting callback (see [LidoOracle.setBeaconReportReceiver](https://docs.lido.fi/contracts/lido-oracle#setbeaconreportreceiver)). We plan to occupy this slot soon with a newly developed contract implementing the [LIP-6 coverage application mechanism](https://research.lido.fi/t/lip-6-in-protocol-coverage-proposal/1468). In contrast, we have a technical vision of the future cross-chain/L2 [upgrades](https://hackmd.io/f3416OoaS2e1l2xu_bX6SA), which may require appending additional callbacks to propagate oracle beacon reports.

## Specification

We propose the following contracts interface. The code below presumes the Solidity v0.8 syntax.

### `OrderedCallbacksArray` contract

The contract provides callbacks storage in the form of addresses dynamic array.

```solidity
modifier onlyVoting()
```
Enforces `msg.sender` to equal `voting` address set upon the contract construction.

```solidity
constructor(address _voting)
```
Init contract and store the provided `_voting` address to arm the `onlyVoting` modifier.
See: `onlyVoting`.

```solidity
function addCallback(address _callback) external override onlyVoting;
```
Adds the provided `_callback` element at the end of the callbacks array (i.e., after its current last element).
* Reverts if `_callback` address is zero.
* Reverts if `msg.sender` is not equal to the stored `voting` address.
* Reverts if the new length of the callbacks array becomes greater than the `MAX_CALLBACKS_CNT` contract-wide constant.
* Emits the `CallbackAdded` event.
See: `callbacksLength`.

```solidity
function insertCallback(address _callback, uint256 _atIndex) external override onlyVoting;
```
Inserts the `_callback` element at the specified by the `_atIndex` param location in the callbacks array.
Elements at and after the `_atIndex` position are shifted one position right to preserve the existing invocation order.
* Reverts if `_callback` address is zero.
* Reverts if `msg.sender` is not equal to the stored `voting` address.
* Reverts if `_atIndex` greater than the length of the callbacks array.
* Reverts if new length of the callbacks array becomes greater than the `MAX_CALLBACKS_CNT` contract-wide constant.
* Emits the `CallbackAdded` event.
See: `callbacksLength`.

```solidity
function removeCallback(uint256 _atIndex) external override onlyVoting
```
Removes element with a given by the `_atIndex` param position from the callbacks array.
Elements after the `_atIndex` position are shifted one position left to preserve the existing invocation order.
* Reverts if `_atIndex` is equal to or greater than the length of the callbacks array.
* Reverts if `msg.sender` is not equal to the stored `voting` address.
* Emits the `CallbackRemoved` event.
See: `callbacksLength`.

```solidity
function callbacks(uint256 _atIndex) external view returns (address);
```
Get the callback element at the specified by the `_atIndex` param position.
* Reverts if `_atIndex` is equal to or greater than the length of the callbacks array.
See: `callbacksLength`.

```solidity
function callbacksLength() external view returns (uint256);
```
Get the current callbacks array length.

```solidity
event CallbackAdded(address indexed callback, uint256 atIndex);
```
Emitted when a new `callback` is added at the `atIndex` position.
See: `insertCallback`, `removeCallback`.

```solidity
event CallbackRemoved(address indexed callback, uint256 atIndex);
```
Emitted when a `callback` was removed from the `atIndex` position.
See: `removeCallbacks`.

### `CompositePostRebaseBeaconReceiver` contract

The contract inherited from `OrderedCallbacksArray` to implement a composite design pattern applicable for the `LidoOracle` contract as `IBeaconReportReceiver`.

```solidity
modifier onlyOracle()
```
Enforces the `msg.sender` to equal `oracle` address which is set upon the contract construction.

```solidity
constructor(address _voting, address _oracle) OrderedCallbacksArray(_voting);
```
Init contract and store the provided `_voting` and `_oracle` addresses to arm the `onlyVoting` and `onlyOracle` modifiers.
See: `OrderedCallbacksArray.onlyVoting`, `onlyOracle`.

```solidity
function processLidoOracleReport(
    uint256 _postTotalPooledEther,
    uint256 _preTotalPooledEther,
    uint256 _timeElapsed
) external override onlyOracle
```
Implements the `IBeaconReceiver` interface, which is suitable for the `LidoOracle` contract.
Iteratively calls `callback.processLidoOracleReport` for each stored `callback` in the callbacks array preserving the order.

## Backward compatibility

* The proposed `CompositePostRebaseBeaconReceiver` could be used with only one `IBeaconReportReceiver` callback inside, providing the same behavior as before by using the wrapped callback directly (without wrapping into a composite receiver).
* Even if the callbacks array is empty, the existing behavior preserves, i.e., there are no additional side-effects incurred for the lido oracle reports except the gas spendings.

## Gas effects

In the case of `ComositePostRebaseBeaconReceiver` used with only one `IBeaconReportReceiver` callback set, spendings increase approximately by the ~4600 gas compared to direct usage of the underlying callback with `LidoOracle` (without composite adapter).

## Security considerations

#### Upgradability
The proposed contracts are non-upgradable for the sake of simplicity. In case of emergency, we can disable them entirely by calling `LidoOracle.setBeaconReportReceiver(address(0))` and redeploy newly developed versions later from scratch without incurring additional proxy-initialization logic and state transitions.

#### Access control (permissions model)
There are two permissioned addresses introduced explicitly: `Voting` and `LidoOracle`. Only `Voting` can add/insert/remove callbacks into the `CompositePostReabaseBeaconReceiver` to make it feasible only by the Lido DAO direct vote-backed will. Only `LidoOracle` is allowed to call `processLidoOracleReport` to prevent propagating various fake oracle reports by the added callbacks or execution in an arbitrary time moment.

If one of the callbacks throws an exception then the whole transaction reverts and other callbacks effects also rollbacks.

#### Misbehaving callback
A misbehaving callback could be easily removed from the current bunch by the DAO vote calling the `remove(misbehavingCallbackIndex)`

## Links

* [LIP-6: in-protocol coverage application mechanism](https://research.lido.fi/t/lip-6-in-protocol-coverage-proposal/1468)
* [Original request launched this proposal](https://research.lido.fi/t/lip-6-in-protocol-coverage-proposal/1468/10)
* [An overview of the design options to break into the L2/cross-chains](https://hackmd.io/f3416OoaS2e1l2xu_bX6SA) — contains proposals to setup additional beacon report receivers.
* [`LidoOracle`](https://docs.lido.fi/contracts/lido-oracle) documentation
* [`CompositePostRebaseBeaconReceiver`](https://github.com/lidofinance/lido-dao/blob/feature/LIP-6/contracts/0.8.9/CompositePostRebaseBeaconReceiver.sol) reference implementation
* [`OrderedCallbacksArray`](https://github.com/lidofinance/lido-dao/blob/feature/LIP-6/contracts/0.8.9/OrderedCallbacksArray.sol) reference implementation
* [`IOrderedCallbacksArray`](https://github.com/lidofinance/lido-dao/blob/feature/LIP-6/contracts/0.8.9/interfaces/IOrderedCallbacksArray.sol) interface definition
* [`IBeaconReportReceiver`](https://github.com/lidofinance/lido-dao/blob/feature/LIP-6/contracts/0.8.9/interfaces/IBeaconReportReceiver.sol) interface definition
