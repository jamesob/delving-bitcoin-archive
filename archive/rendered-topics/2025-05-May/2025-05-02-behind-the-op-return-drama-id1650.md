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

FernandoTheKoala | 2025-05-20 09:14:12 UTC | #7

Hello, just a node runner everyday dude who is a btc educator in his local community. Maybe I can provide some outiside perspective from normie_user_land here: 

[quote="vostrnad, post:3, topic:1650"]
I don’t see a lack of communication avenues between users and developers. We have the mailing list, GitHub issues and even the pull request comments for constructive criticisms of proposed changes (not brigading and repeating already addressed arguments).
[/quote]

I agree that there is no lack of communication avenues between users and devs, but the issues might be: 

1) the narrow scope in the way the communication is carried out: devs seems to wanna talk almost exclusively about purely technical aspects, while users want to incorporate other aspects in the discussion. Don't get me wrong, as already stated users' feeling should not dictate the direction of bitcoin, and the technical is where it's at, but I think it is important to address users' concers in a way that makes them feel heard and is engaging to them (and technical is not engaging neither sufficient for them). 
Now btc has spread (and will spread more and more) to an audience that is little technical, and finding an effective way to comunicate with this audience will become more and more necessary to avoid situation like this whole op_return. With btc growth, the overwhelming majority of node operators will be non devs, and it might be beneficial to take this into account for communication approaches.  

2) While it is true that there are plenty of communication avenues, the info might be too scattered. Let's take this op_return case at hand. As a normie I was very confused by what was going on here, and initially I was persuaded by the "knots ppl". To REALLY understand core reasons and come around it took me more than 20 hours of reading and listening from many sources (stacker news, github, mailing list, pull request comments, btc++ debate on youtube, podcast of Shinobi, convos on nostr). I don't think a normal node runner would invest this kind of time to truly attempting to understand a debate, the goal should be to optimize for time and clarity. Here I have a suggestion in the final section "final thoughts"   

[quote="pandacute, post:1, topic:1650"]
Right now, trust is pretty broken. It can be regained, but not without understanding what went wrong in the first place.
[/quote]

vostrnad you say that trust has not been broken because 95% of network is choosing to run core. Your reply is from 2nd May, today 20th May Kore is at 8,5% of the network (and from what I read after this whole thing many ppl have ordered start9 and decided to run a node with knots, so the numbers should increase). While still irrelevant "in the big picture" I think this is undeniably a signal of loss of trust in core, when this % of ppl suddenly switch away from core it means something that deserves attention is going on. And instead of ignoring it, it can be taken as an oppurtinity to understand if maybe something has been done wrong and how to avoid it happening again in the future  

[quote="vostrnad, post:3, topic:1650"]
Conflict of interest doesn’t really matter here. If a change is proposed that would hurt Bitcoin at someone’s benefit, it can just as well be rejected on the basis of hurting Bitcoin alone. Changes can also be proposed pseudonymously, so there’s really no hope of policing conflicts of interest in the first place. Lastly, pull request comments are meant for reviewing the proposed changes, which is why the comment was hidden as off-topic.
[/quote]

Maybe I see what I wanna see, but here I see exactly what I was talking about in my first point: looking at it only from a technical perspective when you say "...on the basis of hurting bitcoin alone". Humans (and especially non devs) evaluate things not only from technical standpoint, and not taking this into consideration I fear will create more and more disconnetion between devs who sees things only through technical lens, and other non-devs who as "normal humans" look at things with a broader human lens (I mean, how can someone look at it and disregard the conflict of interest, and not taking into consideration from who it came from and the broader context in which it fits?). Please don't get me wrong, I'm not suggesting to shift the primary focus from technical to broader, what I am suggesting is that non-technical things cannot be simply not ackowledged and put aside because they are non-technical and so irrelevant (I am afraid this approach will have the consequence of creating more distrust and unneccesary debates from the general node operators)   

[quote="vostrnad, post:3, topic:1650"]
No, they cannot. Devs only write the code, it is up to the users to choose to run it. Devs can decide to make any changes they want, but without users running their code they have no power.
[/quote]

I think @pandacute is correct when he says it's a "technocracy" in the sense that devs are the only one that can write code, users cannot write code, they can just react/adopt to it and express their questions/concers. I also don't think it is a hepful statement to say "if you don't like anymore what core is doing with the code just run another software". Someone who ran core for years and so has given his trust to it, should be able to have his concerns listened to if there's a change he is unsure of (rather than just silently leaving and run another software becuase he is not able to engage in a form of discussion which he can sustain).  
Is this the kind of non-engagement we want? no discussion and just go somewhere else if you don't like it anymore withouth being able to have a discussion all together?

[quote="gmaxwell, post:5, topic:1650"]
But don’t let me create yet another ‘dismissive’ response. You say here that this is a big change, why do you think it is? I think it is a very small change which should be of no interest to users generally.
[/quote]

 I believe it has been perceived as a big change from part of the community not because of the technical change itself, but because of the "signals" that core has sent with this PR. Below I summarize what has been perceived from outside in a "very raw way":
