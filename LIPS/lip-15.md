---
tags: LIP
lip: 15
title: Protocol safeguards. Resume role
status: Proposed
author: Eugene Mamin, Alexey Potapkin
discussions-to: TBA
created: 2022-05-11
updated: 2022-05-11
---

# Protocol safeguards. Resume role.

## Abstract 

We propose to add `RESUME_ROLE`, which would allow to call `unpause` function of the `Lido.sol` contract to differ (theoretically) entity that can pause from one that can resume it.

## Motivation

By completing the protocol-wide `PAUSE_ROLE` with `RESUME_ROLE` we would allow to push protocol on hold by a special emergency entity (e.g. emergency multisig committee) while resume protocol only by the conventional quorum-reached Lido DAO vote using Aragon Voting. Voting lasts for 72h since [LIP-4](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-4.md) was implemented and it could be too long to wait in case of the protocol emergency.

## Specification

### Role

We're adding a separate role constant in [`Lido.sol`](https://github.com/lidofinance/lido-dao/pull/410/files#diff-6717c01b56e50e5f2f2dbc8827dd7397d92ad5aa09924816266e9e7f7b886113R54)

```
bytes32 constant public RESUME_ROLE = keccak256("RESUME_ROLE");
```

### Checking the role

We're checking the role in [`resume()`](https://github.com/lidofinance/lido-dao/pull/410/files#diff-6717c01b56e50e5f2f2dbc8827dd7397d92ad5aa09924816266e9e7f7b886113R315) method

```
function resume() external {
    _auth(RESUME_ROLE);

    _resume();
}
```

## Backward compatibility

We propose to assign `RESUME_ROLE` to the DAO Voting contract initially (so it would preserve the ability to pause/resume protocol by the DAO vote solely).


## Pull request

The proposed changes can be found in the following [`PR #410`](https://github.com/lidofinance/lido-dao/pull/410/files).