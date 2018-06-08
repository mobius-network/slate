---
title: Mobius API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - shell: curl
  - javascript: Node
  - php: PHP
  - python: Python
  - jsx: React Native

toc_footers:
  - <a href='https://mobius.network/store/developer'>Sign Up for a Developer Key</a>
  - <a href='https://github.com/tripit/slate'>Documentation Powered by Slate</a>

search: true
---

# Mobius DApp Store

## Introduction

The Mobius DApp Store is an open-source, non-custodial platform that makes
payments in cryptocurrency to decentralized apps easy.

This new and innovative architecture empowers developers and entrepreneurs to
accept crypto-based payments in their apps and businesses.

Being non-custodial, Mobius never holds secret keys for users or developers. Any
tokens spent are directly sent between the user and the developer - *Mobius
never touches the money or takes any fees.*

A big advantage of the Mobius DApp Store over centralized competitors such as
the Apple App Store is significantly lower fees (around $0.000001, purely for
transacting on the Stellar network), compared to 30%, for in-app purchases.

## Stellar Network

Mobius DApp Store is built on top of the [Stellar
Network](https://www.stellar.org) blockchain.

Stellar has numerous advantages compared to other blockchains.

Transactions are cheap, close to $0 per transaction.

Average of 4 seconds for transaction confirmation, while the wait time in other
blockchains may be several hours when the network is under a heavy load (*ahem*
Ethereum).

One of the reasons why Stellar is so fast is its [Consensus
Protocol](https://www.stellar.org/developers/guides/concepts/scp.html). To learn
all about the protocol, read the [white
paper](https://www.stellar.org/papers/stellar-consensus-protocol.pdf).

This protocol relies on a Quorom notion where even just portion of the network
can agree on a transaction, which is enough to reach consensus. As a result, all
the network nodes get that same update to its ledger. Adding on top of that
notion, there are also Quorom slices, unique to Stellar, where a node can choose
which nodes he trusts for accurate information.

The Stellar Network allows for the creation of custom assets. Any type of asset
can be traded and exchanged.

> The Stellar distributed network can be used to track, hold, and transfer any
> type of asset: dollars, euros, bitcoin, stocks, gold, and other tokens of
> value. Any asset on the network can be traded and exchanged with any other.

[MOBI](https://stellarterm.com/#exchange/MOBI-mobius.network/XLM-native) is the
custom token issued on the Stellar Network. An asset is identified by issuer account and the asset code (MOBI).

Holding assets from an issuer is an agreement based on trust, or a
[trustline](https://www.stellar.org/developers/guides/concepts/assets.html#trustlines).

> " When you hold assets in Stellar, you’re actually holding credit from a
> particular issuer. The issuer has agreed that it will trade you its credit on
> the Stellar network for the corresponding asset–e.g., fiat currency, precious
> metal–outside of Stellar. Let’s say that Scott issues oranges as credit on the
> network. If you hold orange credits, you and Scott have an agreement based on
> trust, or a trustline: you both agree that when you give Scott an orange
> credit, he gives you an orange.
> 
> When you hold an asset, you must trust the issuer to properly redeem its
> credit. Since users of Stellar will not want to trust just any issuer,
> accounts must explicitly trust an issuing account before they’re able to hold
> the issuer’s credit. In the example above, you must explicitly trust Scott
> before you can hold orange credits.
> 
> To trust an issuing account, you create a trustline. Trustlines are entries
> that persist in the Stellar ledger. They track the limit for which your
> account trusts the issuing account and the amount of credit from the issuing
> account that your account currently holds."

# TODO - Trustline, assets, reorganize and add info

# Getting Started

## DApp Store Overview

The primary data structure of the Mobius DApp Store are Stellar
[Accounts](#accounts). Every user and [application](#application) require an
account. A user can login to the DApp Store with a Mobius Wallet, or by creating
one. To create a Mobius wallet, we give you a Mnemonic Phrase (24 words) for the
secret seed. 3 XLM is required to cover transaction and Mobius account creation
fees.

For every application used, an unique application specific account is created
where the user can deposit MOBI for use within the application. The
application's public key is added as a cosigner in order to access the MOBI. For
the first use, after creating the account, a challenge transaction is signed
from the application to authenticate the user.

# Concepts

## Accounts

Every Stellar account has a public key, a private key and a secret seed. The
public key starts with a G and the private key with a S. The private key is used
for authorizing transactions. Public key is used to get account information or
checking for transactions performed using the private key.

The Mnemonic Phrase is the secret seed. The seed is used to generate both the
public and private key for the account. The seed must be kept secret as it
single handedly provides full access to an account. In traditional public key
cryptography only a keypair is useful, where in Stellar a seed key provides all
the required access to an account.

Using [BIP 44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki) ,
Multi-Account Hierarchy for Deterministic Wallets, an infinite number of
accounts can be generated from a secret seed. The secret seed is used to create
the primary account, others are derived from base.

The user primary account (GUSER) will be the account at index 0. This is the
account that holds the user's MOBI, and XLM required to create derived accounts.

For every application the user accesses, the application ID will be used to
create an account (GUSER_DAPP). A user can deposit MOBI on those accounts to
make payments on the corresponding application.

Every application must have an account with a public key (GDAPP). An application
also holds the private key for this account where it receives MOBI (SDAPP).

Again the private key for the application (SDAPP) is never exposed to our
servers.

In order to receive payments, the application withdraws from the user account
(GUSER_DAPP) which is cosigned by the application account (GDAPP).

## Application

Applications can be submitted to the DApp Store on our website on the
[Developer](https://mobius.network/store/developer) page. See some of the
details and requirements below.

### Authentication Endpoint

Applications must implement an API endpoint which respond to authentication
requests and issue an authentication token. More information on
[authentication](#authentication).

On the **/auth** endpoint the following header must be set:

`Access-Control-Allow-Origin: *`

### Domain

Applications must be hosted on a separate, HTTPS-enabled domain.


## Authentication

When a user opens an app through the DApp Store it tells the app what Mobius
account it should use for payment.

The application needs to ensure that the user actually owns the secret key to
the Mobius account and that this isn't a replay attack from a user who captured
a previous request and is replaying it.

This authentication is accomplished through the following process:

- When the user opens an app in the DApp Store it requests a challenge from the
  application.
- The challenge is a payment transaction of 1 XLM from and to the application
  account. It is never sent to the network - it is just used for authentication.
- The application generates the challenge transaction on request, signs it with
  its own private key, and sends it to user.
- The user receives the challenge transaction and verifies it is signed by the
  application's secret key by checking it against the application's published
public key (that it receives through the DApp Store). Then the user signs the
transaction with its own private key and sends it back to application along with
its public key.
- Application checks that challenge transaction is now signed by itself and the
  public key that was passed in. Time bounds are also checked to make sure this
isn't a replay attack. If everything passes the server replies with a token the
application can pass in to "login" with the specified public key and use it for
payment (it would have previously given the app access to the public key by
adding the app's public key as a signer).

# TODO - Expand section
"10. Authentication: the application responsibility of managing auth token should
be mentioned explicitly. GWT also requires an explanation."


## Payments

After the user completes the authentication process they have a token. They now
pass it to the application to "login" which tells the application which Mobius
account to withdraw MOBI from (the user public key) when a payment is needed.
For a web application the token is generally passed in via a token request
parameter. Upon opening the website/loading the application it checks that the
token is valid (within time bounds etc) and the account in the token has added
the app as a cosigner so it can withdraw MOBI from it.

# TODO - Expand section

# Development

### Ruby

Ruby SDK is available on:

- [Github](https://github.com/mobius-network/mobius-client-ruby)
- [RubyGems](https://rubygems.org/gems/mobius-client)

### Javascript

Javascript SDK is available on:

- [Github](https://github.com/mobius-network/mobius-client-js)
- [npm](https://www.npmjs.com/package/@mobius-network/mobius-client-js)

### Example DApp

We have released a sample DApp that uses the Ruby SDK, [Floppy
Bird](https://github.com/mobius-network/floppy-bird-dapp).

### Testing Environment
