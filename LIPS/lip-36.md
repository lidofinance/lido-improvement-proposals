---
lip: 36
title: NEST — Automated LDO Buyback and Liquidity Provisioning System
status: Implemented
author: Vasiliy Shapovalov, Vitaly Galaichuk, Jen Kopytina, Alexander Belokon, adcv
discussions-to: https://research.lido.fi/t/liquid-buybacks-nest-execution-with-ldo-wsteth-liquidity/10894
created: 2026-04-20
updated: 2026-08-14
---

## Simple Summary

NEST is a deterministic onchain system that creates a direct rule-based economic link between Lido DAO success and LDO. It enables permissionless conversion of an excess share of staking revenue into LDO via CoW Swap and to pair it with wstETH as DAO-owned liquidity in a Curve LDO/wstETH pool. NEST uses explicit spending caps and formulas to determine available surplus. It must be explicitly funded in order to be operational. It tracks staking revenue only; the architecture is designed to accommodate additional revenue sources through future governance votes. NEST launched in treasury-only mode, sending converted LDO directly to the treasury, while LP mode `Stonks` stays deployed and can be switched to by a subsequent DAO vote.

## Motivation

### Background

Lido DAO generates substantial protocol revenue through staking operations. LDO token holders, however, have no direct automatic link to protocol performance. The DAO participants have discussed mechanisms to address this in the [forum thread](https://research.lido.fi/t/liquid-buybacks-nest-execution-with-ldo-wsteth-liquidity/10894). But no formal technical proposal or agreed architecture was proposed.

### Problem

Without a rule-based surplus conversion mechanism, the DAO faces two structural issues:

1. Revenue above operating costs is absorbed into discretionary spend with no formal commitment to token holders.
2. Any future buyback initiative would require designing a mechanism under pressure, without calibrated parameters or governance controls already in place.

Additionally, on-chain LDO liquidity depth is shallow. Without a systematic provisioning mechanism, any buyback program risks slippage-induced execution failures as it consumes available market depth.

### Solution

NEST encodes a governance commitment: when protocol surplus conditions are met, a constrained share of excess staking revenue is automatically converted into LDO and, in LP mode, paired with wstETH and deployed as DAO-owned liquidity. The mechanism is designed to be bounded by spending caps and governance controls, permissionless in normal operation, low-maintenance, and fully controllable by the Lido DAO.

## Specification

### System Architecture

NEST is implemented as two cooperating contracts — the `BuybackAllocator` (budget allocation) and the `BuybackExecutor` (trade execution and liquidity provisioning). They utilize existing contracts like Stonks v2 and OracleRouter, as well as the governance infrastructure.

The system operates in two governance-controlled modes:

1. **LP mode** — acquired LDO is paired with wstETH and deposited into a Curve LDO/wstETH TwoCrypto-NG pool; LP tokens are retained by the `BuybackExecutor` as DAO-owned liquidity.
2. **Treasury-only mode** — acquired LDO is delivered directly to the Aragon Agent treasury.

The active mode is derived from the active Stonks instance's receiver: the executor for LP mode, the treasury for treasury-only mode. Switching modes means pointing the executor at a different Stonks via `setStonks`, which requires a full DAO vote.

`setStonks` **does not** enforce that no Stonks order is in flight: it sweeps an expired tracked order back to the previous Stonks, but a still-live order survives the switch and is **abandoned**, emitting `OrderAbandoned`. Anyone may call its `recoverTokenFrom` after expiry to return the stETH to the previous Stonks. Operationally, allocation should be paused and any live order recovered before switching modes.

This LIP originally proposed deploying NEST in LP mode as the initial operating configuration. Further review surfaced reasons to defer launch-day liquidity provisioning, so NEST was released in **treasury-only mode**. LP mode is fully deployed and can be enabled by a subsequent DAO vote without redeploying any contracts.

### New Contracts

#### `BuybackAllocator`

Central budget engine of the system. Holds the funded stETH and a set of revenue source addresses, aggregates their cumulative revenue, evaluates eligibility, enforces daily and yearly caps, and pays out stETH to a single executor via `allocate()`. Activated once via `activate()`. Exposes `activate()` and governance setters. Order creation, trading, and Stonks/Order management live in the `BuybackExecutor`.

#### `RevenueSource` (abstract)

Common interface for revenue tracking implementations. Each source exposes a single read, `getCumulativeRevenueUSD()`, returning its monotonic, ever-growing total revenue in USD. The allocator sums these on demand and diffs them over time to derive accrued revenue. A source advertises `IRevenueSource` via ERC-165 to be registrable.

A registrable source must be:

- **monolithic**: a single contract is the only reporter of its revenue stream, with no splitting or migration across contracts, since the allocator tracks each stream by diffing one address's cumulative total;
- **always-alive**: `getCumulativeRevenueUSD()` must not revert in normal operation (see [Revenue Reporting Integrity](#revenue-reporting-integrity) for how liveness failures are handled);
- **fully settled before activation or registration**: `activate()` and `addRevenueSource()` snapshot the source's current cumulative total into the revenue baseline, so only growth past that snapshot becomes spendable surplus. Revenue not yet folded into the cumulative figure at snapshot time would later surface as an unintended windfall, so flush it via `convertPendingRevenueToUSD()` before activating or registering.

#### `StakingRevenueSource`

Captures staking revenue from protocol rebases in two stages. After each rebase, `TokenRateNotifier` calls `pushTokenRate()` with the rebase payload. The source takes the treasury's slice of the fee shares minted in that rebase, given by the formula:
`treasuryShares = sharesMintedAsFees × treasuryFee / (modulesFee + treasuryFee)`,
converts it to stETH at the post-rebase rate, and adds the result to a pending bucket. It never touches the oracle on this path, so a price-feed outage cannot revert a rebase.

A separate permissionless `convertPendingRevenueToUSD()` converts the pending stETH bucket to USD via the `OracleRouter` and appends it to the cumulative total; if the oracle is unavailable the conversion waits and retries, so revenue is deferred, not lost.

Authorization for the rebase callback is checked live against `LidoLocator.postTokenRebaseReceiver()`, and replays are rejected by a strictly-increasing report timestamp.

`StakingRevenueSource` is the only revenue source implemented at launch. The modular `RevenueSource` interface and the allocator's source set provide the extension path for future revenue streams. Each new source requires a dedicated contract and a governance vote for deployment and registration.

#### `BuybackExecutor`

Receives stETH from the `BuybackAllocator` and, in LP mode, settled LDO directly from CoW Swap. In treasury-only mode the bought LDO settles straight to the treasury and bypasses this contract. `BuybackExecutor` forwards stETH to Stonks v2 to create CoW Swap orders for the LDO purchase — the full amount in treasury-only mode, half in LP mode, keeping the other half as stETH and wrapping it to wstETH at deposit time. In LP mode, deposits the balanced LDO/wstETH pair into the Curve pool and retains the minted LP tokens with managed withdrawal to the treasury. Tracks and sweeps expired orders, and exposes pass-through functions for Stonks management gated by `EMERGENCY_ROLE`.

### Modified Existing Contracts

#### `Stonks v2`

Stonks v2 Specification can be found [here](https://hackmd.io/p_ZC5s9tRAOMavh5nVOerw). The NEST implementation uses Stonks v2 as the CoW Swap integration layer for order creation and settlement. The `BuybackExecutor` interacts with Stonks v2 to create orders with the appropriate parameters and to manage order lifecycle events. Two Stonks instances were deployed: one with the receiver set to the treasury (treasury-only mode, active since launch) and one with the receiver set to the executor (LP mode).

The existing contracts were updated to support a configurable settlement receiver address. A new `receiver` field was added to the constructor's `InitParams` struct. If set to `address(0)`, it defaults to `AGENT`, preserving backward compatibility for non-NEST deployments. The stored receiver is passed through to each Order during initialization.

#### `Order`

Updated to support a configurable CoW Swap settlement receiver. The `initialize` function accepts a `receiver_` parameter written into `GPv2Order.Data.receiver` in place of the previously hardcoded `AGENT` address.

#### `TokenRateNotifier`

`StakingRevenueSource` depends on the `TokenRateNotifier` contract registered at `LidoLocator.postTokenRebaseReceiver()`, and it expects the **args-bearing observer flavor**: `addObserver` must ERC-165-detect `ITokenRatePusherWithArgs`, and the notifier must forward the full `handlePostTokenRebase` payload, including `reportTimestamp` and `sharesMintedAsFees` to its observers' `pushTokenRate(...)` callback.

The previous `TokenRateNotifier` propagated only an arg-less rate push and did not forward this payload. A new `TokenRateNotifier` that performs the ERC-165 detection and full-payload forwarding was deployed and wired as `postTokenRebaseReceiver` by the activation vote.

---

### Trade Execution

Execution is two independent permissionless calls:

1. **`allocate()` on the `BuybackAllocator`** computes the spendable USD under the surplus model and the caps, converts it to stETH, transfers it to the executor, and calls the executor's allocation hook. If nothing is eligible it emits `AllocationSkipped` and returns without moving funds.
2. **`placeOrder()` on the `BuybackExecutor`** sells the stETH forwarded to Stonks (sized between configured min/max order bounds) for LDO via a CoW Swap order, with `minBuyAmount` computed onchain from `Stonks.estimateTradeOutput`.

On each allocation the executor sets aside stETH still committed in the pipeline and forwards the rest to Stonks — the full amount in treasury-only mode, half in LP mode, keeping the other half to pair as liquidity. Cadence is governed by the clock-based reserve and the fixed cap windows, and by a single-live-order guard on the executor.

#### Eligibility Gates

**`allocate()` reverts** if the contract is not activated (`whenActivated` modifier).

**`allocate()` skips (non-reverting; emit `AllocationSkipped`):** oracle price unavailable, stETH price below floor, no spendable budget (`budgetUSD` non-positive), a daily or yearly cap window exhausted, payout below the per-call minimum.

**`placeOrder()` preconditions (revert on failure):** executor pause, no live tracked order, Stonks balance at or above the minimum order size.

#### Surplus Mechanics

The allocator keeps a single signed running budget, `budgetUSD`. Each state-touching call begins with a checkpoint: it adds the surplus earned since the previous checkpoint to `budgetUSD`, then re-anchors the revenue baseline and the reserve clock to the present moment. Because the budget is accumulated incrementally this way, past payouts and the parameter values in effect during each interval stay permanently folded into `budgetUSD` — nothing is recomputed retroactively. A release then spends only the non-negative part of `budgetUSD`; a negative budget spends nothing and is carried in storage until later surplus lifts it back above zero.

```
delta       = (Σ cumulativeRevenueUSD − lastTotalRevenueUSD − reserveAccrued) × surplusShareBP / 10000
budgetUSD  += delta                                          (signed; may go negative)
spendable   = max(0, budgetUSD)                              (then clamped, below)
```

Execution requires all of the following:

1. The allocator is activated.
2. The oracle returns a stETH/USD price.
3. `stEthPriceUSD ≥ minStEthPriceUSD` — the price floor, inactive when `minStEthPriceUSD` is set `0`.
4. The non-negative part of `budgetUSD` is positive after the cap and balance clamps.

The reserve in `delta` depends only on elapsed days, not on how often `allocate()` runs. A checkpoint moves the reserve clock to the next UTC-day boundary. After that, the reserve adds one `reserveDailyRateUSD` when the day starts and one more for each full day that follows. Partial days add nothing. `activate()` is the one exception: it anchors to the activation day, so that day is charged instead of skipped.

The resulting spendable amount is then clamped by the unused daily and yearly cap windows, the held stETH balance, and the per-call minimum.

In LP mode the executor sells half of the free stETH for LDO and keeps the other half to pair as wstETH; in treasury-only mode the full amount is swapped for LDO.

**Transition behaviors:**

- **Surplus to deficit**: when the reserve outpaces revenue, `delta` goes negative and `budgetUSD` falls, possibly below zero; a negative budget spends nothing and is held in storage until later surplus lifts it back above zero.
- **Deficit to surplus**: as revenue recovers, positive `delta` rebuilds `budgetUSD` and buybacks resume automatically once it crosses back above zero.
- **Share and rate changes**: `setSurplusShareBP` and `setReserveDailyRateUSD` checkpoint the budget at the old value first, so a new share or rate applies only to revenue and days after the call. Budget already banked in `budgetUSD` is unaffected; there is no retroactive reapplication and no reset lever.

---

### Liquidity Provisioning

The `BuybackExecutor` receives settled LDO from CoW Swap and holds the stETH kept aside for pairing. Liquidity provisioning runs as a separate permissionless flow after trade settlement.

`addLiquidity()` flow:

1. Revert if the system is not in LP mode.
2. Read LDO and stETH balances; revert if either is zero.
3. Convert both to USD via the OracleRouter; revert if any price feed is stale or unavailable.
4. Check the pool price divergence gate: revert if the pool EMA diverges from the oracle ratio beyond `poolPriceDivergenceToleranceBps` (bypassed during bootstrap; see [Pool Skew Protection](#pool-skew-protection)).
5. Take the minimum of the two USD values and compute balanced token amounts corresponding to that minimum.
6. Apply the per-call deposit bounds: revert if the balanced pair's USD value is below `minDepositValueUsd`; scale both sides down proportionally if it exceeds `maxDepositValueUsd`.
7. Wrap the stETH portion to wstETH and deposit the LDO/wstETH pair into the Curve pool.
8. Retain minted LP tokens in the executor.

Excess tokens on the larger side remain in the executor for the next `addLiquidity()` cycle. The Curve LDO/wstETH pool starts with zero balance and scales naturally as orders settle — no upfront seeding is required.

#### Pool Deployment

TwoCrypto-NG pools have no pool-level pause, but are not fully immutable: the Curve factory admin can ramp A/gamma and update fee/rebalancing/oracle parameters. Emergency controls are scoped to the NEST contracts (see Emergency Controls). The pool was deployed with the following configuration:

| Parameter            | Value                                               |
| -------------------- | --------------------------------------------------- |
| Pool address         | `0xD7f1dA0a28E39dd0dB70E6Acdc2B49846AD22760`        |
| Name / symbol        | Curve.fi LDO/wstETH · `LDOwstETH-f`                 |
| Coins                | LDO (`0x5A98…1B32`) / wstETH (`0x7f39…2Ca0`)        |
| A                    | `400000`                                            |
| gamma                | `145000000000000` (0.000145)                        |
| mid_fee              | `500000` (0.005%)                                   |
| out_fee              | `5000000` (0.05%)                                   |
| fee_gamma            | `230000000000000` (0.00023)                         |
| allowed_extra_profit | `2000000000000`                                     |
| adjustment_step      | `146000000000000`                                   |
| ma_time              | `601` (seconds)                                     |
| initial_price        | `8075081795329464726918` (~8,075.08 LDO per wstETH) |
| Factory              | `0x98EE851a00abeE0d95D08cF4CA2BdCE32aeaAF7F`        |
| Pool implementation  | `0x934791f7F391727db92BFF94cd789c4623d14c52`        |
| Math implementation  | `0x1Fd8Af16DC4BEBd950521308D55d0543b6cDF4A1`        |
| Views implementation | `0x07CdEBF81977E111B08C126DEFA07818d0045b80`        |
| Factory admin        | `0x97aA696e37659Fb4f0B53824246d802Df40E980A`        |
| Fee receiver         | `0xa2Bcd1a4Efbd04B63cd03f5aFf2561106ebCCE00`        |

#### Pool Bootstrap

The pool was deployed with zero liquidity and holds none while treasury-only mode is active. Applying the divergence check before the pool has reached sufficient depth would block `addLiquidity()` once LP mode is enabled; the check is bypassed during bootstrap and engages once the pool crosses a defined depth threshold, set to $250,000 (`poolBootstrapMinTvlUsd`).

#### Pool Skew Protection

- **Pool price divergence gate.** Compares the Curve pool's internal EMA price from `price_oracle()`, which reports LDO per wstETH and is converted to an LDO/stETH ratio via `wstETH.stEthPerToken()`, against the OracleRouter's LDO/stETH price. If divergence exceeds `poolPriceDivergenceToleranceBps`, the deposit reverts. This is a structural check independent of deposit size.
- **No per-deposit slippage floor.** Curve values an imbalanced deposit at the pool's internal `price_scale`, which moves only gradually and whose EMA updates at most once per block (the state price is capped at `2 × price_scale`), so a deposit cannot be sandwiched in-block by moving the spot price. This in-block resistance is intrinsic to Curve, independent of NEST's gate. The `add_liquidity` call itself passes a minimum-LP-out of `1` (no effective slippage floor), so the divergence gate is the _sole_ guard against the risk Curve's in-block resistance does not cover: depositing while the pool's price has drifted from the external market.
- **The bootstrap window has neither protection.** While pool TVL is below `poolBootstrapMinTvlUsd` the divergence gate is bypassed (see [Pool Bootstrap](#pool-bootstrap)), and the deposit already carries no slippage floor, so a bootstrap deposit runs with no divergence gate and no slippage guard. This is the least-protected window, accepted because a shallow pool's stale EMA cannot converge without the very deposits the gate would otherwise block. The gate evaluates pre-deposit pool TVL, so `poolBootstrapMinTvlUsd` together with the per-call `maxDepositValueUsd` bounds the unguarded exposure: it is the threshold plus at most one capped deposit, after which the gate engages.

#### Partial and Unfilled Orders

wstETH is minted only at deposit time, so the executor never holds idle wstETH. Before each sale it reserves the stETH still committed in the pipeline — the value of held LDO awaiting pairing, stETH sitting on Stonks, and any live order's residual — so the same stETH is never committed twice. An expired order's stETH is swept back to Stonks and re-sold by the next `placeOrder()`.

#### LP Token Management

| Function                                                     | Access         | Description                                                                       |
| ------------------------------------------------------------ | -------------- | --------------------------------------------------------------------------------- |
| `removeLiquidityAndRecoverToTreasury(uint256 lpAmount, ...)` | `MANAGER_ROLE` | Burns LP tokens, unwraps wstETH, sends LDO and stETH to the Aragon Agent treasury |
| `recoverERC20(address token, uint256 amount)`                | `MANAGER_ROLE` | Recovers assets (including LP tokens) to the Aragon Agent treasury                |

---

### Off-chain Keeper

A keeper bot polls the `BuybackAllocator` and `BuybackExecutor` via view functions on a schedule (e.g., every 15 minutes):

1. Call `convertPendingRevenueToUSD()` to settle pending staking revenue when the oracle is healthy.
2. Check `spendable()` → call `allocate()` if eligible.
3. Check `getPlacementStatus()` → call `placeOrder()` if eligible.
4. Check `canAddLiquidity()` → call `addLiquidity()` if eligible.

All functions are permissionless and retryable when preconditions are unmet; skipped or failed calls are retried on the next cycle. The keeper is integrated into the existing off-chain service, reusing that infrastructure.

CoW Protocol requires a matching off-chain order payload submitted to its API before solvers can discover and fill an order. The keeper monitors for order-creation events, constructs the payload, and posts it to the CoW Protocol API. If submission fails, the order remains unfilled until expiry and standard recovery flows apply.

---

## Configuration

### Parameters

| Category              | Parameter                                         | Description                                                                                                                                                                 |
| --------------------- | ------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Price Floor**       | `minStEthPriceUSD`                                | Minimum stETH/USD price for a payout. `0` disables it.                                                                                                                      |
| **Revenue / Reserve** | `reserveDailyRateUSD`                             | Clock-based daily set-aside subtracted from cumulative revenue before surplus                                                                                               |
|                       | `surplusShareBP`                                  | Share of the cumulative surplus that is spendable (basis points). In LP mode the resulting budget is split equally between the LDO purchase leg and the wstETH pairing leg. |
| **Spending Cap**      | `dailyCapUSD`                                     | USD limit per UTC day slot                                                                                                                                                  |
| **Yearly Cap**        | `yearlyCapUSD`                                    | USD limit per fixed 365-day slot. Fixed windows; unused room does not carry over.                                                                                           |
| **Order Sizing**      | `minSpendPerCallUSD`                              | Minimum payout per `allocate()`                                                                                                                                             |
|                       | `minAllowedOrderAmount` / `maxAllowedOrderAmount` | Per-order stETH floor and ceiling on the executor                                                                                                                           |
| **Pool Guards**       | `poolPriceDivergenceToleranceBps`                 | Divergence guard against pool skew                                                                                                                                          |
|                       | `poolBootstrapMinTvlUsd`                          | Pool TVL at/above which the divergence gate engages; below it the gate is bypassed                                                                                          |
| **Deposit Sizing**    | `minDepositValueUsd` / `maxDepositValueUsd`       | Per-`addLiquidity()` USD value floor and cap                                                                                                                                |
| **Revenue Sources**   | `revenueSources`                                  | Array of active revenue source addresses                                                                                                                                    |
| **Active Stonks**     | `stonks`                                          | The active Stonks instance; the operating mode is derived from its receiver.                                                                                                |

All parameters are configurable via Aragon Voting (`DEFAULT_ADMIN_ROLE`) with validation on write where applicable (the `reserveDailyRateUSD` and `minStEthPriceUSD` setters intentionally accept any value, including `0`). Like the rest of DAO treasury operations, this is not under Dual Governance. Invalid configuration reverts and preserves the previous valid state.

### Initial Values

| Parameter                                         | Value                | Notes                                                                                                                           |
| ------------------------------------------------- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `minStEthPriceUSD`                                | `0`                  | Price floor disabled at launch; retained as a future governance lever                                                           |
| `reserveDailyRateUSD`                             | `$109,589`           | $40M annual revenue baseline, round daily literal                                                                               |
| `surplusShareBP`                                  | `5000` (50%)         | 50% of surplus allocated to NEST. In LP mode: half of that budget goes to the LDO purchase leg, half to the wstETH pairing leg. |
| `dailyCapUSD`                                     | `$50,000`            | Primary active safety limiter                                                                                                   |
| `yearlyCapUSD`                                    | `$10,000,000`        | Annual ceiling                                                                                                                  |
| `minSpendPerCallUSD`                              | `$1,000`             | Dust floor per allocation; constructor enforces `> 0` and `≤ dailyCapUSD`                                                       |
| `poolPriceDivergenceToleranceBps`                 | `200` (2%)           | Divergence guard on LP deposits                                                                                                 |
| `poolBootstrapMinTvlUsd`                          | `$250,000`           | Pool TVL at/above which the divergence gate engages                                                                             |
| `minAllowedOrderAmount` / `maxAllowedOrderAmount` | 1 / 20 stETH         | Per-order stETH bounds on the executor                                                                                          |
| `minDepositValueUsd` / `maxDepositValueUsd`       | `$1,000` / `$50,000` | Per-`addLiquidity()` USD bounds                                                                                                 |

### Roles and Authority

| Role                 | Held By                             | Key Powers                                                                                                                                               |
| -------------------- | ----------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `DEFAULT_ADMIN_ROLE` | Aragon Voting                       | Full configuration control on both contracts, `activate` / `setExecutor` / `setStonks`, role management. Requires full DAO vote.                         |
| `MANAGER_ROLE`       | Treasury Management Committee (TMC) | Asset recovery, LP token management (`removeLiquidityAndRecoverToTreasury`, `recoverERC20`), propose EasyTrack motions to fund the allocator with stETH. |
| `EMERGENCY_ROLE`     | Emergency Brakes multisig, TMC      | Pause/unpause the executor and Stonks creation/signatures. Cannot modify configuration.                                                                  |
| `ALLOCATOR_ROLE`     | `BuybackAllocator`                  | Calls the executor's allocation hook. Granted to the allocator at activation.                                                                            |
| —                    | Permissionless Callers              | `allocate()`, `placeOrder()`, `addLiquidity()`, `convertPendingRevenueToUSD()`                                                                           |

At construction the deploying admin held `DEFAULT_ADMIN_ROLE`; `MANAGER_ROLE`, `EMERGENCY_ROLE`, and `ALLOCATOR_ROLE` were granted explicitly as part of the activation vote. EasyTrack is used exclusively for funding operations (stETH transfers to the allocator). All parameter changes require a full Aragon Voting DAO vote.

TMC may unwind LP positions and return underlying assets to treasury, but cannot reassign LP ownership or change system configuration without a full DAO vote.

---

## Emergency Controls

Independent pause domains, each stoppable without affecting the others:

1. **BuybackExecutor pause** — stops `addLiquidity()`, `placeOrder()`, and the allocation hook; because `allocate()` calls the hook in the same transaction, pausing the executor also halts allocation. Assets accumulate safely.
2. **Stonks creation/signature pauses** — stops new order creation or freezes settlement of existing orders independently.
3. **OracleRouter feed deactivation** — price queries can be stopped by deactivating a token's feed (or when a feed is stale/unavailable), causing allocation and liquidity provisioning to skip or revert. The OracleRouter has no global pause; availability is controlled per token via its feed configuration.

Asset recovery is available at all times and is not subject to pause. Recovery to the treasury sends the token as held; `removeLiquidityAndRecoverToTreasury` unwraps withdrawn wstETH to stETH.

**Prolonged deficit:** There is no manual accounting-reset lever. If a prolonged downturn drives `budgetUSD` negative, the allocator simply spends nothing and holds the negative budget in storage until later surplus lifts it back above zero; buybacks resume automatically. Governance can raise `surplusShareBP` or lower `reserveDailyRateUSD` to speed recovery, but these apply only to revenue and days after the change (`DEFAULT_ADMIN_ROLE`, full DAO vote); already-banked budget is never re-based or clawed back.

---

## Rationale

### Cumulative Surplus Model vs. Instant Daily-Spend

The cumulative model was chosen because it prevents two failure modes of an instant daily-spend model: overspending during consecutive good days (no memory of past losses) and the inability to carry forward unused capacity. After a loss period, new surplus must cover the accumulated deficit before buybacks resume — a natural, automatic risk brake without additional oracle dependencies.

Analytics modeling confirmed that quarterly revenue smoothing produces no meaningful improvement in cumulative execution volume over the daily cadence, while adding significant accounting complexity and oracle overhead.

### ETH Price Gate Set to Zero at Launch

The cumulative surplus mechanism provides natural ETH-price sensitivity without an explicit gate. At a $40M/year cost baseline (~$109k/day), the break-even ETH price is approximately $2,700 at current protocol revenues. Below that price, the protocol automatically runs a daily deficit, and no buybacks occur — no price oracle required. The `minStEthPriceUSD` parameter is retained in the design for future governance reactivation if conditions change.

### $50,000 Daily Cap as Primary Safety Limiter

The daily cap is the primary active safety guardrail. Historically, very few days produce surplus exceeding $50k. The cap bounds the USD spend computed at the oracle price each day, so against bad revenue inputs it holds exposure over a ~6-day governance response window to ~$300k. Because that USD budget is converted to stETH at the oracle price, a corrupted stETH/USD feed can still release more real value than the nominal cap implies; the `minStEthPriceUSD` floor (disabled at launch, see below) is the local mitigation and is retained for that purpose. The cap is independently configurable and can be raised by governance as surplus and on-chain liquidity grow.

### Zero-seed Pool Launch

No upfront LDO seeding from the Aragon Agent treasury is required. Seeding the pool from treasury while simultaneously buying LDO from the market is logically contradictory: the two operations work against each other, with treasury supply injection partially offsetting the market buy. There is no mechanical benefit to pre-seeding.

### Pool Selection

Curve v2 (TwoCrypto-NG) was selected because it requires no active management after deployment, compatible with a DAO-operated permissionless system, and its configurable curve shape provides better efficiency than constant-product alternatives for the LDO/wstETH price range.

### Alternatives Considered

| Alternative                          | Reason Rejected                                                                         |
| ------------------------------------ | --------------------------------------------------------------------------------------- |
| Manual monthly buybacks              | Reintroduces discretionary human intervention; not permissionless; regulatory risk      |
| Quarterly revenue smoothing          | No meaningful improvement in execution volume; higher accounting and oracle complexity  |
| Instant daily-spend (non-cumulative) | No memory of losses; risk of overspending during consecutive surplus days               |
| Burn acquired LDO                    | One-time signal for exit-seeking traders, not a durable mechanism for long-term holders |
| Simple revenue percentage split      | Does not enforce surplus discipline; can activate with no actual treasury surplus       |

Expired and unfilled orders are retried through a bounded recovery flow rather than silently skipped: an expired order's stETH is swept back to Stonks and re-sold by the permissionless `placeOrder()` without additional funding. Because wstETH is minted only at deposit time, partial fills leave no wstETH overhang to clean up.

---

## Security Considerations

### Oracle Manipulation

Price feeds (ETH/USD, stETH/USD via two-hop, and LDO/USD via OracleRouter configuration, which may resolve directly or bridge through ETH) are sourced from Chainlink via the shared OracleRouter. All feeds are validated for staleness and answer validity independently. The daily cap bounds the USD spend computed at the oracle price each day to $50k; because that USD budget is converted to stETH at the same oracle price, the cap fully bounds bad-revenue inputs but only partially bounds stETH/USD oracle corruption, for which `minStEthPriceUSD` is the local mitigation. Individual OracleRouter feeds can be deactivated independently for rapid incident response.

### Permissionless Execution Risk

`allocate()` and `placeOrder()` are callable by anyone. Budgets and `minBuyAmount` use on-chain oracle prices at call time; slippage protection is the Stonks-side margin. Daily and yearly caps enforce hard spending bounds at the oracle price.

### Revenue Reporting Integrity

`StakingRevenueSource`'s rebase callback is authorized live against `LidoLocator.postTokenRebaseReceiver()`, and replays are rejected by a strictly-increasing report timestamp. USD conversion is deferred to a retryable permissionless call, so a price-feed outage defers rather than loses revenue.

**Source liveness.** `allocate()` and its checkpoints sum revenue through a per-source `try/catch`, so a source that reverts is skipped — it contributes zero for that call rather than bricking allocation — and resumes contributing once it recovers. The strict, non-guarded read is used only by the admin paths `activate()`, `addRevenueSource()`, and `removeRevenueSource()`. A source that reverts permanently therefore blocks activation, cannot be added, and **cannot be removed** — removal subtracts the source's current cumulative total from the baseline, which reverts if the source reverts. Unregistering a permanently-broken source requires its `getCumulativeRevenueUSD()` to be made callable again (returning any value) first.

### Slashing Events

Revenue tracks the fees actually minted each rebase. During a downturn or slashing recovery, less revenue accrues while the clock-based reserve keeps growing, so the budget shrinks and buybacks pause automatically until revenue rebuilds it. A deeply negative budget self-heals only as surplus returns (there is no manual reset), so the pause is proportional to how long revenue stays below the reserve.

### No asset custody

All purchased assets remain DAO-owned throughout the execution path. Sell-side stETH flows only through the controlled pipeline: Aragon Agent → `BuybackAllocator` → `BuybackExecutor` → `Stonks` → `Order`; an expired order's residual stETH recovers from the Order back to its Stonks. Bought LDO settles from CoW Swap to the configured receiver — the `BuybackExecutor` in LP mode, or the Aragon Agent treasury in treasury-only mode. No unauthorized exit path exists for sell-side stETH.

### Emergency Response Time

Independent pause domains allow targeted response without halting the full system. The Emergency Committee can pause without a full DAO vote. Asset recovery functions are always available and not subject to pause. At a $50k daily cap, the maximum exposure over a 6-day governance response window is ~$300k in oracle-denominated USD.

### Missing Lido AccountingOracle reports

If an AccountingOracle report is delayed or skipped, no rebase fires and StakingRevenueSource accrues no new revenue. The clock-based reserve keeps growing, so surplus shrinks during the gap and buybacks slow or pause. No spike occurs when reports resume, because revenue reflects only the fees actually minted. The system resumes automatically once revenue rebuilds surplus.

---

## References

- Forum discussion: [Liquid Buybacks: NEST Execution with LDO/wstETH Liquidity](https://research.lido.fi/t/liquid-buybacks-nest-execution-with-ldo-wsteth-liquidity/10894)
- Snapshot: [DAO approval of the cumulative surplus model](https://snapshot.box/#/s:lido-snapshot.eth/proposal/0x022e901a6368573d18b150eecda563dd2ee17ad2aa6a0ef9772151cc7ba55187)
- [Ack3 Lido NEST audit report (07-2026)](https://github.com/lidofinance/audits/blob/main/Ack3%20Lido%20NEST%20Audit%20Report%2007-2026.pdf)
- [Stonks Repository](https://github.com/lidofinance/stonks)
- [Stonks v2 working branch](https://github.com/lidofinance/stonks/tree/feature/dao-781-develop-stonks-v2-poc)
- [Stonks v2 Specification](https://hackmd.io/p_ZC5s9tRAOMavh5nVOerw)
- [NEST Pull Request](https://github.com/lidofinance/stonks/pull/28)
- [CoW Protocol documentation](https://docs.cow.fi/)
- [Curve TwoCrypto](https://github.com/curvefi/twocrypto-ng)
- [TokenRateNotifier](https://github.com/lidofinance/core/blob/master/contracts/0.8.9/TokenRateNotifier.sol)
