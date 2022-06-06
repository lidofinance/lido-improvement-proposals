---
lip: 10
title: Proxy initializations and LidoOracle upgrade
status: Implemented
author: Artyom Veremeenko, Sam Kozin, Mihail Semenkin, Eugene Pshenichnyy, Eugene Mamin
discussions-to: https://research.lido.fi/t/lip-10-proxy-initializations-and-lidooracle-upgrade/1616
created: 2022-01-25
updated: 2022-06-06
---

# Proxy initializations and LidoOracle upgrade

This is a technical proposal related to Lido contracts implementation details. It doesn't change product logic.

In a nutshell, we propose an approach for writing initializer functions in Lido proxy contracts which need to be upgraded from time to time. We also propose corresponding changes to `LidoOracle` contract.

## Motivation

There are proxy contracts in Lido which might need upgrades from time to time. This gives birth to several questions related to proxy contract initialization. A brief intro to the topic of proxy contracts initialization can be found [here](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#the-constructor-caveat).

It's a common pattern to have a function called `initialize()` in proxy implementation contract. This function's code is to be executed once right after the contract deployment.
We might need to make an upgrade of implementation which requires additional initialization. The problem under consideration arises when we also need to be able to deploy the contract from scratch. For example during deployments to other (test) networks and local integration testing.

## Background

At the moment only `LidoOracle` contract has encountered this "initializer" problem. The solution currently adopted in `LidoOracle` contract doesn't allow to deploy the contract from scratch.

Current solution is:
- to keep track of contract version in storage;
- increment version during upgrades, which require additional initialization;
- have a single initialize function named `initilize_vN` which performs initializations from version `N-1` to version `N`.

Original `initialize()` function was removed in [this commit](https://github.com/lidofinance/lido-dao/commit/f8406543fa924cea3cec5f0c69e039c859aad92d). Function `initialize_v2()` was added in [this one](https://github.com/lidofinance/lido-dao/commit/46de2b259de84ddd388bb3e0993828a74042d158).
Thus, at the moment it's not possible to deploy `LidoOracle` from scratch as it lacks part of it's initialization logic.

## Solution

Note that there are at least multiple general pros and cons of a solution to consider:
- clarity of intention and straightforwardness of usage;
- implementation difficulty and error-proneness;
- final system difficulty;
- gas price of contract deployment.

### Initializer vs. setter functions

The first question to consider is when to add `initialize_vX()` and when just to add some `setXYZ()` function and call it right after deployment.

We propose to add an initializer on upgrade only if additional initialization is required for the proper functioning of the upgraded contract. Otherwise just to add setter functions.

### Change in version numbering

In `LidoOracle` there is a quite misleading discrepancy between the value of`CONTRACT_VERSION` in storage and `N` in `initialize_vN`. The version in storage is `1` but the last initializer is `v2`.

We propose to change version numbering as follows:
- v0: storage state right after executing contract's deployment bytecode
- vN: storage state after calling either
    - `initialize()` after initial deployment, where `N` is current contract's code version
    - `initialize_vN()` during upgrade from version `N-1` to version `N`

Thus we propose to change the version number in `LidoOracle` storage from `1` right to `3` on the next contract upgrade.

### Should we keep intermediate initializers?

In general, there are at least two approaches to consider:
1. keep intermediate `initialize_vN` functions from each contract upgrade;
2. don't keep intermediate `initialize_vN` functions, leave only two: cumulative `initialize()` and `initialize_vN` for the last upgrade

#### Option 1 (don't keep)

To keep just two initializer functions: cumulative `initialize()` to go from clean state to up-to-date state and `initialize_vN` to go from state at version `N-1` to up-to-date state.

Here is a simplified example code of an upgrade of proxy implementation from version 2 to version 3 without intermediate initializers.

```javascript

contract ProxyImplementation {
    bytes32 internal constant CONTRACT_VERSION_POSITION = 0x...;

    function initialize(address _foo, uint64 _bar) {
        ... // code performing all initializations up to version 3

        initialize_v3(_bar);
    }

    function _initialize_v3(uint64 _bar) external {
        require(CONTRACT_VERSION_POSITION.getStorageUint256() == 2, "WRONG_BASE_VERSION");
        ...
        CONTRACT_VERSION_POSITION.setStorageUint256(3);
    }
```

For a complete example see section "LidoOracle upgrade" at the end of this doc.

#### Option 2 (keep)

Keep all initializer functions `initilize_vX` where `X` goes from 1 to N, where N is the up-to-date version. Add a cumulative `initialize()` which calls every `initialize_vX` function in order.

[Here](https://github.com/krogla/contract_versions/blob/master/contracts/ContractVersions.sol) is an illustration of the approach, using neat modifiers.

Pros:
- no need to mock initializer of the previous version of the contract to write tests for upgrading;
- gives a clearer perspective on contract storage history.

Cons:
- adds entities (intermediate `initialize_vX` functions of no use);
- forces to keep obsolete code, which could be merged into cumulative `initialize()` otherwise;
- increases contract deployment cost.

### Restrictions on calling initilize functions
We still need to use `onlyInit` modifier for `initialize()` in Aragon contracts as Aragon authentication modifiers require it ([see the details](
https://hack.aragon.org/docs/aragonos-building#constructor-and-initialization)).

We also propose not to add any authentication restrictions on calls of `initialize()` and `finalizeUpgrade_vN`. Risk of an attack exploiting this is quite negligible but the restriction would require adding one more role and complicating the code.

### Naming and code composition

We also propose to separate `initialize_vN` into two functions: `_initialize_vN` and `finalizeUpgrade_vN`. Function `finalizeUpgrade_vN` is to be called after contract's source upgrade and it's name expresses the intention clearer. Function `_initialize_vN` is for internal use in cumulative `initialize()` and `finalizeUpgrade_vN`.

Function `finalizeUpgrade_vN` is to be called once and must revert if base version is not correct.

## LidoOracle upgrade

We propose to update `LidoOracle` contract according to the solution chosen.

[Here](https://github.com/lidofinance/lido-dao/pull/374) is `LidoOracle` upgrade according to the solution option 1.
