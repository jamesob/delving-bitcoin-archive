# Pluggable Channel Factories

ZmnSCPxj | 2024-11-11 16:24:40 UTC | #1

As proposed in my SuperScalar document, I want to create an extension supported by common node software for "pluggable channel factories".

Introduction
=========

Several existing Lightning Network node software (CLN, LDK, Eclair, LND) have some concept of "plugin" or "extension" that allows users of the software to extend the base node software in various ways.

Now, channel factories allow channels to be hosted in constructions that are themselves offchain, including my SuperScalar proposal.  In most such designs, the channels in a multiparty channel factory would look just like the current BOLT-specified blockchain-backed channels.  Thus, the existing code for managing channel state and forwarding payments would still work with channels inside channel factories.  However, the channels themselves would be effectively "0-conf" as far as the existing software would know about it; that is, the funding transaction --- the transaction that directly instantiates the channel --- would be unconfirmed, or in other words, kept offchain.

Thus, I propose that we create a general protocol for pluggable channel factories, where the base BOLT-talking node software agree that some 0-conf channel is actually hosted by a channel factory under some instance of some channel factory protocol.  For most operations in BOLT2, the channel software will work without modification, only treating the channel as 0-conf.

My concrete proposal is to include new TLVs for `open_channel` (i.,e. the openv1 flow) that indicate that a new channel is actually hosted in a channel factory, and requiring that the base node software be connected to a channel factory plugin (***how*** is unspecified, only that it is somehow plugged in; it could be a built-in supported channel factory, for instance) of a specific channel factory protocol.  Then, when events becomes necessary, the channel factory plugin can be contacted by the base software to e.g. unilaterally close due to an impending HTLC timeout.

Establishing Channels Inside Channel Factory
=====

A channel can be established with a new `channel_factory` even TLV for `open_channel`. The TLV indicates:

1. A protocol identifier.  This differentiates e.g. SuperScalar from other novel channel-hosting protocols.
2. An instance-of-the-protocol identifier. This differentiates one instance of SuperScalar from another instance,
3. A number-of-blocks-early count.
  - Normally, if an HTLC timelock will expire "soon", the base node software will unilaterally clsoe the channel.
  - However, some factory protocols may require relative timelocks along the path to an actual channel.  This means that the channel factory plugin would need to initiate unilateral exit ***earlier*** than normal.
  - So, in case of an HTLC timelock, this number-of-blocks-early count would impose a minimum timelock allowed, and when the base software needs to signal the channel factory plugin to begin unilateral exit in time to enforce the HTLC.
  - This needs to be established between the base software nodes, as this imposes a shared minimum timelock allowed between them.

If the open is accepted and the initial commitment signing is completed, the channel factory plugins can continue with the establishment of the channel factory (or if the establishment fails (i.e. a potential participant goes offline before it can give necessary signatures) the hosted channels can be cancelled with an `error`).  The `channel_ready` messages would have to be deferred until the plugin actually indicates completion of the factory establishment.

Riding Channel Factories Off Splicing Code
=====

An important fact of channel factories is that as the channel factory state changes, the funding transaction for channels changes.  Thus, a piece of code that handles channel state and forwarding across channels would need to be able to adapt to changes in state.  And because the factory will have more than 2 parties, there is a chance that a change in factory state does not complete due to a lack of full consensus among all parties (i.e. a participant goes offline just before it can send a signature for the new channel factory state).  Thus, when moving between old channel  factory state to new channel factory state, the code that handles channel states needs to handle multiple possible channel funding outpoints simultaneously, at least until the factory protocol has established whether to move to the new state or revert to the current state.

So in summary, in order to support channel factories of any kind, the code that handles channel state:

1.  Needs to be able to change the funding outpoint of the channel.
5. Needs to be able to have multiple funding outpoints of the channel as "valid" simultaneously, until the channel factory can resolve which one is "real".

As it happens, splicing ***has the same problems***:

1. After splicing, the funding outpoint of the channel changes.
2. Until the splicing transaction confirms, the channel may be unilaterally closed using the old funding outpoint or the post-splicing funding outpoint, until the blockchain is able to resolve which one is "real".

Thus, I propose to reuse parts of the channel splicing process for channel factory state changes:

1.  Quiesce the channel first (`stfu`)
2. Message that says "the factory protocol software plugin connected to me said we should go to this new funding outpoint X" and a reply that agrees.
  - This ends the  quiescence state, just like in splicing when the participants have agreed on the new transaction inputs and outputs.
6. In the meantime, use multiple `commitment_signed` messages with `batch` TLVs to sign for both the current factory state and the new factory state.
7. Message that says "the factory prtocol software plugin connected to me said it was able to decide to use the old/new funding outpoint" and a reply that agrees
  - This ends the multi-`commitment_signed` state, just like in splicing when the splice transaction is confirmed or it becomes invalid due to an input being spent in an alternative way.

