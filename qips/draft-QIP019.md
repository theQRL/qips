---
qip: 19
title: Stable Transaction Ordering Policy (STOP)
author: Elliott G. Dehnbostel (@quanta-swap)
layer: core
status: draft/incomplete
comments\_uri: \~
created: 2025-05-30
modified: 2025-06-30
---

# Abstract

This proposal introduces the **Stable Transaction Ordering Policy (STOP)**: a consensus-level rule that forces every block to list its transactions in a deterministic *nonce-price* order.
For each sender, transactions **must** appear in strictly increasing `nonce` order; across different senders, the list **must** be globally sorted by **gas-price descending** (ties broken by sender address and nonce).
Because STOP is enforced during block validation, it is *transaction-pool agnostic*: miners/validators may keep any mempool they like, but a block whose transaction array violates the ordering rule is invalid.
The goal is to limit ad-hoc, potentially adversarial re-ordering at block construction time, reducing (not eliminating) systemic MEV surface area and preventing instability that can arise after a forthcoming smart-contract upgrade (Zond).

---

## Motivation

* **Predictability & safety.**
  Upcoming protocol-level and contract-level features (e.g. deterministic state channels and batch execution proofs) rely on a predictable, verifiable transaction sequence inside each block. Unbounded re-ordering makes formal reasoning about state transitions intractable.

* **MEV reduction.**
  Block producers can shuffle transactions to exploit price differences or execute sandwich attacks. Enforcing a canonical order (highest payer first, per-account nonces respected) neutralises many of these opportunities without harming fee-based prioritisation.

* **Implementation simplicity.**
  STOP is a single, easily-checked invariant: *“Is the list sorted by (gasPrice↓, sender↑, nonce↑) and are nonces contiguous per sender?”* Validation adds < 1 µs on commodity hardware.

* **Greater intra-block “conversation.”**
  With 60-second block times, users often see pending transactions and race to respond before the block seals. Deterministic ordering guarantees that a follow-up transaction placed later in the interval will execute **after** the triggering transaction when both land in the same block, enabling richer back-and-forth workflows (e.g. rapid DEX hedging or oracle-mediated multi-party settlements).

---

## Specification

### 1. Definitions

| Symbol      | Meaning                                                                                          |
| ----------- | ------------------------------------------------------------------------------------------------ |
| `tx`        | A transaction object.                                                                            |
| `addr(tx)`  | Sender address of `tx`.                                                                          |
| `nonce(tx)` | Sender nonce in `tx`.                                                                            |
| `price(tx)` | **Gas price** paid by `tx`. |
| `T`         | Transaction list in a candidate block.                                                           |

### 2. Ordering predicate

A block **MUST** satisfy **both** predicates:

1. **Per-sender monotonicity**
   For every sender `A`, and every pair of indices `i < j` such that `addr(T[i]) == addr(T[j]) == A`

   ```
   nonce(T[i]) + 1 == nonce(T[j])
   ```

2. **Global nonce-price order**
   For every pair of indices `i < j`:

   ```
   if price(T[i]) != price(T[j]):      price(T[i]) > price(T[j])
   else if addr(T[i]) != addr(T[j]):   addr(T[i]) < addr(T[j])   # lexicographic
   else                                nonce(T[i]) < nonce(T[j])
   ```

### 3. Consensus changes

* **Block-validation rule** — starting at activation height **H\_STOP**, a block failing either predicate is **invalid**.
* **No changes** to transaction format, gas accounting, or mempool behaviour are required.

### 4. Activation

Activation by hard fork at **block height NNNNNNN** (to be finalised during proposal/open). Clients must enable STOP rules for blocks ≥ `NNNNNNN`.

---

## Rationale

* **Gas-price-first preserves fee incentives.**
  Sorting primarily by `price` retains the “highest payer first” economic signal while making the ordering transparent.

* **Nonce tie-breaker avoids intra-account deadlock.**
  Contiguous nonces keep per-account semantics identical to the status quo.

* **Address tie-breaker is deterministic and cheap.**
  Lexicographic address order costs nothing to compute and prevents ambiguous ordering when two senders pay the same price.

* **Rejected alternatives**

  * *Timestamp-based ordering* — manipulable and non-deterministic across nodes.
  * *Random shuffling* — removes MEV but destroys fee-priority guarantees.
  * *Auction-style sequencing* — heavier consensus changes, introduces new attack surface.

---

## Look-Ahead Viewers

“Look-Ahead Viewers” (LAVs) are off-chain simulators that attempt to preview the next block’s post-state before it is mined.

1. **Snapshot public mempool.**
2. **Apply STOP ordering** to the snapshot.
3. **Assume profit-maximising builder** who top-fills by gas price until the block-gas limit.
4. **Execute the ordered list** against the current state to derive a candidate future state Σ<sub>t+1</sub>.

With STOP, the only uncertainty is whether a transaction is *included*; ordering is fixed. This collapses a factorial search space into independent yes/no decisions, giving LAVs far higher predictive accuracy.

| Scenario                        | Forecast error |
| ------------------------------- | -------------- |
| Unbounded ordering (status quo) | Very high      |
| STOP enforced                   | Low            |
| Fully fixed block template      | Near-zero      |

LAVs enable:

* **Risk management** — contracts can query imminent state to trigger pre-emptive safeguards.
* **More efficient search** — arbitrageurs can evaluate profitability with fewer failed bundles.
* **Formal verification** — provers can enumerate the small set of reachable post-states.

---

## Backward compatibility

STOP is a **consensus-breaking** change: pre-fork blocks remain valid; post-fork blocks constructed by legacy nodes will be rejected.

Mitigations:

* Long notice period and test-net rollout.
* Reference patches for major clients.
* Blocks before `H_STOP` are **not** re-evaluated, preserving chain history.

---

## Reference implementation (pseudocode)

```
ValidateSTOP(txs, baseFee):
    effPrice(tx):
        if tx.type == DynamicFee:
            cap = min(tx.maxFeePerGas, baseFee + tx.priorityFee)
            return cap
        return tx.gasPrice

    lastPrice = None
    lastAddr  = None
    lastNonce = 0
    seenNonce = {}

    for idx, tx in enumerate(txs):
        p = effPrice(tx)

        if idx > 0:
            if p > lastPrice:                                   fail("gas price order")
            if p == lastPrice and tx.addr < lastAddr:           fail("address order")
            if p == lastPrice and tx.addr == lastAddr and tx.nonce <= lastNonce:
                                                               fail("nonce order")

        lastPrice = p
        lastAddr  = tx.addr
        lastNonce = tx.nonce

        if tx.addr in seenNonce:
            if tx.nonce != seenNonce[tx.addr] + 1:
                fail("non-contiguous nonce")
        seenNonce[tx.addr] = tx.nonce
```

---

## Security Considerations

| Threat                                     | Mitigation                                                                                                            |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------- |
| **Block-producer censorship via ordering** | Producers can still omit txs, but cannot rearrange those they include—removing many front-running & sandwich vectors. |
| **Fee-spoof griefing**                     | Attackers broadcasting many max-price txs to dominate the list already exists; STOP does not worsen it.               |
| **Consensus split risk**                   | Predicate is simple; extensive test vectors and cross-client test-suites will accompany implementation.               |

---

*End of document*
