---
lip: 38
title: "Timeslots in HashConsensus"
status: WIP
author: Raman Siamionau
discussions-to: <Create a new thread on https://research.lido.fi/ and drop the link here>
created: 2026-06-01
---

## Simple Summary

This proposal bundles two related improvements to Lido's `HashConsensus` oracle stack into a single release:

1. **HashConsensus weakness fix** — closes the known denial-of-service vector in the `HashConsensus` contract by bounding the number of stored report variants per frame ([`lidofinance/core#1379`](https://github.com/lidofinance/core/issues/1379)).
2. **Timestamp-based oracle frames** — replaces slot/epoch-based frame logic in `HashConsensus` with wall-clock seconds, so oracle frame lengths stay stable across future Ethereum hardforks allowing dynamic slot duration (including the [EIP-8192](https://eips.ethereum.org/EIPS/eip-8198) quick slots support).

## Abstract

This upgrade abstracts oracle frame logic away from slots and epochs so that future Ethereum consensus-layer timing changes do not disrupt the protocol, and at the same time hardens `HashConsensus` against a denial-of-service vector.

We propose to rewrite the `HashConsensus` contract ([`lidofinance/core`](https://github.com/lidofinance/core)) to express all frame logic in wall-clock seconds (`secondsPerFrame`, `fastLaneSeconds`, `initialTimestamp`, `refTimestamp`) instead of epoch- and slot-based parameters. This decouples oracle frame durations from Ethereum's consensus-layer slot timing and future-proofs oracle operation against changes such as the [EIP-8192](https://eips.ethereum.org/EIPS/eip-8192) quick slots support. The same redeployment also closes a denial-of-service vector in `HashConsensus`: a member can no longer inflate the report-variant array with distinct hashes, because abandoned variant slots are reused, capping the array — and every loop over it — at the committee size, eliminating the unbounded iteration that enables the attack.

Concretely, we propose to deploy an updated `HashConsensus` contract and migrate every contract that consumes its frame reference, in both `lidofinance/core` and the Community Staking Module's separately-deployed fork. Because the two contracts share the same redeployment, the DoS fix and the seconds-based frame rewrite ship together — amortizing the audit, deployment, and operator-coordination effort across both.

## Motivation

### HashConsensus DoS weakness

The current `HashConsensus` contract stores every unique report hash submitted by any oracle member in a per-frame array (`_reportVariants` / `_reportVariantsLength`). Consensus is determined by iterating the full array. Because a single oracle member can submit an unlimited number of distinct hashes, a member can inflate this array, making each subsequent `submitReport` and `_isQuorumReached` call more expensive — the iteration cost grows with every additional report variant stored. This exposes an attack vector in which a single compromised key can block reports from being completely delivered.

The attack works by submitting so many alternative report variants that iterating over them no longer fits within a single transaction's gas limit. At that point report submission can no longer be completed in one transaction and quorum can no longer be reached. The practical consequence is a halt of oracle reporting: for as long as the attack is sustained, `HashConsensus` cannot reach consensus and the protocol cannot process the accounting reports that drive stETH rebases, withdrawal request finalization, and validator exit requests. The DoS therefore degrades core protocol operation, not merely the `HashConsensus` contract in isolation.

### Ethereum slot-time changes (EIP-7782, EIP-8198)

Ethereum's consensus-layer slot timing is in flux. Competing proposals such as [EIP-7782](https://eips.ethereum.org/EIPS/eip-7782) (12 s → 6 s) and [EIP-8198](https://eips.ethereum.org/EIPS/eip-8198) ("Quick Slots", 12 s → 8 s) would reduce slot duration and, consequently, halve or otherwise shorten the duration of all oracle frames currently configured in epochs. Under EIP-7782, for example, performance oracle reporting periods would shrink from roughly 28 days to 14 days, disrupting downstream protocol assumptions.

EIP-8198 goes further by making slot duration a runtime-configurable parameter rather than a hardcoded constant, with the value expected to be tuned iteratively across devnets. Any frame logic anchored to `slotsPerEpoch × secondsPerSlot` is therefore not just fragile across a single hardfork but perpetually exposed to re-tuning. Migrating frame configuration from `epochsPerFrame` to `secondsPerFrame` removes the slot dependency entirely, making frame duration invariant to any present or future slot-timing change.

## Specification

### Overview

The release consists of modifications to the `HashConsensus` contract and related contracts — in both `lidofinance/core` and CSM's separately-deployed fork — to fix the DoS vulnerability and switch to seconds-based frames.

### Rationale

**Why the two changes ship as one release.** The DoS fix and the seconds-based frame rewrite both land in the same redeployed `HashConsensus` contract. Bundling them into a single release amortizes the audit, deployment, and operator-coordination effort, and avoids redeploying `HashConsensus` (and repointing its consumers) twice.

### Technical Specification

---

#### Part 1: HashConsensus — Bounded Report Variant Storage

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

CSM runs a vendored copy of `HashConsensus` with the same `_reportVariants` storage and is independently exposed to this DoS; the same fix is applied there (see Affected Contracts in Part 2).

---

#### Part 2: HashConsensus — Timestamp-Based Frames

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

**Selectors that are deliberately *not* changed.** Two of these keep their exact selectors despite changing argument semantics: `setFrameConfig(uint256,uint256)` (now `secondsPerFrame`/`fastLaneSeconds` instead of `epochsPerFrame`/`fastLaneLengthSlots`) and `submitReport(uint256,bytes32,uint256)` (first argument now a `timestamp` instead of a `slot`). The other two setters *are* renamed — `updateInitialEpoch` → `updateInitialTimestamp` and `setFastLaneLengthSlots` → `setFastLaneSeconds` — so their selectors change.

This is the opposite choice from the view getters above, and it is intentional. The "break the selector loudly" rationale is about protecting *anonymous, out-of-protocol* readers from silently misinterpreting a returned value. It does not apply to these two functions, because both are **state-changing and permissioned/in-protocol only**: `setFrameConfig` is admin-gated (governance), and `submitReport` is restricted to registered committee members. Their callers are a known, coordinated set — the governance tooling and the oracle daemon — that are updated as part of this release, not arbitrary third parties who could be silently broken. Keeping the selectors stable avoids needless churn in that tooling; `submitReport` in particular is the most-simulated function in the system, and a stable selector keeps existing simulation, gas-estimation, and monitoring harnesses working unchanged. The semantics still change, so those in-protocol callers must be updated in lock-step (e.g. the daemon must pass a `refTimestamp` where it previously passed a `refSlot` — see the Off-Chain Oracle Daemon section); the point is only that the risk a selector change guards against is absent here, so the cost of changing it is not worth paying.

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

   The formula above assumes a single, constant slot duration (`genesisTime`/`secondsPerSlot` from the beacon-chain spec), which holds only while the chain has never changed its slot time. A slot-timing change does **not** renumber past slots: slot numbers stay continuous and monotonic, and only the number of seconds each slot spans changes, from the fork onward. The timestamp→slot mapping therefore becomes **piecewise** — slots before the fork are spaced by the old duration, slots after it by the new one, joined at the fork's (slot, timestamp) anchor. EIP-7782 introduces a single such breakpoint (two eras, e.g. 12 s before and 6 s after); EIP-8198 lets the duration be reconfigured repeatedly, i.e. many breakpoints. In either case the daemon must map `refTimestamp` using the slot duration in effect for the era that contains it — walking the schedule of `(anchor slot, anchor timestamp, secondsPerSlot)` segments — rather than dividing by one global constant; the single-constant formula above is just the special case of a chain with no breakpoints yet. The invariant that must hold regardless of the adopted EIP is "resolve to the latest slot at or before `refTimestamp`"; only the arithmetic that maps timestamps to slots changes.

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

| Contract                                             | Change type                                                | Notes                                                                                                                                                                                                                                                                                                                                                                   |
|------------------------------------------------------|------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `HashConsensus.sol`                                  | Upgrade required                                           | The full frame engine: `FrameConfig`/`ConsensusFrame`, all epoch/slot frame math, `_submitReport` deadline and fast-lane comparisons, the seconds-based config setters, and the removal of `getChainConfig()` / renaming of `getInitialRefSlot()`. Also carries the bounded-storage DoS fix (Part 1).                                                                   |
| `BaseOracle.sol`                                     | Upgrade required                                           | `ConsensusReport.refSlot` → `refTimestamp`; the `IReportAsyncProcessor` implementation and `_getCurrentRefSlot()` are opaque pass-through (rename only). Its two consensus calls change: the `getChainConfig()` consistency check is dropped/repointed, and `getInitialRefSlot()` → `getInitialTimestamp()`.                                                            |
| `AccountingOracle.sol`                               | Upgrade required                                           | Interprets the reference as a slot: the `_getSlotTimestamp(slot)` helper computes `GENESIS_TIME + slot * SECONDS_PER_SLOT`, and `timeElapsed = (data.refSlot − prevRefSlot) * SECONDS_PER_SLOT`. With a timestamp reference these conversions must change (the value is already a timestamp / a second-delta), or `onOracleReport` and exit-time math silently corrupt. |
| `ValidatorsExitBusOracle.sol`                        | Rename                                                     | `DataProcessingState.refSlot` + an equality check; opaque pass-through.                                                                                                                                                                                                                                                                                                 |
| `RefSlotCache.sol`, `VaultHub.sol`, `LazyOracle.sol` | Upgrade required                                           | Use the frame reference purely as an opaque `uint48` cache key (no slot arithmetic), so they are functionally unaffected — but the call site must move from the removed `getCurrentFrame()` to `getCurrentFrameV2()`, so they must be updated/redeployed. The cached value simply becomes a timestamp. `ILazyOracle._vaultsDataRefSlot` is a parameter rename.          |
| `OracleReportSanityChecker.sol`                      | Upgrade required (for second-opinion-oracle compatibility) | Uses `refSlot` to align the second-opinion-oracle report with the accounting report during a negative rebase. Only needed if the second-opinion oracle is connected to the protocol; otherwise the change can be skipped.                                                                                                                                               |

**`lidofinance/staking-modules` (CSM):** CSM does **not** depend on `core`'s `HashConsensus`; it vendors its own copies under `src/lib/base-oracle/` and runs a **separate deployed `HashConsensus` instance** for its fee oracle (`src/FeeOracle.sol`). That fork carries the same two problems and must be migrated in lock-step (own PR, own deployment):

| Contract                                    | Change type      | Notes                                                                                                                                                                           |
|---------------------------------------------|------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `HashConsensus.sol` (CSM)                   | Upgrade required | Identical epoch/slot frame math **and** the same unbounded `_reportVariants` storage — so CSM's fee oracle is independently exposed to the same DoS. Both fixes apply here too. |
| `BaseOracle.sol` (CSM)                      | Upgrade required | Same `getChainConfig()` / `getInitialRefSlot()` calls and `ConsensusReport.refSlot` as core's `BaseOracle`.                                                                     |
| `FeeOracle.sol` (deployed as `CSFeeOracle`) | Rename           | Passes `data.refSlot` through only (no slot arithmetic) — easier than `core`'s `AccountingOracle`.                                                                              |
| `TwoPhaseFrameConfigUpdate.sol`             | Upgrade required | Frame-config update utility tied to `setFrameConfig`; updated for the seconds-based config.                                                                                     |

Because CSM's contracts are a separate fork with their own deployment, the `core` changes do not propagate to them automatically. They are in scope of this proposal and must be deployed together with the `core` changes so that CSM's fee oracle receives the DoS fix and the timestamp migration at the same time rather than being left exposed.

**Dual Governance.** Dual Governance references `ValidatorsExitBusOracle` only by address, reading its pause state as a registered sealable (a tiebreaker sealable withdrawal blocker). Since VEBO stays behind its existing proxy (its address is preserved) and its pause mechanism is unaffected by this migration, no change to Dual Governance is required and the sealable wiring continues to work unchanged.

### Deployment Summary

**Redeployed at a new address (standalone, non-upgradeable) — references must be repointed by governance:**

| Contract                        | Repointing required                                                                                         |
|---------------------------------|-------------------------------------------------------------------------------------------------------------|
| `HashConsensus` (core)          | Set as the consensus contract on `AccountingOracle` and `ValidatorsExitBusOracle` (`setConsensusContract`). |
| `HashConsensus` (CSM, vendored) | Set as the consensus contract on CSM's `FeeOracle`.                                                         |
| `OracleReportSanityChecker`     | Only if the second-opinion-oracle path is adopted; update the `LidoLocator` reference.                      |

**Implementation redeployed behind the existing proxy (address preserved):**

| Contract                                    | Notes                                                                                                              |
|---------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| `AccountingOracle`                          | New implementation; proxy upgraded, address unchanged. Carries the `AccountingOracle` semantic-conversion changes. |
| `ValidatorsExitBusOracle`                   | New implementation; proxy upgraded, address unchanged.                                                             |
| CSM `FeeOracle` (deployed as `CSFeeOracle`) | New implementation; proxy upgraded, address unchanged.                                                             |
| `VaultHub`, `LazyOracle`                    | New implementation; proxy upgraded, address unchanged.                                                             |

**Library redeployed and re-linked:**

| Contract                 | Notes                                                                                                                                                                              |
|--------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `RefSlotCache` (library) | Redeployed and re-linked into `VaultHub` / `LazyOracle` as part of their upgrade. The cached value is an opaque `uint48` key with no slot arithmetic; it simply holds a timestamp. |

Note that `HashConsensus` (both core and CSM) is the contract whose **address changes**; any off-chain service, dashboard, or governance config that hardcodes a `HashConsensus` address must be updated alongside the oracle `setConsensusContract` calls.

## Security Considerations

### HashConsensus DoS fix

The bounded storage model eliminates the spam vector entirely: no matter how many distinct hashes a member submits in a frame, only one is retained and evaluated. The cost of `submitReport` is now O(1) in stored variants.

The ability for members to change their submitted hash creates a subtle quorum-formation risk in adversarial conditions: if a member oscillates between two hashes, it could prevent a stable majority. The quorum is currently 5-of-9; a single oscillating member cannot prevent quorum formation if the remaining eight members agree.

## Links

- EIP-7782 (Ethereum slot time reduction): https://eips.ethereum.org/EIPS/eip-7782
- EIP-8198 (Quick Slots — runtime-configurable slot duration): https://eips.ethereum.org/EIPS/eip-8198
- Core repository (`lidofinance/core`): https://github.com/lidofinance/core
- Community Staking Module — forked oracle stack (`lidofinance/staking-modules`): https://github.com/lidofinance/staking-modules
- HashConsensus DoS vulnerability issue (`lidofinance/core#1379`): https://github.com/lidofinance/core/issues/1379
- Affected methods — full inventory of methods called on the changed contracts: https://hackmd.io/@wvZO20ANTOmvV4DSaQp8hQ/B1tPDuxffx

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
