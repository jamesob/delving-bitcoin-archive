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

