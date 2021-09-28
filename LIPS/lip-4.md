---
lip: 4
title: 72h Aragon Votes
status: WIP
author: Victor Suzdalev (kadmil)
discussions-to: https://research.lido.fi/t/increase-the-dao-voting-duration/1048
created: 2021-09-28
updated: -
---

# 72h Aragon Votes

Currently Aragon votes on Lido DAO are 24h long. This timeframe has proved to be short enough for reaching the quorum even on non-contentios votes. While most routine DAO operations are expected to switch to EasyTracks (vehicle for default-pass DAO actions), any custom blockchain action on behalf of DAO would still require the Aragon vote. The proposal is to make Aragon votes longer, allowing more LDO holders to participate.

## Problem

Gathering the quorum on Aragon votes within 24h window proved difficult enough even for non-contentious executive votes such as raising Node Operators' keys limits and refilling liquidity pools' rewards programs.

## Proposed solution

The proposed solution is to make Aragon votes last 72h instead of the current 24h. To achieve this, the DAO would have to upgrade the Voting smart contract, as the current version doesn't allow changing the vote duration.

The Voting contract version where the vote duration can be changed can be found [there](https://github.com/lidofinance/aragon-apps/blob/8c46da8704d0011c42ece2896dbf4aeee069b84a/apps/voting/contracts/Voting.sol)

We've strived to make as little changes as possible so not to introduce any issues to the Voting contract the DAO currently uses.

## Change implementation

The upgrade plan is:
1. Make changes to the Aragon Voting smart contract so the voting duration can be changed. This work is done and can be checked out in [our Aragon fork repo](https://github.com/lidofinance/aragon-apps/blob/8c46da8704d0011c42ece2896dbf4aeee069b84a/apps/voting/contracts/Voting.sol).
1. Deploy the Voting implementation from [our Aragon fork repo](https://github.com/lidofinance/aragon-apps/blob/8c46da8704d0011c42ece2896dbf4aeee069b84a/apps/voting/contracts/Voting.sol).
2. Run the Aragon vote with actions:
   1. Push new app version to the Aragon repo
   2. Upgrade the voting contract to the deployed version from the step 1.
   3. Grant the DAO Agent contract the `UNSAFELY_MODIFY_VOTE_TIME_ROLE` role.
   4. Call `unsafelyChangeVoteTime` method changing vote duration to 72h instead of the current 24h.
3. Once the vote is concluded, enact it and get the 72h votes going forward.

## Change caveats

Any votes started within 48h before the voting upgrade vote start and not enacted before it's enactment would get their voting time frame updated as well. To prevent this side-effect, the voting upgrade must be started once all the running votes are concluded & enacted.
For instance, on a week with single Omnibus vote starting on Thursday, 12PM UTC, the Omnibus should be enacted before starting the vote for Voting upgrade, and the Voting upgrade should start not earlier than 12PM UTC Saturday.

It seems like the best time to start the Voting upgrade vote would be Monday, 11AM UTC. This would allow preserving Omnibus enactment schedule and bring as little distraction to DAO operations as possible.

The increased vote time will reflect in all the activites that currently have a vote as a part of process, e.g. incentivization, changing limits etc.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
