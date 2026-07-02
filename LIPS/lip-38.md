---
lip: 38
title: "VEBO-7002 — Withdrawal-Based Exits and Active Rebalancing"
status: WIP
author: Raman Siamionau (@F4ever)
discussions-to: <Create a new thread on https://research.lido.fi/ and drop the link here>
created: 2026-06-26
---

## Simple Summary

This proposal redesigns the Validators Exit Bus Oracle (VEBO) around EIP-7002 execution-layer withdrawal requests and introduces **Active Rebalancing** — a mechanism that lets the protocol redistribute stake across Node Operators without waiting for organic inflows and outflows.

1. **VEBO-7002** — instead of relying on Node-Operator-supplied pre-signed exit messages and the Ejector as the primary exit tool, VEBO uses EIP-7002 partial withdrawal requests (PWRs) and full withdrawal requests (FWRs) as the primary exit mechanism. This enables precise, partial stake exits from `0x02` validators, reduces unproductive stake time, shortens and sharpens withdrawal finalization, and simplifies the protocol by removing several legacy components.
2. **Active Rebalancing** — a controlled, rate-limited way for the protocol to forcibly move stake toward a target distribution by requesting additional withdrawals from over-target Community Module v2 (CMv2) operators, reusing the same withdrawal-based flow introduced by VEBO-7002.

## Abstract

We propose to rework the Validators Exit Bus Oracle ("VEBO-7002") across its on-chain and off-chain parts.

**On-chain**, we add a **FIFO queue** to the VEBO that stores EIP-7002-based withdrawal requests (partial and full) submitted by the Oracle report, together with a **permissionless handle** that anyone can call to trigger and execute queued requests one by one via the EIP-7002 withdrawal-request contract. A `setPartialWithdrawals` switch lets the protocol turn partial withdrawals off, reverting to FWR-only operation (that could be fulfilled through a Validator Ejector) under extreme EIP-7002 fee conditions.

**Off-chain**, we rework the exit-ordering logic so it selects not only which validators to exit but also the withdrawal amount per validator. This adds support for **partial withdrawal requests** — preferring PWRs against `0x02` validators and falling back to FWRs where PWRs are not applicable — and for **Active Rebalancing**: additional, rate-limited exits against CMv2 operators whose `currentStake` exceeds their `targetStake`, moving the module toward its target distribution.

This redesign also removes components made redundant by the new flow: late-exit penalties and proofs to trigger full withdrawal requests.

## Motivation

### Limitations of the current VEBO

The current VEBO depends on Node Operators uploading pre-signed validator exit messages, and on the Ejector software to broadcast those exits. This design has structural limitations:

- **Full exits only.** The flow exclusively supports full validator exits. Partial withdrawals — increasingly the right tool after EIP-7251 (MaxEB) and `0x02` credentials — are not currently available.
- **Limited upload capacity.** Only a fraction of pre-signed exit messages can be uploaded per Node Operator, to bound the blast radius if an operator's signing infrastructure is compromised. This caps how aggressively exits can be requested.
- **Operational overhead.** Every Node Operator must run and secure dedicated Ejector software, and the protocol must verify that they do.
- **Coarse exit optimization.** Because only specific pre-signed keys can be requested, optimizing exit order (e.g. to minimize sweep delay or to target precise amounts of stake) is difficult.

### Why withdrawal-based exits

EIP-7002 lets the protocol trigger withdrawals from the execution layer using validators' `0x02` withdrawal credentials, without holding pre-signed messages. Building VEBO around PWRs and FWRs:

- Allows **precise stake exit** from `0x02` validators in CMv2 using partial withdrawals, leaving the validator active.
- **Eliminates unproductive stake time** — ETH keeps earning until the moment a withdrawal is actually triggered, instead of sitting idle behind a queued full exit.
- Enables **full exits** via EIP-7002 without unfolding a VEBO report.

### Why active rebalancing

Today, stake redistribution across Node Operators and modules happens only through organic inflows (new deposits) and outflows (withdrawal demand). When a module or an individual operator drifts above its target stake, there is no protocol-level lever to correct it on a controlled schedule. For CMv2 we need a mechanism that can deliberately move the module toward its target distribution while staying compatible with the regular withdrawal and deposit flow. The withdrawal-based VEBO-7002 flow makes such a mechanism natural to express: rebalancing is simply additional, rate-limited exit demand against over-target operators.

## Specification

### Overview

1. **Off-chain Oracle computes exit demand in three sequential phases.**
    - Phase 1 — cover regular **withdrawal-queue demand**.
    - Phase 2 — issue **forced validator exits**.
    - Phase 3 — add **active rebalancing** within CMv2. Phase 3 is optional and can be switched off (leaving only Phases 1–2).
