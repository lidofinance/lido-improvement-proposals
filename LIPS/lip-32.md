---
lip: 32
title: Sanity Checks for stVaults
status: WIP
author: Alexandr Drygin, Greg Shestakov, Victor Petrenko
discussions-to: TBD
created: 2025-09-01
updated: 2025-09-13
---

# LIP-32: Sanity Checks for stVaults

## Simple Summary

This proposal defines sanity checks for stVaults oracle reporting in Lido V3, establishing safeguards against malicious oracle behavior through on-chain validation of vault parameters and introducing a quarantine mechanism for indirect funding to prevent potential attacks.

## Abstract

Lido V3 introduces stVaults as part of a broader staking infrastructure platform, where independent operators manage validators via vault-specific withdrawal credentials contracts while minting stETH backed by their stake. As described in [LIP-31](./lip-31.md), the protocol implements a global accounting system through `VaultHub` - a central collateral registry that tracks each vault's total value and manages external shares minting/burning. This LIP proposes comprehensive sanity checks for the "lazy oracle" system (centered around the `LazyOracle` contract) that reports vault parameters to ensure the integrity of this global accounting framework.

The proposal introduces two funding flow types: direct funding through the `fund()` method (verifiable on-chain), and indirect funding which requires a 3-day quarantine period (with an exception for increases less than 3.5% of vault total value). These mechanisms, combined with parameter validation for `totalValue`, `cumulativeLidoFees`, `liabilityShares`, and `slashingReserve`, create a robust defense against potential oracle manipulation that could compromise the global backing invariant: `stETH.totalSupply() ≤ Core Pool total supply + Σ Staking Vault locked`.

## Motivation

With the introduction of stVaults in Lido V3, the protocol implements a global accounting system where external ether sources can mint stETH through over-collateralization. As outlined in [LIP-31](./lip-31.md), vault parameters are reported through a LazyOracle system where:

- The extended Accounting Oracle publishes Merkle roots of per-vault balances
- `VaultHub` verifies inclusion proofs asynchronously, enabling scalable vault management
- Vault owners submit on-demand reports with Merkle proofs for operations requiring updated vault state through `LazyOracle`

This architecture, while enabling efficient scaling, creates potential attack vectors if the oracle committee is compromised or software is malfunctioning. A misbehaving oracle could manipulate vault parameters to:

- Enable minting of unbacked stETH by inflating `totalValue`, violating the global backing invariant
- Allow withdrawal of collateral by understating `liabilityShares` (external shares minted)
- Avoid penalty payments by manipulating fee and reserve parameters
- Bypass reserve-breach hooks that should trigger when vault buffers drop below RR or FRT thresholds

The most critical risk is that a misbehaving oracle, in collusion with node operators, could potentially steal up to 30% of ETH from the Lido protocol by creating vaults with artificially inflated values, given the global external shares minting limits.

This proposal addresses these risks by implementing comprehensive sanity checks and a quarantine mechanism for funds that cannot be verified on-chain immediately, ensuring the integrity of the global accounting system.

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

### Lazy Oracle Architecture and Global Accounting Integration

As described in [LIP-31](./lip-31.md), the global accounting system implements a "lazy oracle" mechanism (centered around the `LazyOracle` contract) to enable scalable vault management:

1. **Extended Accounting Oracle**: Publishes Merkle roots containing per-vault state (`totalValue`, `cumulativeLidoFees`, `liabilityShares`, `slashingReserve`) 
2. **VaultHub Integration**: Acts as the central collateral registry, verifying inclusion proofs asynchronously
3. **On-demand Reports**: Vault owners submit Merkle proofs when operations require updated state (minting, burning, withdrawals)

This architecture maintains the global backing invariant (`stETH.totalSupply() ≤ Core Pool total supply + Σ Staking Vault locked`) while avoiding unbounded on-chain loops. The LazyOracle contract validates these reports against on-chain state to ensure consistency with the global accounting framework.

### Vault Parameters and Attack Vectors

Each vault report includes four critical parameters that require validation: 

#### 1. totalValue

**Definition:** Sum of execution layer (EL) and consensus layer (CL) balances, including pending deposits and withdrawals.

