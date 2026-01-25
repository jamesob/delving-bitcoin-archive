# Disposing of "dust attack" UTXOs

bubb1es | 2026-01-25 17:20:49 UTC | #1

Hi, I’d like to find a standard/preferred way to dispose of “dust attack” UTXOs in on-chain wallets. In particular the dust I’m looking to get rid of are the ones created by an adversary to trick wallet users into revealing common ownership between otherwise unrelated UTXOs. This happens when the adversary sends dust amount UTXOs to all the anonymous on-chain addresses they are interested in and hope that some of these will be unintentionally spent together with an unrelated UTXO. (see https://bitcoinops.org/en/topics/output-linking/ )

The best data I could find on the distribution of output sizes in the UTXO set is from https://research.mempool.space/utxo-set-report/ . But this report focuses on spam meta protocols and doesn’t break out details for dust attack UTXOs which are generally at or around the core dust limit.

Most modern wallets automatically lock dust amount UTXOs so they are never spent. This solves the problem but has a cost and future risks. The cost is the bloating of the mempool with UTXOs that will never be spent. Risks are that a wallet software bug, restore from keys, or migration to a new wallet could “unlock” the dust. There’s also a risk that a future wallet owner (such a with inheritance) could misunderstand the reason the dust is locked and spend it. In any case there are risks in the future of accidentally de-anonymizing the wallet.

Now that the default minimum relay fee rate (for core 30) has been lowered to 0.1 sats/vb it should be possible to spend typical dust UTXO by creating a transaction that uses the entire amount for fees and has an OP_RETURN output. The transaction only needs to be greater than the minimum relay size of 82 bytes and have a fee > 0.1 sats/vb.

Scenario 1:
P2WPKH: 10.5 vb (overhead) + 68 vb (input) + 5 vb (OP_RETURN output) =  83.5 vb
250, 300, 350 sats input, fee rate is \~ 2.99, 3.59, 4.19 sats/vb.
[testnet4 example](https://mempool.space/testnet4/tx/ee3d1dc998c581b7a7f22e31d4f9cd166f3788210c0356f9bd7da1f16096f3b4) (I think mempool’s size calc is wrong)

Scenario 2:
P2SH 2-of-3 multisig: 10 vb + 297 vb + 1 vb = 308.5 vb
250, 300, 350 sats input, fee rate is 0.81, 0.97, 1.13 sats/vb

Scenario 3:
P2WSH 2-of-3 multisig: 10.5 vb + 104.5 vb + 1 vb =  116 vb
250, 300, 350 sats input, fee rate is 2.16, 2.59, 3.02 sats/vb

(thanks https://bitcoinops.org/en/tools/calc-size/ )

A few risks with implementing this feature in wallets are:

1. if only one or a few wallets support it then that “fingerprints” the user.
2. if multiple dust Tx are broadcast for a wallet at the same time that could correlate them.
3. if fee rates go up these Tx may need to be re-broadcast when rates come back down.
4. signing for dust Tx could be confusing/annoying to multisig and hardware signing device users.

Comments and suggestions welcome! I’m especially interested in hearing from wallet or wallet library developers if this is something they would consider supporting.

-------------------------

