---
qip:
title: "QRLX: a QRC20 token on QRL 2.0 for the legacy chain's coin"
author: LeSplit (@LeSplit)
layer: core
status: "draft/incomplete"
comments_uri:
comments_summary_uri: ~
created: 2026-10-10
modified: 2026-10-10
---

## Abstract

QRL 2.0 would carry two assets instead of one. **Quanta** stays the native coin, used for staking and fees. **QRLX** is a QRC20 token on QRL 2.0 that represents the coin of the legacy proof-of-work chain.

At the migration snapshot every holder receives 1 Quanta and 1 QRLX per coin, and the old balance is retired. The legacy chain keeps mining. Coins mined after the snapshot can be burned on the legacy chain and minted 1:1 as QRLX. The bridge is one-way, holds no reserve, and QRLX has no peg.

This draft is written as an idea file: it describes the concept and its invariants, not an implementation. It is opened for discussion and is not the project's migration plan.

## Motivation

### The core idea

QRL 2.0 is described as superseding QRL 1.x. Holders are to migrate through a smart contract-based process, and current development includes a read-only QRL v1 archive showing balances at a cutoff. This draft asks whether the legacy chain has a continuing purpose instead, and whether that purpose can be held as an asset.

The purpose is cryptographic diversity. The two chains rest on different assumptions:

| Chain | Signature scheme | Family |
| --- | --- | --- |
| Legacy | XMSS | Hash-based |
| QRL 2.0, consensus and addresses | ML-DSA-87 | Lattice-based |
| QRL 2.0, timing unclear | SLH-DSA (SPHINCS+-256s) | Hash-based |

If lattice assumptions failed, coins still on the legacy chain would stay protected by a hash-based scheme. A live legacy chain is a hedge against a break in the scheme QRL 2.0 launches on. QRLX gives that hedge a price and a home in the QRL 2.0 ecosystem.

The third row decides how much this is worth. SLH-DSA was rescheduled to post-mainnet in 2025, but go-qrllib now carries a SPHINCS+-256s wallet implementation for QRL 2.0. If hash-based wallets are usable at mainnet, the gap this draft relies on is small or absent.

### What the argument is not

