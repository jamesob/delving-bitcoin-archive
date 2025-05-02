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

