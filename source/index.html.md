---
title: Mobius API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - ruby: Ruby
  - javascript: Javascript
  - php: PHP

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

```
# TODO
Infograph:

1.
Generates keypair from SUSER
var keypair = StellarSdk.Keypair.fromSecret(seed);

2.
GET /auth
Start a new transaction from returned xdr
var tx = new StellarSdk.Transaction(xdr);
tx.sign(keypair);
var signedChallenge = tx.toEnvelope().toXDR("base64");

3.
POST /auth, params: signedChallenge and keypair.publicKey()
Redirect to /?token=response.data

// Mobius Store Responsability
suser = "SCRBVUQF4N7JNXSEVQKZ4OEOEORC54BURPI3RCKN4FP3CO6AA7EZTJOA"
var keypair = StellarSdk.Keypair.fromSecret(suser);

axios.get(endpoint).then(function(response) {
  var xdr = response.data; // Response from GET /auth
  var tx = new StellarSdk.Transaction(xdr);
  tx.sign(keypair);
  var signedChallenge = tx.toEnvelope().toXDR("base64");
# =>
"AAAAABSMDrl3GEwb3T0yMpgZRv/F7OluSV2il8hXxbAySL6fAAAAZP////////YHAAAAAQAAAABbLmrXAAAAAFsvvFcAAAABAAAAFU1vYml1cyBhdXRoZW50aWNhdGlvbgAAAAAAAAEAAAABAAAAACQ98LCW694lXtYrwD1snbvsW7ihc2DUQofx2MiQ89a+AAAAAQAAAAAkPfCwluveJV7WK8A9bJ277Fu4oXNg1EKH8djIkPPWvgAAAAAAAAAAAJiWgAAAAAAAAAACkPPWvgAAAED4jPipA+5NCVLhuSMn2u8LwepdeJ+3/mcUCmtHsU2viPKudeVm/MV2Jq9VkD9aPlo+cXQqCKewWSag+0sHITAOHmQt2QAAAEAs7+u2uQewEtp4H7MgPplMHhMgNyfkT8zVwp8VJyP1OCTZ2mMaRlWMgIHThPQyqx2wL28EQRSntvHFjlTmvJQC"
  axios({
    url: endpoint,
    method: 'post',
    params: {
      xdr: signedChallenge,
      public_key: keypair.publicKey()
    }
  }).then(function(response) {
    var url = $('#redirect_url').val();
    document.location = url + '?token=' + response.data; // Response from POST /auth
  });
});


# GET /auth
Mobius::Client::Auth::Challenge.call(Rails.application.secrets.app[:secret_key],
12.hours)
# =>
"AAAAABSMDrl3GEwb3T0yMpgZRv/F7OluSV2il8hXxbAySL6fAAAAZP////////YHAAAAAQAAAABbLmrXAAAAAFsvvFcAAAABAAAAFU1vYml1cyBhdXRoZW50aWNhdGlvbgAAAAAAAAEAAAABAAAAACQ98LCW694lXtYrwD1snbvsW7ihc2DUQofx2MiQ89a+AAAAAQAAAAAkPfCwluveJV7WK8A9bJ277Fu4oXNg1EKH8djIkPPWvgAAAAAAAAAAAJiWgAAAAAAAAAABkPPWvgAAAED4jPipA+5NCVLhuSMn2u8LwepdeJ+3/mcUCmtHsU2viPKudeVm/MV2Jq9VkD9aPlo+cXQqCKewWSag+0sHITAO"

# POST /auth
Mobius::Client::Auth::Token.new(Rails.application.secrets.app[:secret_key],
"AAAAABSMDrl3
GEwb3T0yMpgZRv/F7OluSV2il8hXxbAySL6fAAAAZP////////YHAAAAAQAAAABbLmrXAAAAAFsvvFcAAAABAAAAFU1vYml1cyBhdXRoZW
50aWNhdGlvbgAAAAAAAAEAAAABAAAAACQ98LCW694lXtYrwD1snbvsW7ihc2DUQofx2MiQ89a+AAAAAQAAAAAkPfCwluveJV7WK8A9bJ27
7Fu4oXNg1EKH8djIkPPWvgAAAAAAAAAAAJiWgAAAAAAAAAACkPPWvgAAAED4jPipA+5NCVLhuSMn2u8LwepdeJ+3/mcUCmtHsU2viPKude
Vm/MV2Jq9VkD9aPlo+cXQqCKewWSag+0sHITAOHmQt2QAAAEAs7+u2uQewEtp4H7MgPplMHhMgNyfkT8zVwp8VJyP1OCTZ2mMaRlWMgIHT
hPQyqx2wL28EQRSntvHFjlTmvJQC",
"GDPNZJR42G4PVA3235JHCGGTU2J54X4XGJIMQGDJXYGHQDY6MQW5SZ5Y")

# =>
#<Mobius::Client::Auth::Token:0x00007f1628939618
@seed="SBBT7NRAKW43LN6ZGSZI37HEJDYWOC62LIZUL35KHO6NI2SKJLEPVAY4",
@xdr="AAAAACBi6BTSJUodD7akxyqmE2X6CdrCDbDbRfpJ23mYLCAKAAAAZP///////8cNAAAAAQAAAABbLm/PAAAAAFsvwU8AAAABAAAAFU1vYml1cyBhdXRoZW50aWNhdGlvbgAAAAAAAAEAAAABAAAAACQ98LCW694lXtYrwD1snbvsW7ihc2DUQofx2MiQ89a+AAAAAQAAAAAkPfCwluveJV7WK8A9bJ277Fu4oXNg1EKH8djIkPPWvgAAAAAAAAAAAJiWgAAAAAAAAAACkPPWvgAAAEDLI5hubbylPbMMcTgef4+Ylz20ZD1SVcZNuu+evdIr9B/WylJd/vJLcjv8xU5Jny1utwZ9ci5Xh5lh6hKvtq8PHmQt2QAAAEBVSeZj0QDAobtcLcy7ETC24jo/uM82s7ctZ8xc9cU8fEdZV9d9KKnVJdzHx7Axiq3rTuiVjByXjt65oyXZancP",
@address="GDPNZJR42G4PVA3235JHCGGTU2J54X4XGJIMQGDJXYGHQDY6MQW5SZ5Y">

token.validate!
# => true

Mobius::Client::Auth::Jwt.new(Rails.application.secrets.app[:jwt_secret]).encode(token)

# =>
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzUxMiJ9.eyJoYXNoIjoiZmM0YTE2YzJlOTM3YjUwMzc3ZTAyNzRjY2IzYTQ5NTExNGQ4NDk0YTMxODY5MTZkMTQyZDg4Y2RiNzBhZDg0YyIsInB1YmxpY19rZXkiOiJHRFBOWkpSNDJHNFBWQTMyMzVKSENHR1RVMko1NFg0WEdKSU1RR0RKWFlHSFFEWTZNUVc1U1o1WSIsIm1pbl90aW1lIjoxNTI5NzY5OTM1LCJtYXhfdGltZSI6MTUyOTg1NjMzNX0.rgPA3LxcRnZz0BJymlAGGaqC8FQZCRm_nVhREhkRNvdboCAWrHvUZWulqc5n5IERCpsIlAjcS72pe5amgrXdkw
```


