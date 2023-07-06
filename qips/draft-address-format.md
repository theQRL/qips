---
qip: 0
title: Better format of wallet addresses
author: Robert Pösel (@robyer)
layer: core
status: "draft/incomplete"
comments_uri: ~
created: 2023-07-06
modified: 2023-07-06
---

## Abstract

The QRL wallet addresses are currently represented as 79 characters string. Such a long length has many disadvantages for its users. This QIP proposes use of more efficient encoding for address representation to make it shorter (64 characters) and user-friendly. It doesn't require any changes to the internal wallet format, therefore it doesn't affect internal APIs or security, and backwards compatibility is easily maintained.

Note: Ideally, same change to encoding should be applied to the Transaction IDs as well (separate QIP can be created for that).

## Motivation

There are many inconveniences associated with long addresses, for example:

- Visual inspection of the address
   - When a person creates an outgoing transaction, they must carefully check that they are sending funds to the correct address. But the longer the address, the more annoying this activity is. User either completely neglects the check, or checks only the last few characters (which can be abused by bad actors), or generally - the probability of making a mistake during the check increases.
- Manual writing of address
   - Here are the same problems as with visual inspection, plus it's more annoying to manually enter the address somewhere. It is not always possible to rely on QR codes and their scanning, and sometimes it is simply necessary to write it down manually - whether on a computer/phone, or perhaps on paper.
- Bloated user interface in applications
   - Wherever a long address is used, a large amount of space must be reserved for it in the user interface. Either to show an input field for destination address, or to show user's address to copy it, or show addresses in a list of historical transactions, in address book, in block explorer, etc.
   - It's even worse on HW wallets, as they have small display sizes and user needs to go through multiple screens to read the whole address.
- Less friendly when compared to addresses of other cryptocurrencies
   - People like to show their crypto addresses on the web, in their profiles, in a forum signatures or similar places when they want to receive donations from others. Not only do longer addresses take up more space (see the previous point), but in addition, when they are compared or placed next to (for example) a BTC address (34 or 42 or 62 characters) or an ETH address (42 characters), QRL addresses (79 characters) look more technical and less friendly.

It's important to note that length is not the only aspect that must be considered here, but it is the main problem of current format. The addresses should be shorter AND user-friendly.

## Specification

QRL currently uses wallet addresses starting with `Q` followed by 78 hexadecimal characters (`0-9a-f` - which is basically **Base16** encoding).

Proposed format is keeping the `q` (but lower case) followed by 33 characters in **z-base-32** encoding, which is more efficient and user-friendly (see the design considerations in the encoding specification).

For example original addresses (79 characters):
1. `Q01070050d31c7f123995f097bc98209e9231d663dc26e06085df55dc2f6afe3c2cd62e8271a6bd`
2. `Q0109003493192e08affe87d57e254df3e15be3b8709a40f07e0fc550de60696c2d0333f7070e1d`

Would look like this (64 characters):
1. `qyrdoywgudt9trqci6nm53gbyu4jddiud5ouqyarf57k7am5k9a6n3itqoja4pxe`
2. `qyrroyprudrzytm96o9kzhjkp6xoiza7aqnprbhd6b9nibzuypfsn4y3u6hdoh8e`

To summary, change is only in the representation of the address (described in [official docs](https://docs.theqrl.org/developers/address/)) to use (`q` + z-base-32 encoding) instead of (`Q` + base16 encoding).

### z-base-32
The **[z-base-32](http://philzimmermann.com/docs/human-oriented-base-32-encoding.txt)** was created in 2002 as **human-oriented base-32 encoding**.

Citation from Wikipedia:
> z-base-32 is a Base32 encoding designed by Zooko Wilcox-O'Hearn to be easier for human use and more compact. It includes 1, 8 and 9 but excludes l, v and 2. It also permutes the alphabet so that the easier characters are the ones that occur more frequently. It compactly encodes bitstrings whose length in bits is not a multiple of 8 and omits trailing padding characters. z-base-32 was used in the Mnet open source project, and is currently used in Phil Zimmermann's ZRTP protocol, and in the Tahoe-LAFS open source project.

The full specification is [here](http://philzimmermann.com/docs/human-oriented-base-32-encoding.txt).

There are existing libraries for commonly used programming languages:
- Go - https://pkg.go.dev/gopkg.in/corvus-ch/zbase32.v1
- Python - https://pypi.org/project/python-zbase32/
- JavaScript - https://www.npmjs.com/package/zbase32
- and more...

There are also online convertors, like [here](https://cryptii.com/pipes/base32) (you must switch the variant to z-base-32) or [here](https://www.dcode.fr/z-base-32-encoding).

## Rationale

Representing binary data to be human-readable is a common task and there are many different approaches which were created during the years. Wikipedia has a [list of many of them](https://en.wikipedia.org/wiki/Binary-to-text_encoding). Creating new custom encoding format doesn't make much sense, which leaves us to one of the existing ones.

For example original Bitcoin addresses used Base58 encoding ([original reasoning](https://en.wikipedia.org/wiki/Binary-to-text_encoding#/media/File:Original_source_code_bitcoin-version-0.1.0_file_base58.h.png)), but here is a problem that it uses both lower and upper-case characters, which is not user-friendly. And Bitcoin switched to a fully-lowercase format anyway.

Ethereum (and QRL currently) uses simply hexadecimal encoding, which is unnecessary long and also less pleasant, as majority of the characters are numbers, which has full line height as capital letters and feels overwhelming and too technical.

Other Base32 encodings has been considered for its nice balance between its length and user-friendliness, but the [design decisions](http://philzimmermann.com/docs/human-oriented-base-32-encoding.txt) of the z-base-32 regarding human errors makes most sense, and because it's fully lower-case it makes it more pleasant (as opposite to fully upper-case Crockford's Base32 encoding).

## Backward compatibility

Since this is just a visual change, old (current) format of QRL addresses will be still compatible, because it is being converted to different format anyway (same as the new proposed one), which is then used internally in services, APIs, etc.

To provide the backward compatibility for user-facing forms, current software and services should allow user to input addresses in both new and old format. But the GUI should show only addresses in new format, everywhere. Additionally there could be created a simple conversion tool on the official website (and/or in official wallet apps) which would provide conversion from old to new format (and vice-versa) in case user needs to work with some old service or previously written address in old format.

## Security Considerations

There are no security implications related to this change.
