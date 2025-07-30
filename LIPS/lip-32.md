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
We propose to deploy a new version of the GateSeal contract that adds:
- configurable initial expiry;
- on-chain expiry prolongations;
- committee-owned prolongations (limited count);
- pre-deployed, module-specific contracts;
- strict signing schedule.
The new implementation will be written in Vyper 0.4.2, improving security, auditability and expressiveness while maintaining backward compatibility with existing PausableUntil-based contracts.

## Motivation

GateSeal V1 has never been triggered but has been repeatedly re-deployed due to its hardcoded 1-year expiry. This creates governance overhead and operational burden for the Lido DAO. Its inability to prolong lifespan without full redeployment exposes the DAO to unnecessary coordination cycles and short renewal windows. A more sustainable, long-lived model is needed to preserve the utility of the emergency switch without recurring maintenance.

## Specification

### Glossary

- **Prolongation Window** - the active window during which the committee can prolong the contract.
- **Prolongation Period** — the extra time added to the contract with each prolongation.
- **Initial Lifetime** — the time from deployment until the first expiration time.
- **Prolongation Count** — the maximum number of allowed prolongations.
- **DAO Ops Reserve** — a 60-day buffer for DAO Ops to deploy a new GateSeal before the current one expires.

### Rationale

The main tradeoff considered was between maintaining operational safety and reducing coordination overhead. Adding a prolongation mechanism shifts execution of the validity renewal from the DAO to the emergency committee, enforcing drills through real multisig action. The calculated maximum number of prolongations and enforcing liveness checks maintains a balance between flexibility and rigorness. Vyper 0.4.2 brings substantial improvements in security, gas efficiency, and maintainability by fixing known compiler issues (e.g. [GHSA-w9g2-3w7p-72g9 advisory](https://github.com/advisories/GHSA-w9g2-3w7p-72g9)), adding full support for the EVM Cancun hard fork, and refining gas-cost optimizations. For the full changelog, refer to the [official release notes](https://docs.vyperlang.org/en/stable/release-notes.html#v0-4-2-lernaean-hydra). The model stays true to the one-time-use principle to ensure GateSeal remains a high-trust, high-scrutiny tool.

### Technical Specification

- Contract will be written in Vyper 0.4.2.
- `prolongLifetime()` callable only by the committee when unused, unexpired, within the prolongation window, and with remaining prolongations.
- Each call prolongs the lifetime by the **Prolongation Period** and decrements remaining prolongations.
- `seal()` pauses configured contracts for **SEAL_DURATION_SECONDS** and expires the GateSeal immediately.
- **Prolongation Period** is set to **1 year**, ensuring that the committee needs to act only once per contract per year.
- **Prolongation Window** is fixed at **14 days** — as this remains an emergency mechanism, the committee is expected to be available for two planned sessions per year.
- **Initial Lifetime** is set at deployment and must not exceed `2 × Prolongation Period` (**2 years**), while also respecting a lower bound defined by `DAO Ops Reserve + Prolongation Window` (**74 days**), to guarantee a safe buffer before the first prolongation.
- **Prolongation Count** is defined at deployment, enabling custom total lifetimes per contract within global constraints.
- **Maximum Total Lifetime** — calculated as `Initial Lifetime + (Prolongation Period × Prolongation Count)` — must not exceed **5 years**; this is enforced at deployment time.
- The ability to **set custom seal duration limits at deployment** is removed — prior limits (e.g., 4–14 days) are now obsolete. Deployment-time configuration is immutable and subject to deploy-time verification, which provides sufficient safeguards.
- The ability to **choose which contracts are sealed at activation** is removed — each GateSeal now has a predefined scope.
- Full compatibility with existing PausableUntil contracts (e.g., WithdrawalQueue, ValidatorExitBusOracle).

### Test Cases

- Deployment reverts if any parameter exceeds its allowed limits:
  - **seal_duration** is zero
  - **initial_lifetime** is too short or too long
  - number of **sealables** is zero, too many, or contains duplicates
  - calculated total lifetime exceeds the allowed 5-year cap

- Triggering `seal()`:
  - pauses all configured sealables
  - emits a `Sealed` event with correct parameters
  - causes the GateSeal to become expired

- `prolongLifetime()`:
  - succeeds only when called within the configured prolongation window
  - extends the GateSeal's lifetime by **Prolongation Period**
  - decrements the number of prolongations remaining
  - emits a `Prolonged` event with updated values

- `prolongLifetime()` reverts if:
  - called before the prolongation window opens
  - called after the window closes
  - the GateSeal is already sealed (and thus expired)
  - the GateSeal has no prolongations left
  - called by a non-committee account

- `seal()` and `prolongLifetime()` can only be called by the committee; all unauthorized calls revert
- A second call to `prolongLifetime()` within the same window reverts, enforcing one action per window


## Security Considerations

- GateSeal V2 must remain single-use.
- Prolongation is only possible for unused and unexpired contracts, and must be performed within the prolongation window.
- Signature-based prolongation actions double as committee drills. Only unused and unexpired contracts may be prolonged, and only within the prolongation window.

## Failure Modes

- If the committee fails to prolong within the window or uses all prolongations, the GateSeal expires, requiring redeployment. However, the DAO Ops Reserve provides a safety buffer during which the DAO can deploy a replacement GateSeal to ensure continued protection.


## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