## Payments

After the user completes the authentication process they have a token. They now
pass it to the application to "login" which tells the application which Mobius
account to withdraw MOBI from (the user public key) when a payment is needed.
For a web application the token is generally passed in via a token request
parameter. Upon opening the website/loading the application it checks that the
token is valid (within time bounds etc) and the account in the token has added
the app as a cosigner so it can withdraw MOBI from it.

# App

```ruby
# App private key
sdapp = "SBBT7NRAKW43LN6ZGSZI37HEJDYWOC62LIZUL35KHO6NI2SKJLEPVAY4"

# User public key
guser = "GDPNZJR42G4PVA3235JHCGGTU2J54X4XGJIMQGDJXYGHQDY6MQW5SZ5Y"

app = Mobius::Client::App.new(sdapp, guser)
# =>
#<Mobius::Client::App:0x00007fe654093bd8
  @seed="SBBT7NRAKW43LN6ZGSZI37HEJDYWOC62LIZUL35KHO6NI2SKJLEPVAY4",
  @address="GDPNZJR42G4PVA3235JHCGGTU2J54X4XGJIMQGDJXYGHQDY6MQW5SZ5Y">

app.balance
# => 990.0

app.app_balance
# => 10.0

app.authorized?
# => true

app.pay(5)
# =>
#<Hyperclient::Resource self_link:nil attributes:
  #<Hyperclient::Attributes:0x00007fe65480ba50
    @collection={
      "hash"=>"821c60b0a0205c3a208370533e3ecfe90193f3677a33c700c8e593a6205fc09f",
      "ledger"=>9634000,
      "envelope_xdr"=>"AAAAAN7cpjzRuPqDet9ScRjTppPeX5cyUMgYab4MeA8eZC3ZAAAAZACS/5EAAAAFAAAAAAAAAAAAAAABAAAAAAAAAAEAAAAAJD3wsJbr3iVe1ivAPWydu+xbuKFzYNRCh/HYyJDz1r4AAAABTU9CSQAAAADjYK00jeimcjYeRzelt/XDgrtvkTh+GfRfj+iKkKqnIQAAAAAC+vCAAAAAAAAAAAGQ89a+AAAAQM3QmVXHbjbPMCealSg5fSSpXz8hHGwRREQ7hEuywqUp2yZP9lfH5uVbzkCWm7s8/325eCy9WM66m0SEw8eqaQU=",
      "result_xdr"=>"AAAAAAAAAGQAAAAAAAAAAQAAAAAAAAABAAAAAAAAAAA=",
      "result_meta_xdr"=>"AAAAAAAAAAEAAAAEAAAAAwCS/74AAAABAAAAACQ98LCW694lXtYrwD1snbvsW7ihc2DUQofx2MiQ89a+AAAAAU1PQkkAAAAA42CtNI3opnI2Hkc3pbf1w4K7b5E4fhn0X4/oipCqpyEAAAAABfXhAH//////tyCAAAAAAQAAAAAAAAAAAAAAAQCTANAAAAABAAAAACQ98LCW694lXtYrwD1snbvsW7ihc2DUQofx2MiQ89a+AAAAAU1PQkkAAAAA42CtNI3opnI2Hkc3pbf1w4K7b5E4fhn0X4/oipCqpyEAAAAACPDRgH//////tyCAAAAAAQAAAAAAAAAAAAAAAwCS/74AAAABAAAAAN7cpjzRuPqDet9ScRjTppPeX5cyUMgYab4MeA8eZC3ZAAAAAU1PQkkAAAAA42CtNI3opnI2Hkc3pbf1w4K7b5E4fhn0X4/oipCqpyEAAAACThYDAH//////tyCAAAAAAQAAAAAAAAAAAAAAAQCTANAAAAABAAAAAN7cpjzRuPqDet9ScRjTppPeX5cyUMgYab4MeA8eZC3ZAAAAAU1PQkkAAAAA42CtNI3opnI2Hkc3pbf1w4K7b5E4fhn0X4/oipCqpyEAAAACSxsSgH//////tyCAAAAAAQAAAAAAAAAA"
    }>>

app.balance
# => 985.0

app.app_balance
# => 15.0

app.transfer(2, guser)
# =>
#<Hyperclient::Resource self_link:nil attributes:
  #<Hyperclient::Attributes:0x00007fe65460df28
    @collection={
      "hash"=>"ca95d24cd78d8a63ccedcdf0104d414a4bd179bd95c08ed85dd15aa1703ac2d1",
      "ledger"=>9634014,
      "envelope_xdr"=>"AAAAAN7cpjzRuPqDet9ScRjTppPeX5cyUMgYab4MeA8eZC3ZAAAAZACS/5EAAAAGAAAAAAAAAAAAAAABAAAAAAAAAAEAAAAA3tymPNG4+oN631JxGNOmk95flzJQyBhpvgx4Dx5kLdkAAAABTU9CSQAAAADjYK00jeimcjYeRzelt/XDgrtvkTh+GfRfj+iKkKqnIQAAAAABMS0AAAAAAAAAAAGQ89a+AAAAQN+xhjZFQAMa54qO/YfNGpKIx8dOuFsM0yr5HWPzweojvfy3kItuIjzZ8tEcPaBiDFWddfX4JwLjULgJqVCIgQU=",
      "result_xdr"=>"AAAAAAAAAGQAAAAAAAAAAQAAAAAAAAABAAAAAAAAAAA=",
      "result_meta_xdr"=>"AAAAAAAAAAEAAAACAAAAAwCTANAAAAABAAAAAN7cpjzRuPqDet9ScRjTppPeX5cyUMgYab4MeA8eZC3ZAAAAAU1PQkkAAAAA42CtNI3opnI2Hkc3pbf1w4K7b5E4fhn0X4/oipCqpyEAAAACSxsSgH//////tyCAAAAAAQAAAAAAAAAAAAAAAQCTAN4AAAABAAAAAN7cpjzRuPqDet9ScRjTppPeX5cyUMgYab4MeA8eZC3ZAAAAAU1PQkkAAAAA42CtNI3opnI2Hkc3pbf1w4K7b5E4fhn0X4/oipCqpyEAAAACSxsSgH//////tyCAAAAAAQAAAAAAAAAA"
    }>>

app.balance
# => 985.0

app.app_balance
# => 15.0
```

