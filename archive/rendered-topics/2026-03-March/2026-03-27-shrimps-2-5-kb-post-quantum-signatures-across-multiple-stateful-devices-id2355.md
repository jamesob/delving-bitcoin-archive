# SHRIMPS: 2.5 KB post-quantum signatures across multiple stateful devices

jonasnick | 2026-04-03 14:46:49 UTC | #1

**Abstract**: [SHRINCS](https://delvingbitcoin.org/t/shrincs-324-byte-stateful-post-quantum-signatures-with-static-backups/2158) achieves very small hash-based signatures using a stateful signer while still allowing for static backups. However, its efficient stateful path requires transferring state to any new device, which is error-prone, so in practice any restored or secondary device will typically fall back to large stateless signatures. SHRIMPS removes this single-device constraint. In settings where each key is used for only a small number of signatures (as is typical in Bitcoin), a static seed backup can be loaded into many independent stateful signing devices, each producing a ~2564-byte signature at 128-bit security. The construction requires an upper bound on the number of device initializations; with a conservative bound of $n_\text{dev} = 2^{10}$, SHRIMPS signatures are up to three times smaller than SLH-DSA (7856 bytes). SHRIMPS can be combined with SHRINCS: the primary device produces ~324-byte signatures, while any backup device produces signatures under 3 KB.

## Basic SHRIMPS

In SPHINCS+, the parameter $q_s$ bounds the number of signatures that can be securely produced under a single key. Smaller $q_s$ allows smaller signature sizes. The construction of SHRIMPS combines two SPHINCS+ instances under a single public key: a compact instance with $q_s = n_\text{dev}$ where $n_\text{dev}$ is an upper bound on the number of device initializations, and a fallback instance with sufficiently large $q_s$ (e.g., $2^{40}$ or $2^{64}$). The SHRIMPS public key is a hash of the public keys of both instances.

```text
             ┌────────────────────────────┐
             │         Public Key         │
             └──────┬─────────────┬───────┘
                    │             │
         ┌──────────▼──────┐  ┌───▼───────────────┐
         │  Compact path   │  │  Fallback path    │
         │  SPHINCS+ with  │  │  SPHINCS+ with    │
         │  q_s = n_dev    │  │  large q_s        │
         └─────────────────┘  └───────────────────┘
```

A signing device is initialized by loading the seed, which deterministically derives both SPHINCS+ key pairs. To sign, the device looks up its persistent state for this key to determine whether it has signed before: if not, it signs through the compact instance and updates its state; otherwise, it signs through the fallback instance.

A SHRIMPS signature consists of a SPHINCS+ signature under the selected instance and the public key of the other instance (16 bytes[^1]). The verifier reconstructs the signing instance's public key from the SPHINCS+ signature, hashes both public keys to reconstruct $\mathit{pk}$, and compares it to the known public key. The 16-byte sibling public key is the only overhead beyond a standard SPHINCS+ signature.

[^1]: A SPHINCS+ public key consists of a public seed ($\mathit{PK.seed}$, 16 bytes) and a hypertree root ($\mathit{PK.root}$, 16 bytes). Both SHRIMPS instances share the same $\mathit{PK.seed}$, so the sibling only needs to include $\mathit{PK.root}$.

Since each device signs at most once through the compact instance, the total number of compact-path signatures across all devices is at most $n_\text{dev}$, which is exactly the number of signatures the compact instance is parameterized to support.

More generally, the compact instance can allow each device $n_\text{dsig}$ signatures before switching to the fallback, at the cost of increasing $q_s$ to $n_\text{dev} \cdot n_\text{dsig}$. In Bitcoin, keys are commonly used for only a few signatures, so a small $n_\text{dsig}$ keeps most signatures on the compact path.

In Bitcoin wallet setups, initializing a device with a seed is typically a manual process that happens rarely. $n_\text{dev} = 2^{10} = 1024$ is conservative; it is hard to imagine importing a single seed into more than a thousand devices. A device that loses its state and re-initializes from the seed will use the compact path again, consuming an additional signature from the compact instance's $q_s$ budget.

The fallback instance can be any SPHINCS+ parameterization with sufficiently large $q_s$. Using SLH-DSA (SPHINCS+ with $q_s = 2^{64}$), the fallback signature is 7856 bytes; using a SPHINCS+ variant with $q_s = 2^{40}$ from [Hash-based Signature Schemes for Bitcoin](https://eprint.iacr.org/2025/2203), it is less than 4.5 KB.

The following table shows selected compact-path parameter sets, where sizes are shown as the SPHINCS+ signature size plus the 16-byte sibling public key.

| $q_s$    | Parameters                                                           | Size + 16 | Sign cost |
|:---------|:---------------------------------------------------------------------|----------:|----------:|
| $2^{5}$  | W+C P+FP $(k, a, h, d) = (10, 12, 12, 1)$, $w = 16$, $S_{w,n} = 240$ |    2324 B |      2.5M |
| $2^{10}$ | W+C P+FP $(k, a, h, d) = (8, 17, 12, 1)$, $w = 16$, $S_{w,n} = 240$  |    2564 B |      6.8M |
| $2^{10}$ | W+C P+FP $(k, a, h, d) = (12, 12, 12, 1)$, $w = 16$, $S_{w,n} = 240$ |    2708 B |      2.4M |
| $2^{12}$ | W+C P+FP $(k, a, h, d) = (10, 14, 14, 1)$, $w = 16$, $S_{w,n} = 240$ |    2628 B |      9.9M |
| $2^{12}$ | W+C P+FP $(k, a, h, d) = (12, 13, 12, 1)$, $w = 16$, $S_{w,n} = 240$ |    2884 B |      2.7M |
| $2^{14}$ | W+C P+FP $(k, a, h, d) = (8, 17, 16, 1)$, $w = 16$, $S_{w,n} = 240$  |    2580 B |     41.0M |
| $2^{14}$ | W+C P+FP $(k, a, h, d) = (10, 15, 14, 1)$, $w = 16$, $S_{w,n} = 240$ |    2772 B |     10.9M |
| $2^{14}$ | W+C P+FP $(k, a, h, d) = (10, 12, 22, 2)$, $w = 16$, $S_{w,n} = 240$ |    3000 B |      2.5M |

Sign cost is in SHA-256 compression calls. For comparison, SLH-DSA ($q_s = 2^{64}$) produces 7856-byte signatures at 2.3M compression calls. Verification costs 0.30 compressions per signature byte; the $d = 1$ parameter sets above achieve ~0.19 (about 35% lower), while the $d = 2$ sets achieve ~0.25. Each row can be reproduced using the `--params` option of `costs.sage` in [SPHINCS-Parameters](https://github.com/BlockstreamResearch/SPHINCS-Parameters) (commit `f2ea2a2`):
```text
sage costs.sage --params <scheme> <q_s> <k> <a> <h> <d> <w> <S_wn>
```
For example, the second row: `sage costs.sage --params W+C_P+FP 10 8 17 12 1 16 240`

## State management

The compact path requires per-key state: the device stores a counter of compact-path signatures made ($\lceil\log_2(n_\text{dsig}+1)\rceil$ bits).

With key derivation (similar to BIP-32) from a single seed, each derived key is a separate SHRIMPS instance. The device must maintain this state for every derived key, or store a single bit per key indicating that the fallback path should be used.

If the number of compact-path signatures exceeds the $q_s$ budget (for example, because more devices were initialized than anticipated, or because a device failed to update its state), security does not break down immediately. Instead, it degrades gradually. The following table shows how security decreases for the $(k, a, h, d) = (8, 17, 12, 1)$ parameter set as the total number of compact-path signatures grows beyond the $q_s = 2^{10}$ budget:

| Total compact-path signatures | Security   |
|:------------------------------|:-----------|
| $2^{10}$ (budget)             | 128.0 bits |
| $2^{11}$                      | 128.0 bits |
| $2^{12}$                      | 125.1 bits |
| $2^{13}$                      | 120.4 bits |
| $2^{14}$                      | 115.0 bits |
| $2^{15}$                      | 108.9 bits |

As an aside, statefulness is a strong assumption, but the associated risk is localized to individual wallets that mismanage their state. By contrast, alternative post-quantum schemes carry systemic risks: new cryptographic assumptions (lattices, isogenies) or larger signatures.

## Discussion

If we assume stateful wallets and can bound the number of device initializations, SHRIMPS compact-path signatures are smaller than SLH-DSA at lower verification cost and comparable signing cost. With $n_\text{dev} = 2^{10}$ and $n_\text{dsig} = 1$, signatures are 2564 bytes, about three times smaller than SLH-DSA's 7856 bytes. Increasing $n_\text{dsig}$ to allow more signatures per device costs additional bytes and signing time, but even at $n_\text{dsig} = 2^4$ the signature size remains under 3000 bytes.

SHRINCS already assumes stateful wallets, so SHRIMPS can be combined with SHRINCS: the primary device uses SHRINCS's efficient stateful path (~324-byte signatures), while any backup device uses the SHRIMPS compact path instead of falling back to full stateless signatures.

A comparison of key generation costs is left to future work. The parameter sets in this post all constrain verification cost per byte to be at most that of SLH-DSA (~0.30 compressions per byte). Relaxing this constraint (for example, allowing verification cost comparable to Schnorr signature verification per byte) would permit $w = 256$, potentially yielding even smaller signatures. We leave this exploration to future work as well.

---

*Thanks to Mikhail Kudinov and Oleksandr Kurbatov for the discussions that led to SHRIMPS and their feedback on earlier versions of this post.*

-------------------------

niklaus | 2026-04-01 19:12:10 UTC | #2

SLH-DSA signatures under the SLH-DSA-{SHA2|SHAKE}-128s parameter sets are 7,856 bytes (not 7,872).

Is there any reason you don’t use the optimizations (e.g. WOTS+C, PORS+FP) in the fallback SPHINCS+ instance but you do use it in the compact path?

-------------------------

conduition | 2026-04-02 16:26:45 UTC | #3

Cool idea @jonasnick.

@niklaus I think Jonas is not suggesting necessarily that we should mix SLH-DSA with SPHINCS+C - just that either are options. His proposal is more about the structure: Combining two SPHINCS keys, one with low $q_s$ as a primary, and one with high $q_s$ as fallback.

@jonasnick I have a concern about your assumption here: 

[quote="jonasnick, post:1, topic:2355"]
With key derivation (similar to BIP-32) from a single seed, each derived key is a separate SHRIMPS instance. The device must maintain this state for every derived key, or store a single bit per key indicating that the fallback path should be used.
[/quote]

If we want to have efficient wallet standards which provide drop-in replacement for BIP32, this actually can't be the case, since hash-based keys cannot be rerandomized. See [this mailing list post](https://groups.google.com/g/bitcoindev/c/5tLKm8RsrZ0).

If someday we are using hash-based signatures for everyday transactions (a pretty bleak future) then sure, this might hold - In that world, unhardened derivation wouldn't exist, and every change and receiving address would use a fresh hardened child key.

However, for the purposes of providing fallback quantum security to some primary scheme like ECC, this probably would not hold. 

How would you see SHRIMPS being used in that context?

-------------------------

jonasnick | 2026-04-03 15:08:57 UTC | #4

> SLH-DSA signatures under the SLH-DSA-{SHA2|SHAKE}-128s parameter sets are 7,856 bytes (not 7,872).

Fixed, thanks!

> Is there any reason you don’t use the optimizations (e.g. WOTS+C, PORS+FP) in the fallback SPHINCS+ instance but you do use it in the compact path?

No particular reason. There's quite a range of parameters that you could use for the fallback. If $q_s = 2^{30}$  is fine for your application, you can get 3440 byte signatures (WOTS+C, PORS+FP).

-------------------------

jonasnick | 2026-04-06 08:30:15 UTC | #5

@conduition Sorry, I don't understand the concern. I am merely stating the fact that you need to store state for each keypair you derive from the seed. To clarify, "similar to BIP-32" was just referring to the general pattern of deriving multiple keys from a single seed, not claiming compatibility with unhardened derivation.

-------------------------

conduition | 2026-04-06 15:17:27 UTC | #6

apologies, I did a leap of logic there which I failed to properly explain. My concern is about the viability of the design for our use cases, not about the design of the scheme itself. After thinking about it, I realized it applies to SHRINCS as well, though it is not as severely in that case. 

Let me rephrase and elaborate: 

First, note SHRIMPS' signature size savings are only meaningful relative to SPHINCS if a keypair is used at most once (disregarding restored wallets for the moment), because after the first signature a signer _must_ switch to the stateless SPHINCS key. On the other hand, SHRINCS' signature savings are still meaningful if reused, but degrade linearly with usage.

Second, note most wallets receive more than one UTXO using distinct addresses, and if you want to spend $n$ UTXOs, you need $n$ distinct signatures (unless we add something new to consensus). Therefore, we should expect when and if SHRIMPS is needed - when ECC becomes vulnerable - a single wallet that uses SHRIMPS as a fallback will likely need to "rescue" multiple UTXOs by creating _multiple_ SHRIMPS signatures. 

Now for my concern, which is about viability as a fallback signature scheme with the above two properties in mind. 

By designing SHRIMPS (and to some degree, SHRINCS) with the optimistic assumption that a signing wallet will need to sign with a SHRIMPS (or SHRINCS) keypair **only once** under most use-cases, then we imply the assumption that every address in that wallet must commit to a unique SHRIMPS (or SHRINCS) pubkey. This hobbles our ability to create wallets which support some quantum-resistant version of unhardened key derivation. 

More materially, wallets will have a hard design choice to make:

- If we want to support unhardened key derivation with a hash-based fallback key, then we'd likely need to reuse a SHRIMPS keypair to spend many UTXOs, which mostly nullifies the performance savings of SHRIMPS.
- If we want smaller signatures, then we must commit each address in our wallet to a unique SHRIMPS public key, which necessarily forbids unhardened derivation of those addresses.

Interestingly, there's no reason we can't do both in different circumstances. E.g: a watch-only wallet needs unhardened derivation to work, which they take at the cost of PQ signature size, where as a software wallet which has private keys in memory can do hardened derivation all the time, and create a unique SHRIMPS key per address.

@jonasnick What's your take on the tradeoffs here?

-------------------------

