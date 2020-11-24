---
lip: 3
title: Oracle interface and reward algorithm specification
status: WIP
author: Kirill Varlamov
discussions-to: TBD
created: 2020-11-10
updated: 2020-11-10
---

# Oracle interface and reward algorithm specification for v0.2.x

## Problems with v0.1.x

* The funds "in process" (from deposit submission until it gets processed and added to the beacon state's validators list) got treated like a decrease, the behavior was equal to slashing and lead to significant drops in stETH supply. See [lido-dao#110](https://github.com/lidofinance/lido-dao/issues/110)

## How they can be solved

* In-flight funds (submitted but not included in the beacon state) should be taken into account by the Lido contract.

## Proposal

### Bisiness logic of rewards calculation

```python
class LidoContract:
    """Lido.sol"""
    
    def __repr__(self):
        return(f"<Lido BcnValidators:{self.lastBeaconValidators} " +
               f"totalValidators:{self.totalValidators} " +
               f"bufferedBalance:{self.bufferedBalance} " +
               f"getInProcessBalance:{self.getInProcessBalance()} " +
               f"getTotalPooledEth:{self.getTotalPooledEth()}>")
    
    lastBeaconValidators = 0 # online validators reported by oracle (=all validators with retrievable balances)
    lastBeaconBalance = 0
    
    '''Total amount of validators incremented on each deposit.
    Note: was totalDepositCalls before v0.2.x
    Note: this value never decreases
    '''
    totalValidators = 0
    
    """The amount of Ethers received but not yet staked to DepositContract
    equals to web3.eth.getBalance(LidoContract), nominated in wei"""
    bufferedBalance = 0.0
            
    def reportBeacon(self, beaconValidators, beaconBalance): #onlyOracleContract
        """Called by the Oracle Contract (that in turn is called by the Oracle Daemons)

        arguments:
        beaconValidators -- the number of validators seen on beacon in the beaconChain validators index.
                            The validators can have the following states (for lighthouse API):
                            Active,
                            ExitedSlashed,
                            Withdrawable,
                            StandbyForActive,
                            WaitingInQueue,
                            WaitingForFinality,
                            etc..
                            https://github.com/sigp/lighthouse/blob/f8da151b0bec12a707a742260dfe635a6a1890f4/common/eth2/src/types.rs#L247
                      at the state (DEPOSITED or ACTIVE).Validators in activation queue are not accounted
        beaconBalance -- the total sum of all validators' balances on the beacon in wei (1e-18 ETH)
        
        """
        reportedBeaconBalance = beaconBalance # in wei
        reportedBeaconValidators = beaconValidators
        
        #just appeared validators. We assume the base balance is 32 for them
        appearedBeaconValidators = reportedBeaconValidators - self.lastBeaconValidators
        
        rewardBase = (appearedBeaconValidators * 32) + self.lastBeaconBalance
        print("calculated rewardBase", rewardBase)
        
        # If there was profit on the interval
        if reportedBeaconBalance > rewardBase :
            print("There was profit since last report, pay reward")
            reward = reportedBeaconBalance - rewardBase
            print("Pay reward", reward)
        else:
            print("No profit since last reward. Nothing to pay")
        
        self.lastBeaconBalance = reportedBeaconBalance
        self.lastBeaconValidators = reportedBeaconValidators
        
    def getInProcessBalance(self): #public
        """Returns the total base balance (multiple of 32) of validators in transient state (from deposit submission until
        it gets processed and added to the beacon state's validators list).
        The value nominated in wei (1e-18 Ether)
        """
        return (self.totalValidators - self.lastBeaconValidators) * 32.0
        
    def getTotalPooledEth(self): #public    was totalControlledEther in v0.1.x
        """Returns the sum of all the balances in the system.
        Nominated in wei (1e-18 Ether)
        """
        return self.bufferedBalance + self.getInProcessBalance() + self.lastBeaconBalance
    
    def submit(self, eth_amount):
        self.bufferedBalance += eth_amount
        # this is done separately. Combned for simplicity
        deposits_iter = int(self.bufferedBalance / 32)
        for iter in range(deposits_iter):
            self.bufferedBalance -= 32
            self.totalValidators += 1

class OracleContract:
    """LidoOracle.sol"""
    
    def __init__(self, lido):
        self.lido = lido
    
    def reportBeacon(self, epoch, beaconValidators, beaconBalance):
        """Receives reports from oracles. New interface for v0.2
        before v0.2.x was: pushData(uint256 _reportInterval, uint256 _eth2balance)
          
        arguments:
        epoch -- the reportable epoch
        beaconValidators -- the number of validators seen on beacon. This includes all the visible validators
                      (all the states )
        beaconBalance -- the total sum of all validators' balances on the beacon in wei (1e-18)
        
        The Oracle does not do any checks or validation, just checks if the mode achieved on reported value.
        self._tryFinalize()
        
        If yes, it pushes the value to the Lido
        """
        self.lido_contract.reportBeacon(beaconValidators, beaconBalance)
```

#### Behavior illustration

```
In:
oracle_contract = OracleContract(lido_contract)
lido_contract

Out:
<Lido BcnValidators:0 totalValidators:0 bufferedBalance:0.0 getInProcessBalance:0.0 getTotalPooledEth:0.0>

In:
"""User submits 100 ETH, and the funds get deposited to depositContract"""
lido_contract.submit(100)
lido_contract

Out:
<Lido BcnValidators:0 totalValidators:3 bufferedBalance:4.0 getInProcessBalance:96.0 getTotalPooledEth:100.0>

In:
"""the first validator appeared online. 
This is not considered as a reward, no payouts"""
lido_contract.reportBeacon(1, 32)
lido_contract

Out:
calculated rewardBase 32
No profit since last reward. Nothing to pay
<Lido BcnValidators:1 totalValidators:3 bufferedBalance:4.0 getInProcessBalance:64.0 getTotalPooledEth:100.0>

In:
"""Got the first profit. Reward detected and paid"""
lido_contract.reportBeacon(1, 33)
lido_contract

Out:
calculated rewardBase 32
There was profit since last report, pay reward
Pay reward 1
<Lido BcnValidators:1 totalValidators:3 bufferedBalance:4.0 getInProcessBalance:64.0 getTotalPooledEth:101.0>

In:
"""Slashed down"""
lido_contract.reportBeacon(1, 32.5)
lido_contract

Out:
calculated rewardBase 33
No profit since last reward. Nothing to pay
<Lido BcnValidators:1 totalValidators:3 bufferedBalance:4.0 getInProcessBalance:64.0 getTotalPooledEth:100.5>

In:
"""Partial recovery from the loss considered as a reward."""
lido_contract.reportBeacon(1, 33)
lido_contract

Out:
calculated rewardBase 32.5
There was profit since last report, pay reward
Pay reward 0.5
<Lido BcnValidators:1 totalValidators:3 bufferedBalance:4.0 getInProcessBalance:64.0 getTotalPooledEth:101.0>

In:
"""the second validator appeared online. 
This is not considered as a reward, no payouts"""
lido_contract.reportBeacon(2, 65)
lido_contract

Out:
calculated rewardBase 65
No profit since last reward. Nothing to pay
<Lido BcnValidators:2 totalValidators:3 bufferedBalance:4.0 getInProcessBalance:32.0 getTotalPooledEth:101.0>

In:
"""Third validator just appeared, but some others were slashed - No reward"""
lido_contract.reportBeacon(3, 96)
lido_contract

Out:
calculated rewardBase 97
No profit since last reward. Nothing to pay
<Lido BcnValidators:3 totalValidators:3 bufferedBalance:4.0 getInProcessBalance:0.0 getTotalPooledEth:100.0>

In:
"""Loss was partially compensated. Pay reward"""
lido_contract.reportBeacon(3, 97)
lido_contract

Out:
calculated rewardBase 96
There was profit since last report, pay reward
Pay reward 1
<Lido BcnValidators:3 totalValidators:3 bufferedBalance:4.0 getInProcessBalance:0.0 getTotalPooledEth:101.0>
```

## Implementation notices

### Oracle

The existing business logic of [Lido Oracle code](https://github.com/lidofinance/lido-dao/blob/master/contracts/0.4.24/oracle/LidoOracle.sol) basically remains the same as v0.1.x. The key changes are:

* Remove report interval, corresponding getters and setters - now we bind to epochs
* New interface and Report layout
* Implement current reportable epoch checks and if the epoch has been reported - postponed to v0.2.1 (after initial launch on testnet)
* Other synchronisation rules - postponed to v0.2.1 (after initial launch on testnet)
* `mode` function (https://github.com/lidofinance/lido-dao/blob/d8139560efb892bbb479bdc8ffc662929a78ea2c/contracts/0.4.24/oracle/Algorithm.sol#L13) should be adapted/tested with the `struct Report` as a payload.


```
pragma solidity 0.4.24;

contract LidoOracle {
    
    // the variables are tightly packed to 256bit word
    struct Report {
        uint64 epoch;
        uint64 beaconValidators;
        uint128 beaconBalance;
    }
    
    Report public report;
    
    // use conventional ABI encoder, no "experimental features" here
    function reportBeacon(uint64 _epoch, uint64 _beaconValidators, uint128 _beaconBalance) public {
        report = Report({epoch: _epoch, beaconValidators: _beaconValidators, beaconBalance: _beaconBalance});
        // SSTORE occupies 1 slot
        
        // Checks go here
        // Quorum business logic goes here
        // Downstream reporting to Lido goes here
    }
}
```

### Lido

* Implement reward calculations from the cells above. (Reward base calculated in runtime, validators that are in a transient state get accounted)
* Adapt variables naming and interfaces with this spec and oracle contract (rename `pushData` to `reportBeacon`)