Interface to user balance in application.

Attribute | Type | Description
--------- | ---- | -----------
seed | String | DAPP private key.
address | String | Users public key.

### Instance Methods

`#app_balance ⇒ Float`

Returns application balance.

`#balance ⇒ Float`

Returns user balance.

`#authorized? ⇒ Bool`

Checks if user is authorized to use an application.

`#pay(amount, target_address: nil) ⇒ Object`

Makes Payment.

`#transfer(amount, address) ⇒ Object`

Sends money from application account to third party.

# Auth

## Challenge

```ruby
# You can use either the class method shortcut or the instance method.

sdapp = "SBBT7NRAKW43LN6ZGSZI37HEJDYWOC62LIZUL35KHO6NI2SKJLEPVAY4"

challenge = Mobius::Client::Auth::Challenge.call(sdapp, 12.hours)

# =>
"AAAAALqSu1LVgfxgg1XbIrfNevKXDVWULPKcwC9LBl7tZSOxAAAAZP///////zcoAAAAAQAAAABbLSfMAAAAAFsueUwAAAABAAAAFU1vYml1cyBhdXRoZW50aWNhdGlvbgAAAAAAAAEAAAABAAAAACQ98LCW694lXtYrwD1snbvsW7ihc2DUQofx2MiQ89a+AAAAAQAAAAAkPfCwluveJV7WK8A9bJ277Fu4oXNg1EKH8djIkPPWvgAAAAAAAAAAAJiWgAAAAAAAAAABkPPWvgAAAECnyBQ89V53lImPC04ULlENTw+C0eqDjpEu2P4igCYNLxAl58HZU7pojZpE8+REAMhiUAHAqFMkEGYDuuEnB3MF"

challenge = Mobius::Client::Auth::Challenge.new(sdapp)

# =>
#<Mobius::Client::Auth::Challenge:0x00007fe6143ab8d8
  @seed="SBBT7NRAKW43LN6ZGSZI37HEJDYWOC62LIZUL35KHO6NI2SKJLEPVAY4">

challenge.call(12.hours)

# =>
"AAAAAHHVw2RLf63A6FCtixvJ9rvGjuVruiF2SWQxIl/shKnQAAAAZP///////6S0AAAAAQAAAABbLSmPAAAAAFst0k8AAAABAAAAFU1vYml1cyBhdXRoZW50aWNhdGlvbgAAAAAAAAEAAAABAAAAACQ98LCW694lXtYrwD1snbvsW7ihc2DUQofx2MiQ89a+AAAAAQAAAAAkPfCwluveJV7WK8A9bJ277Fu4oXNg1EKH8djIkPPWvgAAAAAAAAAAAJiWgAAAAAAAAAABkPPWvgAAAECton56q0lqrPhOEF/jT7prBV7wfjxkvSgfKm4jyG8HdihCVFDutdvpsd10PJUPdIuLAxmpdsycLYhE/kOp1j4C"
```

