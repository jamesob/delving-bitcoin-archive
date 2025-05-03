# Behind the OP_RETURN Drama

pandacute | 2025-05-02 08:33:27 UTC | #1

I think the whole OP_RETURN debate is missing one important part: a communication link between the developers and the users. The developers are speaking with the developers, the users are speaking with the users, and no one is really trying to understand where the other side is coming from. It's been a lot of aggressive, unproductive back-and-forth, with both sides accusing each other of bad intentions and completely dismissing each other's points.

This is a technical forum aimed at devs, so I'll try to summarize what I've seen in normal_users_land, and the general feeling out there right now. Obviously, I'm not trying to suggest that users *feelings* should be dictate the technical direction of Bitcoin. However, it is important that there is mutual respect and trust between users and devs.

Right now, trust is pretty broken. It can be regained, but not without understanding what went wrong in the first place. 

I saw on IRC some devs suggesting they spend time on podcasts and social media to explain the motivations behind this proposal *before* merging it, instead of merging now and hoping people just accept it. I think that’s a fantastic idea, and the way to go. I particularly liked jonatack's message - “if Core shows humility and patience, it would surprise people and improve things.”

Now, about the proposal itself and what it *feels* like: It’s a big change that came out of nowhere, especially at a time when spam doesn’t seem like a big issue. It’s a bit… shocking? Normally Bitcoin changes are discussed for ages, PRs take so long to get merged, because of all the care that devs put into considering the pros and the cons, but here? We get a mailing list post (that no user is reading, let’s be honest) and a PR with a lot of contributors saying “yeah, let’s do this!”. There should be a bit more research. (Maybe there was, but behind closed doors?). We want to know about all the protocols that are bloating the UTXO set and that could be using OP_RETURN instead. We want to know how many transactions like these are in each block, and how many there will be once the new protocols start getting deployed. We want to see that you did your homework for weeks before trying to change the policies. 

It feels weird—like some sort of attack or a bribe. One protocol is mentioned that would benefit from this change, and then an investor ACKs the PR. Someone points out a potential conflict of interest, and their comment gets hidden. That’s shady, no denying it.

And then, it feels that users have no say. It's a ""technocracy"", where devs have all the power in their hands and can decide where the protocol goes, while completely dismissing users' concerns.

Users do have concerns—lots of them. And I encourage devs to step out and talk to "regular" Bitcoin users. It’s not just a “loud minority”—there are many people genuinely worried: “Can we trust Core devs anymore? Is there even another option?”. But their concerns are dismissed, their comments deleted or hidden, they're treated like "bullies" that devs should ignore, devs should "merge Todd's PR and call it a day".

Without understanding all the technical details, I'd say there's a pretty good chance that the very smart people that work on Core are right once again. Statistically, it's very likely. Communicate to users, explain your motivations, and show that you want to listen to our concerns.

Lastly, as irritating as that might be, users freaking out and holding core developers accountable and flooding Github is a feature, not a bug. Devs aren't the Bitcoin gods, they are humans, and humans can be bribed,  can be coerced, can have conflict of interest, can fck up.

Users care and want to protect Bitcoin. Be grateful. 

Respectfully,
A Bitcoin Dev

-------------------------

1440000bytes | 2025-05-02 11:17:44 UTC | #2

IRC meeting logs: https://bitcoin-irc.chaincode.com/bitcoin-core-dev/2025-05-01

I shared the logs with different LLMs as an experiment. I asked whether they could identify humble and arrogant people in the chat. The results were interesting including some false positives.

-------------------------

vostrnad | 2025-05-02 15:29:21 UTC | #3

Hard disagree with most of your points.

> I think the whole OP_RETURN debate is missing one important part: a communication link between the developers and the users.

I don't see a lack of communication avenues between users and developers. We have the mailing list, GitHub issues and even the pull request comments for constructive criticisms of proposed changes (not brigading and repeating already addressed arguments).

> However, it is important that there is mutual respect and trust between users and devs. Right now, trust is pretty broken.

