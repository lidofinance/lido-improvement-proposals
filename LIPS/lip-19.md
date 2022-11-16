---
lip: 19
title: Easy Track. Limits for committees
status: Proposed
author: Ekaterina Zueva, Victor Suzdalev, Eugene Mamin
discussions-to: https://research.lido.fi/t/lip-19-easy-track-limits-for-committees/3237
created: 2022-08-26
updated: 2022-11-15
---

# LIP-19. Easy Track. Limits for committees

## Simple Summary

This document proposes the implementation of a set of factories for Easy Track. The set is designed to transfer Treasury funds within periodic on-chain limits.

## Motivation

To streamline routine governance operations, some Lido DAO activities are governed by [committees](https://lido.fi/governance#committees). For ease of operations, transactions from committees go through Easy Track. Votings on Easy Track are based on the principle of vetoing. Among other operations, committees can transfer funds from the Treasury. Since the number of committees and the amount of spending is growing significantly, it's highly desirable to have on-chain limits.

Easy Track motions already have the limit on single token transfer size (see [LIP-13](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-13.md)). In addition, there is a need to have separate limits for every committee that are set per period and correlate with the real budgets of the committee.

The proposal describes the design for periodic on-chain limits for committees' payments.

## Specification

### Overview

The idea of using limits is relatively straightforward:
- The DAO sets an on-chain limit per period and period duration for each committee via Aragon voting.
- Every time the committee wants to create a motion, it should be checked whether motion amount is within the committee budget.
- Every time the motion is enacted, the check is performed again. An attempt to overspend must lead to a revert.
- The amount of the enacted motion is automatically added to the expenses of the current period.
- The expenses of the current period are reset to zero at the beginning of a new period.

### Technical specification

It is recommended to read [Easy Track Specification](https://github.com/lidofinance/easy-track/blob/master/specification.md) proceeding with this section.

It is proposed to create a new special set of  [`EVMScriptFactories`](https://github.com/lidofinance/easy-track/blob/master/specification.md#evmscript-factories) and a new `LimitsChecker` contract which is inherited by `AllowedRecipientsRegistry`.

Contracts based on the existing RewardPrograms set (`RewardProgramsRegistry`, `AddRewardProgram`, `RemoveRewardProgram`, `TopUpRewardProgram`) have the following differences from the existing ones:
* The term "RewardProgram" was substituted for a more general "AllowedRecipient",
* `AllowedRecipientsRegistry` (ex-`RewardProgramsRegistry`) stores and manages limits,
* `TopUpAllowedRecipients` (ex-`TopUpRewardProgram`) can check if the payment satisfies the balance for the current period on motion creating and enaction.

The following flow is proposed for storing and interacting with limits:
* As the `TopUpRewardProgram` contract, the new `TopUpAllowedRecipients` contract stores a `token` for payments. It's set on deployment. If it is necessary to make payments in different tokens, then a separate set of factories for each token should be deployed and a limit must be set for each token.
* The `TopUpAllowedRecipients` contract stores limit parameters: the period duration in months (`periodDurationMonths`) and the total limit (`limit`) - the maximum amount of funds that can be spent within the period.
* The expenses of the current period (`spentAmount`) are also stored in the contract and is updated every time a new motion is enacted.
* During checks, the current payment amount is compared with the "spendable balance" (limit - spent amount).
* When a new period starts, the `spentAmount` is reset to zero with the first motion's enactment in the period.
* To track when the new period has begun, the contract stores the time stamp of the end of the current period (`currentPeriodEndTimestamp`).

The following figure provides an overview of the limits flow:

![](https://hackmd.io/_uploads/H1mm_0kLj.png)


### Design tradeoffs

During implementation, we've encountered the following questions and tradeoffs:

#### 1. Workaround for the same check at the start and enactment of motion

We wanted limit checks to happen on both motion's start and enactment. The check should be very simple: to compare the motion amount with the spendable balance.
```solidity
    return _motionAmount <= limit - spentAmount;
```
The existing design of `EasyTrack` is made so that there's only one method to add the check:`createEVMScript`. It is called on both the creation and the enactment but doesn't differentiate them, also, it's stateless and idempotent.

The problem is that we can't start a motion close to the end of the period with extremely few funds left, if spendable balance is less than the motion's amount. But we assume that enactment will be in the next period when the spent amount will be renewed. So we'd like such a motion could be started.

![](https://hackmd.io/_uploads/H1DQOqori.png)

Redesigning and redeploying the entire `EasyTrack` is a tough operational decision. The design space was restricted by the existing implementation.

It was decided to create two checks:
1. The first check was implemented in `createEVMScript` method that happens on start and enactment. A special condition was added to the check to avoid the abovementioned problem. If a motion is created after 72 hours before the period's end, its amount is compared with the total limit, not with `limit - spentAmount`.

```solidity
function isUnderSpendableBalance(uint256 _payoutAmount, uint256 _motionDuration)
    external
    view
    returns (bool)
{
    if (block.timestamp + _motionDuration >= currentPeriodEndTimestamp) {
        return _payoutAmount <= limit;
    } else {
        return _payoutAmount <= limit - spentAmount);
    }
}
```

This method can be also used externally, for example in UI, to show if a motion can be created in current period.

2. The second check was added to the EVM script. EVM script is created inside the factory and executed when motion is enacted. So the check is performed only on the motion's enactment directly before payment. It compares the motion's amount with the current spendable balance and reverts if the motion's amount exceeds the remaining budget.

#### 2. How to understand that a new period has begun and change the corresponding parameters

We need to check that a new period has started and reset the previously accumulated spent amount to zero.`spentAmount` zeroing and `currentPeriodEndTimestamp` shift occur in the `updateSpentAmount` method, which is called immediately before the payment is made when the EVM Script is executed. Thus the first motion enactment in the period resets these parameters:
```solidity
if (block.timestamp >= currentPeriodEndTimestamp) {
    alreadySpent = 0 // reset spend amount if new payments period started
    currentPeriodEndTimestamp = newPeriodEnd // set a new period boundary
}
```
Boundary shifting can happen only when a motion is enacted, also due to the Easy Track design. The only moment that can change the state is the EVM script execution when a motion is enacted.
If no enactment occurred in the current period, then the spending and the end of the period will remain unchanged.  In this case, the `getPeriodState` method will return `alreadySpentAmount` and `spendableBalanceInPeriod` for the period with boundaries from `_periodStartTimestamp` to `_periodEndTimestamp`.

```solidity
function getPeriodState()
    external
    view
    returns (
        uint256 _alreadySpentAmount,
        uint256 _spendableBalanceInPeriod,
        uint256 _periodStartTimestamp,
        uint256 _periodEndTimestamp
    )
```

For the same reason, if the period advanced and no calls to `updateSpentAmount` or `setLimitParameters` were made, then the `spendableBalance` method will return the spendable balance corresponding to the previous period.

```solidity
function spendableBalance() external view returns (uint256)
```

#### 3. How to align the periods with a real calendar.

To better represent the real financial flows, limit periods are delineated by the real-world calendar periods (months, quarters). Year-wide periods start on 1st Jan every year, quarter-wide periods begin on 1st Jan, 1st Apr, and so on. In this case, it is necessary to correctly calculate the timestamp for the end of a new period.

In the `LimitsChecker` contract, we store the duration of the period in months `periodDurationMonths`. It can take values: 1, 2, 3, 6, and 12. And`currentPeriodEndTimestamp` - timestamp of the end of the current period. Strictly speaking in `currentPeriodEndTimestamp` variable stored the beginning of the next period. That is, for the quarter from July to September, `currentPeriodEndTimestamp` will be October 1st. All periodic parameters are used in UTC time.

To calculate the next period boundary, we use the [`IBokkyPooBahsDateTimeContract`](https://github.com/bokkypoobah/BokkyPooBahsDateTimeLibrary/blob/master/contracts/BokkyPooBahsDateTimeLibrary.sol) and it's method `addMonths`. It adds the required number of months (`periodDurationMonths`) to the end of the current period (`currentPeriodEndTimestamp`):

```solidity
function addMonths(uint256 timestamp, uint256 _months)
    external
    pure
    returns (uint256 newTimestamp);
```

#### 4. How to avoid math underflow error while calculating the current balance and what consequences this can lead to.

It is necessary to avoid math underflow error in the method:

```solidity
function _spendableBalance(uint256 _limit, uint256 _spentAmount)
    internal
    pure
    returns (uint256)
{
    return _limit - _spentAmount;
}
```
Generally speaking, `spentAmount` should always be less than `limit`,  except in one case, when the DAO makes the limit less than the current `spentAmount`. Limits are changed in the `setLimitParameters` method. To avoid the underflow error, we added the check:
```solidity
function setLimitParameters(uint256 _limit, uint256 _periodDurationMonths)
    external
    onlyRole(SET_LIMIT_PARAMETERS_ROLE)
{
    ...
    /// set spent to _limit if it's greater to avoid math underflow error
    if (spentAmount > _limit) {
        spentAmount = uint128(_limit);
    }
    ...
}
```
But it led to a nuance. If we first make the `limit` less than the current `spentAmount`, and then increase `limit`, we will lose part of the expenses already spent and can double spend the funds within the limit. To avoid this, the DAO can increase `spentAmount` using the method `updateSpentAmount`. Spent during the period can be restored using the event:
```solidity
event SpendableAmountChanged(
    uint256 _alreadySpentAmount,
    uint256 _spendableBalance,
    uint256 indexed _periodStartTimestamp,
    uint256 _periodEndTimestamp
);
```


## Links

- [Easy Track limits and budgets: ADR](https://research.lido.fi/t/easytrack-limits-and-budgets-architecture-design-record/2537)
- [LIP-13: Easy Track payments limits](https://research.lido.fi/t/lip-13-easy-track-payments-limit/1670)
- [Easy Track Guide](https://docs.lido.fi/guides/easy-track-guide)
- [Easy Track Specification](https://github.com/lidofinance/easy-track/blob/master/specification.md)
