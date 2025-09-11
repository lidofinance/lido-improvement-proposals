---
lip: 32
title: Sanity Checks for stVaults
status: WIP
author: Alexandr Drygin, Greg Shestakov, Victor Petrenko
discussions-to: tbd
created: 2025-09-01
updated: 2025-09-12
---

# LIP-32: Sanity Checks for stVaults

## Simple Summary

This proposal defines sanity checks for stVaults oracle reporting in Lido V3, establishing safeguards against malicious oracle behavior through on-chain validation of vault parameters and introducing a quarantine mechanism for indirect funding to prevent potential attacks.

## Abstract

Lido V3 introduces stVaults, a new staking primitive that allows independent operators to manage validators with their own withdrawal credentials while minting stETH backed by their stake. This LIP proposes a comprehensive set of sanity checks for the stVaults oracle system to prevent malicious behavior and ensure the integrity of vault parameters.

The proposal introduces two funding flow types: direct funding through the `fund()` method, and indirect funding which requires a 3-day quarantine period (with an exception for increases less than 3.5% of vault total value). These mechanisms, combined with parameter validation for `totalValue`, `cumulativeLidoFees`, `liabilityShares`, and `slashingReserve`, create a robust defense against potential oracle manipulation.

## Motivation

With the introduction of stVaults in Lido V3, vault parameters are reported through a [`LazyOracle`](https://hackmd.io/@lido/stVaults-design#313-Essentials-LazyOracle) system where:

- A daily accounting oracle reports Merkle roots of vault states
- Vault owners bring on-demand reports with Merkle proofs for operations with underlying funds

This architecture creates potential attack vectors if the oracle committee is compromised or software is malfunctioning. A misbehaving oracle could manipulate vault parameters to:

- Enable minting of unbacked stETH by inflating `totalValue`
- Allow withdrawal of collateral by understating `liabilityShares`
- Avoid penalty payments by manipulating fee and reserve parameters

The most critical risk is that a misbehaving oracle, in collusion with node operators, could potentially steal up to 30% of ETH (considering the global minting for external shares) from the Lido protocol by creating vaults with artificially inflated values.

This proposal addresses these risks by implementing comprehensive sanity checks and a quarantine mechanism for funds that cannot be verified on-chain immediately.

## Specification

### Existing Sanity Checks

Lido V2's existing oracle sanity checks validate the following [parameters](https://github.com/lidofinance/core/blob/v2.1.0/contracts/0.8.9/sanity_checks/OracleReportSanityChecker.sol#L475):

```solidity
function checkAccountingOracleReport(
    uint256 _timeElapsed, 
    uint256 _preCLBalance, 
    uint256 _postCLBalance, 
    uint256 _withdrawalVaultBalance, 
    uint256 _elRewardsVaultBalance, 
    uint256 _sharesRequestedToBurn, 
    uint256 _preCLValidators, 
    uint256 _postCLValidators 
)
```

stVaults do not affect these existing parameters as:
- stVault validators are excluded from `CLBalance` calculations
- stVault rewards and withdrawals use separate withdrawal credentials
- Burn requests are handled independently

Therefore, existing sanity checks remain unchanged. 

### Lazy Oracle Architecture

To avoid unbounded loops in on-chain code, the `LazyOracle` system:
1. Stores complete vault reports off-chain on a decentralized medium (IPFS)
2. Submits only Merkle roots on-chain
3. Requires vault owners to provide Merkle proofs for operations

This design enables scalability while maintaining security through cryptographic verification.

### Vault Parameters and Attack Vectors

Each vault report includes four critical parameters that require validation: 

#### 1. totalValue

**Definition:** Sum of execution layer (EL) and consensus layer (CL) balances, including pending deposits and withdrawals.

**Attack vectors:**
- **Overstatement:** Enables minting of unbacked stETH up to the vault's limit
- **Understatement:** May trigger inappropriate forced rebalance of the stVault 

#### 2. cumulativeLidoFees

**Definition:** Cumulative fees owed to Lido, denominated in ETH.

**Attack vectors:**
- **Overstatement:** Forces vault owners to pay excessive fees
- **Understatement:** Allows fee avoidance

#### 3. liabilityShares

**Definition:** Amount of stETH shares minted by the vault.

**Attack vectors:**
- **Overstatement:** Freezes excess funds
- **Understatement:** Enables collateral withdrawal earlier than required to mitigate potential slashing risks

#### 4. slashingReserve

**Definition:** Locked assets reserved for slashing penalties.

**Attack vectors:**
- **Overstatement:** Unnecessarily locks vault funds
- **Understatement:** Allows penalty avoidance during slashing events


### Proposed Sanity Checks

Based on the identified attack vectors, we propose the following validation mechanisms: 

#### totalValue Upper Bound

The upper bound for `totalValue` requires special consideration due to the critical attack vector it addresses. See the [detailed analysis](#totalvalue-upper-bound-critical-issue) below.

#### totalValue Lower Bound

**Validation:** `totalValue >= 0`

**Analysis:** While understating `totalValue` could trigger force rebalance, the attack vectors are limited:
- Attackers seeking quick fund extraction would prefer secondary market-swaps over complex oracle manipulation
- Malicious force rebalance causes minimal damage as vault owners retain their stETH

**Decision:** No strict lower bound check beyond non-negativity to avoid impacting normal operations.

#### cumulativeLidoFees Upper Bound

**Validation Formula:** `maxFeeIncrease = (currentTimestamp - previousTimestamp) * 0.18 ETH`

**Rationale:** Since fees are derived only from rewards (not TVL), they cannot exceed total Ethereum issuance. The 0.18 ETH/second rate ensures the check remains valid even under extreme and quite unrealistic conditions:
- All vault ETH concentrated in a single vault
- Entire ETH supply staked through Lido
- Maximum commission rate (1950%) applied

#### cumulativeLidoFees Lower Bound

**Validation:** `newCumulativeFees >= previousCumulativeFees`

**Rationale:** As a cumulative counter, fees must be monotonically increasing.

#### liabilityShares Upper Bound

**Decision:** No check required.

**Rationale:** Overstated `liabilityShares` only temporarily freezes excess funds, which self-corrects with the next accurate report.

#### liabilityShares Lower Bound

**Implementation:** Already enforced in [VaultHub.sol](https://github.com/lidofinance/core/blob/c3401b863ca1eb591ca9de3017ffc209b3a50947/contracts/0.8.25/vaults/VaultHub.sol#L1028):

```solidity
uint256 liabilityShares_ = Math256.max(_record.liabilityShares, _reportLiabilityShares);
```


#### slashingReserve Upper Bound

**Decision:** No check required.

**Rationale:** A malicious oracle can block withdrawals either by:
- Inflating `slashingReserve` (without check)
- Causing report reversion (with check)

Both scenarios prevent withdrawals equally, making the check ineffective.

#### slashingReserve Lower Bound

**Implementation:** Already enforced in [VaultHub.sol](https://github.com/lidofinance/core/blob/cc307168ae939f4af001189af0e93a923b26a4b1/contracts/0.8.25/vaults/VaultHub.sol#L1006):

```solidity
uint256 minimalReserve = Math256.max(CONNECT_DEPOSIT, _reportSlashingReserve);
```

### totalValue Upper Bound: Critical severity

The most severe attack vector involves inflating `totalValue` to mint unbacked stETH. The challenge is that validator balances cannot be verified on-chain without EIP-4788, making the oracle the sole source of truth.

#### Attack Prerequisites
1. Corrupted core protocol oracle
2. Collusion with at least 2 node operators
3. Available stETH minting capacity

#### Attack Sequence
1. Corrupted oracle submits inflated `totalValue` in Merkle root
2. Colluding vault owners submit on-demand reports with inflated values
3. Vaults mint unbacked stETH based on false reports
4. Attackers withdraw stETH through standard processes of the in-protocol WithdrawalQueue

#### Impact
In the worst case, attackers could steal up to 30% of the core protocol's ETH by creating multiple vaults within default OperatorGrid limits.

### Proposed Solution: Direct vs. Indirect Funding

**Critical Assumption:** Oracle corruption must be detectable for this solution to be effective.

#### Funding Types

**1. Direct Funding**
- Occurs through the `fund()` method
- Verifiable on-chain via ETH transfer to vault contract
- No restrictions on amount
- Immediate availability

**2. Indirect Funding**
- Includes validator consolidations, beacon chain deposits
- Cannot be verified on-chain
- Subject to 3-day quarantine period
- Exception: increases ≤ 3.5% of previous `totalValue`

#### Quarantine Mechanism

The quarantine approach prevents sybil attacks that would bypass daily limits. However, it creates a UX challenge for legitimate rewards.

**Solution:** Allow immediate access to normal validation rewards (up to 3.5% increase) while quarantining larger increases.

**Example:**
- Previous `totalValue`: 10,000 ETH
- New `totalValue` with rewards: 10,300 ETH (3% increase)
- Result: No quarantine, immediate availability
- If increase > 3.5%: Amount above 3.5% enters quarantine 

#### Justification for 3.5% Threshold

**For Legitimate Users:**
The 3.5% threshold represents annual rewards for a vault reporting once per year with >22.5M ETH staked. This ensures normal operations remain unaffected.

**Attack Mitigation:**
Limiting immediate increases to 3.5% significantly reduces attack profitability:

**Attack Economics:**
- Required capital: 3,000,000 ETH
- Maximum extraction: 3,000,000 × 0.035 × 0.9 (RR) = 94,500 ETH
- Return: 3.15% in one day

While this represents accelerated annual returns, the massive capital requirement and associated risks make the attack economically unattractive.

## Technical Specification

### inOutDelta On-Chain Cache

The `inOutDelta` parameter tracks the net flow of funds in and out of a vault. To ensure trustless validation, this value is cached on-chain using the `RefSlotCache` library.

#### Data Structure

```solidity
struct Int104WithCache {
    int104 value;           // Current inOutDelta
    int104 valueOnRefSlot;  // inOutDelta at last reference slot
    uint48 refSlot;         // Reference slot number
}
```

#### Implementation

**VaultHub Contract:**
- Maintains two `Int104WithRefSlotCache` objects for the last two reference slots
- Updates cache via `withValueIncrease()` on fund movements
- Retrieves current values using `getCurrentValue()`

**LazyOracle Contract:**
- Accesses historical values via `getCurrentValueOnRefSlot()`
- Validates reports against cached on-chain state

This design eliminates oracle trust requirements for `inOutDelta` validation.

### Implementation of Sanity Checks

#### Total Value Increase Validation

**Direct Funds Verification:**
```solidity
uint256 onchainTotalValueOnRefSlot = uint256(
    int256(uint256(record.report.totalValue)) + 
    _inOutDeltaOnRefSlot - 
    record.report.inOutDelta
);
```

Direct funds update `inOutDelta` on-chain, enabling trustless verification without oracle dependency.

**Indirect Funds Quarantine:**
```solidity
uint256 quarantineThreshold = onchainTotalValueOnRefSlot * 
    (TOTAL_BASIS_POINTS + maxRewardRatioBP) / TOTAL_BASIS_POINTS;
```

Indirect funds exceeding `maxRewardRatioBP` (initially 3.5%) enter quarantine to prevent manipulation while allowing normal rewards processing.

**Quarantine Flow Example:**

| Time | Event | Active Value | Quarantined | Queue |
|------|-------|--------------|-------------|-------|
| T0 | Initial state | 100 ETH | - | - |
| T1 | +50 ETH indirect | 100 ETH | 50 ETH | - |
| T2 | +70 ETH indirect | 100 ETH | 50 ETH | 70 ETH |
| T3 | First quarantine expires | 150 ETH | 70 ETH | - |
| T4 | Second quarantine expires | 220 ETH | - | - |

Direct funds via `fund()` bypass quarantine entirely due to on-chain verification.

#### Total Value Underflow Prevention

**Vulnerability:** Integer underflow during type casting can produce extremely large values:
```solidity
uint256(int256(-1)) = 2^256 - 1
```

**Protection:**
```solidity
if (int256(totalValueWithoutQuarantine) + 
    currentInOutDelta - 
    inOutDeltaOnRefSlot < 0) {
    revert UnderflowInTotalValueCalculation();
}
```

#### Cumulative Fee Validation

**Lower Bound (Monotonicity):**
```solidity
uint256 previousCumulativeLidoFees = 
    obligations.settledLidoFees + obligations.unsettledLidoFees;
    
if (previousCumulativeLidoFees > _cumulativeLidoFees) {
    revert CumulativeLidoFeesTooLow(
        _cumulativeLidoFees, 
        previousCumulativeLidoFees
    );
}
```

**Upper Bound (Rate Limiting):**
```solidity
uint256 maxLidoFees = 
    (vaultsDataTimestamp - record.report.timestamp) * 
    maxLidoFeeRatePerSecond;
    
if (_cumulativeLidoFees - previousCumulativeLidoFees > maxLidoFees) {
    revert CumulativeLidoFeesTooLarge(
        _cumulativeLidoFees - previousCumulativeLidoFees, 
        maxLidoFees
    );
}
```

## Rationale

### Design Decisions

**1. Quarantine Period vs. Daily Limits**

Daily limits were rejected because they can be circumvented through sybil attacks. The quarantine mechanism provides stronger protection while maintaining usability for legitimate users.

**2. 3.5% Threshold Selection**

This threshold balances security with user experience:
- High enough to cover normal staking rewards without delays
- Low enough to limit attack profitability
- Based on empirical data from Ethereum staking yields

**3. No Upper Bound for liabilityShares**

While overstated liabilityShares can freeze funds, this is self-correcting and doesn't create systemic risk, making additional checks unnecessary overhead.

**4. Direct Fund Verification**

Leveraging on-chain `inOutDelta` tracking eliminates oracle trust requirements for direct deposits, significantly reducing attack surface.

### Alternative Approaches Considered

1. **Full on-chain verification**: Requires EIP-4788 implementation, not currently available
2. **Reputation-based limits**: Complex to implement and vulnerable to long-term attacks
3. **Fixed ETH limits**: Poor user experience and doesn't scale with protocol growth

## Test Cases

### Scenario 1: Normal Operations
```
Given: Vault with totalValue = 1000 ETH
When: Oracle reports new totalValue = 1030 ETH (3% increase)
Then: No quarantine applied, funds immediately available
```

### Scenario 2: Large Indirect Deposit
```
Given: Vault with totalValue = 1000 ETH
When: Oracle reports new totalValue = 1100 ETH (10% increase)
Then: 35 ETH immediately available, 65 ETH enters 3-day quarantine
```

### Scenario 3: Direct Funding
```
Given: Vault receives 100 ETH via fund()
When: Oracle reports corresponding totalValue increase
Then: Full amount immediately available (verified on-chain)
```

### Scenario 4: Fee Validation
```
Given: Previous cumulativeLidoFees = 100 ETH, 1 day elapsed
When: Oracle reports cumulativeLidoFees = 116 ETH
Then: Report accepted (increase < 0.18 ETH/second * 86400 seconds)
```

### Scenario 5: Underflow Prevention
```
Given: Report with totalValue that would cause negative intermediate value
When: Contract calculates dynamic totalValue
Then: Transaction reverts with UnderflowInTotalValueCalculation error
```

## Security Considerations

### Known Risks

1. **Oracle Corruption Detection Dependency**: The solution's effectiveness relies on detecting oracle compromise. Without detection, the quarantine mechanism cannot prevent attacks.

2. **3-Day Attack Window**: Even with quarantine, a sophisticated attacker with patient capital could extract 3.5% per report cycle.

3. **Implementation Complexity**: The dual funding mechanism adds complexity that must be carefully managed in user interfaces.

### Mitigations

- Continuous accounting oracle monitoring and anomaly detection
- GateSeal mechanism for the prompt emergency mode response
- Clear user communication about quarantine periods
- Multiple rigorous security audits of implementation

## Conclusion

This proposal establishes a comprehensive framework for validating stVault oracle reports through:

1. **Differentiated funding flows**: Direct (verifiable) vs. indirect (quarantined)
2. **Parameter-specific validations**: Tailored checks for each vault parameter
3. **Economic disincentives**: Making attacks unprofitable through capital requirements

The approach balances security with usability, protecting the protocol from catastrophic attacks while maintaining smooth operations for legitimate users. Implementation should proceed with careful attention to user experience and comprehensive testing of edge cases.
