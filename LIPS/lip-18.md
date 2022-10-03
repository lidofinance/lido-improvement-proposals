---
lip: 18
title: Vault Contract for Self-Cover Purposes
status: Proposed
author: Azat Serikov
created: 2022-09-13
updated: 2022-10-03
discussions-to: https://research.lido.fi/t/lip-18-vault-contract-for-potential-self-cover-purposes/2992
---

# LIP-18: Vault Contract for Potential Self-Cover Purposes

## Simple Summary

The Lido Insurance Fund is planned to be used by the Lido DAO as a transparent store of funds allocated for potential self-cover purposes.


## Motivation

Initially, Lido hedged against slashing penalties via a third party insurance provider. In a [July 2021 vote](https://snapshot.org/#/lido-snapshot.eth/proposal/QmWeMuwkLJ3strPAM58kzLaKzbEPrWTLb1VC93ergrYrbv), the DAO decided to explore an independent approach by marking the funds accrued in the form of protocol fees as potentially usable for self-cover purposes. The enactment of [vote #134](https://vote.lido.fi/vote/134) redirected the flow of protocol fees into the Treasury and fixed the amount allocated for cover. However, there is still no apparent distinction between Insurance and Treasury funds as both are stored under the same contract.

This proposal introduces a dedicated vault contract that will serve as a transparent store for cover funds which can only be retrieved by the [DAO Agent](https://etherscan.io/address/0x3e40D73EB977Dc6a537aF587D48316feE66E9C8c). Additionally, the vault features functions for recovering non-cover assets that were sent to the contract by mistake.

> NOTE: The proposal does not make any assumptions in regards to any policies, restrictions and regulations that may be applied and only covers technical implementation.


## Mechanics

The Insurance Fund is a simple vault that inherits [OpenZeppelin's `Ownable`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.7.3/contracts/access/Ownable.sol) and allows the owner to transfer ether, ERC20, ERC721, ERC1155 tokens from the contract. The owner, which will the Lido DAO Agent, can transfer ownership to another entity with an exception of [zero address](https://etherscan.io/address/0x0000000000000000000000000000000000000000).

## Specification
We propose the following interface for `InsuranceFund`. The code below assumes the Solidity v0.8.10 syntax.

### Constructor
```solidity
constructor(address _initalOwner);
```
Assigns `_initalOwner` as the `owner` of the contract (as inherited from [OpenZeppelin's `Ownable`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.7.3/contracts/access/Ownable.sol)), the sole entity authorized to operate the vault.
 - reverts if `_initalOwner` is zero address;
 - emits `event OwnershipTransferred(address indexed previousOwner, address indexed newOwner)`.

#### Parameters

| Name | Type | Description |
| -------- | -------- | -------- |
| `_initalOwner`     | `address`     | entity which will have access to all state-mutating operations     |


### Function: `owner`
```solidity
function owner() public view returns (address);
```
Returns the current `owner`.


### Function: `renounceOwnership`
```solidity
function renounceOwnership() public pure override;
```
Reverts always.


### Function: `transferOwnership`
```solidity
function transferOwnership(address newOwner) public;
```
Assigns `newOwner` as the `owner`.

- reverts if `msg.sender` is not `owner`;
- reverts if `newOwner` is zero address;
- emits `emit OwnershipTransferred(address indexed previousOwner, address indexed newOwner)`.

#### Parameters

| Name       | Type      | Description |
| --------   | --------  | -------- |
| `newOwner` | `address` | entity which will have access to all state-mutating operations |


### Function: `transferEther`
```solidity
function transferEther(address _recipient, uint256 _amount) external;
```
Transfers ether to an entity from the contract balance.
- reverts if `msg.sender` is not `owner`;
- reverts if `_recipient` is zero address;
- reverts if the contract balance is insufficient;
- reverts if the actual transfer OP fails (e.g. `_recipient` is a contract with no fallback);
- emits `EtherTransferred(address indexed _recipient, uint256 _amount)`.

#### Parameters

| Name         | Type      | Description      |
| ------------ | --------- | ---------------- |
| `_recipient` | `address` | recipient entity |
| `_amount`    | `uint256` | transfer amount |

### Function: `transferERC20`
```solidity
function transferERC20(address _token, address _recipient, uint256 _amount) external;
```
Transfer an ERC20 token to an entity in the specified amount from the contract balance.
- reverts if `msg.sender` is not `owner`;
- reverts if `_recipient` is zero address;
- reverts if the contract balance is insufficient;
- emits `ERC20Transferred(address indexed _token, address indexed _recipient, uint256 _amount)`.

#### Parameters

| Name         | Type      | Description      |
| ------------ | --------- | ---------------- |
| `_token`     | `address` | an ERC20 token   |
| `_recipient` | `address` | recipient entity |
| `_amount`    | `uint256` | transfer amount  |

### Function: `transferERC721`
```solidity
function transferERC721(address _token, address _recipient, uint256 _tokenId, bytes memory _data) external;
```
Transfer a single ERC721 token with the specified id to an entity from the contract balance. A contract recipient must implement `ERC721TokenReceiver` in accordance to [EIP-721](https://eips.ethereum.org/EIPS/eip-721) in order to safely receive tokens.
- reverts if `msg.sender` is not `owner`;
- reverts if `_recipient` is zero address;
- emits `ERC721Transferred(address indexed _token, address indexed _recipient, uint256 _tokenId, bytes _data)`.

#### Parameters

| Name         | Type      | Description      |
| ------------ | --------- | ---------------- |
| `_token`     | `address` | an ERC721 token  |
| `_recipient` | `address` | recipient entity |
| `_tokenId`   | `uint256` | token identifier |
| `_data`      | `bytes`   | byte sequence for `onERC721Received` hook  |

### Function: `transferERC1155`
```solidity
function transferERC1155(address _token, address _recipient, uint256 _tokenId, uint256 _amount, bytes calldata _data) external;
```
Transfer a single ERC1155 token with the specified id in the specified amount to an entity from the contract balance. A contract recipient must implement `ERC1155TokenReceiver` in accordance to [EIP-1155](https://eips.ethereum.org/EIPS/eip-1155) in order to safely receive tokens. 
- reverts if `msg.sender` is not `owner`;
- reverts if `_recipient` is zero address;
- reverts if the contract balance is insufficient;
- emits `ERC721Transferred(address indexed _token, address indexed _recipient, uint256 _tokenId, bytes _data)`.

#### Parameters

| Name         | Type      | Description      |
| ------------ | --------- | ---------------- |
| `_token`     | `address` | an ERC1155 token  |
| `_recipient` | `address` | recipient entity |
| `_tokenId`   | `uint256` | token identifier |
| `_amount`    | `uint256` | transfer amount  |
| `_data`      | `bytes`   | byte sequence for `onERC1155Received` hook  |

> Note! `transferERC1155` does not support multi-token batch transfers.

## Security Considerations
### Upgradability
The proposed contract is non-upgradable.
### Ownership
The ownership can be transferred but not renounced. The initial owner set upon construction is intended to be the Lido DAO Agent.

### Links
- [Lido DAO Agent](https://etherscan.io/address/0x3e40D73EB977Dc6a537aF587D48316feE66E9C8c)
- [LIP 6: In-protocol coverage application mechanism proposal
](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-6.md)
- [Redirecting incoming revenue stream from insurance fund to DAO treasury](https://research.lido.fi/t/redirecting-incoming-revenue-stream-from-insurance-fund-to-dao-treasury/2528)