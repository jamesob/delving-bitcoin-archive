# Post Quantum Signatures and Scaling Bitcoin with STARKs

EthanHeilman | 2025-04-07 16:35:48 UTC | #1

PQ (Post Quantum) signatures present a problem for Bitcoin:

1. We can not ignore them. I strongly believe Bitcoin will need to move to PQ signatures in the near future. The rest of this email is premised on this belief.

2. They are large. Of the three proposed in BIP-360 [^0], the smallest is 1.5kb for the public key + signature [^1]. Without a discount this represents a massive reduction in Bitcoin's transaction volume due to the increase in transaction size of Bitcoin payment using such signatures.

3. Even if we discount PQ signatures and public keys so that the maximum number of transactions that can fit in a block is unchanged we still have the problem that these blocks and transactions will be an order of magnitude bigger. If it is the case that we can handle these extra bytes without degrading performance or decentralization, then consider the head room we are giving up that could be used for scalability.

Beyond this there is also the risk that techniques could be developed to encode JPEGs and other data in these discounted PQ signatures or public keys. BIP-360 takes steps to make an abuse of this discount more difficult by requiring that a PQ signature and public key can only be written to the blockchain if they verify. We do not need PQ Signatures to be completely “JPEG resistant”, they just need PQ signatures to not enable significantly cheaper storage than payments. The degree to which the proposed PQ signature algorithms resist being repurposed as a storage mechanism is an open question and worth investigating.

If it turned out PQ signatures could be used to encode data very cheaply, then Bitcoin faces the dilemma that if you discount PQ signatures, you make the JPEG problem worse and may price out the payment use case. If you don't discount PQ, you price out most people from sending payments in Bitcoin since non-PQ witness data can be used for storage.

## Non-interactive Transaction Compression (NTC)

I want to draw the community's attention to a solution that could not only address these problems but also increase Bitcoin’s scalability (and privacy):

Non-interactive Transaction Compression (NTC) for Transactions supporting PQ signatures. This is sometimes called Non-Interactive Witness Aggregation (NIWA) [^2].

This would require a new transaction type supporting PQ signatures. The miner of a block would then pull out the signatures and hash pointers from transactions to compress transaction data and non-interactively aggregate all the PQ signatures in all the transactions in a block, replacing them with one big STARK (STARKS are a form of SNARK which is PQ). This would make PQ signatures significantly smaller and cheaper than ECDSA and Schnorr signatures.

Consider the following back of the envelope math:

$2$ bytes per Input $= 2$ bytes per TXID, $0$ bytes per signature

$37$ bytes per output $= 32$ bytes pubkey hash + $5$ bytes value (max 2.8m BTC per output)

1-input-2-output transaction would be: $2 + 2\times37 = 76$ bytes

$(4,000,000/76)/(60\times10) = ~87$ txns/sec

You could shave some bytes off the value, or add some bytes to the TXID. [^3] provides a more detailed estimate, proposing 113.5 weight units (WU) for a 1-input-2-output transaction with no address reuse. However it does not consider TXID compression. An account-based model would take this even further to 12 bytes per transaction per block [^4]. This would enable approximately $4,000,000/(12\times60\times10) = 555$ txns/second.

A secondary benefit of having on-chain PQ payments only be ~76 bytes in size is that it fundamentally changes the pricing relationship between payments and on-chain JPEG/complex contracts. The problem with on-chain JPEGs is not that they are possible, but that they are price competitive with payments. At ~76 bytes per payment or better yet ~76 bytes per LN channel open/close, JPEGs no longer present the same fee competition to payments as payments become much cheaper.

Such a system would present scaling issues for the mempool because prior to aggregation and compression, these transactions would be 2kb to 100kb in size and there would be a lot more of them. It is likely parties producing large numbers of transactions would want to pre-aggregate and compress them in one big many input, many output transactions. Aggregating prior to the miner may have privacy benefits but also scalability benefits as it would enable cut-throughs and very cheap consolidation transactions. ~87/txns a second does not include these additional scalability benefits.

### Consolidation and Cut-throughs

Consider an exchange that receives and sends a large number of transactions. For instance between block confirmations customers send the exchange 10 1-input-2-output transactions in deposits and the exchange sends out 10 1-input-2-output transactions in withdrawals. The exchange could consolidate all of the outputs paying the exchange, including chain outputs, into one output and do the same for inputs. This would reduce not just size, but also validation costs.

$$

(10 \times 2 + 20 \times 2 \times 37) + (10 \times 2 + 20 \times 2 \times 37) = 3000 \text{ bytes}

$$

becomes

$$

(10 \times 2 + 11 \times 2 \times 37) + (2 + 11 \times 2 \times 37) = 1650 \text{ bytes}

$$

### Parallelizing and Distributing Proof Construction

If constructing these proofs turned out to be as expensive as performing POW, it would make block generation not progress free. Essentially you'd have a two step POW: proof generation and then the actual POW. Such a scenario would be very bad and cause the biggest miner to always be the one that generates blocks. A critical assumption I am making is that such proof generation is not particularly expensive in the scheme of POW. I am optimistic that proof generation will not be this expensive for two reasons

There are PQ signature schemes which support non-interactive aggregation such as LaBRADOR [5]. Thus, the STARK wouldn’t need to perform the block-wide signature aggregation and would only need to perform transaction compression, cut throughs and consolidation.

