# LN-Symmetry Project Recap

instagibbs | 2024-01-12 19:23:26 UTC | #1

# LN-Symmetry Project

ln-symmetry started with a simple idea: Someone should take the [eltoo](https://blockstream.com/eltoo.pdf)
idea and take it as far as possible to prove out the concept, without requiring community consensus required
actions like softforks before proving it out.

The end result is research-level quality software, with basic functional tests of the common cases:  
1) Ephemeral anchors + v3 usage for anti-pin  
2) channel opens
3) payments (revamped + simplified channel state machine)
4) payments with hops (fast-forwards, aka 0.5 RTT forwards!)
5) unilateral closes
6) Reconnection logic at different stages of channel updates

It did not implement:
1) cooperative closes: There's a lot of spec work to make this better for today's channels, and it would nearly be copy-paste, so I didn't bother
2) Proper persistence of the new keytypes/fields. In other words, if you restart your node, you'll crash. Couldn't quite figure it out
3) anchor spending support: Rusty was working hard at the time on `option_anchors` support; rebasing everything on top now should fix this
4) Gossip for these channels. So if you're playing around with it, you need to use them as private channels

I also got these opens/closes [broadcasted on signet via bitcoin-inquisition](https://mempool.space/signet/tx/9d63375a596ffd2be55624049580267ae07bf464e4d2fddc4043c7c3be79944c#vout=0):


# Key Focuses

1) KISS: I felt almost too stupid to understand the current BOLTs. How much shorter and simpler can I make them with a new channel type using APO-like functionality? Answer is quite a bit, at least within BOLTs 2, 3, and 5.
3) De-risking eltoo technical challenges: What are we missing in our handwaves of an idea? There's always something. Is that something fatal to the idea?

# Key Takeaways

1) Pinning is super hard to avoid. At least 1/3 of the work I bet was designing the system to be as robust against pinning as possible. I think I completed the task, except for HTLC-Success cases which actually need to be pre-signed to be pin-resistant. It's my biggest remaining design bug I know about.
2) eltoo-style channels have even longer than expect htlc expiry deltas to stay secure. No one had actually worked through the state machine before.
3) Much more flexible fee management: All outputs in settlement transactions(except HTLC-Success paths as noted before) are free to spend, allowing endogenous fees from all other output cases of balance and HTLC outputs. Reduces the burden on having a utxo pool for fees to only pay maximum once per unilteral close during the "challenge" period.
4) CTV(emulation) ended up being useful! It removed the necessity of round-trips from the payment protocol, allowing for "fast forwards" that are extremely simple, and would likely reduce payment times if widely adopted. 
5) I'm less convinced than ever that penalties are the way forward in the future. Penalties are, in practice against a competent adversary, only as large as their reserve requirements, so "make sure it works right" seems to be the best deterrent. Make the expected chance of success essentially zero, and the incentive should be to collaboratively close to save fees.

# Key work artifacts

1) Tons of anti-pinning research resulting in many milestones in the [Package Relay Project](https://github.com/bitcoin/bitcoin/issues/27463) including my novel invention of [Ephemeral Anchors](https://github.com/instagibbs/bips/blob/d33cdbd0777700f4fc488d54b90a8795d2c33639/bip-ephemeralanchors.mediawiki) which builds on top of "V3" concept.
2) CLN implementation here: https://github.com/instagibbs/lightning/tree/eltoo_support
3) Bitcoin Core branch with necessary changes for CLN blackbox tests: https://github.com/instagibbs/bitcoin/tree/202207-anyprevout (though this has non-segwit version of ephemeral anchors)
4) or maybe https://github.com/bitcoin-inquisition/bitcoin/releases/tag/v25.1-inq
5) BOLT draft for ln-symmetry(with segwit ephemeral anchors): https://github.com/instagibbs/bolts/tree/eltoo_draft
6) libwally work to support taprooty CLN changes(chunks of this was upstreamed but not all): https://github.com/instagibbs/libwally-core/tree/taproot
7) Notes on PTLCs, where eltoo-style channels get trivial fast-forwards still(aka APO-style fork fixes this): https://gist.github.com/instagibbs/1d02d0251640c250ceea1c66665ec163#single-sig-adaptor-ln-symmetry
8) Backporting anti-pin tech to today's BOLTs with proposed Bitcoin Core policy changes only: https://github.com/instagibbs/bolts/commits/zero_fee_commitment, with [implementation in CLN](https://github.com/instagibbs/lightning/commits/commit_zero_fees/)

