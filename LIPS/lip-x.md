---
lip: <to be assigned>
title: "KDF Release: HashConsensus Hardening, Timestamp-Based Frames, Key Delegation Framework, and Protocol-Wide KDF Adoption"
status: WIP
author: Raman Siamionau
discussions-to: <Create a new thread on https://research.lido.fi/ and drop the link here>
created: 2026-06-01
---

## Simple Summary

This proposal bundles four related improvements to Lido's infrastructure into a single release:

1. **Key Delegation Framework (KDF)** — introduces a general-purpose on-chain delegation layer that sits in front of any permissioned key in the protocol, allowing its operator to rotate the hot signing key instantly, without governance voting.
2. **KDF adoption within Oracle and DSM scope** — specifies the concrete off-chain and on-chain changes required to adopt KDF across Lido Oracles and Council daemons.
3. **HashConsensus vulnerability fix** — closes a denial-of-service vector in the `HashConsensus` contract by bounding the number of stored report variants per frame.
4. **Timestamp-based oracle frames** — replaces slot/epoch-based frame logic in `HashConsensus` with wall-clock seconds, so oracle frame lengths stay stable across future Ethereum hardforks (including the EIP-7782 slot time reduction).

## Abstract

This upgrade pursues two main goals: to improve the security of the protocol, and to abstract existing functionality away from slots and epochs so that future upgrades are simpler to perform.

On the security side, we propose to standardize the Key Delegation Framework (KDF) as a protocol-wide primitive and to adopt it for every key that holds any protocol permission. KDF inserts a factory-deployed, per-entity delegation contract between the protocol and each operator's hot key: governance grants permissions to the delegation contract, while the operator's cold key or multisig controls which hot key is active and can rotate it at any time without a governance vote, or irreversibly lock the contract should the cold key itself be compromised. As the first batch of adopters, we propose to begin using KDF immediately for Oracle and Council operators, so that the response to a key compromise can be measured in minutes rather than the ~10 days a governance vote requires today.

