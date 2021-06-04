| lip: | title: | status: | author: | discussions-to: | created: | updated: |
| --- | --- | --- | --- | --- | --- | --- |
| 3 | Easy Track Frictionless Motions | WIP | Bogdan Kovtun (psirex), Gregory Stepanov (grstepanov) | https://research.lido.fi/t/lip-3-easy-track-motions-for-dao-routine-operations/680 | 2021-06-04 | |

# Summary

## Problem

Lido DAO governance currently relies on Aragon voting model. This means DAO approves or rejects proposals via direct governance token voting. Though transparent and reliable, it is not a convenient way to make decisions only affecting small groups of Lido DAO members. Besides, direct token voting doesn't exactly reflect all the decision making processes within the Lido DAO and is often used only to rubberstamp an existing consensus.
There are a few natural sub-governance groups within the DAO, e.g. validators commitee, financial operations team and LEGO commitee. Every day they need to take routine actions only related to their field of expertise. The decisions they make hardly ever spark any debate in the comunity, and votings on such decisions often struggle to attract wider DAO attention and thus, to pass.

## Proposed solution

We propose Easy Track frictionless motions as a solution to this problem. 
Easy Track motion is a lightweight voting considered to have passed if the minimum objections threshold hasn’t been exceeded. As opposed to traditional Aragon votings, Easy Track motions are cheaper (no need to vote ‘pro’, token holders only have to vote ‘contra’ if they have objections) and easier to manage (no need to ask broad DAO community vote on proposals sparking no debate). 

## Use cases for Easy Track motions

There are three types of votings run periodically by the Lido DAO that are proposed to be wrapped into the new Easy Track motions.
- Node Operators increasing staking limits
- Funds being allocated into reward programs
- Funds being allocated to LEGO program

## Sub-governance groups

There's sub-governance group within the Lido DAO for each of the voting cases listed above. Those groups have been formed intentionally or in a more natural way, and at this point creating and maintaining periodical Aragon votings falls within their remits.
The proposed improvement will require formalizing sub-governance groups:

1. **Node Operators Commitee** to start staking limit increase motions.
2. **Financial Operations Team** to start funds allocation motions (i.e. LEGO and reward programs).

Node Operators Commitee should include withdraw addresses of active node operators partnering with Lido, while Financial Operations Team should consist of established DAO members' addresses and should be created or modified via normal Aragon votings. Financial Operations Team will be represented as a Gnosis Safe Multisig with access to corresponding Easy Track features.

# Specification

