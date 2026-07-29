---
lip: 38
title: "VEBO-7002 — Withdrawal-Based Exits and Active Rebalancing"
status: WIP
author: Raman Siamionau (@F4ever)
discussions-to: <Create a new thread on https://research.lido.fi/ and drop the link here>
created: 2026-06-26
updated: 2026-07-24
---

## Simple Summary

This proposal redesigns the Validators Exit Bus Oracle (VEBO) around EIP-7002 execution-layer withdrawal requests and introduces **Active Rebalancing** — a mechanism that lets the protocol redistribute stake across Node Operators without waiting for organic inflows and outflows.

1. **VEBO-7002** — instead of relying on Node-Operator-supplied pre-signed exit messages and the Ejector as the primary exit tool, VEBO uses EIP-7002 partial withdrawal requests (PWRs) and full withdrawal requests (FWRs) as the primary exit mechanism. This enables precise, partial stake exits from `0x02` validators, reduces unproductive stake time, shortens and sharpens withdrawal finalization, and simplifies the protocol by removing several legacy components.
2. **Active Rebalancing** — a controlled, rate-limited way for the protocol to forcibly move stake toward a target distribution by requesting additional withdrawals from over-target Curated Module v2 (CMv2) operators, reusing the same withdrawal-based flow introduced by VEBO-7002.

## Abstract

We propose to rework the Validators Exit Bus Oracle ("VEBO-7002") across its on-chain and off-chain parts.

**On-chain**, we add a **FIFO queue** to the VEBO that stores withdrawal request intentions (partial and full) submitted by the Oracle report, together with a **permissionless handle** that anyone can call to execute queued intentions one by one, turning each into an actual EIP-7002 withdrawal request submitted to the predeploy. A `setPartialWithdrawals` switch lets the protocol turn partial withdrawals off, reverting to FWR-only operation (that could be fulfilled through a Validator Ejector) under extreme EIP-7002 fee conditions.

**Off-chain**, we rework the exit-ordering logic so it selects not only which validators to exit but also the withdrawal amount per validator. This adds support for **partial withdrawal requests** — preferring PWRs against `0x02` validators and falling back to FWRs where PWRs are not applicable — and for **Active Rebalancing**: additional, rate-limited exits against CMv2 operators whose `currentStake` exceeds their `targetStake`, moving the module toward its target distribution.

This redesign also removes components made redundant by the new flow: late-exit penalties and proofs to trigger full withdrawal requests.

## Motivation

### Limitations of the current VEBO

The current VEBO depends on Node Operators uploading pre-signed validator exit messages and the Ejector software to broadcast those exits. This design has structural limitations:

- **Full exits only.** The flow exclusively supports full validator exits.
- **Limited upload capacity.** Only a fraction of pre-signed exit messages can be uploaded per Node Operator, to bound the blast radius if an operator's signing infrastructure is compromised. This caps how aggressively exits can be requested and what validators can be exited using Ejector software.
- **Operational overhead.** Every Node Operator must run and secure dedicated Ejector software, and the protocol must verify that they do.
- **Coarse exit optimization.** Because only specific pre-signed keys can be requested, optimizing exit order (e.g. to minimize sweep delay or to target precise amounts of stake) is challenging.

### Why withdrawal-based exits

EIP-7002 lets the protocol trigger withdrawals from the execution layer using validators' `0x01` and `0x02` withdrawal credentials. Building VEBO around PWRs and FWRs:

- Allows **precise stake exit** from `0x02` validators in CMv2 using partial withdrawals, leaving the validator active.
- **Eliminates unproductive stake time** — ETH keeps earning rewards until the moment a withdrawal is actually processed, instead of sitting idle behind a queued full exit and waiting for the sweep to reach the exited validator.
- Enables **full exits** via EIP-7002 without unfolding a VEBO report.

### Why active rebalancing

Today, stake redistribution across Node Operators and modules happens only through organic inflows (new deposits) and outflows (withdrawal demand). When a module or an individual operator drifts above its target stake, there is no protocol-level lever to correct it on a controlled schedule. For CMv2 we need a mechanism that can deliberately move the module toward its target distribution while staying compatible with the regular withdrawal and deposit flow. The withdrawal-based VEBO-7002 flow makes such a mechanism natural to express: rebalancing is simply additional, rate-limited exit demand against over-target operators.

## Specification

### Overview

1. **Off-chain Oracle computes exit demand in three sequential phases.**
    - Phase 1 — cover regular **withdrawal-queue demand**.
    - Phase 2 — issue **forced validator exits**.
    - Phase 3 — add **active rebalancing** within CMv2. Phase 3 is optional and can be switched off (leaving only Phases 1–2).
2. **The Oracle reports the ordered withdrawal request intentions on-chain, where they are appended to a single FIFO queue in the VEBO.**
3. **Anyone can execute queued intentions.** A permissionless handle pops them one by one and submits each as an actual partial (PWR) or full (FWR) withdrawal request to the EIP-7002 predeploy, paying the per-request fee.