- btc is not a money database, it's a database for anything, let the market decides for itself
- there is no definition of spam, they are all valid transactions 
- if you are not a dev you cannot understand, trust the experts 
- as a node you are not free to decide what you can relay
- let's focus on miners, not nodes 

I am not saying that this is what core is saying (and after more digging I got that it is not) but I can assure you that many people have perceived it this way from the outside. I don't think the messages have been communicated in the most clear and effective way, and thus they have been misunderstood. As you have seen, the most vocal opposition is comprised of "btc monetary maxis", they look at btc as freedom money and they are here for the revolution and they are very passionate (and I include myself here). I believe most have reacted the way they did because btc primarily as freedom money has been "not protected and seems to be not a given anymore". After what happened I think you core have realized that are 2 things that really heated things up 1) taking away nodes capacity to set limit (perceived as taking freedom away); and 2) defining btc as database for whatever without underlining that the monetary purpose is still the priority and most fundamental aspect of btc. 

[quote="gmaxwell, post:5, topic:1650"]
Please ask about them then!
[/quote]

Do you guys have an avenue to gather users concerns? or you read and participate in the public discussions when an issue arises? 


Other final thoughts:

You devs are amazing and the "unkind" statements that some users have directed at you are not  cool at all, but please look at them as an emotional reaction due to excessive passion for bitcoin as freedom money. 
You guys should focus on the code because that is what you are amazing at, you should not have to spend so much time gathering users concerns and reading through posts to formulate a reply (this is all time taken away from working on code). I believe @AntoineP has done an amazing job at this, but maybe we could have achieved the same without you spending so much time on it. 

In conclusion: 
1) I am afraid more and more of these kind of debate will arise in the future, as future node operators will be less technical and more "generalist", and might be necessary to consider a communication method that incorporate also non-technical aspects in a heavier manner to avoid future waste of time on both parts. 
2) Maybe core could have 2/3 ppl who are not devs but have a solid enough technical knowledge, which are mostly "expert communicators": the role of these people would be to gather users concerns on various platforms, provide a single avenue to which direct users concers, elaborate them and summarise them to present to devs in a orderly fashion, being a bridge between "mostly technical devs" and "mostly moralists average people" and perhaps support explaining the technical aspects incorporating other aspects. Maybe in this way we can avoid wasting time on both sides and also have a more engaging discussion on both sides? I'd be happy to cover this role for free, and like me I'm sure many would be happy to contribute to the cause.  

Like many I am a simple man who just cares about btc as money and wants to help, I hope this external feedback can be useful to you guys.  

Respectfully, FernandoTheKoala

-------------------------

gmaxwell | 2025-05-20 10:57:10 UTC | #8

> I am not saying that this is what core is saying (and after more digging I got that it is not)

Indeed, it's not.  And I think moreover no one who has been carefully following or talking to the involved people could claim that it is.  And so basically these are just lies that has been deployed to manipulate the public and sell them on a narrative.  So I'm not really sure what any of the Bitcoin Developers are to do about it?

Like I said to someone on reddit who complained that core had an "image problem" that, sure if someone goes around and says you beat your spouse you now have an "image problem", but it's often not one you can do much about.  Putting up a sign that says "Certainly not a spouse beater" doesn't help. And going on the attack against the dishonest troll that originated the lies hardly helps when the other half of their narrative is that they're just a poor victim being repressed. 