On the abstraction side, we propose to rewrite the `HashConsensus` contract ([`lidofinance/core`](https://github.com/lidofinance/core)) to express all frame logic in wall-clock seconds (`secondsPerFrame`, `fastLaneSeconds`, `initialTimestamp`, `refTimestamp`) instead of epoch- and slot-based parameters. This decouples oracle frame durations from Ethereum's consensus-layer slot timing and future-proofs oracle operation against changes such as the [EIP-7782](https://eips.ethereum.org/EIPS/eip-7782) slot-time reduction. The same redeployment also closes a denial-of-service vector in `HashConsensus`: the report-variant storage model is reworked so that, instead of accumulating all unique hashes submitted in a frame, the contract stores only the most recent hash per oracle member per frame via a `frameId → memberIndex → Report` mapping, eliminating the unbounded iteration that enables the attack.

Concretely, we propose to deploy an updated `HashConsensus` contract, a new `DelegationFactory` / `DelegationContract` system (repository: [`lidofinance/delegation-execution-authority`](https://github.com/lidofinance/delegation-execution-authority)), and targeted updates to the [`lido-oracle`](https://github.com/lidofinance/lido-oracle) daemon and `DepositSecurityModule`. The adoption scope specifies the integration points: the `lido-oracle` daemon gains delegation support via a `DELEGATION_CONTRACT_ADDRESS` environment variable, and the `DepositSecurityModule` is updated to validate guardian signatures via [EIP-1271](https://eips.ethereum.org/EIPS/eip-1271), enabling council members to operate behind delegation contracts. Adoption is mandatory for these roles; a coordinated migration process is defined for existing Oracle and DSM operators to onboard to the new model without service interruption.

## Motivation

### Hot-key operational risk for Oracle and Council operators

Oracle members and Deposit Security Module Committee council members currently operate with hot EOA private keys stored directly in off-chain bots. These keys:

- Carry meaningful protocol permissions (report submission, pause/unvet signing, deposit message signing).
- Are frequently shared across personnel and systems.
- May be long-lived with an unclear custody history.
- Require a full on-chain governance vote (~10 days) to rotate in the event of compromise.

The 10-day exposure window between a key compromise and its on-chain revocation is an unacceptable operational risk, and the threat is not hypothetical: the May 2025 compromise of a Chorus One oracle key triggered an emergency DAO vote to rotate the affected oracle. While that incident did not exploit a specific vulnerability, it confirmed that adversarial oracle key control is a realistic scenario — and that the only remediation available today, a full governance vote, is far too slow for an active compromise.

### HashConsensus DoS vulnerability

The current `HashConsensus` contract stores every unique report hash submitted by any oracle member in a per-frame array (`_reportVariants` / `_reportVariantsLength`). Consensus is determined by iterating the full array. Because a single oracle member can submit an unlimited number of distinct hashes, a malicious or buggy member can inflate this array, making each subsequent `submitReport` and `_isQuorumReached` call more expensive — the iteration cost grows with every additional report variant stored.

The attack works by submitting so many alternative report variants that iterating over them no longer fits within a single transaction's gas limit. At that point report submission can no longer be completed in one transaction and quorum can no longer be reached. The practical consequence is a halt of oracle reporting: for as long as the attack is sustained, `HashConsensus` cannot reach consensus and the protocol cannot process the accounting reports that drive stETH rebases, withdrawal request finalization, and validator exit requests. The DoS therefore degrades core protocol operation, not merely the `HashConsensus` contract in isolation.

### Ethereum slot-time changes (EIP-7782, EIP-8198)

Ethereum's consensus-layer slot timing is in flux. Competing proposals such as [EIP-7782](https://eips.ethereum.org/EIPS/eip-7782) (12 s → 6 s) and [EIP-8198](https://eips.ethereum.org/EIPS/eip-8198) ("Quick Slots", 12 s → 8 s) would reduce slot duration and, consequently, halve or otherwise shorten the duration of all oracle frames currently configured in epochs. Under EIP-7782, for example, oracle reporting periods would shrink from roughly 28 days to 14 days, disrupting downstream protocol assumptions.

EIP-8198 goes further by making slot duration a runtime-configurable parameter rather than a hardcoded constant, with the value expected to be tuned iteratively across devnets. Any frame logic anchored to `slotsPerEpoch × secondsPerSlot` is therefore not just fragile across a single hardfork but perpetually exposed to re-tuning. Migrating frame configuration from `epochsPerFrame` to `secondsPerFrame` removes the slot dependency entirely, making frame duration invariant to any present or future slot-timing change.

## Specification

### Overview

The release consists of four coordinated changes:

1. Deployment of the `DelegationFactory`, establishing the general-purpose on-chain delegation infrastructure (KDF).
2. An update to the `DepositSecurityModule` contract to verify guardian signatures via EIP-1271, so that Council guardians can operate behind KDF delegation contracts.
3. Modifications to the Oracle and Council daemons to support operating behind delegation contracts.
4. Modifications to the `HashConsensus` contract and related contracts — in both `lidofinance/core` and CSM's separately-deployed fork — to fix the DoS vulnerability and switch to seconds-based frames.

### Rationale

**Why the changes ship as one release.** The proposal groups two otherwise independent tracks — the KDF security work and the `HashConsensus` rework — into a single release to amortize the audit, deployment, and operator-coordination effort.

**Why KDF ships with its adoption scope.** The `DelegationFactory` and `DelegationContract` are inert on their own: without the Oracle and DSM integration they deliver no security benefit and give operators no migration path. Shipping the contracts together with their adoption scope is what actually closes the hot-key risk in this release rather than deferring it.

**Why a purpose-built own delegation contract rather than reusing an existing solution.** A general-purpose delegation or smart-account framework is far more flexible than this role needs, and that unused flexibility is attack and misconfiguration surface. A minimal, non-upgradeable contract instead makes KDF's guarantees structural — admin can never `execute()` or sign, one active delegatee, instant assignment and revocation, irreversible `pause()` — keeping the audited surface small.

### Technical Specification

---

#### Part 1: Key Delegation Framework (KDF) — Contracts

##### Architecture

The delegation layer is composed of two contracts deployed from a single factory:

- **`DelegationFactory`** — a singleton that deploys `DelegationContract` instances. Each permissioned entity (oracle operator, council member) deploys exactly one `DelegationContract` per bot. A single delegation contract **must not** be shared across multiple bots or protocol roles; each independently operating bot requires its own instance.
- **`DelegationContract`** — a non-upgradeable, minimal on-chain delegation proxy. It enforces a one-admin / one-active-delegatee model and supports two integration patterns described below.

##### Roles

The contract has exactly two roles, **admin** and **delegatee**:

| Role                    | Custody                                                    | Capabilities                                                                                                                |
|-------------------------|------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| **Admin**               | Safe multisig or other audited multisig with justification | Assign, reassign, and revoke the delegatee (`assignDelegatee()` / `revokeDelegatee()`); irreversibly `pause()` the contract |
| **Delegatee (Hot Key)** | Hot key in the off-chain daemon                            | Call `execute()` to dispatch transactions (`push`); sign messages verifiable via EIP-1271 (`pull`)                          |

The admin manages the delegatee's lifecycle but can never act *as* the contract. Pausing is the escape hatch for admin-key compromise: because `pause()` is irreversible and permanently disables `execute()` and `isValidSignature()`, a still-trusted admin can neutralise the contract entirely — stripping the delegatee of any usable role under its authority — even when the only safe move is to take the seat offline rather than rotate it.

##### Contract Interface

```solidity
interface IDelegationContract {
    // --- Admin controls ---

    /// @notice Assign (or reassign) the active delegatee.
    ///         Only callable by admin. Takes effect immediately: the new
    ///         delegatee replaces any current one atomically, and the old
    ///         key can no longer call execute() or produce valid signatures.
    /// @param delegatee Address of the incoming delegatee.
    function assignDelegatee(address delegatee) external;

    /// @notice Instantly remove the current delegatee. Only callable by admin.
    ///         Takes effect immediately.
    function revokeDelegatee() external;

    /// @notice Pause the contract, permanently disabling execute() and
    ///         isValidSignature(). Only callable by admin.
    ///         Intended for emergency use when the admin suspects compromise
    ///         of the delegatee or of the contract itself.
    ///         Pausing is irreversible; a new DelegationContract must be
    ///         deployed and governance must reassign the protocol seat.
    function pause() external;

    // --- Push integration ---

    /// @notice Execute an arbitrary call on behalf of this contract.
    ///         Only callable by the current delegatee.
    ///         Reverts if the contract is paused.
    ///         Reverts if the target call reverts, propagating the target's
    ///         revert data — the whole transaction is atomic, so forwarded
    ///         ETH is returned to the delegatee on failure.
    ///         Forwards msg.value to the target to support payable targets.
    ///         After the target call returns, any residual ETH balance on
    ///         this contract is swept back to the delegatee (msg.sender).
    /// @param target  Address to call.
    /// @param data    Call data.
    /// @return result Return data from the call.
    function execute(address target, bytes calldata data)
        external
        payable
        returns (bytes memory result);

    // --- Pull integration ---

    /// @notice EIP-1271 signature validation.
    ///         Returns the magic value iff `signature` was produced by
    ///         the currently assigned delegatee over `hash`.
    ///         Returns a non-magic value (does not revert) if the contract
    ///         is paused or no delegatee is assigned.
    function isValidSignature(bytes32 hash, bytes calldata signature)
        external
        view
        returns (bytes4 magicValue);

    // --- Views ---

    /// @notice Returns the current admin address.
    function getAdmin() external view returns (address);

    /// @notice Returns the current active delegatee, or address(0) if none.
    function getDelegatee() external view returns (address);

    /// @notice Returns true if the contract has been paused.
    function isPaused() external view returns (bool);

    // --- Events ---

    event DelegateeAssigned(address indexed newDelegatee);
    event DelegateeRevoked(address indexed revokedDelegatee);
    event Paused();
}

interface IDelegationFactory {
    /// @notice Deploy a new DelegationContract.
    /// @param admin     Address to set as the contract's admin. Fixed for the
    ///                  lifetime of the contract — there is no on-chain way to
    ///                  change it; replacing the admin requires deploying a new
    ///                  contract and a governance vote to reassign the seat.
    /// @return instance Address of the newly deployed DelegationContract.
    function deploy(address admin)
        external
        returns (address instance);
}
```

##### Integration Models

The `DelegationContract` is designed as a **general integration primitive** for any permissioned bot in the Lido ecosystem. Two complementary patterns are supported; protocol integrations should select the one that fits their verification model.

###### Pull Style — Signature Verification via EIP-1271

The consumer contract calls `isValidSignature(hash, sig)` on the `DelegationContract` when it needs to verify that a message was authorized by the registered entity.

```
Off-chain bot (delegatee)          DelegationContract         Protocol contract
        │                                  │                          │
        │  sign message (ECDSA, hot key)   │                          │
        │                                  │                          │
        │─ signature relayed off-chain ──────────────────────────────►│
        │                                  │                          │
        │                                  │◄──── isValidSignature() ─│
        │                                  │      (hash, sig)         │
        │                                  │                          │
        │                                  │  recover signer,         │
        │                                  │  check == getDelegatee() │
        │                                  │                          │
        │                                  │─► returns magic value ──►│
```

**Security assumptions:**
- The protocol contract must treat the `DelegationContract` address — not the hot key — as the authorized principal. It must never trust the raw signer address extracted from the signature.
- The delegatee key used for signing must correspond to the key currently stored in `getDelegatee()` at the time the signature is verified on-chain. A key rotated after signing but before verification will cause rejection.

**When to use:** Any integration where the protocol collects signatures off-chain and verifies them on-chain in a single transaction. Examples: DSM guardian pause messages, deposit attestations, unvet signatures.

###### Push Style — Transaction Execution via `execute()`

The delegatee calls `execute(target, data)` (with optional ETH value) on the `DelegationContract`, which forwards the call to the target protocol contract. From the target's perspective, `msg.sender` is the `DelegationContract` address.

```
Off-chain bot (delegatee)          DelegationContract         Protocol contract
        │                                  │                          │
        │── execute(target, data) ────────►│                          │
        │   [optional msg.value]           │                          │
        │                                  │── call(target, data) ───►│
        │                                  │   msg.sender ==          │
        │                                  │   DelegationContract     │
        │                                  │◄── result ───────────────│
        │◄── result (or revert) ───────────│                          │
```

`execute()` is `payable` and forwards `msg.value` to the target. This is required to support future permissioned bots that interact with payable protocol functions, such as EIP-7002 triggerable withdrawal requests and EIP-7251 consolidation requests, where a fee must be paid in ETH at call time. After the target call returns, `execute()` sweeps any ETH still held by the contract back to the delegatee, so change refunded by the target (e.g. the excess fee the EIP-7002 / EIP-7251 predeploys return to their caller) is forwarded to the bot rather than stranded on the delegation contract.

**Security assumptions:**
- The target contract must verify `msg.sender == DelegationContract` is a registered permission holder (e.g., oracle committee member, DSM guardian). It must not rely on `tx.origin`.
- The delegatee can call any target with any data; the contract does not restrict the call target. Governance must ensure the `DelegationContract` address holds only the narrowest set of permissions needed.
- If the target call reverts, `execute()` reverts atomically and forwarded ETH is returned to the delegatee. On a *successful* call, any ETH the target refunds to the contract (change) is swept back to the delegatee before `execute()` returns; only ETH the target actually consumes/keeps is non-recoverable. Bots should still send at least the required fee, but overpayment that the target refunds is no longer lost.

**When to use:** Any integration where the bot submits transactions that modify on-chain state and the protocol contract checks `msg.sender` for authorization. Examples: oracle report submission (`submitReportData`) or any permissioned bots.

##### Admin Immutability

The admin address is fixed at deployment and **cannot be changed on-chain** — the contract exposes no admin-transfer function. This is a deliberate security choice: a `changeAdmin`-style call would be the single most damaging action available to a compromised admin, letting an attacker lock the legitimate owners out permanently. Removing it bounds the worst case — even a compromised admin cannot seize ownership of the contract, only act within the delegatee lifecycle, which the legitimate owners and governance can still contain.

Rotating the admin (e.g., moving the seat to a different multisig, or re-keying outside an in-place Safe upgrade) therefore requires deploying a fresh `DelegationContract` and a DAO governance vote to reassign the protocol seat to it. This also prevents silent transfer of committee participation between organizations and avoids legal ambiguity over who is accountable for the seat.

##### Delegatee Assignment and Revocation

Both assignment and revocation take effect immediately, with no scheduling or waiting period:

- **Assignment** (`assignDelegatee(newHotKey)`) replaces the active delegatee atomically. The previous key can no longer call `execute()` or produce valid EIP-1271 signatures from the moment the transaction lands.
- **Revocation** (`revokeDelegatee()`) removes the active delegatee, leaving the contract with no delegatee until a new one is assigned.

The expected sequence is: detect delegatee compromise → `revokeDelegatee()` immediately → notify DAO → `assignDelegatee(newHotKey)`.

##### Emergency Pause

The admin may call `pause()` to permanently disable the contract's `execute()` and `isValidSignature()` functions. This is an **irreversible** operation: a paused contract cannot be un-paused. After pausing:

- All delegatee calls to `execute()` revert.
- All EIP-1271 signature verifications return a non-magic value (not a revert), so upstream callers that check the return value rather than reverting on a bad result will handle this gracefully.
- A new `DelegationContract` must be deployed and governance must reassign the protocol seat to the new address.

The pause is intended for scenarios where the admin believes the admin's own address has been compromised and wishes to ensure no further actions can be taken under its authority. The expected sequence is: detect compromise → `pause()` → notify DAO → deploy new delegation contract → governance vote.

##### Hot-Key Rotation

The admin rotates the active delegatee by calling `assignDelegatee(newHotKey)`. The old key is invalidated atomically as the new one becomes active in the same transaction.

The specific cadence for routine rotation is not mandated here; recommended practices will be maintained on the Lido research forum and may be revised without a governance vote. Contract-level requirements are:

- Rotation must be performed promptly upon any suspected or confirmed hot-key compromise. If a replacement key is not immediately ready, `revokeDelegatee()` should be called first to take the compromised key out of service.
- After assignment, the operator verifies the new delegatee via `getDelegatee()` and updates the daemon configuration before restarting.

For monitoring purposes, the `DelegateeAssigned` event is sufficient to track the current delegatee; no additional view method is required beyond `getDelegatee()`.

##### Admin Address Requirements

The admin is the critical security boundary of the delegation model. The principles below apply; the detailed operational requirements — specific hardware-wallet models, minimum multisig quorums, and approved implementations — are maintained in a dedicated "Key Handling Policy" document on the Lido research forum.

**Strongly recommended: Safe (Gnosis Safe) multisig.** Safe is the preferred admin implementation because it is battle-tested at Lido scale and its upgrade path is expected to include post-quantum (PQ) signature support — an assumption this design explicitly relies on. A Safe-based admin should be upgradeable to PQ-compatible signing without replacing the multisig address or requiring a governance vote for the delegation contract.

##### Permission Model

Governance grants protocol permissions (e.g., oracle committee membership, DSM guardian status) to the **`DelegationContract` address** rather than to the hot EOA directly. The protocol's permission registry is unchanged; only the address type of the permission holder changes.

##### Repository and Implementation

- **Smart contracts**: `lidofinance/delegation-execution-authority` (Solidity + Foundry framework)
  - `DelegationFactory` and `DelegationContract`
  - EIP-1271 signature validation; payable `execute()` with value forwarding; immediate `assignDelegatee()` / `revokeDelegatee()`; irreversible `pause()`; admin-cannot-execute invariant enforced by design
  - Full test suite including mainnet fork tests (Foundry, covering contract logic, factory deployment, access control, EIP-1271, payable execution, assignment/revocation, and pause behaviour)

---

#### Part 2: KDF Adoption — Oracle and DSM Integration

This section defines which components are migrated to the delegation model in this release, the concrete changes required for each, and the operator migration process. Adoption is mandatory for Oracle and DSM roles.

##### Scope of Migration in This Release

| Component                              | Integration style                                                                                                            |
|----------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Lido Oracle (holds delegatee key)      | Push (`execute()`)                                                                                                           |
| `DepositSecurityModule` contract       | Pull (EIP-1271)                                                                                                              |
| Council daemon (holds delegatee key)   | Pull (signs messages, publishes its `DelegationContract` address with each signature, DSM verifies)                          |
| Depositor bot                          | Claims the council-provided `(signature, DelegationContract address)` pairs; re-sorts the deposit array by guardian address. |
| Validator ejector (node-operator side) | Off-chain only — must resolve the active delegatee via `getDelegatee()`. See Validator Ejector section below                 |

All components in the table are migrated together.

---

###### Oracle Daemon — Push Integration

**Pattern:** Push via `execute()`.

**Authorization model:** `HashConsensus` and the report processors (`AccountingOracle`, `ValidatorsExitBusOracle`, and CSM's `FeeOracle`) check `msg.sender` against the registered oracle committee member set. After migration, `msg.sender` is the `DelegationContract` address, which governance must have registered as the committee member before the daemon switches over.

**Daemon configuration:** `DELEGATION_CONTRACT_ADDRESS` environment variable. When set, all oracle contract calls are routed through `DelegationModule`, a Web3py extension that wraps `execute()` dispatch. Startup validation checks: (a) `getDelegatee() == configuredHotKey`, and (b) `delegationContract` address holds oracle committee membership in `HashConsensus`.

**Transitional behavior:** When `DELEGATION_CONTRACT_ADDRESS` is unset, calls are sent directly from the hot key EOA as before. This path exists for development purposes.

---

###### DSM Contract — Pull Integration

**Pattern:** Pull via `isValidSignature()`.

**Authorization model today.** `DepositSecurityModule` does **not** take the guardian address as an input — it *derives* it: every verification path (`_verifyAttestSignatures` for deposits, plus `pauseDeposits` and `unvetSigningKeys`) calls `ECDSA.recover(msgHash, sig.r, sig.vs)` and looks the recovered EOA up in the guardian set via `_isGuardian`.

**Required change — the guardian address must be supplied with the signature.** Because a contract guardian cannot be recovered, the submitter provides the registered guardian address (the `DelegationContract`, or the EOA for a legacy guardian) **explicitly alongside each signature**, and DSM verifies the signature *against that address*:

```solidity
// Per-signature payload gains the guardian address it is signed under:
struct GuardianSignature {
    address guardian;   // registered guardian: DelegationContract or legacy EOA
    bytes32 r;
    bytes32 vs;         // EIP-2098 compact signature
}

function _isValidGuardianSignature(
    address guardian,
    bytes32 msgHash,
    bytes32 r,
    bytes32 vs
) internal view returns (bool) {
    if (!_isGuardian(guardian)) return false;
    if (guardian.code.length > 0) {
        // Contract guardian: EIP-1271 over the 64-byte (2098) signature.
        // The DelegationContract recovers the signer and checks == getDelegatee().
        return IERC1271(guardian).isValidSignature(msgHash, abi.encodePacked(r, vs))
            == IERC1271.isValidSignature.selector;
    }
    // Legacy EOA guardian: recovered signer must equal the claimed guardian.
    return ECDSA.recover(msgHash, r, vs) == guardian;
}
```

For the multi-signature deposit path, sorting/dedup moves from recovered signers to the **explicit guardian addresses** — the array must be strictly ascending by `guardian`, which rejects duplicates and preserves the anti-double-count invariant:

```solidity
address prev;
for (uint256 i = 0; i < sortedGuardianSignatures.length; ++i) {
    GuardianSignature calldata gs = sortedGuardianSignatures[i];
    if (gs.guardian <= prev) revert SignaturesNotSorted();
    if (!_isValidGuardianSignature(gs.guardian, msgHash, gs.r, gs.vs)) revert InvalidSignature();
    prev = gs.guardian;
}
```

**Affected DSM entry points:** each `Signature` / `Signature[]` argument is extended to carry the guardian address as above:

```solidity
function depositBufferedEther(
    uint256 blockNumber,
    bytes32 blockHash,
    bytes32 depositRoot,
    uint256 stakingModuleId,
    uint256 nonce,
    bytes calldata depositCalldata,
    GuardianSignature[] calldata sortedGuardianSignatures   // was Signature[]
) external;

function pauseDeposits(
    uint256 blockNumber,
    GuardianSignature calldata sig   // was Signature[]
) external;

function unvetSigningKeys(
    uint256 blockNumber,
    bytes32 blockHash,
    uint256 stakingModuleId,
    uint256 nonce,
    bytes calldata nodeOperatorIds,
    bytes calldata vettedSigningKeysCounts,
    GuardianSignature calldata sig   // was Signature[]
) external;
```

**Transitional behavior:** The existing ECDSA path is retained, so a guardian still registered as an EOA continues to be verified unchanged. This path exists for development purposes.

---

###### Council Daemon — Pull Integration (Signer Side)

**Pattern:** Pull (the daemon signs; DSM verifies on-chain via `isValidSignature()`).

The council daemon produces ECDSA signatures over the standard DSM message using the **delegatee hot key**. The signed digest is unchanged. Because DSM no longer derives the guardian by recovery (see DSM Contract above), **the council daemon's responsibility is to publish, for each message, both the signature and its registered guardian address** — the daemon's `DelegationContract` address, not the delegatee EOA.

On-chain, DSM is given that guardian address explicitly, sees it is a contract, and calls `isValidSignature(msgHash, abi.encodePacked(r, vs))` on it; the `DelegationContract` recovers the signer from the 64-byte signature and checks it against `getDelegatee()`.

**Required change to the council daemon:** Sign with the delegatee private key, set `DELEGATION_CONTRACT_ADDRESS`, and publish that `DelegationContract` address together with each signature (so consumers know which registered guardian the signature is for). No structural code change is required beyond configuration and attaching the guardian address to each published signature.

**Required change to the depositor bot:** The bot relays the council-provided `(DelegationContract address, signature)` pairs to DSM as-is. The only behavioural change is ordering: when assembling `depositBufferedEther`, the bot must sort the signature array by the council-supplied guardian address (matching DSM's new ascending-by-`guardian` ordering) rather than by recovered signer.

---

###### Validator Ejector (Node-Operator Side) — Off-Chain Update

The VEBO module of `lido-oracle` is covered by the push integration above. Separately, the **validator ejector** run by node operators — the daemon that watches `ValidatorsExitBusOracle` exit request events and sends voluntary exit messages to the consensus layer — performs no on-chain transactions and needs no `DelegationContract` of its own. It does, however, need updates to operate correctly once oracles sit behind delegation contracts:

- Any check that validates the origin of an exit request against a configured oracle address must compare against the `DelegationContract` address (the registered committee member), not the delegatee EOA.
- Any component resolving the oracle's signing address (monitoring, dashboards, message verification) must resolve the current signer via `getDelegatee()` on the delegation contract rather than a static configured address.

---

##### Operator Migration Process

Migration is designed to be zero-downtime. All preparatory steps are completed off the critical path, leaving a single irreversible governance vote that flips every prepared seat to delegation at once. The reassignment of a committee seat from a hot EOA to a `DelegationContract` cannot be undone without a further governance vote, so operators are expected to validate the full flow on the Hoodi testnet before the mainnet vote.

1. **Deploy and configure contracts (operators)**: each operator deploys its `DelegationContract` via `DelegationFactory.deploy(adminAddress)` — the admin being a Safe multisig, fixed for the contract's lifetime (see Admin Immutability in Part 1) — and assigns the active delegatee (its existing hot key) via `assignDelegatee()`.
2. **Publish and verify addresses (operators)**: each operator publishes its `DelegationContract` and admin addresses on the Lido research forum, so the DAO and other operators can verify them ahead of the vote.
3. **Prepare daemons for rotation (operators)**: each operator stages the delegatee key and the `DELEGATION_CONTRACT_ADDRESS` configuration in its Oracle and Council daemons, so they are ready to operate via the delegation contract the moment the seat is reassigned. No key material changes, and the daemons keep operating from the hot EOA until the vote lands.
4. **Set up monitoring (Lido team)**: the Lido team registers every delegation contract in the monitoring infrastructure and begins watching the admin and delegatee addresses for unexpected activity, so anomalies are caught both before and after the seat reassignment.
5. **Governance vote**: a single DAO vote reassigns each Oracle committee and DSM guardian seat from the operator's hot EOA to its `DelegationContract` address — a `HashConsensus` member update for oracles, a `DepositSecurityModule` guardian replacement for guardians; the pre-configured daemons begin routing through the delegation contract without interruption.

Until the governance vote in step 5 executes, operators continue to run as EOA principals; the EOA paths described above keep them online throughout preparation. After the vote, each operator confirms the switchover via `getDelegatee()` and a successful report or guardian message in the following frame, and may rotate the hot key at any time via `assignDelegatee()` with no further governance vote. Adoption is not optional as an end state — every Oracle and DSM operator is required to complete the migration.

---

#### Part 3: HashConsensus — Bounded Report Variant Storage

##### Storage Change

The unbounded `_reportVariants` array is replaced with a two-dimensional mapping keyed by frame ID and member index:

```solidity
// Removed:
mapping(uint256 => ReportVariant) internal _reportVariants;
uint256 internal _reportVariantsLength;

// Added:
struct Report {
    bytes32 hash;
    uint64  refTimestamp;
}
mapping(uint256 => mapping(uint256 => Report)) internal _currentMemberHash;
// frameId => memberIndex => Report
```

Each oracle member has at most one stored report per frame. Old entries with a stale `refTimestamp` are ignored in consensus evaluation, eliminating the need to clear storage between frames.

##### `submitReport` Behavior

Calling `submitReport(uint256 timestamp, bytes32 report, uint256 consensusVersion)` overwrites `_currentMemberHash[frameId][memberIndex]` unconditionally. This allows a member to correct a previously submitted hash within the same frame, which is a functional improvement over the current model.

##### Consensus Evaluation (`_isQuorumReached`)

The consensus check iterates the fixed member set (bounded by committee size), fetching each member's single stored hash for the current frame and counting votes per unique hash. This iteration is O(N) in committee size, not O(N) in submitted variants.

##### Secondary Effect — Oracle Members Can Update Their Report

As a consequence of the new model, oracle members can replace an earlier submission within the same frame. This is beneficial (allows correction of off-chain bugs mid-frame) but introduces the theoretical edge case that, if members repeatedly change their reports without converging, quorum formation could be delayed. Monitoring should alert on frames where consensus is not reached within the fast-lane window.

If a member changes its hash after consensus was already reached and quorum no longer holds, `HashConsensus` calls `discardConsensusReport(refTimestamp)` on the report processor to drop the now-invalid report for that frame.

##### Applicability to CSM

CSM's fee oracle (`src/FeeOracle.sol`, deployed as `CSFeeOracle`) is served by a separately-deployed `HashConsensus` instance built from a vendored copy of the same contract, which carries the identical unbounded `_reportVariants` storage and is therefore independently exposed to this DoS. The same bounded-storage fix is applied to CSM's `HashConsensus` in this release (see Affected Contracts in Part 4).

---

#### Part 4: HashConsensus — Timestamp-Based Frames

##### Motivation

EIP-7782 reduces Ethereum's slot time from 12 s to 6 s. All oracle frame lengths are currently expressed as `epochsPerFrame`, which translates to seconds via `epochsPerFrame × slotsPerEpoch × secondsPerSlot`. After the hardfork, the same `epochsPerFrame` value produces half the intended wall-clock duration. A seconds-based `secondsPerFrame` removes this dependency entirely.

The case is reinforced by EIP-8198 ("Quick Slots"), which proposes to make slot duration a runtime-configurable parameter (targeting a 12 s → 8 s reduction) rather than a hardcoded constant. If slot timing becomes a value that can change between hardforks — or be tuned iteratively across devnets — any frame logic anchored to `slotsPerEpoch × secondsPerSlot` becomes perpetually fragile. Expressing frames directly in wall-clock seconds insulates oracle operation from both EIP-7782 and EIP-8198, and from any subsequent slot-timing change.

##### Interface Modifications

###### `IReportAsyncProcessor` (implemented by `BaseOracle`, and therefore by `AccountingOracle` and `ValidatorsExitBusOracle`; CSM runs a separate forked copy — see Affected Contracts below)

```solidity
// Before:
interface IReportAsyncProcessor {
    function submitConsensusReport(
        bytes32 _report,
        uint256 _refSlot,
        uint256 _deadline
    ) external;

    function discardConsensusReport(uint256 _refSlot) external;
}

// After:
interface IReportAsyncProcessor {
    function submitConsensusReport(
        bytes32 _report,
        uint256 _refTimestamp,
        uint256 _deadline
    ) external;

    function discardConsensusReport(uint256 _refTimestamp) external;
}
```

Existing implementations receive a `uint256` regardless; because `refTimestamp` values are numerically larger than any historical `refSlot`, all on-chain value comparisons (e.g., "must be greater than last processed value") remain valid without modification. Any logic that interprets the value semantically as a slot number is updated in this release as part of the protocol-wide ABI migration.

###### View Functions

```solidity
// Before:
function getFrameConfig() external view returns (
    uint256 initialEpoch,
    uint256 epochsPerFrame,
    uint256 fastLaneLengthSlots
);
function getCurrentFrame() external view returns (
    uint256 refSlot,
    uint256 reportProcessingDeadlineSlot
);
function getMembers() external view returns (
    address[] memory addresses,
    uint256[] memory lastReportedRefSlots
);
function getFastLaneMembers() external view returns (
    address[] memory addresses,
    uint256[] memory lastReportedRefSlots
);
function getConsensusState() external view returns (
    uint256 refSlot,
    bytes32 consensusReport,
    bool isReportProcessing
);

// After:
function getFrameConfig() external view returns (
    uint256 initialTimestamp,
    uint256 secondsPerFrame,
    uint256 fastLaneSeconds
);
function getCurrentFrame() external view returns (
    uint256 refTimestamp,
    uint256 reportProcessingDeadlineTimestamp
);
function getMembers() external view returns (
    address[] memory addresses,
    uint256[] memory lastReportedRefTimestamps
);
function getFastLaneMembers() external view returns (
    address[] memory addresses,
    uint256[] memory lastReportedRefTimestamps
);
function getConsensusState() external view returns (
    uint256 refTimestamp,
    bytes32 consensusReport,
    bool isReportProcessing
);
```

###### `MemberConsensusState` Struct

```solidity
struct MemberConsensusState {
    uint256 currentFrameRefTimestamp;      // renamed from currentFrameRefSlot
    bytes32 currentFrameConsensusReport;
    bool    isMember;
    bool    isFastLane;
    bool    canReport;
    uint256 lastMemberReportTimestamp;     // renamed from lastMemberReportRefSlot
    bytes32 currentFrameMemberReport;
}
```

###### Setter / Admin Functions

```solidity
// Before:
function setFrameConfig(uint256 epochsPerFrame, uint256 fastLaneLengthSlots) external;
function submitReport(uint256 slot, bytes32 report, uint256 consensusVersion) external;
function updateInitialEpoch(uint256 initialEpoch) external;
function setFastLaneLengthSlots(uint256 fastLaneLengthSlots) external;

// After:
function setFrameConfig(uint256 secondsPerFrame, uint256 fastLaneSeconds) external;
function submitReport(uint256 timestamp, bytes32 report, uint256 consensusVersion) external;
function updateInitialTimestamp(uint256 initialTimestamp) external;
function setFastLaneSeconds(uint256 fastLaneSeconds) external;
```

###### New Function

```solidity
function getInitialTimestamp() external view returns (uint256);
```

###### Removed Functions

```solidity
// Removed — superseded by getInitialTimestamp():
function getInitialRefSlot() external view returns (uint256);

// Removed — no longer required for frame math under seconds-based frames:
function getChainConfig() external view returns (
    uint256 slotsPerEpoch,
    uint256 secondsPerSlot,
    uint256 genesisTime
);
```

`getInitialRefSlot()` is replaced by `getInitialTimestamp()`, and `getChainConfig()` is dropped because frame math no longer depends on chain config. All in-protocol consumers of these methods are updated to the timestamp-based interface in this release.

**Note — external consumers.** These two functions are public view methods that out-of-protocol integrations (block explorers, dashboards, indexers, analytics, third-party contracts) may read. Removing them is a breaking ABI change beyond Lido's own codebase, so before removal it requires dedicated research to enumerate external dependents and an advance announcement so integrators can move to the timestamp-based equivalents. If that research surfaces hard external dependencies that cannot migrate in time, retaining the two methods as thin read-only compatibility shims should be reconsidered rather than removing them outright.

##### Events and Errors

All events and custom errors that reference `refSlot`, `slot`, `epoch`, `epochsPerFrame`, `fastLaneLengthSlots`, or `initialEpoch` must be updated to their timestamp-based equivalents. Off-chain services that parse these events (oracle daemons, indexers, monitoring tools) will require corresponding updates before or at the time of contract upgrading.

##### Off-Chain Oracle Daemon — Resolving `refTimestamp` to a Reference Slot

On-chain the frame is now anchored to a wall-clock `refTimestamp`, but an oracle report is still built against the consensus-layer state at a concrete **slot**. The daemon must therefore resolve the frame's `refTimestamp` to a reference slot before assembling a report:

1. **Read the frame reference.** Fetch `refTimestamp` from `HashConsensus.getCurrentFrame()`.
2. **Resolve it to a slot.** If a slot boundary falls exactly on `refTimestamp` — `(refTimestamp − genesisTime) % secondsPerSlot == 0` — that slot is the reference slot. If no slot lands exactly on `refTimestamp`, the daemon takes the **latest slot at or before `refTimestamp`** (the previous slot):

   ```
   refSlot = (refTimestamp − genesisTime) / secondsPerSlot   // integer division (floor)
   ```

   The formula above assumes a single, constant slot duration (`genesisTime`/`secondsPerSlot` from the beacon-chain spec). The exact resolution logic depends on which slot-timing change is ultimately adopted and must track it: a one-off `secondsPerSlot` reduction (EIP-7782) keeps this single-constant formula but with a new value, whereas a runtime-configurable or piecewise schedule (EIP-8198) requires walking the slot-duration schedule active at `refTimestamp` rather than dividing by one constant. The invariant that must hold regardless of the adopted EIP is "resolve to the latest slot at or before `refTimestamp`"; only the arithmetic that maps timestamps to slots changes.

3. **Build the entire report on that single slot.**
4. **Reference that slot in the report.** The report still carries the resolved `refSlot` so that every consumer and verifier can confirm the whole report was built against the same slot.

##### Migration Path

The new `HashConsensus` contract is initialized through a finalize-upgrade method that receives the full seconds-based frame configuration (`secondsPerFrame`, `fastLaneSeconds`). Rather than passing `initialTimestamp` in by hand, the method derives it on-chain from the legacy contract: it reads the old `getInitialRefSlot()` and `getChainConfig()` and computes

```
initialTimestamp = genesisTime + initialRefSlot × secondsPerSlot
```

i.e. the wall-clock timestamp of the legacy initial reference slot. Deriving the anchor on-chain from the contract being replaced guarantees the timestamp-based frame boundary lines up exactly with the legacy slot-based boundary, eliminating the risk that an off-chain miscalculation introduces a reporting gap or overlap at the migration point. The `BaseOracle` initialization flow that currently calls `getInitialRefSlot()` will call `getInitialTimestamp()` instead.

##### Affected Contracts and Migration Scope

The slot-to-timestamp migration touches two independent codebases, each with its own deployed `HashConsensus` instance. Because the `refSlot`/`refTimestamp` value is treated as an opaque, monotonically increasing reference by almost every consumer, most call sites need only an interface/name update; the only places requiring real logic changes are the frame math inside `HashConsensus` and the handful of sites that interpret the reference value *semantically as a slot number*.

**`lidofinance/core`:**

| Contract                                             | Change type                                    | Notes                                                                                                                                                                                                                                                                                                                                                                   |
|------------------------------------------------------|------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `HashConsensus.sol`                                  | Upgrade required                               | The full frame engine: `FrameConfig`/`ConsensusFrame`, all epoch/slot frame math, `_submitReport` deadline and fast-lane comparisons, the seconds-based config setters, and the removal of `getChainConfig()` / renaming of `getInitialRefSlot()`. Also carries the bounded-storage DoS fix (Part 3).                                                                   |
| `BaseOracle.sol`                                     | Upgrade required                               | `ConsensusReport.refSlot` → `refTimestamp`; the `IReportAsyncProcessor` implementation and `_getCurrentRefSlot()` are opaque pass-through (rename only). Its two consensus calls change: the `getChainConfig()` consistency check is dropped/repointed, and `getInitialRefSlot()` → `getInitialTimestamp()`.                                                            |
| `AccountingOracle.sol`                               | Upgrade required                               | Interprets the reference as a slot: the `_getSlotTimestamp(slot)` helper computes `GENESIS_TIME + slot * SECONDS_PER_SLOT`, and `timeElapsed = (data.refSlot − prevRefSlot) * SECONDS_PER_SLOT`. With a timestamp reference these conversions must change (the value is already a timestamp / a second-delta), or `onOracleReport` and exit-time math silently corrupt. |
| `ValidatorsExitBusOracle.sol`                        | Rename                                         | `DataProcessingState.refSlot` + an equality check; opaque pass-through.                                                                                                                                                                                                                                                                                                 |
| `RefSlotCache.sol`, `VaultHub.sol`, `LazyOracle.sol` | Rename                                         | Use `getCurrentFrame()`'s first return purely as an opaque `uint48` cache key (no slot arithmetic); functionally unchanged. The cached value simply becomes a timestamp. `ILazyOracle._vaultsDataRefSlot` is a parameter rename.                                                                                                                                        |
| `OracleReportSanityChecker.sol`                      | Upgrade required (for ZK-oracle compatibility) | Uses `refSlot` to align the ZK-oracle report with the accounting report during a negative rebase. Only needed if the ZK oracle is connected to the protocol; otherwise the change can be skipped.                                                                                                                                                                       |

**`lidofinance/community-staking-module` (CSM):** CSM does **not** depend on `core`'s `HashConsensus`; it vendors its own copies under `src/lib/base-oracle/` and runs a **separate deployed `HashConsensus` instance** for its fee oracle (`src/FeeOracle.sol`). That fork carries the same two problems and must be migrated in lock-step (own PR, own deployment):

| Contract                                    | Change type      | Notes                                                                                                                                                                           |
|---------------------------------------------|------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `HashConsensus.sol` (CSM)                   | Upgrade required | Identical epoch/slot frame math **and** the same unbounded `_reportVariants` storage — so CSM's fee oracle is independently exposed to the same DoS. Both fixes apply here too. |
| `BaseOracle.sol` (CSM)                      | Upgrade required | Same `getChainConfig()` / `getInitialRefSlot()` calls and `ConsensusReport.refSlot` as core's `BaseOracle`.                                                                     |
| `FeeOracle.sol` (deployed as `CSFeeOracle`) | Rename           | Passes `data.refSlot` through only (no slot arithmetic) — easier than `core`'s `AccountingOracle`.                                                                              |
| `TwoPhaseFrameConfigUpdate.sol`             | To remove        | Frame-config update utility tied to the old slot-based `setFrameConfig`.                                                                                                        |

Because CSM's contracts are a separate fork with their own deployment, the `core` changes do not propagate to them automatically. They are in scope of this proposal and must be deployed together with the `core` changes so that CSM's fee oracle receives the DoS fix and the timestamp migration at the same time rather than being left exposed.

### Deployment Summary

**Newly deployed (KDF):**

| Contract                           | Deployed by                    | Notes                                                                                                                    |
|------------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| `DelegationFactory`                | DAO / deployer                 | Single new deployment establishing the KDF infrastructure.                                                               |
| `DelegationContract` (one per bot) | each operator, via the factory | One instance per Oracle/Council bot; deployed during the operator migration (Part 2), not by a single governance action. |

**Redeployed at a new address (standalone, non-upgradeable) — references must be repointed by governance:**

| Contract                        | Repointing required                                                                                                            |
|---------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| `HashConsensus` (core)          | Set as the consensus contract on `AccountingOracle` and `ValidatorsExitBusOracle` (`setConsensusContract`).                    |
| `HashConsensus` (CSM, vendored) | Set as the consensus contract on CSM's `FeeOracle`.                                                                            |
| `DepositSecurityModule`         | Update the `LidoLocator` reference and re-grant its deposit role; update any monitoring/config that hardcodes the DSM address. |
| `OracleReportSanityChecker`     | Only if the ZK-oracle path is adopted; update the `LidoLocator` reference.                                                     |

**Implementation redeployed behind the existing proxy (address preserved):**

| Contract                                    | Notes                                                                                                              |
|---------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| `AccountingOracle`                          | New implementation; proxy upgraded, address unchanged. Carries the `AccountingOracle` semantic-conversion changes. |
| `ValidatorsExitBusOracle`                   | New implementation; proxy upgraded, address unchanged.                                                             |
| CSM `FeeOracle` (deployed as `CSFeeOracle`) | New implementation; proxy upgraded, address unchanged.                                                             |

**Not redeployed unless the optional rename is applied:**

| Contract                                           | Notes                                                                                                                      |
|----------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| `RefSlotCache` (library), `VaultHub`, `LazyOracle` | Functionally unchanged (opaque reference value). Redeploy only if the cosmetic `refSlot` → `refTimestamp` rename is taken. |

Note that `HashConsensus` (both core and CSM) is the contract whose **address changes**; any off-chain service, dashboard, or governance config that hardcodes a `HashConsensus` address must be updated alongside the oracle `setConsensusContract` calls.

---

## Security Considerations

### HashConsensus DoS fix

The bounded storage model eliminates the spam vector entirely: no matter how many distinct hashes a member submits in a frame, only one is retained and evaluated. The cost of `submitReport` is now O(1) in stored variants.

The ability for members to change their submitted hash creates a subtle quorum-formation risk in adversarial conditions: if a member oscillates between two hashes, it could prevent a stable majority. The quorum is currently 5-of-9; a single oscillating member cannot prevent quorum formation if the remaining eight members agree.

### Key Delegation Framework

The delegation model shifts trust from "this hot key is the oracle" to "this delegation contract is the oracle." The delegation contract is non-upgradeable and minimal in scope. The admin Safe multisig is the new security boundary.

**Admin cannot act as bot by design.** The admin address has no ability to call `execute()` or produce EIP-1271-valid signatures. This is enforced at the contract level: the admin is kept maximally cold and can only manage the delegatee lifecycle.

**Pull integration (EIP-1271):** Consumer contracts must treat the `DelegationContract` address as the authorized principal, never the recovered signer EOA. A paused contract returns a non-magic value rather than reverting — consumers that check the return value will fail closed, but consumers that assume a revert on failure may inadvertently accept a paused contract's output.

**Push integration (`execute()`):** The delegatee can call any target. Permission grants must be narrowly scoped per contract, enforced by the one-contract-per-bot rule. ETH is forwarded to the target and any change is swept back to the delegatee after the call, so overpayment refunded by the target is recovered; only ETH the target actually consumes is non-recoverable.

**Admin compromise.** An attacker with admin access can immediately assign a delegatee they control and have it act within the seat's permissions — there is no delay and no reaction window, so this is the model's most serious failure mode and the reason the admin must be a cold, high-threshold multisig. The attacker still cannot call `execute()` or sign directly (the admin-cannot-execute invariant holds), so they must route actions through an assigned delegatee, and the `DelegateeAssigned` event fires on-chain for monitoring to catch. The blast radius is bounded to that single contract's permissions.

As a last resort — while the legitimate admin still controls the key — it can `pause()` the contract. `pause()` is irreversible by design: once paused, no action can be taken under the contract's authority by anyone, and an attacker who later gains admin access cannot un-pause it. The cost is that a new contract must be deployed and governance must reassign the seat, but this is preferable to leaving an attacker able to act through the contract. This irreversibility must be explicit in operator training material.

**Delegatee compromise.** An attacker with only the delegatee key can act within the contract's protocol permissions but cannot assign a new delegatee, cannot pause, and cannot perform any admin operation. The admin can instantly revoke the delegatee via `revokeDelegatee()`.

**One contract per bot.** Mandatory. Sharing a delegation contract across roles means a single compromise affects multiple protocol functions. Enforced at the governance permission-grant step.

## Links

- EIP-1271 (standard signature validation): https://eips.ethereum.org/EIPS/eip-1271
- EIP-7782 (Ethereum slot time reduction): https://eips.ethereum.org/EIPS/eip-7782
- EIP-8198 (Quick Slots — runtime-configurable slot duration): https://eips.ethereum.org/EIPS/eip-8198
- Core repository (`lidofinance/core`): https://github.com/lidofinance/core
- Delegation contracts repository (`lidofinance/delegation-execution-authority`): https://github.com/lidofinance/delegation-execution-authority
- Oracle daemon (`lidofinance/lido-oracle`): https://github.com/lidofinance/lido-oracle
- Community Staking Module — forked oracle stack (`lidofinance/community-staking-module`): https://github.com/lidofinance/community-staking-module
