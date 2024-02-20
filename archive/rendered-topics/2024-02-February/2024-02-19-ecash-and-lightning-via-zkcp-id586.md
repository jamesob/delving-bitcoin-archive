# Ecash and lightning via ZKCP

ajtowns | 2024-02-19 11:57:37 UTC | #1

Is it possible to link ecash mints to the lightning network without losing ecash's anonymity or adding any additional trust? I believe it is. (This is an expansion of [a tweet](https://twitter.com/ajtowns/status/1748866818067116506))

## Background

Lightning uses HTLCs to make atomic payments: that is you have a tuple $(H, T, A)$ of a hash, timeout and amount, that is atomically resolved by revealing a preimage $P$ where $H$ is the result of running sha256 over $P$.

Ecash used a blind mint, where the mint issues coins that are backed by its holding of bitcoin, and payments are authorised by revealing a pair $(K, S)$ where $S$ is a blind signature of $K$ calculated as $m\cdot H2C(K)$ with $m$ being the mint's private key.

The mint is trusted in three ways:
 1. it's trusted not to lose/spend/steal all the funds backing the coins it issues;
 2. it's trusted to correctly use its private key when doing blind signatures; and
 3. it's trusted to accept coins it issues either to replace them for new coins when ecash is being transferred to a new owner, or to redeem it for the bitcoin it's backed by.

What we'd like is to be able to both pay the mint over lightning for issuing new tokens, and to redeem existing tokens for bitcoin to be received over the lightning network.

## Model

Because ecash transactions naturally require the mint to be involved, either to validate the coin has not already been spent or to mint a new coin, we only consider a random bitcoin user interacting with the mint over lightning.

Because different cryptography schemes are involved (hashing for lightning, blind signatures for ecash) we use a [zero-knowledge contingent payment](https://bitcoincore.org/en/2016/02/26/zero-knowledge-contingent-payments-announcement/).

## Issuing new ecash

First, we'll consider sending bitcoin over lightning and receiving ecash. For this, the user will first need to calculate a random blinded challenge for the mint as normal (ie, $A = H2C(K) + rG$). This should be sent to the mint out of band, who will calculate:

 * $B = mA$
 * $H=SHA256(B)$
 * $Z$ a ZKP that it knows a value $m$ such that $H=SHA256(mA)$ and $M=mG$ where $M,A,H$ are public parameters and $m$ is private.

$Z$ is provided back to the user, along with a lightning invoice to pay the HTLC $(H,T,A)$. If the user pays the invoice, they obtain the preimage $B=mA$ and can unblind the signature (calculating $C=B-rM$), giving them ecash, and the mint's bitcoin balance increases by A.

Because the mint is trusted anyway, it is probably fine for the mint to perform a trusted setup for the zero-knowledge proofs here. After all, if the mint wants to steal funds, it can just spend the bitcoin, and shut all its servers down -- no fancy cryptography required. However, if the zero-knowledge system did not require a trusted setup, I think this would remove the second element of trust in the mint ("it's trusted to correctly use its private key when doing blind signatures") -- that is, the mint would not be able to do an invalid blind signature, resulting in an invalid token being held by the user.

## Redeeming ecash

In order to redeem ecash, you provide $C,K$ and the mint checks that $C$ is a signature for $K$ (ie that $C=m\cdot H2C(K)$), and then logs $K$ as having been spent. In order to do use this to atomically release funds over lightning, the following protocol should work:

 * user sends an invoice for redeeming the coin denominated by $K$
 * mint calculates the correct signature $C=m\cdot H2C(K)$ and the hash $H=SHA256(C)$
 * mint sends an HTLC of $(H,T,A)$ to the user, paying the invoice provided the user does have the correct blinded signature $C$
 * user claims funds $A$ by revealing preimage $C$
 * once the HTLC resolves, the mint marks $K$ as having been spent (NB: while the HTLC is unresolved, $K$ must be marked as reserved, otherwise there is a risk of a doublespend)

(No ZKP seems to be required for this direction)

-------------------------

calle | 2024-02-19 14:12:47 UTC | #2

Nice writeup, I especially like the "Issuing new ecash" part. We had thought of implementing something similar in Cashu in the issuing process but decided against implementing it for now because it would require more interaction with the user's lightning wallet (they need to obtain the preimage in their Lightning wallet, then use it in their ecash wallet).

However, good news: we already have a ZKP implemented as a proof of knowledge of `m` as a DLEQ proof, which you can find here: https://github.com/cashubtc/nuts/blob/94a621d1014a8687269f58ecadc5ef167dce546f/12.md

As far as I can see, your construction is missing a crucial part though: the transferrability of these tokens from user to user without the mint being able to link the transfer to the Lightning invoice (i.e., keeping the ecash privacy intact). Since the mint does not know the `K` the user requests a signature on, it's hard to make sure that the claim of a token remains intact when passed form user to user (and all sorts of other issues arise, such as Alice being able to re-mint a token committing to the same invoice again using a different `r` after having sent it to another user Carol).

This has been the main challenge for my research in the past months. Good news though, I think I've found a way to do it and will write it up once I have the code to show it's viable. The rest is almost exactly as you propose it here.

I really enjoyed reading this, cheers!

-------------------------

ajtowns | 2024-02-20 01:47:06 UTC | #3

[quote="calle, post:2, topic:586"]
As far as I can see, your construction is missing a crucial part though: the transferrability of these tokens from user to user without the mint being able to link the transfer to the Lightning invoice (i.e., keeping the ecash privacy intact).
[/quote]

If you want to transfer the tokens, you just do a regular ecash transfer (generate a new blinded value and pay the mint to sign it by spending your existing coin) -- the "issuing new ecash" part only reveals blinded values (A and B) to the mint, then the unblinded value (C, K) is what's used later to authorise transfers.

[quote="calle, post:2, topic:586"]
Since the mint does not know the `K` the user requests a signature on, it’s hard to make sure that the claim of a token remains intact when passed form user to user (and all sorts of other issues arise, such as Alice being able to re-mint a token committing to the same invoice again using a different `r` after having sent it to another user Carol).
[/quote]

Maybe I'm misunderstanding something; but:

 * transfering a token to a new user means telling the mint "here's $C_1,K_1$ please mark that as spent, and issue me a new blind signature for $A_2$"
 * re-minting the same token with a different $r$ means you'll end up with the same signature pair $C, K$; but because the mint tracks spent coins, you'll only be able to spend it once, despite having paid twice to have it minted, meaning the mint will end up with more bitcoin backing its issued coins than there are coins to be redeemed

[quote="calle, post:2, topic:586"]
However, good news: we already have a ZKP implemented as a proof of knowledge of `m` as a DLEQ proof,
[/quote]

Nice!

-------------------------

