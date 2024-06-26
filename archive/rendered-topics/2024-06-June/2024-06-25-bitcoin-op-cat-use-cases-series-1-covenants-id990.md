# Bitcoin OP_CAT Use Cases Series #1: Covenants

sCrypt-ts | 2024-06-25 22:35:27 UTC | #1

# Bitcoin OP_CAT Use Cases Series #1: Covenants

The OP_CAT opcode has potential to enhance the flexibility and functionality of Bitcoin, if reactivated. It allows developers to create impressive new features such as covenants. In the first of a series, we demonstrate a trustless sale of an Ordinal NFT using a layer-1 covenants made possible by re-enabling OP_CAT, without intermediaries.

![|700x700](upload://yWkKBtwoyqXwa6qxVCH2QmDJK4K.jpeg)

# OP_CAT

OP_CAT stands for “concatenate” in Bitcoin’s scripting language. It was originally included in the scripting language but was disabled due to concerns over potential vulnerabilities. OP_CAT allows concatenation of two strings on the stack.

It has been reactivated on Bitcoin Cash and SV since 2018 and has been used extensively to significantly enhance scripting capabilities while maintaining security. A few prominent use cases are:

* Quantum-resistant Lamport signatures: https://scryptplatform.medium.com/quantum-resistant-bitcoin-using-lamport-signatures-77a2c5030448
* Vaults: https://scryptplatform.medium.com/non-custodial-bitcoin-vaults-880c781effa7
* Tree signatures: [https://scryptplatform.medium.com/tree-signatures-8d03a8dd3077](https://scryptplatform.medium.com/tree-signatures-8d03a8dd3077#:~:text=Tree%20signature%20brings%20several%20salient,instead%20of%20all%20public%20keys)

On BTC, the reintroduction of OP_CAT has garnered interest lately. It has been assigned the Bitcoin Improvement Proposal (BIP) [number 347](https://github.com/bitcoin/bips/blob/5d774404790ce79d45588974b9964260a55bcbcd/bip-0347.mediawiki), marking the first step towards potentially implementing this long-discussed software upgrade.

# sCrypt vs Script

Our series will explore a wide range of use cases of OP_CAT. All of them will be implemented in [sCrypt](https://scryptplatform.medium.com/introduce-scrypt-a-layer-1-smart-contract-framework-for-btc-b8b39c125c1a) and open sourced, for anyone to independently try and verify on signet, where [OP_CAT has been activated](https://delvingbitcoin.org/t/bitcoin-inquisition-25-2/838).

Up till now, most discussions on OP_CAT on have been hypothetical and theoretical. One primary reason for the dearth of practical demonstrations is the immense difficulty of reasoning about and coding in Bitcoin Script, which is a low-level assembly language.

With sCrypt, a Typescript-based [domain specific language](https://en.wikipedia.org/wiki/Domain-specific_language) (DSL), developers can write Bitcoin smart contracts leveraging OP_CAT directly in TypeScript, one of the world’s most popular programming languages, used by millions of developers daily. sCrypt contracts are then compiled into Bitcoin Script.

We demonstrate sCrypt makes previously complex and intractable Script constructions as easy as developing Web2 applications, accelerating OP_CAT-enabled innovations. These empirical experiments provide data points and insights to the ongoing OP_CAT discourse.

# Covenants

In the context of Bitcoin, a covenant refers to a mechanism that can impose constraints on how future transactions involving a specific set of coins can be spent. Covenants are essentially restrictions or rules embedded within Bitcoin transactions, designed to control the conditions under which the coins can be transferred or spent. For example, they can restrict the addresses to which the coins can be sent, or require certain scripts to be included in future transactions.

One straightforward way to add covenants is to introduce new opcode such as OP_CHECKTEMPLATEVERIFY (CTV) in [BIP 119](https://github.com/bitcoin/bips/blob/5d774404790ce79d45588974b9964260a55bcbcd/bip-0119.mediawiki). Alternatively, it turns out covenants can be implemented by using a trick if OP_CAT were reactivated.

We first wrote about the introspection trick in **2020** using ECDSA signatures, which we called OP_PUSH_TX and its optimal variant.


## OP_PUSH_TX
One common misconception regarding Bitcoin script is that its access is only limited to the data provided in the…
scryptplatform.medium.com
](https://scryptplatform.medium.com/op-push-tx-3d3d279174c1?source=post_page-----0318052f02b2--------------------------------)

## Optimal OP_PUSH_TX
Since we implemented OP_PUSH_TX, a plethora of smart contracts have been built using this powerful primitive. As these…
scryptplatform.medium.com
](https://scryptplatform.medium.com/optimal-op-push-tx-ded54990c76f?source=post_page-----0318052f02b2--------------------------------)

Later on, it was written by Andrew Poelstra in 2021 and extended to Schnorr signatures.

## so some railing sail assails
This is the first in a series of posts about about covenants in Bitcoin using Taproot and a (hypothetical) CAT opcode…
www.wpsoftware.net
](https://www.wpsoftware.net/andrew/blog/cat-and-schnorr-tricks-i.html?source=post_page-----0318052f02b2--------------------------------)

## Covenants Using Schnorr Signatures

BTC’s Taproot upgrade introduced Schnorr signatures (see [BIP-340](https://github.com/bitcoin/bips/blob/5d774404790ce79d45588974b9964260a55bcbcd/bip-0340.mediawiki)). The signature scheme uses the same underlying elliptic curve as in ECDSA.

Suppose we want to sign using the key pair ***(x, P = xG)***. We generate an ephemeral key pair ***(k, R = kG)***. The signature is defined as the tuple (R, s), where

> ***s = k + xe***

***e*** is a hash of three things concatenated: our public key ***P’s*** x-coordinate, the x-coordinate of ephemeral ***R***, and our transaction data/preimage.

> ***e = H(Rx || Px || transaction)***

If we choose *k* = 1 and *x* = 1, we can simplify the equation:

> ***s = 1 + e***

By constructing such a signature within script and passing it to an OP_CHECKSIG, we can constrain chunks of the transaction data. In short, we get a working covenant with OP_CAT, which allows calculating the equation in script. You can read more about this trick in the aforementioned articles.

# Ordinal Locks

A seller transfers her ordinal NFT into a covenant smart contract (called an [ordinal lock](https://docs.1satordinals.com/ordinal-lock)) in a taproot output that can only be redeemed if the spending transaction pays the seller the asking price. Traditional Ordinals marketplaces based on Partially Signed Bitcoin Transactions (PSBTs) generally do not share these PSBT listings with each other. Ordinal lock can potentially create a global shared on-chain order book with more liquidity and transparency, which is public for everyone to access.

## Implementation

The signature equation above requires us to add 1 to a 256-bit integer. BTC currently only allows arithmetic in integers of 32 bits long, so we need to find a workaround.

Because ***e*** is derived from our transaction data, we can malleate the transaction until it hashed to a value of ***e*** that ends with 0x01 (or any value less than 0xFF) as we did in [optimal PUSHTX](https://scryptplatform.medium.com/optimal-op-push-tx-ded54990c76f). This takes 256 tries on average, which is instantaneous on modern hardware. For this example, we iterate on the value of ***nLockTime*** in the transaction, starting from 0*,* which won’t have an effect on the transaction’s validity/finality.

Once we have a value ending with 0x01, we can simply cut off the last byte and append 0x02 to it within the script to arrive at the correct value for ***s***.

Let us implement a simple function:

```
 @method()
    private checkSHPreimage(shPreimage: SHPreimage): void {
        const e = sha256(OrdListing.ePreimagePrefix + shPreimage.sigHash)
        assert(e == shPreimage._e + toByteString('01'), 'invalid value of _e')
        const s = OrdListing.Gx + shPreimage._e + toByteString('02')
        assert(this.checkSig(Sig(s), PubKey(OrdListing.Gx)))
        const sigHash = sha256(
            OrdListing.preimagePrefix +
                shPreimage.txVer +
                shPreimage.nLockTime +
                shPreimage.hashPrevouts +
                shPreimage.hashSpentAmounts +
                shPreimage.hashSpentScripts +
                shPreimage.hashSequences +
                shPreimage.hashOutputs +
                shPreimage.spendType +
                shPreimage.inputNumber +
                shPreimage.hashTapLeaf +
                shPreimage.keyVer +
                shPreimage.codeSeparator
        )
        assert(sigHash == shPreimage.sigHash, 'sigHash mismatch')
    }
```
If the code execution succeeds, it leaves us with a verified preimage of ***e***, i.e., variable *shPreimage*. Line 4 enforeces ***e*** ending with 0x01.

We then retrieve the *hashOutputs* field from *shPreimage*, which allows us to constrain the spending transactions’ outputs.

-------------------------

