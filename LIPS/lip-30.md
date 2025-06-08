---
lip: 30
title: Extended ether backing for stETH
status: WIP
author: Alexey Potapkin, Eugene Mamin, Eugene Pshenichnyi
discussions-to: TBA
created: 2024-12-06
updated: 2025-06-08
---

# Extended ether backing for stETH

## Simple Summary

Introduce overcollaterized accounting system for the stETH, enabling backing via ether supply outside of the Lido Core staking pool.

The stETH token improves collaterilization to manage slashing risks, remaining fungible and redeemable 1:1 via primary market.

## Motivation

The current implementation of the stETH accounting system presumes that stETH gets minted 1:1 against the ether submitted to the Lido Core pool which means that stETH fungibility layer is tightly coupled with the Lido Core pool validator set, its parameters, policies, and state.

Given that staking setups have become more nuanced, customizable, and fine-tunable, this proposal describes the accounting system where stETH fungibility layer extended beyond Lido Core pool validator set only to enable external staking setups contribute to the total stETH token supply.

This approach broaden options and future compatibility: new direct integrations with node operators, use cases for institutions, and products of the emerging staking flavors and technologies that can enjoy stETH liquidity layer access.

## Abstract

The Lido on Ethereum protocol currently tracks pooled ether under its direct management — buffered ether, validators’ balances on the Consensus Layer (CL), and transient validator states — forming the protocol's ["total pooled ether"](https://docs.lido.fi/contracts/lido#gettotalpooledether).

This proposal extends the accounting model by including an additional ether supply external to the protocol’s direct purview (aside of the Lido Core staking pool) for stETH, allowing customized and isolated staking approaches beyond the established ones via Staking Router and corresponding Staking Modules. The changes are additive to the currently existing Lido version and extends the operations of minting, burning, redeeming, reward and penalties handling, and fee distribution. 

### High-level token design concepts

#### Design principles to uphold

Three main properties of the stETH token are to be preserved.

#### Collaterilization

External ether as backing: every minted stETH has underlying ether source, either inside the Lido Core pool or external ether locked (i.e., acting as the external stETH minting collateral) by the protocol. New accounting model suggests overcollatalized minting approach for the external part of the token supply to manage slashing risks against the relaxed staking setup requirements behind this part of collateral.

#### Redemability

Every minted stETH must be redeemable either Lido Core pool, or the external minting collateral. 

This collateral must be potentially accessible by the protocol under certain conditions (e.g., collateral shortage or Lido Core pool depletion), allowing write off minted stETH external ether source liability by moving collateralized ether to the Lido Core pool in 1:1 mannder.

#### Fungibility

The stETH token contract remains to be ERC-20 deployed under the same address on Ethereum mainnet.