Generates challenge transaction on developer's side.

Attribute | Type | Description
----------|------|------------
seed | String | DAPP private key.

### Class Methods

`.call(seed, expire_in = Mobius::Client.challenge_expires_in) ⇒ Object`

Constructor shortcut to generate challenge.

### Instance Methods

`#call(expire_in = Mobius::Client.challenge_expires_in) ⇒ Object`

`challenge_expires_in # => Equals to 1d by default`

Generates challenge transaction (1 XLM) signed by the SDAPP. This transaction is
never sent to the network.

## Jwt

```ruby
# Secret Key
jwt_secret = "431714aa54beec753975eaffba3db12d43d4ee52cafeb3ddcdbea05903e3a3ee78ff1f49d56b23df165 97bc15f6d6099aef2f668aa38f957ffc960a5445aa8fb"

# Generates a new token object, to validate challenge transaction.
token = Mobius::Client::Auth::Token.new(
  Rails.application.secrets.app[:secret_key], # SA2VTRSZPZ5FIC.....I4QD7LBWUUIK
  params[:xdr],                               # Challenge transaction
  params[:public_key]                         # User's public key
)


# Validates challenge transaction. It must be:
#   - Signed by application and requesting user.
#   - Not older than 10 seconds from now (see Mobius::Client.strict_interval`)
token.validate!
# => true

