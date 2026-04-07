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

CircuitBreaker is an emergency pause contract that allows the Lido DAO to grant pause authority to multisig committees (pausers). Pausers can instantly pause their assigned contracts bypassing the DAO vote. Each pause is single-use: the pauser authority is cleared upon trigger and must be explicitly restored by the DAO. Pausers must periodically prove liveness to remain authorized.

## Motivation

### Background

During an emergency, the DAO cannot respond quickly due to the on-chain governance process which takes days to enact a decision. The protocol requires a mechanism that enables designated committees to pause critical contracts immediately, without waiting for a governance vote.

The current mechanism is [GateSeals](https://github.com/lidofinance/gate-seals), temporary, single-use pause contracts deployed with a fixed committee, pause duration, set of pausable contracts, and an expiry of up to one year. A GateSeal expires either upon trigger or after one year since deployment, whichever occurs first.

### Problem

Each GateSeal cycle requires a new deployment, a Snapshot vote, and an on-chain vote that revokes the pause role from the previous GateSeal address and grants it to the new one. This process repeats annually. Adding a new pausable contract requires deploying a new GateSeal. Replacing an unresponsive committee requires deploying a new GateSeal and re-granting all permissions.

### Solution

CircuitBreaker replaces GateSeals with a single permanent contract. Under normal operation, committees extend their own authority by sending periodic heartbeats without a DAO vote required. The DAO only votes when something changes: adding a new pausable contract or replacing a committee, neither of which happens frequently. The pause role is granted to the CircuitBreaker address once per pausable contract. No redeployment or role rotation is necessary.


## CircuitBreaker

CircuitBreaker manages emergency pausing for the protocol. The admin (DAO Agent) registers pausable contracts and assigns a pauser to each one. The admin configures two global parameters: pause duration and heartbeat interval. Pausers prove liveness through periodic heartbeats; a pauser who fails to do so loses the ability to pause. Upon a successful pause, the pauser authority is cleared. The DAO must explicitly reassign the pauser to the contract.

### Roles

**Admin.** The DAO Agent. The admin registers pausable contracts, assigns pausers, and configures pause duration and heartbeat interval. The admin cannot invoke a pause.

The admin address is immutable, set in the constructor. This eliminates the entire class of ownership transfer exploits and accidental admin loss. If the Lido Agent changes address (e.g. due to a DAO executor migration), redeploying CircuitBreaker is trivial compared to the rest of the protocol migration effort. 

**Pauser.** A multisig committee approved by the DAO. A pauser can pause contracts assigned to it and send heartbeats. A pauser cannot modify any configuration.

### Deployment

The constructor accepts seven parameters: immutable admin address, immutable min/max bounds for pause duration, immutable min/max bounds for heartbeat interval, and initial values for both parameters. Initial values must fall within their respective bounds. Bounds are set in the constructor (as opposed to hardcoded constants), as testnet deployments may require different ranges for testing purposes.

### Pausable-pauser pair

Each pausable contract is assigned exactly one pauser. This provides clear accountability in an emergency. However, a single pauser can be responsible for multiple pausable contracts. The admin registers, replaces, or removes a pauser for a given pausable. When a pauser is assigned, its heartbeat is refreshed automatically, meaning newly assigned pausers start in an active state. All registered pausables are tracked onchain in an enumerable set for better storage transparency. The list of pausers can be derived from the set of pausables.

CircuitBreaker does not verify that it holds the pause role on the target contract or that the target implements the expected interface. These properties can change after registration: the pause role can be revoked, and the contract's implementation can change via a proxy upgrade. Verifying them at registration time would not provide any guarantee at trigger time. Instead, correctness at the time of trigger is treated as a trust assumption.

### Pause duration

Pause duration is a single global value within immutable bounds, adjustable by the admin. The same duration applies to every pause regardless of the target contract. The purpose of the pause is to give the DAO enough time to assess the situation and act through governance. That timeline depends on how long a DAO vote takes, not on which contract was paused.

### Pausing

To trigger a pause, the pauser specifies the target pausable contract. The contract verifies that the caller is the registered pauser for that contract and that the caller's heartbeat has not expired. It then refreshes the heartbeat, clears the pauser assignment, invokes the pause on the target contract for the configured duration, and verifies that the target reports as paused. If the post-condition is not satisfied, the transaction reverts. A pauser can trigger at most one pause per contract per assignment. 

A reentrancy guard prevents a malicious pausable from re-entering CircuitBreaker to pause a different pausable contract.

To pause multiple contracts in a single transaction atomically, the pauser must construct a multi-send batch externally.

### Heartbeat

Each pauser must periodically send a heartbeat to remain authorized. The heartbeat serves as a liveness proof: it demonstrates that the committee is ready to respond in an emergency. CircuitBreaker requires one heartbeat per pauser regardless of the number of assigned contracts.

When a heartbeat is sent, the contract computes an expiry timestamp (the current block timestamp plus the heartbeat interval) and stores it. The pauser is considered live until the block timestamp exceeds this stored expiry. A pauser whose expiry has passed can neither pause nor send another heartbeat. The only way to restore authorization is for the DAO to reassign the pauser, which sets a new expiry.

The heartbeat interval is a global value within immutable bounds, adjustable by the admin. Because each pauser's expiry is computed and stored at heartbeat time, changing the interval has no effect on existing expiry timestamps. The new interval applies only to subsequent heartbeats and registrations.

### Design decisions

**One pauser per pausable.** One pauser per contract provides clear accountability. Allowing multiple pausers for the same contract introduces ambiguity regarding responsibility.

**Single-use pause.** Clearing the authority upon trigger limits trust exposure. A compromised committee can trigger a single pause per assigned contract.

**Heartbeat expiry.** A committee that cannot prove liveness in due time is considered unreliable and cannot be trusted to act in an emergency.

**Single pause duration.** The pause window exists for the DAO to assess the situation and act. This timeline is determined by governance timelines, not by properties of the individual paused contract.

**Immutable admin.** Preventing ownership transfer eliminates transfer-related exploits and accidental admin loss. In the unlikely event the admin address must change, a new CircuitBreaker is deployed.

## Trust Assumptions

- The admin is assumed to be honest.
- A pauser is a trusted DAO-approved multisig at the time of assignment.
- A pausable contract is trusted at the time of assignment.
- CircuitBreaker holds the necessary pause role on the target contract at the time of trigger.
- Pausable contracts are expected to implement the IPausableUntil interface.

```solidity
interface IPausableUntil {
    function isPaused() external view returns (bool);
    function pauseFor(uint256 _duration) external;
}
```

## Integration

### DAO Agent

The DAO Agent is set as the admin of CircuitBreaker in the constructor. This routes all configuration changes through Dual Governance, giving stETH holders the ability to veto changes to emergency pause authority.

### Dual Governance

CircuitBreaker operates under Dual Governance ([LIP-28](./lip-28.md)). The DAO Agent that serves as CircuitBreaker's admin routes all calls through the Dual Governance timelock. Admin operations (registering a pauser, adjusting pause duration, adjusting heartbeat interval) are submitted as governance proposals and are subject to the full Dual Governance flow: proposal submission, the dynamic timelock (minimum 3 days, up to 45 days depending on stETH signalling), and the veto window. The pause duration should be sufficient to cover the base Aragon vote duration and minimum Dual Governance timelock. If the situation requires a longer pause, it can be extended through an Aragon vote or the ResealManager.

### Reseal Manager

Dual Governance includes a Reseal Committee that can extend a temporary pause to an indefinite one via the ResealManager. The ResealManager holds its own pause and resume roles on pausable contracts, independent of CircuitBreaker. Both mechanisms can coexist: CircuitBreaker triggers the initial pause, and the Reseal Committee can extend it if the situation requires prolonged pause while Dual Governance is active.

## Security Considerations

**Single point of failure.** A vulnerability in CircuitBreaker affects all pausers and all pausable contracts. CircuitBreaker addresses this by minimizing the attack surface: the contract has no proxy, no inheritance, no libraries, and no external dependencies. The admin is immutable, eliminating the governance surface around ownership.

**Compromised pauser.** A compromised committee can trigger a pause on each of its assigned contracts. CircuitBreaker limits the damage by making the pause single-use: the assignment is cleared on trigger, so the compromised committee cannot pause the same contract again without a DAO vote to reassign it.

**Lost pauser keys.** A pauser that loses access to its keys cannot act. The heartbeat mechanism surfaces unresponsive pausers: once the expiry passes, the pauser loses authorization. The DAO can reassign the pausable to a new committee.

**Malicious pausable.** A pausable contract may become malicious after registration (e.g. through a proxy upgrade). CircuitBreaker eliminates reentrancy using the Check-Effects-Interactions pattern and a reentrancy lock. CircuitBreaker holds no protocol roles beyond the pause role, so a malicious pausable that gains control during the call has no additional surface to exploit.

## Migration from GateSeals

The migration transfers emergency pause authority from the existing GateSeals to CircuitBreaker. It is executed as a single atomic governance vote.

### Step 1. Deploy CircuitBreaker

CircuitBreaker is deployed with the DAO Agent as admin and the agreed-upon bounds for pause duration and heartbeat interval. The deployment is permissionless and does not require a governance vote.

### Step 2. DAO vote

A single on-chain vote performs all of the following:

1. Grant the pause role on each pausable contract to the CircuitBreaker address.
2. Register each pausable-pauser pair on CircuitBreaker.
3. Revoke the pause roles from the existing GateSeal addresses.

Revoking GateSeal roles in the same vote eliminates ambiguity for committees about which mechanism to use. Atomic execution ensures there is no intermediate state where neither mechanism is active.

### Step 3. Verification

With the vote enactment, off-chain monitoring switches to track CircuitBreaker state and alert on approaching deadlines.

## References

- [CircuitBreaker source code](https://github.com/lidofinance/circuit-breaker)
- [GateSeals](https://github.com/lidofinance/gate-seals)
- [LIP-28: Dual Governance](./lip-28.md)