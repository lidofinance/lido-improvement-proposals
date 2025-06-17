---
lip: 32
title: GateSeal V2.0 — Streamlining one-time emergency buttons
status: WIP
author: Alexey Komarov
discussions-to: <https://research.lido.com/t/gateseal-v2-0-proposal-thread>

created: 2025-06-04
requires: 
implementation: 
---

## Simple Summary

GateSeal V2.0 introduces a more flexible, long-lived version of Lido’s emergency circuit breaker. It aims to preserve one-time emergency protection while drastically reducing governance and operational overhead by enabling on-chain expiry prolongation and modernizing the codebase.

## Abstract
We propose to deploy a new version of the GateSeal contract that will support a configurable initial expiry, permits on-chain expiry prolongations and has increased the pause duration. The new implementation will be written in Vyper 0.4.1, improving security, auditability and expressiveness while maintaining backward compatibility with existing PausableUntil-based contracts.

## Motivation

GateSeal V1 has never been triggered but has been repeatedly re-deployed due to its hardcoded 1-year expiry. This creates governance overhead and operational burden for the Lido DAO. Its inability to prolong lifespan without full redeployment exposes the DAO to unnecessary coordination cycles and short renewal windows. A more sustainable, long-lived model is needed to preserve the utility of the emergency switch without recurring maintenance.

## Specification

### Overview

GateSeal V2 will retain the one-time activation logic but allow:
- Contract Lifetime: between 6 and 24 months (set at deployment);
- Total Lifetime: up to 5 years (fixed);
- Prolongation count: min 0; max = `Total Lifetime / Contract Lifetime - 1` (calculated at deployment);
- Prolongation window: 
  - opens 2 months before the contract’s expiry (fixed);
  - remains active for 1–2 weeks (set at deployment);
- Pause duration up from 14 to 21 days (fixed).

### Rationale

The main tradeoff considered was between maintaining operational safety and reducing coordination overhead. Adding a prolongation mechanism shifts execution of the validity renewal from the DAO to the emergency committee, enforcing drills through real multisig action. The calculated maximum number of prolongations and enforcing liveness checks maintains a balance between flexibility and rigorness. Vyper 0.4.1 brings substantial improvements in security, gas efficiency, and maintainability by fixing known compiler issues (e.g. [GHSA-w9g2-3w7p-72g9 advisory](https://github.com/advisories/GHSA-w9g2-3w7p-72g9)), adding full support for the EVM Cancun hard fork, and refining gas-cost optimizations. For the full changelog, refer to the [official release notes](https://docs.vyperlang.org/en/stable/release-notes.html#v0-4-1-tokara-habu). The model stays true to the one-time-use principle to ensure GateSeal remains a high-trust, high-scrutiny tool.

### Technical Specification

- Contract will be written in Vyper 0.4.1.
- `prolongLifetime()` callable only by the committee when unused, unexpired, within the prolongation window, and with remaining prolongations.
- Each call extends `expiry_timestamp` by the initial lifetime and decrements `prolongations_remaining`.
- `seal()` pauses configured contracts for `SEAL_DURATION_SECONDS` and expires the GateSeal immediately.
- `MAX_SEAL_DURATION_SECONDS` equals 21 days.
- Full compatibility with existing PausableUntil contracts (e.g., WithdrawalQueue, ValidatorExitBusOracle).

### Test Cases

- Deployment reverts if any parameter exceeds its allowed limits (e.g., Total Lifetime > 5 years).
- Triggering `seal()` pauses contracts and expires the GateSeal.
- `prolongLifetime()` called before expiry increases `expiry_timestamp` and reduces remaining prolongations.
- `prolongLifetime()` reverts if the GateSeal is expired, already sealed, outside the window, or no prolongations remain.

## Security Considerations

- GateSeal V2 must remain single-use.
- Prolongation is only possible for unused and unexpired contracts, and must be performed within the prolongation window.
- The calculated maximum number of prolongations ensures eventual expiry.
- Signature-based prolongation actions double as committee drills. Only unused and unexpired contracts may be prolonged, and only within the prolongation window.

## Failure Modes

- If the committee fails to prolong within the window or uses all prolongations, the GateSeal expires, requiring redeployment.
- Misconfigured parameters could reduce accountability or shorten usability.
- Extended pause duration still respects dual-governance emergency limits.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
