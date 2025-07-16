# Chain Code Delegation: Private Access Control for Bitcoin Keys

jurvis | 2025-07-15 00:41:07 UTC | #1

Hi all,

Over the past several months, @jesseposner and I have been exploring a new approach to collaborative custody called Chain Code Delegation. By withholding BIP-32 chain codes and sharing only scalar tweaks at signing time, this technique allows custodians to enforce certain policies like spending velocity controls without ever possesing an XPUB; granting them full view of a key’s entire key tree. Below, we lay out the core concepts behind this design.

# Introduction

When sending a transaction with Script, the multisig redeem script contains all public keys associated with the quorum. Therefore, the predominant key arrangement for collaborative custodian services gives the custodian substantial visibility into the transaction history of its counterparties. For example, with a 2-of-3 multisig script, with public key A, public key B, and public key C, if a transaction signs for A and B, the redemption script will also include C, despite there not being a signature for C. Thus, a collaborative custodian can monitor the blockchain for any transactions that include their public keys to identify transactions that have been spent by their counterparties, regardless of whether or not they participated in the transaction.

One solution to this problem is to forgo the use of Script and instead use multiparty computation (MPC), such as with FROST, as described by Nick Farrow [0]. Another solution is to use disjoint spending paths with Tapscript. For example, a 2-of-3 can be expressed in a Tapscript with a branch for each 2-of-2 combination of the 3 keys. Spending from 1 of the branches would not reveal the public keys associated with the other branches. However, neither of these approaches work with ECDSA.

Another approach that requires neither Tapscript nor MPC, is to deprive the collaborative custodian of a chain code, and instead provide the custodian with BIP-32 scalar tweaks as needed for signing. In BIP-32 hierarchical wallets, an extended key pairs a standard key with a 32-byte chain code, which together enable derivation of an entire child-key tree. By withholding the chain code, the custodian only ever holds a non-extended keypair. When a transaction needs signing, the counterparty computes the required scalar tweak and shares only that tweak. Lacking the chain code, the custodian cannot derive any other child keys or spot public keys in redeem scripts they didn’t sign – so they only learn about the transactions they are explicitly given.

In addition to its privacy benefits, chain code delegation limits security blast radius: without the chain code (or undisclosed tweaks), the custodian’s key is effectively unspendable for any UTXOs they haven’t already signed. Tweaks are only revealed when a transaction is about to spend its associated UTXOs, which means those outputs are often consumed immediately. Even if the custodian’s systems are compromised or a tweak is leaked, an attacker then has only a very narrow window to race and sign that spend. Once the transaction confirms, the tweak is exhausted and becomes useless for any future signing.

This technique can be extended to other settings for enforcing access control policies relating both to information access for a key as well as the scope of UTXOs for which a key can sign.

# Setup

In a typical setup, the collaborative custodian derives a child key from a BIP32 seed and sends the xpub to their counterparty. Then, the counterparty uses the xpub to derive addresses from multisig scripts, each with distinct child keys derived from the xpub. However, with chain code delegation, the collaborative custodian instead generates a standard (i.e. non-extended) key pair and provides the public key to the counterparty.

The custodian does not generate a chain code. Instead, the counterparty generates a chain code on behalf of the collaborative custodian, but does not disclose the chain code to the custodian. By combining the chain code and the custodian’s public key, the counterparty is able to construct an xpub for the custodian.

# Signing

When the counterparty requests that the collaborative custodian sign a transaction, it derives the BIP32 scalar tweak from the xpub (i.e. the value `parse_256(I_L)` from BIP32) and provides it to the custodian:

```
# Inputs:
#   chain_code : 32-byte chain code (hidden from custodian)
#   P_par      : custodian’s parent public key (compressed)
#   i          : child index for spending

I   = HMAC-SHA512(key = chain_code,
                 data = serP(P_par) || ser32(i))
I_L = I[0:32]            # left half
t_i = parse256(I_L)      # scalar tweak mod n
```

Then, to sign for the child key, the custodian computes the child public key by adding the tweak to the custodian’s public key, and computes the child private key by adding the tweak to the custodian’s private key:

```
# Counterparty → Custodian: send t_i
# EC context:
#   G         = curve generator
#   point_add = EC point addition
#   scalar_mul= EC scalar multiplication
#   n         = curve order

# 2a. Child public key
P_i = point_add(P_par, scalar_mul(G, t_i))

# 2b. Child private key
k_i = (k_par + t_i) mod n
```

With the child private key k_i, the custodian produces a standard signature (e.g. Schnorr or ECDSA) over the transaction’s sighash:

