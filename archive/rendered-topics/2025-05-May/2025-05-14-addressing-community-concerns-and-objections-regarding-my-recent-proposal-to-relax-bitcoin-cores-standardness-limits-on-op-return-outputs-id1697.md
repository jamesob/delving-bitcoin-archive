# Addressing community concerns and objections regarding my recent proposal to relax Bitcoin Core's standardness limits on OP_RETURN outputs

AntoineP | 2025-05-14 15:47:13 UTC | #1

Hi everyone.

I recently proposed that Bitcoin Core lifts its standardness limits on `OP_RETURN` outputs. The reason is that they are not binding anymore (using an input witness to store arbitrary data is 4 times less expensive) but create perverse incentives for users of the public unconfirmed transactions relay network that need to store data specifically in transaction outputs. Such users turned to storing their data in fake public keys, creating forever-unspendable outputs that every single user will have to store in their UTxO set forever. For more details on the motivations for the change, and relay policy in general, see my [blog post](https://antoinep.com/posts/relay_policy_drama) on the topic.

This proposal was heavily mediatized, and severe mischaracterizations of the change being proposed led to genuine concerns among the community. A better communication from my part could have avoided unnecessary worries among bitcoiners and a lot of wasted time to everybody.

In an attempt to right this wrong, i have collected objections community members have raised across the board (on Github, the Bitcoin development mailing list, X, podcasts, at conferences, ..) to address them in this post. These are actual objections and concerns raised by community members, taken literally with little or no reformulation to address the precise statement.

## On the proposed `OP_RETURN` limits change

#### Schrödinger filters: there is a double standard whereby "Core" is claiming that filters don't work but actually they do because we need to change the `OP_RETURN` limit.

First, it is possible to store data onchain in larger `OP_RETURN` outputs than allowed by Bitcoin Core standardness limits by using [private bridges](https://slipstream.mara.com) or [alternative p2p relay](https://github.com/petertodd/bitcoin/tree/libre-relay-v29.0) networks.

However, those require relying on a central point of failure or a not as robust p2p relay network. These may be acceptable options for people willing to store arbitrary data, but not for time-sensitive transactions as typically used by Bitcoin scaling solutions. Developers of such solutions want to be able to broadcast their time-sensitive transactions through the public network of Core nodes because of the robustness and censorship resistance guarantees it provides.

In any case, the limit is not preventing those new applications from relaying their transactions through the public network. They just modify their transactions and use fake public keys instead. This method gives them standard transactions, but forces all Bitcoin users to store those outputs in the UTxO set forever because they are not spendable. This is why i suggested removing the limit, since they are going to store the data anyways just in a more harmful manner.

Therefore, the two are compatible. The limit does not prevent storing arbitrary data *by relying on a third party*. The limit does prevent applications from storing data through `OP_RETURN`s *and also use the public relay network*. Therefore the applications turn to still relaying through the public network, but to do so they use a more harmful way of storing the data than `OP_RETURN` outputs (namely, unspendable outputs).

#### Nirvana argument: this change is being pushed by people who believe the lack of a perfect solution to fight arbitrary data storage onchain means we should give up fighting it altogether.

No. This change is proposed to remove a perverse incentive for applications that want to store small amount of data in transaction outputs to use fake public keys instead of an `OP_RETURN`.

#### `OP_RETURN` was only ever made standard at all as a tolerated form of data carrying to prevent more harmful forms of data carrying (fake pubkeys/script hashes/bare multisig/etc). To serve that purpose, it only needs to be ~40 bytes. The ~80 byte allowance is beyond sufficient for its intended purpose.

What we believe an application may or may not need is irrelevant. This is what they store today, through fake public keys in unspendable outputs. The question is whether we make it possible for them to do it in a less harmful way.

#### The proper response to a spam attack is not to cave to the spammers.

I would personally disagree that a zero-knowledge proof (in the case of Citrea) for a Bitcoin scaling solution is "spam".

Now, what is qualified as "spam" does end up in the chain anyways. Just right now it would be stored in public key data by creating an unspendable output. Either way it ends up onchain, just in the latter case it is *also* stored forever by all Bitcoin users in their UTxO set.

Preventing "public key stuffing" is not on the table, therefore the next best thing is to at least offer a non-harmful way for application developers to store this small amount of data.

#### People won't move from using fake pubkeys to op_return to be nice to us is ludicrous because it'd cost them 4x more. This is just bluff to kill the filters.

This is incorrect. Using fake pubkeys is as expensive as using `OP_RETURN` outputs. In fact it is marginally more expensive because you need an above-dust output value and multiple outputs if you have more than 32 bytes of data to store.

Since it does not cost more (or even slightly less) to use `OP_RETURN` outputs in place of fake pubkeys, it is plausible that application developers would choose the least harmful way of storing data. That is, unless they have already built and deployed their application. Once it is in production, there is substantial cost to them to come back and change their construct to use a method that comes at no (or very little) benefit to them. This is why we need to act now, to make the less harmful option available before such applications are deployed.

#### Removing all limits on OP_RETURNs creates a perverse incentive: transactions can now carry large, non-economic payloads while avoiding long-term storage costs.

Transactions can already carry large payloads of data. Transactions pay a fee once, regardless of how long they are stored. Transactions are always stored forever by unpruned nodes. The proposed change does not affect this.

Relaying larger `OP_RETURN` outputs on the public network may marginally reduce the cost of using them. However the data stored in `OP_RETURN` outputs cost 4 times as much as data stuffed in an input witness, which is standard today and already relayed on the public network.

#### A given quantity of data stored in a op_return output has a far larger negative externality on the throughput than the same data stored in a witness.

This objection points to the fact that since the witness data is discounted, using it leaves more block space for other transactions than storing the same amount of data in an `OP_RETURN` output. However, since the witness data is discounted it also costs the user 4 times less. Therefore for a given fixed cost a user may use as much block space with either method.

#### Boiling frog: Core developers are trying to gradually make Bitcoin a universal database instead of just money, one step at a time.

This is an unfounded accusation attempting to harm the very people who have been maintaining the Bitcoin network for the past >15 years. By default we should be very skeptical of the claim.

Regardless of the ethics of this attack, it does not contain any argument to address. We should focus on the substance of arguments and not speculate about would-be motivations of people involved in the discussion.

Contributors in favour of this change have provided extended rationale for their support as well as detailed technical argumentation for why this change is safe and bolsters Bitcoin’s decentralization and thus monetary properties. Therefore unless arguments are presented to back it, we can safely ignore this claim.

#### Core is trying to accommodate the Taproot Wizards who are a self-declared attack on Bitcoin which tries to corrupt the ecosystem.

This is also an unfounded accusation. Likewise, the presumption should be in favour of people with a proven track record of maintaining the Bitcoin network for over a decade (of course this does not mean their opinions shouldnt be taken blindly either). Likewise, it should be ignored or even rejected (and its author treated with suspicion) unless tangible proof to back up this serious accusation is presented.

The author has not provided any argument for why this change would accommodate the "Taproot Wizards" company. Even if it did, that a change benefits a company is not a reason to prevent it if it benefits the Bitcoin network and its users. A change should be weighed on its own merits and not based on the (speculated) motivations of its author.

A decision process which values the author's identity or speculated motivations for a change over the substance of a change is extremely brittle. It could be exploited to push through a change negatively affecting the system or to prevent achieving a change positively affecting the system. A sufficiently robust decision process necessarily puts first and foremost the content of an argument, not who makes the argument.

#### Core contributors actually believe that Bitcoin should be used for anything rather than just money. They should argue that instead of rationalizing with dishonest arguments such as UTxO bloat which is not the real reason they are pushing it.

This is yet another speculation on the motivation of people engaged in the discussion. Ultimately a proposed change will be evaluated based on its content, not the would-be motivations of the author. If someone believes the change is bad they should argue against it, not try to speculate about the motivations of the author or other people in favour of the change.

Besides, at least some Core contributors [publicly stated](https://gnusha.org/pi/bitcoindev/QMywWcEgJgWmiQzASR17Dt42oLGgG-t3bkf0vzGemDVNVnvVaD64eM34nOQHlBLv8nDmeBEyTXvBUkM2hZEfjwMTrzzoLl1_62MYPz8ZThs=@wuille.net) they believe storing JPEGs onchain is stupid and/or wished demand for this usage would go away. This view is compatible with the view that the `OP_RETURN` limit should be removed.

#### Antoine Poinsot paid Peter Todd to open the pull request to Bitcoin Core.

Neither myself, nor any other Bitcoin Core contributor pay Peter Todd to open the Github pull request to Bitcoin Core changing the `OP_RETURN` standardness limits ([#32359](https://github.com/bitcoin/bitcoin/pull/32359)). I emailed Peter Todd to re-open his previous pull request ([#28130](https://github.com/bitcoin/bitcoin/pull/28130)) which implemented what i was proposing. Opening a PR on my own with the same code would be a bad look: it's essentially stealing credits. I guess i also wanted to let Peter have his "W": he was proven right that keeping this unnecessary limit would cause more harm than good.

That being said, whether i (or anyone else) paid Peter Todd to propose a change to Bitcoin Core should not matter. If proposed changes are considered on the basis of who proposes them rather than their content, we have a much bigger problem than the standardness limit on the size of `OP_RETURN` outputs. Fortunately, this is not the case. This can be exhausting and frustrating, but the Bitcoin Core decision process ultimately evaluates changes on their own merits, not based on who proposes them.

#### Taking a step back from the technical details, Core is trying to push a contentious change that obviously everybody is annoyed with.

This is not obvious, at all. I would posit that most Bitcoin users do not care about this change. I would even go further and posit that if honestly presented the change and its rationale, most Bitcoin users would decide they don't need to care about it. Maybe from within the social media bubble of some Bitcoin enthusiasts, it appears to be undesirable (although even there there seems to be a large number of people in favour).

That said, the change being controversial among Bitcoin enthusiasts is an issue. I believe cohesion among different parts of the Bitcoin community is important and if people have genuine concerns about how a change to Bitcoin Core is carried out, those should be addressed.

The genuine concerns are the result of severe mischaracterizations of the proposed change. I hope this post as well as my previous [presentation of the change](https://antoinep.com/posts/relay_policy_drama) help clear up the misunderstandings.

#### As the reference implementation, Core has a responsibility of steering away from controversy as much as possible. It is irresponsible for Core to merge this change because of the number of people objecting to it on the Github pull request.

As the reference implementation, i think Core should be conservative and always err on the side of caution when in doubt. It is not the same as steering away from controversy, which is clearly impossible but also undesirable. Bitcoin users should be the primary concern of Bitcoin Core. This may clash with the goal of steering away from controversy, as something which benefits all Bitcoin users may be (or made to be) controversial.

There is many things wrong with the second point. First of all, Bitcoin Core should not make decisions based on headcounts but on the merit of the change itself. Second of all, Bitcoin Core should weigh all dispersed users equally and not favour a concentrated minority voicing their opinion on the pull request. Third of all, there is a bias towards considering objections to the change (since those are concentrated on a single pull request thread) more than objections to keeping the status quo (since there is not a single pull request thread centralizing those).

#### The way this is being rushed is a red flag.

None of this is being rushed. Peter Todd initially [proposed](https://github.com/bitcoin/bitcoin/pull/28130) this change 2 years ago. My [email to the mailing list](https://gnusha.org/pi/bitcoindev/rhfyCHr4RfaEalbfGejVdolYCVWIyf84PT2062DQbs5-eU8BPYty5sGyvI3hKeRZQtVC7rn_ugjUWFnWCymz9e9Chbn7FjWJePllFhZRKYk=@protonmail.com) was on April 17th. It received some amount of discussion in the following days and went silent for about a week until Peter Todd opened the [pull request](https://github.com/bitcoin/bitcoin/pull/32359) implementing the change to the Bitcoin Core Github repository on April 27th. The pull request did not get merged (in fact it was now closed), but even if it did it would have taken another 5 months before the change is released to Bitcoin users and probably about a year before it gets widely adopted on the network.

Bitcoin Core does not rush changes. Although the proposed change would have close to no consequence for Bitcoin users not taking advantage of `OP_RETURN` outputs, it is no exception.

#### This is making changes for the sake of making changes when nobody asked for this.

The proposed change was motivated: it intends to remove a perverse incentive pushing users to use fake public keys to store data in transaction outputs instead of `OP_RETURN` outputs.

#### This proposal won't allow to reach the stated goal (limiting the misuse of witness data).

Limiting the misuse of witness data is not the stated goal of the proposal. The goal of the proposal is to remove a perverse incentive pushing users to use fake public keys to store data in transaction outputs instead of `OP_RETURN` outputs.

#### Introducing filters is what Satoshi did, this is a departure from his legacy.

This is an appeal to authority. It is also incorrect.

Although who gives an argument is often a good smoke test to decide whether to spend time thinking about it or rebutting it, arguments ultimately need to be weighed on their own merits. Needless to say, Satoshi has been both right and wrong. Fetishizing his past opinions is working against making good decisions.

Also, it is Gavin Andresen who introduced standardness in commit [`a206a23980c15cacf39d267c509bd70c23c94bfa`](https://github.com/bitcoin/bitcoin/commit/a206a23980c15cacf39d267c509bd70c23c94bfa) not Satoshi. Standardness was introduced for DoS reasons, not to hinder propagation of undesired usage.

#### This type of change should be treated with no less scrutiny than a hard fork, because that's essentially what this is

This change does not touch consensus rules. This change is therefore not a hard fork.

#### That may sound like hyperbole, but it really isn't. If this PR is merged, Bitcoin as we know it changes forever in the most fundamental way imaginable: the reference implementation explicitly turning the Bitcoin network into an arbitrary data storage system, instead of evolving it as a decentralized currency.

It is hyperbole. This change does not affect users willing to store large amounts of data simply because data in an `OP_RETURN` output costs 4x more than data stuffed into an input witness. The latter is the method commonly used today to store large amounts of data, notably through the widely used "inscriptions". It is completely unplausible such uses would move to use a method 4 times more expensive.

This change will affect applications that need, although it's more expensive, to store a small amount of data in the output(s) of a transaction for technical reasons. If they used multiple unspendable outputs today, it would make it marginally cheaper for them to use a single `OP_RETURN` output instead. This is good as it incentivizes less harmful behaviour (albeit only marginally).

Therefore, that lifting the standardness size limit on `OP_RETURN` outputs will "change Bitcoin forever in the most fundamental way imaginable" is a wild claim that is not backed by anything.

#### While provably unspendable outputs are preferable to dust outputs in some contexts, the absence of any guardrails could make it easier for bad actors to stress the network, especially in combination with existing vulnerabilities.

Without a clear definition of what the author means by "stressing the network" or a description of the supposed vulnerabilities being referenced, there is no way to tell if this is a reasonable objection or an unsubstantiated vague claim.

#### Data stored in an OP_RETURN output doesn't need to be chunked like it does in an inscription. This presents a risk for node runners as it allows anyone to store unobfuscated data on their disk.

Onchain data is obfuscated by Bitcoin Core when stored on disk. Bitcoin Core only started doing so recently, and an existing block chain on disk will not be obfuscated when upgrading. Therefore old nodes may store unobfuscated `OP_RETURN` data that ends up onchain. However this is already possible at a trivial cost to an attacker: for instance as i write this post Slipstream's feerate is 6 sats/vb so storing a 10k bytes malware file in an `OP_RETURN` would cost only ~$60 to an attacker (at $100k/BTC). Note this cost is the total cost, not just the premium. This total cost is also lower than it would have been if the attacker had to pay the past 5 years average feerate. Therefore this change does not materially affect this concern.

This change would however directly affect mempool data (which is also stored on disk) since a large `OP_RETURN` would previously not be accepted in a Bitcoin Core node's mempool. But mempool data is now always obfuscated when stored to disk, therefore this is not an issue.

#### Considering the floodgates of absolutely horror this PR is going to bring upon Bitcoin node runners, there absolutely needs to be a way for existing users to protect themselves and retroactively apply this on-disk obfuscation to existing node data.

As explained in the previous section, the concern of storing harmful unobfuscated data exists regardless of the change. The change does not materially exacerbate it. Bitcoin Core user worried about this can use [this tool](https://gnusha.org/pi/bitcoindev/1e353962-1665-4bc5-8a35-e349fdf4832cn@googlegroups.com) from Bitcoin Core contributor Andrew Toth to obfuscate block chain data on their node without having to resync.

#### This PR lowers the bar for storing large contiguous chunks arbitrary data on the systems of thousands of node runners worldwide from "you must get a miner to mine this for you" down to "you have to broadcast it and pay a fee".

Nowadays "you must get a miner to mine this large `OP_RETURN` for you" really means "you must be able to use a website". This is not exactly a very high bar, therefore this change does not materially lower it.

#### I think the bar to bypass IsStandard is very underestimated. The fact that we don't see huge OP_RETURNs all the time proves its overall effectiveness in protecting the network as a first line of defense.

This is confusing correlation and causation. There may simply not have been enough demand for large `OP_RETURN` outputs, after all `OP_RETURN` data is 4 times more expensive than witness data (used for instance by inscriptions).

It is also trivial to get large `OP_RETURN` outputs mined. For instance Ben Carman's [OP_RETURN bot](https://opreturnbot.com) accepts large messages and MARA's [Slipstream](https://slipstream.mara.com) accepts large `OP_RETURN`s.

#### The `-datacarrier` option should be kept on Bitcoin Core regardless of whether the default is changed.

Essentially the argument goes like:
> You say that once the default is changed, setting `-datacarrier` on my node will have no observable effect at the network level. Fair enough, but this does not justify removing the option as I want control over what goes into my mempool and what I am facilitating to relay.

This is an objection to the implementation of the change rather than the change itself. I personally disagree and will argue against, but either way it should not be treated as an objection to making the change. In fact the pull request implementing the change in a manner that removes the option was closed (see [#32359](https://github.com/bitcoin/bitcoin/pull/32359)) in favour of another pull request which does the same but keeps the startup option (see [#32406](https://github.com/bitcoin/bitcoin/pull/32406)).

Bitcoin Core does not provide an option for all possible ways the relay policy may be tweaked, because there are too many ways it can be tweaked and this does not provide value to the user. Bitcoin Core should only have options that provide value to the user. If the default becomes to not limit the size of `OP_RETURN` outputs, then manually changing the option cannot conceivably have any network-level impact (defaults are sticky). Even if it has no global effect, one may still think it could be useful for someone locally. However the only effect this would have locally is to shoot a transacting user in the foot because they would be blinded to part of the transaction traffic they are competing for block space with, and would shoot a mining user because they would be blinded to part of the traffic using block space and it would therefore slow down their reception of blocks by adding roundtrips to fetch transaction they were blinded to.

Because the option does not provide value to a user if the limit is lifted by default, i believe a pull request lifting the limit should remove the option. However i won't die on this hill, and it seems most other Bitcoin Core contributors favour the implementation that does not remove the startup option ([#32406](https://github.com/bitcoin/bitcoin/pull/32406)).

#### As a node runner I should be allowed to configure my mempool and not have it filled with what I consider spam. This can make running a node very RAM expensive.

Regarding the configuration option, see the previous point. Regarding mempool RAM usage, it is limited by default and relaying transactions using larger `OP_RETURN` data does not negatively impact RAM usage compared to other types of transaction.

#### This PR is completely pointless from a technical and incentives perspective unless combined with something like [#28408](github.com/bitcoin/bitcoin/pull/28408) to make OP_RETURN the "proper" way to store data.

This is missing the point of the pull request. That it does not incentivize to use `OP_RETURN` outputs to store large amounts of data is on purpose. It does not intend to make `OP_RETURN` outputs the "proper" way to store data. It intends to offer a less harmful alternative to using fake public keys in unspendable outputs for applications that need to store a small amount of data in a transaction output.

#### I am also concerned that in the future we might see a large scale bypassing of the Core standarness rules, but it’s unclear whether we’ve reached that point yet. We can always reconsider dropping the OP_RETURN limit once it becomes truly problematic - but, and this is a genuine question, are we there yet?

It seems unlikely that we see a large scale bypassing of Core's standardness rule unless there is significant demand for a certain type of transactions. If this is the case, Core can (and should) adapt its standardness rules.

Dropping the `OP_RETURN` size limit is unrelated to whether Core's standardness rules are being bypassed. Application developers just use other, standard, means of storing data instead. Only these alternative means (namely fake public keys in unspendable outputs) are more harmful than simply using an `OP_RETURN`. Dropping the limits is proposed to offer a less harmful alternative, not to prevent them from bypassing the rules.

#### Raising UTxO bloat as a concern is beyond ironic when considering applications like Citrea may only cause trivial amount of bloat while inscriptions, which Core refuses to filter, are responsible for over 8 GiB of bloat in the past couple of years.

Raising the `OP_RETURN` limit and filtering inscriptions are two separate issues. All other things equal in the current situation, lifting the `OP_RETURN` limit will not incentivize more data storage but will remove a perverse incentive to bloat the UTxO set with **unspendable** outputs. This may have only a marginal impact when considering only Citrea. But Citrea is the canary in the coal mine for the presence of perverse incentives (for instance other BitVM side-systems will probably use similar techniques). We should act before the incentives currently in place lead to more bloat from future applications that may or may not hear our concerns about the harm inflicted on all Bitcoin users.

This point is also [discussed below](#on-the-broader-point-of-relay-filters) in the separate question of whether Bitcoin Core should try to prevent relay of inscriptions.

#### Bitcoin Core dev should rather focus on Bitcoin as money.

This statement contains a built-in assumption which remains to be shown. It is also absurd to pre-suppose that Core devs, who as a group have been maintaining Bitcoin for more than a decade, have not been "focused on Bitcoin as money". The seemingly tautological statement "Bitcoin as money" itself remains to be defined.

Lifting the size restriction on `OP_RETURN` outputs is not an endorsement of a specific (valid) usecase of the Bitcoin network. It is simply a realization that the limit is doing more harm than good by creating perverse incentives.

#### Bitcoin Core lifting the `OP_RETURN` limit is sending a signal that data storage is welcome, whereas before it was only tolerated.

The motivation for the proposed change has been clearly stated. Whether someone would read into it that "storing large amounts of data on Bitcoin is now welcome" rather than what is actually written ("as you are already storing data, please use this less harmful manner instead") is ultimately a matter of subjective interpretation.

More importantly, we are well past the point of "sending a signal". The main driver of block space demand in the past years has been onchain data storage, by means orthogonal to the current proposal. If people want to store data, they can already do it, in a way that is 4 times cheaper.

#### Increasing the size of `OP_RETURN` is an existential threat to Bitcoin.

It is not. If it was, Bitcoin would be utterly uninteresting to begin with. What makes Bitcoin interesting is that it can provide censorship-resistant money in an adversarial scenario (with real attackers, not simple graffitis). Some flaws make it more brittle than most imagine, but if it was that reliant on the good will of its participants it would be too fragile to be worth anything.

#### If you don't upgrade Core devs are going to accuse you of censorship.

This is a baseless accusation. It cannot be accepted as an argument against raising the standardness limit on the size of `OP_RETURN` outputs.

This claim assumes that if newer version of Bitcoin Core loosen their standardness rules, they will treat older versions with tighter relay rules as "censoring" transactions. This would lead to contributors to the Bitcoin Core project then going after people who haven't yet upgraded their node and insulting them of being censors. This is absurd.

The confusion probably arises from the loaded language. In transaction relay, a node may relay a transaction that is not accepted by another. This is fine, happens regardless of standardness rule changes (for instance your peer has a smaller mempool min fee, has a conflicting transaction, etc..), and is not called "censorship" anywhere. Furthermore, Bitcoin Core frequently loosens its standardness rules: to accept version 3 transactions, to allow to send to future Segwit versions, to allow to send to an `OP_RETURN` script, etc... Never was it used by Bitcoin Core contributors to go after users and insult them (this feels silly to even type).

## On the broader point of relay filters

Although **it is not affected by the proposal**, discussions surrounding `OP_RETURN` limits inevitably restarted the old debate about filtering inscriptions. As these two separate topics often get discussed as one, i thought it would be productive to also address objections to the lack of filtering of inscriptions in Bitcoin Core.

I think Bitcoin Core contributors should put together a post presenting the project's vision about mempool policy (and so should Knots). This post is not that, i want to focus here on addressing specific objections raised. Failing that, i think [this](https://gnusha.org/pi/bitcoindev/9c50244f-0ca0-40a5-8b76-01ba0d67ec1bn@googlegroups.com), [this](https://gnusha.org/pi/bitcoindev/QMywWcEgJgWmiQzASR17Dt42oLGgG-t3bkf0vzGemDVNVnvVaD64eM34nOQHlBLv8nDmeBEyTXvBUkM2hZEfjwMTrzzoLl1_62MYPz8ZThs=@wuille.net) and [this](https://gnusha.org/pi/bitcoindev/Y9PPvBiWOXpBefmD@camus) posts present fairly consensual views among Bitcoin Core contributors working on relay policy adjacent stuff.

#### Bitcoin always had filters. Not reacting to inscriptions is departing from 15 years of precedent.

Most Bitcoin nodes on the network have historically been Bitcoin Core nodes, which enforces tighter constraints (standardness rules) on unconfirmed transactions it relays. Standardness rules were [introduced for DoS protection](https://github.com/bitcoin/bitcoin/pull/29769#issuecomment-2029843185). They are also used to provide upgrade hooks, making soft forks substantially safer. Finally they were also used as a mild nudge to direct some usages in a certain direction.

The attempt at renaming standardness rules into "filters" creates confusion about what they can achieve. They cannot block or materially hinder usage of valid transactions for which there is substantial economic demand, as the "filters" renaming would imply. At best they can disincentivize isolated usages for which there is very small economic demand, or nudge application design in a specific direction (as was done with the standardization of a limited `OP_RETURN` output type in 2013). For instance, they can nudge application developers to use a specific design before the application is built, but once it is being actively used there is no going back.

It was always possible to store large amounts of data in Bitcoin transactions. By effectively increasing the block size, Segwit necessarily also increased the data storage surface. We did not see it because there was just no substantial economic demand for it. Had there been, Bitcoin Core's standardness rules would have had to adapt (as they did when introducing the `OP_RETURN` output type).

Adapting the Bitcoin Core standardness rules to the economical reality of the Bitcoin network usage is necessary and not ahistorical.

#### Relay filters have historically been successful at preventing "spam".

The Bitcoin Core standardness rules have historically matched the demanded usage(s) of the Bitcoin network. There is no evidence of any unwanted usage that was stiffled by tightening standardness rules. Therefore this assertion is incorrect and can only resort to unfalsifiable counterfactuals ("had the standardness rules been looser we would have seen more unwanted usage").

#### Raising UTxO bloat as a concern is ironic when considering applications like Citrea may only cause trivial amount of bloat while inscriptions, which Core refuses to filter, are responsible for over 8 GiB of bloat in the past couple of years.

This is confusing UTxO set usage and UTxO set bloat. While there is no normative language for Bitcoin concepts, the latter usually refers to **forever unspendable** outputs. No matter what we do those outputs will sit forever in every single user's UTxO set. Spendable outputs, even if uneconomical to spend, can always be swept by someone incentivized out of band or a good samaritan (as was done in the past).

In addition to being more harmful, unspendable outputs are also easier to fix by stopping to actively prevent users of the public relay network from using a less harmful method. Uneconomical output creation is a whole other ball game, and not specific to inscriptions.

Finally, UTxO set bloat has been primarily driven by BRC-20 and Stamps. The former is a metaprotocol which only needs a small amount of metadata. The latter is a self-declared attack on Bitcoin. While it may be possible to work against the storage of large amounts of data, metaprotocols will unfortunately always be possible. Stamps, especially as they recently moved to using P2WSH, are indistinguishible from legitimate outputs.

#### See, inscription respect the dust limit therefore filters are working!

This statement is misleading due to the lack of definition for "working". This assertion is often used to claim that standardness rules are able to effectively deter unwanted usage (here dust outputs), but the fuzzy definition lets the person turn around and state that "working" means any cost above zero (even if trivially low).

Even assuming that inscriptions would rather use 0-value outputs, this merely proves that the cost of re-designing the inscription protocol on top of private bridges or alternative p2p relay networks is higher than the few hundreds sats (a few dozen cents at $100k/BTC) necessary to create an above dust output when making a transfer carrying 100000 times more value to its user (whatever we think of this valuation).

Therefore asserting that standarness rules are an effective deterrent against unwanted behaviour is incorrect. Imagine if Bitcoin Core raised its standardness dust limit to 10k sats, 100k sats, 1M sats? Do you think Bitcoin users would similarly increase their expenses or... just ditch Bitcoin Core?

#### Filtering will reduce supply and therefore suppress demand for inscriptions by increasing the cost of using them.

This argument posits that 1) additional standardness rules in Bitcoin Core would significantly reduce the availability of inscriptions and 2) that the rarefication of inscriptions would drive down demand for them until they become irrelevant and stop using Bitcoin block space.

As discussed in the previous sections, 1) is already incorrect. Bitcoin Core updating its standardness rules to prevent the relay of inscriptions (assuming it can even win the cat-and-mouse game) would only lead to a slight, and temporary, cost increase to their users. They would have to switch to private bridges to miners and alternative p2p relay networks.

Even if 1) was correct, 2) does not follow. Reducing supply for a good does not (all other things equal) drive down demand for it. In addition, we can say from experience that in the meme-based economy of inscriptions (and NFTs more generally) making something rarer may (significantly) drive demand for it. More demand would create a strong incentive for creating more supply, resulting in the opposite effect to the one intended.

#### Bitcoin should not try to optimize for more things than being money. Anything else is diluting focus.

I don't think this claim is controversial, but it is vague which may be misleading. Bitcoin is not someone, it's not optimizing for anything. Bitcoin is a well-defined, existing, system. I think the intended meaning here "Bitcoin should be designed for payments, not data storage". Fine, but Bitcoin is already designed.

You may argue we (who's we?) need to change Bitcoin's design but this is orthogonal to making inscriptions non-standard by Bitcoin Core relay policy.

#### We should not be bidding for block space against non-monetary usecases.

This implies Bitcoin needs to be changed to prevent storing arbitrary data onchain. This is not possible to do entirely. It may be possible to limit the quantity of arbitrary data that can be stored onchain at once, but this does not achieve the stated goal (because data storage is still possible, only reduced, and because meta-protocols are still possible) and comes at the cost of an enormous change to how Bitcoin operates which is likely to be controversial (for good reasons).

In any case this is orthogonal to whether Bitcoin Core should relay inscriptions.

I would like to also add that by Bitcoin's very nature, it will always be necessary to bid for block space against other users. Constrained resources and other people's usage driving up the price for everyone was at the core of Bitcoin's greatest historical controversy and we should be wary of anyone fanning these flames of resentment. Changing Bitcoin to partially prevent onchain data storage may significantly hinder progress toward making Bitcoin more scalable and therefore accessible to a greater number of users simultaneously.

#### This is the first step toward preventing arbitrary data storage at the consensus level. The threat of making it consensus invalid may be enough to deter usage entirely.

Then a consensus change needs to actually be proposed, and proponents need to rally support for it from all Bitcoin users. However i don't think such a controversial consensus change, which massively changes Bitcoin's inner working without completely achieving its stated goals, stands any chance of being adopted by Bitcoin users. Which is probably why it was never actually proposed in the first place.

## On the separate question of Github moderation

Moderation on the Bitcoin Core Github repository further worsened perception of the proposed change. Although it is necessarily connected, this is a separate topic from whether the change is desirable or not. For a discussion of Bitcoin Core's Github moderation policy see @glozow's [post on the topic](https://x.com/glozow/status/1918379285045211402).

-------------------------

