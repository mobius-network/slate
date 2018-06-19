---
title: Mobius API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - shell: curl
  - javascript: Node
  - php: PHP
  - python: Python
  - jsx: React Native

toc_footers:
  - <a href='https://mobius.network/store/developer'>Sign Up for a Developer
    Key</a>
  - <a href='https://github.com/tripit/slate'>Documentation Powered by Slate</a>

search: true
---

# Topics

## Introduction

Official libraries for the Mobius DApp Store are available in Ruby, Javascript,
PHP and Unity.

Ruby SDK is available on:

- [Github](https://github.com/mobius-network/mobius-client-ruby)
- [RubyGems](https://rubygems.org/gems/mobius-client)

Javascript SDK is available on:

- [Github](https://github.com/mobius-network/mobius-client-js)
- [npm](https://www.npmjs.com/package/@mobius-network/mobius-client-js)

We have released a sample DApp that uses the Ruby SDK, [Floppy
Bird](https://github.com/mobius-network/floppy-bird-dapp).

## Accounts

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

Applications must implement an API endpoint which respond to authentication
requests and issue an authentication token. More information on
[authentication](#authentication).

On the **/auth** endpoint the following header must be set:

`Access-Control-Allow-Origin: *`

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


## Payments

After the user completes the authentication process they have a token. They now
pass it to the application to "login" which tells the application which Mobius
account to withdraw MOBI from (the user public key) when a payment is needed.
For a web application the token is generally passed in via a token request
parameter. Upon opening the website/loading the application it checks that the
token is valid (within time bounds etc) and the account in the token has added
the app as a cosigner so it can withdraw MOBI from it.

## Errors

Error Types:
- AccountMissing
- AuthorisationMissing
- InsufficientFunds
- MalformedTransaction
- TokenExpired
- TokenTooOld
- TrustlineMissing
- Unauthorized
- UnknownKeyPairType

# Resources

## App

## Auth

## Blockchain

## FriendBot

# CLI
