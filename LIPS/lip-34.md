---
lip: 34
title: CircuitBreaker
status: Proposed
author: Azat Serikov
discussions-to: TBD
created: 2026-03-20
---

# LIP-34: CircuitBreaker

## Simple Summary

CircuitBreaker lets the DAO delegate emergency pause authority to multisig committees. These committees can instantly pause critical contracts without a DAO vote. Each pausable contract has exactly one assigned pauser. The pauser can pause a contract only once and must be reassigned by the DAO to be able to pause again. Pausers must maintain a periodic heartbeat to remain authorized.

## Motivation

### Background

In an active exploit, the DAO cannot respond quickly enough due to vote duration. The protocol needs a mechanism for designated committees to pause critical contracts instantly, without waiting for governance.

The current mechanism is [GateSeals](https://github.com/lidofinance/gate-seals) - temporary, one-off contracts deployed with a fixed committee, pause duration, set of pausable contracts, and an expiry of up to one year. A GateSeal expires on trigger or after a year since deployment, whichever comes first.

### Problem

The redeployment cycle creates operational burden: deploy a new GateSeal, hold a snapshot vote, run an on-chain vote revoking the pause roles from the old GateSeal and granting them to a new one. Adding a new pausable contract requires deploying a new GateSeal. Swapping a dead committee requires deploying a new GateSeal and re-granting all permissions.

### Solution

CircuitBreaker is a permanent contract that replaces the temporary GateSeal mechanism. Like an electrical circuit breaker, it trips under fault conditions, protects the system, and is reset by an authorized party. It does not self-destruct after tripping.

The DAO grants the pause role on a pausable contract to the CircuitBreaker address, and that grant survives all pause cycles. Adding a new pausable contract or swapping a committee requires a single governance vote item. No redeployment, no role rotation.

## CircuitBreaker

A single contract holds pause authority for the pausable contracts. The DAO Agent (admin) maps each pausable contract to one pauser and configures two global parameters: pause duration and heartbeat interval. Pausers prove liveness through periodic heartbeats. An expired heartbeat blocks pausing. On trigger, the pauser assignment is cleared and the DAO must explicitly reassign the pauser. The contract uses no proxy, no inheritance, and no libraries. The admin address is immutable.

### Roles

**Admin.** The DAO Agent. Immutable, set in the constructor and cannot be changed. Assigns pausers, configures pause duration and heartbeat interval. Cannot pause.

**Pauser.** A multisig committee approved by the DAO. Can pause contracts it is assigned to and send heartbeats. Cannot configure anything.

### Deployment

The constructor takes seven parameters: admin address, min/max bounds for pause duration, min/max bounds for heartbeat interval, initial pause duration, and initial heartbeat interval. Initial values must fall within their respective bounds. Bounds are immutable but configurable per network as testnet deployments may require different bounds for testing purposes.

### Pausable-pauser mapping

Each pausable contract maps to exactly one pauser. One pauser can have multiple pausables. The admin assigns or replaces a pauser via `setPauser(pausable, pauser)`. Passing `address(0)` removes the assignment. On non-zero assignment, the pauser's heartbeat is automatically updated, meaning newly assigned pausers start as active.

The contract does not verify that CircuitBreaker holds the pause role on the target pausable contract or that the target implements the `IPausable` interface correctly. This is the DAO's responsibility during the governance vote that assigns the pauser.

### Pause duration

A single global value within set bounds, applied uniformly to all pausables on trigger. Updatable by the admin. Suggested bounds 5 to 30 days.

### Heartbeat

The heartbeat is tied to the pauser address. One heartbeat transaction proves liveness for every contract that pauser manages.

A pauser is active when `block.timestamp <= latestHeartbeat[pauser] + heartbeatInterval`. The heartbeat blocks both pausing and heartbeat renewal: a pauser whose heartbeat has expired cannot pause or refresh. A committee that cannot prove liveness should not be trusted to respond in an emergency.

The heartbeat interval is a single global value within bounds, updatable by the admin. Changing it retroactively affects all existing pausers, i.e. a reduction can make a pauser inactive or, vice versa, an interval increase can make a pauser active again.

### Pausing

When a pauser calls triggers a pause, the contract verifies the caller is the assigned pauser with an active heartbeat, updates the heartbeat, clears the pauser assignment, calls `pausable.pauseFor(pauseDuration)` on the target contract, and verifies it is paused as a post-condition. If the post-condition fails, the transaction reverts.

The pauser assignment is cleared before the external call to the pausable. A transient storage reentrancy guard prevents cross-pausable reentrancy through malicious `pauseFor` implementations.

After a successful pause, the DAO must call `setPauser` to reassign the pauser (or assign a new pauser). This bounds trust delegation: one pause per assignment. Batching multiple pauses is done externally via multisig multi-send.

### Design decisions

**One pauser per pausable.** Multiple pausers per pausable create ambiguity about responsibility. A 1:1 mapping keeps accountability clear.

**Single-use pause.** Clearing the assignment on trigger limits trust. A compromised committee fires exactly once per assigned contract.

**Heartbeat gates pausing.** Enforcing liveness blocks expired pausers from acting. A pauser who fails to maintain liveness is considered unreliable and cannot be trusted to act in an emergency.

**One pause duration for all contracts.** Pause duration is defined by the governance timings, not the specifics of the contract.

**Immutable admin.** Eliminates ownership transfer exploits and accidental admin loss. If the admin needs to change (which is extremely rare), a new CircuitBreaker needs to be deployed.

## Trust Assumptions

- Admin is always honest. Admin can make mistakes.
- Pauser is a DAO-approved multisig at assignment time. Pauser can become compromised, lose keys, or make mistakes.
- CircuitBreaker has the necessary pause roles at trigger time.
- Pausable is trusted at assignment time. Pausable can become malicious. Pausable implements `IPausable`:

```solidity
interface IPausable {
    function isPaused() external view returns (bool);
    function pauseFor(uint256 _duration) external;
}
```

Lido's `PausableUntil` base contract satisfies this interface.

## Integration

The DAO grants the pause role on a pausable contract to the CircuitBreaker address and assigns the pauser in the same vote.

CircuitBreaker operates under Dual Governance ([LIP-28](./lip-28.md)). Admin calls (`setPauser`, `setPauseDuration`, `setHeartbeatInterval`) go through Dual Governance. Emergency `pause` calls bypass governance which is the contract's purpose. Reassignment after a pause goes through normal governance and is subject to veto.

## Security Considerations

**Single point of failure.** A bug in CircuitBreaker affects all pausers and protected contracts. Mitigation: the contract is designed with simplicity as the core principle: immutable admin, no proxy, no inheritance, no libraries, and no external dependencies beyond `IPausable`.

**Compromised pauser.** A compromised pauser can fire pause an assigned contract. Mitigation: the pause authority is single-use which limits impact.

**Lost pauser keys.** A pauser that loses keys retains its assignment. Mitigation: when the heartbeat expires, the pauser loses authority.

**Malicious pausable.** A malicious `pauseFor` implementation could attempt reentrancy. Mitigation: the contract uses a transient storage reentrancy guard.

**Off-chain operational complexity.** Risk concentrates off-chain: multisig compromise, key management, response time during active exploits. Mitigation: offchain alerting.

## References

- [CircuitBreaker source code](https://github.com/lidofinance/circuit-breaker)
- [GateSeals](https://github.com/lidofinance/gate-seals)
- [LIP-28: Dual Governance](./lip-28.md)