# Current Status
On hold while I focus on the mempool side of things. I hope this work helps derisk conversations about future softforks and channel designs.

-------------------------

ProofOfKeags | 2024-01-05 18:09:14 UTC | #2

Incredible work and post. This kind of content makes me glad I'm subscribed to emails.

[quote="instagibbs, post:1, topic:359"]
eltoo-style channels have even longer than expect htlc expiry deltas to stay secure. No one had actually worked through the state machine before.
[/quote]

Can you elaborate on precisely why this is the case. I'm not sure I intuit why this is necessary. In theory the injunction period in the existing protocol is two rounds max: breach and justice. In eltoo, my understanding is that it is still two rounds: breach and fast-forward.

[quote="instagibbs, post:1, topic:359"]
CTV(emulation) ended up being useful! It removed the necessity of round-trips from the payment protocol, allowing for “fast forwards” that are extremely simple, and would likely reduce payment times if widely adopted.
[/quote]

I'm thoroughly unsurprised by this as CTV/Covenant schemes can, in principle, give you state-graph cut-through on any state machine encoded into Bitcoin transactions. However, I do have a question as to whether alternative covenant schemes also give you this property? I suspect but am not certain that they can.

-------------------------

instagibbs | 2024-01-05 18:41:17 UTC | #3

[quote="ProofOfKeags, post:2, topic:359"]
Can you elaborate on precisely why this is the case. I’m not sure I intuit why this is necessary. In theory the injunction period in the existing protocol is two rounds max: breach and justice. In eltoo, my understanding is that it is still two rounds: breach and fast-forward.
[/quote]

Fairly simple to explain and obvious in retrospect:
1) Alice and Bob do some channel ops
2) Alice then sends `update_signed` which is a MuSig2 partial sig
3) Bob smirks now that he's the only one with the full signature, goes silent
4) Alice goes to chain with prior state, waits however long the "update phase" has a timelock for the "settle transaction" to be valid
5) Right before timelock expires, Bob publishes the final state
6) wait for this timelock to expire... publish settle tx
7) finally can settle HTLCs

