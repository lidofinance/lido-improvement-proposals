---
lip: 17
title: MEV-Boost allowed relays list for Lido
status: Proposed
author: George Avsetsyn, Artem Veremeenko, Eugene Mamin, Isidoros Passadis
discussions-to: https://research.lido.fi/t/lip-17-mev-boost-relays-allowed-list-for-lido/2885
created: 2022-08-30
updated: 2022-09-13
---

# MEV-Boost relays allowed list for Lido

>DISCLAIMER. [MEV extraction policy](https://research.lido.fi/t/discussion-draft-for-lido-on-ethereum-block-proposer-rewards-policy/2817
) for Lido DAO is an ongoing topic and has not been finalized yet. This proposal is about additional tech spares.

## Simple Summary

The on-chain allowed relays list is planned to be used by Node Operators participating in the Lido protocol after the Merge to extract MEV according to the expected Lido policies.

## Motivation

It's proposed that Node Operators use [`MEV-Boost`](https://github.com/flashbots/mev-boost) infrastructure developed by Flashbots to support MEV extraction through the open market mechanics as a current PBS solution that has a market fit.

The proposed allowed list is intended to be a source of truth for the set of possible relays allowed to be used by Node Operators. In particular, Node Operators would use the contract to keep their software configuration up-to-date (setting the necessary relays once Lido DAO updates the set).

## Mechanics

The contract represents simple registry storage.

Anyone can access the storage in a permissionless way through the disclosed `view` methods or by collecting the emitted events covering all storage modifications.

