	QIP: 002
	Layer: 2
	Title: MessageTransaction Encoded Message Standard
	Author: Scott Donald
	Comments-Summary: 
	Comments-URI: 
	Status: Open
	Type: Proposal
	Created: 03 November 2018

# A standard message encoding format to indicate encoded data in MessageTransaction transactions

## Abstract

The QRL network supports arbitrary messages up to 80 bytes in length to be stored on chain through the [MessageTransaction](https://github.com/theQRL/QRL/blob/v1.1.6/src/qrl/core/txs/MessageTransaction.py#L8) transaction subtype.

There is the capability for second layer clients to read and interpret the data contained within these message transactions, and format interfaces accordingly. This can be seen with the currently implemented `Document Notarisation` transaction type found on both the [QRL Wallet](https://github.com/theQRL/qrl-wallet/blob/v1.0.4/imports/ui/pages/tools/notarise/start.js#L71) and [QRL Explorer](https://github.com/theQRL/block-explorer/blob/2b11358f31415812bd374fb572c6ab9c8a06e9ad/imports/ui/components/tx/tx.html#L124) applications, and further implemented in the [explorer-helpers](https://github.com/theQRL/explorer-helpers/blob/v0.0.7/index.js#L356) repository.


## Motivation

This QIP aims to solidify the pre-existing message encoding standard such that other encoded messages have a base layer to build upon, such as the [coinvote](https://github.com/theQRL/qips/pull/2#issuecomment-434810654) proposal mentioned by @surg0r.

## Specification

The following will describe the base requirements to indicate a message contains encoded data, and provide further context on actual usage as has been implemented in the `Document Notarisation` transaction type.

There are a total of 80 bytes available in a MessageTransaction transaction that are usable. For the purposes of describing the format in this document, we will represent all data as HEX strings. It is worth noting however that the data is later converted to binary bytes for storage by a QRL node.

To indicate a message is an encoded message, the first two bytes are reserved for the following HEX string:

`AFAF`

The subsequent one byte should indicate a unique encoded message transaction type, that for [historical reasons](#backward-compatibility) should _not_ start with the HEX string `A`. Each proposal to the community for a new encoded message type will be allocated a unique HEX string for this byte for client implementations.

Eg: `00`, `B0` etc.

The remaining sixty-seven bytes contain any data relevant to the encoded message, and should be proposed to the community through a QIP. 

If this QIP is accepted, a record of accepted message transaction sub type and their respective QIP should be kept updated on (docs.theqrl.org)[https://github.com/theQRL/docs.theqrl.org] such that client implementations have the technical detail available to implement with ease.

#### Document Notarisation Specification

The following describes the structure of the `Document Notarisation` message transaction sub type for historical purposes.

First 2 Bytes: `AFAF` - Indicates encoded message.
Subsequent Half Byte: `A` - Indicates this is a document notarisation message transaction sub type.
Subsequent Half Byte: `1`, `2`, or `3` indicating which hash function has been used to notarise the document as per below:

    `1` - SHA1 (Now deprecated in user interfaces when notarising a document)
    `2` - SHA256
    `3` - MD5 (Now deprecated in user interfaces when notarising a document)

The remaining 77 bytes are reserved for both the hash of the document, and a variable amount of free form text depending which hash function was use. For each hash function above, the following describes the remaining 77 bytes utilisation.

    `1` - SHA1 - Subsequent 20 bytes store the SHA1 hash, with the remaining 57 bytes available for free form text.
    `2` - SHA256 - Subsequent 32 bytes store the SHA1 hash, with the remaining 45 bytes available for free form text.
    `3` - MD5 - Subsequent 16 bytes store the SHA1 hash, with the remaining 61 bytes available for free form text.

## Backward compatibility

The original `Document Notarisaion` transaction type contained only a half byte of information to indicate the transaction type the encoded message contained. This unfortunately was a design oversight that limited that maximum unique message transaction sub types to 16 globally. To ensure this transaction type continues to function within the context of this QIP, and that the message encoding standard can contain a larger number of transaction subtypes, an exception must be made to limit the ability for new transaction sub types to start with the hex string `A`.

## Implementation

No implementation work is required for this QIP as it simply states a standard for encoding messages using the QRL networks `MessageTransaction` transaction type.
