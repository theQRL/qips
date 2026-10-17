---
qip:
title: "Keep QRL 1.0 running alongside QRL 2.0"
author: alami (@alamistoneq-hub)
layer: core
status: "draft/incomplete"
comments_uri:
comments_summary_uri: ~
created: 2026-10-11
modified: 2026-10-11
---

## Abstract

Keep the QRL 1.0 chain running after the QRL 2.0 snapshot instead of archiving it. Holders get their full balance on 2.0 as planned and keep their 1.0 coins too. 2.0 keeps the QRL ticker and 1.0 takes a new one.

The QRL Foundation is the preferred steward, running 1.0 in maintenance mode. It also has a direct stake, since the 8.45 million QRL it has held since genesis would be credited on 2.0 and stay live on 1.0 as well. If it prefers not to run 1.0, it can hand its 1.0 coins to a new maintainer. If neither happens by an agreed date, the current plan proceeds.

## Motivation

- Holders keep both hash-based proof-of-work (1.0) and lattice-based proof-of-stake with smart contracts (2.0).
- 1.0's accounts and consensus don't depend on lattices. 2.0's validators sign with ML-DSA-87, so its consensus rests on lattices even after hash-based wallets arrive.
- Nobody is forced to move, and 1.0's record of running without interruption since 2018 continues.
- Miners keep a home.
- Shutting down 1.0 hands critics an easy line, "QRL shut down its original chain," and leaves the door open to an unofficial continuation that nobody maintains, since a halt only stops nodes that update. For a project built on long-term security, that risks tying the QRL name to unreliability, which is hard to undo. Keeping 1.0 avoids all of this and keeps QRL's record of running without interruption since 2018.

## Specification

1. **No halt.** 1.0 keeps producing blocks after the snapshot, and its balances stay spendable.
2. **Snapshot as planned.** Every 1.0 balance is credited on 2.0 as already planned.
3. **Key rule.** XMSS sub-keys are one-time. The 2.0 claim tool signs with each address's highest unused sub-key below index 8,192, while 1.0 wallets keep using the lowest, so no sub-key is used on both chains.
4. **Tickers.** 2.0 keeps QRL. 1.0 takes a new ticker, and the Foundation has the final say.
5. **Steward.**
   - **Option A (preferred).** The Foundation runs 1.0 in maintenance mode, security fixes only. It can hire a maintainer, including a community member, and can switch to Option B later. 2.0's staff and timeline are untouched.
   - **Option B.** The Foundation sells or grants its post-snapshot 1.0 coins to a maintainer chosen through an open call. The coins sit in a public multisig with a release schedule.
   - **Fallback.** If neither option is confirmed by an agreed date before the snapshot, the current plan proceeds.

## Rationale

Unlike a token on 2.0 (PR #47), keeping the chain needs no hard fork and no bridge, and the hash-based protection stays with the coins.

The Foundation is preferred because it already has the knowledge, infrastructure and trust. It also stands to benefit. Its 8.45 million QRL, held untouched since genesis, would be credited on 2.0 and stay live on 1.0 as well.

Different people want different things from a chain. Some want smart contracts and staking, which 2.0 offers. Others want a simple store of value, which 1.0 already is. Keeping both serves both groups instead of making everyone choose.

1.0 is also in a category of its own, a post-quantum proof-of-work store of value with hash-based keys, running without interruption since 2018. New projects are launching into this space, such as Quantus in 2026, but with lattice signatures. The market sets the value of each asset, and this QIP makes no claim about price.

## Backward compatibility

No consensus change on either chain. The only tooling change is the key rule in the 2.0 claim tool. Exchanges list 2.0 as QRL and decide for themselves whether to support 1.0 under its new ticker.

## Reference Implementation

None needed beyond the key rule in the claim tool.

## Security Considerations

- **Mining.** Lower hashrate makes attacks cheaper. The existing 300-block reorg limit, exchange confirmation rules and public hashrate monitoring reduce this. Merge mining or a reorg penalty can follow later through their own QIPs if needed.
- **Key reuse.** Covered by the key rule, which matters whenever 1.0 keeps running, official or not. Users with few unused sub-keys left should move their 1.0 coins to a fresh address before claiming.
- **Replay.** The chains use different signature schemes and transaction formats, so transactions can't be replayed between them. Claim messages should carry a distinct prefix so they can't be mistaken for 1.0 transactions.
- **Scams.** Holders already have their 1.0 coins and never need to claim them.

## Considerations

- Does the Foundation prefer Option A or B?
- What ticker should 1.0 use?
- QIP 16 was approved on active votes after falling short of its approval threshold. The editors are asked to state in advance how turnout and active votes will be weighed.