**Attack vectors:**
- **Overstatement:** Enables minting of unbacked stETH by inflating the vault's locked value, violating the global backing invariant
- **Understatement:** May trigger inappropriate forced rebalance when the vault falsely appears to breach the Force Rebalance Threshold (FRT) 

#### 2. cumulativeLidoFees

**Definition:** Cumulative fees owed to Lido, denominated in ETH.

**Attack vectors:**
- **Overstatement:** Forces vault owners to pay excessive fees
- **Understatement:** Allows fee avoidance

#### 3. liabilityShares

**Definition:** External shares minted against the staking vault position, representing the vault's stETH liability that rebases with the token.

**Attack vectors:**
- **Overstatement:** 
    - Freezes excess funds by inflating the vault's recorded liability
    - Blocks legit disconnect from the VaultHub and Lido protocol (i.e., can block an exit escape hatch for vaults)
- **Understatement:** 
    - Enables premature collateral withdrawal, potentially violating the reserve ratio (RR) requirements
    - Allows premature disconnect from the VaultHub and Lido protocol

#### 4. maxLiabilityShares

**Definition:** Maximum external shares minted against the staking vault position within the latest AccountingOracle-reported frame (between reference slots). This value is used to calculate the vault's locked amount and prevent manipulation of collateral requirements.

**Invariant:** Greater or equal than `liabilityShares` and currently stored on-chain `maxLiabilityShares`

**Purpose:** 
- Tracks the peak liability within an oracle period to ensure proper collateralization
- Used in locked value calculation
- Prevents vault owners from manipulating locked requirements through rapid mint/burn cycles (e.g., within a single transaction)

**Attack vectors:**
- **Overstatement:** 
    - Freezes excess funds by inflating the vault's recorded max liability per frame
    - Blocks legit disconnect from the VaultHub and Lido protocol (i.e., can block an exit escape hatch for vaults)
- **Understatement:** 
    - Enables premature collateral withdrawal, potentially violating the reserve ratio (RR) requirements
    - Allows premature disconnect from the VaultHub and Lido protocol

#### 5. slashingReserve

**Definition:** Locked assets reserved for slashing penalties.

**Attack vectors:**
- **Overstatement:** 
    - Unnecessarily locks vault funds
    - Blocks legit disconnect from the VaultHub and Lido protocol (i.e., can block an exit escape hatch for vaults)
- **Understatement:** 
    - Allows penalty avoidance during slashing events
    - Allows premature disconnect from the VaultHub and Lido protocol

### Proposed Sanity Checks

Based on the identified attack vectors, we propose the following validation mechanisms: 

#### totalValue Upper Bound