# Converts issued token into JWT and returns, this will be the token used by the user in query param.
jwt = Mobius::Client::Auth::Jwt.new(jwt_secret).encode(token)
# =>
"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzUxMiJ9.eyJoYXNoIjoiNjZkZGQwYzRmNTgwOGRhZjExYjUwYWIzOTQ3NmJiYjE0NDdkYzc5ZDM5NGRjODBlNDQyMjFlODlmNjVlOTQ1MiIsInB1YmxpY19rZXkiOiJHRFBOWkpSNDJHNFBWQTMyMzVKSENHR1RVMko1NFg0WEdKSU1RR0RKWFlHSFFEWTZNUVc1U1o1WSIsIm1pbl90aW1lIjoxNTI5NjgxNjExLCJtYXhfdGltZSI6MTUyOTc2ODAxMX0.Ie2VXa3DqCV5q67dG5Cc8SlPVsYQFNLEBvSreFaIa4ONJAiR8WRRfP3VkiN8DTwVJVxJ1ePg5_gXTdZv04sqMg"

# Token sent in query param, on a request
token_s = "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzUxMiJ9.eyJoYXNoIjoiNjZkZGQwYzRmNTgwOGRhZjExYjUwYWIzOTQ3NmJiYjE0NDdkYzc5ZDM5NGRjODBlNDQyMjFlODlmNjVlOTQ1MiIsInB1YmxpY19rZXkiOiJHRFBOWkpSNDJHNFBWQTMyMzVKSENHR1RVMko1NFg0WEdKSU1RR0RKWFlHSFFEWTZNUVc1U1o1WSIsIm1pbl90aW1lIjoxNTI5NjgxNjExLCJtYXhfdGltZSI6MTUyOTc2ODAxMX0.Ie2VXa3DqCV5q67dG5Cc8SlPVsYQFNLEBvSreFaIa4ONJAiR8WRRfP3VkiN8DTwVJVxJ1ePg5_gXTdZv04sqMg"

# Decodes JWT into token to identify user_dapp account and to initialize the @app object.
@token ||= Mobius::Client::Auth::Jwt.new(Rails.application.secrets.app[:jwt_secret]).decode!(token_s)
# =>
#<OpenStruct
hash="66ddd0c4f5808daf11b50ab39476bbb1447dc79d394dc80e44221e89f65e9452",
public_key="GDPNZJR42G4PVA3235JHCGGTU2J54X4XGJIMQGDJXYGHQDY6MQW5SZ5Y",
min_time=1529681611, max_time=1529768011>
```

Generates JWT (short for Json Web Token) based on valid token transaction signed by both parties and decodes JWT into hash.

Attribute | Type | Description
----------|------|------------
secret | String | JWT Secret

### Constants

`ALG = "HS512".freeze`

Used JWT algorithm.

### Instance Methods

`#encode(token) ⇒ String`

