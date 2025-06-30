---
qip: 19
title: Stable Transaction Ordering Policy (STOP)
author: Elliott G. Dehnbostel (@quanta-swap)
layer: core
status: draft/incomplete
comments_uri: ~
created: 2025-05-30
modified: 2025-06-30
--------------------

# Abstract

The **Stable Transaction Ordering Policy (STOP)** introduces a consensus‑level rule that forces every block to list its transactions in a deterministic *nonce‑price* order.
For each sender, transactions **must** appear in strictly increasing `nonce` order; across different senders, the list **must** be globally sorted by **gas‑price descending** (ties broken by sender address and nonce).

By removing the builder’s freedom to permute included transactions, STOP simultaneously:

* **Shrinks the MEV attack surface** — sandwiching and generalized frontrunning require flexible ordering; a fixed order makes many such strategies uneconomical.
* **Reduces effective user‑to‑execution latency** — with ordering fixed, off‑chain “look‑ahead viewers” can forecast state changes in **milliseconds**, letting users and contracts react within the same 60‑second block interval.

Because STOP is enforced during block validation, it is *transaction‑pool agnostic*; miners/validators may keep any mempool they like, but a block whose transaction array violates the rule is invalid.

---

## Motivation

* **Predictability & safety**
  Upcoming protocol‑level and contract‑level features (deterministic state channels, batch‑execution proofs, the Zond upgrade) rely on a predictable, verifiable transaction sequence. Arbitrary re‑ordering makes formal reasoning about state transitions intractable.

* **MEV reduction**
  Builders today shuffle transactions to extract value via sandwiching, back‑running, or gas‑price manipulation. Enforcing a canonical order (highest payer first, per‑account nonces respected) neutralises many of these opportunities while preserving fee‑based prioritisation.

* **Sub‑second interactivity**
  With 60‑second block times, users often observe pending txs and broadcast responses. Deterministic ordering lets *look‑ahead viewers* calculate the exact post‑state **within milliseconds**, allowing contracts and traders to engage in multi‑step “conversations” inside a single block.

* **Implementation simplicity**
  STOP is a single invariant — *“Is the list sorted by (gasPrice↓, sender↑, nonce↑) and are nonces contiguous per sender?”* — that costs < 1 µs to verify.

---

## Specification

### 1. Definitions

| Symbol      | Meaning                                |
| ----------- | -------------------------------------- |
| `tx`        | A transaction object.                  |
| `addr(tx)`  | Sender address of `tx`.                |
| `nonce(tx)` | Sender nonce in `tx`.                  |
| `price(tx)` | **Gas price** paid by `tx`.            |
| `T`         | Transaction list in a candidate block. |

### 2. Ordering predicate

A block **MUST** satisfy **both** predicates:

1. **Per‑sender monotonicity**
   For every sender `A`, and every pair of indices `i < j` such that `addr(T[i]) == addr(T[j]) == A`

   ```
   nonce(T[i]) + 1 == nonce(T[j])
   ```

2. **Global nonce‑price order**
   For every pair of indices `i < j`:

   ```
   if price(T[i]) != price(T[j]):      price(T[i]) > price(T[j])
   else if addr(T[i]) != addr(T[j]):   addr(T[i]) < addr(T[j])   # lexicographic
   else                                nonce(T[i]) < nonce(T[j])
   ```

### 3. Consensus changes

* **Block‑validation rule** — from activation height **H\_STOP** onward, a block failing either predicate is **invalid**.
* No changes to transaction format, gas accounting, or mempool behaviour are required.

### 4. Activation

Activation by hard fork at **block height NNNNNNN** (to be finalised during proposal/open). Clients must enable STOP rules for blocks ≥ `NNNNNNN`.

---

## Rationale

* **Gas‑price‑first preserves fee incentives** — highest payer still wins inclusion ordering.
* **Nonce tie‑breaker avoids intra‑account deadlock** — per‑account semantics remain identical.
* **Address tie‑breaker is deterministic & cheap** — lexicographic compare is free.

### Rejected alternatives

* *Timestamp‑based ordering* — manipulable, non‑deterministic.
* *Random shuffling* — MEV‑proof but destroys fee prioritisation.
* *Auction‑style sequencing* — heavy consensus churn, new attack surface.

---

## Look‑Ahead Viewers (LAVs)

LAVs are off‑chain simulators that forecast the next block’s post‑state:

1. **Snapshot** the public mempool.
2. **Apply STOP ordering.**
3. **Fill** up to the gas limit assuming a profit‑maximising builder.
4. **Execute** to derive state Σ<sub>t+1</sub>.

With ordering fixed, the only randomness is *inclusion*. This turns a factorial search into independent yes/no decisions, giving sub‑second forecasts rather than today’s hand‑wavy guesses.

| Scenario                        | Forecast error |
| ------------------------------- | -------------- |
| Unbounded ordering (status quo) | Very high      |
| STOP enforced                   | Low            |
| Fully fixed block template      | Near‑zero      |

### Benefits unlocked by millisecond forecasts

* **Risk management** — contracts can self‑protect before the block closes.
* **Efficient search** — arbitrageurs spend less gas on failed bundles.
* **Rich intra‑block dialogues** — users & bots chain multiple moves inside one 60‑sec interval.

---

## Backward compatibility

STOP is **consensus‑breaking**: pre‑fork blocks remain valid; post‑fork blocks built by legacy nodes are rejected.

* Long notice period + test‑net rollout.
* Reference client patches provided.
* Blocks < `H_STOP` are not re‑evaluated.

---

## Reference implementation (pseudocode)

```python
ValidateSTOP(txs, baseFee):
    effPrice(tx):
        if tx.type == DynamicFee:
            return min(tx.maxFeePerGas, baseFee + tx.priorityFee)
        return tx.gasPrice

    lastPrice = None
    lastAddr  = None
    lastNonce = 0
    seenNonce = {}

    for idx, tx in enumerate(txs):
        p = effPrice(tx)

        if idx > 0:
            if p > lastPrice:                                   fail("gas‑price order")
            if p == lastPrice and tx.addr < lastAddr:           fail("address order")
            if p == lastPrice and tx.addr == lastAddr and tx.nonce <= lastNonce:
                                                               fail("nonce order")

        lastPrice = p
        lastAddr  = tx.addr
        lastNonce = tx.nonce

        if tx.addr in seenNonce and tx.nonce != seenNonce[tx.addr] + 1:
            fail("non‑contiguous nonce")
        seenNonce[tx.addr] = tx.nonce
```

---

## Security Considerations

| Threat                                | Mitigation                                                               |
| ------------------------------------- | ------------------------------------------------------------------------ |
| **Ordering‑based MEV & frontrunning** | Fixed order removes re‑shuffling leverage.                               |
| **Fee‑spoof griefing**                | Existing risk; not worsened.                                             |
| **Consensus split risk**              | Predicate is trivial to cross‑implement; extensive test vectors planned. |

---

*End of document*
