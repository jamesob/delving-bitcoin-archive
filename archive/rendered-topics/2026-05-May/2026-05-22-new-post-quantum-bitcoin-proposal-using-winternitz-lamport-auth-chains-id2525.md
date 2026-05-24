# New Post Quantum Bitcoin Proposal using WInternitz + Lamport auth chains

opus-lux | 2026-05-22 01:04:56 UTC | #1

Hello bitcoin community!

I am a cryptographic researcher who has been working on creating the lightest weight possible post quantum signature scheme for bitcoin. This is not just an idea but a working implementation that is currently live on 16 different EVM chains. The website URL is block_opuslux.ar.io, there you can read the security essay that explains in detail how signing with Winternitz + Lamport works along with my bitcoin proposal outline. The project along with the proposals will be made open source very soon. The contracts will be verified on chain within 24 hours.

Now that we have that out of the way let me break down why I think that Lamport + Winternitz is the right solution for creating a post quantum bitcoin. Nearly every post quantum signature that exists today uses Winternitz at its core. The problem with using winternitz signatures is that they are inherently one time signatures. This creates a problem, if you want to use Winternitz as your post quantum signing method you must build a system to make sure that the keys are one time only! This is exactly what I did using a Lamport authorization chain and some cryptography that starts from your seed phrase. Each link in the Lamport authorization chain authorizes the upload of one Winternitz public key. Then the Winternitz public key can be used to verify the winternitz signature for the message. Since Winternitz is one of the smallest possible quantum signatures it is extremely lightweight meaning that it can be added easily through Taproot. Also adjusting the w parameter in the winternitz equation to 256 instead of 16 means that the signatures are smaller and more lightweight than anything else I have seen. I would love to get some feedback on this idea and make my bitcoin proposal public. You can reach me at opusluxofficial@proton.me or opus-lux on github! Thank you for taking the time to consider this option! 

Best, Opus Lux !

-------------------------

murch | 2026-05-23 16:09:30 UTC | #2

Hi opus-lux,
thanks for sharing your idea. It sounds potentially interesting, but I don't understand your proposal well-enough, yet. Is there a more comprehensive description somewhere?

[quote="opus-lux, post:1, topic:2525"]
Each link in the Lamport authorization chain authorizes the upload of one Winternitz public key. Then the Winternitz public key can be used to verify the winternitz signature for the message.

[/quote]

I'm not sure I fully understand this part. Would output scripts based on this construction commit to exactly one Winternitz public key, or would they commit to the whole authorization chain? The former would be an issue, as we need to be able to produce more than one signature for an output script, e.g., when our first transaction attempt does not succeed, we want to be able to make a higher feerate transaction, or when we create transactions with inputs from multiple users and a participant defects.

-------------------------

opus-lux | 2026-05-23 21:03:26 UTC | #3

HI Murch,

Thanks for taking time to read my idea and respond. To answer your question regarding the authorization chain it works like this. At wallet setup, you build a Lamport chain which is a sequence of hashes rooted in your master secret. Imaginine hashing your secret to get value #1, then hashing the result of that to get value #2, and so on until you hit 50,000. You keep all of the values private except for value number 50,000 which you publish on chain. Now to verify your Winternitz public key upload you reveal value 49,999, the blockchain checks does value 49,999 hash to get 50,000. If it does then your Winternitz public key upload is authenticated. The Winternitz public key is only valid to sign one message, then its retired forever.

Also, yes there is a detailed security essay along with a bitcoin proposal outline that explain exactly how this works and fits into the existing Tapscript framework.

Security essay:  block_opuslux.ar.io/#/essay

Bitcoin outline: block_opuslux.ar.io/#/bitcoin-proposal

Please let me know if you have any other questions or want to know more! I am happy to explain this in as much detail as you want. Thanks again for taking the time to read my ideas and respond!

-------------------------

murch | 2026-05-23 22:31:11 UTC | #4

If I understand it right, there are multiple issues that I would consider problems for Bitcoin adoption:
- It sounds like it would require every node would need to keep track of all Lamport chains and all nullifiers. This would imply linear growth of the minimal state stored by every full node in the number of payments.
- The "Tapscript Leaf Structure" seems to commit to exactly one Winternitz public key. This would mean that for every UTXO we could only make a single attempt to spend it. We would be incapable of bumping fees, and would generally become unable to participate in multi-user transactions as any other participant could force us to sign more than once.
- It seems to me that this scheme reveals separate UTXOs as being owned by the same recipient upon spending. That would be a significant departure from the usual privacy expectations in Bitcoin --- address reuse is generally considered malpractice on Bitcoin.

-------------------------

opus-lux | 2026-05-24 00:36:42 UTC | #5

Excellent reply let me address each point.

1) State growth concern (Partially valid)

