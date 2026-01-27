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

ajtowns | 2026-01-26 04:46:10 UTC | #2

[quote="bubb1es, post:1, topic:2215"]
The transaction only needs to be greater than the minimum relay size of 82 bytes
[/quote]

The minimum relay size is 65 bytes since [Bitcoin Core 25.0](https://bitcoincore.org/en/releases/25.0/). Here's [a mainnet example of a 75 byte tx](https://mempool.space/tx/cdaf44120c3dd94aae3771e90dcb0059a0c556e82abf9f9c741565275aa68f7a) (13 bytes of which is OP_RETURN data).

Doing an `ANYONECANPAY|ALL` signature spending to a single 0 sat, 3-byte OP_RETURN output of `"ash"` ("ashes to ashes, dust to dust"?) would be enough to ensure you avoid going below the 65 byte limit, and also would allow txs to be combined, for a slight saving in blockspace (23 bytes per input) and a matching slight increase in feerate.

-------------------------

remyers | 2026-01-26 15:49:40 UTC | #3

I like the idea! I can’t think of any way to make it more fee efficient than what @ajtowns suggested.

My only real concern is that people are careful about publishing their dust spends in a way that ensures privacy. It would be counter productive to clean up your dust outputs and then broadcast them from your home IP all at the same time. The same caution must be exercised as for regular spends from dusted addresses.

-------------------------

AdamISZ | 2026-01-26 19:13:32 UTC | #4

[quote="remyers, post:3, topic:2215"]
It would be counter productive to clean up your dust outputs and then broadcast them from your home IP all at the same time.

[/quote]

You make a very good point, but I guess it’s worth mentioning that there’s a big difference between doing this from a full node vs not. Albeit the normal question about spy nodes trying to triangulate applies, right, but then it’d be worth observing that if someone is able to “get through” the full node defence \[1\] and if you’re choosing to use a home, or in any case fixed, IP, you don’t have any strategy to not couple your utxos anyway … again, *if* that attack is active and it works, big if!

(And of course, Tor exists).

\[1\] i don’t know the state of the art on this, I only know that such attacks were outlined, hypothesized, and I vaguely recall, evidenced in the past. We don’t have dandelion but we have other things, right.. not sure but I guess p2p encryption doesn’t really change the threat much.

-------------------------

bubb1es | 2026-01-27 00:51:18 UTC | #5

Thanks for the relay size correction and sighash and OP_RETURN data suggestions. I love giving users de-dusting their wallets the option to help others by combining higher fee-rate de-dusting txs with lower rate de-dusting txs.

I’ve created a simple [“ddust” CLI app](https://github.com/bubb1es71/ddust) that demonstrates how to create these sort of de-dusting transactions. If there’s interest I’ll add the logic to combine unconfirmed de-dust txs.

I also fixed (I hope) my tx size and fee rate estimates:

Scenario 1: P2WPKH

* base size: 65 B = 10 B (overhead) + 41 B (P2WPKH input) + 14 B (OP_RETURN output, “ash” data)
* witness data: 108 B
* virtual size: 92.5 vB
* 294, 300, 325 sats input, fee rate is \~ 3.16, 3.23, 3.49 sats/vB
* [example (signet) ddust tx](https://mempool.space/signet/tx/d1fa15a5f8f3b535682270a83ab7d5fcd9e580065754184147152e6c73c90029)

Scenario 2: P2SH 2-of-3 multisite

* base size: 315 B = 10 B + 295 B + 10 B (OP_RETURN output, no data)
* witness data: 0
* virtual size: 315 vB
* 294, 300, 325 sats input, fee rate is \~ 0.93, 0.95, 1.03 sats/vB

Scenario 3: P2WSH 2-of-3 multisig

* base size: 65 B = 10 B + 41 B + 14 B (OP_RETURN output, “ash” data)
* witness data: 255 B
* virtual size: 129.25 vB
* 294, 300, 325 sats input, fee rate is \~ 2.28, 2.33, 2.52 sats/vB

-------------------------

bubb1es | 2026-01-27 00:24:53 UTC | #6

Thanks! For the tx broadcasting I agree it’s a risk but as @AdamISZ notes broadcast one of these de-dusting tx from a full-node is no worse for privacy than for sending any other transaction. 

In my example [ddust](https://github.com/bubb1es71/ddust) tool I provide an option to broadcast with your (required) local bitcoind node.

-------------------------

