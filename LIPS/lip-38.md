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
    - enforces the exit limits (VEBO rate limit on balance exit and TWG rate limit on requests count);
    - enforces the configurable rebalancing bounds (rate limit, thresholds);
    - holds partial withdrawal requests off/on switch (full withdrawal requests only mode);
    - verifies that partial withdrawals target `0x02` validators;

This separation keeps all economic and prioritization policy in upgradeable off-chain code, while the on-chain layer stays a thin, auditable set of primitives and limits.