Nowhere in the discussion will you find a bitcoin core developer saying "your opinion is worthless because you're not a developer" -- they will respond to specific technical inaccuracies with technical corrections, sure.  But by contrast you constantly see broad dismissal of developers as some kind of mere technicians (or suggesting that some people have PHDs that they can't understand the real world) as if they can't be expert Bitcoin maximalists and also competent with software.  Sure there are myopic people that only do one thing, but generally competence is correlated and people that are excellent in one area are often pretty good in others too.  I fully expect anyone commenting on this issue will be able to have useful thoughts on any part of it.

In my 15-ish year experience it seems to be a truism in Bitcoin is that the people who are most loudly yelling about their values are the people who are using them for marketing, rather than the people who believe in them most strongly.  I think a lot of the long time developers would put many of those supposed "monetary maxis" to shame by any number of metrics-- including the fact that they could have chosen to gain unfathomable additional wealth by producing scamcoins *and didn't*.  Let me know when you find a monetary maxi influencer that has had a hit put out on them because of defending their views on Bitcoin or spend years of their life defending a billion dollar lawsuit trying to undermine Bitcoin's monetary policy.  

In my view when actually believe in something you just don't need to crow about it *all* the time.

Following the person who yells the loudest is how people fall for conmen, 'cause *anyone* can yell and it's a lot easier to say what people want to hear when you don't care if you mean what you say or don't care if its true.

(I'm not accusing any particular people in this of being conmen, yet, but I have to say the plainly false claims that any of the core folks support that "merely a database" view is a bad sign)

If you found this view unhelpful -- sorry! But with no examples of the dismissive response means I don't know how to be anything but meta-dismissive. :frowning:  To the extent that problems have been created because  some have inaccurately portrayed Bitcoin Core the parties responsible are the ones with the inaccuracies.

You are probably more able to correct that wrong than anyone perceived as associated with the project because at least you wouldn't justify more of the oppression narrative.  All of Bitcoin is the responsibility of everyone who cares about Bitcoin.

> Do you guys have an avenue to gather users concerns?

I'm not a bitcoin developer (not for many years),  but many of the developers involved in the discussion-- including the authors of both pull requests, the person who initiated the discussion, multiple project maintainers-- have been in the public discussion on here, reddit, stacker, bitcointalk, X... I've seen them get little to no response to their messages, and I can say that I've received practically none with the net effect that the considerable time I've spend feels like a substantial waste of time.

[Aside, thank you for ending my seventeen day wait here, I only wish you were also the original poster! :P ]

-------------------------

FernandoTheKoala | 2025-05-24 13:52:55 UTC | #9

[quote="gmaxwell, post:8, topic:1650"]
Indeed, it’s not. And I think moreover no one who has been carefully following or talking to the involved people could claim that it is. And so basically these are just lies that has been deployed to manipulate the public and sell them on a narrative. So I’m not really sure what any of the Bitcoin Developers are to do about it?
[/quote]

I agree with you, I'm not suggesting to go on the attack of "dishonest troll", the focus should not be on this. Rather I'm suggesting core to  focus on itself and the way it communicates with the public. 
I believe the message absorbed by some audience  is different than the message core intended to communicate (which resulted in 10% of network moving away from core) and we should ask ourselves how this happened. Maybe some public is too emotional/technical illiterate and is blindly persuaded by influencers? (sure, there's always some of this). But I fear there is also a disconnection in the communication between devs-public besides the fact that the information is too scatterd to be directly and easily accessed by the average node-runner. Also, it almost feels like devs operate/think on a different level compared to the public, and it's hard to find a meeting point where both parties can really understand each other. 
Again, maybe this is just an empty worry of mine and it will not be an issue again, but I suspect more of these "disconnected" conversation devs-public might happen in the future unless something changes

[quote="gmaxwell, post:8, topic:1650"]
(I’m not accusing any particular people in this of being conmen, yet, but I have to say the plainly false claims that any of the core folks support that “merely a database” view is a bad sign)
[/quote]

Myself (as a simple node runner) I did not get the idea that core was supporting btc as "merely a database", but I also did not get the idea that core was prioritizing btc as "money database" (and this not because of influencers, but by watching btc++ debate myself and reading mailing-list and delvingbticoin).  
I personally would have liked to see more discussion on pro/cons of change at consensus level to stop "spams", and what are the frictions that have/are preventing this consensus change (since both parties don't like spam on btc, why haven't we revisited again this topic where both parties are aligned? it has been superficially mentioned many times but never really explored)  

Also, the message I absorbed from core is (and please any core dev correct me if I got it wrong): "look, filters don't work and spam will get in the block anyway so keeping filters limit as they are is pretty useless (objectively true). If there's demand and ppl pay for it miners will accept these tx for the fees because they chase profit (also objectively true). We already have a mining centralization issue, and leaving things as they are might creates even more mining centralization (objectively true). So let's lift op_ret limit hoping they will use this avenue and stop using other avenues that are harmful to the network". I think everything is not-up-for-debate until the last step. I'm not saying the last step is wrong, I'm just saying that it is not unquestionable like the other steps and perhaps deserved a more in depth explanation, in my head something like "given the situation either we change things at consensus level to get rid of spams altogether (but we already discussed about this and could not find consensus, let's reopen the discussion if you want), then the second best thing is to lift op_return and see if we get ppl to use op_retur  and avoid greater mining centralization. Also, the majority is of the opinion that the best defense is not to "censor" this kind of tx but rather to let the market sort it out and hopefully contain them through fees in a way that it does not heavily prevent/impact financial tx. IF, and only IF, this change does result in the market pushing out finacial tx and making btc as money not usable, then rest assured we will intervene". 

Regarding the last part maybe this is me as monetary-maxis who wants to be constantly reassured that core 1st priority is btc as money and all other use-case come later, and maybe this is me who wants to define btc as primarily money while we should all collectively decide what bitcoin is by letting the market sort itself out (honestly I'm still thinking about this), but I believe that explicitely and directly knowing how core stands in regard to this might not have created the confusion and antagonisation that we witnessed. I think many "knots" ppl simply wanted to know if  we are still aligned that money use is still 1st priority, and I think this was not unquestionably clear in the many discussions

I'm aware that some of my impressions above are wrong, but nonetheless I shared them in the hope of better understanding where the disconnection of communication might have happened 

Respectfully, a well-meaning pleb

-------------------------

