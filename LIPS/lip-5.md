---
lip: 5
title: Mitigations for deposit front-running vulnerability
status: Proposed
author: Eugene Pshenichnyy, Sam Kozin, Victor Suzdalev, Vasiliy Shapovalov
discussions-to: https://research.lido.fi/t/mitigations-for-deposit-front-running-vulnerability/1239
created: 2021-10-15
updated: 2021-10-15
---

# Mitigations for deposit front-running vulnerability

## Abstract

On Tuesday, Oct 5, the vulnerability allowing the malicious Node Operator to intercept the user funds on deposits to the Beacon chain in the Lido protocol was reported to [Immunefi](https://immunefi.com/). On Wednesday, Oct 6, [the short-term fix was implemented](https://blog.lido.fi/vulnerability-response-update/). **Currently, no user funds are at risk, but the deposits to the Beacon chain are paused**.

This document outlines the long-term mitigation allowing deposits in the Lido protocol.

The vulnerability could only be exploited by the Node Operator front-running the `Lido.depositBufferedEther` transaction with direct deposit to the [DepositContract](https://etherscan.io/address/0x00000000219ab540356cbb839cbe05303d7705fa) of no less than 1 ETH with the same validator public key & withdrawal credentials different from the Lido's ones, effectively getting control over 32 ETH from Lido.

To mitigate the vulnerability, Lido contracts should be able to check that Node Operators' keys hadn't been used for malicious pre-deposits. `DepositContract` provides the Merkle root encoding of all the keys used in staking deposits. We propose to establish the **Deposit Security Committee** checking all deposits made and approving the current `deposit_data_root` for Lido deposits.

The idea was outlined on [research forum post](https://research.lido.fi/t/mitigations-for-deposit-front-running-vulnerability/1239#d-approving-deposit-contract-merkle-root-7) as the option `d)`.

The Merkle root data should be checked & signed by the guardian committee off-chain to save gas costs. Anyone with the signed `depositRoot` can call `depositBufferedEther` sending the ether from the Lido buffer to the `DepositContract`. In case any deposit happens before the `depositBufferedEther` transaction, the Merkle root on the `DepositContract` is changed and the Lido deposit transaction is reverted.

```solidity=
function depositBufferedEther(
    bytes32 depositRoot,
    Signature[] memory signatures
) external {
    bytes32 onchainDepositRoot = IDepositContract(depositContract).get_deposit_root();
    require(depositRoot == onchainDepositRoot, "deposit root changed");
    
    require(_verifySignatures(depositRoot, signatures), "invalid signatures");
    
    _depositBufferedEther();
}
```

The outlined approach requires relatively small changes of the protocol, preserving the current operations flow between the protocol, Node Operators, and the Lido DAO.

To implement the proposed mitigation, the DAO would have to:
1) implement the upgrade for the protocol smart contracts;
2) establish a committee;
3) write & check the software for monitoring deposits, approving the Merkle roots & exchanging and publishing signed approval messages.

As all things require a significant amount of work & time, we propose to prepare to start with a small committee to unblock deposits in the protocol as soon as possible, simultaneously assembling a bigger team to run a guardian daemon as well.


## NodeOperatorsRegistry keyOpIndex and re-org protection

