---
lip: <to be assigned>
title: "Key Delegation Authority"
status: WIP
author: Raman Siamionau
discussions-to: <Create a new thread on https://research.lido.fi/ and drop the link here>
created: 2026-06-01
---

## Simple Summary

This proposal bundles four related improvements to Lido's infrastructure into a single release:

1. **Key Delegation Framework (KDF)** — introduces a general-purpose on-chain delegation layer that sits in front of any permissioned key in the protocol, allowing its operator to rotate the hot signing key instantly, without governance voting.
2. **KDF adoption within Oracle and DSM scope** — specifies the concrete off-chain and on-chain changes required to adopt KDF across Lido Oracles and Council daemons.
3. **HashConsensus vulnerability fix** — closes a denial-of-service vector in the `HashConsensus` contract by bounding the number of stored report variants per frame ([`lidofinance/core#1379`](https://github.com/lidofinance/core/issues/1379)).
4. **Timestamp-based oracle frames** — replaces slot/epoch-based frame logic in `HashConsensus` with wall-clock seconds, so oracle frame lengths stay stable across future Ethereum hardforks (including the EIP-7782 slot time reduction).

## Abstract

This upgrade pursues two main goals: to improve the security of the protocol, and to abstract existing functionality away from slots and epochs so that future upgrades are simpler to perform.

On the security side, we propose to standardize the Key Delegation Framework (KDF) as a protocol-wide primitive and to adopt it for every key that holds any protocol permission. KDF inserts a factory-deployed, per-entity delegation contract between the protocol and each operator's hot key: governance grants permissions to the delegation contract, while the operator's cold key or multisig controls which hot key is active and can rotate it at any time without a governance vote, or irreversibly lock the contract should the cold key itself be compromised. As the first batch of adopters, we propose to begin using KDF immediately for Oracle and Council operators, so that the response to a key compromise can be measured in minutes rather than the ~10 days a governance vote requires today. Beyond incident response, periodic hot-key rotation is itself a valuable security practice, and KDF makes it possible to perform it routinely.

On the abstraction side, we propose to rewrite the `HashConsensus` contract ([`lidofinance/core`](https://github.com/lidofinance/core)) to express all frame logic in wall-clock seconds (`secondsPerFrame`, `fastLaneSeconds`, `initialTimestamp`, `refTimestamp`) instead of epoch- and slot-based parameters. This decouples oracle frame durations from Ethereum's consensus-layer slot timing and future-proofs oracle operation against changes such as the [EIP-7782](https://eips.ethereum.org/EIPS/eip-7782) slot-time reduction. The same redeployment also closes a denial-of-service vector in `HashConsensus`: a member can no longer inflate the report-variant array with distinct hashes, because abandoned variant slots are reused, capping the array — and every loop over it — at the committee size, eliminating the unbounded iteration that enables the attack.

Concretely, we propose to deploy an updated `HashConsensus` contract, a new `DelegationFactory` / `DelegationContract` system (repository: [`lidofinance/delegation-execution-authority`](https://github.com/lidofinance/delegation-execution-authority)), and targeted updates to the [`lido-oracle`](https://github.com/lidofinance/lido-oracle) daemon and `DepositSecurityModule`. The adoption scope specifies the integration points: the `lido-oracle` daemon gains delegation support via a `DELEGATION_CONTRACT_ADDRESS` environment variable, and the `DepositSecurityModule` is updated to verify guardian signatures by recovering the signer and checking it against the guardian's `DelegationContract.getDelegatee()`, enabling council members to operate behind delegation contracts. Adoption is mandatory for these roles; a coordinated migration process is defined for existing Oracle and DSM operators to onboard to the new model without service interruption.

## Motivation

### Hot-key operational risk for Oracle and Council operators

Oracle members and Deposit Security Module Committee council members currently operate with hot EOA private keys stored directly in off-chain bots. These keys:

- Carry meaningful protocol permissions (report submission, pause/unvet signing, deposit message signing).
- May be long-lived with an unclear custody history.
- Require a full on-chain governance vote (~10 days) to rotate in the event of compromise.

By their very nature, these bots must operate with hot keys: the signing key lives on the machine, which makes it inherently exposed and comparatively easy to compromise. The quorum requirements these roles operate under already absorb most of the risk of any single such key being compromised — a lone compromised member cannot, on its own, move the protocol — so the long voting window is not in itself the core concern. The real problem is operational: rotating a hot key today requires a full on-chain governance vote, drawing in the dev team and token holders for what should be a routine key-management action. That makes rotation burdensome both for proactive, periodic rotation and, more acutely, for responding to an active compromise. And because a replacement key is only authorized once the vote executes, an operator that stops trusting a suspect key cannot use a new one in the meantime — the seat simply stops participating until governance acts.

### HashConsensus DoS vulnerability

The current `HashConsensus` contract stores every unique report hash submitted by any oracle member in a per-frame array (`_reportVariants` / `_reportVariantsLength`). Consensus is determined by iterating the full array. Because a single oracle member can submit an unlimited number of distinct hashes, a member can inflate this array, making each subsequent `submitReport` and `_isQuorumReached` call more expensive — the iteration cost grows with every additional report variant stored. This exposes an attack vector in which a single compromised key can block reports from being completely delivered.

The attack works by submitting so many alternative report variants that iterating over them no longer fits within a single transaction's gas limit. At that point report submission can no longer be completed in one transaction and quorum can no longer be reached. The practical consequence is a halt of oracle reporting: for as long as the attack is sustained, `HashConsensus` cannot reach consensus and the protocol cannot process the accounting reports that drive stETH rebases, withdrawal request finalization, and validator exit requests. The DoS therefore degrades core protocol operation, not merely the `HashConsensus` contract in isolation.

### Ethereum slot-time changes (EIP-7782, EIP-8198)

Ethereum's consensus-layer slot timing is in flux. Competing proposals such as [EIP-7782](https://eips.ethereum.org/EIPS/eip-7782) (12 s → 6 s) and [EIP-8198](https://eips.ethereum.org/EIPS/eip-8198) ("Quick Slots", 12 s → 8 s) would reduce slot duration and, consequently, halve or otherwise shorten the duration of all oracle frames currently configured in epochs. Under EIP-7782, for example, oracle reporting periods would shrink from roughly 28 days to 14 days, disrupting downstream protocol assumptions.

EIP-8198 goes further by making slot duration a runtime-configurable parameter rather than a hardcoded constant, with the value expected to be tuned iteratively across devnets. Any frame logic anchored to `slotsPerEpoch × secondsPerSlot` is therefore not just fragile across a single hardfork but perpetually exposed to re-tuning. Migrating frame configuration from `epochsPerFrame` to `secondsPerFrame` removes the slot dependency entirely, making frame duration invariant to any present or future slot-timing change.

## Specification

### Overview

The release consists of four coordinated changes:

1. Deployment of the `DelegationFactory`, establishing the general-purpose on-chain delegation infrastructure (KDF).
2. An update to the `DepositSecurityModule` contract to verify guardian signatures against the guardian's `DelegationContract.getDelegatee()`, so that Council guardians can operate behind KDF delegation contracts.
3. Modifications to the Oracle and Council daemons to support operating behind delegation contracts.
4. Modifications to the `HashConsensus` contract and related contracts — in both `lidofinance/core` and CSM's separately-deployed fork — to fix the DoS vulnerability and switch to seconds-based frames.

### Rationale

**Why the changes ship as one release.** The proposal groups two otherwise independent tracks — the KDF security work and the `HashConsensus` rework — into a single release to amortize the audit, deployment, and operator-coordination effort.

**Why KDF ships with its adoption scope.** The `DelegationFactory` and `DelegationContract` are inert on their own: without the Oracle and DSM integration they deliver no security benefit and give operators no migration path. Shipping the contracts together with their adoption scope is what actually closes the hot-key risk in this release rather than deferring it.

**Why a purpose-built own delegation contract rather than reusing an existing solution.** A general-purpose delegation or smart-account framework is far more flexible than this role needs, and that unused flexibility is attack and misconfiguration surface. A minimal, non-upgradeable contract instead makes KDF's guarantees structural — admin can never `execute()` or sign, one active delegatee, immediate revocation with a cooldown-gated assignment, irreversible `terminate()` — keeping the audited surface small.

### Technical Specification

---

#### Part 1: Key Delegation Framework (KDF) — Contracts

##### Architecture

The delegation layer is composed of two contracts deployed from a single factory:

- **`DelegationFactory`** — a singleton that deploys `DelegationContract` instances. Each permissioned entity (oracle operator, council member) deploys exactly one `DelegationContract` per bot. A single delegation contract **must not** be shared across multiple bots or protocol roles; each independently operating bot requires its own instance.
- **`DelegationContract`** — a non-upgradeable, minimal on-chain delegation contract. It enforces a one-admin / one-active-delegatee model and supports two integration patterns described below.

##### Roles

The contract has exactly two roles, **admin** and **delegatee**:

| Role                    | Custody                                                    | Capabilities                                                                                                                    |
|-------------------------|------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| **Admin**               | Safe multisig or other audited multisig with justification | Assign, reassign, and revoke the delegatee (`assignDelegatee()` / `revokeDelegatee()`); irreversibly `terminate()` the contract |
| **Delegatee (Hot Key)** | Hot key in the off-chain daemon                            | Call `execute()` to dispatch transactions (`push`); sign messages that integrators verify against `getDelegatee()` (`pull`)     |

The admin manages the delegatee's lifecycle but can never act *as* the contract. Termination is the escape hatch for admin-key compromise: because `terminate()` is irreversible — it permanently disables `execute()` and clears the delegatee (so `getDelegatee()` returns `address(0)`) — a still-trusted admin can neutralize the contract entirely, stripping the delegatee of any usable role under its authority.

##### Contract Interface

```solidity
interface IDelegationContract {
    // --- Admin controls ---

    /// @notice Assign (or reassign) the active delegatee.
    ///         Only callable by admin. The previous delegatee is dropped at
    ///         once and the new one becomes effective only after the contract's
    ///         cooldown (`getCooldown()` seconds).
    ///         Reverts if delegatee == admin.
    /// @param delegatee Address of the incoming delegatee.
    function assignDelegatee(address delegatee) external;

    /// @notice Remove the current and any pending delegatee. 
    ///         Only callable by admin.
    function revokeDelegatee() external;

    /// @notice Terminate the contract, permanently disabling execute().
    ///         Only callable by admin.
    ///         Also clears the active delegatee (as revokeDelegatee), so
    ///         getDelegatee() returns address(0) after termination.
    ///         Intended for emergency use when the admin is suspected compromised.
    ///         Termination is irreversible; a new DelegationContract must be
    ///         deployed and governance must reassign the protocol seat.
    function terminate() external;

    // --- Push integration ---

    /// @notice Execute an arbitrary call on behalf of this contract.
    ///         Only callable by the current delegatee.
    ///         Reverts if the contract is terminated.
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

    // --- Views ---

    /// @notice Returns the current admin address.
    function getAdmin() external view returns (address);

    /// @notice Returns the currently *effective* delegatee, or address(0) if
    ///         none. A delegatee assigned within the last `getCooldown()`
    ///         seconds is not yet effective — this returns the address(0)
    ///         until the cooldown elapses. Returns address(0) once the
    ///         contract is terminated.
    function getDelegatee() external view returns (address);

    /// @notice Returns the pending (not-yet-effective) delegatee and the
    ///         timestamp at which it becomes effective, or (address(0), 0) if
    ///         no assignment is in cooldown.
    function getPendingDelegatee() external view returns (address delegatee, uint256 activeFrom);

    /// @notice Cooldown in seconds between assigning a delegatee and it
    ///         becoming effective. Set in the constructor and immutable
    ///         thereafter.
    function getCooldown() external view returns (uint256);

    /// @notice Returns true if the contract has been terminated.
    function isTerminated() external view returns (bool);

    // --- Events ---

    event DelegateeAssigned(address indexed newDelegatee, uint256 activeFrom);
    event DelegateeRevoked(address indexed revokedDelegatee);
    event Terminated();
}

interface IDelegationFactory {
    /// @notice Deploy a new DelegationContract.
    /// @param admin     Address to set as the contract's admin. Fixed for the
    ///                  lifetime of the contract — there is no on-chain way to
    ///                  change it; replacing the admin requires deploying a new
    ///                  contract and a governance vote to reassign the seat.
    /// @param delegatee Initial active delegatee, set in the constructor (and
    ///                  effective immediately — the cooldown applies only to
    ///                  later reassignments). Pass address(0) to deploy with no
    ///                  delegatee and assign one later via assignDelegatee().
    /// @param cooldown  Seconds a reassigned delegatee waits before becoming
    ///                  effective (see assignDelegatee). Set in the constructor
    ///                  and immutable thereafter; may be 0 to disable the
    ///                  cooldown.
    /// @return instance Address of the newly deployed DelegationContract.
    function deploy(address admin, address delegatee, uint256 cooldown)
        external
        returns (address instance);

    /// @notice Emitted for each DelegationContract deployed by the factory,
    ///         so the full set of KDF instances can be discovered on-chain
    ///         (e.g. for the monitoring registry) by indexing this event.
    event DelegationContractDeployed(
        address indexed instance,
        address indexed admin,
        address delegatee
    );
}
```

##### Integration Models

The `DelegationContract` is designed as a **general integration primitive** for any permissioned bot in the Lido ecosystem. Two complementary patterns are supported; protocol integrations should select the one that fits their verification model.

###### Pull Style — Signature Verification against `getDelegatee()`

The integrator (protocol) contract verifies a message itself: it recovers the signer from the signature and checks that it matches the registered entity's current delegatee, read via `getDelegatee()`.

The integrator must know which `DelegationContract` to check. The address can be relayed alongside the signature (as the DSM integration does), or — for stronger binding — embedded directly in the signed message, so the signature is bound to a specific delegation contract and the integrator reads the delegator target from the message.

```
Off-chain bot (delegatee)          DelegationContract          Protocol contract
        │                                  │                           │
        │  sign message (ECDSA, hot key)   │                           │
        │                                  │                           │
        │─ signature + DelegationContract addr relayed off-chain ─────►│
        │                                  │                           │
        │                                  │◄──── getDelegatee() ──────│
        │                                  │───── delegatee ──────────►│
        │                                  │                           │
        │                                  │   integrator checks:      │
        │                                  │   ecrecover(sig)==        │
        │                                  │   delegatee, and addr is  │
        │                                  │   a registered principal  │
```

**Security assumptions:**
- The integrator must treat the `DelegationContract` address — not the recovered hot key — as the authorized principal: it checks that the supplied `DelegationContract` is a registered entity (e.g. guardian) and that the recovered signer equals that contract's `getDelegatee()`.
- The signing key must match `getDelegatee()` at the time of on-chain verification. A key rotated (or the contract terminated, which sets `getDelegatee()` to `address(0)`) after signing but before verification causes rejection — failing closed.
- The integrator **must reject a zero delegatee.** `ecrecover` returns `address(0)` for a malformed signature, and `getDelegatee()` returns `address(0)` when no delegatee is assigned (revoked/terminated/unassigned). Comparing the two naively would let a garbage signature validate as `0 == 0`, so the integrator must require `getDelegatee() != address(0)` (equivalently, reject a recovered `address(0)`) before accepting.

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

`execute()` is `payable` and forwards `msg.value` to the target. This is required to support future permissioned bots that interact with payable protocol functions, such as EIP-7002 triggerable withdrawal requests and EIP-7251 consolidation requests, where a fee must be paid in ETH at call time. After the target call returns, `execute()` sweeps any ETH still held by the contract back to the delegatee, so change refunded by the target is forwarded to the bot rather than stranded on the delegation contract.

**Security assumptions:**
- The target contract must verify `msg.sender == DelegationContract` is a registered permission holder (e.g., oracle committee member, DSM guardian). It must not rely on `tx.origin`.
- The delegatee can call any target with any data; the contract does not restrict the call target. Governance must ensure the `DelegationContract` address holds only the narrowest set of permissions needed.
- If the target call reverts, `execute()` reverts atomically and forwarded ETH is returned to the delegatee. On a *successful* call, any ETH the target refunds to the contract (change) is swept back to the delegatee before `execute()` returns; only ETH the target actually consumes/keeps is non-recoverable.

**When to use:** Any integration where the bot submits transactions that modify on-chain state and the protocol contract checks `msg.sender` for authorization. Examples: oracle report submission (`submitReportData`) or any permissioned bots.

##### Admin Immutability

The admin address is fixed at deployment and **cannot be changed on-chain** — the contract exposes no admin-transfer function. This is a deliberate security choice: a `changeAdmin`-style call would be the single most damaging action available to a compromised admin, letting an attacker lock the legitimate owners out permanently. Removing it bounds the worst case — even a compromised admin cannot seize ownership of the contract, only act within the delegatee lifecycle, which the legitimate owners and governance can still contain.

Rotating the admin (e.g., moving the seat to a different multisig, or re-keying outside an in-place Safe upgrade) therefore requires deploying a fresh `DelegationContract` and a DAO governance vote to reassign the protocol seat to it. This also prevents silent transfer of committee participation between organizations and avoids legal ambiguity over who is accountable for the seat.

##### Delegatee Assignment and Revocation

Assignment is cooldown-gated; revocation is immediate:

- **Assignment** (`assignDelegatee(newHotKey)`) drops the previous delegatee at once and schedules the new one, which becomes effective only after the contract's `cooldown` elapses (immediately if the cooldown is 0). **During the cooldown the contract has no effective delegatee** — `getDelegatee()` returns `address(0)`, so the seat is inactive until the new key activates. Reassigning before the cooldown elapses overwrites the pending one and restarts the cooldown. The cooldown is a reaction window: an unexpected assignment (e.g. from a compromised admin) is visible via the `DelegateeAssigned` event and `getPendingDelegatee()` for `cooldown` seconds before the new key can act — and during that window no key can act at all.
- **Revocation** (`revokeDelegatee()`) is immediate and clears both the current and any pending delegatee, leaving the contract with no delegatee until a new one is assigned. No cooldown applies — *inaction (no active delegatee) is always safer than a malicious action*.

The cooldown is set in the constructor and is immutable; it may be 0. The recommended value for both Oracle and DSM delegation contracts is **1 hour** — long enough to give monitoring and the DAO a reaction window against an unexpected assignment, short enough that the seat's brief inactivity during a rotation is tolerable under quorum. Because a rotation takes the seat inactive for the cooldown, the expected sequence on a suspected delegatee compromise is simply `assignDelegatee(newHotKey)` (which drops the compromised key immediately and brings the new one online after the cooldown), or `revokeDelegatee()` if no replacement is ready yet.

##### Emergency Termination

The admin may call `terminate()` to permanently disable the contract's `execute()` function; it also clears the active delegatee (equivalent to `revokeDelegatee()`) as part of the same call. This is an **irreversible** operation: a terminated contract cannot be reactivated. After termination:

- All delegatee calls to `execute()` revert.
- `getDelegatee()` returns `address(0)`, so any pull-style integrator that resolves the active delegatee through it (to verify a signature) fails closed and does not keep trusting the last delegatee.
- A new `DelegationContract` must be deployed and governance must reassign the protocol seat to the new address.

Termination is intended for scenarios where the admin believes the admin's own address has been compromised and wishes to ensure no further actions can be taken under its authority. The expected sequence is: detect compromise → `terminate()` → notify DAO → deploy new delegation contract → governance vote.

##### Hot-Key Rotation

The admin rotates the active delegatee by calling `assignDelegatee(newHotKey)`. The old key stops being accepted immediately, and the new key becomes effective after the cooldown — so the seat is inactive for the cooldown duration during a rotation (tolerable under quorum). Plan routine rotations accordingly.

The specific cadence for routine rotation is not mandated here; recommended practices will be maintained on the Lido research forum and may be revised. Contract-level requirements are:

- Rotation must be performed promptly upon any suspected or confirmed hot-key compromise. If a replacement key is not immediately ready, `revokeDelegatee()` should be called first to take the compromised key out of service.
- After assignment, the operator verifies the new delegatee via `getDelegatee()` and updates the daemon configuration before restarting.

For monitoring purposes, the `DelegateeAssigned` event (carrying the `activeFrom` timestamp) flags every assignment the moment it is scheduled, and `getPendingDelegatee()` / `getDelegatee()` expose the pending and effective delegatee on-chain — so an unexpected rotation can be caught during its cooldown.

##### Admin Address Requirements

The admin is the critical security boundary of the delegation model. The principles below apply; the detailed operational requirements — specific hardware-wallet models, minimum multisig quorums, and approved implementations — are maintained in a dedicated "Key Handling Policy" document on the Lido research forum.

**Strongly recommended: Safe (Gnosis Safe) multisig.** Safe is the preferred admin implementation because it is battle-tested at Lido scale and its upgrade path is expected to include post-quantum (PQ) signature support — an assumption this design explicitly relies on. A Safe-based admin should be upgradeable to PQ-compatible signing without replacing the multisig address or requiring a governance vote for the delegation contract.

##### Permission Model

Governance grants protocol permissions (e.g., oracle committee membership, DSM guardian status) to the **`DelegationContract` address** rather than to the hot EOA directly. The protocol's permission registry is unchanged; only the address type of the permission holder changes.

##### Repository and Implementation

- **Smart contracts**: `lidofinance/delegation-execution-authority` (Solidity + Foundry framework)
  - `DelegationFactory` and `DelegationContract`
  - `getDelegatee()`-based delegation; payable `execute()` with value forwarding; cooldown-gated `assignDelegatee()` and immediate `revokeDelegatee()`; irreversible `terminate()`; admin-cannot-execute invariant enforced by design
  - Full test suite including mainnet fork tests (Foundry, covering contract logic, factory deployment, access control, payable execution, assignment/revocation, and termination behavior)

---

#### Part 2: KDF Adoption — Oracle and DSM Integration

This section defines which components are migrated to the delegation model in this release, the concrete changes required for each, and the operator migration process. Adoption is mandatory for Oracle and DSM roles.

##### Scope of Migration in This Release

| Component                              | Integration style                                                                                                            |
|----------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Lido Oracle (holds delegatee key)      | Push (`execute()`)                                                                                                           |
| `DepositSecurityModule` contract       | Pull (DSM recovers signer, checks against `getDelegatee()`)                                                                  |
| Council daemon (holds delegatee key)   | Pull (signs messages, publishes its `DelegationContract` address with each signature, DSM verifies)                          |
| Depositor bot                          | Claims the council-provided `(signature, DelegationContract address)` pairs; re-sorts the deposit array by guardian address. |
| Validator ejector (node-operator side) | Off-chain only — must resolve the active delegatee via `getDelegatee()`. See Validator Ejector section below                 |

All components in the table are migrated together.

---

###### Oracle Daemon — Push Integration

**Pattern:** Push via `execute()`.

**Authorization model:** `HashConsensus` and the report processors (`AccountingOracle`, `ValidatorsExitBusOracle`, and CSM's `FeeOracle`) check `msg.sender` against the registered oracle committee member set. After migration, `msg.sender` is the `DelegationContract` address, which governance must have registered as the committee member before the daemon switches over.

**Daemon configuration:** `DELEGATION_CONTRACT_ADDRESS` environment variable. When set, all oracle contract calls are routed through `DelegationModule`, a Web3py extension that wraps `execute()` dispatch. Startup validation checks: (a) `getDelegatee() == configuredHotKey`, and (b) `delegationContract` address holds oracle committee membership in `HashConsensus`.

**Transitional behavior:** When `DELEGATION_CONTRACT_ADDRESS` is unset, calls are sent directly from the hot key EOA as before. This path exists for development, for the migration window, and as a long-term fallback (e.g. if native account-abstraction delegation later meets these needs directly).

---

###### DSM Contract — Pull Integration

**Pattern:** Pull — DSM recovers the signer and checks it against the guardian's `getDelegatee()`.

**Authorization model today.** `DepositSecurityModule` does **not** take the guardian address as an input — it *derives* it: every verification path (`_verifyAttestSignatures` for deposits, plus `pauseDeposits` and `unvetSigningKeys`) calls `ECDSA.recover(msgHash, sig.r, sig.vs)` and looks the recovered EOA up in the guardian set via `_isGuardian`.

**Required change — the guardian address must be supplied with the signature.** A contract guardian cannot be recovered from a signature, so the submitter provides the registered guardian address (the `DelegationContract`, or the EOA for a legacy guardian) **explicitly alongside each signature**. DSM keeps doing the ECDSA recovery itself; for a contract guardian it then checks the recovered signer against that contract's `getDelegatee()`.

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
    address recovered = ECDSA.recover(msgHash, r, vs);

    if (guardian.code.length > 0) {
        // Contract guardian: recovered signer must be the contract's delegatee.
        // getDelegatee() returns address(0) if revoked/terminated; the explicit
        // non-zero check stops a malformed signature (recovered == address(0))
        // from matching a contract with no active delegatee (0 == 0).
        address delegatee = IDelegationContract(guardian).getDelegatee();
        return delegatee != address(0) && delegatee == recovered;
    }
    // Legacy EOA guardian: recovered signer must equal the claimed guardian.
    return recovered == guardian;
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
    GuardianSignature calldata sig   // was Signature
) external;

function unvetSigningKeys(
    uint256 blockNumber,
    bytes32 blockHash,
    uint256 stakingModuleId,
    uint256 nonce,
    bytes calldata nodeOperatorIds,
    bytes calldata vettedSigningKeysCounts,
    GuardianSignature calldata sig   // was Signature
) external;
```

**Transitional behavior:** The existing ECDSA path is retained, so a guardian still registered as an EOA continues to be verified unchanged. This path exists for development, for the migration window, and as a long-term fallback (e.g. if native account-abstraction delegation later meets these needs directly).

---

###### Council Daemon — Pull Integration (Signer Side)

**Pattern:** Pull (the daemon signs; DSM recovers the signer and checks it against the guardian's `getDelegatee()`).

The council daemon produces ECDSA signatures over the standard DSM message using the **delegatee hot key**. The signed digest is unchanged. Because DSM no longer derives the guardian by recovery (see DSM Contract above), **the council daemon's responsibility is to publish, for each message, both the signature and its registered guardian address** — the daemon's `DelegationContract` address, not the delegatee EOA.

On-chain, DSM is given that guardian address explicitly, sees it is a contract, recovers the signer from the signature, and checks that it equals the contract's `getDelegatee()`.

**Required change to the council daemon:** Sign with the delegatee private key, set `DELEGATION_CONTRACT_ADDRESS`, and publish that `DelegationContract` address together with each signature (so consumers know which registered guardian the signature is for). No structural code change is required beyond configuration and attaching the guardian address to each published signature.

**Required change to the depositor bot:** The bot relays the council-provided `(DelegationContract address, signature)` pairs to DSM as-is. Two behavioral changes are required:

1. **Filtering (on receival):** accept an incoming message only if its signer is the **current delegatee** of the referenced guardian's `DelegationContract` (resolved via `getDelegatee()`), instead of matching against a delegation-contract address. Messages signed by a rotated-out key are dropped.
2. **Ordering (on submission):** when assembling `depositBufferedEther`, sort the signature array by the council-supplied guardian address (matching DSM's new ascending-by-`guardian` ordering) rather than by recovered signer.

---

###### Validator Ejector (Node-Operator Side) — Off-Chain Update

The VEBO module of `lido-oracle` is covered by the push integration above. Separately, the **validator ejector** run by node operators — the daemon that watches `ValidatorsExitBusOracle` exit request events and sends voluntary exit messages to the consensus layer — performs no on-chain transactions and needs no `DelegationContract` of its own. It does, however, need updates to operate correctly once oracles sit behind delegation contracts:

- Any check that validates the origin of an exit request against a configured oracle address must compare against the `DelegationContract` address (the registered committee member), not the delegatee EOA.
- For verifying signatures on oracle messages, the signer is always an **EOA** — the **delegatee** under delegation, or the directly-permissioned EOA in the legacy setup. The ejector keeps a whitelist of accepted signing addresses and must populate it with that signing EOA, i.e. the delegatee — **not** the `DelegationContract` address, which never signs. When an operator rotates the delegatee, the whitelist must be updated to the new delegatee EOA (the same way the oracle's signing key is configured today).

---

##### Operator Migration Process

Migration is designed to be zero-downtime. All preparatory steps are completed off the critical path, leaving a single irreversible governance vote that flips every prepared seat to delegation at once. The reassignment of a committee seat from a hot EOA to a `DelegationContract` cannot be undone without a further governance vote, so operators are expected to validate the full flow on the Hoodi testnet before the mainnet vote.

1. **Deploy and configure contracts (operators)**: each operator deploys its `DelegationContract` via `DelegationFactory.deploy(adminAddress, hotKey, cooldown)` — the admin being a Safe multisig, fixed for the contract's lifetime (see Admin Immutability in Part 1); the active delegatee (its existing hot key) set in the same transaction; and a `cooldown` of **1 hour** for both Oracle and DSM contracts (reduced or 0 on testnet). (Passing `address(0)` for the delegatee and assigning it later via `assignDelegatee()` is also supported.)
2. **Publish and verify addresses (operators)**: each operator publishes its `DelegationContract` and admin addresses on the Lido research forum, so the DAO and other operators can verify them ahead of the vote.
3. **Prepare daemons for rotation (operators)**: each operator stages the delegatee key and the `DELEGATION_CONTRACT_ADDRESS` configuration in its Oracle and Council daemons, so they are ready to operate via the delegation contract the moment the seat is reassigned. No key material changes, and the daemons keep operating from the hot EOA until the vote lands.
4. **Set up monitoring (Lido team)**: the Lido team registers every delegation contract in the monitoring infrastructure and begins watching the admin and delegatee addresses for unexpected activity, so anomalies are caught both before and after the seat reassignment.
5. **Governance vote**: a single DAO vote reassigns each Oracle committee and DSM guardian seat from the operator's hot EOA to its `DelegationContract` address — a `HashConsensus` member update for oracles, a `DepositSecurityModule` guardian replacement for guardians; the pre-configured daemons begin routing through the delegation contract without interruption.

Until the governance vote in step 5 executes, operators continue to run as EOA principals; the EOA paths described above keep them online throughout preparation. After the vote, each operator confirms the switchover via `getDelegatee()` and a successful report or guardian message in the following frame, and may rotate the hot key at any time via `assignDelegatee()` with no further governance vote. Adoption is not optional as an end state — every Oracle and DSM operator is required to complete the migration.

---

#### Part 3: HashConsensus — Bounded Report Variant Storage

The DoS comes from the unbounded `_reportVariants` array: a member submitting many distinct hashes grows it without limit, so the loops over it (`submitReport`'s hash lookup and `_isQuorumReached`) become arbitrarily expensive.

In new solution each member has exactly one live vote, so at most committee-size variants ever have non-zero support; switching hashes decrements the old variant's support, and a slot at zero support is abandoned. Instead of always appending a new variant, the contract reuses an abandoned slot when one exists and only grows the array when every slot is still live:

```diff
 uint64 varIndex = 0;
+uint64 reuseIndex = type(uint64).max;  // first abandoned (support == 0) slot, if any
 while (varIndex < variantsLength && _reportVariants[varIndex].hash != report) {
+    if (reuseIndex == type(uint64).max && _reportVariants[varIndex].support == 0) {
+        reuseIndex = varIndex;
+    }
     ++varIndex;
 }
 ...
 } else {  // new hash
+    if (reuseIndex != type(uint64).max) varIndex = reuseIndex;  // reuse abandoned slot
+    else _reportVariantsLength = ++variantsLength;              // otherwise grow
     support = 1;
     _reportVariants[varIndex] = ReportVariant({hash: report, support: 1});
-    _reportVariantsLength = ++variantsLength;
 }
```

This caps `_reportVariantsLength` — and every loop over the variants — at the committee size, no matter how many hashes a member submits. All existing semantics are preserved: resubmitting the identical hash still reverts (`DuplicateReport`); a different hash still moves the member's vote and triggers `discardConsensusReport` if it drops a reached consensus below quorum.

CSM runs a vendored copy of `HashConsensus` with the same `_reportVariants` storage and is independently exposed to this DoS; the same fix is applied there (see Affected Contracts in Part 4).

---

#### Part 4: HashConsensus — Timestamp-Based Frames

##### Motivation

There is a fairly high probability that Ethereum will change the slot duration, and possibly the epoch length, in the future — several EIPs already propose exactly this (e.g. [EIP-7782](https://eips.ethereum.org/EIPS/eip-7782), reducing slot time to 6 s, and [EIP-8198](https://eips.ethereum.org/EIPS/eip-8198) "Quick Slots", making slot duration runtime-configurable), though none has yet been scheduled into a fork. Oracle frame lengths are currently expressed as `epochsPerFrame`, which translates to seconds via `epochsPerFrame × slotsPerEpoch × secondsPerSlot`. Any such change would alter the wall-clock duration of every frame — under a 12 s → 6 s reduction, for instance, the same `epochsPerFrame` would yield half the intended duration.

Rather than re-tune `epochsPerFrame` for each potential change, we express frames directly in wall-clock seconds (`secondsPerFrame`). This removes the dependency on slot/epoch timing entirely, so frame duration stays stable across any present or future slot-timing change — whether a one-off reduction or a runtime-configurable schedule.

##### Interface Modifications

###### `IReportAsyncProcessor` (implemented by `BaseOracle`, and therefore by `AccountingOracle` and `ValidatorsExitBusOracle`; CSM runs a separate forked copy — see Affected Contracts below)

```diff
 interface IReportAsyncProcessor {
     function submitConsensusReport(
         bytes32 _report,
-        uint256 _refSlot,
+        uint256 _refTimestamp,
         uint256 _deadline
     ) external;

-    function discardConsensusReport(uint256 _refSlot) external;
+    function discardConsensusReport(uint256 _refTimestamp) external;
 }
```

Existing implementations receive a `uint256` regardless; because `refTimestamp` values are numerically larger than any historical `refSlot`, all on-chain value comparisons (e.g., "must be greater than last processed value") remain valid without modification. Any logic that interprets the value semantically as a slot number is updated in this release as part of the protocol-wide ABI migration.

###### View Functions

```diff
-function getFrameConfig() external view returns (
-    uint256 initialEpoch,
-    uint256 epochsPerFrame,
-    uint256 fastLaneLengthSlots
-);
+function getFrameConfigV2() external view returns (
+    uint256 initialTimestamp,
+    uint256 secondsPerFrame,
+    uint256 fastLaneSeconds
+);
-function getCurrentFrame() external view returns (
-    uint256 refSlot,
-    uint256 reportProcessingDeadlineSlot
-);
+function getCurrentFrameV2() external view returns (
+    uint256 refTimestamp,
+    uint256 reportProcessingDeadlineTimestamp
+);
-function getMembers() external view returns (
-    address[] memory addresses,
-    uint256[] memory lastReportedRefSlots
-);
+function getMembersV2() external view returns (
+    address[] memory addresses,
+    uint256[] memory lastReportedRefTimestamps
+);
-function getFastLaneMembers() external view returns (
-    address[] memory addresses,
-    uint256[] memory lastReportedRefSlots
-);
+function getFastLaneMembersV2() external view returns (
+    address[] memory addresses,
+    uint256[] memory lastReportedRefTimestamps
+);
-function getConsensusState() external view returns (
-    uint256 refSlot,
-    bytes32 consensusReport,
-    bool isReportProcessing
-);
+function getConsensusStateV2() external view returns (
+    uint256 refTimestamp,
+    bytes32 consensusReport,
+    bool isReportProcessing
+);
```

**Why the `V2` suffix.** Each of these view getters changes its return **semantics** from slots/epochs to timestamps, so it is given a `V2` name to change its selector. If the name were kept, the selector would be identical (the getters take no arguments) and an out-of-protocol caller of e.g. `getFrameConfig()` would keep compiling and silently receive a timestamp where it expected an epoch — the most dangerous outcome.

###### `MemberConsensusState` Struct

```diff
 struct MemberConsensusState {
-    uint256 currentFrameRefSlot;
+    uint256 currentFrameRefTimestamp;
     bytes32 currentFrameConsensusReport;
     bool    isMember;
     bool    isFastLane;
     bool    canReport;
-    uint256 lastMemberReportRefSlot;
+    uint256 lastMemberReportTimestamp;
     bytes32 currentFrameMemberReport;
 }
```

###### Setter / Admin Functions

```diff
-function setFrameConfig(uint256 epochsPerFrame, uint256 fastLaneLengthSlots) external;
-function submitReport(uint256 slot, bytes32 report, uint256 consensusVersion) external;
-function updateInitialEpoch(uint256 initialEpoch) external;
-function setFastLaneLengthSlots(uint256 fastLaneLengthSlots) external;
+function setFrameConfig(uint256 secondsPerFrame, uint256 fastLaneSeconds) external;
+function submitReport(uint256 timestamp, bytes32 report, uint256 consensusVersion) external;
+function updateInitialTimestamp(uint256 initialTimestamp) external;
+function setFastLaneSeconds(uint256 fastLaneSeconds) external;
```

###### New Function

```diff
+function getInitialTimestamp() external view returns (uint256);
```

###### Removed Functions

```diff
-// Superseded by getInitialTimestamp():
-function getInitialRefSlot() external view returns (uint256);
-
-// No longer required for frame math under seconds-based frames:
-function getChainConfig() external view returns (
-    uint256 slotsPerEpoch,
-    uint256 secondsPerSlot,
-    uint256 genesisTime
-);
```

`getInitialRefSlot()` is replaced by `getInitialTimestamp()`, and `getChainConfig()` is dropped because frame math no longer depends on chain config. All in-protocol consumers of these methods are updated to the timestamp-based interface in this release.

**Note — external consumers.** These two functions are public view methods that out-of-protocol integrations (block explorers, dashboards, indexers, analytics, third-party contracts) may read. Removing them is a breaking ABI change beyond Lido's own codebase, so before removal it requires dedicated research to enumerate external dependents and an advance announcement so integrators can move to the timestamp-based equivalents. If that research surfaces hard external dependencies that cannot migrate in time, retaining the two methods as thin read-only compatibility shims should be reconsidered rather than removing them outright.

##### Events and Errors

All events and custom errors that reference `refSlot`, `slot`, `epoch`, `epochsPerFrame`, `fastLaneLengthSlots`, or `initialEpoch` must be updated to their timestamp-based equivalents. Off-chain services that parse these events (oracle daemons, indexers, monitoring tools) will require corresponding updates before or at the time of contract upgrading.

##### Off-Chain Oracle Daemon — Resolving `refTimestamp` to a Reference Slot

On-chain the frame is now anchored to a wall-clock `refTimestamp`, but an oracle report is still built against the consensus-layer state at a concrete **slot**. The daemon must therefore resolve the frame's `refTimestamp` to a reference slot before assembling a report:

1. **Read the frame reference.** Fetch `refTimestamp` from `HashConsensus.getCurrentFrameV2()`.
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
| `RefSlotCache.sol`, `VaultHub.sol`, `LazyOracle.sol` | Upgrade required                               | Use the frame reference purely as an opaque `uint48` cache key (no slot arithmetic), so they are functionally unaffected — but the call site must move from the removed `getCurrentFrame()` to `getCurrentFrameV2()`, so they must be updated/redeployed. The cached value simply becomes a timestamp. `ILazyOracle._vaultsDataRefSlot` is a parameter rename.          |
| `OracleReportSanityChecker.sol`                      | Upgrade required (for ZK-oracle compatibility) | Uses `refSlot` to align the ZK-oracle report with the accounting report during a negative rebase. Only needed if the ZK oracle is connected to the protocol; otherwise the change can be skipped.                                                                                                                                                                       |

**`lidofinance/community-staking-module` (CSM):** CSM does **not** depend on `core`'s `HashConsensus`; it vendors its own copies under `src/lib/base-oracle/` and runs a **separate deployed `HashConsensus` instance** for its fee oracle (`src/FeeOracle.sol`). That fork carries the same two problems and must be migrated in lock-step (own PR, own deployment):

| Contract                                    | Change type      | Notes                                                                                                                                                                           |
|---------------------------------------------|------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `HashConsensus.sol` (CSM)                   | Upgrade required | Identical epoch/slot frame math **and** the same unbounded `_reportVariants` storage — so CSM's fee oracle is independently exposed to the same DoS. Both fixes apply here too. |
| `BaseOracle.sol` (CSM)                      | Upgrade required | Same `getChainConfig()` / `getInitialRefSlot()` calls and `ConsensusReport.refSlot` as core's `BaseOracle`.                                                                     |
| `FeeOracle.sol` (deployed as `CSFeeOracle`) | Rename           | Passes `data.refSlot` through only (no slot arithmetic) — easier than `core`'s `AccountingOracle`.                                                                              |
| `TwoPhaseFrameConfigUpdate.sol`             | Upgrade required | Frame-config update utility tied to `setFrameConfig`; updated for the seconds-based config (kept for future use, not removed).                                                  |

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

**Admin cannot act as bot by design.** The admin address has no ability to call `execute()`, and is never the delegatee returned by `getDelegatee()`, so it cannot produce signatures any integrator will accept. This is enforced at the contract level: the admin is kept maximally cold and can only manage the delegatee lifecycle.

**Pull integration (`getDelegatee()`):** Integrator contracts must treat the `DelegationContract` address as the authorized principal — checking both that it is a registered entity and that the recovered signer equals its `getDelegatee()`.

**Push integration (`execute()`):** The delegatee can call any target. Permission grants must be narrowly scoped per contract, enforced by the one-contract-per-bot rule. ETH is forwarded to the target and any change is swept back to the delegatee after the call, so overpayment refunded by the target is recovered; only ETH the target actually consumes is non-recoverable.

**Admin compromise.** An attacker with admin access can assign a delegatee they control, but it cannot act until the `cooldown` elapses — and the `DelegateeAssigned` event (with its `activeFrom`) and `getPendingDelegatee()` expose the pending assignment on-chain for the whole window. This gives the DAO and monitoring a reaction window before the malicious key becomes effective: the legitimate admin can `terminate()`. The attacker also cannot call `execute()` or sign directly (the admin-cannot-execute invariant holds), so they must route actions through an assigned delegatee, and the blast radius is bounded to that single contract's permissions. With a zero cooldown there is no such window.

As a last resort — while the legitimate admin still controls the key — it can `terminate()` the contract. `terminate()` is irreversible by design: once terminated, no action can be taken under the contract's authority by anyone, and an attacker who later gains admin access cannot reverse it. The cost is that a new contract must be deployed and governance must reassign the seat, but this is preferable to leaving an attacker able to act through the contract. This irreversibility must be explicit in operator training material.

**Delegatee compromise.** An attacker with only the delegatee key can act within the contract's protocol permissions but cannot assign a new delegatee, cannot terminate, and cannot perform any admin operation. The admin can instantly revoke the delegatee via `revokeDelegatee()`.

**One contract per bot.** Mandatory. Sharing a delegation contract across roles means a single compromise affects multiple protocol functions. Enforced at the governance permission-grant step.

## Links

- EIP-7782 (Ethereum slot time reduction): https://eips.ethereum.org/EIPS/eip-7782
- EIP-8198 (Quick Slots — runtime-configurable slot duration): https://eips.ethereum.org/EIPS/eip-8198
- Core repository (`lidofinance/core`): https://github.com/lidofinance/core
- Delegation contracts repository (`lidofinance/delegation-execution-authority`): https://github.com/lidofinance/delegation-execution-authority
- Oracle daemon (`lidofinance/lido-oracle`): https://github.com/lidofinance/lido-oracle
- Community Staking Module — forked oracle stack (`lidofinance/community-staking-module`): https://github.com/lidofinance/community-staking-module
- HashConsensus DoS vulnerability issue (`lidofinance/core#1379`): https://github.com/lidofinance/core/issues/1379

---

```my python
                           (o)(o)
                          /     \
                         /       |
                        /   \ .. |
          ________     /    /\__/
  _      /        \   /    /   \\
 / \    /  ____    \_/    /
//\ \  /  /    \         /
V  \ \/  /      \       /
    \___/        \_____/
```