To create a feature as flexible and extendable as possible, we propose implementing Easy Track functionality within several contracts rather than building a single contract for it. 
The main contract will be the Easy Track Registry, and separate Executor contracts will inherit from it to make different types of Easy Track motions possible.
This architecture aims to support further extension by adding new types of Easy Track motions without replacing the whole system.
> The statement below is abstract, please visit [specification.md](https://github.com/lidofinance/easytrack/blob/master/README.md) for full detailed specification.

## Easy Track Registry

Easy Track Registry is the main contract containing the basic motion logic and data shared by all the Easy Track types, including who has started the motion, when, for how long LDO holders will be able to submit objections, what would be the minimum disapproval objections threshold, current objections submitted. 
Registry will also store motion related encoded byte type data. We propose using byte data type due to various sets of parameters required for different motion types.
```solidity=
struct Motion {
    uint256 id;
    address executor;
    uint256 duration;
    uint256 startDate;
    uint256 snapshotBlock;
    uint256 objectionsThreshold;
    uint256 objectionsAmount;
    uint256 objectionsAmountPct;
    bytes data;
}
```
Besides, Easy Track Registry will store a mapping listing active motions (the ones created but neither enacted nor canceled yet). Once enacted or canceled, the motion will be permanently deleted, since there is really no need to interact with past motions in any way.
The Registry contract should be owned by DAO and include some methods only accessible to Lido DAO, i.e.:
- setting motion duration;
- setting objections threshold;
- setting maximum active motions count;
- adding or removing motion types.


## Easy Track Executors

Logic for specific motions will be stored in Executor contracts. These contracts will implement `IEasyTrackExecutor` interface.

```solidity=
interface IEasyTrackExecutor {
    function beforeCreateMotionGuard(
        address _caller,
        bytes memory _data
    ) external;

    function beforeCancelMotionGuard(
        address _caller,
        bytes memory _motionData,
        bytes memory _cancelData
    ) external;

    function execute(
        bytes memory _motionData,
        bytes memory _enactData
    ) external returns (bytes memory);
}
```

List of Executors added is stored in the Registry and only the DAO can add more or remove them. In similar way, each Easy Track Executor stores the Registry contract address in order to control access to Executor's methods.
Both on motion creation and motion cancelation, methods `beforeCreateMotionGuard` and `beforeCancelMotionGuard` check if the `msg.sender` matches the Easy Track Registry address.

### Node Operators Easy Track Executor

Node Operators Easy Track Executor is an Executor contract for Node Operators Commitee to start or cancel motions to increase their staking limits.
Contract stores address of `NodeOperatorsRegistry` contract:

```solidity=
NodeOperatorsRegistry public nodeOperatorsRegistry;
```

When creating a new motion, multiple checks run as follows:
- `NodeOperatorsRegistry` contains node operator with given id.
- Node operator is not disabled.
- Withdrawal address of the node operator matches the address of `_caller`.
- New value of `_stakingLimit` is greater than the current value and less than or equal to the number of total signing keys of the node operator.

When enacting a passed motion, additional checks run:
- `NodeOperatorsRegistry` contains node operator with given id.
- Node operator is not disabled.
- New value of `_stakingLimit` is greater than the current value and less than or equal to the number of total signing keys of the node operator.

### Rewards Easy Track Executors

Rewards Easy Track Executors implements logic to control the whitelist of allowed LDO recipients and to transfer LDO into one or multiple allowed recipient addresses.
There are three types of operations managed by Rewards Easy Track motions and thus three Executor contracts to manage each type of motions:
- adding a recipient address into the allowed recipients list (i.e. adding a new reward program into Lido rewards stack)
- removing a recipient address from the allowed recipients list
- transfering LDO into a whitelisted recipient address (i.e. refilling an ongoing Lido reward program)

Any of these actions can only be initiated as a motion by Financial Operations Team Multisig.

### Lego Easy Track Executor

LEGO is Lido Ecosystem Grants Organization funded from the Lido Treasury. Funds allocation is a routine process handled by Financial Operations Team and thus can be optimized with Easy Track.
Lego Easy Track Executor implements logic to allocate LDO, stETH or ETH for LEGO grants and make sure the methods can only be called by Financial Operations Team Multisig.
Contract stores address of LEGO program:

```solidity=
address private legoProgram;
```



# Sanity and security

Easy Track motions are meant to be as frictionless as possible, and thus require reliable security mechanisms and sanity checks. To prevent malicious actions we propose putting following limitations in place:
- It should be impossible to set objections threshold at more than 5% of LDO supply. Default objections threshold should be set much lower at 0.5% of LDO supply. Easy to start – easy to reject.
- It should be impossible to set motion duration at less than 42 hours. For any motion there should be enough time for DAO to submit objections and reject it. For more urgent issues there's always an option to start a regular Aragon voting.
- It should be impossible to spam motions. The default limit for simultaneous active motions will be set at 48 and it can only be increased up to 128 by the DAO.

# UI

Beside contracts, Easy Track requires dedicated UI providing functionality similar to Aragon voting app.
We propose designing and building four views to overview and manage Easy Track motions as follows:
- Motion list view showing all the ongoing votings, their current status and time left to submit objections. On this page, it will also be possible to look through historical motions passed and canceled. Data for this historical section will be collected via subgraph.
- Motion view showing details for specific motion (description, status, current objections submitted) and action button (submit objections).
- Motion creation view to start new motions. This functionality will be only available to respective sub-governance group.
- Additional static page listing sub-governance groups and commitees to make it clear who are entities entitled to start motions and what specific motion types can be started by each group.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