Returns decoded JWT.

`#decode!(jwt) ⇒ Hash`

Returns JWT.

## Sign

```ruby
seed = "SCRBVUQF4N7JNXSEVQKZ4OEOEORC54BURPI3RCKN4FP3CO6AA7EZTJOA"
# =>
"SCRBVUQF4N7JNXSEVQKZ4OEOEORC54BURPI3RCKN4FP3CO6AA7EZTJOA"

xdr = "AAAAAAKkwpEYoh4WhghDAaVE1PAxldny4jBw8nLw/hX17WSaAAAAZP///////x7wAAAAAQAAAABbMOQWAAAAAFsyNZYAAAABAAAAFU1vYml1cyBhdXRoZW50aWNhdGlvbgAAAAAAAAEAAAABAAAAACQ98LCW694lXtYrwD1snbvsW7ihc2DUQofx2MiQ89a+AAAAAQAAAAAkPfCwluveJV7WK8A9bJ277Fu4oXNg1EKH8djIkPPWvgAAAAAAAAAAAJiWgAAAAAAAAAABkPPWvgAAAEArkIA+SpNLPZd7g1PO3CD858zSl9UyjENEkfOzB4iggouV71JwQH/QiBIwDzCd8LS7l3Ay6Jz7zq7/NHDbZlkH"
# =>
"AAAAAAKkwpEYoh4WhghDAaVE1PAxldny4jBw8nLw/hX17WSaAAAAZP///////x7wAAAAAQAAAABbMOQWAAAAAFsyNZYAAAABAAAAFU1vYml1cyBhdXRoZW50aWNhdGlvbgAAAAAAAAEAAAABAAAAACQ98LCW694lXtYrwD1snbvsW7ihc2DUQofx2MiQ89a+AAAAAQAAAAAkPfCwluveJV7WK8A9bJ277Fu4oXNg1EKH8djIkPPWvgAAAAAAAAAAAJiWgAAAAAAAAAABkPPWvgAAAEArkIA+SpNLPZd7g1PO3CD858zSl9UyjENEkfOzB4iggouV71JwQH/QiBIwDzCd8LS7l3Ay6Jz7zq7/NHDbZlkH"

address = "GASD34FQS3V54JK62YV4APLMTW56YW5YUFZWBVCCQ7Y5RSEQ6PLL5COZ"
# =>
"GASD34FQS3V54JK62YV4APLMTW56YW5YUFZWBVCCQ7Y5RSEQ6PLL5COZ"

sign = Mobius::Client::Auth::Sign.call(seed, xdr, address)
# =>
"AAAAAAKkwpEYoh4WhghDAaVE1PAxldny4jBw8nLw/hX17WSaAAAAZP///////x7wAAAAAQAAAABbMOQWAAAAAFsyNZYAAAABAAAAFU1vYml1cyBhdXRoZW50aWNhdGlvbgAAAAAAAAEAAAABAAAAACQ98LCW694lXtYrwD1snbvsW7ihc2DUQofx2MiQ89a+AAAAAQAAAAAkPfCwluveJV7WK8A9bJ277Fu4oXNg1EKH8djIkPPWvgAAAAAAAAAAAJiWgAAAAAAAAAACkPPWvgAAAEArkIA+SpNLPZd7g1PO3CD858zSl9UyjENEkfOzB4iggouV71JwQH/QiBIwDzCd8LS7l3Ay6Jz7zq7/NHDbZlkHHmQt2QAAAEBfDFgsjXzRcskEwgkGW9iiHJZX7hMS3/Vf+FOPkLuitD6mk9QlL/IV5q/G61YCA4YqmQpIpf8UUWsioiw1zoIN"

```

Signs challenge transaction on user's side.

Attribute | Type | Description
----------|------|------------
seed | String | Users private key
xdr | String | Challenge Transaction xdr
address | String | Developers public key

### Class Methods

`.call(seed, xdr, address) ⇒ [String] Base64-encoded transaction envelope`

Constructor shortcut to call instance method.

### Instance Methods