A **`setPartialWithdrawals(bool)` switch** can turn partial withdrawals off, making VEBO emit FWRs only; `ExitRequested` events are emitted for all FWRs so a simplified Ejector can fulfill them via voluntary exits without EIP-7002 fees.

### Rationale

#### Single FIFO queue

Withdrawal request intentions (both PWRs and FWRs) are stored in a single FIFO queue and processed in submission order, each becoming an actual EIP-7002 withdrawal request only once executed (see [Flow 2](#flow-2--execute-withdrawal-requests)). We chose this model because:

- **Simplicity.** A single queue is simple to iterate on-chain, with no need to manage concurrency.
- **Permissionless safety.** VEBO fully controls ordering and concurrency, which protects against permissionless interference and mitigates [FWR-replay DDoS](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-30.md#61-attack-vector-vulnerability-from-uncontrolled-creation-of-twrs).

The trade-offs are:

- **No skipping.** Intentions must be executed in order; an intention that is no longer valid still must be executed to process order further. We deliberately do not add a skipping mechanism: EIP-7002 requests are cheap, so executing a now-invalid intention wastes only a negligible fee, which is not worth the extra on-chain complexity.
- **No execution-order choice.** Because order is fixed, there is no room for execution-time optimization (e.g. minimizing FWR sweep delay). This is acceptable because we expect to use mostly PWRs for CMv2, so no sweep optimization is needed. Instances of CSM (0x01 and 0x02) do not support PWRs due to the FIFO stake distribution mechanism and the requirement for the key to be fully bonded.
- **PWR/FWR concurrency.** PWRs and FWRs share one queue and contend for the same EIP-7002 throughput; this concurrency will be mitigated within VEBO (in how it orders and batches requests). Concurrency matters at the CL level for a given validator: a PWR cannot be executed while the validator is already exiting (an FWR/voluntary exit is pending), and an FWR cannot be executed while the validator still has an unprocessed PWR pending — either combination causes the later request to be rejected on the CL and re-requested later (see [Shared iterator mechanics](#shared-iterator-mechanics) and [Security Considerations](#security-considerations)).

#### Deprecating the Ejector as the primary tool

The Ejector cannot process PWRs, the new system expects a low FWR volume, and historical mainnet data shows that EIP-7002 fee spikes are short-lived and resolve before VEBO could react. With a declining validator count and no planned reduction in EIP-7002 throughput, PWR pricing should stay favorable. The Ejector is therefore demoted to a **fallback** role, used only when partial withdrawals are turned off via `setPartialWithdrawals(bool)` (FWR-only operation).

### Technical Specification

The design splits cleanly along the oracle boundary:

- **Off-chain Oracle** owns all *ordering* logic — which module, which Node Operator, and which validator keys (with what amounts) should exit next, and all active-rebalancing decision-making. It produces the per-report list of withdrawal request intentions.
- **On-chain** protocol side — a thin set of primitives that:
    - ingests the Oracle report and appends its withdrawal request intentions to a single FIFO queue;
    - exposes a permissionless handle to execute queued intentions, turning each into an actual withdrawal request against the EIP-7002 contract;
    - enforces the exit limits (TWG rate limit on extracted balance);
    - enforces the configurable rebalancing bounds (rate limit, thresholds);
    - holds partial withdrawal requests off/on switch (full withdrawal requests only mode);

#### Off-chain Oracle: ordering and rebalancing logic

Per report, the Oracle produces the ordered list of withdrawal request intentions. It runs the exit-order iterator ([`exit_order_iterator.py`](https://github.com/lidofinance/lido-oracle/blob/f352b8552b717ebf8ecaf3bd0f5cf6785909ec17/src/services/exit_order_iterator.py#L106)) — generalized from "pick the next validators to fully exit" to "pick the next validators and the amount (PWR/FWR) to withdraw" — across **three sequential phases**, each with its own demand source.

##### Shared iterator mechanics

All three phases reuse the same machinery. Two governance-tunable parameters drive it, both stored in the `OracleDaemonConfig` smart contract so they can be changed without an Oracle release:

- **`EXIT_ITERATION_CHUNK`** (default **32 ETH**) — the fixed unit of demand the iterator allocates per step.
- **`MIN_PARTIAL_WITHDRAWAL`** (default **8 ETH**) — the minimum withdrawable balance a validator must have above the min-effective-balance floor to serve a PWR, and the lower bound of any single PWR.

**Iteration granularity.** The iterator walks exit demand in fixed `EXIT_ITERATION_CHUNK` (32 ETH) steps: each iteration allocates one chunk of demand to the top-ranked operator (and one of its validators), then re-ranks. This keeps the ordering balanced across operators and maps naturally onto withdrawal sizing — a chunk becomes a PWR of up to `EXIT_ITERATION_CHUNK` from a `0x02` validator, or an FWR.

**PWR vs. FWR threshold.** A validator can only serve a PWR if it has enough balance above the effective-balance floor to give up. If **no** validator has more than `MIN_PARTIAL_WITHDRAWAL` (8 ETH) of withdrawable balance above the floor, a **full withdrawal request** is used against the top-ranked validator instead. Otherwise a partial withdrawal is taken from that validator, bounded to the `[MIN_PARTIAL_WITHDRAWAL, EXIT_ITERATION_CHUNK]` (8–32 ETH) range per validator per iteration step. Across iteration steps within the same report, the ranking criterion can allocate multiple chunks to the same validator. These are unified into a single PWR intention for that validator.

**No FWR over an unprocessed PWR.** The Oracle MUST NOT issue an FWR for a validator that still has an unprocessed PWR (in the FIFO queue or not yet observed as processed on the CL). The consensus layer rejects a full withdrawal request while a partial one for the same validator is pending, so such an FWR would be wasted and need re-requesting. The iterator therefore excludes validators with in-flight PWRs from FWR selection and only escalates a validator from PWR to FWR in a later report, once the PWR has cleared. See [Security Considerations](#security-considerations).

**Per-report exit limit.** The iterator stops accumulating requests once the report's cumulative exited balance (the `max_current_exit_balance` accumulator) reaches the per-report exit limit — the sanity checker's `maxBalanceExitRequestedPerReportInEth` parameter introduced in [LIP-35](lip-35.md). This limit is shared across all three phases. Requests are weighed with the **same formula as the on-chain TWG limiter** — `amountGwei` for a PWR, max effective balance by WC type (32/2048 ETH) for an FWR; see [rate limiter invariants](#triggerablewithdrawalsgateway-twg).

##### Phase 1 — Cover withdrawal-queue demand

Phase 1 covers the current **withdrawal-queue (WQ) demand**, issuing PWR-first instead of full exits if it is possible. It walks that demand in `EXIT_ITERATION_CHUNK` steps, ranking validator at each step by, in order:

1. **Forced-exit deviation** — operators over a force-exit limit first, by how far `currentValidators − targetValidators` exceeds the threshold.
2. **Soft-exit deviation** — same idea for soft-exit limits, after forced exits.
3. **Module share-rate deviation** — modules above their `priority_exit_share_threshold` (relative to total protocol balance) get elevated priority.
4. **Target-stake deviation** — operators furthest **above** their target stake exit first, where `targetStake = moduleStake × operatorWeight / moduleTotalWeight`. Operator weights come from [LIP-35](lip-35.md): read on-chain via `getOperatorWeights` for CMv2, aggregated via the Meta Registry for CMv1, and a uniform baseline weight elsewhere.
5. **Already-requested validators** — validators that already have PWR demand allocated to them **within this report** rank first, but only while their remaining balance stays above `MIN_ACTIVATION_BALANCE`. This concentrates exited balance into as few validators as possible, minimizing the number of distinct PWRs by taking more balance out of one validator before moving to the next.
6. **Biggest validator first** — among the operator's validators, prefer the one with the **largest balance**. This maximizes the balance a single PWR can extract while leaving the validator active, favoring partial over full withdrawals.
7. **Lowest validator index** — within an operator, ascending validator index.

##### Phase 2 — Forced validator exits

After WQ demand is covered, the iterator satisfies any **forced-exit obligations**.

1. **Forced-exit deviation** — operators over a force-exit limit first, by how far `currentValidators − targetValidators` exceeds the threshold.
2. **Lowest validator index** — within an operator, ascending validator index.

##### Phase 3 — Active rebalancing

This phase applies **only to the CMv2 staking module** and adds exit demand against operators drifting above their target stake, pushing the module back toward its target distribution.

**Data model.** Phase 3 (and Phase 1 criterion 4) uses the following definitions:

- **`operatorWeight` / `moduleTotalWeight`** — CMv2 operator weights introduced in [LIP-35](lip-35.md), read on-chain via `getOperatorWeights`. `moduleTotalWeight` is the sum of weights over operators participating in the calculation (see the grace period below).
- **`targetStake`** = `moduleStake × operatorWeight / moduleTotalWeight`, where `moduleStake` is the total balance of CMv2 validators at the report's refSlot, **excluding** operators inside the grace period.
- **`currentStake`** — the sum of the operator's active validator balances at refSlot, **subtract in-flight PWR amounts**: withdrawal request intentions already queued in the VEBO FIFO plus partial withdrawals pending on the CL (`pending_partial_withdrawals`).
- **Request types.** All CMv2 validators use `0x02` credentials; rebalancing prefers PWRs but **may issue an FWR** where a partial withdrawal cannot serve the demand.

**Configuration.** All Phase 3 parameters live as keys in the existing **`OracleDaemonConfig`** key-value store.

| `OracleDaemonConfig` key                | Type        | Meaning                                                                                                                             |
|-----------------------------------------|-------------|-------------------------------------------------------------------------------------------------------------------------------------|
| `ACTIVE_REBALANCING_ENABLED`            | bool        | Global on/off for Phase 3.                                                                                                          |
| `ACTIVE_REBALANCING_RATE_LIMIT`         | uint (ETH)  | Max stake exitable for rebalancing per VEBO report.                                                                                 |
| `ACTIVE_REBALANCING_OPERATOR_EXCESS_BP` | uint (bp)   | Operator-level trigger: `currentStake` over `targetStake` as a share of `targetStake`.                                              |
| `ACTIVE_REBALANCING_MODULE_EXCESS_BP`   | uint (bp)   | Module-level trigger: the sum of over-target stake across all operators as a share of total module stake.                           |
| `ACTIVE_REBALANCING_GRACE_PERIOD`       | uint (days) | Days since first key activation before an operator counts toward `moduleStake`/`moduleTotalWeight` and participates in rebalancing. |

**Trigger.** Phase 3 requires `ACTIVE_REBALANCING_ENABLED` to be set, and runs when **one or more** of the following conditions holds:

1. **Operator-level** — at least one operator's `currentStake` exceeds `targetStake × (1 + ACTIVE_REBALANCING_OPERATOR_EXCESS_BP)`;
2. **Module-level** — the sum of stake above target stake across all operators within the module exceeds a defined percentage of the total module stake.
   `Σ max(0, currentStake − targetStake) > moduleStake × ACTIVE_REBALANCING_MODULE_EXCESS_BP`
   (summed over operators outside the grace period).

While neither condition holds, Phase 3 stays dormant, so small transient drifts don't generate exit churn.

A **new-operator grace period** applies: an operator whose first key activated less than `ACTIVE_REBALANCING_GRACE_PERIOD` days ago is **excluded from the rebalancing calculation entirely** — its balance does not count toward `moduleStake`, its weight does not count toward `moduleTotalWeight`, and it is neither ranked nor targeted by rebalancing exits. Grace-period operators remain eligible to receive deposits via the normal deposit allocation.

**Bounds.** The total rebalancing stake added to the report is capped at `ACTIVE_REBALANCING_RATE_LIMIT`.

**Ordering.** When it runs, the iterator ranks over-target CMv2 operators by, in order:

1. **Target-stake deviation** — node operators rank by the deviation `currentStake − targetStake`; the operator furthest **above** target ranks first.
2. **Already-requested validators** — validators with PWR demand already allocated **within this report** rank first, concentrating exited balance into as few validators as possible.
3. **Biggest validator first** — prefer the largest-balance validator so exits stay partial and keep the validator active.
4. **Lowest validator index** — within an operator, ascending validator index.

#### On-chain Oracle

On-chain, the protocol takes the Oracle's ordered withdrawal request intentions and turns them into EIP-7002 withdrawals. This happens in **two flows**:

- **Report submission** — the Oracle submits a packed report; each intention is decoded and appended to a single FIFO queue.
- **Execute withdrawal requests** — a permissionless handle pops queued intentions and pushes each down the request path to the EIP-7002 predeploy, where it becomes an actual withdrawal request:

```
VEBO-7002 (FIFO queue)  ->  TriggerableWithdrawalsGateway (TWG)  ->  WithdrawalVault  ->  EIP-7002 predeploy
  report submission                 rate limits, fee refund             encoding          withdrawal request
```

##### Flow 1 — Report submission

The Oracle calls `submitReportData` with a packed report. The report keeps the existing encoding — a `dataFormat` selector plus a `data` blob of fixed-width records packed contiguously — but introduces a **new `dataFormat` version** whose record carries the withdrawal **amount**. The current record (`DATA_FORMAT_LIST_WITH_KEY_INDEX = 2`, introduced in [LIP-35](lip-35.md)) is 72 bytes; the new record appends an 8-byte `amount`:

```
/// MSB <--------------------------------------------------------------------------------------------------- LSB
/// |  3 bytes   |   5 bytes    |    8 bytes     |   8 bytes  |      48 bytes       |      8 bytes          |
/// |  moduleId  |  nodeOpId    | validatorIndex |  keyIndex  |   validatorPubkey   |  amount (gwei) *new*  |
```

`amount` (uint64, gwei) — **new**; `0` = full withdrawal (FWR), `> 0` = partial withdrawal (PWR).

`keyIndex` is **retained** from format 2: it lets the contract check against the staking module's key registry that the reported pubkey is a real registered key and determine its withdrawal-credentials type (`0x01`/`0x02`) on-chain — this is what backs the `0x02` validation of PWRs at submission.

On submission the contract decodes each record into an `ExitRequest` struct — a withdrawal request intention — and appends it to the FIFO queue.

When `setPartialWithdrawals` is off, the report is still in the new format but every record MUST carry `amount == 0` (FWR-only); if any record carries a non-zero amount, the protocol reverts the entire `submitReportData` call.

##### Flow 2 — Execute withdrawal requests

Anyone calls the permissionless `processExitRequests(count)`, forwarding the per-request EIP-7002 fee. It pops the next `count` intentions from the FIFO in order and pushes each down the path above: **TWG** applies the exit limits and refunds excess fee, then the **WithdrawalVault** encodes an actual withdrawal request and submits it to the EIP-7002 contract.

**Cost model.** EIP-7002 fees for all VEBO-path requests are paid by the `processExitRequests` caller (excess is refunded via the TWG). This release **deliberately removes** the previous charge-back of forced-exit fees to Node Operator bond (CSM's `_fulfillExitObligations`, driven by the `onValidatorExitTriggered` notification): with the notification hooks gone, forced-exit fees are no longer billed to the misbehaving operator and are borne by the executor instead. The trade this buys: forced exits no longer depend on operator compliance at all — the protocol withdraws the stake itself via EIP-7002.

```solidity
interface IValidatorsExitBusOracle7002 {
    /// A single queued withdrawal request intention — not yet an EIP-7002 request.
    /// amountGwei == 0  -> full withdrawal (FWR) once executed
    /// amountGwei  > 0  -> partial withdrawal (PWR) once executed
    struct ExitRequest {
        uint32  moduleId;       // matches the 3-byte moduleId field width in the packed report record
        uint64  nodeOperatorId; // matches the 5-byte nodeOpId field width in the packed report record
        uint64  amountGwei;     // moduleId + nodeOperatorId + amountGwei pack into a single storage slot
        bytes   pubkey;         // dynamic type placed last so it doesn't break packing of the fields above
    }

    /// Flow 1: Oracle submits the ordered, packed report; intentions are appended to the FIFO queue.
    function submitReportData(bytes calldata report, uint256 contractVersion) external;

    /// Flow 2: permissionless — pop the next `count` queued intentions and execute each as an
    /// actual withdrawal request down the EIP-7002 path. Caller forwards the per-request fee.
    function processExitRequests(uint256 count) external payable;

    /// Number of intentions waiting in the FIFO queue.
    function unprocessedRequestsCount() external view returns (uint256);

    /// Returns the queued intention at FIFO offset `index` from the head of the queue
    /// (`index == 0` is the next intention `processExitRequests` will execute), letting
    /// permissionless executors and monitoring tools inspect the queue before calling it.
    function getExitRequest(uint256 index) external view returns (ExitRequest memory);

    /// FWR-only fallback switch (privileged).
    function setPartialWithdrawals(bool enabled) external;
    function partialWithdrawalsEnabled() external view returns (bool);

    /// Emitted for every queued intention (validator ejector can use this event
    /// to fulfill exits when partial withdrawals are turned off).
    /// `validatorIndex` is carried by the packed report record and is included
    /// so the ejector can keep matching pre-signed exit messages by index.
    event ExitRequested(
        uint32  indexed moduleId,
        uint64  indexed nodeOperatorId,
        uint64  validatorIndex,
        bytes   pubkey,
        uint64  amountGwei
    );
}
```

##### Scope — contracts to change

| Contract                                | Change                                                                                                                                                                                                                                                                                           |
|-----------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `ValidatorsExitBusOracle` (→ VEBO-7002) | New report `dataFormat` with `amount`; FIFO queue storage; permissionless `processExitRequests` handle; `setPartialWithdrawals` switch.                                                                                                                                                          |
| `TriggerableWithdrawalsGateway` (TWG)   | Add `triggerWithdrawals` (partial and full, used by VEBO-7002); **retain `triggerFullWithdrawals`** unchanged for existing consumers (CSM `Ejector`); drop `_notifyStakingModules`; rate limiter changed to bound **extracted balance** instead of request count, shared by both entry points.   |
| `StakingRouter`                         | Remove `onValidatorExitTriggered` (no longer called by the TWG) and `reportValidatorExitDelay` (late-exit-penalty accounting); revoke the two orphaned role grants.                                                                                                                              |
| Staking Modules (CSM)                   | Remove the module-side late-exit logic and penalty accounting: the `onValidatorExitTriggered` implementations, exit-delay-penalty bookkeeping fed by `reportValidatorExitDelay`, and (CSM) the `_fulfillExitObligations` bond charge-back; see [Staking Modules](#staking-modules-nor-sdvt-csm). |
| `ValidatorExitDelayVerifier`            | **Removed entirely** — the exit-delay proof contract is obsolete once late-exit penalties are gone.                                                                                                                                                                                              |
| `WithdrawalVault`                       | Already supports partial withdrawals — implementation upgrade behind the existing proxy, with the new TWG as authorized caller (immutable constructor parameter). The proxy address is the protocol's withdrawal-credentials target and never changes.                                           |
| `LidoLocator`                           | Register the new TWG address, and remove the `ValidatorExitDelayVerifier` entry.                                                                                                                                                                                                                 |
| `OracleDaemonConfig`                    | Add the iterator params (`EXIT_ITERATION_CHUNK`, `MIN_PARTIAL_WITHDRAWAL`) and the active-rebalancing keys.                                                                                                                                                                                      |
| EasyTrack factories                     | **Removed**: the VEBO-bypass exit-request factories are removed and their `EVMScriptExecutor` roles on the VEBO revoked; see [EasyTrack factories](#easytrack-factories).                                                                                                                        |

##### `ValidatorsExitBusOracle` → VEBO-7002

The FIFO queue, the report `dataFormat` with `amount`, and the `processExitRequests` / `submitReportData` / `setPartialWithdrawals` surface are described in Flows 1–2 above.

**Cutover.** Legacy report hashes with exit data not yet delivered at the moment of the upgrade are **abandoned** — the new format cannot deliver them, and no legacy delivery path is retained. This is accepted: the underlying exit demand re-emerges organically from validator balances and is re-covered by subsequent VEBO-7002 reports.

The `setPartialWithdrawals` toggle is the FWR-only fallback: when **off**, the contract accepts FWRs only. `ExitRequested` still fires for every FWR so a Validator Ejector can fulfill them via voluntary exits without EIP-7002 fees. This is the safe mode under sustained extreme EIP-7002 fees.

**Switch semantics:**

- **Role.** `setPartialWithdrawals` is gated by a dedicated role held by **DAO governance in both directions** — turning the mode off and back on each require a vote.
- **Validity at submission.** Reports are validated against the **live** switch state in `submitReportData`: while the switch is off, any record with `amount > 0` reverts the whole report. A report built and quorum-agreed before a mid-frame flip therefore reverts; that frame is skipped and the next report is built FWR-only.
- **Already-queued PWRs.** PWRs sitting in the FIFO when the switch turns off **remain in the queue and must still be executed** — there is no dismissal. At the same time, the off-chain VEBO **must not count queued-but-unexecuted PWRs** as covering WQ demand when computing subsequent reports, because under the fee conditions that motivate the switch there is no guarantee of when they execute.

On `submitReportData`, VEBO-7002 **MUST validate that every record with `amount > 0` (a PWR) targets a `0x02` (compounding-credentials) validator**.

##### `TriggerableWithdrawalsGateway` (TWG)

**Unified request path.** The TWG becomes the single entry point for **all** withdrawal requests — partial and full. A new `triggerWithdrawals` method carrying a per-request `amount` (`0` = FWR, `> 0` = PWR) is added for VEBO-7002. The existing full-exit-only **`triggerFullWithdrawals` is retained with an unchanged external signature**, so current consumers — the CSM (`voluntaryEject` and strikes-based ejection via `ValidatorStrikes._ejectByStrikes`), keep working across the upgrade without a redeploy or role migration:

```solidity
interface ITriggerableWithdrawalsGateway {
    // Field widths match `ExitRequest` and the packed report record for consistency.
    // Unlike `ExitRequest`, this struct is only ever a `calldata` parameter, never
    // storage — Solidity does not pack calldata/memory struct fields (each is ABI-encoded
    // to its own 32-byte word regardless of declared width), so narrower types here save
    // no gas; they exist purely to keep the same value ranges across both interfaces.
    struct WithdrawalRequest {
        uint32  moduleId;       // matches the 3-byte moduleId field width in the packed report record
        uint64  nodeOperatorId; // matches the 5-byte nodeOpId field width in the packed report record
        uint64  amount;         // gwei; 0 = full withdrawal (FWR), > 0 = partial withdrawal (PWR)
        uint8   wcType;         // validator withdrawal-credentials type: 0x01 or 0x02
        bytes   pubkey;         // dynamic type placed last, consistent with ExitRequest
    }

    /// New single path for partial and full withdrawal requests (used by VEBO-7002).
    /// Applies the extracted-balance rate limit, forwards each request to the
    /// WithdrawalVault, and refunds any excess EIP-7002 fee to `refundRecipient`.
    function triggerWithdrawals(
        WithdrawalRequest[] calldata requests,
        address refundRecipient
    ) external payable;

    /// Retained: full-withdrawal-only entry point with the existing external
    /// signature, kept for backward compatibility with deployed consumers
    /// (CSM Ejector). Internally routed through the same request path and the
    /// same extracted-balance rate limit as `triggerWithdrawals` (each FWR
    /// counted at the validator's max effective balance).
    function triggerFullWithdrawals(/* unchanged signature */) external payable;
}
```

Both entry points draw from the **same** extracted-balance rate limit, so the retained method cannot be used to bypass the per-frame limits.

As part of this it **drops `_notifyStakingModules`**: the current TWG calls back into staking modules (via the Staking Router's `onValidatorExitTriggered`) whenever it triggers an exit, which existed for the late-exit-penalty accounting that this release removes.

Note that this is a **behavioral change for the retained `triggerFullWithdrawals`** as well: its external signature is unchanged, but it no longer notifies staking modules on exit. Any module-side logic hanging off that notification (e.g. CSM's EL-fee charge-back against operator bond) stops firing. This is a deliberate removal — see the [cost model](#flow-2--execute-withdrawal-requests) in Flow 2.

**Rate limiter.** The limiter is **balance-denominated** — it caps the total extracted balance per frame. Three invariants govern it:

1. **One weight formula.** A request weighs `amountGwei` when `amountGwei > 0` (PWR), otherwise the validator's max effective balance by WC type — 32 ETH for `0x01`, 2048 ETH for `0x02` (FWR). The off-chain iterator's per-report exit limit, the submission-time sanity check, and the TWG limiter MUST all apply **exactly this formula** — any divergence would make a quorum-agreed report revert deterministically at `submitReportData`, frame after frame. Max-EB (rather than the validator's actual balance) is used for FWRs because it is verifiable on-chain from `wcType` alone; the cost is conservative budget consumption when FWRs dominate.
2. **Frame max limit floor.** The configured limit MUST be ≥ 2048 ETH, so the head-of-queue request — whatever its weight — always fits in a frame and the FIFO can never jam permanently on a single max-weight FWR.
3. **Prefix-processing on exhaustion.** When a `processExitRequests(count)` batch would exceed the remaining frame budget, the TWG processes the maximal prefix that fits and stops; it does not revert the batch and never skips over the blocking request.

##### `StakingRouter`

Two exit-related hooks are removed, both tied to the late-exit-penalty accounting this release drops:

- **`onValidatorExitTriggered`** — the TWG no longer notifies modules on exit, so this callback (and its module-interface counterpart) is deleted.
- **`reportValidatorExitDelay`** — the entry point through which the `ValidatorExitDelayVerifier` reported proven exit delays for penalty accounting; with penalties gone, nothing calls it.

Removing the hooks orphans their access-control grants, so the upgrade vote MUST also **revoke the corresponding roles**: the TWG's grant to call `onValidatorExitTriggered` and the `ValidatorExitDelayVerifier`'s grant to call `reportValidatorExitDelay`.

##### Staking Modules (NOR, SDVT, CSM)

Once the TWG stops calling `onValidatorExitTriggered` and `StakingRouter.reportValidatorExitDelay` is removed, the module-side counterparts of those hooks become unreachable. This release removes them rather than leaving them as dead code:

- **`onValidatorExitTriggered` implementations** (NOR, SDVT, CSM) — deleted from each module; nothing calls them once the TWG no longer notifies on exit.
- **Late-exit-penalty accounting** — the bookkeeping fed by `reportValidatorExitDelay` (tracking proven exit delays and applying the corresponding penalty) is removed from the modules that implement it.
- **CSM's `_fulfillExitObligations` bond charge-back** — the EL-fee-on-forced-exit charge against operator bond, already dead in practice once the TWG stops notifying (see the [cost model](#flow-2--execute-withdrawal-requests) in Flow 2), is removed from CSM.

##### `ValidatorExitDelayVerifier`

**Removed entirely.** This contract proved on-chain that a validator's exit was delayed beyond the allowed window, feeding the late-exit penalty via `StakingRouter.reportValidatorExitDelay`. Once late-exit penalties are removed there is nothing to prove or report, so the contract and its `LidoLocator` registration are deleted.

##### `WithdrawalVault`

The WithdrawalVault already supports EIP-7002 partial withdrawals (variable `amount`), so it needs no functional change. Its proxy address is the target of the protocol's withdrawal credentials and therefore **cannot move**; the change is an **implementation upgrade behind the existing proxy** — a new implementation constructed with the new TWG as its authorized caller (immutable constructor parameter). The `LidoLocator` entry for the vault is unchanged.

##### `LidoLocator`

Register the new TWG implementation address so the rest of the protocol resolves them after the upgrade, and **remove the `ValidatorExitDelayVerifier` entry**.

##### `OracleDaemonConfig`

This release **adds the following keys** to the contract (iterator params, see [Shared iterator mechanics](#shared-iterator-mechanics); active-rebalancing keys, see [Phase 3 — Active rebalancing](#phase-3--active-rebalancing)):

| Key                                     | Type        | Default | Meaning                                                                                     |
|-----------------------------------------|-------------|---------|---------------------------------------------------------------------------------------------|
| `EXIT_ITERATION_CHUNK`                  | uint (ETH)  | 32      | Fixed unit of demand the iterator allocates per step.                                       |
| `MIN_PARTIAL_WITHDRAWAL`                | uint (ETH)  | 8       | Min withdrawable balance above the floor for a validator to serve a PWR; PWR lower bound.   |
| `ACTIVE_REBALANCING_ENABLED`            | bool        | false   | Global on/off for Phase 3.                                                                  |
| `ACTIVE_REBALANCING_RATE_LIMIT`         | uint (ETH)  | —       | Max stake exitable for rebalancing per VEBO report.                                         |
| `ACTIVE_REBALANCING_OPERATOR_EXCESS_BP` | uint (bp)   | —       | Operator-level trigger: `currentStake` over `targetStake` as a share of `targetStake`.      |
| `ACTIVE_REBALANCING_MODULE_EXCESS_BP`   | uint (bp)   | —       | Module-level trigger: sum of over-target stake across operators as a share of module stake. |
| `ACTIVE_REBALANCING_GRACE_PERIOD`       | uint (days) | —       | Days since first key activation before an operator counts in rebalancing calculations.      |

#### EasyTrack factories

The EasyTrack factories that interact with VEBO (e.g. the VEBO-bypass exit flow, which lets a committee submit exit requests directly without waiting on Oracle quorum) are **removed** in this release.

## Security Considerations

- **Permissionless interference / DDoS.** The single FIFO queue keeps concurrency entirely under VEBO control and avoids FWR-replay attack surface.
- **PWR/FWR concurrency on one validator.** The consensus layer rejects a full withdrawal request while the validator still has an unprocessed partial withdrawal request. So if VEBO issues a PWR and then an FWR for the same validator, the FWR is rejected by the CL and must be re-requested later. To avoid this waste, **VEBO MUST NOT issue an FWR for a validator that still has an unprocessed PWR** — it should wait until the PWR is processed (or only ever escalate a validator from PWR to FWR across separate reports once the PWR has cleared).
- **Balance-denominated TWG limit.** Changing the TWG rate limiter to bound extracted balance (instead of request count) caps how much stake can leave per frame, so a stream of large partial withdrawals cannot exceed the intended ETH-denominated limit. Note this makes FWRs the expensive request against the limit: an FWR consumes the validator's full max effective balance (up to 2048 ETH for `0x02`) from the frame budget, so a burst of FWRs can exhaust the TWG limit far faster than PWRs. The frame-budget floor (≥ 2048 ETH) and prefix-processing guarantee the queue still advances at least one request per frame in the worst case.
- **EIP-7002 fee risk.** Organic-usage modeling shows fee escalation is a minimal credible risk at realistic batch sizes; turning `setPartialWithdrawals` off (FWR-only operation) provides a safe path under sustained extreme fees. [EIP-7002 Partial Withdrawal Economics](https://hackmd.io/G82dyK7lQZWmO9vYNJ7fjA?view#EIP-7002-Fee-Dynamics)
- **No cost enforcement in FWR-only mode.** With the bond charge-back removed, the FWR-only fallback (exits fulfilled by the Validator Ejector via fee-less voluntary exits) carries no economic penalty for an operator that ignores an exit request — precisely the mode where the protocol prefers not to pay EIP-7002 fees itself. Accepted: the fallback is expected to be short-lived (fee spikes historically resolve quickly), and the protocol can still force any individual exit through EIP-7002 by paying the spiked fee if an operator stalls.
- **Migration safety.** The explicit enable/disable control ensures active rebalancing cannot interfere with migration sequencing.
- **Grace period.** Excluding new operators from the rebalancing calculation entirely (no contribution to `moduleStake`/`moduleTotalWeight`, not ranked, not targeted) prevents newly onboarded operators from inducing immediate rebalancing pressure. Residual risk: a drained operator can re-register under a fresh identity and re-receive stake through normal deposit allocation while immune to rebalancing for the grace window; this is bounded by CMv2 bond requirements and `ACTIVE_REBALANCING_RATE_LIMIT`.

## Failure Modes

- **EIP-7002 fee spike** — mitigation: turn `setPartialWithdrawals` off for FWR-only operation; monitor execution-layer withdrawal-request fees. When `setPartialWithdrawals` is off, the VEBO demand-prediction phase (covering WQ requests) **must not count any queued-but-unexecuted PWRs** as covering demand — those PWRs may not be executed, so the demand they would have covered is re-covered by FWRs in the report. Queued PWRs still execute eventually (no dismissal), so the same demand can be covered twice; this over-withdrawal is accepted and returns to the buffer (see [switch semantics](#validatorsexitbusoracle--vebo-7002)).
- **No-op / doomed requests** (validator already exiting, slashed, or otherwise unaffected by the request) — there is **no dismissal mechanism**: every queued request MUST still be executed to advance the FIFO, even if it takes no effect on the CL.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

<!-- my monkey

   w  c(..)o w
    \__(-)__/
       /\
      /  \
     w   w

-->
