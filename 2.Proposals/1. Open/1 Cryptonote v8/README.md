	QIP: Migrate to Cryptonote V8
	Layer: Node
	Title: Migrate to Cryptonote v8
	Author: Andrew Chiw
	Comments-Summary: 
	Comments-URI: https://github.com/theQRL/qips/pull/2
	Status: Open
	Type: Mining
	Created: 2018-10-30
# QIP 1: Migrate to Cryptonote v8 (Cryptonote Variant 2)

## Abstract
This QIP explores the feasibility of migrating to Cryptonote v8, and continuing to change the POW algorithm just like Monero.

## Motivation
ASIC resistance is the most obvious reason, so that GPU miners can participate, thus making mining truly decentralized, instead of the select few who can afford to buy and setup ASICs.

## Implementation at library level
Currently, qryptonight implements Cryptonote v7 via hooking into a few important files in xmr-stak, and putting a SWIG wrapper around it.

Since xmr-stak 2.5.0 (931bd5fef17f908afc62836ae7b6ea087d1441ca, unify cpu cryptonight implementations), the internals have changed enough that qryptonight needs to be modified, and xmr-stak needs to be forked.

Once this is done, having qryptonight use Cryptonote v8 is as simple as changing this line:

```
Cryptonight_hash<1>::template hash<cryptonight_monero, false, false>(input.data(), input.size(), output.data(), &context);
```

to this:
```
Cryptonight_hash<1>::template hash<cryptonight_monero_v8, false, false>(input.data(), input.size(), output.data(), &context);
```

## Implementation at node level
This breaks the following tests:

```
TestChainManagerReal.test_add_block
TestChainManagerReal.test_multi_output_transaction_add_block
TestChainManagerReal.test_simple_add_block
TestBlockHeader.test_hash
TestBlockHeader.test_hash_nonce
TestQryptonight.test_hash
TestQryptonight.test_hash2
TestQryptonight.test_hash3
```

From these tests it appears that the code does not depend too much on the POW algorithm, which is good news.

## Backwards compatibility
Both algorithms will need to be present in qryptonight, and qryptonight must be modified to accept an extra argument that specifies which hashing algorithm should be used.

The best place for the node to decide which hashing algorithm should be used  is `BlockHeader`, specifically `generate_headerhash()` and `validate()`.

The exact block height from which Cryptonote v8 should be used instead of v7 should be defined in `config.dev`.

## Considerations
This work will have to be duplicated in go-qrl.

Should Monero's future POW algorithms require a different amount of memory than Cryptonight v7/v8, further modifications need to be made to qryptonight and xmr-stak in `cryptonote_alloc_ctx()`.
