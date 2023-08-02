---
qip: 18
title: "Re-Distribution of ERC-20 Unmigrated Quanta to Active Nodes"
author: DonFixi (@DonFixi)
layer: meta
status: "draft/"Nuke'em"
comments_uri: ~
created: 2022-03-14
---

## Abstract

To incentivise node running decentralisation while reducing an external enginnered side-attack Quantum Computing (QC) minor risk. 


## Motivation

There is almost half million of unmigrated Quanta as per address Q010500e587fd07bb1c5ace98956d9aa6c347e7114ccb4ec7183baa54804cc8b974e91cc3b5617e

These native QRL tokens are exchangable for QRL ERC-20 tokens as per initial 2017 ETH from the ERC contract 0x697beac28B09E122C4332D163985e8a73121b97F at 1:1 ratio. 

There was an atutomatic migration QRL ERC-20 -> Quanta window that the team deployed back in 2017. That window was closed 4 years ago. 

To migrate them today, there is a non-public disclosed manual migration proccess which involves the QRL ERC-20 owner to stablish communication with the QRL foundation.

As QC grows, ETH and ERC remains vulnerable to factorization algorithms such as Shor's, there is a minor chance that a malicious actor with access to powerfull QC would try to breach one, if not several, QRL ERC-20 in near future.

Hereby, since its has been 4 years already, I do propose here a fair QRL re-distribution of these non-claimed Quanta funds to all active node peers in the QRL chain. 


## Specification

Mathematical Distribution Model TBD - could be X per block/luck, a simple decay curve, per uptime running, or a combiantion. 

## Rationale

The idea is to one, mitigate once for all external QC risk, and two, promote node running. 

Specially for those running low power node devices such a Raspberry Pi, until Proof-of-Stake arrives and beyond 🚀 

## Backward compatibility

Those that don't migrate their QRL ERC-20 tokens from ETH chain on a set time, well though luck. 

A liability study shall be conducted before this QIP passes or pushes the final "burning" code. 

The "Robin Hood Effect" may not land equally for those who still have QRL ERC-20 tokens under control.  

## Implementation

A hard-fork will likely be required in order to set the new distributive model for the unmigrated address.

## Security Considerations

This QIP suggest the mitigation of an external QC side-attack while promoting and incetivising a healthy quanta redistribution for the active participants.

Nuke from orbit--it's the only way.