Minting or burning new stETH does not trigger stETH token rebase, rebases remain solely driven by the establed `AccountingOracle` report lifecycle as a part of the main phase of the report. Rewards and penalties accrues for stETH rebase remain to be defined by the Lido Core pool validator set (i.e., stETH minted against the external minting collateral doesn't change the embedded stETH staking rebase rate or, more generally, its APR). 

### Key accounting changes

> For simplicity, 'external ether' means 'external stETH minting ether collateral', and 'external shares' are stETH shares minted against that ether source.

#### Total pooled ether update

The total pooled ether (`totalPooledEther`) considering the external balance becomes:

```math
\text{totalPooledEther} = \text{bufferedEther} + \text{CLbalance} + \text{transientEther} + \text{externalEther}.
```

If we denote ether supply under the Lido Core pool as `internalEther`:

```math
\text{internalEther} = \text{bufferedEther} + \text{CLbalance} + \text{transientEther}
```

```math
\text{internalShares} = \text{totalShares} - \text{externalShares}
```

and take into account that stETH share rate is defined by the Lido Core pool:

```math
\text{shareRate} = \frac{\text{totalPooledEther}}{\text{totalShares}} = \frac{\text{internalEther}}{\text{internalShares}}
```

Then:

```math
\text{externalEther} = \text{externalShares} \times \text{shareRate} = \frac{\text{externalShares} \times \text{internalEther}}{\text{internalShares}}
```

NB: Any changes in `shareRate` affect `externalEther` as a part of the regular stETH token rebase induced by `AccountingOracle`.
Therefore, the external ether source must must maintain the reserve sufficient to update the total collaterized ether amount as a part of the token rebase.

#### Mint external shares

A newly introduced "external ether accounting contract" (trusted by the [`Lido/stETH`](https://docs.lido.fi/contracts/lido) protocol contract) is allowed to mint stETH shares to a specified recipient.

The amount of external stETH shares minted MUST be fully backed via an equivalent external ether amount locked by the external ether accounting contract.

> NB: The mint operation MUST NOT incur the stETH token rebase.

Upon external shares minting the `Lido` contract increases the stored `externalShares`, i.e., denoting:

- $\Delta ETH_{\text{ext}}$ as the external ether amount to be used for minting stETH,
- $\Delta S_{\text{ext}}$ as the number of external shares to be minted and backed by $\Delta ETH_{\text{ext}}$,
- $\text{shareRate} = \frac{\text{totalPooledEther}}{\text{totalShares}}$ as the current stETH in-protocol share rate maintaining `1 stETH = 1 ETH` submit and redemption target balance,

one can conclude the relationship between incremental external shares and ether amounts is given by:

```math
\Delta ETH_{\text{ext}} = \Delta S_{\text{ext}} \times \text{shareRate}.
```

Inversely, if the external ether contribution $\Delta ETH_{\text{ext}}$ is known:

```math
\Delta S_{\text{ext}} = \frac{\Delta ETH_{\text{ext}}}{\text{shareRate}}.
```

After minting, the external shares value gets updated as:

```math
\text{externalShares}_{\text{new}} = \text{externalShares}_{\text{old}} + \Delta S_{\text{ext}}
```

#### Burn external shares

The "external accounting contract" can also burn a specified amount of stETH shares from its balance to unlock the corresponding external ether collateral,
decreasing the stored `externalShares`, i.e., denoting:

- $\Delta S_{\text{ext, burn}}$ as the number of external shares to burn:

```math
\Delta S_{\text{ext, burn}} = \frac{\Delta ETH_{\text{ext, burn}}}{ \text{shareRate}}.
```

Thus, the external shares value becomes:

```math
\text{externalShares}_{\text{new}} = \text{externalShares}_{\text{old}} - \Delta S_{\text{ext, burn}}.
```

#### Rebalance

Rebalance allows writing off `externalShares` by moving the corresponding $\Delta ETH_{\text{ext}}$ external ether to the Lido Core pool without token rebase and forcing the corresponding shares amount to become internal:

Rebalance invariants:
- 1) Total pooled ether amount remains the same:
```math
totalPooledEther_{old} = totalPooledEther_{new}
```
- 2) Total minted shares amount remains the same:
```math
totalShares_{old} = totalShares_{new}
```

To rebalance $\Delta ETH_{\text{ext}}$ external ether:

Internal ether amount updates by moving ether to the Lido Core pool:
```math
internalEther_{new} = internalEther_{old} + \Delta ETH_{\text{ext}}
```
External shares decrease
```math
externalShares_{new} = externalShares_{old} - \frac{\Delta ETH_{\text{ext}}}{shareRate}
```
Internal shares increase to the same amount
```math
internalShares_{new} = internalShares_{new} + \frac{\Delta ETH_{\text{ext}}}{shareRate}
```

Rebalance can be used to improve the collateral reserve during its shortfall, e.g., to handle penalties and slashings.

#### Lock and unlock

To mitigate the risks of collateral shortage due to token rebases, penalties and slashings, additional reserve is locked upon minting as a part of the overall collateral.

This reserve depends on the risk approach used and, in general, must be enough to cover normal protocol operation and non-catastrophic slashings for the external ether used to back minted stETH.

#### Oracle reports

Accounting oracle report happen in the same cadence.

Token rebase implements in a way that `externalEther` gets updated according to the newly delivered Lido Core pool changes after reward and fees distributed.

##### Rewards and fees



### Failure modes

#### Slashing handling

#### Redemption of the depleted pool

#### Bad debt handling

### Security considerations

### Backward compatibility

### Test cases

### Total pooled ether

### stETH rebase rewards

### stETH rebase fees

### External ether update

### Risks addressed

TBA

### Risks to be managed

TBA

## Specification

### Mint external shares

#### Function: `mintExternalShares(address receiver, uint256 sharesAmount)`

    Input:
        `receiver`: The address that will receive the newly minted external stETH.
        `sharesAmount`: The number of shares to mint.
    Actions:
        - Compute the corresponding external ether amount `deltaExternalBalance` at the current stETH's `shareRate`.
        - Increase `externalBalance` by `deltaExternalBalance`.
        - Mint `sharesAmount` of stETH to receiver.

### Burn external shares

#### Function: `burnExternalShares(uint256 sharesAmount)`

    Input:
        `sharesAmount`: The number of shares to burn from the caller’s balance.
    Actions:
        - Compute the external ether redemption ΔETHext, burnΔETHext, burn​.
        - Decrease `externalBalance` by ΔETHext, burnΔETHext, burn​.
        - Burn `sharesAmount` of stETH from the caller’s balance.

### 

## Rationale

### Abstracting external ether

The design does not require the protocol to 'understand' the nature of external ether. It only requires trusted external accounting, ensuring that the external ether source is credible. The external ether could represent aggregator-controlled validator balances or, potentially, other complex Ethereum and broader ecosystem mechanisms.

### Why track external shares instead of external ether directly

stETH is defined by its share mechanics, and all protocol logic revolves around shares and their rate. By expressing external contributions and withdrawals in terms of stETH shares, the protocol remains consistent. Ether is only an input or output measure, while shares define the in-protocol liquidity and distribution rules.

### Why token rebase only through Lido Core

Lido Core acts as a validation performance benchmark oracle in a sense that stETH reward rate isn't changing by allowing external ether to be used for minting, meaning that the Core pool represents the opinionated validator set voted in by the Lido DAO striving for balance between decentralization, efficiency, and resilience.

## Security considerations

### External shares backing

External shares should be reasonably overcollaterized to accommodate
locked external ether increase due to the positive stETH token rebase.

Overcollaterization has to be controlled with an additional ether reserve and its rate against the stETH minted.

### stETH redemptions risk

The Lido Core pool must maintain reasonable amount of `internalShares` and `internalEther` to preserve
redeemability of the stETH token (backed both by internal and external ether supply).

To mitigate these risks, minting security limits should be enforced together with properly aligned incentives between internal and external supply sides.

### Implementation risk

A faulty implementation could allow malicious actors to mint or burn stETH at will, breaking the token’s economic model.

Rigid code reviews, external audits, well-defined access controls, emergency mechanisms, and thorough sophisticated test coverage is essential.

### Staking limits and pause

Pausing staking is unaffected by external shares per se.

However, if the protocol is under stress or paused, external share minting/burning might also be restricted.

### One new minter and one new burner for stETH

This LIP introduces a new class of minter and burner (the external accounting contract),
increasing the importance of access control, deployment configuration, and monitoring.

The burning is allowed only from the own external accounting contract balance.

## Failure modes

### External ether drained

If external ether sources fail or experience a severe mass-slashing event, external shares remain in circulation whilst bad debt accrues in the stETH external ether supply.

Losses can be covered by:
- replenishing the external ether source with additional funds;
- implementing a coverage application mechanism by burning stETH from the donor sources;
- socializing the losses via token rebase (i.e., as it would have been with staking penalties) among all stETH holders as a part of an explicit governance decision.

## Reference implementation

The reference testnet implementation is available on GitHub: [feat: stVaults](https://github.com/lidofinance/core/pull/874).

## Links and references

- [Lido V3 Whitepaper: RFC draft](https://research.lido.fi/t/lido-v3-whitepaper-rfc/10124)
- [Lido "Bring Your Own Validator" Alternative Design](https://mixbytes.io/blog/lido-bring-your-own-validator-alternative-design)
- [Lido rebase documentation](https://docs.lido.fi/contracts/lido#rebase)
- [stETH share mechanics](https://docs.lido.fi/guides/lido-tokens-integration-guide#steth-internals-share-mechanics)
- [stVaults testnet deployments](https://docs.lido.fi/deployed-contracts/hoodi-lidov3/)
- [stVaults technical design](https://hackmd.io/@lido/stVaults-design)
- [stVaults risk assessment framework](https://research.lido.fi/t/risk-assessment-framework-for-stvaults/9978)
- [stVaults fee structure](https://research.lido.fi/t/stvaults-fees-approach/9979)