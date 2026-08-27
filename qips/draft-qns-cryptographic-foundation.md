---
qip:
title: "QNS Cryptographic Foundation: SHAKE256 and ML-DSA-87 Precompiles"
author: DigitalGuards (@DigitalGuards)
layer: core/security
status: draft/incomplete
comments_uri:
comments_summary_uri:
created: 2026-08-22
updated: 2026-08-27
---

## Abstract

This QIP defines the post-quantum contract primitives that form the cryptographic foundation of QRL Name Service (QNS) and remain reusable by other QRL protocols. It standardizes the existing ML-DSA-87 verification precompile at address `0x03` and adds SHAKE256 with a fixed 64-byte output at address `0x06`. The verifier operates over a fixed 64-byte message representative with an explicit FIPS 204 context string. It receives the signature and public key because ML-DSA has no public-key recovery operation equivalent to `ecrecover`.

SHAKE256 returns one 64-byte QRL 2.0 virtual-machine word. Successful ML-DSA-87 verification returns the canonical 64-byte word ending in `0x01`. The current go-qrl implementation returns empty data for invalid or malformed verification, following an `ecrecover`-style convention. The QRL implementation lead has confirmed that empty data versus a canonical boolean remains open before release. Out-of-gas behavior follows the normal precompile call path. The proposal also specifies Hyperion global builtins named `shake256` and `mldsa87verify` so contract authors can call the operations without hand-building static calls. During review, Hyperion maps both candidate failure forms to `false` and accepts only the exact success word as `true`.

The reference implementation adds a named QRL 2.0 post-quantum precompile rule to go-qrl. At activation it changes the slot `0x03` message representative from the legacy 32-byte frame to the ratified 64-byte frame and adds SHAKE256 at the confirmed unused slot `0x06`. Fresh QRL 2.0 networks activate the rule at genesis. Hyperion compiler support, QNS contract and SDK consumers, explicit artifact target metadata, and a predeployment live-network probe share the same slot map. The aligned end-to-end composition passed on 2026-08-26. Final gas ratification remains a review item before this draft advances.

## Motivation

QRL 2.0 contracts need a practical way to verify post-quantum authorization. The ECDSA `ecrecover` model recovers a public key or signer address from an elliptic-curve signature. ML-DSA verification requires the public key as an explicit input and returns only a validity result. Implementing ML-DSA-87 in contract bytecode would add substantial execution cost, code size, and consensus risk.

SHAKE256 is used throughout the QRL cryptographic stack and provides a fixed-width message representative for contract protocols. A native 64-byte digest composes naturally with the QRL 2.0 64-byte word size and with ML-DSA-87 verification. QNS is the anchor consumer: signed naming records need post-quantum authorization, stable domain separation, and deterministic verification inside Hyperion contracts. The established wire context remains `QNS-SIGN-v1`, preserving the reviewed signed bytes.

## Specification

### Addresses and activation

This proposal covers two precompiled contracts at the following QRL execution addresses:

| Slot | Operation |
| --- | --- |
| `0x03` | ML-DSA-87 detached signature verification |
| `0x06` | SHAKE256 with a 64-byte output |

The addresses are encoded as native 64-byte QRL addresses with the slot number in the least significant byte. Before this rule activates, slot `0x03` retains the legacy 32-byte message-representative frame and slot `0x06` has no precompile behavior. At activation, slot `0x03` uses the 64-byte frame specified here and slot `0x06` provides SHAKE256. Both operations are active from genesis on the next QRL 2.0 testnet, as confirmed by the QRL implementation lead on 2026-08-25. A network that already has blocks under the earlier map MUST schedule the same rule at a coordinated timestamp.

### SHAKE256 precompile

The SHAKE256 precompile accepts the complete call input as an arbitrary byte string. It returns exactly 64 bytes equal to `SHAKE256(input, 64)`.

Required gas is:

```text
240 + 48 * ceil(input_length / 64)
```

An empty input has zero words and costs 240 gas. Implementations MUST calculate the word count and multiplication without integer wraparound. An unrepresentable cost saturates to the maximum gas integer and therefore cannot execute under a smaller gas limit.

### ML-DSA-87 verification precompile

The verifier accepts one raw byte string with this exact layout:

| Field | Length |
| --- | ---: |
| `messageRepresentative` | 64 bytes |
| `publicKey` | 2592 bytes |
| `signature` | 4627 bytes |
| `contextLength` | 1 byte |
| `context` | 0 to 255 bytes |

The fixed portion is 7284 bytes. Total valid input length is 7284 to 7539 bytes inclusive. `contextLength` is an unsigned byte and MUST equal the number of trailing context bytes. Any missing, excess, or inconsistent byte makes the input malformed.