We could make use of recursive STARKs [^6] to allow miners to parallelize proof generation to reduce latency or to decentralize proof generation. Users creating transactions could perform non-interactive coinjoins with other users or settlement/batching. This would not only take proof generation pressure off of the miners and reduce the strain on the mempool but in some circumstances it would provide privacy if used with payjoin techniques like receiver side payment batching [^7].

The approach we are proposing treats the STARK the miner produces as free from a blocksize perspective. This is important for bootstrapping because it means that fees are significantly cheaper for a transaction, even if it is the only compressed transaction in the block. This encourages adoption. Adoption helps address the chicken and egg problem of wallets and exchanges not investing engineering resources to support a new transaction type if no one is using it and no one wants to use it because it isn't well supported. By having a single format, built into the block we both accelerate the switch over and prevent a fragmented ecosystem that might arise from doing this in Bitcoin script. Fragmentation reduces the scalability benefits because validators have to validate multiple STARKs and reduces the privacy benefits because there are many coinjoins, rather than each being a coinjoin.

Even if our approach here turns out to be infeasible, we need a way to reduce the size of PQ signatures in Bitcoin. The ability to move coins, including the ability to move coins that represent JPEGs, is the main functionality of Bitcoin. If we make storage/JPEG too price competitive with the ability to transfer coins, we destroy that essential functionality and decrease the utility of Bitcoin for everyone. Currently moving coins securely requires at least one 64 byte signature, which is an unfortunate tax on this most vital of all use cases. I believe removing that tax with signature aggregation will be beneficial for all parties.

## Closing Thoughts

Consider the world of PQ signatures in Bitcoin without STARKs:

- The large size of PQ signatures will make it more expensive for users to use them prior to the invention of a CRQC (Cryptographically Relevant Quantum Computer). This means that most outputs will not be protected by PQ signatures. Once a CRQC arises there will be a rush to move funds under the protection of PQ signatures but due to the large size of PQ signatures the fees will be too expensive for most outputs. Users will instead need to move their funds to centralized custodial wallets that can use a small number of outputs. In such a world it will be much harder and expensive to self-custody.

- Without a solution here the large sizes of PQ signatures will limit Bitcoin's functionality to move coins using on-chain payments. This will also favor centralized custodians and erode the decentralized nature of Bitcoin.

None of this is an argument against adopting BIP-360 or other PQ signatures schemes into Bitcoin. On the contrary, having PQ signatures in Bitcoin would be a useful stepping stone to PQ transaction compression since it would allow us to gain agreement on which PQ signature schemes to build on. Most importantly, in the event of a CRQC being developed it will be far better to have uncompressed PQ signatures in Bitcoin than none at all.

Acknowledgements:

These ideas arose out of correspondence with Hunter Beast. I want to thank Neha Narula, John Light, Eli Ben-Sasson for their feedback, Jonas Nick for his feedback and his idea to use LaBRADOR for signature aggregation, Tadge Dryja for suggesting the term “JPEG resistance” and his ideas around its feasibility. I had a number of fruitful discussions over lunch with members of the MIT DCI and on the Bitcoin PQ working group. These acknowledgements should not be taken as an agreement with or endorsement of the ideas in this email.

This originally appeared on the bitcoin-dev mailinglist as [Post Quantum Signatures and Scaling Bitcoin (Apr 4, 2025)](https://groups.google.com/g/bitcoindev/c/wKizvPUfO7w) and is also on my homepage as [Some Thoughts on Post Quantum Signatures and Scaling Bitcoin](https://www.ethanheilman.com/x/32/index.html). 

[^0]: Hunter Beast, BIP-360: QuBit - Pay to Quantum Resistant Hash (2025) https://github.com/bitcoin/bips/pull/1670/files#

[^1]: Benchmark Report: Post-Quantum Cryptography vs secp256k1 https://github.com/cryptoquick/libbitcoinpqc/blob/0dc6f7e7f09b2c8cbedea3ad2705e423d6716425/benches/REPORT.md

[^2]: Ruben Somsen, SNARKs and the future of blockchains (2020) https://medium.com/@RubenSomsen/snarks-and-the-future-of-blockchains-55b82012452b

[^3]: John Light, Validity Rollups on Bitcoin (2022) https://github.com/john-light/validity-rollups/blob/fecc3433d8e95ab40263db8c2941794e0115d2d5/validity_rollups_on_bitcoin.md

[^4]: Vitalik Buterin, An Incomplete Guide to Rollups (2021)

https://vitalik.eth.limo/general/2021/01/05/rollup.html

[^5]: Aardal, Aranha, Boudgoust, Kolby, Takahashi, Aggregating Falcon Signatures with LaBRADOR (2024) https://eprint.iacr.org/2024/311

[^6]: Gidi Kaempfer, Recursive STARKs (2022) https://www.starknet.io/blog/recursive-starks/

[^7]: Dan Gould, Interactive Payment Batching is Better (2023) https://payjoin.substack.com/p/interactive-payment-batching-is-better

[^8] John Tromp, Fee burning and Dynamic Block Size (2018) https://lists.launchpad.net/mimblewimble/msg00450.html

-------------------------