To reliably check that the next deposit is safe for the protocol, it is necessary to check that there is no intersection between the set of all historical public keys used for deposits in the `DepositContract` and the set of all ready to use public keys in [NodeOperatorRegistry](https://github.com/lidofinance/lido-dao/blob/master/contracts/0.4.24/nos/NodeOperatorsRegistry.sol). This means that it is not enough to keep track of only the `deposit_data_root`, but it is also necessary to keep track of public key set in the `NodeOperatorRegistry`.

One could build the same Merkle-tree solution for the `NodeOperatorRegistry`, but this would require a major smart contract rewrite. Therefore, we propose to introduce a monotonically increasing counter, which we will increment for every modification of the depositable public key set, when:
* a Node Operator's key(s) is added;
* a Node Operator's key(s) is removed;
* a Node Operator's approved keys limit is changed;
* a Node Operator was activated/deactivated.


The pair of `depositRoot` from `DepositContract` and `keysOpIndex` from `NodeOperatorRegistry` provides all the data required for the guardian committee to sign & the deposit contract to check before sending the deposits. 

However, since the `keysOpIndex` doesn't uniquely correspond to the set of keys possible for the deposit (like Merkle root), it's possible to reorganize the chain and maliciously use the signatures of the committee members who observed a different state during the signing.

To protect against hostile chain re-org, we also propose to pass the `blockNumber` and `blockHash` to `depositBufferedEther`.


```solidity=

function depositBufferedEther(
    bytes32 depositRoot,
    uint256 keysOpIndex,
    uint256 blockNumber,
    bytes32 blockHash,
    Signature[] memory sortedGuardianSignatures
) external {
    bytes32 onchainDepositRoot = IDepositContract(DEPOSIT_CONTRACT).get_deposit_root();
    require(depositRoot == onchainDepositRoot, "deposit root changed");
    
    uint256 onchainKeysOpIndex = INodeOperatorsRegistry(nodeOperatorsRegistry).getKeysOpIndex();
    require(keysOpIndex == onchainKeysOpIndex, "keys op index changed");
    require(blockhash(blockNumber) == blockHash, "unexpected block hash");
    
    require(_verifySignatures(depositRoot, sortedGuardianSignatures), "invalid signatures");
    
    _depositBufferedEther();
}
```

After committee members sign the pair of `depositRoot` and `keysOpIndex`, then in the case when one of the sets changes, the off-chain proof expires and deposit transaction reverts.

 
## Deposit Security Committee

As stated above, we propose to establish the **Deposit Security Committee** dedicated to ensuring the safety of deposits on the Beacon chain:
1) monitoring the history of deposits and the set of Lido keys available for the deposit, signing and disseminating messages allowing deposits;
2) signing the special message allowing anyone to pause deposits once the malicious Node Operator pre-deposits are detected.

Each member must generate an EOA address to sign messages with their private key. The addresses of the committee members will be added to the smart contract.

To make a deposit, we propose to collect a quorum of 2/3 of the signatures of the committee members. Members of the committee can collude with node operators and steal money by signing bad data that contains malicious pre-deposits. To mitigate this we propose to allow a single committee member to stop deposits and also enforce space deposits in time (e.g. no more than 150 deposits with 150 blocks in between them), to provide the single honest participant an ability to stop further deposits even if the supermajority colludes.

The guardian himself, or anyone else who has a signed pause message, can call `pauseDeposits` that pauses `DepositSecurityModule`.

To prevent a replay attack, the guardians sign the block number at the time of which malicious pre-deposits are observed. After a certain number of blocks (`pauseIntentValidityPeriodBlocks`) message becomes invalid.

```solidity=
function pauseDeposits(uint256 blockNumber, Signature memory sig) external {
    address guardianAddr = msg.sender;
    int256 guardianIndex = _getGuardianIndex(msg.sender);

    if (guardianIndex == -1) {
        bytes32 msgHash = keccak256(abi.encodePacked(PAUSE_MESSAGE_PREFIX, blockNumber));
        guardianAddr = ECDSA.recover(msgHash, sig.r, sig.vs);
        guardianIndex = _getGuardianIndex(guardianAddr);
        require(guardianIndex != -1, "invalid signature");
    }

    require(
        block.number - blockNumber <= pauseIntentValidityPeriodBlocks,
        "pause intent expired"
    );

    if (!paused) {
        paused = true;
        emit DepositsPaused(guardianAddr);
    }
}
```

This design ensures the protocol's robustness even with a single honest committee member. The impact is limited to about, say 4800 ETH at most (150 keys allowed within a time window, 32 ETH deposited to every key). The false-positive pause would stop the deposits for a couple of days because only DAO should be able to unpause deposits.