Verification invokes the FIPS 204 ML-DSA-87 verification operation with `context` as the context string, `messageRepresentative` as the message, `signature` as the detached signature, and `publicKey` as the public key.

Success return data is fixed:

- Valid signature: 63 zero bytes followed by `0x01`.

Failure return data remains a proposal decision between:

- Current implementation: empty return data.
- Canonical boolean alternative: 64 zero bytes.

Consensus implementations MUST select one failure form before release and apply it identically to invalid signatures and malformed input. Callers written during the review period should treat either candidate failure form as false and reject every other noncanonical value.

Malformed input includes any length outside the allowed range. The verifier MUST NOT recover, derive, or return an account address. Applications MUST define and enforce their own binding between the supplied ML-DSA public key and an identity, account, or authorization record.

Required gas is 125000 for every input, matching the current go-qrl implementation. This value remains open to adjustment through proposal review if cross-platform benchmarks or denial-of-service analysis justify a change. A call with less than the required gas fails with the execution layer's normal out-of-gas result and returns no value.

### Hyperion builtins

The QRL 2.0 Hyperion build exposes these pure global functions:

```hyperion
function shake256(bytes memory input) pure returns (bytes64)
function mldsa87verify(
    bytes64 messageRepresentative,
    bytes memory signature,
    bytes memory publicKey,
    bytes memory context
) pure returns (bool)
```

The compiler packs verifier arguments as `messageRepresentative || publicKey || signature || uint8(context.length) || context` and issues a static call to `0x03`. It rejects component lengths outside the specified bounds through a malformed precompile call, which preserves the fixed precompile gas charge. Empty return data and the canonical 64-byte zero word map to `false` during protocol review. Only exactly 64 returned bytes whose value is one map to `true`; other noncanonical return data maps to `false`. SHAKE256 issues a static call to `0x06`, requires exactly 64 returned bytes, and reverts on missing, short, or oversized successful return data.

### Test vectors

SHAKE256 implementations MUST produce these outputs:

```text
SHAKE256("", 64)
46b9dd2b0ba88d13233b3feb743eeb243fcd52ea62b81b82b50c27646ed5762f
d75dc4ddd8c0f200cb05019d67b592f6fc821c49479ab48640292eacb3b7c4be

SHAKE256("abc", 64)
483366601360a8771c6863080cc4114d8db44530f8f1e1ee4f94ea37e78b5739
d5a15bef186a5386c75744c0527e1faa9f8726e462a12a4feb06bd8801e751e4
```

The reference go-qrl suite carries a reproducible ML-DSA-87 vector in `core/vm/testdata/precompiles/mldsa87_verify.json`, regenerated by `TestMLDSA87VerifyVectorProvenance`: seed = 32 bytes of `0x51`, `messageRepresentative` = `SHAKE256("QNS local integration vector", 64)` = `8b4452b0...96f4a5ed`, context = `QNS-SIGN-v1`, deterministic (non-hedged) FIPS 204 signing. The valid entry MUST return the success word; the mutated-digest and wrong-context entries MUST fail. The suite also mutates each input field of a fresh keypair to confirm failure. The full serialized vector is attached to the proposal-status submission.

## Rationale

The current QRL execution registry assigns ML-DSA-87 verification to `0x03`. Slot `0x06` is unused and is assigned to SHAKE256. Existing operations occupy `0x01`, `0x02`, `0x04`, and `0x05`.

A fixed 64-byte SHAKE256 output matches `bytes64` and one QRL 2.0 virtual machine word. A separate SHAKE256 operation is useful beyond signature verification and avoids coupling message hashing to one application protocol.

The verifier uses a fixed 64-byte message representative to make the packed layout unambiguous and to bound consensus work. Contracts can compute it with the SHAKE256 precompile or receive a digest computed elsewhere. This construction is an application protocol built from pure ML-DSA-87 over 64 bytes. It MUST NOT be labeled as the distinct HashML-DSA mode unless an implementation actually follows that standard's prehash procedure and identifiers.

The context remains caller supplied because FIPS 204 context strings provide protocol domain separation. The 255-byte bound follows the ML-DSA interface. Applications should select a stable, non-empty context and treat changes as a signing-protocol version change.

The current empty-data failure behavior follows a familiar `ecrecover`-style convention. A canonical zero word gives direct low-level callers a fixed-width boolean and avoids special return-size handling. The Hyperion builtin provides a stable boolean interface across both candidates by accepting only the canonical success word as true. Execution failure remains reserved for insufficient gas or an internal consensus implementation failure.