No, it is not. Bitcoin Core developers and maintainers are doing an excellent job, as evidenced e.g. by around 95% of the network choosing Bitcoin Core as their node implementation, despite multiple alternatives being available.

> It’s a big change that came out of nowhere

It's not a big change, it just allows people to send transactions through the network that they are already submitting directly to miners. It also didn't come out of nowhere, lifting the OP_RETURN limits has been talked about for ages and especially since people started putting arbitrary data into witnesses instead (which was over 2 years ago).

> One protocol is mentioned that would benefit from this change

The protocol mentioned would not really benefit from this change, rather everyone else would benefit if this protocol could switch from unspendable UTXOs to OP_RETURN outputs.

> and then an investor ACKs the PR. Someone points out a potential conflict of interest, and their comment gets hidden. That’s shady, no denying it.

Conflict of interest doesn't really matter here. If a change is proposed that would hurt Bitcoin at someone's benefit, it can just as well be rejected on the basis of hurting Bitcoin alone. Changes can also be proposed pseudonymously, so there's really no hope of policing conflicts of interest in the first place. Lastly, pull request comments are meant for reviewing the proposed changes, which is why the comment was hidden as off-topic.

> It’s a ““technocracy””, where devs have all the power in their hands and can decide where the protocol goes, while completely dismissing users’ concerns.

No, they cannot. Devs only write the code, it is up to the users to choose to run it. Devs can decide to make any changes they want, but without users running their code they have no power.

> But their concerns are dismissed, their comments deleted or hidden, they’re treated like “bullies” that devs should ignore, devs should “merge Todd’s PR and call it a day”.

As far as I can tell, all the arguments (that is, the few that actually had merit) against lifting OP_RETURN limits have been adequately addressed, in which case it's indeed fine to merge the PR.

> users freaking out and holding core developers accountable and flooding Github is a feature, not a bug

There is no need to hold developers accountable. You either run the software they provide to you (for free, at that), or you don't. Brigading pull request comments only wastes everyone's time and definitely contributes to the divide you seem to be so keen on resolving.

-------------------------

1440000bytes | 2025-05-03 13:09:53 UTC | #4

Is Bitcoin core maintained for users or regular contributors (most of them never use bitcoin)?

-------------------------

gmaxwell | 2025-05-03 02:47:55 UTC | #5

Always ask yourself,  "What do I (think) I know and why do I know it?"

I've been reading the public discussions-- it's how I'm aware of this issue at all.  I'm seeing an awful lot of wookie defense.    Bitcoin is important. Spam is bad. Apple pie is delicious, and that's why this change is destroying bitcoin!   

If you feel like you're not seeing a response to an argument it can be because it's a non-sequitur and there just isn't anything to respond to.

But don't let me create yet another 'dismissive' response.  You say here that this is a big change,  why do you think it is?   I think it is a very small change which should be of no interest to users generally.

>  PRs take so long to get merged, 

Most don't.  

>  (Maybe there was, but behind closed doors?)

I doubt it. I saw it, and recognized that it was obviously the right thing to do, and if anything it is significantly overdue.  I would have acked it but I'm not currently involved in development.  It's worth mentioning that I happily defended putting in that limit in the first place.

There is perhaps a bit of an inverse chestertons fence here-- you know the idea that you shouldn't remove a fence without understanding why it was there?  It's probably a bad idea to assume bad faith when the people who put in a fence go to take it down. :slight_smile: 

I had no discussion with anyone about it before seeing the public discussion except for over a year ago I became aware of poor 0.5RTT block reconstruction rates (partially caused by the lack of this change) and nagged a few people along the lines of WTF is no one doing anything about this???? (and then proceeded to be disappointed while nothing was done).

[quote="pandacute, post:1, topic:1650"]
Users do have concerns—lots of them.
[/quote]

Please ask about them then!

-------------------------

1440000bytes | 2025-05-03 17:07:38 UTC | #6

[quote="gmaxwell, post:5, topic:1650"]
Always ask yourself, “What do I (think) I know and why do I know it?”
[/quote]

Thank you sir. We never had this thought.

-------------------------