2. **The Oracle reports the ordered requests on-chain, where they are appended to a single FIFO queue in the VEBO.**
3. **Anyone can execute items in the queue.** A permissionless handle pops queued items one by one and submits them to the EIP-7002 predeploy as partial (PWR) or full (FWR) withdrawal requests, paying the per-request fee.

A **`setPartialWithdrawals(bool)` switch** can turn partial withdrawals off, making VEBO emit FWRs only; `ExitRequested` events are emitted for all FWRs so a simplified Ejector can fulfill them via voluntary exits without EIP-7002 fees.

### Rationale

#### Single FIFO queue

Withdrawal requests (both PWRs and FWRs) are stored in a single FIFO queue and processed in submission order. We chose this model because:

- **Simplicity.** A single queue is simple to iterate on-chain, with no need to manage concurrency.
- **Permissionless safety.** VEBO fully controls ordering and concurrency, which protects against permissionless interference and mitigates FWR-replay DDoS.

The trade-offs are:

- **No skipping.** Items must be executed in order; an item that is no longer valid still must be executed to process order further. We deliberately do not add an item-skipping mechanism: EIP-7002 requests are cheap, so executing a now-invalid request wastes only a negligible fee, which is not worth the extra on-chain complexity.
- **No execution-order choice.** Because order is fixed, there is no room for execution-time optimization (e.g. minimizing FWR sweep delay). This is acceptable because we expect to use mostly PWRs, so no sweep optimization is needed.
- **PWR/FWR concurrency.** PWRs and FWRs share one queue and contend for the same EIP-7002 throughput; this concurrency will be mitigated within VEBO (in how it orders and batches requests).

#### Deprecating the Ejector as the primary tool

The Ejector cannot process PWRs, the new system expects a low FWR volume, and historical mainnet data shows that EIP-7002 fee spikes are short-lived and resolve before VEBO could react. With a declining validator count and no planned reduction in EIP-7002 throughput, PWR pricing should stay favorable. The Ejector is therefore demoted to a **fallback** role, used only when partial withdrawals are turned off via `setPartialWithdrawals(bool)` (FWR-only operation).

### Technical Specification

The design splits cleanly along the oracle boundary:

- **Off-chain Oracle** owns all *ordering* logic — which module, which Node Operator, and which validator keys (with what amounts) should exit next, and all active-rebalancing decision-making. It produces the per-report list of withdrawal requests.
- **On-chain** protocol side — a thin set of primitives that:
    - ingests the Oracle report and appends its requests to a single FIFO queue;
    - exposes a permissionless handle to push queued requests to the EIP-7002 contract;
    - enforces the exit limits (TWG rate limit on extracted balance);
    - enforces the configurable rebalancing bounds (rate limit, thresholds);
    - holds partial withdrawal requests off/on switch (full withdrawal requests only mode);

#### Off-chain Oracle: ordering and rebalancing logic