`#call(seed, xdr, address) ⇒ [String] Base64-encoded transaction envelope`

Adds signature to given transaction.

## Token

```ruby
auth_xdr =
"AAAAAAKkwpEYoh4WhghDAaVE1PAxldny4jBw8nLw/hX17WSaAAAAZP///////x7wAAAAAQAAAABbMOQWAAAAAFsyNZYAAAABAAAAFU1vYml1cyBhdXRoZW50aWNhdGlvbgAAAAAAAAEAAAABAAAAACQ98LCW694lXtYrwD1snbvsW7ihc2DUQofx2MiQ89a+AAAAAQAAAAAkPfCwluveJV7WK8A9bJ277Fu4oXNg1EKH8djIkPPWvgAAAAAAAAAAAJiWgAAAAAAAAAACkPPWvgAAAEArkIA+SpNLPZd7g1PO3CD858zSl9UyjENEkfOzB4iggouV71JwQH/QiBIwDzCd8LS7l3Ay6Jz7zq7/NHDbZlkHHmQt2QAAAEBfDFgsjXzRcskEwgkGW9iiHJZX7hMS3/Vf+FOPkLuitD6mk9QlL/IV5q/G61YCA4YqmQpIpf8UUWsioiw1zoIN"

token = Mobius::Client::Auth::Token.new(seed, auth_xdr, address)
# =>
#<Mobius::Client::Auth::Token:0x00007f8bcc56b490
@seed="SCRBVUQF4N7JNXSEVQKZ4OEOEORC54BURPI3RCKN4FP3CO6AA7EZTJOA",
@xdr="AAAAAAKkwpEYoh4WhghDAaVE1PAxldny4jBw8nLw/hX17WSaAAAAZP///////x7wAAAAAQAAAABbMOQWAAAAAFsyNZYAAAABAAAAFU1vYml1cyBhdXRoZW50aWNhdGlvbgAAAAAAAAEAAAABAAAAACQ98LCW694lXtYrwD1snbvsW7ihc2DUQofx2MiQ89a+AAAAAQAAAAAkPfCwluveJV7WK8A9bJ277Fu4oXNg1EKH8djIkPPWvgAAAAAAAAAAAJiWgAAAAAAAAAACkPPWvgAAAEArkIA+SpNLPZd7g1PO3CD858zSl9UyjENEkfOzB4iggouV71JwQH/QiBIwDzCd8LS7l3Ay6Jz7zq7/NHDbZlkHHmQt2QAAAEBfDFgsjXzRcskEwgkGW9iiHJZX7hMS3/Vf+FOPkLuitD6mk9QlL/IV5q/G61YCA4YqmQpIpf8UUWsioiw1zoIN",
@address="GASD34FQS3V54JK62YV4APLMTW56YW5YUFZWBVCCQ7Y5RSEQ6PLL5COZ">

token.time_bounds
# =>
#<Stellar::TimeBounds:0x00007f8bcc568740 @attributes={:min_time=>1529930774, :max_time=>1530017174}>

token.validate!
# => true

token.hash
# TODO
*** Mobius::Client::Error::TokenTooOld Exception:
```
Checks challenge transaction signed by user on developer's side.

Attribute | Type | Description
----------|------|------------
seed | String | Users private key
xdr | String | Auth Transaction xdr
address | String | Developers public key

### Instance Methods

`#hash(format = :binary) ⇒ [String] Transaction hash`

Returns transaction hash.

`#validate!(strict = true) ⇒ [Boolean] Transaction hash`

Validates transaction signed by developer and user.

#### Parameters:
- strict [Bool] – if true, checks that lower time limit is within `Mobius::Client::strict_interval` seconds from now.

Default value of `Mobius::Client::strict_interval` is `10s`.

