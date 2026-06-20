# Privately running wallet privacy heuristics on your own UTXOs without needing to run your own node

vazertuche | 2026-06-20 18:56:54 UTC | #1

Privacy-heuristic tools have gotten a lot of attention lately. [am-i.exposed](https://am-i.exposed) lets you paste an xpub or an address and get a read on how much your wallet's history leaks. More recently supertestnet opened a WIP wallet-fingerprinting panel for btc-rpc-explorer that gives a heuristic "which wallet likely created this transaction" verdict from on-chain signals alone. (nVersion, nLockTime / anti-fee-sniping, RBF signaling, input script types, low-R grinding, BIP-69 ordering, change-type matching, and so on.) People are clearly interested in finding out what their transactions reveal, which is good for the ecosystem.

What stood out to me was how many of them pasted their xpub into a public website to get that read, instead of running the analysis privately against their own node. An xpub handed to a third party is the entire wallet.

This is a writeup of a system that addresses that. It lets anyone, including people who don't run their own node, run these privacy heuristics over their own wallet without revealing to any server which transactions, or which wallet, they are asking about. The heuristic verdicts are precomputed per transaction and published as encrypted per-block filters. A wallet can only decrypt the verdict for an outpoint it is actually interested in, and the server never learns which outpoints those are.

## 1. The heuristics

For every transaction, the builder computes a privacy classification and packs it into a 24-bit tag. It is a superset of the signals that am-i.exposed and the wallet-fingerprinting work surface, and the taxonomy borrows heavily from am-i.exposed, which I want to credit directly. There are roughly 17 heuristics. They group into linkage and clustering signals (common-input-ownership, address reuse, peel-chain membership, entity clustering against a known-address index), structural signals (CoinJoin participation, consolidation, dust, script-type mixing across inputs, multisig, OP_RETURN presence), fingerprint-style signals (transaction version, locktime / anti-fee-sniping, RBF signaling, deterministic ordering caps), and an aggregate entropy or anonymity-set estimate that includes an explicit "anonymity set is none" flag.

No single heuristic here is new, and most are well known on this forum. What I think is useful is that a wallet can get the combined verdict for its own transactions without anyone learning which transactions those are. floppy's wallet-software fingerprinting is the kind of signal that could be added to this tag channel later (see point 6).

## 2. How a wallet gets a private verdict

The server publishes, per block, a filter with two columns.

Column A is a BIP-158-style GCS membership set over `SipHash(txid)`: values sorted, delta-encoded, Golomb-Rice compressed. `P` is chosen adaptively per block (incremented until the reduced hash space is collision-free for that block, capped at 32) so Column-A ranks map one-to-one onto Column B.

Column B is a bit-packed array of the per-transaction heuristic tags, but each tag is stored as a small codebook index (`code_bits`, currently 11) and encrypted so it can only be read by someone entitled to that specific transaction. The codebook maps each distinct 24-bit tag to an index, and the header pins the version via `codebook_id`.

A client that matches a txid in Column A reads exactly one code from Column B and decrypts it with a per-outpoint key it can only derive by spending a token (below):

```
tag_key = HMAC-SHA256(key = gamma.serialize_compressed(), msg = "kyc-tag")
nonce   = deterministic from (txid, block_hash)
```

The short Column-B code is decrypted with an AES-256-CTR keystream. So the heuristic verdict for a transaction sits in public, per-block data, but is only legible to a wallet that controls (and has paid to query) that outpoint.

\- The per-outpoint key comes from a VOPRF

The server holds a secret scalar `v`, publishes `V = v*G`, and the key for an outpoint is `gamma = v*H2C(outpoint)`. The wallet obtains `gamma` without revealing the outpoint, via an oblivious PRF:

```
P     = H2C(outpoint)
alpha = rho * P            // client blinds with fresh rho
beta  = v * alpha          // server evaluates, never sees P
gamma = rho^(-1) * beta    // client unblinds, gamma = v * H2C(outpoint)
```

Because `rho` is fresh each time, repeated queries on the same outpoint produce uncorrelated `alpha` values.

Querying is metered by single-use tokens, blind-signed at purchase so spends cannot be linked to issuance:

```
P_i     = Poseidon(token_seed, pack_id, i) * H_TOKEN
sigma_i = v * P_i           // server's blind signature (with a per-token DLEQ proof)
nullifier = Poseidon(token_seed, pack_id, token_index)   // one-time spend tag
```

A query is one round trip carrying `(Groth16 proof, alpha_query)`. There is no login and no credential. The ZK proof is the authentication, and the OPRF query is a separate, unlinkable point. The server verifies the proof, records the `nullifier` (rejecting duplicates), evaluates `beta = v*alpha_query`, and returns it.

Groth16 on BN254 with BabyJubJub in-circuit arithmetic. There are 3 public inputs: `nullifier`, `current_month`, `revocation_root`. `PK_server` is a hardcoded R1CS constant, so key substitution is not even expressible. The constraints, briefly:

```
1. Base-point derivation   P_i = Poseidon(seed, pack_id, idx)*H_TOKEN
     blocks arbitrary-point injection: a free P_i lets a prover set
     P_i=k*G, sigma_i=k*PK_server and forge a signature the server never made.
2. Nullifier binding        Poseidon(seed,pack_id,idx) == nullifier
3. DLEQ verification        recompute Chaum-Pedersen w/ Poseidon FS   [critical]
     proves sigma_i = v*P_i; without it sigma_i is a free witness = free tokens.
4. Expiration check         bit_decompose(expiration_month-current_month,18)
5. Activation check         bit_decompose(current_month-start_month,18)
6. Revocation non-inclusion walk depth-16 SMT from Poseidon(pack_id)
     proves the pack isn't in the chargeback revocation set.
```

The full constraint-by-constraint reasoning is in the spec.

\-- Transport and decoy fetches

All traffic defaults to Tor, via an embedded Arti client to the server's native `.onion` service, with no exit node, so the server never sees a client IP. And because the choice of which filters you download could itself leak which blocks you care about, each real block is fetched alongside decoys sampled from a discrete Laplace distribution centered on the real height (defaults: 10 decoys, roughly a +-4,000-block spread), deduplicated and sorted so real and decoy heights are indistinguishable.

## 3. Threat model

Assume a fully malicious server: full control of software, database, and network, able to observe all requests, modify responses, and collude with chain-analysis firms. The claim is that it learns nothing beyond four things: that a valid spend occurred, the public `nullifier` (unlinkable to any token, pack, or user, because Poseidon is preimage-resistant), the global `current_month`, and the `revocation_root` it published itself. The OPRF blinding hides the queried outpoint, the proof reveals nothing about the witness, and Tor hides the network identity. The verdict you read for your own coins never tells the server which coins they were.

## 4. Why this matters

Most of the people who would benefit from privacy-heuristic feedback are the people least equipped to run it safely. They don't run a node, so the path of least resistance is to paste an xpub into someone else's website. If we can give those users the same insight while leaking nothing, two things follow. They stop training themselves to hand wallets to strangers, and seeing their own leakage in concrete terms gives them a reason to care about privacy in the first place: coin control, avoiding address reuse, label hygiene, and over time rethinking the habit of buying only from KYC exchanges. The heuristics get people in the door. The education is what I actually care about.

## 5. A future direction: provenance tags and watching anonymity sets decay

The same encrypted tag channel can carry more than structural heuristics. One direction I find worth exploring, and would want to do carefully, is a provenance or KYC-exposure bit, and in particular watching a CoinJoin's anonymity set shrink over time. A freshly mixed output starts with a large set, but as siblings get consolidated, reused, or sent to identified entities, the effective set decays. Surfacing that decay privately, per output, would let a user know when a coin's mix has quietly stopped protecting them. This is scoped as future work, not part of what is implemented today.

## 6. Limitations and what is not yet implemented

Key rotation is operationally expensive. `PK_server` being a constant is a security choice, but rotating `v` currently needs a new trusted setup. The planned fix is FROST threshold management of `v`, and no FROST code exists yet.

An OHTTP fallback is planned but not built. Tor `.onion` is the only privacy transport implemented today.

Groth16 needs a trusted setup. The proving and verifying keys, and the tooling to reproduce the setup, are public so it can be audited or rebuilt. However, given that the users privacy is primary secured by the VOPRF, having a trusted setup seems a fair tradeoff. (Although I will try to get a large group of public individuals to participate in a signing ceremony to get a 1 of N trusted setup. I've also vibecoded a Halo2 trustless setup version of the circuit but have not deployed or tested it heavily yet.)

Heuristic correctness is a data-quality problem, not a cryptographic one. The cryptography hides which verdict you read; it does not make the verdict right. A wrong tag would be privately and verifiably wrong.

## 7. What I'm looking for

Mostly cryptographic review. Does proof-as-authentication with a separate, unlinkable VOPRF query point hold, or is there a binding attack? Is reusing `v` as both the filter-encryption key and the OPRF evaluation key sound, given the per-outpoint key is `gamma = v*H2C(outpoint)` followed by HMAC? Is anything in the Section 4 threat model overclaimed? And separately, are the heuristics in Section 1 the right set, and what would you add or drop?

Circuit, proving and verifying keys, and setup-reproduction tooling: **\[link: circuit repo\]**. Full written specification: **\[link: spec\]**. The server and key material are not public, by design.

I would rather hear that something is broken now than after people rely on it.

## Disclosure

To be upfront with this community: this is the cryptographic core of a product, a fork of Sparrow Wallet with a paid token model where tokens are sold in packs and payment is what gates blind-sign issuance is what I have built. I led with the heuristics and the construction on purpose, because I want the scheme judged on its technical merits first, not its business model. Also, I've laid out in my whitepaper explicit mechanisms to implement revenue sharing with open-source wallets that implement this protocol since revenue is the most difficult part for open source wallets to maintain. Everything above is the implemented design. Critique is welcome, including the harsh kind. Whitepaper can be found on our website [www.btcmedusa.com](http://www.btcmedusa.com) as well as links to our GitHub.

-------------------------

