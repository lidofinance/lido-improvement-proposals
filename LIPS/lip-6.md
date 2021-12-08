---
lip: 6
title: In-protocol coverage
status: WIP
author: Sam Kozin, Eugene Mamin
discussions-to: TBA
created: 2021-12-03
updated: 2021-12-09
---

# In-protocol coverage proposal

## Summary

Specifies an in-protocol mechanism of applying coverage to address slashing/penalties events.

## Motivation

Currently, Lido has no adopted and well-defined mechanism of applying coverage for stakeholders' losses due to validators pentalties, slashing and other conditions. We propose an in-protocol solution to precisely specify coverage application without breaking existing principles, agreements, and integrations with stETH token. The proposal is aimed to improve the overall technical transparency and completeness of the Lido protocol regarding applying coverage.

## Mechanics

We propose to provide an on-chain in-protocol mechanism of applying coverage by burning stETH token. It relies on the rebasing nature of the stETH. The basic account [balance calculation](https://docs.lido.fi/contracts/lido#rebasing) defined by stETH contract is the following:
```solidity
balanceOf(account) = shares[account] * totalPooledEther / totalShares
```
So burning someone's shares (e.g. decreasing `totalShares` count) leads to increasing all of the other accounts' balances. 

We propose to deploy a dedicated `SelfOwnedStEthBurner` contract which accepts burning requests by locking caller-provided stETH tokens. Those burning requests are initially set by the contract to a pending state. Actual burning happens within an oracle (`LidoOracle`) beacon report to prevent additional fluctuations of the existing stETH token rebase period.

We also distinguish two types of shares burn requests:
- request to cover slashing event (e.g. decreasing of the total pooled ETH amount between the two consecutive oracle reports)
- request to burn shares for any other cases (non-cover)

The proposed contract has two separate counters for the burnt shares: cover and non-cover ones. The contract should have exclusive access to the stETH shares burning. Also, it's highly desirable to only allow burning stETH from the contract's own balance.

Finally, `SelfOwnedStEthBurner` has logs and public getters to provide external access for a proper accounting and monitoring. 

### Sending a burn request

The user sends a burn request transaction to the `SelfOwnedStEthBurner` contract by providing some stETH tokens and indicating the burning request type: cover or non-cover. The contract locks the provided stETH amount on its own balance and marks this amount for future burning.

The contract also contains a pair of internal counters to collect the total amount to be burnt upon the next oracle report.

### Burning execution on oracle report

The contract enacts pending burning requests as part of the next oracle report by burning all associated stETH tokens from its own balance. After that, the contract increments burnt shares counters (cover and non-cover burnt shares are counted separately) and emits events.

The contract tracks `LidoOracle` reports by implementing the `IBeaconReportReceiver` interface for a callback invocation (thanks to the implemented [LIP-2 Oracle upgrade](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-2.md)).

The main reason for performing token burn exactly as part of a Lido oracle update is to preserve a predictable and well-known rebase period since **_every_** burning event also leads to token rebase. External integrations and oracles may rely on the existing rebase period (which is about one-day).

#### Gas consumption considerations

Each quorum-reaching beacon report currently costs around 461000 gas.

Adding the mechanism described in this proposal would increase the consumption as follows (with a 20% accuracy margin): by 720 gas for the case when there are no pending burn requests (base recurrent case); by 110000 gas when any number of burn requests of a single type is to be enacted (either cover or non-cover); by 160000 gas in the case when both cover a non-cover requests are to be enacted.

This represents **less than a 0.2% increase for the base case** when no burn requests are pending, a 24% increase when applying one type of burn requests, and 35% increase when applying both types of burn requests.

### Shares burnt counter

We propose to store the total amount of shares ever burnt, distinguishing cover and non-cover shares, by maintaining two separate counters inside the `SelfOwnedStEthBurner` contract: `totalCoverSharesBurnt` and `totalNonCoverSharesBurnt`. The counters are increased when actual stETH burn is performed as part of the Lido Oracle report.

This allows to split any stETH rebase into two sub-components: the rewards-induced rebase and cover application-induced rebase, which can be done as follows:

1. Before the rebase, store the previous values of both counters, as well as the value of stETH share price:
   ```solidity
   prevCoverSharesBurnt = SelfOwnedStEthBurner.totalCoverSharesBurnt()
   prevSharePrice = stETH.totalSupply() / stETH.getTotalShares()
   ```
2. After the rebase, perform the following calculations:
   ```solidity
   sharesBurntFromOldToNew = SelfOwnedStEthBurner.totalCoverSharesBurnt() - prevCoverSharesBurnt;
   newSharePriceAfterCov = stETH.totalSupply() / (stETH.getTotalShares() + sharesBurntFromOldToNew);
   newSharePrice = stETH.totalSupply() / stETH.getTotalShares();
   
   // rewards-induced share price increase
   rewardPerShare = newSharePriceAfterCov - prevSharePrice;
   
   // cover-induced share price increase
   nonRewardSharePriceIncrease = newSharePrice - prevSharePrice - rewardPerShare;
   ```

The proposed mechanism allows integrations like [Anchor/bETH](https://docs.anchorprotocol.com/protocol/bonded-assets-bassets/bonded-eth-beth) to reimburse the holders of the derivative token for the losses from prior slashing events by calculating `rewardPerShare`. To be exact, the bETH integration is designed as follows:

1. When a user submits stETH, the equivalent amount of bETH is minted, and the provided stETH is locked on the bETH contract address.
2. When a positive stETH rebase happens, the resulting increase of the amount of stETH held on bETH contract address is forwarded to the Anchor protocol.
3. When a negative rebase happens, the bETH/stETH exchange rate (caluclated as `stETH.balanceOf(bETH) / bETH.totalSupply()`) decreases, becoming less than 1. This means that a bETH holder would get less than one stETH for one bETH burnt:
    > Losses from slashing events are equally shared amongst all bETH tokens, lowering the calculated value of a bETH token. stETH accounts for slashing by pro-rata decreasing the token balance of all stETH holders. The stETH balance held by the bETH smart contract also decreases, decreasing the bETH exchange rate. 

This means that, since applying cover leads to a positive stETH rebase, it cannot be used to reimburse bETH holders from prior negative stETH rebases. In contrast, implementing the calculations detailed above as part of the positive rebase handling, and forwarding only the rewards-generated part of the balance increase to Anchor, allows recovering the bETH/stETH rate to 1.

The Anchor integration already has the calculations proposed above implemented using a plug-in adapter for retrieving the total number of shares burnt for cover purposes. The current version of the adapter just returns zero so it will need to be replaced if/after the proposed changes are made to the protocol. See [`AnchorVault.collect_rewards#434`](https://github.com/lidofinance/anchor-collateral-steth/blob/39cd11d9d628c1de6f4f481f3992f2d1fca41ecf/contracts/AnchorVault.vy#L434) for the possible sketch of usage with similar integrations.

### Burn permissions

The `Lido` contract (and to be precise, the whole protocol) has only one function which allows stETH burning, [`burnShares`](https://docs.lido.fi/guides/protocol-levers#lido). It requires the caller to have the `BURN_ROLE` permission.

Currently, `BURN_ROLE` is assigned to the `Voting` contract. This proposal requires that only `SelfOwnedStEthBurner` contract is to be allowed stETH burning. It's vital for implementing the aforementioned calculations of splitting a rebase to cover- and rewards- induced parts.

Also, we propose to enforce fine-grained permission control by the Aragon [ACL parameters interpretation](https://hack.aragon.org/docs/aragonos-ref#parameter-interpretation) to only allow `SelfOwnedStEthBurner` to burn stETH from its own balance.
 
So, more formally, we propose the following permissions changes:
- Revoke `BURN_ROLE` from `Voting`.
- Assign `BURN_ROLE` to `SelfOwnedStEthBurner` requiring the `_account` argument of `burnShares` to equal the `SelfOwnedStEthBurner` address.

## Discussion

Let's discuss the proposed approach to cover application, its upsides and downsides, and possible alternative approaches.

### The proposed approach: burning stETH

Pros:
- Fairly straightforward in terms of contract logic
- No off-chain intervention needed
- Doesn't touch the deployed Lido contracts code
- Provides a generalized way of performing positive balance rebases, both for coverage and non-coverage purposes
- The cover must be converted to stETH before it can be applied, which can either increase the market demand for stETH (in the case stETH is bought from the market) or bring additional stake to the protocol (in the case stETH is obtained by staking ETH)

Cons: 
- Smart-contract development is required
- The cover must be first converted to stETH before it can be applied, which might complicate the logic of obtaining the cover

### Alternative approach 1: pushing Ether without minting stETH

Implement the `Lido.submitWithoutMinting()` function that adds the received Ether to the pool but doesn't mint any stETH in return. This will increase the total protocol-controlled Ether amount without increasing the total stETH shares amount, resulting in a positive stETH rebase.

This function could be then used within an oracle report callback to submit cover in form of ETH instead of burning stETH-expressed cover. This would be equivalent to burning some amount of stETH shares which can be trivially calculated and used to increment the burnt shares counters.

Pros:
- Uses less gas than burning stETH
- Doesn't require the cover to be converted to stETH which might simplify the cover application
- Brings additional stake to the protocol
- No burn permissions are needed for the cover application smart contract. Burn permissions can be revoked from all entities

Cons:
- Requires upgrading the protocol smart contracts
- Cover cannot be applied without increasing the validator activation queue which can be undesired if the queue is already crowded (as it will lead to further staking APR dilution)
- Requires additional manual intervention to prevent the new stake resulting from the cover application from being potentially forwarded to the node operator that has undergone slashing

### Alternative approach 2: explicit payouts and airdrops with other tokens

A trivial approach would be to distribute the cover (in form of ETH, stETH, LDO, or some other token) to all stakers via an airdrop.

Pros:
- The cover can be distributed in any Ethereum-based asset.

Cons:
- Requires non-trivial and error-prone off-chain computations to calculate the amount of cover for each direct and non-direct staker address. These computations will get more and more complex as new stETH integrations appear.
- Gas-costly: initializing the airdrop contract with the snapshot of all stakers' balances will consume a lot of gas. Also, getting the cover will cost gas for each individual staker and might make no economical sense for smaller stakers.
- Requires an explicit action from each staker which means that stakers will have to monitor the announcements to get the cover.
- May add volatility to the cover asset on mass payouts.

## Specification

We propose the following contract interface.
The code below presumes the Solidity v0.8 syntax.

### Function: getCoverSharesBurnt
```solidity
function getCoverSharesBurnt() returns (uint256) external view
```
Returns the total cover shares ever burnt.

### Function: getNonCoverSharesBurnt
```solidity
function getNonCoverSharesBurnt() returns (uint256) external view
```
Returns the total non-cover shares ever burnt.

### Function: requestStEthBurn
```solidity
function requestStEthBurn(uint256 stEth2Burn, bool isCover) external
```
Transfers `stEth2Burn` stETH tokens from the message sender and irreversibly locks these on the burner contract address. Internally converts `stETH2Burn` amount into underlying shares amount (`stEth2BurnAsShares`) and marks the converted amount for burning by increasing `coverSharesBurnRequested` and `nonCoverSharesBurnRequested` counters.

* Must transfer `stEth2Burn` stETH tokens from the message sender to the burner contract address.
* Reverts if no stETH provided (`stEth2Burn == 0`).
* Reverts if no stETH transferred (allowance exceeded).
* Emits the `StEthBurnRequested(isCover, msg.sender, stEth2Burn, stEth2BurnAsShares)` event.

### Function: processLidoOracleReport
```solidity
function: processLidoOracleReport(uint256 _postTotalPooledEther, 
                                  uint256 _preTotalPooledEther,
                                  uint256 _timeElapsed) external
```
Enacts cover/non-cover burning requests and logs cover/non-cover shares amount just burnt. Increments `totalCoverSharesBurnt` and `totalNonCoverSharesBurnt` counters. Resets `coverSharesBurnRequested` and `nonCoverSharesBurnRequested` counters to zero.

Does nothing if there are no pending burning requests.

See: [`IBeaconReportReceiver.processLidoOracleReport`](https://docs.lido.fi/contracts/lido-oracle#receiver-function-to-be-invoked-on-report-pushes).

* Must be called as part of an oracle quorum report.
* Reverts if the message sender is not `LidoOracle`.
* Emits the `StEthBurnt(true, coverSharesBurnRequestedAsStEth, coverSharesBurnRequested)` event for an executed cover stETH burning.
* Emits the `StEthBurnt(false, nonCoverSharesBurnRequestedAsStEth, nonCoverSharesBurnRequested)` event for an executed non-cover stEth burning.

### function getExcessStEth
```solidity
function getExcessStEth() return (uint256) external view  
```
Returns the stETH amount belonging to the burner contract address but not marked for burning.

### function recoverExcessStETH
```solidity
function recoverExcessStETH() external 
```
Transfers the excess stETH amount (e.g. belonging to the burner contract address but not marked for burning) to the Lido treasury address set upon the contract construction.

See: `getExcessStEth`

* Must do nothing if `getExcessStEth` returns 0 (zero), e.g. there is no excess stETH on the contract's balance.
* Emits the `ExcessStETHRecovered` event if `getExcessStEth` is non-zero.

### Function: recoverERC20
```solidity
function recoverERC20(address token, uint256 amount) external
```
Transfers a given `amount` of an ERC20-token (defined by the `token` contract address) belonging to the burner contract address to the Lido treasury address.
* Reverts if the `amount` is 0 (zero).
* Reverts if `token` address is 0 (zero).
* Reverts if trying to recover stETH (`token` address equals to the `stETH` address).
* Emits the `ERC20Recovered` event.

### Function: recoverERC721
```solidity
function recoverERC721(address token, uint256 token_id) external
```
Transfers a given `token_id` of an ERC721-compatible NFT (defined by the `token` contract address) belonging to the burner contract address to the Lido treasury address.
* Reverts if `token` address is 0 (zero).
* Emits the `ERC721Recovered` event. 

### Event: StEthBurnRequested
```solidity
    event StEthBurnRequested(
        bool indexed isCover,
        address indexed requestedBy,
        uint256 amount,
        uint256 sharesAmount
    );
```
Emitted when a new stETH burning request is added by the `requestedBy` address.

See: `requestStEthBurn`.

### Event: StEthBurnt
```solidity
    event StEthBurnt(
        bool indexed isCover,
        uint256 amount,
        uint256 sharesAmount
    );
```
Emitted when the stEth `amount` (corresponding to `sharesAmount` shares) burnt for the `isCover` reason.

See: `processLidoOracleReport`.

### Event: ExcessStETHRecovered
```solidity
    event ExcessStETHRecovered(
        address indexed requestedBy,
        uint256 amount,
        uint256 sharesAmount
    );
```
Emitted when the excessive stEth `amount` (corresponding to `sharesAmount`) recovered (e.g. transferred) to the Lido treasure address by `requestedBy` sender.

See: `recoverExcessStETH`.

### Event: ERC20Recovered
```solidity
    event ERC20Recovered(
        address indexed requestedBy,
        address indexed token,
        uint256 amount
    );
```
Emitted when the ERC20 `token` recovered (e.g. transferred) to the Lido treasure address by `requestedBy` sender.

See: `recoverERC20`.

### Event: ERC721Recovered
```solidity
    event ERC721Recovered(
        address indexed requestedBy,
        address indexed token,
        uint256 token_id
    )
```
Emitted when the ERC721-compatible `token` (NFT) recovered (e.g. transferred) to the Lido treasure address by `requestedBy` sender.

See: `recoverERC721`.

## Security Considerations

#### The proposed contract is non-upgradable and non-ownable

There are no intermediate proxies. All of the external addresses are set only on the deployment stage. In case of emergency or crucial update, we would deploy a new contract and reset `IBeaconReportReceiver` callback address on `LidoOracle`. A newly deployed replacement contract should be initialized with previously accumulated cover/non-cover counter values.

#### Burning is only allowed on the `SelfOwnedStEthBurner` contract address

The `BURN_ROLE` that's assigned to the `SelfOwnedStEthBurner` is restricted by the Aragon ACL parameters. That way, `Lido.burnShares()` method could only be executed for the `SelfOwnedStEthBurner` contract address by the contract itself and there are no other shares burning interfaces exists.

#### There is no way to call `processOracleReport` from any address except the `LidoOracle` contract

An explicit pre-condition (`require(msg.sender == Lido.getOracle()`) is checked for the message sender address.

## Failure modes

#### DAO decides to grant a shares burning permission to someone else

It can break the proposal concept of maintaining global (protocol-wide) shares burnt counters. 
So the `SelfOwnedStEthBurner` contract will become almost useless by providing incomplete and even incorrect information about ever burnt shares.

#### Someone apply slashing coverage using non-cover request type

The Anchor Protocol will receive incorrect rewards.

#### Someone apply non-cover using cover request type

Anchor/bETH token holders will lose some rewards. 

## Reference implementation

Reference implementation sketch of the `SelfOwnedStEthBurner` interface is given below
```solidity=
interface IERC20;
interface IStEth;
interface ILido;

interface IBeaconReportReceiver /* already exists in the Lido sources */ {
    function processLidoOracleReport(uint256 _postTotalPooledEther,
                                     uint256 _preTotalPooledEther,
                                     uint256 _timeElapsed) external;
}

contract SelfOwnedStEthBurner is IBeaconReportReceiver {
    uint256 private coverSharesBurnRequested;
    uint256 private nonCoverSharesBurnRequested;
    
    uint256 private totalCoverSharesBurnt;
    uint256 private totalNonCoverSharesBurnt;
    
    address public immutable LIDO;
    address public immutable TREASURY;

    event StEthBurnRequested(
        bool indexed isCover,
        address indexed requestedBy,
        uint256 amount,
        uint256 sharesAmount
    );

    event StEthBurnt(
        bool indexed isCover,
        uint256 amount,
        uint256 sharesAmount
    );

    event ExcessStETHRecovered(
        address indexed requestedBy,
        uint256 amount,
        uint256 sharesAmount
    );

    event ERC20Recovered(
        address indexed requestedBy,
        address indexed token,
        uint256 amount
    );
        
    function getCoverSharesBurnt() returns (uint256) external view {
        return totalCoverSharesBurnt;
    }
    
    function getNonCoverSharesBurnt() returns (uint256) external view {
        return totalNonCoverSharesBurnt;
    }
    
    function getExcessStEth() return (uint256) external view {
        uint256 sharesBurnRequested = (coverSharesBurnRequested + nonCoverSharesBurnRequested);
        uint256 totalShares = IStEth(LIDO).sharesOf(address(this));

        require (totalShares >= sharesBurnRequested);
        
        return IStEth(LIDO).getPooledEthByShares(totalShares - sharesBurnRequested);
    }    
    
    constructor(address _treasury, address _lido) 
    {
        require(_treasury != address(0), "TREASURY_ZERO_ADDRESS");
        require(_lido != address(0), "LIDO_ZERO_ADDRESS");
        
        TREASURY = _treasury;
        LIDO = _lido;
    }
    
    function requestStEthBurn(uint256 stEth2Burn, bool isCover) external {
        require(stEth2Burn > 0);
        require(IStETH(LIDO).transferFrom(msg.sender, address(this), stEth2Burn));
        
        uint256 sharesAmount = IStETH(LIDO).getSharesByPooledEth(stEth2Burn);
        
        emit StEthBurnRequested(isCover, msg.sender, stEth2Burn, sharesAmount)

        if (isCover) { 
            coverSharesBurnRequested += sharesAmount;
        } else {
            nonCoverSharesBurnRequested += sharesAmount;
        }
    }
    
    function recoverExcessStETH() external {
        uint256 excessStEth = getExcessStEth();
        
        if (excessStEth > 0) {
            uint256 excessSharesAmount = IStETH(LIDO).getSharesByPooledEth(excessStEth);
            
            emit ExcessStETHRecovered(msg.sender, excessStEth, excessSharesAmount);

            IStEth(LIDO).transfer(TREASURY, excessStEth);
        }
    }
    
    //don't accept ether
    function () {
        throw;
    }
   
    function recoverERC20(address token, uint256 amount) {
        require(amount > 0);
        require(token != address(0));
        require(token != LIDO);

        emit ERC20Recovered(msg.sender, token, amount);
        
        IERC20(token).transfer(TREASURY, amount);
    }
    
    function processLidoOracleReport(uint256 _postTotalPooledEther,
                                     uint256 _preTotalPooledEther,
                                     uint256 _timeElapsed) external {
        
        require(msg.sender == ILido(LIDO).getOracle(), "APP_AUTH_FAILED");

        uint256 memCoverSharesBurnRequested = coverSharesBurnRequested;
        uint256 memNonCoverSharesBurnRequested = nonCoverSharesBurnRequested;

        uint256 burnAmount = memCoverSharesBurnRequested + memNonCoverSharesBurnRequested;

        if (burnAmount > 0) {
            if (memCoverSharesBurnRequested > 0) {            
                totalCoverSharesBurnt += memCoverSharesBurnRequested;   

                uint256 coverStEthBurnAmountRequested = IStEth(LIDO).getPooledEthByShares(memCoverSharesBurnRequested);
                emit StEthBurnt(true /* isCover */, coverStEthBurnAmountRequested, memCoverSharesBurnRequested);

                coverSharesBurnRequested = 0;        
            }

            if (memNonCoverSharesBurnRequested > 0) {            
                totalNonCoverSharesBurnt += memNonCoverSharesBurnRequested;

                uint256 nonCoverStEthBurnAmountRequested = IStEth(LIDO).getPooledEthByShares(memNonCoverSharesBurnRequested);
                emit StEthBurnt(false /* isCover */, nonCoverStEthBurnAmountRequested, memNonCoverSharesBurnRequested);

                nonCoverSharesBurnRequested = 0;
            }        
            
            ILido(LIDO).burnShares (address(this), burnAmount);
        }
    }
}
```
## Links 

- Lido DAO discussion on the 3rd parties coverage: https://research.lido.fi/t/should-lido-use-third-party-insurance-providers/757
- Research on the self-coverage options: https://blog.lido.fi/offline-slashing-risks-are-self-cover-options-enough/
- Sketch for the coverage support on Anchor Protocol integration: https://github.com/lidofinance/anchor-collateral-steth/blob/39cd11d9d628c1de6f4f481f3992f2d1fca41ecf/contracts/AnchorVault.vy#L422
- Preliminary smoke tests on Anchor Protocol integration:
https://github.com/lidofinance/anchor-collateral-steth/blob/00cddd633ffe855a133a8c3ee024833bf6551d50/tests/test_insurance.py
- Anchor Protocol Bonded ETH (bETH) token spec: https://docs.anchorprotocol.com/protocol/bonded-assets-bassets/bonded-eth-beth
- Aragon ACL parameter intepretation: https://hack.aragon.org/docs/aragonos-ref#parameter-interpretation