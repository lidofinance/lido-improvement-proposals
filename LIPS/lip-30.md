---
lip: 30
title: Expanding stETH liquidity layer with over-collateralized minting
status: WIP
author: Alexey Potapkin, Eugene Mamin, Eugene Pshenichnyi, Max Merkulov
discussions-to: TBA
created: 2024-12-06
updated: 2025-06-14
---

# Expanding stETH liquidity layer with over-collateralized minting

## Simple Summary

Introduce an **over‑collateralised accounting system** for **stETH**, enabling the token to be backed by ether supplied **outside the Lido Core pool** while preserving full fungibility and 1:1 redeemability.

## Abstract

Lido’s next proposed upgrade (known as Lido V3) generalises stETH minting rules. Any entity that can algorithmically prove possession of ≥100% collateral **plus a safety reserve** may mint stETH against the *effective* portion of that collateral.  The system refers to such entities as staking vaults. A staking vault might be just a delegated staking vault, separate vault-centered staking pool, a sophisticated staking product with the staking vault in its code, or any possible future staking gadget.

New key concepts and changes:

1. **Collateral registry (aka VaultHub)** – on‑chain ledger of every vault’s *total value* (TV) and *locked value* (or just `locked`).  
2. **Accounting Oracle v3** – reports aggregated state root of the staking vaults to drive collateral registry bookkeeping in an asynchronous manner.  
3. **External shares mint and burn** – LidoCore enforces `stETH.totalSupply() ≤ Σ core pool total supply + Σ locked`, refusing mints that would exceed it.  
4. **Reserve‑breach hooks** – if a staking vault reserve buffer falls below governance‑set limits, Core blocks further mints from that staking vault and may trigger containment routines.

The proposal **does not prescribe** how an staking vault implements reserves or validator operations; it only defines **the proofs and invariants** the core contract must be able to verify and enforce.

## Motivation

The V2 model implicitly assumes that every new ETH deposit is equal‑risk socialized and immediately mintable 1:1. As Lido scales to heterogeneous validator sets and diverse product lines, that assumption is no longer holds. Without structural safeguards a single high‑risk source of ether supply for stETH could:
* slash enough stake to force a global negative rebase, or  
* mint stETH minutes before an exit, externalising risk to the broader holder base.

By hard‑coding **over‑collateralisation at the protocol level** and measuring backing **per‑vault**, the design contains such tail risks while unlocking new ether supply lines.  stETH remains a *single*, fungible liquidity layer; risk is isolated at the source, not socialised.

## Specification

### Glossary

| Concept | Definition | 
|---------|------------|
| **Staking Vault** | Any contract or module that locks ETH and requests stETH mints. |
| **Total Value (TV)** | Sum of all ETH the staking vault controls on EL + CL, incl. pending deposits. | 
| **Reserve Ratio (RR)** | Minimum share of TV that **should not** be represented by stETH. |
| **Force Rebalance Threshold (FRT)** | Minimum share of TV that incurs force rabalance if becomes represented by stETH, `FRT < RR` |
| **Locked** | `locked = TV × (1 – RR)` (floored) |
| **Liability** | stETH value minted against the staking vault position (rebases with stETH) |
| **Global backing invariant** | `stETH.totalSupply() ≤ Σ core pool total supply + Σ locked` |
| **Reserve Breach** | If `TV – liability < RR × TV`, staking vault enters *unhealthy* state. |
| **Bad debt** | If `liability > TV`, staking vault enters *bad debt* state |

#### Design principles to uphold

Three key properties of the stETH token are to be preserved.

#### Collaterilization

Every minted stETH has corresponding ether-nominated value, either inside the Lido Core pool or external ether locked (i.e., acting as the external stETH minting collateral) by the protocol. 

New accounting model suggests overcollatalized minting approach for the external part of the token supply to manage slashing risks against the diverse staking setups without enforced policies and requirements behind this part of collateral except only those covered in the current proposal.

#### Redemability

Every minted stETH must be redeemable either Lido Core pool, or the external minting collateral. 

This collateral must be potentially accessible by the protocol under certain conditions (e.g., collateral shortage or Lido Core pool depletion), allowing writing-off minted stETH external ether source liability by moving the corresponding external ether to the Lido Core pool (treating 1 stETH = 1 ETH).

#### Fungibility

The existing stETH token contract remains to be ERC-20 deployed under the same address on Ethereum mainnet.