```
# Compute the digest to sign:
msg_hash = compute_sighash(unsigned_tx, utxo_set, sighash_flag)

# Produce signature:
sig = sign(k_i, msg_hash)
```

By partitioning tweak derivation (off-chain) from key usage (on-device), chain code delegation lets the custodian sign exactly when needed—while never gaining the ability to derive or observe any other child keys or transactions.

# Change Output Validation
To enforce spending limits, a collaborative custodian must verify that change outputs are being sent back to their counterparty. This is typically done by providing the custodian with a descriptor during setup so that the custodian can derive change addresses and match them against the change outputs. However, with chain code delegation, the custodian doesn’t have access to any chain codes, and hence does not have access to xpubs or a descriptor.

Therefore, at signing time, the counterparty would need to supply a list of scalar tweaks, ${t_i}$, one per signer at index i. 

For each change output, the counterparty computes:
```
# Input:
#   chain_code: 32-byte chain code withheld from custodian
#   P_par:      parent public key (compressed)
#   i:          child index (e.g. change output index)

# 1. Compute HMAC-SHA512 pseudo-random data
I   = HMAC-SHA512(key = chain_code,
                 data = serP(P_par) ‖ ser32(i))
I_L = I[0:32]        # left 32 bytes

# 2. Parse tweak scalar
t_i = parse256(I_L)  # integer mod n
```

Then, with the received tweak `t_i`, and parent key `P_par`, the custodian computes:

```
P_i = point_add(P_par, scalar_mul(G, t_i))
```

Which would yield the exact child public key `P_i` that should appear in the change output.

Then, it would build the expected scriptPubKey: 

`change_script_i = OP_0 || SHA256( sorted_multi(P_1, P_2, P_3, …) )`

And verify that it matches the output script pub key it sees in the transaction.

This works because during setup, the custodian is provided the non-extended parent public keys for the change keys, such that it can use those parent keys to validate change outputs by reference to the scalar tweaks that were used to derive the change child keys used in the change outputs.

# Privacy

Without the chain code, the custodian only learns about transactions that it has signed. For privacy sensitive transactions, the counterparty can purposely sign without the custodian, and the custodian will not be able to detect those transactions merely by reference to its key material.

For full privacy, in which the custodian doesn’t even learn about transactions that it has signed, blind Schnorr signatures can be used in conjunction with chain code delegation. With this technique, the counterparty can apply the BIP32 tweak to the signature it receives from the custodian. We can do this because of Schnorr’s linear form:

$$
\begin{align*}
s &= r + c * x \\
s + c * t_i &= r + c(x + t_i)
\end{align*}
$$

For custodians that enforce policies when signing, predicate blind signatures [1] can be used in conjunction with blind Schnorr signatures, such that zero-knowledge proofs assert any arbitrary predicate about the transaction being signed.

# Security

The custodian’s key cannot be used to sign for any UTXOs without the chain code or a scalar tweak. This substantially reduces the blast radius of a compromised custodian key.

As the custodian learns of scalar tweaks, it is signing transactions that spend the associated UTXOs, which quickly limits the usefulness of the child keys that the tweak derives. If a custodian’s systems are fully compromised, at most an attacker would be able to race the counterparty to sign for UTXOs as scalar tweaks are disclosed. Of course, in addition to the custodian key, such an attacker would also need to have compromised sufficient keys to sign for the multisig script.

This mechanism is also useful for limiting the blast radius of a key in other contexts. For example, if a bitcoin user has a signing key that they store on a mobile phone, such a user might wish to limit the power of that key because of the large attack surface of the execution environment. A specialized device, such as a hardware wallet, could be used to generate a chain code for the mobile phone’s key, and then selectively disclose scalar tweaks. The mobile phone would only be able to spend UTXOs for which it has received tweaks, allowing the user to determine the UTXOs the mobile phone has access to at any given time, for both past and future UTXOs.

-------

References
* [0] Private Collaborative Custody with FROST: https://gist.github.com/nickfarrow/4be776782bce0c12cca523cbc203fb9d/
* [1] Concurrently Secure Blind Schnorr Signatures: https://eprint.iacr.org/2022/1676

-------------------------

marathon-gary | 2025-07-16 20:15:30 UTC | #2

This seems to be similar to [blinded xpubs](https://github.com/mflaxman/blind-xpub) but with an even greater reduction in "blast radius" since after revealing the blinded path for the xpub, any usage thereof is now compromised and the custodian is able to view any future usage of the xpub derived keys.

Very cool.

-------------------------

