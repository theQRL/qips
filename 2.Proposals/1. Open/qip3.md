	QIP: 
	Layer: qrl-node, webwallet
	Title: Extended address functionality from master XMSS wallet
	Author: Peter Waterland, Kaushal Singh
	Comments-Summary: 
	Comments-URI: 
	Status: draft
	Type: improvement
	Created: 4/11/2018



# Title
Extended address functionality from master XMSS wallet

## abstract
A suggested change to generate multiple deterministic addresses which may be signed using the XMSS tree associated with a user master address.

## Motivation
Currently QRL uses XMSS to sign transactions and create a single re-usable address for users. Prior to signing a transaction the XMSS tree must be generated with varying computational cost dependent upon the signature capability of the address. Further signature capability may be added by signing slave XMSS trees to extend this scheme infinitely.

However, it would be valuable for users to have access to more than one address from which to send or receive QRL, without the prospect of adding significant computation time and worsening the user experience by generating additional XMSS trees. Given QRL is a public blockchain, such a facility would improve privacy for those not wishing to unnecessarily reveal their primary address balance and transaction history to a payor. i.e. a payor may be given an address to send funds to other than the user master address. Additionally, the easy creation of deterministic addresses allows privacy extension through deliberate usage of one of more change addresses to obfuscate user holdings.

## Specification

A 51 byte unique seed is used to generate a user wallet in the node or webwallet. The first three bytes are extended seed encoding with the following 48 bytes of random entropy providing the cryptographic seed used in PRF generation of the XMSS tree and subsequent master address.

In addition to the master XMSS tree, the 48 byte random seed may, in a cryptographically secure fashion, be used to generate a deterministic sequence of pseudorandom 32 byte values. This sequence can be trivially generated computationally yet retain the appearance of uniform random output when revealed to an external observer. Furthermore, the seed may not be determined using any or all of the PRF sequence values.

Example solution:

`prf_output = SHAKE128(SHA_256(SHA_256(SHA_256(seed[:32] ^ SHA_256(seed[32:])))))`

i.e. the first 32 bytes of the seed are xored with a hash of the last 16 bytes of the seed and the result is iteratively hashed three times then passed as input into SHAKE128, where output may be derived as a 256 bit pseudorandom deterministic number sequence.


Existing address format:

An existing standard address is created with the following format (pseudocode):

`QRL_address = encoding_bytes + hash_function(xmss_pk)`

where `encoding_bytes` includes the required hash function, signature scheme, xmss tree height etc

where `xmss_pk = master_xmss_tree_merkle_root + xmss_public_seed`


Suggested new extensible address creation:

Assuming a PRF sequence, `prf[n]`, an additional address may be simply created like:

`extended_address_n = encoding_bytes + hash_function(xmss_pk + prf[n])`

In this way, with each new value in the `prf` sequence a resultant new address is created for use.

Additionally, a user field such as the hash of a pin or seed phrase may be added to generate a different series of addresses.

i.e. `extended_address_n = encoding_bytes + hash_function(xmss_pk + prf[n] + hash_of_user_generated_data)`

For ease the address `encoding_bytes` must match the master address XMSS tree encoding to allow appropriate decoding during spend transaction processing.


## Change addresses

QRL uses re-usable addresses. An example transaction may move part of the funds from address `a` to `b`. After such a standard transfer transaction an outside observer can see precisely what funds remain in the user controlled master address `a`. The addition of multiple deterministic addresses to a master address means that the following is now possible: all funds may move from `a` to `b`, `c` and perhaps `d`. The balance of a is now zero, and an outside observer will not know that some funds were moved to address `b` and that the user controlling `a` actually controls the addresses `c` and `d`. Whilst the effective final balances of the two users in the transfer transaction remain the same -- to an outside observer the precise movements and control of funds are obfuscated.

## Backwards compatibility

Optional fields (`.extended_pk_address_prf` and `.extended_pk_address_user`) to be concatenated and hashed with pk must be added into the transaction logic in the base `Transaction` class. Minor state changes to associate extended addresses with existing master address pk's following a spend transaction (to preserve OTS safety). 

Higher level changes required in terms of tracking, displaying balances and allowing spend functionality from within the webwallet. This latter point will require internal discussion.