Minting or burning new stETH does not trigger stETH token rebase, rebases keep happenning inside the establed `AccountingOracle` report lifecycle as a part of the main phase of the report data delivery. Rewards and penalties, accrued for stETH rebase, remain to be defined by the Lido Core pool validator set (i.e., stETH minted against the external minting collateral doesn't change the embedded stETH staking APR). 

### Key accounting changes

> For simplicity, 'external ether' means 'staking vault minting collateral', and 'external shares' are stETH shares minted against staking vaults.

#### Total pooled ether update

The total pooled ether (`totalPooledEther`) including the staking vault ether becomes:

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

> NB: Any changes in `shareRate` affect `externalEther` as a part of the regular stETH token rebase induced by `AccountingOracle`.
Therefore, the staking vault must must maintain the reserve sufficient to update the total collaterized ether amount as a part of the token rebase.

#### Mint external shares

The amount of external stETH shares minted MUST be fully backed via a staking vault (including `RR`).

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

A staking vault can also burn a specified amount of stETH shares from its balance to unlock the corresponding external ether collateral,
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

#### Oracle reports

Accounting oracle report happen in the same cadence.

Token rebase implements in a way that `externalEther` gets updated according to the newly delivered Lido Core pool changes after reward and fees distributed.

##### Rewards, penalties, and fees

Rewards and penalties, accrued by the Lido Core pool validator set define the stETH token rebase:

Transient balance as observed during the report gathering:
```math
transientCLBalance_{report} = 32ETH \times (CLValidators_{report} - CLValidators_{pre})
```
Principal CL balance as observed during the report: 
```math
principalCLbalance_{report} = CLbalance_{pre} + transientCLBalance_{report}
```
Rewards accrued as observed during the report:
```math
CLrewards_{report} = [CLbalance_{report} + withdrawalVault.balance_{report}] - principalCLbalance_{report}
```
Internal ether received the following update:
```math
internalEther_{\text{post}} = 
    internalEther_{pre} + CLrewards_{report} + elRewardsVault.balance_{report} - wqWithdrawals_{report}
```
Internal shares before fees:
```math
internalShares_{\text{post}} = 
    internalShares_{pre} - sharesToBurn_{report}
```

TODO: penalties, fees, bad debt internalizing

## Specification (minimal)

### Total value

#### Function: `totalPooledEther()`

TODO

### Mint external shares

#### Function: `Lido.mintExternalShares(address receiver, uint256 sharesAmount)`

    Input:
        `receiver`: The address that will receive the newly minted external stETH.
        `sharesAmount`: The number of shares to mint.
    Actions:
        - Compute the corresponding external ether amount `deltaExternalBalance` at the current stETH's `shareRate`.
        - Increase `externalBalance` by `deltaExternalBalance`.
        - Mint `sharesAmount` of stETH to receiver.

### Burn external shares

#### Function: `Lido.burnExternalShares(uint256 sharesAmount)`

    Input:
        `sharesAmount`: The number of shares to burn from the caller’s balance.
    Actions:
        - Compute the external ether redemption ΔETHext, burnΔETHext, burn​.
        - Decrease `externalBalance` by ΔETHext, burnΔETHext, burn​.
        - Burn `sharesAmount` of stETH from the caller’s balance.

### Rebalance

TODO

## Rationale

### Abstracting external ether

The design does not require the protocol to 'understand' the nature of external ether. It only requires trusted external accounting, ensuring that the external ether source is credible. The external ether could represent aggregator-controlled validator balances or, potentially, other complex Ethereum and broader ecosystem mechanisms.

### Why track external shares instead of external ether directly

stETH is defined by its share mechanics, and all protocol logic revolves around shares and their rate. By expressing external contributions and withdrawals in terms of stETH shares, the protocol remains consistent. Ether is only an input or output measure, while shares define the in-protocol liquidity and distribution rules.

### Why token rebase only through Lido Core

Lido Core acts as a validation performance benchmark oracle in a sense that stETH reward rate isn't changing by allowing external ether to be used for minting, meaning that the Core pool represents the opinionated validator set voted in by the Lido DAO striving for balance between decentralization, efficiency, and resilience.

## Security considerations

### No unbacked mint

Invariant is enforced upon every mew stETH mint.

For the already minted, external shares should stay reasonably overcollaterized to accommodate locked increase due to the historically-positive stETH token rebase.
Overcollaterization is controlled with the appropriately chosen `RR` and `FRT` values to allow fine-grainted control:
- block new minting requests when the `RR` threshold breached
- allow force rebalance to be made by the protocol when the `FRT` threshold breached

### Collateral updates

TODO: `RR` and `FRT` values should be chosen to: 
- address slashing risks according to the approved level
- have a reasonable time offset between mint capacity exhaustion due to breaching `RR` and force-rebalance activation due to `FRT`
- account for the oracle report cadence and asynchronous nature of the report delivery

### Oracle tamper resistance

TODO: continues quorum-enforced sumbission + on-chain sanity‑checker pattern.

### Isolation

TODO: staking vault bug cannot drain Lido Core supply; worst case, staking vault’s own `TV` is lost and its stETH mints halt.

### stETH redemptions risk

The Lido Core pool must maintain reasonable amount of `internalShares` and `internalEther` to preserve
redeemability of the stETH token (backed both by internal and external ether supply).

To mitigate these risks, minting security limits should be enforced together with properly aligned incentives between internal and external supply sides.

Under severe Lido Core pool depleting conditions, the protocol may require redemption requests to be handled using staking vaults, attributing the `redemptions` counter nominated in ether accordingly. That would require from vault either voluntary burning `stETH` from its balance, decreasing simultaneously `liability` and `redemptions`, or rebalancing the `redemptions` amount, satisfying the assigned obligation.

### Implementation risk

A faulty implementation could allow malicious actors to mint or burn stETH at will, breaking the token’s economic model.

Rigid code reviews, external audits, well-defined access controls, emergency mechanisms, and thorough sophisticated test coverage is essential.

### Staking limits and pause

Pausing staking is unaffected by external shares handling.

However, if the protocol is under stress or paused, external share minting/burning might also be temporary unoperational.

### One new minter/burner for stETH

This LIP introduces a new source of minting and burning, increasing the importance of access control, deployment configuration, and monitoring.

To limit the control surface, all minting and burning of external shares is authorized only via the single `VaultHub` contract. There is a global minting limit preventing external shares to exceed the sane limits upon initial proposal adoption.

## Failure modes

### External ether drained

If external ether sources fail or experience a severe mass-slashing event, external shares remain in circulation whilst bad debt accrues in the stETH external ether supply (i.e., the `liability > TV` invariant broken for a vault or a group of vaults).

Losses can be covered by:
- replenishing the staking vaults accrued bad debt with additional funds;
- socializing the bad debt among vaults containing slashed validors of the same node operator;
- executing a self-coverage application (see [LIP-18](./lip-18.md));
- internalizing the losses to protocol, decreasing stETH token rebase (as it would have been with the Lido Core pool staking penalties) within the next oracle report

### Emergency pause

The collateral registry contract (`VaultHub`) and auxiliary parts implement an emergency pause mechanism, allowing to minimize the potential impact of the discovered vulnerability or unspecified critical protocol state. 

Paused state prevents:
- a new staking vault from being registered in the collateral registry;
- already registered vaults can't be disconnected;
- stETH can't be minted or burned against staking vaults;
- ether can't be funded or withdrawn from the registered vaults;
- rebalance operations are paused;
- staking vaults can't receive oracle reports;
- ether can't be deposited to beacon chain from staking vaults
- assigned obligations of staking vaults (redemptions and fees) can't be settled

## Rationale

TODO:
- a single fungibility layer (one token, many backers) simplifies existing and future DeFi integration, avoiding liquidity fragmentation;
- source‑level risk accounting – every staking vault self‑insures via locked reserves; honest participants are unaffected by others’ failures;
- upgradeable & extensible – new staking vault type would require only an oracle plug‑in and a separate collateral registry entry
- minimal Lido Core pool surface – core gains a few new contracts (Collateral registry (VaultHub) + Lido/Accounting updates + upgraded Oracle) and tight invariant checks yet remains agnostic to business logic inside staking vaults and strategies built atop.

## Backward compatibility

TODO: 
- existing deposit flow stays intact
- wstETH wrappers on L1 and L2 are remain compatible with stETH
- withdrawal NFTs and oracle consumer APIs remain untouched

The only observable changes are:
- `totalSupply` may grow more slowly than beacon‑chain TVL (because of reserves).    
- New events added, the old ones are untouched
    
No action required from existing stETH/wstETH integrators.

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