---
lip: 32
title: GateSeal V2 — Streamlining one-time emergency buttons
status: WIP
author: Alexey Komarov
discussions-to: <https://research.lido.com/t/gateseal-v2-0-proposal-thread>

created: 2025-06-04
requires: 
implementation: 
---

## Simple Summary

GateSeal v2 upgrades Lido’s emergency circuit breaker from a temporary solution into a long‑lived safety mechanism. It improves ease of use and maintenance, reduces governance overhead, and keeps the single‑use safety guarantees.

## Abstract

We propose to deploy a new GateSeal blueprint in Vyper 0.4.2. The upgrade replaces short‑lived, single‑use GateSeals with a version that supports configurable initial lifetimes and capped on‑chain prolongations. A standardized fixed signing window lets one committee manage multiple GateSeals with minimal overhead. Deployment practices are updated to group sealables by attack vectors, enabling rapid, targeted emergency response without in‑incident contract selection. The single‑use nature of the emergency tool is preserved.

## Motivation

GateSeal v1 was intentionally designed as an inconvenient temporary solution to motivate DAO Ops to find a permanent solution. Since GateSeal has become the permanent solution, it must be streamlined and designed for long‑term reliability and maintainability. As a result, the original design now leads to avoidable governance overhead, outdated hard‑coded limits, error‑prone incident response, and repeated committee coordination burden

1. **Governance overhead.** Replacing an expiring GateSeal requires a full governance cycle, even if the committee remains operational.
   *Committee-owned prolongations* prove liveness and remove routine replacement votes.