Per report, the Oracle produces the ordered list of withdrawal requests. It runs the exit-order iterator ([`exit_order_iterator.py`](https://github.com/lidofinance/lido-oracle/blob/f352b8552b717ebf8ecaf3bd0f5cf6785909ec17/src/services/exit_order_iterator.py#L106)) — generalized from "pick the next validators to fully exit" to "pick the next validators and the amount (PWR/FWR) to withdraw" — across **three sequential phases**, each with its own demand source.

##### Shared iterator mechanics

All three phases reuse the same machinery. Two governance-tunable parameters drive it, both stored in the `OracleDaemonConfig` smart contract so they can be changed without an Oracle release:

- **`EXIT_ITERATION_CHUNK`** (default **32 ETH**) — the fixed unit of demand the iterator allocates per step.
- **`MIN_PARTIAL_WITHDRAWAL`** (default **8 ETH**) — the minimum withdrawable balance a validator must have above the effective-balance floor to serve a PWR, and the lower bound of any single PWR.

**Iteration granularity.** The iterator walks exit demand in fixed `EXIT_ITERATION_CHUNK` (32 ETH) steps: each iteration allocates one chunk of demand to the top-ranked operator (and one of its validators), then re-ranks. This keeps the ordering balanced across operators and maps naturally onto withdrawal sizing — a chunk becomes a PWR of up to `EXIT_ITERATION_CHUNK` from a `0x02` validator, or an FWR.

**PWR vs. FWR threshold.** A validator can only serve a PWR if it has enough balance above the effective-balance floor to give up. If **no** validator has more than `MIN_PARTIAL_WITHDRAWAL` (8 ETH) of withdrawable balance above the floor, a **full withdrawal request** is used against the top-ranked validator instead. Otherwise a partial withdrawal is taken from that validator, bounded to the `[MIN_PARTIAL_WITHDRAWAL, EXIT_ITERATION_CHUNK]` (8–32 ETH) range per validator.

**Per-report exit limit.** The iterator stops accumulating requests once the report's cumulative exited balance (`max_current_exit_balance`) reaches the per-report exit limit. This limit is shared across all three phases: Phase 1 draws against it first, and Phases 2–3 consume whatever remains.

##### Phase 1 — Cover withdrawal-queue demand

Phase 1 covers the current **withdrawal-queue (WQ) demand**, issuing PWR-first instead of full exits. It walks that demand in `EXIT_ITERATION_CHUNK` steps, ranking validator at each step by, in order:

1. **Forced-exit deviation** — operators over a force-exit limit first, by how far `currentValidators − targetValidators` exceeds the threshold.
2. **Soft-exit deviation** — same idea for soft-exit limits, after forced exits.
3. **Module share-rate deviation** — modules above their `priority_exit_share_threshold` (relative to total protocol balance) get elevated priority.
4. **Target-stake deviation** — operators furthest **above** their target stake exit first, where `targetStake = moduleStake × operatorWeight / moduleTotalWeight`.
5. **Already-requested validators** — validators that already have PWR demand allocated to them **within this report** rank first, but only while their remaining balance stays above `MIN_ACTIVATION_BALANCE`. This concentrates exited balance into as few validators as possible, minimizing the number of distinct PWRs by taking more balance out of one validator before moving to the next.
6. **Biggest validator first** — among the operator's validators, prefer the one with the **largest balance**. This maximizes the balance a single PWR can extract while leaving the validator active, favoring partial over full withdrawals.
7. **Lowest validator index** — within an operator, ascending validator index.

##### Phase 2 — Forced validator exits

After WQ demand is covered, the iterator satisfies any **forced-exit obligations**.

1. **Forced-exit deviation** — operators over a force-exit limit first, by how far `currentValidators − targetValidators` exceeds the threshold.
2. **Lowest validator index** — within an operator, ascending validator index.

##### Phase 3 — Active rebalancing

This phase applies **only to the CMv2 staking module** and adds exit demand against operators drifting above their target stake, pushing the module back toward its target distribution.

**Configuration.** All Phase 4 parameters live as keys in the existing **`OracleDaemonConfig`** key-value store.

| `OracleDaemonConfig` key                | Type        | Meaning                                                                                     |
|-----------------------------------------|-------------|---------------------------------------------------------------------------------------------|
| `ACTIVE_REBALANCING_ENABLED`            | bool        | Global on/off for Phase 3.                                                                  |
| `ACTIVE_REBALANCING_RATE_LIMIT`         | uint (ETH)  | Max stake exitable for rebalancing per VEBO report.                                         |
| `ACTIVE_REBALANCING_OPERATOR_EXCESS_BP` | uint (bp)   | Operator-level trigger: `currentStake` over `targetStake` as a share of `targetStake`.      |
| `ACTIVE_REBALANCING_GRACE_PERIOD`       | uint (days) | Grace period before a new operator can affect the module's stake balance used to rebalance. |

**Trigger.** Phase 3 runs only if `ACTIVE_REBALANCING_ENABLED` is set. 

A **new-operator grace period** applies: operators created and whose first key activated less than `ACTIVE_REBALANCING_GRACE_PERIOD` days ago cannot *affect* rebalancing calculations, but remain eligible to *receive* redistributed deposits via the normal deposit allocation.

**Bounds.** The total rebalancing stake added to the report is capped at `ACTIVE_REBALANCING_RATE_LIMIT`.

**Ordering.** When it runs, the iterator ranks over-target CMv2 operators by, in order:

1. **Target-stake deviation** — an operator is over-target when its `currentStake` exceeds `targetStake × (1 + ACTIVE_REBALANCING_OPERATOR_EXCESS_BP)`; eligible operators furthest **above** that threshold rank first.
2. **Already-requested validators** — validators with PWR demand already allocated **within this report** rank first, concentrating exited balance into as few validators as possible.
3. **Biggest validator first** — prefer the largest-balance validator so exits stay partial and keep the validator active.
4. **Lowest validator index** — within an operator, ascending validator index.

#### On-chain Oracle

On-chain, the protocol takes the Oracle's ordered requests and turns them into EIP-7002 withdrawals. This happens in **two flows**:

- **Report submission** — the Oracle submits a packed report; each request is decoded and appended to a single FIFO queue.
- **Execute withdrawal requests** — a permissionless handle pops queued requests and pushes them down the request path to the EIP-7002 predeploy:

```
VEBO-7002 (FIFO queue)  ->  TriggerableWithdrawalsGateway (TWG)  ->  WithdrawalVault  ->  EIP-7002 predeploy
  report submission                 rate limits, fee refund             encoding         withdrawal request
```

##### Flow 1 — Report submission

The Oracle calls `submitReportData` with a packed report. The report keeps the existing encoding — a `dataFormat` selector plus a `data` blob of fixed-width records packed contiguously — but introduces a **new `dataFormat` version** whose record carries the withdrawal **amount**. Today's record is 64 bytes; the new record appends an 8-byte `amount`:

```
/// MSB <----------------------------------------------------------------------------------- LSB
/// |  3 bytes   |   5 bytes    |    8 bytes     |      48 bytes       |      8 bytes          |
/// |  moduleId  |  nodeOpId    | validatorIndex |   validatorPubkey   |  amount (gwei) *new*  |
```

`amount` (uint64, gwei) — **new**; `0` = full withdrawal (FWR), `> 0` = partial withdrawal (PWR).

On submission the contract decodes each record into an `ExitRequest` struct and appends it to the FIFO queue. 

When `setPartialWithdrawals` is off, the report is still in the new format but every record MUST carry `amount == 0` (FWR-only); a non-zero amount is rejected.

##### Flow 2 — Execute withdrawal requests

Anyone calls the permissionless `processExitRequests(count)`, forwarding the per-request EIP-7002 fee. It pops the next `count` requests from the FIFO in order and pushes each down the path above: **TWG** applies the exit limits and refunds excess fee, then the **WithdrawalVault** encodes the request and submits it to the EIP-7002 contract. A request rejected at the predeploy is dropped and must be re-submitted by a later Oracle report.

```solidity
interface IValidatorsExitBusOracle7002 {
    /// A single queued withdrawal request.
    /// amountGwei == 0  -> full withdrawal request (FWR)
    /// amountGwei  > 0  -> partial withdrawal request (PWR)
    struct ExitRequest {
        uint256 moduleId;
        uint256 nodeOperatorId;
        bytes   pubkey;
        uint64  amountGwei;
    }

    /// Flow 1: Oracle submits the ordered, packed report; requests are appended to the FIFO queue.
    function submitReportData(bytes calldata report, uint256 contractVersion) external;

    /// Flow 2: permissionless — pop and deliver the next `count` queued requests to the
    /// EIP-7002 withdrawal-request path. Caller forwards the per-request fee.
    function processExitRequests(uint256 count) external payable;

    /// Number of requests waiting in the FIFO queue.
    function unprocessedRequestsCount() external view returns (uint256);

    /// FWR-only fallback switch (privileged).
    function setPartialWithdrawals(bool enabled) external;
    function partialWithdrawalsEnabled() external view returns (bool);

    /// Emitted for every request (validator ejector can use this events 
    /// to fulfill exits when partial withdrawals are turned off).
    event ExitRequested(
        uint256 indexed moduleId,
        uint256 indexed nodeOperatorId,
        bytes pubkey,
        uint64 amountGwei
    );
}
```

##### Scope — contracts to change

| Contract                                | Change                                                                                                                                                                  |
|-----------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `ValidatorsExitBusOracle` (→ VEBO-7002) | New report `dataFormat` with `amount`; FIFO queue storage; permissionless `processExitRequests` handle; `setPartialWithdrawals` switch.                                 |
| `TriggerableWithdrawalsGateway` (TWG)   | Single path for all withdrawal requests (partial and full); drop `_notifyStakingModules`; rate limiter changed to bound **extracted balance** instead of request count. |
| `StakingRouter`                         | Remove `onValidatorExitTriggered` (no longer called by the TWG) and `reportValidatorExitDelay` (late-exit-penalty accounting).                                          |
| `ValidatorExitDelayVerifier`            | **Removed entirely** — the exit-delay proof contract is obsolete once late-exit penalties are gone.                                                                     |
| `WithdrawalVault`                       | Already supports partial withdrawals — redeploy only to point at the new TWG address (immutable authorized caller).                                                     |
| `LidoLocator`                           | Register the upgraded TWG, and remove `validatorExitDelayVerifier` addresses.                                                                                           |
| `OracleDaemonConfig`                    | Add the iterator params (`EXIT_ITERATION_CHUNK`, `MIN_PARTIAL_WITHDRAWAL`) and the active-rebalancing keys.                                                             |

