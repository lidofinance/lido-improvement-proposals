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

GateSeal V2 introduces a more flexible, long-lived version of Lido’s emergency circuit breaker. It aims to preserve one-time emergency protection while drastically reducing governance and operational overhead by enabling on-chain expiry prolongation and modernizing the codebase.

## Abstract

We propose to upgrade the GateSeal blueprint to include:
- configurable initial expiry;
- on-chain expiry prolongations;
- committee-owned prolongations (limited count);
- strict signing schedule.

The new implementation will be written in Vyper 0.4.2, improving security, auditability and expressiveness while maintaining backward compatibility with existing PausableUntil-based contracts.

## Motivation

[GateSeal V1 served as a temporary mechanism](https://github.com/lidofinance/gate-seals/blob/564955d1ec30ee832179ebcef131392a5fe8ab69/README.md?plain=1#L22) that intentionally introduced operational overhead to prompt the DAO to develop a long-term sustainable solution. Now that GateSeal has become a permanent part of Lido’s emergency infrastructure, its architecture must evolve — from a short-term, incentive-based tool to a reliable and maintainable mechanism.

## Specification

### Glossary

- **Prolongation Window** - the active window during which the committee can prolong the contract.
- **Prolongation Period** — the extra time added to the contract with each prolongation.
- **Expiry Timestamp** — the on-chain timestamp after which the GateSeal expires unless prolonged. This is the primary lifetime reference in GateSeal v2.
- **Initial Lifetime** — effectively represented by the period from deployment until the first **Expiry Timestamp**. 
- **Prolongation Limit** — the maximum number of allowed prolongations.
- **Pre-Expiration Offset** — the time buffer for DAO Ops to deploy a new GateSeal before the current one expires.

### Rationale

The main tradeoff considered was between maintaining operational safety and reducing coordination overhead. Adding a prolongation mechanism shifts the execution of validity renewals from the DAO to the emergency committee, enforcing drills through real multisig action. The calculated maximum number of prolongations and the enforcement of liveness checks maintain a balance between flexibility and rigor. Vyper 0.4.2 brings substantial improvements in security, gas efficiency, and maintainability by fixing known compiler issues (e.g. [GHSA-w9g2-3w7p-72g9 advisory](https://github.com/advisories/GHSA-w9g2-3w7p-72g9)), adding full support for the EVM Cancun hard fork, and refining gas-cost optimizations. For the full changelog, refer to the [official release notes](https://docs.vyperlang.org/en/stable/release-notes.html#v0-4-2-lernaean-hydra). The model stays true to the one-time-use principle to ensure GateSeal remains a high-trust, high-scrutiny tool.

### Technical Specification

- The contract will be written in Vyper 0.4.2.
- `prolong_lifetime()` callable only by the committee when unused, unexpired, within the prolongation window, and with remaining prolongations.
- Each call prolongs the lifetime by the **Prolongation Period** and decrements remaining prolongations.
- `seal_some()` and `seal_all` pause configured contracts for **SEAL_DURATION_SECONDS** and expires the GateSeal immediately.

- `Prolongation Period` (in seconds), `Prolongation Window` (in seconds), `Pre-Expiration Offset` (in seconds), together with `Expiry Timestamp` and`Prolongation Limit` are all set at the time of each GateSeal deployment; 
- The former 4–14‑day `Seal Duration` limit is removed.

    *All of this, on the one hand, provides us with greater flexibility in configuring parameters, but on the other hand, carries the risk of deploying with incorrect values. Therefore, Tech and Analytics contributors are responsible for determining the correct parameters to be deployed, while internal and external auditors verify that the on-chain deployment matches the agreed values.*

- The commttee's ability to select which contracts must be sealed is removed.    
    *The committee no longer picks contracts to pause; instead, it chooses among GateSeals, each with a predefined, fixed set of sealables for faster incident response.*
   
- The factory remains parameter-agnostic and only supplies the GateSeal blueprint.
- The **maximum total lifetime**, calculated as `Initial Lifetime + (Prolongation Period × Prolongation Limit)` must not exceed **5 years**. This is enforced during deployment.
- `Initial Lifetime` and `Prolongation Period` must be ≥ `Pre-Expiration Offset + Prolongation Window`.
- `Initial Lifetime`  must be ≤ `2 × Prolongation Period` also. 
- Full compatibility with existing PausableUntil contracts (e.g., WithdrawalQueue, ValidatorExitBusOracle).

### Test Cases

- **Deployment reverts if:**
  - `seal_duration_seconds` is zero
  - `sealing_committee` is the zero address
  - the `sealables` list is empty, contains more than 10 entries, includes duplicates, or contains the zero address
  - the `sealables` list contains an EOA (non-contract) address
  - `expiry_timestamp` is set in the past
  - `initial_lifetime` is shorter than `prolongation_window_seconds + pre_expiration_offset_seconds`
  - `initial_lifetime` exceeds `2 × prolongation_period_seconds`
  - `prolongation_period_seconds` is shorter than `prolongation_window_seconds + pre_expiration_offset_seconds`
  - the calculated total lifetime (`initial_lifetime + prolongation_period_seconds × prolongation_limit`) exceeds the 5-year cap

- **Triggering `seal_all()` or `seal_some()`:**
  - may only be called by the committee; all unauthorized calls revert
  - pauses all configured sealables for `SEAL_DURATION_SECONDS`
  - emits a `Sealed` event for each sealable with correct parameters
  - immediately marks the GateSeal as expired
  - if any sealable call fails, the transaction reverts with a bitmap reason string encoding failed indices, and the GateSeal remains not expired

- **Calling `prolong_lifetime()`:**
  - may only be called by the committee; all unauthorized calls revert
  - only succeeds if called within the configured prolongation window; attempts before or after the window revert
  - reverts if the GateSeal is already expired or if no prolongations remain
  - extends the lifetime by `Prolongation Period` and decrements the prolongations remaining
  - emits a `Prolonged` event with updated values
  - a second call within the same window reverts, enforcing one action per window

- **Expired state & window semantics:**
  - after natural expiry, `seal_all()`, `seal_some()` and `prolong_lifetime()` revert with `"GateSeal: expired"`
  - after `seal_some()` or `seal_all()` (forced expiry), `is_expired()` returns true and prolongation window bounds become zero (`get_prolongation_window_start() == 0`, `get_prolongation_window_end() == 0`)

- **Immutables & getters:**
  - `get_prolongation_period_seconds()` equals the configured `PROLONGATION_PERIOD_SECONDS`
  - `get_prolongation_window_seconds()` equals `PROLONGATION_WINDOW_SECONDS`
  - `get_pre_expiration_offset_seconds()` equals `PRE_EXPIRATION_OFFSET_SECONDS`
  - `get_prolongations_remaining()` equals the configured `prolongation_limit` at deployment, decremented after each prolongation
  - `get_seal_duration_seconds()` equals the configured `SEAL_DURATION_SECONDS`
  - `get_expiry_timestamp()` returns the current expiry timestamp
  - `get_sealing_committee()` equals the configured committee address
  - `get_sealables()` returns the configured sealables array


## Security Considerations

- GateSeal V2 must remain single-use.
- Prolongation is only possible for unused and unexpired contracts, and it must be performed within the prolongation window.


## Failure Modes

- If the committee fails to prolong within the window or uses all prolongations, the GateSeal expires, requiring replacement. However, the Pre-Expiration Offset provides a safety buffer during which the DAO can deploy a new GateSeal instance to ensure continued protection.


## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