Your concern is correct on the mechanics: the nullifier set grows monotonically at 32 bytes per WOTS-39 spend and cannot be pruned the way the UTXO set prunes spend outputs. This is an engineering tradeoff, not a design flaw. The UTXO set, which every node already maintains, is \~5-8 GB and grows with every unspent output. The nullifier set grows at 32 bytes per WOTS-39 spend. At 10% adoption of today's transaction volume (\~300K tx/day), that's roughly 350 MB/year which is modest compared to the existing state burden. You are right that there is a concrete optimization path not yet in my spec: since each Lamport chain has a fixed length N committed in the genesis OP_RETURN, a node can detect when the final slot is spent and prune all N nullifier entries plus the genesis entry in a single atomic step. When the chain is exhausted no future spend using that genesis is valid. This would make the nullifier set self-pruning per chain, bounding live state to 0(active_chains \* N) rather than 0(all-time spends). This is worth adding to the BIP for node optimization.  For comparison: ZCash has had a monotonically growing nullifier set since 2016 and the network functions without issue.

2) One time use / multi user transactions (Misunderstanding?)

WOTS-39 is implemented as a Tapscript leaf inside a P2TR output. Standard Taproot outputs have two independent spending paths. 

1- Key path: a direct Schnorr signature against the tweaked internal key. No script executed. Standard Bitcoin privacy and flexibility

2- Script path: one of the committed tapscript leaves, which is where our OP_WOTS_VERIFY lives.

A WOTS-39 UTXO can always be spent via the Schnorr key path. The user never has to touch the WOTS script path unless they specifically want post-quantum security on that spend. This means:

* RBF / fee bumping: Sign a replacement transaction with the Schnorr key. No WOTS involvement, exactly like Bitcoin works today.
* \-CoinJoin / multi-user transactions: Participate via key path Schnorr signature. Other participants never see WOTS structure. PSBT flows unchanged
* Lightning: Key path Schnorr. Unaffected

The one-time-use constraint only applies to the WOTS script path. One-time use is a fundamental mathematical property of all Winternitz-family schemes, not a flaw in this implementation. Every WOTS-39 UTXO has exactly one quantum secure spend available (via the script path), and unlimited classical spends available (via the key path). So to directly answer your question users participate in multi-user transactions using Schnorr without WOTS signatures, and thats by design not a workaround.

3- Privacy / Linkability (valid, wallet layer concern)

Your concern is real, when you spend a WOTS-39 UTXO via the script path, the witness reveals genesis_txid and root_authTip. Any two spends from the same authorization chain reveal the same pair of values, allowing a chain-analysis observer to link them to the same entity. This is analogous to Bitcoin address reuse which is considered malpractice for the same reason. 

With this comment you've identified the most important flaw in my current proposal. You're correct that the shared genesis_txid model links UTXO's at spend time, and that SPINCS/XMSS proposals don't have this problem, their privacy model is identical to Bitcoin's HD wallet standard. 

So, I will be revising my BIP to decouple key derivation from the UTXO outpoint. The fix is to replace the outpoint-derived anchor in key generation with a pre-computable per UTXO nonce, a value that the wallet can generate deterministically before the funding transaction exists. The UTXO outpoint is still computed at spend time, but only for the nullifier, not for key derivation. 

Why this change would achieve full privacy?

At spend time, the script path reveals: schnorr_xonly_i wots_nonce wotsPK_i.  Since wots_nonce_i = SHA256(mskWOTS || "wots39-nonce-v1" || i) and mskWOTS is the wallet's master secret, the nonces are computationally indistinguishable from random 32-byte values to any outside observer. Two different UTXOs reveal two different nonces with no mathematical relationship visible on-chain. There is no shared genesis_txid, no shared root_authTip, no linkage whatsoever. This would be identical to how SPHINCS+ and XMSS achieve privacy: fresh independent key material per UTXO, committed in the output script, no external registration event. 

If we remove the genesis commit, then how would we verify Winternitz key uploads?

The Tapscript commitment is the key upload. When the funding transaction creates a UTXO whose tapscript commits to wotsPK_i, that is the registration of that WOTS key. It happens at UTXO creation time. No prior genesis transaction is needed, for the same reason that creating a P2TR output doesn't require pre-registering your Schnorr key. 

I will be revising my BIP while keeping the core innovation intact. The genesis commitment machinery will be removed because it was solving a problem that bitcoin doesn't have. The result will be a simpler, more private, and more Bitcoin-native design that matches the privacy model of every other serious PQC Bitcoin Proposal.

Thank you murch for bringing the core issue with my current proposal to my attention. I will work on revising my BIP to work with Bitcoin's privacy model and update the community with my revision. I think Winternitz sill holds great promise as a lightweight quantum secure signature scheme for Bitcoin. Thanks again for your thoughtful consideration and helping me make improvements to my proposal! Your feedback is deeply appreciated!

Best, Opus Lux!

-------------------------