-------------------------

renepickhardt | 2024-11-14 12:45:29 UTC | #2

I have been wondering about channel factories and how to extend the bolts to support them quite a bit. For example I think besides your SuperScalar proposal VTXOs in Ark are a natural candidate to be funding TX for channels. 

While many LSPs consider channel factories a proper use for the last mile problem and are willing to consider unannounced channels I am wondering about extensions to [BOLT7](https://github.com/lightning/bolts/blob/aa5207aeaa32d841353dd2df3ce725a4046d528d/07-routing-gossip.md) where it states for example: 

> A node:
>
> * SHOULD monitor the funding transactions in the blockchain, to identify  channels that are being closed.
> * if the funding output of a channel is spent and received 12 block confirmations:
>  * SHOULD be removed from the local network view AND be considered closed.

Channels within a factory cannot be monitored on the blockchain. Similarly the announcement of such channels will be tricky as bolt7 currently states: 

> * MUST set `bitcoin_key_1` and `bitcoin_key_2` to `node_id_1` and `node_id_2`'s respective `funding_pubkey`s.

However as [laid out in my research I believe routing nodes will eventually need channel factories for their liquidity management to fulfill and facilitate payment requests](https://github.com/renepickhardt/Lightning-Network-Limitations). Thus those channels may need to be announced and we should wonder how we can extend gossip to do so. I understand that there are ideas in future gossip to remove the tight coupling and spam prevention. 

In any case I believe a protocol for pluggable channel factories should give providers of channel factories an API to sign of that they created a channel and a method for to nodes to monitor if the channel was closed.

-------------------------

ZmnSCPxj | 2024-11-14 15:04:03 UTC | #3

>>A node:
>>
>>* SHOULD  monitor the funding transactions in the blockchain, to identify channels that are being closed.
>
> ...

Indeed, the equivalent of BOLT7 requirement to monitor the blockchain for unilateral exit of the factory would need to be implemented by the plugin, not the base software.

A note however is that even if the factory layer is closed and its hosted channels directly published onchain, the channel can keep operating. So the base node software would still monitor the blockchain as normal for channels inside channel factories; even if the factory layer is closed, it may end up with channels directly onchain instead, and the channels can still continue to operate as channels.

Of course, there may be factories where closing the factory also closes the hosted channels (possibly in some edge cases, such as a single SuperScalar instance timing out and being resolved using the timeout branch, where channels are never published; this "should not" happen, but accidents can happen, such as an LSP getting forcibly shut down somehow) so there is a need for the factory plugin to inform its attached base node software to forget an in-factory channel.

> Similarly the announcement of such channels will be tricky...

Some years back (I forgot exactly when) there was some discussion about generalizing gossip, where a node only needs to point to some onchain fund that it can show proof it has shared n-of-n control over, and that would let the node gossip some number of channels (i.e. it shows 1 mBTC of onchain fund that it can prove it has one shard of n-of-n control over, we let it gossip up to N * 1mBTC of total channel capacity).

This is because Lightning uses the Bitcoin blockchain UTXO set to limit the size of the gossiped public network; this is the REAL reason we even require channel announcements to point to an unspent TXO.

So I think that gossip change needs to be put in place first before we can discuss gossip for channels inside factories.  So for now, I am deliberately saying "out of scope" for this bit; I want to focus on the protocol that gets channels hosted in factories.

(off-topic, but for the public network at least, instead of hosting channels inside a factory, an alternative is my sidepools idea. Basically, instead of a N > 2 offchain mechanism containing N=2 mechanisms, we have N=2 mechanisms directly on the blockchain and a parallel N > 2 mechanism ("to the side", hence "sidepool"). The public routing nodes can then swap funds on the N > 2 mechanism to "reload" capacity on the N=2 mechanisms. This gets us some of the nice properties of both worlds: no need to change the gossip protocol (all the existing public 2-of-2 channels remain as plain onchain 2-of-2 channels and existing pathfinding algos that assume nodes and 2-party channels remain operational, and still using the Bitcoin UTXO set as a backstop against uncontrolled growth of the gossiped public network) while still allowing effective use of liquidity (with something better than Decker-Wattenhofer, we can swap on the N > 2 mechanism "JIT", effectively doing a "JIT rebalance" whenever a channel is depleted and we need to forward over it; this requires all parties to be online, but public forwarding nodes have a strong incentive to be as close to 100% uptime anyway; even with just Decker-Wattenhofer we can probably do a single large "everyone, rebalance!" once a day and that will still get us 90% of the benefit of full N > 2 mechanisms).)

-------------------------

ZmnSCPxj | 2024-12-02 16:11:13 UTC | #4

Concrete bLIP spec: https://github.com/lightning/blips/pull/56

-------------------------