Alternatives considered include a new opcode, contract-level cryptographic code, arbitrary-length signed messages, and a verifier that derives a QRL address. Precompiles fit the existing execution architecture. Fixed field lengths remove parser ambiguity. Identity binding remains an application decision because multiple address and key registration schemes can consume the same verifier.

## Backward compatibility

The pre-activation go-qrl map assigns the legacy 32-byte ML-DSA-87 frame to native address `0x03`. This proposal changes that interface to 64 bytes at the same activation boundary that adds SHAKE256 behavior at the currently unused address `0x06`. Networks can observe different success, gas, and return-data behavior after activation.

The next QRL 2.0 testnet activates both operations from genesis, so every participant runs the same registry from block one. Any network with earlier blocks requires coordinated execution-client activation at a timestamp boundary. Review artifacts identify the `qrl2-pq-v1` target, compiler binary hash, source-tree hash, and artifact hashes. The QNS deployment path requires that target in network configuration and executes live slot `0x03` plus `0x06` probes before its first transaction.

The go-qrl tracer fixtures demonstrate that precompile activation is observable beyond return data. The existing `0x03` verifier changes tracing, and the activated `0x06` SHAKE256 call changes the inner-call gas and return values. The reconciled activated fixtures passed both focused tracer tests and `go test ./...`. Mixed clients can diverge in execution and tracing as soon as either address is called.

The standardized `0x03` interface preserves the current implementation's address, raw layout, and gas charge. Its failure return convention remains a release decision and may affect direct low-level callers. Existing contract bytecode that does not call `0x03` or `0x06` retains its behavior.

## Reference Implementation

The review implementation consists of:

- go-qrl precompile registration, gas accounting, SHAKE256 execution, and ML-DSA-87 verification.
- Hyperion type-system, legacy code generator, IR code generator, formal-model, documentation, and execution-host support.
- QNS Hyperion contract and TypeScript SDK examples using `QNS-SIGN-v1`.
- A Kurtosis configuration for a local 64-byte QRL 2.0 network.

The reviewed go-qrl base declares `github.com/theQRL/go-qrllib v0.8.0` and currently resolves it through a `replace` directive to `github.com/rgeraldes24/go-qrllib v0.1.1-0.20260707094212-a6d78f111b1f`. The replacement module declares the official module path. The reviewed module checksum is `h1:yhR6S+o8Fz2DZojtOAvyORd8msr+vyehEmZjDrxvVw8=` and its `go.mod` checksum is `h1:cJalbgwzfscRXz7gqwPmmeC2wxB/QJh631N1dpihXuI=`. Consensus release provenance must name the actually resolved commit and either move to an audited official release containing that code or explicitly ratify the replacement before activation.

The aligned local composition pinned `cyyber/qrysm@b53fd7c4` and `theQRL/qrl-genesis-generator@6a11fbce` because the published Qrysm images inspected during validation predated the 64-byte changes. A tracked generator patch set `qrl2PQPrecompilesTime: 0`. On 2026-08-26 the network produced blocks, deployed six QNS contracts, exercised native 64-byte forward and reverse resolution, passed all nine direct and wrapped PQ phases, and passed eight live lifecycle and authorization subtests. Every observed Kurtosis service mapping used a loopback host address. The enclave and stable RPC proxy were stopped after validation.

On the aligned compiler, Hyperion's CHC engine, backed by Z3 4.12.1, proved 36 source-coupled QNS security targets. These cover exact digest-boundary dispatch, every byte and index bound of `QNS-SIGN-v1`, deterministic formal calls, resolver capability predicates, unauthorized transition models, and reverse-index arithmetic. The proof gate rejects unsafe, unproved, unavailable, unsupported, missing, or unexpected targets. Hyperion models the cryptographic operations as deterministic uninterpreted functions, so concrete cryptographic security remains grounded in the resolved implementations and differential vectors. The complete Hyperion suite reported 7,184 passing tests on the same review tree.

The initial community review is pinned to these public references:

| Component | Reviewed source | Review thread |
| --- | --- | --- |
| go-qrl | [`39be3a8`](https://github.com/DigitalGuards/go-qrl/commit/39be3a83cb956adc9796ceee1783f97df3f7e2a5) | [cyyber/go-qrl#191](https://github.com/cyyber/go-qrl/pull/191) |
| Hyperion | [`6f862206`](https://github.com/DigitalGuards/hyperion/commit/6f862206ff34bce56098cc23b4b3f575e0ea3c5f) | [cyyber/hyperion#18](https://github.com/cyyber/hyperion/pull/18) |
| QNS contracts, SDK, vectors, and composition | [`e20a25f`](https://github.com/DigitalGuards/myqrlwallet-qns/commit/e20a25fca84d3eb5299ec80c5f154b60224183e4) | [DigitalGuards/myqrlwallet-qns#4](https://github.com/DigitalGuards/myqrlwallet-qns/pull/4) |
| Qrysm beacon and validator | [`b53fd7c4`](https://github.com/cyyber/qrysm/commit/b53fd7c488f3f0d1d4163b270afac1749eed954b) | Source pin |
| Genesis generator | [`6a11fbce`](https://github.com/theQRL/qrl-genesis-generator/commit/6a11fbcee762af14d188507f071d08ac5782fa69) plus [activation patch](https://github.com/DigitalGuards/myqrlwallet-qns/blob/e20a25fca84d3eb5299ec80c5f154b60224183e4/docker/qrysm/qrl-genesis-generator-qrl2-pq.patch) | Source pin |

The deterministic serialized ML-DSA-87 interoperability vector is published with [the QNS SDK fixtures](https://github.com/DigitalGuards/myqrlwallet-qns/blob/e20a25fca84d3eb5299ec80c5f154b60224183e4/sdk/src/fixtures/qns-vectors.json).

## Security Considerations

The ML-DSA public key is attacker-controlled input. Successful cryptographic verification proves that the signature matches that key, message representative, and context. It does not prove that the key belongs to a claimed QRL account. Every consuming contract must validate the key-to-identity binding required by its protocol.

Context strings are security boundaries. Reusing the same context and message format across unrelated protocols can enable cross-protocol signature reuse. Applications should define a versioned context and a canonical, injective message encoding.

Hashing and signing must agree exactly. A protocol that signs raw messages while a contract verifies `SHAKE256(message, 64)`, or that uses a different context, will reject valid user intent. Wallets and contracts should publish shared test vectors for the complete message construction.

Consensus clients must pin an audited ML-DSA-87 implementation and validate that arbitrary fixed-length signatures and public keys cannot panic, allocate without bounds, or produce platform-dependent results. Differential vectors should cover valid signatures, each mutated field, empty and maximum contexts, malformed lengths, and out-of-gas execution.

Dependency declarations must identify the code that consensus builds actually resolve. A module version paired with a fork replacement can obscure the reviewed source if release notes mention only the declared version. Build manifests and QIP evidence should record the resolved module path, pseudoversion, commit, and checksum.

Gas pricing must cover worst-case verification cost on supported validator hardware with a conservative margin. The fixed charge prevents malformed inputs from receiving a discount, but the current value remains subject to benchmark review. Five local runs measured medians near 323 ns for SHAKE256 over 64 bytes and 180 microseconds for ML-DSA-87 verification. At 125000 gas, a 30 million gas block permits at most 240 verifications, approximately 43 milliseconds at that measured median before scheduling and execution overhead. Cross-platform validator measurements and worst-case distributions are still required.

SHAKE256 gas calculation must resist integer overflow. Both precompiles must return newly owned or immutable output buffers so concurrent execution cannot corrupt results.

## Open review items

1. Resolved on 2026-08-27: DigitalGuards is the initial author and champion. Additional co-authors can be added during QRL team review.
2. Resolved on 2026-08-25: go-qrl gates the 64-byte slot `0x03` frame and slot `0x06` behind one timestamp rule. The next QRL 2.0 testnet sets that timestamp to zero in genesis. Existing networks can assign a later coordinated timestamp.
3. Ratify or adjust the current 125000 gas charge using cross-platform benchmark data.
4. Resolved on 2026-08-27: the reference table links the complete deterministic ML-DSA-87 serialized interoperability vector and its source provenance.
5. Confirm whether the core API should call the first field `messageRepresentative`, `digest`, or another consensus term.
6. Resolved for the review implementation on 2026-08-25: compiler artifacts declare the `qrl2-pq-v1` target and exact compiler/source/artifact hashes. QNS deployment requires that configured target and probes both precompiles before any transaction. A future general-purpose Hyperion release can expose explicit multi-fork target selection if it must emit bytecode for older networks.
7. Select empty return data or the canonical 64-byte zero word for invalid and malformed verification.
8. Publish the exact go-qrllib module and checksum used by consensus builds. The QRL implementation lead confirmed on 2026-08-24 that the replacement at commit `a6d78f111b1f` is a temporary testing dependency and that all replacements are updated before the new testnet.
9. Resolved on 2026-08-25: the QRL implementation lead ratified the 64-byte message-representative width. The reference go-qrl implementation, this draft, the Hyperion builtin, and the QNS consumer all read a 64-byte field with a fixed portion of 7284 bytes. The previously deployed verifier read a 32-byte field (`common.HashLength`, fixed portion 7252) and changes with the next testnet release; pre-release callers of `0x03` MUST NOT assume the 32-byte frame.