Any modification of the set (i.e., internal storage modification) is allowed only by [the general Lido DAO governance process](https://lido.fi/governance#regular-process), which is implemented on-chain through the Aragon voting from the initial deployment stage. However, it's still possible to assign a dedicated management entity for adding/removing relay items.

The proposed `MEVBoostAllowedRelaysList` contract is non-upgradable, and its owner is intended to be initialized with the [Lido DAO Aragon Agent](https://etherscan.io/address/0x3e40D73EB977Dc6a537aF587D48316feE66E9C8c) address on mainnet upon the deployment phase.

The dedicated management `manager` entity is initialized with a zero address and can be set later by the contract's owner to be able to add/remove relays. The possible candidate to bear such responsibilities without sacrificing governance security is [Easy Track](https://easytrack.lido.fi/) (in this case  [`EVMScriptExecutor (Easy Track)`](https://docs.lido.fi/deployed-contracts#easy-track) should be set as `manager`).

### Relay information structure

It's proposed to use the following structure:

```vyper
struct Relay:
    uri: String[MAX_STRING_LENGTH]
    operator: String[MAX_STRING_LENGTH]
    is_mandatory: bool
    description: String[MAX_STRING_LENGTH]
```

where:

- `uri` is the relay's URI to fetch the data
- `operator` is the name of the entity running the relay
- `is_mandatory` is supposed to distinguish between optional and mandatory relays
- `description` is designated to store any additional info (unspecified at the current stage)

### Adding relay

To add a new relay, `owner` or `manager` (if assigned) should call `add_relay` method, passing all necessary relay structure's params described above.

### Removing relay

To remove a relay, `owner` or `manager` (if assigned) should call `remove_relay` method, passing the relay's `uri`.

### Updating relay

The proposed flow to update the relay's info is to re-add the relay (call `remove_relay` and then `add_relay`).

>NB: the order of the relays in the list after re-adding might change (due to the array-based implementation).

### Reading the current allowed relays

Anyone is allowed to call the following methods to retrieve the current relays set:

- `get_relays_amount` to check how many relays are currently allowed by Lido DAO
- `get_relays` to retrieve all currently allowed relays
- `get_relay_by_uri` to retrieve the allowed relay details by uri
- `get_allowed_list_version` to read lastly bumped version number of the allowed list

### Allowed list versioning

The principal purpose of the proposed contract is to be used for generating configurations containing up-to-date allowed relays.

To facilitate config generation process, the contract contains the previously mentioned `get_allowed_list_version` view method to check whether the lastly used allowed relays list was updated or not.

The version bumps on every add/remove operation and doesn't have additional semantical meaning and numbering schemes.

### Events

All storage modification functions emit at least a single event containing all necessary data to reproduce the changes by external indexers:

- `RelayAdded` (once a relay was allowed)
- `RelayRemoved` (once a previously allowed relay was removed)
- `AllowedListUpdated` (once allowed list version bumped)
- `OwnerChanged` (once the owner is changed)
- `ManagerChanged` (once management entity is assigned or dismissed)
- `ERC20Recovered` (once some ERC-20 tokens successfully recovered)

### Permissions

The contract has a mutable owner set upon the deployment phase. It's presumed that the owner will be set to the [`Lido DAO Aragon Agent`](https://etherscan.io/address/0x3e40D73EB977Dc6a537aF587D48316feE66E9C8c) address. The owner can be changed by a call of `change_owner` only by the current owner.

An additional `manager` is allowed to add or remove relays (initially is set to zero address). Only the contract's owner is allowed to assign or dismiss `manager`.

## Specification

We propose the following interface for `MEVBoostAllowedRelaysList`.
The code below presumes the Vyper v0.3.6 syntax.

### Constructor

```vyper
@external
def __init__(owner: address)
```

Stores internally `owner` as the contract's instance owner.
Initializes `manager` with a zero address (no manager is assigned).

- Reverts if `owner` is zero address.

### Function: change_owner

```vyper
@external
def change_owner(owner: address)
```

Change current `owner` to the new one.

- Reverts if called by anyone except the current owner.
- Reverts if `owner` is the current owner.
- Reverts if `owner` is zero address.
- Emits `OwnerChanged(owner)`.

### Function: set_manager

```vyper
@external
def set_manager(manager: address)
```

Sets `manager` as the current management entity.

- Reverts if called by anyone except the owner.
- Reverts if `manager` is equal to the previously set value.
- Reverts if `manager` is zero address.
- Emits `ManagerChanged(manager)`.

### Function: dismiss_manager

```vyper
@external
def dismiss_manager()
```

Dismisses the current management entity.

- Reverts if called by anyone except the owner.
- Reverts if no `manager` was set previously.
- Emits `ManagerChanged(empty(address))`.

### Function: get_owner

```vyper
@view
@external
def get_owner() -> address
```

Retrieves the current contract owner.

### Function: get_manager

```vyper
@view
@external
def get_manager() -> address
```

Retrieves the current manager entity (returns zero address if no entity is assigned).

### Function: get_relays_amount

```vyper
@view
@external
def get_relays_amount() -> uint256
```

Retrieves the current total amount of allowed relays.

### Function: get_relays

```vyper
@view
@external
def get_relays() -> DynArray[Relay, MAX_NUM_RELAYS]
```

Retrieves all of the currently allowed relays.

### Function: get_relay_by_uri

```vyper
@view
@external
def get_relay_by_uri(relay_uri: String[MAX_STRING_LENGTH]) -> bool
```

Retrieves the relay with the provided uri.

- Reverts if found no relay.

### Function: get_allowed_list_version

```vyper
@view
@external
def get_allowed_list_version() -> uint256:
```

Retrieves the current version of the allowed relays set.

### Function: add_relay

```vyper
@external
def add_relay(
    uri: String[MAX_STRING_LENGTH],
    operator: String[MAX_STRING_LENGTH],
    is_mandatory: bool,
    description: String[MAX_STRING_LENGTH]
):
```

Append relay to the allowed set where params correspond to the previously described [Relay structure](#Relay-information-structure).
Bumps the allowed list version.

- Reverts if called by anyone except the owner or manager (if assigned).
- Reverts if relay with provided `uri` already allowed.
- Reverts if `uri` is empty.
- Emits `RelayAdded(uri, relay)`.
- Emits `AllowedListUpdated(new_allowed_list_ver)`.

### Function: remove_relay

```vyper
@external
def remove_relay(uri: String[MAX_STRING_LENGTH]):
```

Remove the previously allowed relay from the set.
Bumps the allowed list version.

- Reverts if called by anyone except the owner or manager (if assigned).
- Reverts if relay with provided `uri` is not allowed.
- Reverts if `uri` is empty.
- Emits `RelayRemoved(uri, uri)`.
- Emits `AllowedListUpdated(new_allowed_list_ver)`.

### Function: recover_erc20

```vyper
@external
def recover_erc20(token: address, amount: uint256, recipient: address)
```

Transfers ERC20 tokens from the contract's balance to the `recipient`.

- Reverts if called by anyone except the owner.
- Reverts if `transfer` reverted.
- Reverts if `recipient` is zero address.
- Emits `ERC20Recovered(token, amount, recipient)`.
- Emits `Transfer(self.address, recipient, amount)` of the `token`'s contract (if ERC20-compliant).

### Event: RelayAdded

```vyper
event RelayAdded:
    uri_hash: indexed(String[MAX_STRING_LENGTH])
    relay: Relay
```

Emitted when a new relay was added.

See: `add_relay`.

### Event: RelayRemoved

```vyper
event RelayRemoved:
    uri_hash: indexed(String[MAX_STRING_LENGTH])
    uri: String[MAX_STRING_LENGTH]
```

Emitted when a relay was removed.

See: `remove_relay`.

### Event: AllowedListUpdated

```vyper
event AllowedListUpdated:
    allowed_list_version: indexed(uint256)
```

Emitted when either relay was added or removed.

See: `add_relay`, `remove_relay`.

### Event: OwnerChanged

```vyper
event OwnerChanged:
    new_owner: indexed(address)
```

Emitted when new owner was set.

See: `change_owner`.

### Event: ManagerChanged

```vyper
event ManagerChanged:
    new_manager: indexed(address)
```

Emitted when either new manager set or dismissed.

See: `set_manager`, `dismiss_manager`.

### Event: ERC20Recovered

```vyper
event ERC20Recovered:
    token: indexed(address)
    amount: uint256
    recipient: indexed(address)
```

Emitted when ERC20 tokens were recovered.

See: `recover_erc20`.

## Security Considerations

### Upgradability and mutability

The proposed contract is non-upgradable. There is a mutable owner address that is set upon construction and can be changed by the current contract owner.

There is an additional entity allowed to add/remove relays. The entity is easily assignable/revokable only by the contract's owner.

In case of an emergency or important update, it would be necessary to dismiss the manager (if assigned) and re-deploy a new contract.

### Storage modification is restricted

The contract's owner is eligible for any possible storage modification.

The additional manager is supposed to be used as a management entity able to add/remove relays.

### Sanity caps and limits

There are the following caps to prevent unlimited storage accesses and unbounded loop cases:

- `MAX_STRING_LENGTH` to be used for all `String[MAX_STRING_LENGTH]` types, proposed to set its value to `1024`.
- `MAX_NUM_RELAYS` to be used as maximum allowed relays amount, proposed to set its value to `40`.

## Reference implementation

The reference implementation of the proposed `MEVBoostAllowedRelaysList` contract is available on the [Lido GitHub](https://github.com/lidofinance/mev-boost-relay-allowed-list/blob/26ec6791c2466e784a894b8867db71d8de620745/contracts/MEVBoostRelayAllowedList.vy).

## Links

- [Ongoing discussion for block proposer rewards](https://research.lido.fi/t/discussion-draft-for-lido-on-ethereum-block-proposer-rewards-policy/2817)
- [Ethereum MEV Extraction and Rewards - Discussion & Policy Groundwork](https://research.lido.fi/t/ethereum-mev-extraction-and-rewards-discussion-policy-groundwork/2461)
- [[Proposal] optimal MEV policy for Lido](https://research.lido.fi/t/proposal-optimal-mev-policy-for-lido/2489)
- [LIP-12: On-chain part of the rewards distribution after the Merge](https://research.lido.fi/t/lip-12-on-chain-part-of-the-rewards-distribution-after-the-merge/1625)
