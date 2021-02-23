---
lip: 2
title: Oracle contract upgrade to v2
status: Implemented
author: Denis Glotov
discussions-to: none
created: 2021-02-22
updated: 2021-12-22
---

# Oracle contract upgrade to v2

The following new features are added to oracle contract.


## Callback function to be invoked every time the quorum is reached among oracle daemons data.


## Use only one epoch per frame to simplify vote accounting.


## Bound oracles' possible reports from both sides by a configurable relative change value.


## Add calculation of rewards APR.


## Allow oracles members under quorum count.

The most likely reason for removing an oracle member is a malicious oracle. It's better to have an
oracle with no quorum (members can be added or quorum lowered) than an oracle with a quorum but a
malicious member in it.


## Add events to cover all states change and getters for accessing the current state details.
