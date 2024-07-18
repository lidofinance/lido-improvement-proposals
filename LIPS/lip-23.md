---
lip: 23
title: Negative rebase sanity check with second opinion
status: Proposed
author: Alexey Potapkin, Greg Shestakov, Eugene Pshenichnyi, Victor Petrenko
discussions-to: https://research.lido.fi/t/lip-23-negative-rebase-sanity-check-with-second-opinion/7543
created: 2024-04-17
updated: 2024-06-18
---

# LIP-23: Negative rebase sanity check with second opinion

## Abstract

Improve the safety check for the accounting report in the case of a negative rebase, reducing the possible impact size, but with the requirement for a second opinion for extreme cases. As a result impact size of current sanity check reduced by 26 times to 0.19% per day.

## Motivation

The `AccountingOracle` contract is a fundamental component of the Lido protocol that delivers, amongst other data, the aggregate of all Lido validators' Beacon Chain balances (`clBalance` in the report) to the protocol, thereby facilitating the daily rebase of the stETH token. The accuracy of this value is critical to the integrity of the protocol, and it relies on a committee of independently operated Oracle daemons in a 5-of-9 configuration. The protocol could be harmed if this committee is compromised, malfunctions, or colludes. This risk is acknowledged and constrained by a sanity check that restricts the possible discrepancy in balance that Oracle can report.

The current approach to sanity checking allows the Oracle committee to bring up to a 5% reduction of TVL in each report. Given that the governance reaction time has a lower bound of 72 hours, the malicious or compromised Oracles could reduce reported TVL by 15-20%, invoking mass liquidations on lending markets and dropping the price of stETH. However, a real negative rebase has very distinct features that we can take into account to reduce the impact and attack surface while allowing frictionless operation of the protocol even during a mass slashing event.

While a high-precision check is possible using zero-knowledge technologies and EIP-4788, we decided to start with the simplier solution. It limits the impact in a simple but effective way. It privides an easy way to improve the robustness of the solution later.

## Changelog

- 1.2.1 - 2024-07-11: getOracleReportLimits() backward compatibility considerations 
- 1.2.0 - 2024-06-18: Request withdrawal vault balance from second opinion oracle
- 1.1.0 - 2024-06-05: Add paragraph about safe call for checkAccountingOracleReport()
- 1.0.1 - 2024-05-23: Add full forum link to LIP discussion
- 1.0.0 - 2024-05-22: First public version
- 0.0.1 - 2024-04-17: Initial version

## Specification

The proposed sanity check operates on the rebase value of the report

$$
clRebaseValue_{i} = 
clBalance_{i} + withdrawalVaultBalance_{i} - clBalance_{i-1} - clDepositsAppeared_{i}
$$

where

- $clBalance_i$ — the aggregate balance of all Lido validators on Beacon Chain for the i-th report day.
- $clDepositsAppeared_i$ — ether that was deposited to Beacon Chain appeared as new validators in i-th report day.
- $withdrawalVaultBalance_i$ — the amount of ether that was withdrawn from the Beacon Chain to the Lido withdrawal vault on the moment of i-th report day.

and 

$$
negativeClRebaseSum_{18} = \sum_{k=0}^{17} 
\begin{cases}
	 |clRebaseValue_{i - k}| &, if\ clRebaseValue_{i - k} < 0 \\
	 0 &, if\ clRebaseValue_{i - k} \geq 0
\end{cases}
$$

In a way, that if $clRebaseSumNegative_{18} > maxClRebaseNegativeSum_{18}$ the check MUST try to retreive a second opinion on the report values. The check is considered failed if no second opinion is available or the second opinion does not match the report.

$$ maxClRebaseNegativeSum_{18} = maxInitialSlashings_{18} + maxPenalties_{18} $$

$$ maxInitialSlashings_{18} = 1 ETH * (clValidators_i - clValidatorsExited_{i-18}) $$

$$ maxPenalties_{18} = 0.101 ETH * (clValidators_i - clValidatorsExited_{i-54})$$

where

$clValidators_i$ — cumulative number of ever appeared Lido validators on Beacon Chain on the i-th report

