---
qip:
title: "Beacon Block Roots for QRL Contracts"
author: DigitalGuards (@DigitalGuards)
layer: core
status: draft/incomplete
comments_uri:
comments_summary_uri:
created: 2026-10-01
updated: 2026-11-18
---

## Abstract

This draft brings Ethereum's [EIP-4788](https://eips.ethereum.org/EIPS/eip-4788), "Beacon block root in the EVM", to QRL 2.0. EIP-4788 has run on Ethereum since the Cancun upgrade of March 2024, and its mechanism is kept as it is. Each activated execution block commits to the root of its beacon parent in its header. Before ordinary transactions execute, the execution client records that root in a protocol-maintained history contract under the block's timestamp. The consensus client supplies the root and checks it against the actual parent of the enclosing beacon block.

Contracts can then verify consensus-state proofs against the same chain's authenticated history. A staking pool, for example, can prove a finalized checkpoint through the parent beacon state, authenticate the header named by that checkpoint, and read validator records and balances from that header's post-state. Finality and freshness checks stay explicit at the application boundary.

What changes for QRL is the adaptation to 64-byte VM words and addresses, the way the history contract is installed and the handling of a failed system call. Roots remain 32 bytes, and the [Rationale](#rationale) lists every difference. A reference implementation exists in go-qrl and Qrysm behind an activation time that is unset by default. A local network built from it reached finality with the feature active from genesis. The reserved addresses, history capacity, gas budget, Engine method names and activation mechanism used there are experimental values. This draft asks the QRL team to review the design and to assign them.

## Motivation

Beacon block roots are cryptographic accumulators. A root commits to the whole consensus state, so a short Merkle proof against it can establish any fact that state holds. Today a QRL contract has no authenticated view of the consensus layer, and an application must run its own consensus anchor or trust a report. Recording the roots through consensus rules gives every contract the same source of proof authentication. Anyone can supply the proofs, independently of the party a contract deals with.

Ethereum gave this motivation for EIP-4788 and named staking pools, restaking, smart contract bridges and MEV mitigations as uses. On QRL the following uses stand out. Each rests on fields that Qrysm's beacon state and blocks already carry.

- **Staking pools.** A pool contract can read each validator's withdrawal recipient, effective balance, slashed flag, its activation, exit and withdrawable epochs, and its current balance. Its accounting of rewards, penalties and exits can then rest on proven values, with no reporting committee as the source of truth.
- **Operator bonds and slashing cover.** A contract can forfeit an operator's bond or pay a cover claim on a proven slashing or balance loss.
- **Reward accounting for any withdrawal recipient.** A consensus withdrawal credits an account without a transaction, receipt or log, so the receiving contract cannot tell it apart from any other incoming balance. A proof of a block's withdrawal list gives a reward splitter, a treasury or a multisig the exact amount per validator.
- **Proposer facts.** The beacon header names the proposer of a slot and the block's payload names its fee recipient. A tip-sharing agreement can detect a block whose fees went elsewhere, and proposer commitments become checkable.
- **Validator identity in other applications.** An account can prove that it is a validator's withdrawal recipient. That supports stake-weighted voting, validator registries and verified validator names in a name service or an explorer.
- **Finality and history.** A contract can require that an event sits in a finalized block before it acts, which serves bridges, swap settlement and oracles. Through the state's block-root and historical-summary accumulators it can also reach block hashes, receipts and state roots older than the 256 blocks that `BLOCKHASH` covers, for example for balance snapshots in votes and token distributions.

The primitive supplies authentication only. Each application still chooses its proof schema, finality requirements, freshness policy and accounting rules.

## Specification

### Terminology

"Parent root" means the SSZ hash tree root of the parent beacon block, which is the `parent_root` of the current beacon block. "Write timestamp" means the timestamp of the current execution block. "Activation" means the fork timestamp from which these rules apply. Values marked experimental are those of the reference implementation. They are collected under [Open review items](#open-review-items).

### Consensus binding and execution header

1. Add a fork-gated, exactly 32-byte `parent_beacon_block_root` to the execution header's RLP encoding, block hash, JSON representation and copy and accessor paths. The reference implementation appends it as the last header field and names it `parentBeaconBlockRoot` in JSON.
2. Preserve historical header encoding before activation. At and after activation the field is mandatory. A header whose field presence disagrees with activation is invalid.
3. The consensus client derives the expected value from the enclosing beacon block's parent root on import and from the proposed parent during block construction. Proposer-supplied metadata cannot override that relationship.
4. The Engine API supplies the value to payload building and to import. The execution client includes it when it reconstructs the header and checks the block hash. A payload built for a different parent root must be distinguishable in payload IDs and cache keys.
5. The parent root travels as an Engine payload attribute and as a separate argument on payload import. It need not become an additional SSZ field inside Qrysm's `ExecutionPayloadZond`, because the enclosing beacon header already commits to its parent. Fork-specific wrappers, generated codecs and validation still require coordinated changes.
6. An execution client alone cannot establish that an arbitrary root names the correct beacon parent. Correctness requires both the consensus-side binding check and the execution-side commitment and state transition check.

### History operation

At the start of every activated execution block, before ordinary transactions, perform a system write using the current block's timestamp and the parent root. The same operation is required when constructing payloads, importing blocks, re-executing history and processing blocks with zero transactions. Transaction tracing and other replay paths that start directly from parent state must also run this operation before replaying the first transaction. A shared helper must give normal execution and tracing identical root reads.

For a history length `H`, the contract stores the write timestamp in slot `timestamp mod H` and the root in slot `H + (timestamp mod H)`. A read succeeds only if the stored timestamp equals the queried one and is nonzero. A timestamp collision replaces the previous pair. Entries are ordinary contract storage, so they persist in state and roll back with the rest of execution state during a reorg.

The experimental value is `H = 8191`, the buffer length of EIP-4788. Capacity selection must consider actual QRL slot timing and proof-supply latency. With a 60-second slot grid and this prime capacity, the same slot-indexed timestamp recurs after 8191 slots, approximately 5.69 days. Skipped slots, timing changes and genesis offsets must be tested against the actual timestamp sequence. The proposal makes no availability promise for a historical root beyond its retained entry.

### History contract interface

| Operation | Authorized caller                        | Input                                                                        | Output                    |
| --------- | ---------------------------------------- | ---------------------------------------------------------------------------- | ------------------------- |
| Read      | Any contract or externally owned account | Exactly 64 bytes containing a big-endian timestamp with value in `1..2^64-1` | Exactly 32 raw root bytes |
| Write     | Reserved protocol system caller          | Exactly 32 parent-root bytes; timestamp from execution context               | Empty                     |

The read input is one native VM word whose upper 56 bytes must be zero. This is a dedicated raw interface without a function selector. Every other input reverts: a call that carries value, a read of the wrong length, a zero or oversized timestamp, a timestamp whose slot holds a different entry, and a write of the wrong length. The caller comparison uses all 64 address bytes.

`CALLDATALOAD` and `MSTORE` operate on 64-byte words in QRVM. The runtime therefore places the 32-byte root in the high half of its storage word and returns exactly those 32 bytes. Ethereum's deployed EIP-4788 bytecode would give incorrect word and address behavior and cannot be reused. A Hyperion consumer must check the 32-byte return size and convert those bytes to `bytes32` explicitly.

The reference client assembles the runtime from its own opcode table for a given system caller and capacity. For the experimental system caller and `H = 8191` it is 173 bytes with Keccak-256 hash `0x8d39b7c3450e45e7f3a73b8af0dc10a494c5d88b0d5d83f794f0194c65b80b84`:

```text
0x346100a757339ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffe146100825736604014156100a757600035a0156100a757a060401c6100a757a0611fffb006a054a214156100a757611fff015460005260206000f35b36602014156100a75742156100a75742611fff420655600035611fff4206611fff0155005b60006000fd
```

### Installation

The experimental history address is the 64-byte address whose first 62 bytes are `0xff` and whose last two bytes are `0x47 0x88`. The experimental system caller is 63 bytes of `0xff` followed by `0xfe`, which is EIP-4788's `SYSTEM_ADDRESS` widened to 64 bytes. Neither has a public-network allocation.

The client installs the runtime itself. No deployment transaction is involved.

- Activation at genesis. The client adds the history account to the genesis state with the exact runtime, nonce 1 and empty storage. An allocation at the reserved address is accepted when it is empty (no code, nonce 0, no storage) or already holds the exact runtime with nonce 1 and no storage, and its balance is kept. Any other allocation there is a genesis error. The genesis header carries a zero parent root and no history write takes place, as in EIP-4788.
- Scheduled activation. In the first block at or after activation, before the first write, the client installs the runtime at the reserved address. The address must have nonce 0, no code and empty storage at that point; any other state makes the block invalid. An existing balance is kept, so that a transfer to the reserved address before the fork cannot stop activation.

Before every write the client checks that the code at the history address equals the expected runtime. Address allocation must rule out collisions with existing accounts on any network that activates the feature after genesis.

### System call rules

The write is a system operation with these properties:

- The caller is the reserved system caller, the value is zero and the input is the 32 parent-root bytes.
- It runs with a dedicated, bounded gas budget. The experimental budget is 100,000 gas, and the reference helper refuses any system call above 1,000,000 gas. EIP-4788 uses 30,000,000.
- It consumes no sender nonce or balance and pays no fee. It produces no receipt and no logs and does not count toward the block's gas used. Its storage effects are part of the block's state root.
- Access-list warmth, refund counters and original storage values are reset afterwards, so the write cannot change the gas cost of ordinary transactions.
- A precompile address is refused as a target.

A block is invalid if the runtime is missing, differs from the expected code, runs out of its gas budget, reverts or returns any output. This is stricter than EIP-4788. There the call fails silently when no code exists, and go-ethereum ignores any other failure of it. The stricter rule is the one that [EIP-7002](https://eips.ethereum.org/EIPS/eip-7002) adopted for its own system call, and it requires protocol review.

The `pending` RPC view has no consensus-selected beacon parent. It skips the write for its own timestamp, as go-ethereum's pending block does. Reads of earlier canonical timestamps keep working there.

### Engine API

The reference implementation adds three experimental methods and advertises them in capability exchange. Names and versions are open.

| Method                                     | Addition                                                      |
| ------------------------------------------ | ------------------------------------------------------------- |
| `engine_forkchoiceUpdatedWithBeaconRootV1` | Payload attributes carry `parentBeaconBlockRoot`              |
| `engine_getPayloadWithBeaconRootV1`        | Returns a payload that was built for an activated timestamp   |
| `engine_newPayloadWithBeaconRootV1`        | Takes the parent root as a second argument beside the payload |

Each method is tied to the fork by timestamp. The existing V2 methods refuse an activated timestamp or a supplied root with the unsupported-fork error, and the new methods refuse a timestamp before activation. A call to the wrong `getPayload` method is refused before the payload is resolved, so the build keeps running and the correct method still returns it. Node startup and capability exchange must expose an incompatible client pairing before normal block production resumes.

### Finality consumption

The following Merkle generalized indices locate fields in the SSZ trees of Qrysm's `BeaconStateZond` and `BeaconBlockHeader`. They were derived from the schema at Qrysm commit [`4b66929d`](https://github.com/theQRL/qrysm/commit/4b66929ddec8e46a9053401533a8da29c70b9831) and are examples tied to that schema.

| Field                                                 | Generalized index |
| ----------------------------------------------------- | ----------------- |
| Beacon header `state_root`                            | 11                |
| Beacon state `finalized_checkpoint`                   | 52                |
| Beacon state finalized checkpoint epoch               | 104               |
| Beacon state finalized checkpoint root                | 105               |
| Beacon header to post-state finalized epoch, composed | 744               |
| Beacon header to post-state finalized root, composed  | 745               |
| Beacon state validator list                           | 43                |
| Beacon state balance list                             | 44                |

A staking consumer obtains a recent exported parent root, proves its post-state's finalized checkpoint epoch and root, verifies the exact header named by that root, then verifies state proofs against that header's `state_root`. The consumer enforces monotonic checkpoints and its own freshness bounds. Zero and genesis checkpoints need an explicit bootstrap policy.

A finalized checkpoint at epoch `E` may name an earlier occupied block if the epoch boundary slot was skipped. Consumers must authenticate the header that the checkpoint names and must accept a header whose slot is earlier than `E * slots_per_epoch`. The authenticated snapshot is that occupied block's post-state. Deriving a later empty-slot state requires additional transition evidence.

Proof suppliers remain responsible for providing every field needed by an application's accounting rules. Exporting a root does not by itself prove portfolio completeness, classify receipts, reconcile execution cutoffs or account for zero-balance terminal validators. Schema upgrades require new proof paths or a version-aware verifier.

## Rationale

QRL 2.0's clients descend from Ethereum's and followed their releases until December 2023, go-qrl up to go-ethereum 1.13.1 and Qrysm up to Prysm 4.1.1. Ethereum's Cancun and Deneb upgrade, which carries EIP-4788, was still unfinished then and went live in March 2024. The QRL clients kept the feature set that Ethereum had in production at that point and removed the unfinished code of the next upgrade in later cleanups, in go-qrl with [`ce7ef87c`](https://github.com/theQRL/go-qrl/commit/ce7ef87cb0415cd256634011791bb38bc7df48c4) in April 2024. This proposal brings the finished design in, adapted to QRL.

The parent root is already available before current-block execution. Exporting the current beacon block root would create a circular dependency with the execution result. Timestamp indexing makes the system independent of skipped beacon slots, provided consumers query by the write timestamp.

A dedicated history contract preserves ordinary `BLOCKHASH` semantics. Keeping roots at 32 bytes preserves current SSZ commitments. A separate proposal for recipient-authorized validator exits can use this primitive for contract-side discovery without making its exit handler depend on application proofs.

An alternative is direct export of a finalized root. That approach must specify which pre-state or post-state checkpoint is exported, its matching epoch, activation behavior and how both clients validate it. The parent-root primitive supports a broader set of proofs and keeps finality selection explicit in the consumer.

This proposal keeps EIP-4788's structure and departs from it in the following places.

| Topic            | Ethereum (EIP-4788)                                            | This proposal                                                   |
| ---------------- | -------------------------------------------------------------- | --------------------------------------------------------------- |
| Widths           | 32-byte words, 20-byte addresses                               | 64-byte words and addresses; roots stay 32 bytes                |
| Read input       | 32 bytes                                                       | One 64-byte word, limited to a nonzero `uint64`                 |
| Installation     | Deployment transaction from a single-use account               | Client installs the runtime at genesis or at activation         |
| Missing code     | Call fails silently                                            | Block is invalid                                                |
| Failed call      | Failure ignored by go-ethereum                                 | Block is invalid                                                |
| System gas       | 30,000,000                                                     | Bounded budget, experimentally 100,000                          |
| Engine transport | Cancun `forkchoiceUpdatedV3` and `newPayloadV3` carry the root | Three dedicated experimental methods; naming open               |
| History capacity | 8191 entries at 12-second slots, about 27 hours                | 8191 entries at 60-second slots, about 5.69 days; capacity open |

## Backward compatibility

This requires a coordinated consensus upgrade. Before activation, historical block hashes, header encodings and execution behavior must remain byte-for-byte compatible. Activated headers require new validation and Engine capability support. Old clients cannot validate the new rules.

Existing application verifiers are immutable or schema-specific in some deployments. New deployments or explicitly designed adapters may consume the primitive. Existing verifier behavior must not be assumed to change automatically.

## Reference Implementation

Both clients carry the proposal on public branches. The feature stays off unless an activation time is configured: `qrlBeaconRootsTime` in the execution chain configuration and `EXPERIMENTAL_BEACON_ROOT_TIME` in the beacon configuration.

| Component | Reviewed source | Review thread |
| --- | --- | --- |
| go-qrl | [`55a9167a`](https://github.com/DigitalGuards/go-qrl/commit/55a9167a96cfbc2c9a4786b74f1db34f08d65f80), on cyyber/go-qrl main [`d4f0d2ef`](https://github.com/cyyber/go-qrl/commit/d4f0d2effde840d4a2c59637d353ef2d72a1b6aa) | [DigitalGuards/go-qrl#1](https://github.com/DigitalGuards/go-qrl/pull/1) |
| Qrysm | [`c171191a`](https://github.com/DigitalGuards/qrysm/commit/c171191a53589ffcadf69e8a81eab22f7944d9b4), on theQRL/qrysm main [`4b66929d`](https://github.com/theQRL/qrysm/commit/4b66929ddec8e46a9053401533a8da29c70b9831) | [DigitalGuards/qrysm#1](https://github.com/DigitalGuards/qrysm/pull/1) |
| Local network package | [`64ad94e5`](https://github.com/DigitalGuards/qrl-package/commit/64ad94e53eb6636213015f57613354db14dede97) | [cyyber/qrl-package#23](https://github.com/cyyber/qrl-package/pull/23) |

On 18 November 2026 a local network built from these two revisions passed all 10 checks of its harness. The network ran one execution client, one beacon client and one validator client with 64 validators, six-second slots and eight-slot epochs on chain ID `3151978`, with the feature active from genesis. The checks were:

- Capability exchange advertises the three Engine methods.
- Genesis installs the runtime, carries a zero parent root and has no history entry.
- For canonical, execution-validated beacon blocks, the execution header's commitment equals the beacon block's `parent_root` and the history contract returns the same root for the block's timestamp. This held at slot 3 and at the finalized slot.
- Reads with an empty, zero or oversized timestamp revert, and so does a 32-byte write attempt from an ordinary account.
- The `pending` view keeps canonical entries readable.
- A Hyperion contract reads the current root inside a transaction, and transaction and block trace replay return the same root. The fixture transaction used 30,912 gas.
- History survives a graceful execution-client restart and the chain continues.
- Payload import accepts the correct parent root, rejects a wrong one as invalid and refuses the earlier import method after activation.
- The earlier `getPayload` method is refused for a root-bound build with error `-38005`, after which the new method still returns that build.
- The network finalized beacon slot 16 without optimistic status while the execution head reached block 31.

The validator proposed 42 blocks and missed one proposal, in the slot that followed the deliberate restart, because the restarted execution client had lost its cached payload.

Unit and integration tests pass in the affected go-qrl packages (22 packages, including the fork tests in `core`, `core/types`, `miner`, `params` and `qrl/catalyst`). In Qrysm the three smaller affected packages pass in full and the fork's tests pass in the two large ones. An earlier revision of this work first passed its network checks on 2 October 2026, on the clients of late September.

This validation has limits. It used a single node, so reorgs, multi-node synchronization and crash recovery are untested. Activation after genesis is covered by unit tests only. The gas cost of the write has not been benchmarked on validator hardware. No contract yet verifies a finalized-checkpoint proof on chain against an exported root.

## Security Considerations

- Validate the parent-root relationship in Qrysm and the corresponding header and state effects in go-qrl. Missing either validation permits self-consistent forged metadata.
- Require exact timestamps and lengths. Reject stale ring entries, oversized timestamp values and padded return data that a consumer could misinterpret.
- Preserve ordinary transaction accounting and access-state behavior around the system operation. Test state snapshots, nested execution and reorg rollback.
- Treat the root as authenticated relative to the canonical execution chain. An exported parent may still be unfinalized. External applications also need their ordinary transaction-finality policy.
- Check the finalized checkpoint and header as a linked proof. A valid branch beneath an unauthenticated root grants no authority.
- A verifier must compute every generalized index itself, from the schema and the position it queries, and must never accept an index from the caller. Lido's withdrawal verifier on Ethereum accepted a caller-supplied index for historical blocks. It became forgeable once execution requests let any account place chosen bytes in beacon blocks, and was fixed before any exploit ([post-mortem](https://research.lido.fi/t/security-disclosure-post-mortem-csverifier-weak-validation-of-the-historical-block-gindex-user-funds-remain-safe/10466)). The same care applies on QRL, in particular once execution requests exist.
- Bound system work and publish a verified runtime hash. Because a failed system call invalidates the block, a bad fork-installed runtime can halt every activated block.
- This primitive alone gives no transaction-tip enforcement, no instant exits and no complete staking accounting.

## Open review items

1. Reserved addresses for the history contract and the system caller, with a collision check on every network that activates after genesis.
2. History capacity `H`, weighed against QRL slot timing and the latency of proof suppliers.
3. The system gas budget, after measurement of the write on validator hardware.
4. The failure rule: invalid block on missing code or a failed call, where EIP-4788 fails silently.
5. Client-installed runtime or a deployment transaction, and the ratified runtime bytecode and hash.
6. Engine method names and versions, and the capability and startup behavior for mismatched clients.
7. The fork that carries activation, including the rule for a network that activates after genesis.
8. Whether a standard proof-verifier library for the finality path should accompany the primitive.

## References

- [EIP-4788](https://eips.ethereum.org/EIPS/eip-4788), "Beacon block root in the EVM", by Alex Stokes, Ansgar Dietrichs, Danny Ryan, Martin Holst Swende and lightclient. This proposal ports its design. It went live on Ethereum mainnet with the Cancun upgrade on 13 March 2024 ([EIP-7569](https://eips.ethereum.org/EIPS/eip-7569)).
- [EIP-7002](https://eips.ethereum.org/EIPS/eip-7002), "Execution layer triggerable withdrawals", by Danny Ryan, Mikhail Kalinin, Ansgar Dietrichs, Hsiao-Wei Wang, lightclient and Felix Lange. Source of the failed-system-call rule.