that's 2x the update phase timelock, not 1x. I know there's some "sandwiching" you can do to compress it a bit(conversations between LN engineers I didn't understand!), but it's north of my once expected 1x.

This game also shows up in the "layered eltoo" design from https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-January/002448.html which was supposed to get rid of the additional delta values altogether.

[quote="ProofOfKeags, post:2, topic:359"]
I’m thoroughly unsurprised by this as CTV/Covenant schemes can, in principle, give you state-graph cut-through on any state machine encoded into Bitcoin transactions. However, I do have a question as to whether alternative covenant schemes also give you this property? I suspect but am not certain that they can.
[/quote]

OP_CTV is probably the ideal here? I never validated it end to end or carefully looked though. I emulated it for a few more vbytes using APO.

-------------------------

instagibbs | 2024-01-05 19:18:51 UTC | #4

Ah, forgot one more issue left:

In spec/implementation, we only pass around a single nonce per round. This is correct *unless* you use the "optimistic" updates from "simplified update" protocol proposed originally by Rusty. In other words, you can never send updates out of turn, with the associated latency. You have to prod your counter-party to give up their turn first and wait for them to respond. To fix the out of turn optimistic updates, you would need another nonce and associated logic to manage it safely.

I never implemented the optimistic sending of updates, but didn't ban it outright and it's still in the spec: https://github.com/instagibbs/bolts/blob/a17b60f42077a785c625430e8f6e8e2828d4d898/XX-eltoo-peer-protocol.md#eltoo-simplified-operation

-------------------------

rustyrussell | 2024-01-10 17:01:16 UTC | #5

Is there a description for exactly how you used CTV to avoid a round-trip? I'm clearly missing something here...

-------------------------

instagibbs | 2024-01-10 19:56:47 UTC | #6

[quote="rustyrussell, post:5, topic:359, full:true"]
Is there a description for exactly how you used CTV to avoid a round-trip? I’m clearly missing something here…
[/quote]

An attempt was made to explain, along with the definition of `CovSig`, here: https://github.com/instagibbs/bolts/blob/a17b60f42077a785c625430e8f6e8e2828d4d898/XX-eltoo-transactions.md#rationale-1

proposed method to remove round-trips without CTV-like application, or the annex usage: https://github.com/instagibbs/bolts/issues/5

-------------------------

ajtowns | 2024-01-14 05:52:08 UTC | #7

[quote="rustyrussell, post:5, topic:359, full:true"]
Is there a description for exactly how you used CTV to avoid a round-trip? I’m clearly missing something here…
[/quote]

The idea is:
 * I want to propose a new state; but I can't just give you signed transactions for the new state, unless I'm sure I can claim my funds from that state. Normally that would mean I give you the signature for the update tx after I've received your signature for the settlement tx. But that's an extra round.
 * So instead have the update tx use CTV (or APO-simulating-CTV) to allow spending via a settlement tx without the settlement tx needing a signature.
 * But then that means that spending the update tx is only possible if you know the CTV commitment to the settlement tx, which would normally mean O(n) storage, since you need to be able to spend every historical update tx to the current one, and each update tx can have a different settlement tx.
 * So instead we have the update tx store that information in the annex field, so that it can be recovered if an old update tx is broadcast, without needing to be stored.

The alternative approach is:

 * Give a partial adaptor signature for the update tx, where Bob completing the signature allows Alice to reconstruct a valid signature to broadcast the settlement tx

But that means doing adaptor signatures, and (I think) requires the spend of the update output in the settlement tx to have two CHECKSIGs (Alice with a normal sig, Bob via the adaptor recovery), so got put in the "too hard" basket for now.

-------------------------

ajtowns | 2024-01-14 06:13:17 UTC | #8

I had a go at trying this out and failed pretty miserably. Some problems:

 * the feature bit used for eltoo was 50 which overlaps with zeroconf channels. should perhaps be 0xcc32? (ie the two byte ascii string 'L2', plus 32768 to put it into the experimental feature bits range) of course, if you do that, cln listpeers decides to put 13000 "0" characters in the "features" field, which gets pretty annoying
 * trying to create non-eltoo channels is just broken anyway, so connecting to normal peers isn't useful
 * creating a channel between two nodes does seem to work
 * paying an invoice doesn't seem to work: it couldn't find a path to its peer somehow
 * cooperative shutdown doesn't seem to work; the closing node thinks its closing, the other node doesn't (`eltoo_channeld-chan#1: billboard perm: Bad shutdown` maybe?).
 * unilateral shutdown also doesn't seem to work; segfault/nullptr deref? after sending the unilateral shutdown when trying to resolve txid
 * restarting nodes after stopping/crashing doesn't work -- the `per_commit_remote` value stored in the db isn't a valid secp point, so loading it fails

I did manage to broadcast my [update](https://mempool.space/signet/tx/babe14326bf9ca2a180247e0e46970095403451c0b3b2b4c684aa353b419e9b7) and [settlement](https://mempool.space/signet/tx/21f854714debb0db46adc9d446a242e9a1d502dfe9761ac490ae8862fc0254b1#vin=0) txs and pay for them with ephemeral anchors manually (the update tx was in the logs, and the settlement tx was able to be reconstructed just by dropping in the update tx's txid).

I then got stuck trying to figure out the private keys in order to actually reclaim those funds, and that's where I'm at for now.

-------------------------

instagibbs | 2024-01-14 12:44:16 UTC | #9

[quote="ajtowns, post:8, topic:359"]
paying an invoice doesn’t seem to work: it couldn’t find a path to its peer somehow
[/quote]

This [should work](https://github.com/instagibbs/lightning/blob/be3919d9814579e6c8fb42c49c2efc9fcb5cb91f/tests/test_eltoo.py#L135), if you're doing hops you'll need to provide a route hint since these channels are private

[quote="ajtowns, post:8, topic:359"]
I then got stuck trying to figure out the private keys in order to actually reclaim those funds, and that’s where I’m at for now.
[/quote]

Everything not [under test](https://github.com/instagibbs/lightning/blob/be3919d9814579e6c8fb42c49c2efc9fcb5cb91f/tests/test_eltoo.py) was probably not implemented and certainly not tested.

-------------------------

ajtowns | 2024-01-15 06:56:27 UTC | #10

[quote="instagibbs, post:9, topic:359"]
This [should work](https://github.com/instagibbs/lightning/blob/be3919d9814579e6c8fb42c49c2efc9fcb5cb91f/tests/test_eltoo.py#L135), if you’re doing hops you’ll need to provide a route hint since these channels are private
[/quote]

Yeah, I'm not sure what was going on there, might just be that the amounts I was trying were too large (I think I tried 25 sBTC on a 50 sBTC channel?) and splitting into smaller amounts was what confused things or something. When I tried debugging I got lost in the maze of other failures  :slight_smile:

-------------------------

cguida | 2024-01-16 16:35:24 UTC | #11

Thanks @instagibbs for the detailed explainer! I'm working on rebasing the 283 commits in your CLN impl so it can be ready for APO / LNHANCE / TXHASH.

Lots of new stuff for me to learn, so it'll probably be somewhat slow :grimacing: 

If anyone wants to collaborate, please feel free to reach out! :slight_smile:

-------------------------

cguida | 2024-01-25 14:42:33 UTC | #12

If anyone wants to try out instagibbs' CLN implementation, I was able successfully force close a channel on inquisition signet using this method:

https://gist.github.com/chrisguida/43e482e684727daa618423e52a14d09a

Let me know if it works/breaks for you!

Edit: here's the entire thing in markdown!

The following is one way to broadcast an LN-symmetry (formerly `eltoo`) settlement tx on inquisition signet, using instagibbs' original APO eltoo implementation on CLN.

1) build and install inquisition signet, you will need this fork of bitcoind for support of APO, ephemeral anchors, and taproot annex blobs
```
git clone https://github.com/bitcoin-inquisition/bitcoin
cd bitcoin
./autogen.sh
./configure
make -j$(( $(nproc) - 1))
sudo make install
```

2) build and install instagibbs' `eltoo_support` branch of CLN
```
cd ..
git clone https://github.com/instagibbs/lightning -b eltoo_support
cd lightning
./configure --enable-developer --enable-experimental-features
make -j$(( $(nproc) - 1))
sudo make install
```

3) launch bitcoind on signet and make sure it connects to ajtowns' inquisition node
```
bitcoind -signet -daemon -server -txindex=1 -addnode=inquisition.bitcoin-signet.net
```
4) get some signet coins
```
bitcoin-cli -signet createwallet default
bitcoin-cli -signet loadwallet default
bitcoin-cli -signet getnewaddress
```

- Go to https://signetfaucet.com/ for up to 0.01 sbtc at a time
- You can also try `contrib/signet/getcoins.py`
- If nothing else works you can ask for signet coins on IRC in #bitcoin-signet

5) wait for IBD to complete, then launch 2 inquisition nodes using `startup_regtest.sh`
```
sed -i 's/regtest/signet/g' contrib/startup_regtest.sh
source contrib/startup_regtest.sh
start_ln
```

