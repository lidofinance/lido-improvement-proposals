---
lip: 28
title: External ether backing for stETH
status: WIP
author: Alexey Potapkin, Eugene Mamin, Eugene Pshenichnyi
discussions-to: TBA
created: 2024-12-06
updated: 2024-02-17
---

# External ether backing for stETH

## Simple Summary

Introduce extended accounting principles for the Lido on Ethereum protocol, enabling the protocol to manage stETH minting backed by ether (ETH) that is outside of the Lido Core staking pool.

These rules allow for the controlled minting and burning of stETH shares, ensuring that external ether sources become part of the total ether backing stETH, while preserving key invariants, accounting integrity, and protocol solvency.

## Abstract

The Lido on Ethereum protocol currently tracks pooled ether under its direct management — buffered ether, validators’ balances on the Consensus Layer (CL), and transient validator states — forming the protocol's ["total pooled ether"](https://docs.lido.fi/contracts/lido#gettotalpooledether).

This proposal extends the accounting model to include an additional ether supply external to the protocol’s direct purview (aside of the Lido Core staking pool).

### Mint external shares

A newly introduced "external ether accounting contract" (trusted by the [`Lido/stETH`](https://docs.lido.fi/contracts/lido) protocol contract) is allowed to mint stETH shares to a specified recipient.
The amount of external stETH shares minted MUST be fully backed via an equivalent external ether amount locked by the external ether accounting contract.

NB: The mint operation MUST NOT incur the stETH token rebase.

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

### Burn external shares

The "external accounting contract" can also burn a specified amount of stETH shares from its own balance to unlock the corresponding ether amount from the external accounting contract,
decreasing the stored `externalShares`, i.e., denoting:

- $\Delta S_{\text{ext, burn}}$ as the number of external shares to burn:

```math
\Delta S_{\text{ext, burn}} = \frac{\Delta ETH_{\text{ext, burn}}}{ \text{shareRate}}.
```

Thus, the external shares value becomes:

```math
\text{externalShares}_{\text{new}} = \text{externalShares}_{\text{old}} - \Delta S_{\text{ext, burn}}.
```

### Total pooled ether

The total pooled ether (`totalPooledEther`) considering the external balance becomes:

```math
\text{totalPooledEther} = \text{bufferedEther} + \text{CLbalance} + \text{transientEther} + \text{externalEther}.
```

If we denote:

```math
\text{internalEther} = \text{bufferedEther} + \text{CLbalance} + \text{transientEther}
```

```math
\text{internalShares} = \text{totalShares} - \text{externalShares}
```

and taking into account:

```math
\text{shareRate} = \frac{\text{totalPooledEther}}{\text{totalShares}} = \frac{\text{internalEther}}{\text{internalShares}}
```

Then:

```math
\text{externalEther} = \text{externalShares} \times \text{shareRate} = \frac{\text{externalShares} \times \text{internalEther}}{\text{internalShares}}
```

NB: Any changes in `shareRate` affect `externalEther` as a part of the regular stETH token rebase induced by `AccountingOracle`.
Therefore, the external ether accounting contract MUST update the locked ether amount as a part of the token rebase.

### stETH rebase rewards

### stETH rebase fees

### External ether update



## Motivation

- Flexibility: locked ether outside of the Lido Core pool can be used as liquid stETH shares.
- Uniform accounting: by incorporating external ether into the regular stETH rebase mechanics, the entire system remains consistent and leverages existing logic.
- Broader options and future compatibility: the approach suggested fosters new integrations, use cases, and products that manage validator sets or external vaults while still benefiting from stETH liquidity.

## Design assumptions

- External ether as backing: external shares are always intended to be fully backed by the external ether. The system treats this external ether similarly to protocol-held ether in terms of calculation for stETH's `shareRate`.
- No spurious out-of-order rebases: minting or burning external shares does not by itself trigger a rebase. Rebases remain driven by the `AccountingOracle` reporting only.
- Burn/mint system access: Only a designated core contract for external accounting, trusted by the `Lido/stETH` core protocol contract, can invoke external share minting/burning.

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

#### Function: `burnExternalShares`

## Rationale

### Abstracting external ether

The design does not require the protocol to 'understand' the nature of external ether. It only requires trusted external accounting, ensuring that the claimed external ether is credible. The external ether could represent aggregator-controlled validator balances or, potentially, other complex DeFi arrangements.

### Why track external shares instead of external ether directly

stETH is defined by its share mechanics, and all protocol logic revolves around shares and their rate. By expressing external contributions and withdrawals in terms of stETH shares, the protocol remains consistent. Ether is only an input or output measure, while shares define the in-protocol liquidity and distribution rules.

### Why extend rebase

Incorporating external ether into rebase ensures that the `shareRate` calculation remains a single cohesive formula. Without merging external ether into rebase logic, the system would have to manage multiple share rate domains, creating fragmentation and complexity.

### Rewards and fees

Because all stETH holders share the same `shareRate`, any changes to total pooled ether (including externally sourced rewards or penalties) are proportionately distributed. This ensures fairness and uniformity in how the protocol accounts for and distributes external inputs (socialization angle).

## Security considerations

### External shares backing

External shares should be reasonably overcollaterized to accommodate
locked external ether increase due to the positive stETH token rebase.

Overcollaterization has to be controlled with an additional ether reserve and its rate against the stETH minted.

### stETH redemptions risk

The Lido Core pool must maintain reasonable amount of `internalShares` and `internalEther` to preserve
redeemability of the stETH token (backed both by internal and external ether supply).

To mitigate these risks, minting security limits should be enforced together with properly aligned incentives
between internal and external supply sides.

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

If external ether sources fail or experience a slash, external shares remain in circulation.

Losses can be covered by:

- replenishing the external ether source with additional funds;
- implementing a coverage application mechanism;
- socializing as penalties among all stETH holders as a part of a one-off DAO vote.

## Reference implementation

A preliminary draft of the implementation is available in the [feat: stVaults](https://github.com/lidofinance/core/pull/874).

## Links and references

- [Lido rebase documentation](https://docs.lido.fi/contracts/lido#rebase)
- [stETH share mechanics](https://docs.lido.fi/guides/lido-tokens-integration-guide#steth-internals-share-mechanics)
- [stVaults technical design](https://hackmd.io/@lido/stVaults-design)