$clValidatorsExited_i$ - cumulative number of Lido validators ever exited from Beacon Chain on the i-th report

>NOTE 1: In the case if $i-18$ or $i-54$ report is missing, it's safe to use the closest earlier report to construct $maxClRebaseNegativeSum_{18}$.

>NOTE 2: During the bootstrap period, while historical data is not available, it's recommended to use 
$maxInitialSlashings_{18} = 1 ETH * clValidators_i$ until $i-18$ th report is available, and $maxPenalties_{18} = 0.101 ETH * clValidators_i$ until ${i-54}$ th report is available. 

### Second opinion

To start, it is proposed that there be no second opinion source but the interface to plug the desired provider later. The desired second opinion provider would be a ZKP-based oracle, like the ones proposed on the forum once they are ready: 

-   [[ZKLLVM] Trustless ZK-proof TVL oracle](https://research.lido.fi/t/zkllvm-trustless-zk-proof-tvl-oracle/5028)
-   [DendrETH: A trustless oracle for liquid staking protocols](https://research.lido.fi/t/dendreth-a-trustless-oracle-for-liquid-staking-protocols/5136)
-   [ZK Lido Oracle powered by Succinct](https://research.lido.fi/t/zk-lido-oracle-powered-by-succinct/5747)

Other options, like 3rd party Oracle comittee or even a multisig-controlled manual quasi-oracle, may be considered. The Oracle interface must be implemented as: 
```
interface SecondOpinionOracle { 
	function getReport(uint256 refSlot) external view returns ( 
		bool success, 
		uint256 clBalanceGwei, 
		uint256 withdrawalVaultBalanceWei, 
		uint256 totalDepositedValidators, 
		uint256 totalExitedValidators 
	); 
}
```
> NOTE 1: All parameters in the interface are mandatory, though not all are used in the proposed sanity check implementation. The intention is to enable the more extensive use of ZK-based data in future checks.

> NOTE 2: [Withdrawal Vault](https://docs.lido.fi/contracts/withdrawal-vault) balance is essential for the correct sanity check. Without it, in case of oracle collusion, it is possible to "hide" funds in the Withdrawal Vault contract. Such a situation could lead to a negative rebase, which would not be detected by the sanity check.

### Matching

As we're considering ZKP-based oracles as a baseline for the second opinion provider, the inherent limitations should be considered. We know from the proposals on the forum that the inclusion criterion for a validator to be considered belonging to Lido differs from Lido Oracles and may result in an error, given that anyone can spawn a validator with Lido's withdrawal credentials. Hence, ZK-proved values can indicate only the upper limit at this stage.

However, as soon as such an attack costs ether, we can assume that for `clBalance`, there is a lower boundary defined by the economic viability of the attack.

So, there MUST be a parameter for the sanity check, `clBalanceErrorUpperLimit`, that can define the error tolerance level for `clBalance` matching.

## Rationale

### New sanity check value 
We assume that the sanity check value should allow every validator to be slashed and penalized for the whole window length, without sanity check triggering. That means that the theoretical maximal negative rebase equals all validators which are not exited from CL multiplied by the maximal amount of penalties per validator. 
Within the formula, we calculated the total number of validators which are still in CL and assumed that all of them were slashed and inactive for the whole 18-days window length period, along with all validators exited before the window are also slashed and add attestation penalties if slashing happened less than 36 days before the window start. Midterm slashing penalty and inactivity leak are not taken into account and would require a second opinion. [More on the calculation part here](https://docs.google.com/document/d/1e3R2lAl4xXm0vwqNFB02tvv57jjza9gK9zLkM7Jt1Ow/edit)

As a result maximal decrease per validator in 18-days window equal to 1.101 ETH (0.101 ETH for attestation penalties and 1 ETH for initial slashing penalty). Compared to the existing mechanism such improvement lowers the current sanity check by aproximately 26 times from a 5% decrease per day to a 0.19% decrease per day.


## Security considerations

### Dual Governance clashing

Considering the current design of the [Dual governance](https://research.lido.fi/t/dual-governance-design-and-implementation-proposal/7131), it is possible that the protocol will face a deadlock. If the protocol is in a rage quit state, votes cannot be executed unless all stETH is withdrawn from the rage quit contract. Withdrawal requires an Oracle report, so if something (for example, extremely high midterm slashing penalties) would cause the trigger of the sanity check, the protocol enters a dead-lock situation, where stETH can not be withdrawn due to the lack of Oracle reports, Oracle reports do not pass due to sanity check, and the sanity check cannot be changed since votes are blocked by Dual Governance veto.


### No second opinion in the beginning

Possible risks are divided in a way that negative events that can happen swiftly fit under the limits and happen freely without triggering this sanity check, but larger ones, being extremely rare, can be predicted and give DAO time to react: 18 days in case of large correlated slashing and, at least, 4 days — in the case of the mass inactivity leak. So, starting without a second opinion may be considered safe until Dual Governance is implemented.

### Missed reports 

In case Oracle reports are missed for some reason, the calculation window may be increased (for example 3 days of missed reports will one-time add 3 days to the calculation window). Since the proposed design takes into account cumulative validators set, such an event may trigger a false positive sanity check, in case all validators are slashed on day x-18, and reports are missed on day x-1,x,x+1 (while no new validators are being added for at least 36 days before such scenario). 

In this case, the amount of accumulated attestation penalties would be higher than 0.101 ETH per validator, but since this amount of penalties is calculated for a validator set of 100000 validators, for a bigger set such a scenario wouldn't trigger a false positive sanity check. Once a trustless second opinion is added, this consideration will be fully mitigated.
### clValidators forging

In this sanity check we rely on $clValidators$ value that Oracle brings with each report. But if oracle committee is colluded they can bring any values in this field. 
Therefore, it's not an issue, because $clValidators$ is capped by the number of validators deposited by Lido protocol, which is fixed onchain and their lower bound is the previous report value. So, it's not exploitable from this side. 

### MVI changes 

Values in this doc are calculated considering current Ethereum issuance and vote weights; in the case of MVI implementation, values should be recalculated to reflect changes.

## Backward compatibility

There is no need for any changes in the Oracle daemon. It MUST work as intended without modification and explicit knowledge about additional sanity checks. It will be able to utilize fastlane mechanics and reach a consensus but will fail to submit report data in the case of a significant decrease until the second opinion is ready. But it will retry until it finally succeeds. However, it can be optimized to avoid this polling loop and reduce resource utilization.

However, new sanity checker implies changes in the report limits. The off-chain [oracle daemon](https://github.com/lidofinance/lido-oracle) consumes the limits via the `getOracleReportLimits()` function. For that reason, `oneOffCLBalanceDecreaseBPLimit` should be marked as deprecated, and all new parameters should be added at the end of the list.

Also, proposed approach does not require any additional data from Beacon Chain, so oracles are not to be modified. 

The `checkAccountingOracleReport()` function for `OracleReportSanityChecker` is currently a view function. With the planned improvements, it is neccessary to remove view modifier from the function to make on-chain writes to keep track of the reports over the timespan. It is decided to not change the corresponding `IOracleReportSanityChecker` interface in the `Lido` contract because the contract is written with Solidity 0.4.24 which makes view and non-view calls the same in the resulting EVM bytecode. However, calls to the `checkAccountingOracleReport()` need to be restricted to the Lido contract. It will prevent malicious interference with report data used for rebase calculations.

## Links

- [Accounting oracle specification](https://docs.lido.fi/guides/oracle-spec/accounting-oracle)
- [Lido oracle report handling](https://docs.lido.fi/contracts/lido#oracle-report)
- [[ZKLLVM] Trustless ZK-proof TVL oracle](https://research.lido.fi/t/zkllvm-trustless-zk-proof-tvl-oracle/5028)
- [DendrETH: A trustless oracle for liquid staking protocols](https://research.lido.fi/t/dendreth-a-trustless-oracle-for-liquid-staking-protocols/5136)
- [ZK Lido Oracle powered by Succinct](https://research.lido.fi/t/zk-lido-oracle-powered-by-succinct/5747)