False-positive stop reveals the rogue committee member, which is good for the protocol. Overall, DoS seems not to be a big problem, and if it would be, the DAO can change the mitigation.

Below is the pseudocode of the `depositBufferedEther` function, updated following the above considerations.

```solidity=
function depositBufferedEther(
    uint256 maxDeposits,
    bytes32 depositRoot,
    uint256 keysOpIndex,
    uint256 blockNumber,
    bytes32 blockHash,
    Signature[] memory sortedGuardianSignatures
) external {
    bytes32 onchainDepositRoot = IDepositContract(DEPOSIT_CONTRACT).get_deposit_root();
    require(depositRoot == onchainDepositRoot, "deposit root changed");

    require(!paused, "deposits are paused");
    require(quorum > 0 && sortedGuardianSignatures.length >= quorum, "no guardian quorum");

    require(maxDeposits <= maxDepositsPerBlock, "too many deposits");
    require(block.number - lastDepositBlock >= minDepositBlockDistance, "too frequent deposits");
    require(blockhash(blockNumber) == blockHash, "unexpected block hash");

    uint256 onchainKeysOpIndex = INodeOperatorsRegistry(nodeOperatorsRegistry).getKeysOpIndex();
    require(keysOpIndex == onchainKeysOpIndex, "keys op index changed");

    _verifySignatures(depositRoot, keysOpIndex, sortedGuardianSignatures);

    ILido(LIDO).depositBufferedEther(maxDeposits);
    lastDepositBlock = block.number;
}
```

## On-Chain Deposit Flow

Instead of adding the code directly to the `Lido` contract, we propose to extract the described functionality to a separate non-upgradable contract – `DepositSecurityModule`.

This approach will reduce the gas price of an unsuccessful deposit transaction (e.g. due to deposit data root change) and reduce the number of modifications to the `Lido` contract - it would be necessary only to wrap `depositBufferedEther` with ACL role and allow only `DepositSecurityModule` contract to call the function.

`DepositSecurityModule` implements `Ownable` interface, so only the owner could change configuration parameters such as guardian addresses list and security parameters(e.g `minDepositBlockDistance`, `maxDeposits`,...).

