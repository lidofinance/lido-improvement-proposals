---
lip: 36
title: NEST — Automated LDO Buyback and Liquidity Provisioning System
status: Proposed
author: Vasiliy Shapovalov, Vitaly Galaichuk, Jen Kopytina, Alexander Belokon, adcv
discussions-to: https://research.lido.fi/t/liquid-buybacks-nest-execution-with-ldo-wsteth-liquidity/10894
created: 2026-04-20
updated: 2026-05-05
---

## Simple Summary

NEST is a deterministic onchain system that creates a direct rule-based economic link between Lido DAO success and LDO. It enables permissionless conversion of an excess share of staking revenue into LDO via CoW Swap and to pair it with wstETH as DAO-owned liquidity in a Curve LDO/wstETH pool. NEST uses explicit spending caps and formulas to determine available surplus. It must be explicitly funded in order to be operational. At launch, it will track staking revenue only; the architecture is designed to accommodate additional revenue sources through future governance votes. NEST can also operate in treasury-only mode if the DAO decides to skip LP provisioning and send converted LDO directly to the treasury.

## Motivation

### Background

Lido DAO generates substantial protocol revenue through staking operations. LDO token holders, however, have no direct automatic link to protocol performance. The DAO participants have discussed mechanisms to address this in the [forum thread](https://research.lido.fi/t/liquid-buybacks-nest-execution-with-ldo-wsteth-liquidity/10894). But no formal technical proposal or agreed architecture was proposed.

### Problem

Without a rule-based surplus conversion mechanism, the DAO faces two structural issues:

1. Revenue above operating costs is absorbed into discretionary spend with no formal commitment to token holders.
2. Any future buyback initiative would require designing a mechanism under pressure, without calibrated parameters or governance controls already in place.

Additionally, on-chain LDO liquidity depth is shallow. Without a systematic provisioning mechanism, any buyback program risks slippage-induced execution failures as it consumes available market depth.

### Solution

NEST encodes a governance commitment: when protocol surplus conditions are met, a constrained share of excess staking revenue is automatically converted into LDO and, in LP mode (the initial configuration proposed by this LIP), paired with wstETH and deployed as DAO-owned liquidity. The mechanism is designed to be bounded by spending caps and governance controls, permissionless in normal operation, low-maintenance, and fully controllable by the Lido DAO.

## Specification

### System Architecture

NEST is implemented as two cooperating but independently operable subsystems — Trade Execution and Liquidity Provisioning. They utilize existing contracts like Stonks v2 and OracleRouter, as well as the governance infrastructure.

The system operates in two governance-controlled modes:

1. **LP mode** — acquired LDO is paired with wstETH and deposited into a Curve LDO/wstETH TwoCrypto-NG pool; LP tokens are retained by the LiquidityProvisioner as DAO-owned liquidity.
2. **Treasury-only mode** — acquired LDO is delivered directly to the Aragon Agent treasury.

The active mode is set via the `liquidityProvisioner` address on the NESTController: a non-zero address enables LP mode; `address(0)` enables treasury-only mode, requires Stonks v2 instance with the receiver set to AGENT. Switching between modes requires a full DAO vote.

This LIP proposes deploying NEST in **LP mode as the initial operating configuration**. Treasury-only mode is available as a governance-controlled fallback that could be enabled by a subsequent DAO vote without redeploying the core contracts.

### New Contracts

#### `NESTController`

Central coordinator of the system. Holds an array of revenue source addresses, aggregates revenue from active sources on each trigger, evaluates eligibility gates, enforces daily and annual caps, and creates CoW Swap orders via Stonks v2. In LP mode, wraps half of the stETH budget to wstETH and transfers it to the LiquidityProvisioner. Exposes pass-through functions for Stonks and Order management accessible by the Treasury Management Committee (TMC) and Emergency Committee.

#### `RevenueSource` (abstract)

Common interface for revenue tracking implementations. Each source stores the latest normalized daily revenue in USD, the report timestamp, and an immutable staleness window set at deployment. Exposes `getRevenue()` returning `(uint256 revenueUSD, uint256 reportTimestamp, bool isStale)`.

Revenue from sources covering different time periods is normalized to a daily USD rate before storage:

```
dailyRevenueUSD = revenueUSD × 86400 / (reportTimestamp − lastReportTimestamp)
```

This ensures values from different reporting cadences are directly comparable. The controller sums normalized values from all active sources without per-source period tracking.

#### `StakingRevenueSource`

Captures staking revenue from protocol rebases. Integrates with Lido's `TokenRateNotifier` contract via the `ITokenRatePusher` interface; must be registered with `TokenRateNotifier` as part of NEST deployment.

After each rebase, the source computes the DAO treasury's share of staking revenue by back-deriving it from the stETH/wstETH rate delta, total internal share count, and treasury fee parameters from the StakingRouter. The result is converted to USD via the OracleRouter and stored along with the block timestamp.

```math
\text{revenueStEth} = \frac{\Delta\text{rate} \times \text{internalShares} \times \text{treasuryFee}}{\text{TOKEN\_RATE\_SCALE} \times (\text{basePrecision} - \text{modulesFee} - \text{treasuryFee})}
```

```math
\text{revenueUSD} = \frac{\text{revenueStEth} \times \text{stEthUsdPrice}}{\text{PRICE\_SCALE}}
```

If the rate does not increase, the source records zero revenue without updating the rate baseline. After a slashing event, the rate baseline remains at its pre-slash level and revenue stays zero until the protocol fully recovers past the previous high-water mark, ensuring a buyback pause proportional to slash severity.

Access to `pushTokenRate` is restricted to `REPORTER_ROLE` holders (granted to `TokenRateNotifier` at deployment).

`StakingRevenueSource` is the only revenue source integrated at launch. The modular `RevenueSource` interface and the controller's source array provide the extension path for future revenue streams. Each new source requires a dedicated contract, a governance vote for deployment and registration, and appropriate staleness configuration.

#### `LiquidityProvisioner`

Receives settled LDO directly from CoW Swap and wstETH from the NESTController. Deposits balanced amounts of both into the Curve LDO/wstETH pool. Retains minted LP tokens with managed withdrawal and transfer capabilities. Handles excess wstETH cleanup via a permissionless cooldown-gated unwrap function.

### Modified Existing Contracts

#### `Stonks v2`

Stonks v2 Specification can be found [here](https://hackmd.io/p_ZC5s9tRAOMavh5nVOerw). The NEST implementation uses Stonks v2 as the CoW Swap integration layer for order creation and settlement. The NESTController interacts with Stonks v2 to create orders with the appropriate parameters and to manage order lifecycle events. On deployment we will deploy two dedicated Stonks instances for LP and treasury-only modes, with the receiver set according to the mode's requirements.

The existing contracts will be updated to support a configurable settlement receiver address. A new `receiver` field is added to the constructor's `InitParams` struct. If set to `address(0)`, it defaults to `AGENT`, preserving backward compatibility for non-NEST deployments. The stored receiver is passed through to each Order during initialization.

#### `Order`

Updated to support a configurable CoW Swap settlement receiver. The `initialize` function accepts a `receiver_` parameter written into `GPv2Order.Data.receiver` in place of the previously hardcoded `AGENT` address.

---

### Trade Execution

Any caller invokes `triggerExecution()` on the NESTController with no parameters. The function:

1. **Verifies infrastructure preconditions and runs daily accounting** — checks pause state and revenue source registration; runs accounting at most once per day (gated by `lastAccountingTimestamp`) to update cumulative allocation state.
2. **Evaluates eligibility gates** — surplus, cumulative capacity, oracle quotability, ETH price floor, budget constraints. Emits `ExecutionSkipped` and returns `address(0)` if any eligibility gate fails.
3. **Derives sell amount** — in treasury-only mode: full constrained budget. In LP mode: half of constrained budget.
4. **Creates order** — sends stETH to Stonks v2 for LDO purchase via CoW Swap.
5. **Wraps and transfers wstETH** _(LP mode only)_ — wraps the other half of stETH to wstETH and transfers it to the LiquidityProvisioner.
6. **Updates state** — `lastTriggerOrderTimestamp` is set only when an order is actually created.

The controller uses two independent timestamps to prevent DoS via accounting-trigger griefing:

- `lastAccountingTimestamp` gates accounting cadence; prevents double-counting.
- `lastTriggerOrderTimestamp` gates order creation; prevents multiple orders per accounting cycle.

If accounting runs but an eligibility gate blocks the order, the keeper can call again later that day. Accounting will be skipped on retry, but order creation remains available.

#### Eligibility Gates

**Infrastructure gates (revert on failure):** execution pause, revenue sources registered, accounting cadence, order-already-created-in-cycle.

**Eligibility gates (non-reverting; emit `ExecutionSkipped` + return `address(0)`):** surplus positive, cumulative capacity non-negative, oracle quotability, ETH price floor, annual cap, stETH balance, minimum order size.

#### Surplus Mechanics

Revenue surplus accumulates cumulatively. Each day's surplus increases the total buyback allocation; each day's loss reduces it. Buybacks are only allowed while cumulative unrealized capacity remains non-negative.

```
surplus = totalDailyRevenue − dailyRevenueThresholdUSD
dailyAllocation = surplus × surplusShareBps / 10000
allocatedForBuybacksUSD += dailyAllocation
unrealizedBuybacksUSD = allocatedForBuybacksUSD − cumulativeBuybacksUSD
```

Execution requires all of the following:

1. `lastDailyAllocationUSD > 0` implies today's surplus is positive.
2. `unrealizedBuybacksUSD ≥ 0` before today's allocation update — the protocol has not already spent more than it has earned.
3. Oracle quotability and staleness checks.
4. `ethPriceUSD > ethPriceFloorUSD` — inactive when floor is `0`.

Budget per trigger: `min(dailyCapUSD, lastDailyAllocationUSD)`, then clamped by annual cap and stETH balance. In LP mode, this constrained budget is split equally: one half is sold for LDO via Stonks, the other half is wrapped to wstETH and transferred to the LiquidityProvisioner. In treasury-only mode, the full budget is swapped for LDO.

**Transition behaviors:**

- **Surplus to deficit**: daily allocations go negative; `unrealizedBuybacksUSD` turns negative; buybacks pause until new surplus covers the deficit.
- **Deficit to surplus**: positive allocations gradually rebuild `allocatedForBuybacksUSD`; buybacks resume once `unrealizedBuybacksUSD` crosses back above zero.
- **Share changes**: affect only future allocations; past committed allocations are non-retroactive.

---

### Liquidity Provisioning

The LiquidityProvisioner receives settled LDO from CoW Swap and wstETH from the NESTController. Liquidity provisioning runs as a separate permissionless flow after trade settlement.

`addLiquidity()` flow:

1. Read LDO and wstETH balances; revert if either is zero.
2. Convert both to USD via the OracleRouter; revert if any price feed is stale or unavailable.
3. Take the minimum of the two USD values and compute balanced token amounts corresponding to that minimum.
4. Deposit LDO and wstETH into the Curve pool with a minimum mint amount slippage guard.
5. Retain minted LP tokens in the provisioner.

Excess tokens on the larger side remain in the provisioner for the next `addLiquidity()` cycle. The Curve LDO/wstETH pool starts with zero balance and scales naturally as orders settle — no upfront seeding is required.

#### Pool Deployment

Deployment parameters, including initial price, A/gamma coefficients, fee structure, and oracle settings, will be published on the forum two weeks before deployment. Curve pools have no admin or pause controls post-deployment and are immutable once live. Emergency controls are scoped to the NEST contracts (see Emergency Controls).

#### Pool Bootstrap

The pool launches with zero liquidity. Applying the divergence check before the pool has reached sufficient depth would block addLiquidity() at launch; the check is bypassed during bootstrap and engages once the pool crosses a defined depth threshold. The threshold value will be published on the forum two weeks before the on-chain vote.

#### Pool Skew Protection

Two-layer protection scheme:

1. **Pool price divergence gate** compares the Curve pool's internal EMA price from `price_oracle()` against the OracleRouter LDO/wstETH price. If divergence exceeds `poolPriceDivergenceToleranceBps`, the deposit reverts. This is a structural check independent of deposit size.
2. **LP token slippage guard** sets a minimum mint amount from `calc_token_amount()` discounted by `poolSlippageToleranceBps`, guarding against MEV sandwich attacks during the deposit transaction.

#### Excess wstETH

Each trigger in LP mode wraps and transfers wstETH equal to the sell half. Partial or unfilled orders leave excess wstETH in the provisioner with no LDO counterpart. There are two resolution paths:

- **`retryFromStonks()`** recovers stETH from an expired Order to Stonks and creates a new order. Shares ETH price floor and oracle quotability gates with `triggerExecution()` but bypasses budget gates since stETH was already committed.
- **`unwrapExcessWstEth()`** unwraps excess wstETH to stETH and returns it to the NESTController after a cooldown (one order duration in seconds after the last order timestamp). Uses a clamp target accounting for all sell-side stETH still in the pipeline. After transfer, `accountForReturnedExcess()` on the controller adjusts both `annualSpendAccumulatorUSD` and `cumulativeBuybacksUSD` to prevent double-counting.

#### LP Token Management

| Function                                         | Access               | Description                                                          |
| ------------------------------------------------ | -------------------- | -------------------------------------------------------------------- |
| `removeLiquidity(uint256 lpAmount)`              | `MANAGER_ROLE`       | Burns LP tokens, withdraws LDO and wstETH back to provisioner        |
| `recoverERC20(address token, uint256 amount)`    | `MANAGER_ROLE`       | Recovers assets to the Aragon Agent treasury                         |
| `transferLpTokensTo(address to, uint256 amount)` | `DEFAULT_ADMIN_ROLE` | Transfers LP tokens to an arbitrary address — requires full DAO vote |

---

### Off-chain Keeper

A keeper bot polls the LiquidityProvisioner and NESTController via view functions on a schedule (e.g., every 15 minutes):

1. Check `canUnwrapExcessWstEth()` → call `unwrapExcessWstEth()` if eligible.
2. Check `canAddLiquidity()` → call `addLiquidity()` if eligible.
3. Check `canTriggerExecution()` → call `triggerExecution()` if eligible.
4. Check `canRetryFromStonks()` → call `retryFromStonks()` if eligible.

All functions are permissionless and idempotent; failed calls are retried on the next cycle. The keeper is integrated into the existing `MotionEnacterBot`, reusing existing infrastructure.

CoW Protocol requires a matching off-chain order payload submitted to its API before solvers can discover and fill an order. The keeper monitors for order-creation events, constructs the payload, and posts it to the CoW Protocol API. If submission fails, the order remains unfilled until expiry and standard recovery flows apply.

---

## Configuration

### Parameters

| Category                                     | Parameter                         | Description                                                                                                                                                                                                                 |
| -------------------------------------------- | --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Gate Thresholds**                          | `ethPriceFloorUSD`                | ETH price eligibility condition. `0` disables the gate.                                                                                                                                                                     |
| **Revenue**                                  | `dailyRevenueThresholdUSD`        | Daily revenue floor for surplus computation                                                                                                                                                                                 |
|                                              | `surplusShareBps`                 | Share of daily surplus allocated to NEST per trigger (basis points). In LP mode, this allocation is split equally between the LDO purchase leg and the wstETH pairing leg.                                                  |
| **Spending Cap**                             | `dailyCapUSD`                     | USD limit per trigger. Must be strictly less than `annualCapUSD`.                                                                                                                                                           |
| **Annual Cap**                               | `annualCapUSD`                    | USD limit on spend committed within a fixed 365-day period that resets when elapsed. Returned stETH from expired or unfilled orders is credited back via `accountForReturnedExcess()`, partially restoring annual headroom. |
| **Order Sizing**                             | `minOrderSizeUSD`                 | Minimum viable budget after all gates constrain it                                                                                                                                                                          |
| **Price Controls**                           | `orderPriceProtectionBps`         | Discount on estimated trade output to absorb price movement between submission and on-chain inclusion                                                                                                                       |
| **Pool Guards**                              | `poolSlippageToleranceBps`        | Slippage guard on Curve deposits (MEV protection)                                                                                                                                                                           |
|                                              | `poolPriceDivergenceToleranceBps` | Divergence guard against pool skew                                                                                                                                                                                          |
| **Revenue Sources**                          | `revenueSources`                  | Array of active revenue source addresses                                                                                                                                                                                    |
| **Provisioner**                              | `liquidityProvisioner`            | LiquidityProvisioner address. `address(0)` = treasury-only mode.                                                                                                                                                            |
| **Staleness Window** (on the RevenueSources) | `stalenessWindow`                 | Maximum allowed age of revenue source data before it is considered stale.                                                                                                                                                   |

All parameters are configurable via Aragon Voting (`DEFAULT_ADMIN_ROLE`) with validation on write. Like the rest of DAO treasury operations, this is not under Dual Governance. Invalid configuration reverts and preserves the previous valid state.

### Proposed Initial Values

| Parameter                  | Value                           | Notes                                                                                                                                 |
| -------------------------- | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `ethPriceFloorUSD`         | `0`                             | Gate disabled at launch; retained as a future governance lever                                                                        |
| `dailyRevenueThresholdUSD` | ~$109,589 (`$40,000,000 / 365`) | $40M annual revenue baseline                                                                                                          |
| `surplusShareBps`          | `5000` (50%)                    | 50% of daily surplus allocated to NEST. In LP mode: half of that budget goes to the LDO purchase leg, half to the wstETH pairing leg. |
| `dailyCapUSD`              | `$50,000`                       | Primary active safety limiter                                                                                                         |
| `annualCapUSD`             | `$10,000,000`                   | Annual ceiling                                                                                                                        |
| `poolSlippageToleranceBps` | `200` (2%)                      | Curve deposit MEV guard                                                                                                               |
| `stalenessWindow`          | `1 day`                         | Revenue data older than this is rejected                                                                                              |

### Roles and Authority

| Role                                                     | Held By                             | Key Powers                                                                                                                                                     |
| -------------------------------------------------------- | ----------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `DEFAULT_ADMIN_ROLE` + `MANAGER_ROLE` + `EMERGENCY_ROLE` | Aragon Voting                       | Full configuration control on both contracts, all emergency controls, asset recovery, role management. Requires full DAO vote.                                 |
| `MANAGER_ROLE` + `EMERGENCY_ROLE`                        | Treasury Management Committee (TMC) | Asset recovery, LP token management (`removeLiquidity`, `recoverERC20`), emergency pause/unpause, propose EasyTrack motions to fund the controller with stETH. |
| `EMERGENCY_ROLE`                                         | Emergency Committee / Multisig      | Pause controls on all NEST contracts, emergency order operations (cancel, revoke relayer). Cannot modify configuration.                                        |
| —                                                        | Permissionless Callers              | `triggerExecution()`, `retryFromStonks()`, `addLiquidity()`, `unwrapExcessWstEth()`                                                                            |

EasyTrack is used exclusively for funding operations (stETH transfers to the controller). All parameter changes require a full Aragon Voting DAO vote.

TMC may unwind LP positions and return underlying assets to treasury, but cannot reassign LP ownership or change system configuration without a full DAO vote.

---

## Emergency Controls

Five independent pause domains, each stoppable without affecting the others:

1. **NESTController execution pause** — stops `triggerExecution()` and `retryFromStonks()`. Assets accumulate in the controller.
2. **LiquidityProvisioner liquidity pause** — stops `addLiquidity()`. `unwrapExcessWstEth()` remains callable.
3. **Revenue source pauses** — stops data acceptance. Pausing all sources halts execution via `NoActiveRevenueSources` revert without engaging the execution pause.
4. **Stonks creation/signature pauses** — stops new order creation or freezes settlement of existing orders independently.
5. **OracleRouter pause** — stops price queries, causing trade execution and liquidity provisioning to revert.

Asset recovery is available at all times and is not subject to pause. NESTController and LiquidityProvisioner auto-unwrap wstETH to stETH before transferring to treasury.

**Accounting reset:** If cumulative buyback accounting becomes untenable — for example, after prolonged slashing recovery where accumulated deficit would block buybacks indefinitely — governance can call `resetBuybackAccounting` to zero out `allocatedForBuybacksUSD`, `cumulativeBuybacksUSD`, and `lastDailyAllocationUSD`. The next accounting cycle starts fresh. Requires a full DAO vote (`DEFAULT_ADMIN_ROLE`).

---

## Rationale

### Cumulative Surplus Model vs. Instant Daily-Spend

The cumulative model was chosen because it prevents two failure modes of an instant daily-spend model: overspending during consecutive good days (no memory of past losses) and the inability to carry forward unused capacity. After a loss period, new surplus must cover the accumulated deficit before buybacks resume — a natural, automatic risk brake without additional oracle dependencies.

Analytics modeling confirmed that quarterly revenue smoothing produces no meaningful improvement in cumulative execution volume over the daily cadence, while adding significant accounting complexity and oracle overhead.

### ETH Price Gate Set to Zero at Launch

The cumulative surplus mechanism provides natural ETH-price sensitivity without an explicit gate. At a $40M/year cost baseline (~$109k/day), the break-even ETH price is approximately $2,700 at current protocol revenues. Below that price, the protocol automatically runs a daily deficit, and no buybacks occur — no price oracle required. The `ethPriceFloorUSD` parameter is retained in the design for future governance reactivation if conditions change.

### $50,000 Daily Cap as Primary Safety Limiter

The daily cap is the primary active safety guardrail. Historically, very few days produce surplus exceeding $50k. In the event of oracle corruption or bad revenue inputs, the cap ensures maximum exposure over a ~6-day governance response window is bounded at ~$300k. The cap is independently configurable and can be raised by governance as surplus and on-chain liquidity grow.

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

Expired and unfilled orders are retried through a bounded recovery flow (`retryFromStonks()`) rather than silently skipped. This is a resolved design decision: recovered stETH is reused for a new order without additional funding, while excess wstETH from partial fills is unwrapped and returned to the controller via a cooldown-gated permissionless function.

---

## Security Considerations

### Oracle Manipulation

Price feeds (ETH/USD, stETH/USD via two-hop, LDO/ETH) are sourced from Chainlink via the shared OracleRouter. All feeds are validated for staleness and answer validity independently. The daily cap bounds maximum exposure per trigger to $50k regardless of oracle accuracy. The OracleRouter can be paused independently for rapid incident response.

### Permissionless Execution Risk

`triggerExecution()` is callable by anyone. Budget computation uses on-chain oracle prices at call time, with `orderPriceProtectionBps` reducing `minBuyAmount` to absorb price movement between submission and inclusion. Daily and annual caps enforce hard spending bounds regardless of oracle precision.

### Revenue Reporting Integrity

`StakingRevenueSource` access to `pushTokenRate` is restricted to `REPORTER_ROLE` holders (granted to `TokenRateNotifier` at deployment). Out-of-order calls by unauthorized callers are rejected. Revenue sources can be individually paused.

### Slashing Events

After a slashing event, the revenue rate baseline stays at its pre-slash level. Staking revenue returns zero until the protocol fully recovers, creating a natural buyback pause proportional to slash severity. Governance can reset accounting if accumulated deficit becomes indefinitely blocking.

### No asset custody

All purchased assets remain DAO-owned throughout the execution path. stETH flows only through the controlled pipeline: Aragon Agent → NESTController → Stonks → Order → LiquidityProvisioner (LP mode) or Aragon Agent (treasury-only mode). No unauthorized exit path exists for sell-side stETH.

### Emergency Response Time

Five independent pause domains allow targeted response without halting the full system. The Emergency Committee can pause without a full DAO vote. Asset recovery functions are always available and not subject to pause. At a $50k daily cap, the maximum exposure over a 6-day governance response window is ~$300k.

### Missing Lido AccountingOracle reports

If an AccountingOracle report is delayed or skipped, no rebase fires and StakingRevenueSource is not updated. Once it ages past STALENESS_WINDOW_SECONDS, triggerExecution reverts with RevenueSourceStale, halting accounting and order creation until a fresh report arrives. The next pushTokenRate normalizes the elapsed-period revenue back to a daily rate, so a multi-day gap produces a single proportionally-scaled allocation rather than a spike. No buybacks occur during the gap and the system resumes automatically.

---

## References

- Forum discussion: [Liquid Buybacks: NEST Execution with LDO/wstETH Liquidity](https://research.lido.fi/t/liquid-buybacks-nest-execution-with-ldo-wsteth-liquidity/10894)
- [Stonks Repository](https://github.com/lidofinance/stonks)
- [Stonks v2 working branch](https://github.com/lidofinance/stonks/tree/feature/dao-781-develop-stonks-v2-poc)
- [Stonks v2 Specification](https://hackmd.io/p_ZC5s9tRAOMavh5nVOerw)
- [NEST Pull Request](https://github.com/lidofinance/stonks/pull/28)
- [CoW Protocol documentation](https://docs.cow.fi/)
- [Curve TwoCrypto](https://github.com/curvefi/twocrypto-ng)
- [TokenRateNotifier](https://github.com/lidofinance/core/blob/master/contracts/0.8.9/TokenRateNotifier.sol)