2. **Outdated hard-coded limits.** The factory enforces a 4–14 day seal duration that conflicts with [Dual Governance](https://blog.lido.fi/dual-governance-explained/) timelines and any future changes.
   *Removing arbitrary limits* avoids costly redesign cycles when governance timings evolve.

   Historically, hard‑coded “guardrails” in the contract helped prevent deployments with invalid parameters, but such bounds become obsolete as context evolves (e.g., Dual Governance) and start hindering legitimate scenarios. GateSeal v2 replaces embedded limits with process controls: Tech & Analytics prepare and justify parameter values; DAO Ops deploy strictly with the agreed values; internal and external auditors verify that on‑chain parameters match the agreed specification. The same approach applies both to lifting the 4–14 day seal duration limit and to new parameters (`prolongation_window`, `prolongation_extension`, `pre_expiration_offset`).

3. **Error‑prone incident response.** In GateSeal v1 the committee must manually select contracts during an incident, which:
- creates a human‑error risk (sealing wrong/unnecessary contracts or missing critical ones); and
- combined with the single‑use constraint, leaves no opportunity to correct mistakes or pause additional contracts.

   Solution:
   - Attack‑vector–based deployment approach: the DAO defines logical contract groups at deployment, eliminating in‑incident selection.
   - Preset activation: `seal_all()` is the primary path that seals all predefined sealables at once.
   - Tactical fallback: if `seal_all()` cannot complete, the committee should use `seal_some()` to seal the subset that can be sealed within the same attack‑vector scope.

4. **Operational scaling burden.** In the future, a single committee may be responsible for multiple GateSeals; without a standardized fixed signing window, the committee’s coordination burden (e.g., number of gatherings) is likely to grow in proportion to their number.
   *A standardized fixed signing window* lets the committee prolong multiple GateSeals in a single session.

## Specification

### Glossary

- **Prolongation Window** – active window during which the committee can prolong the contract.
- **Prolongation Extension** – extra time added on each prolongation.
- **Expiry Timestamp** – on-chain timestamp after which the GateSeal expires unless prolonged.
- **Initial Lifetime** – period from deployment until the first expiry timestamp.
- **Prolongation Limit** – maximum number of allowed prolongations.
- **Pre-Expiration Offset** – buffer for DAO Ops to deploy a replacement before expiry.

### Design Overview

High‑level architecture and responsibilities:

- **Contracts**
  - `GateSealV2` (Vyper 0.4.2): emergency sealing and lifetime management; single‑use semantics upon activation.
  - `GateSealFactory` (Vyper 0.4.2): parameter‑agnostic blueprint provider.

- **Roles**
  - Sealing committee: prolong/activate.
  - Tech & Analytics contributors: determine and justify deployment parameters.
  - DAO Ops: deploy; coordinate Snapshot voting; conduct the on‑chain voting ceremony.
  - Auditors: verify that on‑chain parameters match the agreed specification.

- **Lifetime model**
  - Expiry timestamp with capped on‑chain prolongations by the committee.
  - A standardized fixed signing window across GateSeals to batch renewals in a single session.

- **Activation modes**
  - `seal_all()` as the primary preset activation that seals the full predefined scope and immediately expires the contract.
  - `seal_some(address[])` as a tactical fallback within the same deployment scope; also single‑use.

- **Deployment model and scope**
  - Attack‑vector–based deployment: sealables are predefined per logical attack vector; up to 10 sealables per GateSeal.

- **Parameterization**
  - Only arbitrary hard‑coded limits are removed; core invariants remain enforced to preserve system correctness (see Technical Specification).



### Technical Specification

- Implementation uses **Vyper 0.4.2** for compiler security fixes, Cancun EVM support and gas improvements.
- `prolong_lifetime()`
  - callable only by the sealing committee;
  - valid when the GateSeal is unused, unexpired, within the prolongation window and has remaining prolongations;
  - adds the **Prolongation Extension** to the expiry timestamp and decrements remaining prolongations.
- `seal_all()`
  - pauses all configured contracts for `SEAL_DURATION_SECONDS`;
  - expires the GateSeal immediately.
- `seal_some(address[])`
  - pauses a subset of sealables;
  - still expires the GateSeal immediately.
- Parameters (`prolongation_extension_seconds`, `prolongation_window_seconds`, `expiration_buffer_seconds`, `expiry_timestamp`, `prolongation_limit`, `seal_duration_seconds`, `sealables`, `sealing_committee`) are immutable and set at deployment.
- The factory remains parameter-agnostic and only supplies the blueprint.
- **Total lifetime** (`Initial Lifetime + Prolongation Extension × Prolongation Limit`) must not exceed **5 years**.
- `Initial Lifetime` and `Prolongation Extension` ≥ `Expiration Buffer + Prolongation Window`.
- `Initial Lifetime` ≤ `2 × Prolongation Extension`.
- **Maximum 10 sealables** per GateSeal to maintain manageable scope while allowing comprehensive protocol coverage.
- Full compatibility with existing `PausableUntil` contracts.
- **Governance workflow:** Tech & Analytics define parameters, DAO Ops deploy with those parameters, auditors verify the deployment.

### Test Cases

Implementation test cases cover (non‑exhaustive):

- Deployment reverts if:
  - `seal_duration_seconds` is zero;
  - `sealing_committee` is the zero address;
  - `sealables` is empty, exceeds 10 entries, contains duplicates, EOAs, or the zero address;
  - `expiry_timestamp` is in the past;
  - `initial_lifetime` < `prolongation_window + expiration_buffer_seconds`;
  - `initial_lifetime` > `2 × prolongation_extension`;
  - `prolongation_extension` < `prolongation_window + expiration_buffer_seconds`;
  - calculated total lifetime exceeds 5 years.
- `seal_all()` / `seal_some()`
  - committee‑only; pause targeted contracts and emit `Sealed` events;
  - expire the GateSeal immediately; if any sealable call fails, the transaction reverts with a bitmap reason and the GateSeal stays unexpired.
- `prolong_lifetime()`
  - committee‑only; reverts outside the prolongation window, after expiry, or when no prolongations remain; updates expiry timestamp and emits `Prolonged`.
 - After expiry, any call to `seal_all`, `seal_some` or `prolong_lifetime` reverts with `"GateSeal: expired"`.


## Security Considerations

- GateSeal remains single‑use: any sealing call or manual expiry prevents further actions.
- Preset activation (`seal_all`) removes real‑time selection and reduces coordination overhead;

## Failure Modes

- Missing the prolongation window or exhausting prolongations causes expiry; the **Expiration Buffer** provides buffer time for DAO Ops to deploy a replacement.
- If `seal_all()` cannot complete, the committee should use `seal_some()` to seal the subset that can be sealed.



## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