Note: this branch of CLN does not support restarting the node, so you will have to nuke your node and start over if your node stops for any reason. So make sure you send all signet coins out of the node before stopping it, or your coins will be lost.

6) Send some coins to Node 1
```
l1-cli newaddr
bt-cli sendtoaddress <addr>
```

7) Wait for the deposit to confirm, then open a 100k-sat channel from Node 1 to Node 2
```
l2-cli getinfo
l1-cli connect <node-2-id@127.0.0.1:7272>
l1-cli fundchannel <node-2-id> 100000sat
```

8) Wait a few confirmations for the channel to become active, then optionally pay a 10k-sat invoice from Node 1 to Node 2:
```
l2-cli invoice 10000000 test test
l1-cli pay <bolt11>
```

Now let's force close the channel! (This implementation doesn't support mutual closes, so don't try or your funds will get stuck) 

9) Get the raw last_update_tx from the `listpeers` call:
```
l1-cli listpeers
```

10) Broadcast it:
```
bt-cli sendrawtransaction <last_update_tx>
```

The tx will get stuck in your node's mempool because it has an ephemeral anchor, which must be spent in order for the tx to be relayed.

11) CPFP using the ephemeral anchor. Take note of the txid of both the ephemeral anchor and one of our bitcoind wallet's UTXOs. We'll also need a new address to receive our CPFP:
```
bt-cli decoderawtransaction <last_update_tx>
bt-cli listunpsent
bt-cli getnewaddress
```

12) Construct a new tx that spends both outputs back to our bitcoind wallet. I used 1,000 sats for the fee, probably a bit overkill, but this is fake money anyway:
```
bt-cli createrawtransaction '[{"txid":"<update-tx-txid>", "vout": <ephemeral-anchor-vout>}, {"txid":"<bitcoind-wallet-UTXO-txid>", "vout":<bitcoind-wallet-UTXO-vout>}]' '{"<bitcoind-wallet-address>":"<UTXO-BTC-amount-minus-0.00001000>"}'
```

