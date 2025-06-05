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

We propose to deploy a new version of the GateSeal contract that will support a configurable initial expiry of up to 3 years and allow a limited number of on-chain expiry prolongations. Each prolongation may increase the contract's lifetime by up to 6 months, subject to conditions proving the committee’s liveness. The maximum pause window will also be increased from 14 to 21 days. The new implementation will be written in Vyper 0.4.1, improving security, auditability and expressiveness while maintaining backward compatibility with existing PausableUntil-based contracts.

## Motivation

GateSeal V1 has never been triggered but has been repeatedly re-deployed due to its hardcoded 1-year expiry. This creates governance overhead and operational burden for the Lido DAO. Its inability to prolong lifespan without full redeployment exposes the DAO to unnecessary coordination cycles and short renewal windows. A more sustainable, long-lived model is needed to preserve the utility of the emergency switch without recurring maintenance.

## Specification

### Overview

GateSeal V2 will retain the one-time activation logic but allow:
- An initial expiry up to 3 years (set at deployment),
- Up to N prolongations (e.g., 5), each increasing validity by up to 6 months,
- Prolongations callable only if the seal is unused, unexpired, and has remaining credit,
- A new pause duration limit of up to 21 days,
- Deployment using Vyper 0.4.1.

### Rationale

The main tradeoff considered was between maintaining operational safety and reducing coordination overhead. Adding a prolongation mechanism shifts responsibility for validity renewal from the DAO to the committee, enforcing drills through real multisig action. Limiting prolongations and enforcing liveness checks maintains a balance between flexibility and rigor. The switch to Vyper 0.4.1 removes compiler-level bugs and increases auditability. The model stays true to the one-time-use principle to ensure GateSeal remains a high-trust, high-scrutiny tool.

### Technical Specification

- Contract will be written in Vyper ≥0.4.1.
- Public method `extendLifetime()` will be gated by internal checks:
  - `not activated`
  - `not expired`
  - `prolongations_remaining > 0`
- When called, `prolong()` increases the `expiry_timestamp` by a fixed interval (e.g., 6 months).
- The `pause()` method behaves identically to V1 and triggers self-destruction after invocation.
- Max `pause_until` will increase from 14 to 21 days.
- Full compatibility with existing PausableUntil contracts (e.g., WithdrawalQueue, ValidatorExitBusOracle).

### Test Cases

- `pause()` is called → GateSeal self-destructs → state is frozen for up to 21 days.
- `prolong()` is called before expiry → expiry timestamp increases.
- `prolong()` fails if:
  - Contract is expired,
  - Already paused,
  - No remaining prolongations.

## Security Considerations

- GateSeal V2 must remain single-use.
- Only unused and unexpired contracts may be prolonged.
- Limited number of prolongations ensures eventual expiry.
- Signature-based prolongation actions double as committee drills.
- Increased maximum pause duration still fits within dual governance emergency limits.

## Failure Modes

- If the committee becomes inactive and all prolongations are used, the GateSeal will expire, requiring a redeploy.
- Incorrectly configured parameters (e.g., excessive max prolongations or duration) may reduce accountability.
- Misuse of the prolongation function is mitigated by strict internal checks and public transparency of calls from the reference deployment factory.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
