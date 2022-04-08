---
title: Easy Track payments limit 
tags: LIPs
blockchain: ethereum
lip: lip-13
status: Implemented
author: Alexey Potapkin, Eugene Pshenichnyy, Victor Suzdalev, Eugene Mamin
discussions-to: https://research.lido.fi/t/lip-13-easy-track-payments-limit/1670
---

# Easy Track payment limits

## Simple summary

A simple security rule that limits Easy Track Motions while moving treasury 
funds.

## Abstract 

We propose to add the Aragon ACL permission that will whitelist only required 
asset types and limit their amounts for each payment initiated by Easy Track. 
The permission is set and updated only by explicit DAO vote.

## Motivation

[Easy Track](https://github.com/lidofinance/easy-track) proved to be a safe and 
effective tool that helped streamline DAO operations and get rid of some routine
votings. We want to extend it further, adding more Motion types, but some 
security concerns exist.

Currently, Motions have access to the Lido treasury (the 
[DAO Agent](https://etherscan.io/address/0x3e40D73EB977Dc6a537aF587D48316feE66E9C8c) 
contract) through 
[Finance](https://etherscan.io/address/0xB9E5CBB9CA5b0d659238807E84D0176930753d86), 
while access is authorized by 
[DAO Voting](https://etherscan.io/address/0x2e59A20f205bB85a89C53f1936454680651E618e)
employing Aragon [ACL](https://etherscan.io/address/0x9895f0f17cc1d1891b6f18ee0b483b6f221b37bb) 
permission. And this permission controls only the source of the token transfer 
and doesn't cover any other constraints (i.e., limits, destinations, frequency 
of payouts), which could be considered too permissive for Easy Track voting
model (requiring explicit objections to reject a vote, and, otherwise, let it 
pass).

## Specification

### Overview

We propose to add permission for Easy Track on creating payments in the Finance 
contract that will limit the amount of funds that can be transferred from 
the treasury in one transaction using hard-coded constraints, implemented 
by logical permission parameters. 

### Rationale
To provide a safe foundation for the further extension of Easy Track we want 
to move from unlimited payments to distinct monthly or yearly budgets for 
each Easy Track Motion type, managed by DAO.

To achieve that, there were several options considered: 
1. Simple limit to the transferring funds using ACL permission.
    - Requires only voting to be deployed 
    - Protects from the catastrophic scenarios
    - Doesn't support budgets
    - Can't discriminate between different Motions.
    - Could be combined with other options
2. `IACLOracle`-based auth delegate permission with budgets. 
    - Requires a separate contract to be deployed
    - Can enforce budgets
    - Can discriminate different Motions in a hacky way
3. Introduce limits and budgets to every individual Motion
    - Requires redeploying all existing Motions
    - Can enforce budgets
    - Increase the development difficulty for the new Motion types
4. Improve permission model of `EVMScriptFactoriesRegistry.sol`.
    - Requires redeploying the whole graph of Easy Track contracts
    - Can do budgets
    - Rather complex to implement

So, ACL permission with parameters was chosen as the quickest solution without 
any significant drawbacks. Although it does not provide an exhaustive set of 
required constraints, but it will reduce damage in almost every possible 
unwanted scenario and can be implemented swiftly.

Meanwhile, we're going to continue the hardening to enable fine-grained budgets 
and limits, but it requires more time to discuss, set the requirements, develop,
review and audit.

### Technical specification

According to the 
[specification](https://github.com/lidofinance/easy-track/blob/master/specification.md), 
there is the 
[`EVMScriptExecutor`](https://etherscan.io/address/0xFE5986E06210aC1eCC1aDCafc0cc7f8D63B3F977#code) 
contract that enacts a passed Motion to execution. If the Motion suppose 
to transfer any assets, it calls the

`newImmediatePayment(address _token, address _receiver, uint256 _amount, string _reference)`

method of the [Aragon Finance](https://etherscan.io/address/0x836835289a2e81b66ae5d95b7c8dbc0480dcf9da#code) 
contract. To authorize it, permission with `CREATE_PAYMENT_ROLE` is granted to 
`EVMScriptExecutor` through 
[Aragon ACL](https://etherscan.io/address/0x9f3b9198911054b122fdb865f8a5ac516201c339#code). 
Presently, this permission does not limit the amount of tokens to be transferred. 

1. We need to revoke the existing "wide" permission
`acl.revokePermission(EVMScriptExecutor, Finance, CREATE_PAYMENTS_ROLE)`
2. Then grant similar permission, but with parameters that limit the amount and 
the type of asset to be transferred.

`acl.grantPermissionP(EVMScriptExecutor, Finance, CREATE_PAYMENTS_ROLE, params)`,

where `params` is a logic predicate packed in `uint[]` in a special way 
described in 
[Aragon ACL documentation](https://hack.aragon.org/docs/aragonos-ref#parameter-interpretation). 
That predicate applied to the arguments of `newImmediiatePayment` method MUST 
resolve to true if the asset type and the amount is under the limits, and MUST 
be false otherwise.

So, the required logic can be described with the following pseudocode:
```
if (_token == ETH) return _amount <= ETH_LIMIT
else if (_token == LDO) return _amount <= LDO_LIMIT
else if (_token == stETH) return _amount <= stETH_LIMIT 
else return false
```
, where `_amount` and `_token` are the arguments passed to `newImmediatePayment`
method.

You can see the example of code, encoding these parameters, 
[here](https://github.com/lidofinance/scripts/blob/easytrack_permissions/scripts/vote_adding_permission.py#L36-L69). 

### Proposed limits

As initial values to start with, we propose:
- 1,000 ETH
- 1,000 stETH
- 5,000,000 LDO
- 100,000 DAI

Also, this limits will be working in whitelist mode and transferring all other 
ERC-20 tokens will be prohibited. 

### Test cases 

You can see the test cases for the proposal in the 
[relevant GitHub PR](https://github.com/lidofinance/scripts/pull/33)

### Various considerations

- There are no backward compatibility issues. The proposal requires only 
configuration changes that should limit unwanted behavior.
- It isn't a fool-proof solution replacing the Easy Track Motions review process. 
But it could reduce a cumulative damage significantly.
- Any permission change requires DAO voting.

## Links

* [Easy Track LIP](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-3.md)
* [Easy Track code](https://github.com/lidofinance/easy-track)
* [Guide to Easy Track](https://docs.lido.fi/guides/easy-track-guide)
* [Aragon ACL docs](https://hack.aragon.org/docs/aragonos-ref#parameter-interpretation)
* [LIP-13 implementation](https://github.com/lidofinance/scripts/pull/33)