After deploying and setting up the contract, the ownership must be transferred to the [DAO Agent](https://etherscan.io/address/0x3e40D73EB977Dc6a537aF587D48316feE66E9C8c).


```solidity=
contract DepositSecurityModule is Ownable {
    function setMinDepositBlockDistance(uint256 newValue) external onlyOwner;
    function setMaxDeposits(uint256 newValue) external onlyOwner;
    function setPauseIntentValidityPeriodBlocks(uint256 newValue) external onlyOwner;

    function addGuardians(address[] memory addresses, uint256 newQuorum) external onlyOwner;
    function addGuardian(address addr, uint256 newQuorum) external onlyOwner;
    function removeGuardian(address addr, uint256 newQuorum) external onlyOwner;
    function setGuardianQuorum(uint256 newValue) external onlyOwner 
    
    /**
     * The function might be called by the guardian directly   
     * OR by passing a valid guardian's signature.
     *
     * blockHeight is required to prevent reply attack.
     */
    function pauseDeposits(uint256 blockHeight, Signature memory sig) external;
    function unpauseDeposits() external onlyOwner;
    
    /**
     * Calls Lido.depositBufferedEther(maxDeposits),
     * which is not callable in any other way.
     */
    function depositBufferedEther(
        uint256 maxDeposits,
        bytes32 depositRoot,
        uint256 keysOpIndex,
        uint256 blockNumber,
        bytes32 blockHash,
        Signature[] memory sortedGuardianSignatures
    ) external;
}
```

## Off-chain part

### Council Daemon
Each committee member has to run Council Daemon that monitors the validators' public keys in the `DepositContract` and `NodeOperatorRegistry`. The daemon must have access to the committee member's private key to be able to perform ECDSA signing. 

The daemon constantly watches all updates in `DepositContract` and `NodeOperatorRegistry`:
* If the state is correct, it signs `DEPOSIT_MESSAGE` and emits it to an off-chain message queue. The daemon must regularly publish fresh signed `DEPOSIT_MESSAGE` (e.g. every 50 blocks) because the old ones will expire every 256 blocks.
* If the state has malicious pre-deposits, it signs `PAUSE_MESSAGE` at the current block, emits it to a message queue, and attempts to send `pauseDeposits()` tx. The daemon must publish signed pause messages until the `DepositSecurityModule ` pause.

```javascript=
DEPOSIT_MESSAGE = concat(
    ATTEST_MESSAGE_PREFIX,
    depositRoot,
    keysOpIndex,
    blockNumber,
    blockHash 
);

PAUSE_MESSAGE = concat(
    PAUSE_MESSAGE_PREFIX,
    blockNumber
);
```

`ATTEST_MESSAGE_PREFIX` and `PAUSE_MESSAGE_PREFIX` should be retrieved from `DepositSecurityModule` smart contract.

### Depositor Bot

The proposal makes the deposits in Lido permissioned: one must gather the quorum of signatures and pass several on-chain checks to make a deposit. Therefore, this process must be automated.

Lido dev team (and potentially anyone with access to the message queue) will run a deposit bot that will deposit buffered ether. The bot will listen to an off-chain message queue and:
* if there is a “something's wrong” message, try to pause a protocol with high gas
* else if there’s a quorum on the fresh deposit data root, there’s money in the buffer and the gas is low enough, try to call `depositBufferedEther`

## Additional changes not related to the vulnerability

1. Mitigation for the issue: [a Node Operator can circumvent DAO validator key approval](https://github.com/lidofinance/lido-dao/issues/141);
2. Set `DEFAULT_MAX_DEPOSITS_PER_CALL` to 150 in [Lido.sol#59](https://github.com/lidofinance/lido-dao/blob/master/contracts/0.4.24/Lido.sol#L59);
3. Prevent creating node operators with [a non-zero staking key limit](https://github.com/lidofinance/lido-dao/blob/master/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L112);
4. Add batch key removal methods for [removeSigningKey](https://github.com/lidofinance/lido-dao/blob/master/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L260) and [removeSigningKeyOperatorBH](https://github.com/lidofinance/lido-dao/blob/master/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L272);
5. Emit an event on [trimUnusedKeys](https://github.com/lidofinance/lido-dao/blob/master/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L207) call.
    
## Action Plan

It is very important to unpause the deposits to the Beacon chain, so we assume the deployment in several stages:

1) multiple committee members on a centralized message queue, council-daemon, deposit bot, and monitoring;
2) multiple committee members on a p2p network.

Updating the protocol and deploying smart contracts are required only once.

### Smart contract deploy plan

1. Deploy `Lido.sol` implementation (no initialization required)
2. Deploy `NodeOperatorsRegistry.sol` implementation (no initialization required)
3. Deploy `DepositSecurityModule.sol`
4. Initialize `DepositSecurityModule` (add  guardians and set the quorum)
5. Transfer ownership to the DAO Agent
6. Create voting with several items:
    * Publishing new implementation of `Lido` to the APM repo;
    * Updating implementation of `Lido` app with a new one;
    * Publishing new implementation of `NodeOperatorsRegistry` to the APM repo;
    * Updating implementation of `NodeOperatorsRegistry` app with a new one
    * Granting new permission `DEPOSIT_ROLE` for `DepositSecurityModule`;
    * Increase limits for Node Operators to the currently available key numbers.

## Links 

- Pull request with all smart contracts changes: https://github.com/lidofinance/lido-dao/pull/357
- DepositSecurityModule.sol: https://github.com/lidofinance/lido-dao/blob/feature/deposit-frontrun-protection-upgrade/contracts/0.8.9/DepositSecurityModule.sol
- Mitigations research forum post: https://research.lido.fi/t/mitigations-for-deposit-front-running-vulnerability/1239
- Blog post with events log: https://blog.lido.fi/vulnerability-response-update/