#### Raises:
- Unauthorized — if one of the signatures is invalid
- Invalid — if transaction is malformed or time bounds are missing
- Expired — if transaction is expired (current time outside it's time bounds)

`#time_bounds ⇒ [Stellar::TimeBounds]`

Returns time bounds for given transaction.

#### Raises:
- Unauthorized – if one of the signatures is invalid.
- Invalid – if transaction is malformed or time bounds are missing.


# Blockchain

## Account

```ruby
```

Service class used to interact with account on Stellar network.

Attribute | Type | Description
----------|------|------------
keypair | Stellar::Keypair | Account keypair

### Instance Methods


`account ⇒ Stellar::Account`

Returns Stellar::Account instance for given keypair.

`authorized?(to_keypair) ⇒ Boolean`

Returns true if given keypair is added as cosigner to current account.

`balance(asset = Mobius::Client.stellar_asset) ⇒ Float`

Returns balance for given asset.

`info ⇒ Stellar::Account`

Requests and caches Stellar::Account information from network.

`next_sequence_value ⇒ Integer`

Invalidates cache and returns next sequence value for given account.

`reload! ⇒ Object`

Invalidates account information cache.

`trustline_exists?(asset = Mobius::Client.stellar_asset) ⇒ Boolean`

Returns true if trustline exists for given asset and limit is positive.


## AddCosigner

```ruby
```

Adds account as cosigner to the other account.

Attribute | Type | Description
----------|------|------------
keypair | Stellar::Keypair | Account keypair
cosigner_keypair | Stellar::Keypair | Cosigner account keypair
weight | Integer | Cosigner weight, default: 1

### Class Methods

`.call(keypair, cosigner_keypair, weight) ⇒ Object`

Constructor shortcut to sign account.

### Instance Methods

`#call ⇒ Object`

Add cosigner to account.

## CreateTrustline

```ruby
```

Creates unlimited trustline for given asset.

Attribute | Type | Description
----------|------|------------
keypair | Stellar::Keypair | Account keypair
asset | Stellar::Asset | Asset instance, defaults to Mobius::Client.stellar_asset

### Constants

`LIMIT=922337203685`

ruby-stellar-base doesn't support unlimited yet.


### Class Methods

`.call(keypair, asset) ⇒ Object`

Constructor shortcut to create trustline.

### Instance Methods

`#call ⇒ Object`

Creates trustline for given asset.

## FriendBot

```ruby
```

Calls Stellar FriendBot

Attribute | Type | Description
----------|------|------------
keypair | Stellar::Keypair | Account keypair

### Class Methods

`.call(keypair) ⇒ Object`

Constructor shortcut to call friendbot.

### Instance Methods

`#call ⇒ Object`

Call Stellar friendbot.

## KeyPairFactory

```ruby
```

Transforms given value into Stellar::Keypair object.

Attribute | Type | Description
----------|------|------------
subject | ... | Given value to identify keypair.

`subject` can be any of the following:

- [String]
- [Stellar::Account]
- [Stellar::PublicKey]
- [Stellar::SignerKey]
- [Stellar::Keypair]

### Instance Methods

`#produce(subject) ⇒ Stellar::Keypair`

Generates Stellar::Keypair from subject, use Stellar::Client.to_keypair as shortcut.

# Errors

`Mobius::Client::Error::AccountMissing`

Raised if stellar account is missing on network.

`Mobius::Client::Error::AuthorisationMissing`

Raises if account does not contain MOBI trustline.

`Mobius::Client::Error::InsufficientFunds`

Raised if there is insufficient balance for payment.

`Mobius::Client::Error::MalformedTransaction`

Raised if transaction in question has invalid structure.

`Mobius::Client::Error::TokenExpired`

Raised if transaction has expired.

`Mobius::Client::Error::TokenTooOld`

Raised if transaction has expired (strict mode).

`Mobius::Client::Error::TrustlineMissing`

Raises if account does not contain MOBI trustline.

`Mobius::Client::Error::Unauthorized`

Raised in transaction in question has invalid or does not have required signatures.

`Mobius::Client::Error::UnknownKeyPairType`

Raised if unknown or empty value has passed to KeyPairFactory.

# FriendBot

```ruby
```

Calls Stellar FriendBot

Attribute | Type | Description
----------|------|------------
seed | String | Account private key
amount | Integer | Defaults to 1000

### Class Methods

`.call(seed, amount) ⇒ Object`

Constructor shortcut to call Mobius friendbot.

### Instance Methods

`#call ⇒ Object`

Call Mobius friendbot.
