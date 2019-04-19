	QIP: 007
	Layer: 
	Title: Support Tokenization of Securities
	Author: IMac318
	Comments-Summary: 
	Comments-URI: 
	Status: 
	Type: 
	Created: 04/19/2019

{{TOC}}

# Support Tokenization of Securities

## Abstract

One of the biggest potential use cases of blockchains in the future is the tokenization of existing securities to be bought and sold by valid parties. Being the only existing blockchain that is quantum resistant from genesis and adding the functionality of smart contracts, QRL is in a unique spot to capitalize on this opportunity. The QRL team should evaluate whether the functionality of the smart contracts that are currently in progress can support the tokenization of securities on the QRL blockchain. If that is not the case, the team should consider adding additional functionality that allows for such tokenization.


## Motivation

The current usefulness of blockchain is highly debated, and there is much conversation about what "killer app" will eventually drive adoption. One current use of blockchain is speculation, and it is very attractive in this area due to the ability to buy fractions of a coin or token. In contrast, many traditional assets require the purchase of an integer number of the asset. For example, it is difficult to obtain a fraction of a stock or real estate property. 

Traditional assets, including stocks, real estate, and derivatives, are estimated to be worth a combined amount of over one quadrillion dollars (source: https://money.visualcapitalist.com/worlds-money-markets-one-visualization-2017/). There is incentive to tokenize existing securites and offer new securities on the blockchain, including the "removal of middlemen... to lower fees, faster deal execution, free market exposure, larger potential investor base, automated service functions, and lack of financial institution manipulation" (source: https://medium.com/@apompliano/the-official-guide-to-tokenized-securities-44e8342bb24f). In addition, security tokenization would raise the credibility of blockchain in the eyes of many traditional investors, as there would be "real value" in the underlying assets.

As of now, QRL is the only existing blockchain that is quantum resistance from genesis and implementing smart-contract functionality. Such functionality is necessary for the tokenization of securities, as the buying and selling of securities is restricted by KYC/AML laws. Given QRL's unmatched quantum resistance and future-oriented security, the QRL blockchain could be a very attractive platform for security tokenization.

The purpose of this QIP is to urge the QRL team to evaluate whether or not the in-progress smart-contract functionality can support the creation and exchange of security tokens. QRL smart contracts are limited in functionality compared to ETH (where security token standards are currently being created), so the author does not automatically assume that such functionality currently exists.  If it does not, consideration should be given to add such functionality.


## Specification

QRL already has the ability to create tokens, but to be compliant with regulations, it must be possible to restrict transactions of security tokens between wallets if the sending and receiving wallets do not meet certain criteria (example: https://thesecuritytokenstandard.org/); therefore, it would have to be possible to make these checks before a token transaction is admitted to the blockchain.


## Implementation

Several entities are working on platforms for tokenization of securities. Polymath, Harbor, Securrency, and tZero are prominent examples. It may be worth reaching out to them for evaluation and implementation.
