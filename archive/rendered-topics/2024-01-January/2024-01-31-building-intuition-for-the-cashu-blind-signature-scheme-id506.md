# Building Intuition for the Cashu Blind Signature Scheme

thunderbiscuit | 2024-01-31 15:07:20 UTC | #1

This post unpacks the workflow of the ecash blind signature scheme used in [Cashu](https://github.com/cashubtc/nuts) and explained [here (Ruben Somsen)](https://gist.github.com/RubenSomsen/be7a4760dd4596d06963d67baf140406) and [here (David Wagner)](https://cypherpunks.venona.com/date/1996/03/msg01848.html) in a more approachable way.

# Part 1
## Prerequisites
**Prereq 1: Diffie-Hellman Points**. Given two users Alice and Bob, each with a private/public key pair ($A = aG$ and $B = bG$ respectively), an elliptic curve Diffie-Hellman point is a point on the curve which both parties can derive independently using each others' public keys. Assuming Alice knows Bob's public key and Bob knows Alice's, both of them can arrive at a shared, common point (public key) $C$ on the curve using the following:
$$
\begin{align*}
C &= aB = bA \\
C &= a(bG) = b(aG) = abG
\end{align*}
$$

The equations above state that Alice can arrive at point $C$ by multiplying Bob's public key $B$ with her private key $a$, and that Bob can arrive at _the exact same point_ $C$ by multiplying Alice's public key with his private key $b$.

This ability to derive a shared common secret $C$ is used everywhere in modern day networking, where parties can, simply by exchanging public keys across a potentially unsecured communication layer, arrive at a common secret _known only to them_, which they can then use to encrypt further communication between each other.

Note that to separately arrive at this common point, both parties need access to these 2 things:
1. Each other's public keys
2. Their individual private keys

You can think of the Diffie-Hellman point as something you can arrive at from two sides, as long as each side has its private key:
```txt
Alice's private key   --->   D-H Point   <---   Bob's private key
```

If either party does not have its own private key, they cannot access this D-H point by themselves.

<br>

**Prereq 2: You can "prove" you don't know the private key to a point**. You can prove that you don't know the private key to a given point on the curve by showing how the coordinates of the point were derived from the hash of a given message. To help us in this "proof", we'll use a function
```python
hash_to_curve(message) = PointCoordinates
```

A user who created a point on the curve starting from a private key (multiplying that private key with the generator point and using the resulting point, e.g. $A = aG$) would end up with coordinates to that point, say `Coord1`. Finding a message that hashed to those same coordinates, e.g.
```
hash(message) = Coord1
```
would require breaking the preimage resistance property[^1] of the hash function (finding an input given a target output). Relying on the assumption that this is not feasible, we can assume that a user who knows the private key to a point would not be able to _also_ know a message that hashed to the point's coordinates. We also know that one of the property of elliptic curves is that given a point on the curve, one cannot feasibly compute its corresponding private key, and together these two mean that:

**_If you know the private key to a point, you don't know the preimage that would hash to this point's coordinates, and alternatively, if you know the preimage (message) to a set of coordinates, you don't know the private key to that point._**

# Part 2: Naive, non-blinded tokens
Let us start with the "naive" version of the protocol, one where no blinding happens on the tokens. This is simply a description of how you could do a token system with a mint and users (we'll add privacy in Part 3).

Imagine a mint, `Mike`, who agrees to give users tokens they can use to interact with his API service. To keep things simple, we'll assume Mike only offers tokens that are worth $10 each. You can buy a token from Mike (we'll say that Mike "mints" a token) by transferring him $10 out-of-band. What would these tokens look like and how could Mike create them?

One way Mike can achieve this is by publishing his public key $K$. He needs a way to create tokens that he can prove were issued by him, and makes his scheme public on his website by saying:

_If you send me a public key for which you don't know the private key, I will know that I alone can produce the Diffie-Hellman point $C$ between my public key and yours. I will only give out this point $C$ once I have received a $10 payment. Each of those Diffie-Hellman points is then worth $10, which you can redeem with me at any time._
<br>

The important part here is that anyone can produce a Diffie-Hellman point between Mike and a public key they have if they know the private key for that point (this is done using $aK = C$, where $C$ is the Diffie-Hellman point, see prerequisite 1 above). Mike knows this, and so can't accept _just any_ Diffie-Hellman point between his $K$ and any given public key. For Mike to consider a point $C$ valid as a token, a user has to prove to him that they don't know the private key to their point used in the D-H point $C$. If that is the case, Mike knows they could not have derived point $C$ by themselves, and therefore that _he must have been the one to give it to them_ (and since he never gives out D-H points without payments, the point is a valid proof that a payment of $10 was made to him sometime in the past).

The scheme can be thought of as the following steps:
1. Alice creates a public key she provably doesn't know the private key to:
```python
hash_to_curve(message) = PointA
```
2. She shares it with Mike, and sends $10
3. Mike sees the payment and the public key (point), and issues the associated Diffie-Hellman point $C$:
$$
kA = C
$$
4. Alice now has point $C$ in her pocket.
5. When she needs to redeem the token, she shows Mike point $C$ and point $A$. Mike says "sure but can you prove that I gave you that point and you didn't just derive it yourself?". Alice proves that she could not have derived point $C$ by herself by providing the secret `message` she used to create point `A`.
6. Mike computes:
$$
\begin{align}
hash\_to\_curve(message) &= PointA \\
kA &= C
\end{align}
$$

and knows that Alice could not have found point $C$ by herself, and he therefore must have been the one to give it to her. He considers the token valid, and Alice can purchase what she needs with the token.

From the steps described above, we find that what Alice needs to redeem her token are two pieces of data: 
1. The point $C$
2. The secret $message$

In this sense, a token can be thought of as the pair $(C, ~message)$.

As is, this system allows Mike to offer electronic tokens that are then redeemable to him at a later time. He does, however, know exactly which tokens (and who paid for them) get redeemed when, because to issue point $C$ he needs to know point $A$, and he can associate that point $A$ with Alice when she makes the initial payment to him. This means that when Alice redeems the token, he'll know she's the one redeeming it. Part 3 explores how Alice can gain privacy against Mike the mint by slightly altering the points $C$ and $A$, but in a way that still allows Mike to know he was the one to issue the token.

To recap, the "naive" token scheme is basically this:

_Alice goes to Mike the mint and says "here is $10, please give me the D-H point between your public key and this other public key. I can't generate this D-H point myself because I don't know the private key to this public key". The next week, Alice goes back to Mike and says "here's a D-H point between your public key and this other public key I have. You can verify that you were the one to give this D-H point to me because I can prove that I don't know the private key to mine, and therefore the D-H point must have been created by you."_

[^1]: *Premiage resistance* is the property that a hash function is hard to invert, that is, given an output of the hash function it should be computationally infeasible to find an input that maps to that element.

# Part 3
In Part 2 we outlined the general scheme used by Mike the mint and Alice the user to create and redeem tokens. Our scheme so far, however, suffers a serious flaw: all tokens are associated with the user who requested them, and therefore Mike can tell:
1. _When_ a user redeems _which_ of their tokens
2. If a _different_ user redeems a token that they had not originally requested (say Bob redeems token A, which Mike knows was coming from Alice), Mike can infer that the token was used as payment in a transaction between Alice and Bob.

Together, these are severe privacy concern for users of the tokens, and render the "naive" scheme unusable in practice.

## Goal
A better version of this would be one in which anyone can redeem any of Mike's tokens without Mike knowing who requested them in the first place, if they have been exchanged in between, and how long it has been since their creation. This poses a problem because when Alice comes to redeem her token (the DH point given to her by Mike), he remembers giving her that specific point!

What we need is a way for Alice to cryptographically "fiddle" with the DH point in such a way that the token is still valid when she brings it back to Mike (i.e. Mike can recognize that she could only have gotten that point from him), but is different than the original so that Mike can't associate the original from the modified, making it impossible for him to tell when and to whom the token was issued in the first place.

## Blinding and Un-Blinding Tokens
In practice, the way Alice achieves this is by sending Mike for signature (computation of the DH point) a key she has _blinded_, and later on is able to _unblind_ it.

We call this key a `BlindedMessage`, and the notation for is is `B_`. It consists of the actual public key she wishes to use _added_ to a random other public key (remember that we can add points on elliptic curves). Alice can use the key she used in Part 2 (`A = hash_to_curve(x)`) and add key `R`, (`R = rG`). Point `R` here is simply a point Alice creates by choosing a random private key `r` and saving it. The `BlindedMessage` is therefore defined as the addition of those two points:
```txt
BlindedMessage = A + R
BlindedMessage = A + rG
```

She sends that BlindedMessage (again just a point on the curve) to Mike, who produces the DH point between this public key and his (`K`) like so:
```txt
C_ = k(BlindedMessage)
```

We call this `C_` point the _blinded signature_, and he sends it back to Alice, who can now unblind it by performing the following steps:
```txt
C_ = k(A + rG)
C_ = kA + krG
C_ = kA + rK
C_ - rK = kA + rK - rK
C = kA
```

Note that in Part 2, the DH point Mike provides Alice is simply his private key `k` multiplied by her public key `A`, i.e. (`D-H = kA`). In the steps outlined above, she can get to `kA` through a roundabout way that involves her not providing `A` _directly_ to Mike, which means when she redeems it later on, Mike will have never seen point `A` and cannot associate it with her.

This point `C` is called the _unblinded key_, and is the key she would have gotten if she just provided Mike with her key `A` in the first place.

Alice can, later on, go to Mike and present the pair `(C, x)`. Mike can attest that x produces point `A` by doing `hash_to_curve(x) == A` (confirming that Alice doesn't know the private key to this particular point), and that `C` is indeed the D-H point between `A` and his public key `K` (`C == kA`). Given these two facts, he could only be the one having provided this point to Alice at a prior time. He verifies that this particular point `C` has not been redeemed by anyone else before, and if it has not, considers the token valid!

## Tokens as payments between users
Because this token is valid and has good privacy propeties, it might be accepted as payment by others. Alice can pay Bob with the token `(C, message)` and Bob can redeem the token with Mike, who has no idea an exchange was made between the two.

-------------------------

moonsettler | 2024-02-12 16:47:19 UTC | #2

[quote="thunderbiscuit, post:1, topic:506"]
Because this token is valid and has good privacy propeties, it might be accepted as payment by others. Alice can pay Bob with the token `(C, message)` and Bob can redeem the token with Mike, who has no idea an exchange was made between the two.
[/quote]

in the basic scheme there is no way to verify the signature is correct by anyone but the mint. there is also no one else that can tell if it's a double spend.

since the mint is the only one that can tell if a signature is his or that it's not a double spend, this also means he has the ability to commit fraud and claim that a token has already been spent and was known to him.

fraud proofs are possible to be carried out in front of an arbitration body or the public, in a challenge response manner, that requires a different multi-step sequence of issuance and redemption.

the DLEQ proof given by the cashu mint (to avoid any potential secret tagging, ie the mint using different keys for each token) can be used to prove the mint's private key was used to create the token, which is a valid token, BUT to prove that you have to reveal your blinding factor undoing the unlikability of issuance and redemption.

it is theoretically possible to transfer ecash tokens offline between parties or similarly to how the hawala network works, that requires the secret to be a hash of additional spend conditions enforced by the mint. therefore implementing pubkey spend, time locked redeem 'scripts'. this ofc hurts fungibility pretty bad.

-------------------------

ZmnSCPxj | 2024-02-13 02:34:17 UTC | #3

Another fungibility hit here is that each token is that it has no information attached to it other than the fact that it *presumably* exists.  What this means is, suppose you use an ecash scheme where you have tokens represent 1 satoshi of value. To transfer 10mBTC, you need 1 million tokens, each ~65 bytes (or 64 if you do the X-only point trick) or ~64Mb.

To lower the cost of transferring, practical mints using the BDHKE scheme need to issue multiple denominations of tokens. For example, tokens of value 1, 2, 4, 8, 16,...powers of 2 satoshi each, and have operations to split/combine tokens. This allows for O(log N) tokens to be transferred to transfer a specific value, instead of O(n). The problem is that for BDHKE each denomination requires its own minting key. This effectively reduces the anonymity set drastically, since each denomination is technically its own token, and you may be using denominations that just so happen to be rarely used by others.

An improvement is to use anonymous credential schemes.  A credential can commit to one or more scalar values, and some credential schemes also can commit to one or more point values. The scheme used in WabiSabi, for example, commits to a point, and that point is a Pedersen commitment to an amount in satoshis.  Each credential therefore hides its own amount and does not represent a single fixed amount like in BDHKE.  While each token has its own possibly-unique denomination, we can make this a fungibility non-problem by designing mints so that they provide semantics similar to MimbleWimble, specifically: the ability to present multiple Pedersen commitments as inputs, then present multiple Pedersen commitments as outputs, and then let it be publicly-verifiable that sum of inputs == sum of outputs + delta blinding factors and a signature of the delta blinding factors.  This allows every denomination to be fungible with every other denomination.  The WabiSabi implementation I believe has a generic 2-input 2-output operation, as well as free minting of credentials that provably commit to a 0-valued amount to allow the same generic 2-input 2-output operation to split a single credential (you get your current credential with a specific amount, then ask for a freebie 0-value credential, as the 2 inputs, then get out 2 outputs that have the split you want).

Anonymous credentials do need a zk-proof that the amount lies within a range, which is generally implemented by proving that you either have an amount 0 or an amount equal to a power of 2, then have log N such proofs to prove that you have a total amount that is between 0 to N-1. This means that transferring of value still requires O(log N) data, but with better privacy (each denomination IS unique but are also all interfungible with each other, instead of the multi-denomination BDHKE token case where every denomination is not directly fungible with other denominations).

----

Finally: ALL ecash mints have rugpull capability.  A mint has the private key to construct a token / credential, and the mint can issue itself (or a secret conspirator) a token/credential that contains an amount it did not receive as a deposit or in exchange for an existing token/credential.  This can then be publicly shown and publicly withdrawn from the mint in the backing system (onchain or Lightning) as if it were just another withdrawal, and then at  some point the mint has no backing for tokens it already issued to non-conspirators.

In theory a proof-of-reserves can be built, but in an anonymous credential system, it would be difficult to associate a credential with some amount that the mint knows and therefore can build a proof-of-reserve for. With multi-denomination BDHKE tokens, the mint can claim that it currrently has in circulation some number of tokens of a specific denomination as well as a list of the blinded tokens it issued and a list of unblinded tokens it already invalidated (the difference in length of both lists must then be equal to the claimed number of tokens of that denomination issued), and a client can check the proof-of-reserves by checking that the blinded forms of its held tokens are in the list of issued blinded tokens and its unblinded form is not in the list of unblinded tokens it invalidated, as well as that all the claimed issued tokens times denominations sum up to the proven reserves of the mint. But see the above issue with privacy vs multi-denomination BDHKE as used in Cashu ---- the reason the mint can ***make*** proof-of-reserves is precisely due to the lower privacy.

Proof-of-reserves can also be worked around by the above technique of overprinting tokens and then not including the secret overprinted tokens in the proof-of-reserve, thus allowing a rugpull to be done at any later time after the proof-of-reserve snapshot is taken.

-------------------------

moonsettler | 2024-03-02 03:01:54 UTC | #4

[quote="ZmnSCPxj, post:3, topic:506"]
ALL ecash mints have rugpull capability.
[/quote]

it's a complete non-issue for credit ecash.

-------------------------