It is not cost of production. [QIP 16](https://github.com/theQRL/qips/blob/master/qips/QIP016.md) recorded that most legacy hashrate comes from for-profit miners who leave when price falls. Difficulty then drops and the cost of mining follows price down. Electricity sets no floor under QRLX.

## Specification

### Assets

| Asset | Lives on | Role | Created by |
| --- | --- | --- | --- |
| Quanta | QRL 2.0, native | Staking, fees | Snapshot, QRL 2.0 issuance |
| QRLX | QRL 2.0, QRC20 token | Legacy coin | Snapshot, bridge mints |

### Flow

```text
      LEGACY CHAIN  (PoW, XMSS)
 +----------------+  +----------------+
 | balance at     |  | coin mined     |
 | snapshot       |  | after snapshot |
 +----------------+  +----------------+
     |          |           |
     | credited |           | burned
     |          |           |
 ====|==========|===========|==== one-way,
     v          v           v     no return
 +--------+ +----------------------+
 | Quanta | | QRLX                 |
 +--------+ +----------------------+
       QRL 2.0  (PoS, ML-DSA)
```

### Snapshot

1. At the snapshot height, each legacy address is credited 1 Quanta and 1 QRLX per legacy coin held.
2. Legacy balances existing at the snapshot height are retired. They cannot be spent or bridged afterwards.
3. The legacy chain continues to produce blocks and coinbase rewards.

### Bridge

```text
 Holder         Legacy chain        QRL 2.0
   |                 |                 |
   |--- burn tx ---->|                 |
   |                 |--- records ---->| anchor
   |                 |                 | verify
   |<----------- mint QRLX ------------|
```

1. A holder of post-snapshot legacy coin burns it in a transaction that names a QRL 2.0 recipient.
2. QRL 2.0 anchors legacy chain records, then verifies the burn against them.
3. QRL 2.0 mints the same amount of QRLX to the recipient.

### Invariants

Any implementation must keep all five:

1. Mining never mints Quanta.
2. The bridge burns. It holds no reserve, so there is no vault.
3. The bridge is one-way: legacy to QRL 2.0 only.
4. QRL 2.0 anchors legacy chain records before trusting them.
5. There is no peg. QRLX floats.

### Left unspecified

- Burn transaction format on the legacy chain. One candidate is to carry the recipient using the standard message encoding from [QIP 2](https://github.com/theQRL/qips/blob/master/qips/QIP002.md).
- Burn verification on QRL 2.0: light client, checkpoints, or relayers. The legacy XMSS signature-verification precompile in go-qrl is a possible building block.
- Confirmations required before a mint.
- QRLX contract ownership, and who is allowed to call mint.

## Rationale

**A token, not a second native coin.** A token needs no change to QRL 2.0 issuance or consensus economics. Quanta stays the only asset securing the chain.

**Burn-and-mint, not lock-and-mint.** A locking bridge builds up a reserve that becomes the most valuable target in the system. Burning leaves nothing to steal.

**One-way.** A return path would need the legacy chain to verify QRL 2.0 state, which means more consensus work on a chain this draft otherwise leaves alone. It would also let a QRL 2.0 failure travel back to the legacy chain, which defeats the diversity argument.

**No peg.** A peg needs a reserve or an issuer willing to defend it. This draft has neither.

**QRLX at snapshot.** Every holder gets exposure to both assets without having to choose. The alternative is to mint QRLX only through the bridge.

## Backward compatibility

Retiring snapshot balances is a consensus change to the legacy chain and requires a hard fork there. Mining, coinbase rewards and XMSS addresses are otherwise unchanged.

On QRL 2.0, the snapshot allocation gains a second asset, so migration tooling must credit both Quanta and QRLX.

Exchanges and wallets go from one listed asset to two, plus a legacy coin that is still live. Tickers, deposits and custody each need a plan, and QIP 16 already named exchange availability as the uncertain part of a hard fork timeline.

This draft conflicts with the announced direction, in which QRL 2.0 supersedes QRL 1.x and QRL v1 is archived at a cutoff. It would need that plan to change, not just to be extended.

## Reference Implementation

None. To be determined if the idea survives discussion.

## Security Considerations

**The protection does not travel with the token.** QRLX lives on QRL 2.0 and is secured by QRL 2.0 signatures. Only coins still on the legacy chain, not yet bridged, carry XMSS protection. This undercuts the main value argument and is the point reviewers should press hardest.

**The hedge may already be gone.** Once hash-based signing is usable on QRL 2.0, holders get that protection without the legacy chain. A SPHINCS+-256s wallet implementation and an XMSS verification precompile already exist in the QRL 2.0 codebase.

**Retiring balances removes most of the hedge.** After the snapshot, the only coin still protected by XMSS on the legacy chain is coin mined afterwards. Everything allocated at the snapshot sits on QRL 2.0.

**Burn, mint, reorganize.** If legacy hashrate collapses, the chain becomes cheap to attack. An attacker could burn, receive QRLX, then reorganize the burn away. QIP 16 names a 300 block reorg limit as the main mitigation for 51% attacks on the legacy chain, so a mint should not happen before a burn is deeper than that limit.

**Burn verification is the trust boundary.** Relayers add a trusted party. Checkpoints add a trusted signer set. A light client adds on-chain verification cost. Each option changes who can mint unbacked QRLX.

**Supply.** QRLX supply is the snapshot allocation plus whatever legacy mining emits afterwards. QIP 16 keeps the legacy target at 105m, so bridge mints are bounded by the remaining emission, provided the legacy emission rules do not change again.

**Purpose.** A legacy system with no reason of its own to exist tends to fade. If the diversity argument fails, QRLX has no other basis for value.

## Considerations

Open questions for discussion:

- How does QRL 2.0 verify a legacy burn: light client, checkpoints, or relayers?
- What secures the legacy chain if hashrate collapses?
- Is the snapshot QRLX allocation worth the added complexity, or should QRLX exist only through the bridge?
- QIP 16 said emission would be re-evaluated when Proof-of-Stake arrives. Under this draft the remaining legacy emission becomes QRLX only. Is that the intended outcome?
- Is there enough miner, node and exchange support for a legacy hard fork that retires snapshot balances?

## Note

This document is intentionally abstract. It describes the idea and the invariants, not transaction formats, contracts or parameters. The diagrams are plain text so they read the same on GitHub, on the QIP site, and when pasted into an LLM agent. Share it with yours to critique it, simulate the attack cases, or work out one of the unspecified parts.