13) Sign and broadcast it
```
bt-cli signrawtransactionwithwallet <new-cpfp-unsigned-tx>
bt-cli sendrawtransaction <new-cpfp-signed-tx>
```

14) Check to make sure the update tx has been relayed:
```
bt-cli getmempoolentry <update-txid>
```

If it says `"unbroadcast": false`, you're good to go.

Once it gets confirmed in a block, you should be able to see the update tx at https://mempool.space/signet/tx/update-txid. If both txs say doesn't show up in the next block, reach out to #bitcoin-signet and see if there's anything wrong with inquisition relay.

15) Wait a few blocks for the challenge period to end, then grab the settlement tx and broadcast it too:
```
l1-cli listpeers
bt-cli sendrawtransaction <last_settlement_tx>
```

16) Make sure you CPFP this tx as well, as above, as it also has an ephemeral anchor.
17) Wait for the settlement tx to confirm. Find it on mempool.space as before.
18) To save your coins, wait 100 blocks for the channel to be forgotten, then sweep all funds from CLN back into bitcoind. Now you can safely stop your CLN nodes.

Congratulations, you've just force closed your first LN-symmetry channel on inqusition signet!

-------------------------

rustyrussell | 2024-02-24 04:13:31 UTC | #13

[quote="ajtowns, post:7, topic:359"]
I want to propose a new state; but I can’t just give you signed transactions for the new state, unless I’m sure I can claim my funds from that state. Normally that would mean I give you the signature for the update tx after I’ve received your signature for the settlement tx. But that’s an extra round.
[/quote]

For my own clarification here, BTW: we do this in the current protocol without any issues, but for LN-symmetry we split the tx into two parts (update and settlement), so now the peer could spend the new update and withhold the new settlement.

> But then that means that spending the update tx is only possible if you know the CTV commitment to the settlement tx, which would normally mean O(n) storage, since you need to be able to spend every historical update tx to the current one, and each update tx can have a different settlement tx.

AFAICT you only need two, assuming you wait for a reply before sending another update?  You would always fast-forward to the latest update tx.

> So instead have the update tx use CTV (or APO-simulating-CTV) 

Adapter signatures is nicer, BUT note that CTV is more optimal than adapter signatures here!  However, neither is actually ideal (requiring another tx for fees): what we *want* is a commitment scheme which lets the spender extract fee from its own output, and/or bring their own fee input.

-------------------------

instagibbs | 2024-02-26 14:57:10 UTC | #14

[quote="rustyrussell, post:13, topic:359"]
AFAICT you only need two, assuming you wait for a reply before sending another update? You would always fast-forward to the latest update tx.
[/quote]

Might be talking past each other, but the counter-party may play an old update state to which you've forgotten the opening of the taproot commitment to your update. You could remember O(n) fields rather than use the annex, of course.

[quote="rustyrussell, post:13, topic:359"]
Adapter signatures is nicer, BUT note that CTV is more optimal than adapter signatures here! However, neither is actually ideal (requiring another tx for fees): what we *want* is a commitment scheme which lets the spender extract fee from its own output, and/or bring their own fee input.
[/quote]

I like CTV-like approach for another reason: it trivially generalized to multi-party channels. That said, yeah optimally we'd have some sort of TXHASH-like construct for settlement that would allow single tx endo/exo fees depending on how much is locked up in HTLCs and the like.

-------------------------

rustyrussell | 2024-02-27 06:21:18 UTC | #15

[quote="instagibbs, post:14, topic:359"]
Might be talking past each other, but the counter-party may play an old update state to which you’ve forgotten the opening of the taproot commitment to your update. You could remember O(n) fields rather than use the annex, of course.
[/quote]

If they play an old update, you spend it with the new update? Wasn't that the entire trick? Obviously I'm missing something (I'm guessing some Taproot detail?)

-------------------------

ajtowns | 2024-02-27 06:27:47 UTC | #16

[quote="rustyrussell, post:15, topic:359"]
If they play an old update, you spend it with the new update? Wasn’t that the entire trick? Obviously I’m missing something (I’m guessing some Taproot detail?)
[/quote]

The update tx commits to the settlement tx in the tapscript path, but that means that spending the update tx means you have to know the settlement tx even if you want to do some other spend than the settlement tx, either in order to reconstruct the tapscript that includes the settlement commitment, or to reconstruct the merklepath to some other tapscript (since the merkleroot commits to the script that commits to the settlement tx) or to do a key path signature (since you need to tweak the internal key signature via the merkle root which commits to the script that commits to the settlement tx).

-------------------------