The upper bound for `totalValue` requires special consideration due to the critical attack vector it addresses. See the [detailed analysis](#totalvalue-upper-bound-critical-severity) below.

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

**Rationale:** Overstated `liabilityShares` only temporarily freezes excess funds, which self-corrects with the next accurate report, also check is partially implemented together with `maxLiabilityReport` checks in LazyOracle:

```solidity
if (_reportedMaxLiabilityShares < _reportedLiabilityShares 
    || _reportedMaxLiabilityShares < record.maxLiabilityShares) {
    revert InvalidMaxLiabilityShares();
}
```

#### liabilityShares Lower Bound

**Implementation:** Enforce against VaultHub's recorded value:

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

**Implementation:** Enforce against VaultHub's defined `CONNECT_DEPOSIT`:

```solidity
uint256 minimalReserve = Math256.max(CONNECT_DEPOSIT, _reportSlashingReserve);
```

#### maxLiabilityShares Validation

**Definition:** Maximum liability shares minted during the current oracle period, used to calculate locked value.

**Validation:** `_maxLiabilityShares >= _liabilityShares && _maxLiabilityShares >= record.maxLiabilityShares`

**Implementation:** Enforce in LazyOracle:

```solidity
if (_maxLiabilityShares < _liabilityShares || _maxLiabilityShares < record.maxLiabilityShares) {
    revert InvalidMaxLiabilityShares();
}
```

**1-tx Looping Attack Prevention:**

The VaultHub implements a critical protection against rapid mint/burn cycles that could manipulate the locked value:

```solidity
// From VaultHub's per-vault report application:
//
// current maxLiabilityShares will be greater than the report one
// if any stETH is minted on funds added after the refslot
// in that case we don't update it (preventing unlock)
if (_record.maxLiabilityShares == _reportMaxLiabilityShares) {
    _record.maxLiabilityShares = uint96(Math256.max(_record.liabilityShares, _reportLiabilityShares));
}
```

This mechanism prevents the following attack sequence within a single transaction:
1. Bring ETH to vault (increasing totalValue)
2. Mint stETH (increasing locked and maxLiabilityShares)
3. Burn stETH (decreasing liabilityShares but not maxLiabilityShares)
4. Submit oracle report (attempting to reduce maxLiabilityShares)
5. Withdraw ETH (exploiting reduced locked requirement)

By not updating `maxLiabilityShares` when it differs from the reported value, the system maintains the peak collateral requirement until the next oracle period.

**Rationale:** Ensures monotonic increase of max liability shares within an oracle period to prevent manipulation of locked value calculations and maintain proper collateralization.

### totalValue Upper Bound: Critical severity

The most severe attack vector involves inflating `totalValue` to mint unbacked stETH through external shares. The challenge is that validator balances cannot be verified on-chain without [EIP-4788](https://eips.ethereum.org/EIPS/eip-4788), making the oracle the sole source of truth for vault state.

#### Attack Prerequisites
1. Corrupted Accounting Oracle that publishes malicious Merkle roots
2. Available external shares minting capacity within global limits
3. Permissioned stETH minting mode: collusion with vault operators to submit false on-demand reports

#### Attack Sequence
1. Corrupted oracle submits inflated `totalValue` in Merkle root via extended Accounting Oracle
2. Colluding vault owners submit on-demand reports with Merkle proofs to VaultHub
3. VaultHub calculates inflated `locked` value: `locked = inflatedTV × (1 – RR)`
4. Vaults call `mintExternalShares()` to mint unbacked stETH based on false locked values
5. Attackers withdraw stETH through standard WithdrawalQueue processes

#### Impact
In the worst case, attackers could steal up to the stETH minted against the external ether supply (projected value is 30%) of the core protocol's ETH by exploiting the global external shares minting limits, violating the fundamental backing invariant: `stETH.totalSupply() ≤ Core Pool total supply + Σ Staking Vault locked`.

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

### Configurable Parameters

The LazyOracle contract includes several configurable parameters that can be updated via governance:

```solidity
struct Storage {
    uint64 quarantinePeriod;           // Time period for quarantine (max 30 days)
    uint16 maxRewardRatioBP;           // Max reward ratio in basis points (max 655.35%)
    uint64 maxLidoFeeRatePerSecond;    // Max Lido fee rate per second (max 10 ETH/s)
}
```

**Parameter Constraints:**
- `quarantinePeriod`: 0 to 30 days
- `maxRewardRatioBP`: 0 to 65,535 (0% to 655.35%)
- `maxLidoFeeRatePerSecond`: 0 to 10 ETH per second

These parameters can be updated by accounts with the `UPDATE_SANITY_PARAMS_ROLE` through:
```solidity
function updateSanityParams(
    uint256 _quarantinePeriod,
    uint256 _maxRewardRatioBP,
    uint256 _maxLidoFeeRatePerSecond
) external onlyRole(UPDATE_SANITY_PARAMS_ROLE)
```

### Vault Report Parameters

The LazyOracle validates the following parameters in each vault report:

```solidity
function updateVaultData(
    address _vault,
    uint256 _totalValue,                // Total vault value (EL + CL)
    uint256 _cumulativeLidoFees,        // Cumulative fees owed to Lido
    uint256 _liabilityShares,           // Current external shares liability
    uint256 _maxLiabilityShares,        // Peak liability within oracle period
    uint256 _slashingReserve,           // Slashing protection reserve
    bytes32[] calldata _proof           // Merkle proof
)
```

**Parameter Descriptions:**
- `_totalValue`: Sum of execution and consensus layer balances
- `_cumulativeLidoFees`: Monotonically increasing fee counter
- `_liabilityShares`: Current stETH shares minted against vault
- `_maxLiabilityShares`: Maximum shares minted within current oracle period
- `_slashingReserve`: Reserve amount for slashing protection

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
- Accesses historical values via `getValueForRefSlot()`
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

#### Max Liability Shares Validation

**Upper Bound:**
```solidity
if (_maxLiabilityShares < _liabilityShares || _maxLiabilityShares < record.maxLiabilityShares) {
    revert InvalidMaxLiabilityShares();
}
```

## Integration with Global Accounting System

The sanity checks described in this proposal are a critical component of Lido V3's global accounting framework defined in [LIP-31](./lip-31.md). The integration ensures that:

### 1. Maintaining the Global Backing Invariant

All sanity checks ultimately protect the fundamental invariant:
```
stETH.totalSupply() ≤ Core Pool total supply + Σ Staking Vault locked
```

By validating `totalValue` increases, the system prevents vaults from inflating their `locked` amount (`locked = TV × (1 – RR)`), which would allow minting unbacked external shares.

### 2. VaultHub Coordination

The LazyOracle reports are processed by VaultHub, which:
- Maintains the on-chain ledger of vault states
- Enforces minting limits based on validated `totalValue` and reserve ratios
- Triggers reserve-breach hooks when vaults fall below RR or FRT thresholds
- Coordinates rebalancing operations when needed

### 3. External Shares Lifecycle

The sanity checks ensure proper external shares accounting:
- **Minting**: Validated `totalValue` determines maximum mintable external shares
- **Burning**: Accurate `liabilityShares` tracking ensures proper debt reduction
- **Rebalancing**: Quarantine mechanism prevents rapid extraction through inflated values

#### maxLiabilityShares Lifecycle

The `maxLiabilityShares` parameter plays a crucial role in maintaining collateral integrity throughout the external shares lifecycle:

1. **Initial State**: Set to 0 when vault is connected
2. **During Minting**: Updated to the new liability amount whenever shares are minted
3. **Oracle Period Tracking**: Maintains the peak liability within each oracle reporting period
4. **Report Processing**: 
   - If on-chain `maxLiabilityShares` equals reported value: Update to max(current liability, reported liability)
   - If on-chain value differs: Maintain current value to prevent manipulation
5. **Lock Calculation**: Used to compute `locked = liabilityValue + max(reserve, minimalReserve)` where liabilityValue is derived from `maxLiabilityShares`

This mechanism ensures that vaults cannot reduce their collateral requirements by rapidly minting and burning shares within the same oracle period.

### 4. Oracle Trust Minimization

The combination of on-chain `inOutDelta` tracking and quarantine mechanisms reduces trust in the oracle:
- Direct funds are fully verifiable without oracle input
- Indirect funds face time delays, limiting attack profitability
- Multiple oracle reports are required for large-scale attacks, increasing detection probability

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

1. **Full on-chain verification**: Requires EIP-4788 implementation, computationally and gas heavy, traversing the whole Ethereum validator set
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
Then: Full amount immediately available (no quarantine due to on-chain verification)
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
- [GateSeal](https://docs.lido.fi/contracts/gate-seal) mechanism for the prompt emergency mode response
- Clear user communication about quarantine periods
- Multiple rigorous security audits of implementation

## Conclusion

This proposal establishes a comprehensive framework for validating stVault oracle reports through:

1. **Differentiated funding flows**: Direct (verifiable) vs. indirect (quarantined)
2. **Parameter-specific validations**: Tailored checks for each reported vault parameter
3. **Economic disincentives**: Making attacks unprofitable through capital requirements
4. **Global accounting integration**: Ensuring consistency with the VaultHub-based architecture

The approach balances security with usability, protecting the protocol from catastrophic attacks while maintaining smooth operations for legitimate users. Implementation should proceed with careful attention to user experience and comprehensive testing of edge cases.

## References

- [LIP-31: Expanding stETH liquidity layer with over-collateralized minting](./lip-31.md) - Defines the global accounting framework and VaultHub architecture