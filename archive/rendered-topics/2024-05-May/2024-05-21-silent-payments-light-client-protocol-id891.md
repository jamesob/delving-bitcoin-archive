# Silent Payments: Light Client Protocol

setavenger | 2024-05-23 10:04:49 UTC | #1

During the development of a couple light clients I jotted down some notes and created a very early draft for a [Light Client Specification](https://github.com/setavenger/BIP0352-light-client-specification). The appendix of the SP BIP was used as a basis for this. The specification was designed with two goals in mind. (1) Reduce the computational burden for light clients, and (2) Minimise the bandwidth requirements. Both goals have to be achieved without compromising privacy for the user. The protocol is designed such that a light client can connect to any public indexing server without giving out more information than "I'm interested in this block".

There have been a couple short discussions around this Light Client Protocol. Now that an early draft exists I would like to get more people involved in the process. The draft can be found here: https://github.com/setavenger/BIP0352-light-client-specification

The computational burden is reduced by generating a tweak index. To further reduce the required computations (and also bandwidth) we can apply cut-through to the transactions. So pruning tweaks for transactions where all taproot UTXOs are spent. Bandwidth is reduced through various measures which mainly revolve around only providing the data necessary for a light client to find and spend a UTXO.

The basic flow for a client to receive is as follows:

Per block:

1. Fetch the tweaks (possibly filtered for dust limit)
2. Compute the possible pubKeys for n = 0
3. Fetch taproot-only filter (BIP 158)
4. Compare the pubKeys against a taproot-only filter
    - If no match: go to 1. with block_height + 1
    - Else: continue with 5.
5. Fetch simplified UTXOs
6. Scan according to the BIP (one could reuse pubKeys from 2. here)
7. Collect all matched UTXOs and add to wallet
8. Go to 1. with block_height + 1

As mentioned in the specification a scriptPubKey that has already received funds should be tracked as well. The reasoning can be found in the specification.

-------------------------

cygnet3 | 2024-05-22 12:03:45 UTC | #2

I have one minor nitpick about the format of the `utxos` endpoint from the spec, which is given as:

```json
[
  {
    "txid": "355cce4314a...",
    "vout": 1,
    "value": 1000000,
    "scriptpubkey": "...",
    "block_height": 0,
    "block_hash": "...",
    "timestamp": 1715205985,
    "spent": false
  },
  {
    "txid": "355cce4314a...",
    "vout": 2,
    "value": 1000000,
    "scriptpubkey": "...",
    "block_height": 0,
    "block_hash": "...",
    "timestamp": 1715205985,
    "spent": false
  },
  ...
]
```

I think it's better to group the utxos by their `txid`. Since Step 2 only calculates the `ScriptPubKey` for `k=0`, we still need to discover the `ScriptPubkey`s where `k>0` in Step 6. The only way to do that is to group all the outputs of a transaction first, and then scan them at once.

Since the `utxos` aren't requested that often (only when the filter returns a match), it's not that much of a hassle to just group them on the client side. Still I think every client will have to do this, so might as well account for it.

-------------------------

josibake | 2024-05-22 11:38:16 UTC | #3

Thanks for posting! Still need to read through the linked proposal, but some initial thoughts:


[quote="setavenger, post:1, topic:891"]
Fetch the tweaks (possibly filtered for dust limit)
[/quote]

Ideally, this is a parameter set by the client. If bandwidth isn't a concern for the client, they may want to know about *any* silent payment (e.g. `dust_limit=0`). For a more bandwidth constrained client, they may only be interested in pulling in UTXOs to the wallet that would be immediately spendable and set something much higher (e.g. `dust_limit=5000`), or may want to avoid UTXOs that were created as part of a dusting attack.
[quote="setavenger, post:1, topic:891"]
Fetch taproot-only filter (BIP 158)
[/quote]

Do you have any numbers for how using a taproot filter vs an off-the-shelf BIP158 filter impacts the bandwidth? If it's not much of a difference, we might be better off reusing BIP158 filters as is, since there is an existing use case for these and nodes already create them. Furthermore, as taproot adoption increases, I would expect the size difference between a BIP158 filter and a taproot-only filter to be negligible.
[quote="setavenger, post:1, topic:891"]
Fetch simplified UTXOs
[/quote]

I'm assuming you're fetching simplified UTXOs here so that you can scan with labels, since this requires access to the transaction outputs? An alternative would be to just request the full block as soon as you get a hit: this would allow the client to scan with labels since they would have the full transactions, and the client would also have all the necessary information needed for spending the transaction.

More of a general comment: is there every a reason a client would want the tweak data and not a filter or vice versa? If not, seems better to always give them to the client together.

-------------------------

setavenger | 2024-05-23 10:05:13 UTC | #4

[quote="cygnet3, post:2, topic:891"]
Since Step 2 only calculates the `ScriptPubKey` for `k=0`, we still need to discover the `ScriptPubkey`s where `k>0` in Step 6. The only way to do that is to group all the outputs of a transaction first, and then scan them at once.
[/quote]

I'm not sure I'm understanding the problem you're describing. Scanning over all outputs you should 100% be able to find all outputs (also those with `k<0`).
blindbitd uses [this function](https://github.com/setavenger/go-bip352/blob/27de6d8b3f8aaf0d555e044d7d6edbf0ddb518a3/receive.go#L27) to scan for outputs.

edit: Looking a bit closer at my implementation, I think I see where the confusion comes from. blindbitd uses all taproot outputs of a block as an input for the scan function. Tweaks are not labeled with their corresponding txid. So we have to scan over all outputs of a block. Then you should always find all outputs. This means scanning over all outputs with `k>0` as well. I believe the [PR for Bitcoin core](https://github.com/bitcoin/bitcoin/pull/28241) does not store txids as well. In Oracle I have an index that stores the txid but only the tweaks are served to the client.

-------------------------

setavenger | 2024-05-22 12:50:23 UTC | #5

[quote="josibake, post:3, topic:891"]
Ideally, this is a parameter set by the client.
[/quote]

BlindBit does exactly this. I will make sure that this is emphasised in the Specification.

[quote="josibake, post:3, topic:891"]
Do you have any numbers for how using a taproot filter vs an off-the-shelf BIP158 filter impacts the bandwidth?
[/quote]

Not yet. I will try to get some numbers.

[quote="josibake, post:3, topic:891"]
I’m assuming you’re fetching simplified UTXOs here so that you can scan with labels, since this requires access to the transaction outputs? An alternative would be to just request the full block as soon as you get a hit: this would allow the client to scan with labels since they would have the full transactions, and the client would also have all the necessary information needed for spending the transaction.
[/quote]

The main idea was to reduce bandwidth. I wanted to condense the block in such a way that only the relevant information is included. Downloading the entire block would probably include quite a bit of information that the light client will never need. I believe, having raw block data would also require the light client to do some extra work with regards to parsing and finding eligible transactions, right? With the current specification the client can directly use all the information and has to do minimal work on its own side. With this method the client still finds all labels and can immediately spend as well.

Also in general simplified UTXOs might not be the correct wording for this. The basic idea is that it's a data structure which contains all necessary information to find and properly spend a UTXO.

[quote="josibake, post:3, topic:891"]
More of a general comment: is there every a reason a client would want the tweak data and not a filter or vice versa? If not, seems better to always give them to the client together.
[/quote]

This is probably an artefact from testnet, as there are a lot of blocks were no tweaks exist and we can save bandwidth by not requesting filters. On mainnet this is not the case. Apart from that I can't think of a good reason. I will merge those two steps into one.

-------------------------

cygnet3 | 2024-05-22 15:28:27 UTC | #6

[quote="setavenger, post:4, topic:891"]
blindbitd uses all taproot outputs of a block as an input for the scan function.
[/quote]

I see, I agree that also works.

[quote="setavenger, post:4, topic:891"]
Tweaks are not labeled with their corresponding txid. So we have to scan over all outputs of a block.
[/quote]
In Donation wallet, we keep a mapping between `ScriptPubKey -> tweak` that we calculated during Step 2. Even though that is not a direct mapping from `txid -> tweak`, you can still indirectly look up the `tweak` of a transaction by looking at the outputs. I think this avoids having to scan all outputs of a block for every tweak, without needing to tag the tweaks with their `txid`. However, it does require a grouping of the outputs.

-------------------------

setavenger | 2024-05-22 17:24:46 UTC | #7

[quote="cygnet3, post:6, topic:891"]
In Donation wallet, we keep a mapping between `ScriptPubKey -> tweak` that we calculated during Step 2. Even though that is not a direct mapping from `txid -> tweak`, you can still indirectly look up the `tweak` of a transaction by looking at the outputs. I think this avoids having to scan all outputs of a block for every tweak, without needing to tag the tweaks with their `txid`. However, it does require a grouping of the outputs.
[/quote]

The UTXOs already contain the txids so grouping is already possible on the client side if necessary. The question is whether this should happen on the indexing or client side. I would not agree that grouping avoids scanning all the outputs. It only avoids doing it a second -> nth time if you've found an output on `k=0`. 

But if we were to offload grouping to the indexer, how would it look? How is the grouping done for donation wallet, is there any additional overhead for doing so? Especially on a per request basis. Which structure do you currently use? Something like this:
```json
{
    "txid1.." : [
        {
            "txid": "txid1...",
            "vout": 1,
            "value": 1000000,
            "scriptpubkey": "...",
            "block_height": 0,
            "block_hash": "...",
            "timestamp": 1715205985,
            "spent": false
         },
         {
            "txid": "txid1...",
            "vout": 2,
            "value": 1000000,
            "scriptpubkey": "...",
            "block_height": 0,
            "block_hash": "...",
            "timestamp": 1715205985,
            "spent": false
        },
        ...
    ],
    "txid2": [
        {
            "txid": "txid2...",
            "vout": 2,
            "value": 1000000,
            "scriptpubkey": "...",
            "block_height": 0,
            "block_hash": "...",
            "timestamp": 1715205985,
            "spent": false
          },
    ]
}
```

-------------------------

cygnet3 | 2024-05-22 19:17:29 UTC | #8

[quote="setavenger, post:7, topic:891"]
Which structure do you currently use? Something like this:
[/quote]
When testing out Blindbit oracle as a backend, we just [convert the utxo array into a map](https://github.com/cygnet3/donationwallet/blob/f07f4e13013c168049e9d595d9bdffd24991f777/rust/src/blindbit/logic.rs#L144) and then loop over the values. That has the same structure as the json you posted. It could be simplified by removing `txid` from the output struct.

If not using blindbit, we use a BIP158 client and request the full block, so that has the complete transaction structure.

-------------------------

setavenger | 2024-05-23 10:04:41 UTC | #9

[quote="cygnet3, post:8, topic:891"]
When testing out Blindbit oracle as a backend, we just convert the utxo array into a map and then loop over the values. That has the same structure as the json you posted. It could be simplified by removing `txid` from the output struct.
[/quote]

Is there any noticeable overhead created by converting on the client side? I think we should benchmark these approaches to see how they compare. Then we can better say whether grouping should be a hard requirement on the indexing side.

[quote="cygnet3, post:8, topic:891"]
If not using blindbit, we use a BIP158 client and request the full block, so that has the complete transaction structure.
[/quote]

How does that approach work in general? Something like this?
1. Fetch tweaks + Filters
2. Create ScriptPubKeys (with mapping)
3. Find match
4. Download entire block
5. Scan transactions and find outputs

Where step 5 is optimised due to mapping and grouping. Did I get this right?

You mentioned that you've tried BlindBit Oracle as well, are there any noticeable differences in required bandwidth? With BlindBit the goal was to avoid downloading entire blocks to reduce bandwidth, but I never made an actual comparison.

-------------------------

josibake | 2024-06-02 09:25:01 UTC | #10

[quote="setavenger, post:9, topic:891"]
Is there any noticeable overhead created by converting on the client side?
[/quote]

The two benefits I see for providing outputs grouped by $txid$ are reduced bandwidth and less computation for the client in the event an output is found

## Bandwidth

Outputs coming from the same transaction will all have the same $txid$, so we can reduce the data sent to the client by $(n_{outputs} -1)\cdot n_{transactions} \cdot 32$ bytes. Similarly, it looks like there are a few more repeated fields for each UTXO, namely blockhash and block number[^1]. So the final result would be something like `block_hash: { txid1: [output1, output2..], txid2: [output3, output4..] }`

## Computation

Once an output is found and the simplified UTXOs have been fetched, assuming outputs are not grouped by $txid$, a client would need to do the following:

1. Verify $SPK_{k=0}$ is not a false positive by finding it in the list and removing it
2. Create $SPK_{k=1}$ and check the entire list
3. Repeat with $k++$ every time an output is found

Whereas if the list is grouped by $txid$ (either by the client or the server), the client finds the $txid$ in step one and iterates over the outputs corresponding to $txid$ vs outputs for the entire block. This is arguably only a small improvement, but server side grouping means we can save bandwidth for the client and, as @cygnet3 points out, if every client will end up grouping it seems better to do the work one time on the server.

[^1]: It's not immediately clear to what the additional fields are for, i.e. block hash, block number, timestamp. In the case of block info, seems like we could use block hash or block number, depending on what the wallet needs it for?

-------------------------

setavenger | 2024-06-02 13:07:30 UTC | #11

[quote="josibake, post:10, topic:891"]
The two benefits I see for providing outputs grouped by txidtxidtxid are reduced bandwidth and less computation for the client in the event an output is found
[/quote]

No doubt there.

[quote="josibake, post:10, topic:891"]
if every client will end up grouping it seems better to do the work one time on the server
[/quote]

This is a valid statement. I was coming from a perspective of doing these transformations on-the-fly per request. As of now the BB Oracle architecture would require on-the-fly mapping. The reason being that UTXOs need to be updated individually to change the spent state. Grouping UTXOs in storage would probably generate some overhead for sync times. Might be negligible though.

[quote="josibake, post:10, topic:891"]
It’s not immediately clear to what the additional fields are for, i.e. block hash, block number, timestamp. In the case of block info, seems like we could use block hash or block number, depending on what the wallet needs it for?
[/quote]

It's just an optional metadata field. `block_height` or `block_hash` should suffice if the wallet does extra requests to get block meta data. `timestamp` seemed like an obvious use case. The optional fields are not very refined yet and just ideas I threw out there. None of these fields are technically required to spend the outputs which is why I put them in optional to begin with.
If included it does make sense to not have them repeated in every output and rather aggregate on a higher level.

-------------------------

harding | 2024-06-05 02:13:35 UTC | #12

[quote="setavenger, post:1, topic:891"]
1. Fetch the tweaks (possibly filtered for dust limit)
2. Compute the possible pubKeys for n = 0
3. Fetch taproot-only filter (BIP 158)
4. Compare the pubKeys against a taproot-only filter
    - If no match: go to 1. with block_height + 1
    - Else: continue with 5.
5. Fetch simplified UTXOs
6. Scan according to the BIP (one could reuse pubKeys from 2. here)
7. Collect all matched UTXOs and add to wallet
8. Go to 1. with block_height + 1
[/quote]

I'm concerned clients only performing step 5 (fetching transaction data) in response to untrusted data from step 1 (tweaks) and step 3 (pubkeys).  If a client fetches all three pieces of data from the same server (or from different servers that are colluding), the server can potentially unmask users in step five by lying in steps 1 and 3.

For example: Server operator Mallory wants to discover the IP address of a user with SP address _x_.  Mallory creates a fake payment to _x_, giving her a tweak and an output that indicate a payment to _x_.  Instead of creating the tweaks and filters for the next block honestly, Mallory creates them using only the fake payment, distributing them to all of her users.  The only user who matches on that fake data is the owner of _x_, so that person is the only person who performs step 5 (downloading transaction data); this reveals their network ID to Mallory (e.g., their IP address if they use a direct connection).

Downloading tweaks and filters from different servers doesn't help.  Even if we can be sure the servers aren't colluding, whoever controls the filter distribution server can always force a match by lying.  I think whoever controls the tweak distribution server can also force a match, but I'm not 100% sure on the EC math to do that.

What I think is best is similar to what Wasabi does with its custom BIP158 implementation:

1. Client downloads untrusted tweaks and filter from the server (ideally using something like an ephemeral Tor connection)
2. On match, client downloads the corresponding full block from a random full node (ideally using a different network identity, such as a different ephemeral Tor connection)

Given the large number of block-serving full nodes, this reduces the chance that the client connects to a full node controlled by the server.  Additionally, given the modestly large number of existing BIP158 clients that are already occasionally downloading arbitrary blocks from full nodes, even if the client did connect to a full node controlled by the server, the server couldn't be sure that the peer requesting a particular block was the peer it was targeting.  This is true on regular IP, with increased privacy guarantees available to users of Tor ephemeral addresses or similar protocols.

Thus I think there's a lot of advantage to using full blocks and connections to regular full nodes in step 5 of this protocol.  The downside of full blocks over minimized blocks is increased bandwidth, but my guess is that most SP users will receive less than one SP payment per day, so the bandwidth cost of a full block is less than 4 MB per day. Those receiving more payments can probably easily afford the increased bandwidth costs (about 600 MB/day in the worst case).

-------------------------

josibake | 2024-06-05 09:12:48 UTC | #13

Thanks for the concrete example, @harding !

[quote="harding, post:12, topic:891"]
Downloading tweaks and filters from different servers doesn’t help. Even if we can be sure the servers aren’t colluding, whoever controls the filter distribution server can always force a match by lying. I think whoever controls the tweak distribution server can also force a match, but I’m not 100% sure on the EC math to do that.
[/quote]

I'm not convinced it's sufficient to only control filter distribution. A sender creates a silent payment output as $P = B_{spend} + hash(a\cdot B_{scan} | k)\cdot G$, and the light client recipient finds the output by calculating $P = B_{spend} + hash(A_{server}\cdot b_{scan} | k)\cdot G$ and checking if $P \in taprootfilter$. The key is the ECDH step inside the hash which creates a shared secret between sender and recipient. So for this attack to work, Mallory must control both the tweak server and the filter server to force a match, i.e. Mallory needs to give the client a specific $A_{fake}$ that when multiplied by $b_{scan}$ will match a $P_{fake}$ output in the filter. Additionally, Mallory would need to collude with (but not control) the server providing the "simplified UTXOs" / block data.

All that being said, I think it's very likely that Mallory will control all three endpoints, or at least control the tweak and filter endpoints and be able to collude with whoever is providing the block data. In the case where Mallory controls tweaks and filters and can collude, they can link IPs to BIP352 addresses without detection (assuming nobody audits the data Mallory is returning from the tweak end point[^1]) by blaming the the "fake hits" on the filter false positive rate.

[^1]: Interestingly, the only one Mallory can get away with lying about is the filter: tweak data and simplified UTXOs are both public data sets, making it easy for anyone with access to the full blockchain to audit the data Mallory is returning and detect fake payments. But for this attack to be successful, Mallory must return fake tweaks to more than one client, making it very likely that Mallory will be caught.

[quote="harding, post:12, topic:891"]
Given the large number of block-serving full nodes, this reduces the chance that the client connects to a full node controlled by the server. Additionally, given the modestly large number of existing BIP158 clients that are already occasionally downloading arbitrary blocks from full nodes, even if the client did connect to a full node controlled by the server, the server couldn’t be sure that the peer requesting a particular block was the peer it was targeting. This is true on regular IP, with increased privacy guarantees available to users of Tor ephemeral addresses or similar protocols.
[/quote]

I think this is the strongest argument for using full blocks vs "simplified utxos": reusing the Bitcoin p2p network makes it much harder for Mallory to collude and is also indistinguishable from BIP158 client traffic. I also think you make a very good point regarding the bandwidth usage: If $B$ is using a lot of bandwidth due to receiving many payments, its very likely $B$ can afford to run a full node. In fact, if $B$ is receiving a high volume of payments ( > 1 per day), its even more important they run their own node as this is the only way to trustlessly verify that these payments are in fact legitimate payments.

In summary, it seems to me the tradeoff is regular audits vs. more bandwidth: a single server returning tweak data, filters and simplified UTXOs with regular audits gives the same level of privacy as getting full block data from the p2p network at the cost of more bandwidth usage as a function of payment frequency.

-------------------------

harding | 2024-06-05 10:07:31 UTC | #14

[quote="josibake, post:13, topic:891"]
So for this attack to work, Mallory must control both the tweak server and the filter server to force a match,
[/quote]

If Mallory created one of the taproot-paying transactions in the block (e.g. Mallory pays Mallory'), then she can create a filter for an alternative transaction (not included in the block) where she used the same input(s) to pay the victim's SP address _x_.  That means she only needs control over the filter server.

[quote="josibake, post:13, topic:891"]
a single server returning tweak data, filters and simplified UTXOs with regular audits gives the same level of privacy as getting full block data from the p2p network
[/quote]

Strongly disagree here.  Audits only tell you that the server was honest in the past.  The victims of a recently compromised server probably won't find much solace in knowing that their loss of privacy was detected by auditors who will discourage others from using that server in the future.

[quote="josibake, post:13, topic:891"]
tweak data and simplified UTXOs are both public data sets, making it easy for anyone with access to the full blockchain to audit the data Mallory is returning and detect fake payments
[/quote]

They're only public datasets if everyone has the same block, but reorgs are possible, so auditing can be somewhat challenging.

There's an expensive version of the attack I describe where a completely legitimate block is created with only a transaction that matches the target wallet.  In that case, the tweaks and filters can be 100% legit and yet the server will still learn the network identity of the victim.

Between the free and expensive versions of the attack is a variant of the well-known dust-spamming attack where the server operator spends small amounts of bitcoin to the target _x_ SP address and keeps track of which network identities download the blocks of "simplified UTXOs" containing those transactions.  If network identity _X_ is the only one that downloaded all of the corresponding blocks, there's a high probability that they control the _x_ SP address.

In all these cases, acting like a BIP158 client (ideally using ephemeral Tor identities like Wasabi) significantly boosts the client's chance of remaining private.  Of course, operating a full node provides even stronger privacy because it performs exactly the same network operations whether transactions belong to the wallet or not (i.e., it has information theoretic perfect privacy).

-------------------------

josibake | 2024-06-05 12:02:10 UTC | #15

[quote="harding, post:14, topic:891"]
If Mallory created one of the taproot-paying transactions in the block (e.g. Mallory pays Mallory’), then she can create a filter for an alternative transaction (not included in the block) where she used the same input(s) to pay the victim’s SP address *x*. That means she only needs control over the filter server.
[/quote]

Ah, got it! So in this case Mallory is getting the tweak data included in the index through an honest payment and then reusing that same tweak to create fake outputs in the filter. This also defeats the auditing mechanism in that anyone auditing the tweak data would see a tweak corresponding to Mallory's honest payment. The only way to detect suspicious behavior would be to recreate the filter based on taproot data in the block and compare it to Mallory's filter: if they don't match, something funny is going on. So this pretty much invalidates everything I said in the previous post :sweat_smile: 


[quote="harding, post:14, topic:891"]
Strongly disagree here. Audits only tell you that the server was honest in the past. The victims of a recently compromised server probably won’t find much solace in knowing that their loss of privacy was detected by auditors who will discourage others from using that server in the future.
[/quote]

Fair point; this can only identify that a server *has* misbehaved, but if the attack is targeted the damage is already done.

[quote="harding, post:14, topic:891"]
In all these cases, acting like a BIP158 client (ideally using ephemeral Tor identities like Wasabi) significantly boosts the client’s chance of remaining private. Of course, operating a full node provides even stronger privacy because it performs exactly the same network operations whether transactions belong to the wallet or not (i.e., it has information theoretic perfect privacy).
[/quote]

You've convinced me that always using the full block is best, especially taking into consideration the savings from not downloading the full block is minor optimization w.r.t expected light client payment activity. Another advantage that occurred to me is it simplifies an SP light client protocol to "provide tweak data and a taproot filter," and leaves sourcing block data up to the client. It is worth mentioning parsing the full block is more work for a mobile client (vs "simplified UTXOs"), but again this extra work only happens when an output is found;t he overall goal of this protocol is to avoid having the client do lots of work for transactions that are *not* payments.

-------------------------

setavenger | 2024-06-08 19:50:22 UTC | #16

[quote="harding, post:12, topic:891"]
I’m concerned clients only performing step 5 (fetching transaction data) in response to untrusted data from step 1 (tweaks) and step 3 (pubkeys). If a client fetches all three pieces of data from the same server (or from different servers that are colluding), the server can potentially unmask users in step five by lying in steps 1 and 3.
[/quote]

Thanks for raising this issue!

[quote="harding, post:12, topic:891"]
What I think is best is similar to what Wasabi does with its custom BIP158 implementation:

1. Client downloads untrusted tweaks and filter from the server (ideally using something like an ephemeral Tor connection)
2. On match, client downloads the corresponding full block from a random full node (ideally using a different network identity, such as a different ephemeral Tor connection)
[/quote]

While I do agree with the general approach, I'm not sure downloading tweak data via tor will be feasible for everyone. I think an always/often-on scanning server with no hard time constraints can do that. Using mobile this might be a bit tricky. 

[quote="josibake, post:15, topic:891"]
You’ve convinced me that always using the full block is best, especially taking into consideration the savings from not downloading the full block is minor optimization w.r.t expected light client payment activity.
[/quote]

Would like to run a benchmark to see what the actual difference is w.r.t bandwidth. I can imagine cases where a wallet for donations might receive frequent payments leading to several blocks needing to be downloaded. 
If the actual bandwidth savings are low, going with blocks is a good way to go.

-------------------------

