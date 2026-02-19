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

bubb1es | 2026-01-30 16:03:43 UTC | #7

:waving_hand: @0xB10C I'm also trying to determine the nature and number of obvious dust attack UTXOs, do you know if there’s anyone in your new NOC group also looking into this?

-------------------------

0xB10C | 2026-01-30 18:20:30 UTC | #8

I'm not aware of anyone looking into it, no. Congratulations, you are now the person looking into it! :slight_smile: 


You can use the `dumptxoutset` RPC and e.g. this script

https://github.com/bitcoin/bitcoin/blob/01651324f4e540f6b96cff31e89752c3f9417293/contrib/utxo-tools/utxo_to_sqlite.py 

to export the current UTXO set. From there, it's probably just a few SQL queries to figure out how many close-to-dust UTXOs remain and when they were created.

There is also a PR for a utxo to CSV script open. If you end up using it, feedback on the PR is surely welcome.

https://github.com/bitcoin/bitcoin/pull/34324

-------------------------

bubb1es | 2026-01-31 22:51:19 UTC | #9

Thanks for the vote of confidence! I’ve written my own [tool](https://github.com/bubb1es71/dusts) to parse the utxodump data and already have some preliminary results. Hope this stimulates some interest and discussion on the topic.

![dusts-data|640x480](upload://idyDTjt1KeUbANbNyWaCQDqXimx.png)

EDIT:

The cumulative graph by script type is more interesting.

![dusts-data|640x480](upload://5oCZEJ4i4R3G7IZpQvZhLwMQxkg.png)

-------------------------

andrewtoth | 2026-02-02 15:52:36 UTC | #10

[quote="remyers, post:3, topic:2215"]
It would be counter productive to clean up your dust outputs and then broadcast them from your home IP all at the same time.

[/quote]

[quote="AdamISZ, post:4, topic:2215"]
We don’t have dandelion

[/quote]

Just wanted to point out that Private Broadcast (https://github.com/bitcoin/bitcoin/pull/29415) will be released in Bitcoin Core v31. It solves this privacy issue, obviating the need for dandelion.

-------------------------

sipa | 2026-02-02 16:16:58 UTC | #11

[quote="andrewtoth, post:10, topic:2215"]
It solves this privacy issue, obviating the need for dandelion.
[/quote]

I think that's an exaggeration, and while they're both privacy improvements, they're also partially orthogonal. Dandelion improves transaction relay privacy in general across the network. Private broadcast only improves the first hop, and does so by relying on privacy networks, which may not be accessible to everyone, and bring their own trade-offs.

-------------------------

andrewtoth | 2026-02-02 17:58:28 UTC | #12

[quote="sipa, post:11, topic:2215"]
they’re also partially orthogonal

[/quote]

I don’t see how they’re orthogonal. They both have the same goal of removing the link between a transaction and the originator’s IP (or persistent Tor/I2P address). Is there another goal that we wish to achieve that I am missing?

[quote="sipa, post:11, topic:2215"]
Dandelion improves transaction relay privacy in general across the network. Private broadcast only improves the first hop

[/quote]

If the first hop is done through a private broadcast, then it achieves the above goal.
I’m not sure how you mean “in general across the network” to be an improvement. Dandelion requires multiple hops and other nodes to be honest throughout the hops to achieve the same goal.

-------------------------

sipa | 2026-02-02 20:49:23 UTC | #13

[quote="andrewtoth, post:12, topic:2215"]
I don’t see how they’re orthogonal. They both have the same goal of removing the link between a transaction and the originator’s IP (or persistent Tor/I2P address). Is there another goal that we wish to achieve that I am missing?
[/quote]

No, you're right. In my mind Dandelion had privacy advantages beyond hiding transaction origin, but reading material from the time it was proposed, that doesn't seem to be the case.

> If the first hop is done through a private broadcast, then it achieves the above goal.

If that's available and reliable, yes. But I don't think that's true in general. Tor/I2P may not be available in all environments, or desirable as it relies on somewhat centralized directories. I2P doesn't have those, but the hidden-service-only model means it's relatively cheap to Sybil attack (spining up tons of I2P Bitcoin nodes cheaply which don't relay transactions might be enough to make I2P-private-broadcast unreliable).

I didn't mean to suggest that Dandelion is better, or even a worthwhile thing to pursue in addition to private broadcast; I think its [DoS concerns](https://bitcoin.stackexchange.com/a/81504/208) mostly make it a non-starter. I was just a bit triggered by your implication that transaction privacy problems are solved with private broadcast, while I wouldn't say that for something that is still opt-in with trade-offs. I may have jumped the gun in reading that much into your claim, though.

-------------------------

bubb1es | 2026-02-07 04:33:08 UTC | #14

I’m happy to see this thread included in the recent Optech newsletter! My plans going forward are:

1. continue to gather feedback on this approach (or any others) for disposing of dust
2. gather more on-chain data to improve my estimates of how big a problem this is
3. try to implement combining dust spend tx found in the mempool (as proposed by @ajtowns)
4. if a safe and effective approach is found to dispose of dust draft a BIP for it
5. reach out to wallet developers and see if they’d like to incorporate the feature

Call to action is for help on any of the above. :folded_hands:

-------------------------

gmaxwell | 2026-02-07 17:31:00 UTC | #15

https://github.com/petertodd/dust-b-gone

-------------------------

bubb1es | 2026-02-08 10:25:38 UTC | #16

Wow this is great prior work and supports the axiom there are no new ideas in bitcoin! or at least the obvious ones were written about in the first epoch.

I see some pros and cons to Peter’s approach. It’s certainly more efficient and avoids making timing data publicly available by consolidating and broadcasting dust spend transactions through his server. But even if you connect via Tor it could be logging and collecting timing data to associate your dust. And having multiple people running these servers doesn’t help since you wouldn’t know which are run by the same entity.

I prefer my proposed approach of leaving it up to the users to privately broadcast via their nodes and possibly coordinate grouping adhoc by finding others in the mempool you can spend with.  Wallet software should be able to put in some safeguards to prevent disposing of multiple dust UTXOs at the same time.

Thank you for sharing this and I’ll dig a bit more into his implementation to see if I’ve missed anything.

-------------------------

harris | 2026-02-19 10:43:14 UTC | #17

@bubb1es Thanks for reviving the issue of dust attack and thanks for your Rust based dedust cli tool. I read through the discussion here and it is interesting to know that this problem actually exists and it is not a new idea at all and there have been attempts to address it. I am interested in taking the project further and collaborating on improving the solution. I was wondering if you are still active on the dedust Rust cli and if so, I would be happy to get involved and start contributing.

Proposals would be e.g. dust attack detection logic, Combining transactions using ANYONECANPAY (ajtowns' suggestion) and finally we can include a PSBT workflow for hardware wallets.

Please let me know if this aligns with your plans. For context, i intend to work on this as a portfolio project for the BOSS Challenge 2026.

-------------------------

