# Changes to BIP-360 - Pay to Quantum Resistant Hash (P2QRH)

EthanHeilman | 2025-07-07 16:01:15 UTC | #1

We made the following changes to [BIP-360 (Pay to Quantum Resistant Hash) PR](https://github.com/bitcoin/bips/pull/1670):

* P2QRH (Pay to Quantum Resistant Hash) is now taproot (P2TR) but with the quantum vulnerable key-spend path removed.
* PQ signatures have been moved to a future BIP (coming soon).
* The plan for PQ signatures is to redefine OP_SUCCESSx opcodes: OP_CHECKMLSIG

Below we go into these changes one by one, see BIP-360 PR for full details ([BIP-360 mediawiki render as of 7/7/2025](https://github.com/bitcoin/bips/blob/77dbeee502b606fda2006ae53f4014cef612aacd/README.mediawiki)).

P2QRH is now script-spend only P2TR (taproot), i.e. no quantum vulnerable key-spend. P2QRH outputs commit directly to the tapleaf merkle root computed by taproot.

The scriptPubKey for a P2QRH output is:

OP_PUSHNUM_3 OP_PUSHBYTES_32 <tapleaf merkle root>

Advantages of this approach

1. We can reuse taproot code, but just skip taptweak steps.
2. Everyone who understands P2TR, already understands P2QRH.
3. By supporting tapscript and tapleaf, it supports everything that supports tapscript.
4. P2QRH protects tapscript outputs against long-exposure attacks. This is a big win because long-exposure attacks will be practical before short-exposure attacks. Note: protecting against short-exposure attacks requires PQ signatures.
5. P2QRH gives us similar functionality as the much discussed option of disabling key-spends in P2TR on Q-Day (when quantum attacks become practical), but with the added benefit that the ecosystem can upgrade well before Q-Day. This removes the risks of attempting a consensus change during an emergency or acting too late.

We moved PQ signatures specification out of BIP-360 so that P2QRH can be debated independently of the debate over PQ signature algorithms. This allows us to move forward on P2QRH without forcing a commitment to any particular algorithm.

BIP-360 includes a purely informational plan for adding PQ signature algorithms to tapscript. This plan to add tapscript PQ signature verification opcodes for ML-DSA (CRYSTALS-Dilithium) and SLH-DSA (SPHINCS+) via OP_SUCCESSx. This allows separate activation of PQ signature algorithms if desired and provides a pattern for adding new signature algorithms in the future. No new tapleaf version needed. The full specification will be given in a new BIP.

-------------------------

stevenroose | 2025-07-14 09:31:50 UTC | #2

If you're going the route of adding post-quantum crypto as just opcodes, then I would strongly suggest renaming the p2qrh soft-fork simply pay-to-tapscript or pay-to-mast. It's not quantum-specific and there is definitely demand from other parts of the ecosystem for something like p2ts.

-------------------------

EthanHeilman | 2025-07-14 15:09:06 UTC | #4

> there is definitely demand from other parts of the ecosystem for something like p2ts.

I agree with that BIP-360 (P2QRH) independent of its value to quantum resistance is a useful output type to have regardless.

> If you’re going the route of adding post-quantum crypto as just opcodes, then I would strongly suggest renaming the p2qrh soft-fork simply pay-to-tapscript or pay-to-mast.

Under other circumstances I would be in favor of naming this pay-to-tapscript (P2TS) or pay-to-tapleaf (P2TL), but under the threat of a post-quantum world there is a strong rationale for naming this P2QRH. In a post-quantum world, we do not want user's getting confused with P2TS (quantum safe output) and P2TR (quantum vulnerable output). P2QRH very clearly communicates that it is a quantum resistance output and distinguishes itself from P2TR.

This is also why we use Segwit version 3 rather than version 2. It makes the address contain an r,  `bc1(r)...`, so "r for resistant". This functions as a mnemonic and makes it easier for users to remember. Allowing users to, at a glance, determine if an output is Quantum-Resistant or not based it being bc1r... is helpful for avoiding accidents and letting users assess which if any of their addresses are vulnerable to long-exposure attacks. We need to help users not spend funds to to a bc1p... address if bc1p... addresses are vulnerable.

-------------------------

stevenroose | 2025-07-14 18:26:10 UTC | #5

Allow me to give some reasons of why I disagree here. Note that it's just naming though, arguably not the most important part of a technical proposal..

- The quantum threat might never materialize and a p2tr-like output type can still be useful without it. If this proposal is advertised as a solution for the quantum threat, it might not get any attention until that threat exists, which might be never.
- The name "pay-to-quantum-resistant-hash" is confusing as it breaks with the convention of output types, that describe what you're paying into. "pubkey-hash" being the hash of a pubkey, that can sign to spend, "script-hash" being the hash of a script, that should be fulfilled to spend. No "quantum resistant [thing]" is hashed here, so it sounds like you're just paying to a hash, as if spending would be done by simply providing the preimage.
- "quantum resistant hash" also sounds like you're using some better more quantum-resistant hash function, but from reading the BIP it seems like you're proposing to re-use SHA256 which in my limited QC understanding is quantum-safe? Can be seen as misleading.
- Addresses are to be read by the sender and the sender doesn't really care if the receiver's wallet is quantum-resistant. Using letters in the address to convey information also sets a bad UX precedent as they're not intended to be used that way.
- In the scenario where a quantum threat is imminent or already passed and we are in a post-quantum world, I hope we don't need address letters or output naming for developers to know what outputs they should be using.
- Also, in such a world, any additional new feature we add to bitcoin will hopefully also be quantum resistant and I hope we won't be naming every new output type as "pay to quantum resistant something". Even other new features might be added before. Will we add OP_QUANTUMRESISTANTTXHASH, OP_CHECKQUANTUMRESISTANTCONTRACTVERIFY, etc?

TL;DR: I think pay-to-tapscript is useful outside of the QC context and association with it might delay interest until QC seems more relevant.

-------------------------

EthanHeilman | 2025-07-14 19:35:52 UTC | #6

I like your objections, thanks for posting them. Let me respond to them.

>  [1] If this proposal is advertised as a solution for the quantum threat, it might not get any attention until that threat exists, which might be never.

I would not be working on it if there was not a pressing threat of quantum computers. Is it useful for other things, sure, but that is not the motivation behind this proposal.

> [2]  The name “pay-to-quantum-resistant-hash” is confusing as it breaks with the convention of output types, that describe what you’re paying into.

I don't disagree here. Technically following the naming convention it would be something like P2QRTLRH (pay-to-quantum-resistant-tapleaf-root-hash). That seems too long.

If everyone wants to activate P2QRH but they want a different name, I would not object. I do worry about the bikeshedding potential of arguing over the exact name before the PR is even merged into the BIPs repo.

> [3]  reading the BIP it seems like you’re proposing to re-use SHA256 which in my limited QC understanding is quantum-safe? Can be seen as misleading

SHA256 is quantum-resistant. If SHA256 pre-image security is broken, Bitcoin is in big trouble.

> [4] Addresses are to be read by the sender and the sender doesn’t really care if the receiver’s wallet is quantum-resistant. Using letters in the address to convey information also sets a bad UX precedent as they’re not intended to be used that way.

Senders often do care if the receiver has their funds stolen prior to receiving it. Users are also likely to examine their addresses to see if their funds are vulnerable long-exposure attacks, this makes it easier for them to do so.

Being able to determine the output type via the address is useful and we already do this with bc1... to tell someone it is a segwit (mainnet) address. I suspect this is a matter of personal taste and I'd be interested in seeing a larger discussion on this question to gauge preferences.

> [5] In the scenario where a quantum threat is imminent or already passed and we are in a post-quantum world, I hope we don’t need address letters or output naming for developers to know what outputs they should be using.

I hope the same thing, but wallets are sometimes slow to upgrade and people may be using steel wallets with the address written down. All things being equal, we should choose the option that reduces the chance of user error even if we aren't sure how big of a chance that is.

> [6] Also, in such a world, any additional new feature we add to bitcoin will hopefully also be quantum resistant and I hope we won’t be naming every new output type as “pay to quantum resistant something”. Even other new features might be added before. Will we add OP_QUANTUMRESISTANTTXHASH, OP_CHECKQUANTUMRESISTANTCONTRACTVERIFY, etc?

If we add a new quantum resistant output type after everyone has upgraded their outputs, then it doesn't really matter anymore. Everyone will have upgraded. We are past the point of danger. 

Any opcodes added via OP_SUCCESSx or tapleaf version would be compatible with P2QRH. I could be wrong, but I am not sure we will need a new output type for sometime.

What is the benefit of putting in the word QUANTUMRESISTANT in front of new opcodes as you are proposing? 

> TL;DR: I think pay-to-tapscript is useful outside of the QC context and association with it might delay interest until QC seems more relevant.

I'm curious what you want such an output for? If you don't care about quantum attacks, use P2TR with a NUMS point. Sure P2QRH would save you 32 bytes in transaction size. What is your use case here?

-------------------------

cryptoquick | 2025-07-14 20:30:49 UTC | #7

Honestly I'd be open to renaming it to Pay to Tap Script Hash, or P2TSH. It rhymes better with P2WSH that way.

I'll admit I've done worse things just to get an ACK on my BIP :joy_cat: 

Should I go ahead and make the PR?

-------------------------

cryptoquick | 2025-07-14 20:55:06 UTC | #8

mkay, I had some time to put together the change. here's what it looks like:
https://github.com/cryptoquick/bips/pull/26
@stevenroose If you find this satisfactory and we merge this, I just ask that you ACK the bitcoin/bips thread considering your name would now be on the BIP :joy_cat:

-------------------------

stevenroose | 2025-07-15 00:03:39 UTC | #9

[quote="EthanHeilman, post:6, topic:1811"]
I’m curious what you want such an output for? If you don’t care about quantum attacks, use P2TR with a NUMS point. Sure P2QRH would save you 32 bytes in transaction size. What is your use case here?
[/quote]

Exactly for this. Many second-layer protocols using covenants would not want the key-spend option and hence will be wasting 32 bytes in witness space. Also, primitive CAT-based covenant smart contract would be more easily composed without the need for EC tweaking inside Script. F.e. proving an output is the mast root of 2 scripts is easy CAT+SHA, while proving the same in taproot means needing to do a tweak which currently we can't.

-------------------------

stevenroose | 2025-07-15 00:05:51 UTC | #10

[quote="cryptoquick, post:8, topic:1811"]
@stevenroose If you find this satisfactory and we merge this, I just ask that you ACK the bitcoin/bips thread considering your name would now be on the BIP
[/quote]

I prefer that name, yes. No need to add my name, it would mean I'd have to do a more thorough review of the BIP and I don't currently have the time for that. I might take a closer look in the next few months. I do believe a p2ts primitive is useful.

-------------------------

sipa | 2025-07-15 01:13:12 UTC | #11

That sounds like it's trying to undo the privacy advantage that Taproot was intended to provide: incentivizing having a cooperative everyone-agrees key path option, which is cheaper to use than anything else.

If you feel that it's unreasonable to effectively impose a penalty on using the script path option, which is not there due to any resource being consumed, but simply due to not providing the possibility of having just a script path, consider the following.

Imagine a proposal to add account balances to the UTXO set, which can be incremented and decremented, Ethereum-style. It would simplify some things, complicate other things, but due to the removal of a need for change outputs, it would also make things cheaper for users in general. However, it would do so at the cost of incentivizing non-private behavior, equivalent to address reuse today. You could argue that the option of fresh balances per transaction still exists, at a cost, for those who care about it. But privacy is a common good - it doesn't work if only those who need it get it. Thus, I think such a proposal would not get much traction, because the protocol shouldn't incentivize non-private behavior.

I feel this is similar. Yes, it's technically possible to provide a cheaper option. But it's better for everyone if it's not available for anyone. I understand that (in the context of this thread) there may be other reasons to consider a non-tweaked Merkle root of scripts outputs. But arguing for this because of a desire to avoid a cost only born by protocols that can't be made to use a cooperative key path spends, seems like an anti-feature to me.

-------------------------

dr-orlovsky | 2025-07-15 11:19:21 UTC | #12

Few non-important comments

[quote="EthanHeilman, post:1, topic:1811"]
P2QRH protects tapscript outputs against long-exposure attacks.
[/quote]

Unless wallet descriptor is exposed

[quote="EthanHeilman, post:4, topic:1811"]
This is also why we use Segwit version 3 rather than version 2
[/quote]

From my understanding the main reason of using version 3 is the fact that in order to re-use version 2 you need a different payload size, while your size is exactly 256 bits (hash), and it is better to use a newer version than extend it with some unneeded byte(s), consuming blockchain space forever.

-------------------------

EthanHeilman | 2025-07-15 14:48:49 UTC | #13

> From my understanding the main reason of using version 3 is the fact that in order to re-use version 2 you need a different payload size, while your size is exactly 256 bits

I haven't heard this before. I can only speak for myself, but that was not a consideration. Can you provide more details about this?

The motivation, as far as I am aware, was to take an opinionated stance that SegWit versions should not be used in order. Rather they should be used to signal the type of output in an address. So in theory P2QRH might use SegWit version 3 for "bc1(r)", then later another output might use SigWit version 2 to "bc1(z)" for say a (z)ero knowledge output). That is we are not using SegWit versions as incrementing version numbers, but are disregarding the numeric order to use them as flags.

It is worth having a conversation about if this is the right way to use SegWit versions. If the community is strongly against it, then I'd personally rather have P2QRH than Segwit version based output flagging. 

My argument in favor of output flagging is as follows: Having SegWit versions used out of order is mildly confusing for new bitcoin-core developers, but reduces user error. My philosophy is, outside of extreme examples, to always take the trade-off in favor of usability.

I'd classify arguments against into three categories:
- There is a technical reason SegWit versions must increment that Ethan missed.
- There is no net usability benefit or the complexity is too high for bitcoin-core.
- There isn't enough support in the community for idea of using SegWit versions as flags and it holds up agreement on BIP-360.

-------------------------

stevenroose | 2025-07-15 15:09:55 UTC | #14

@sipa How about a simple taproot v2 where you still commit to a 32-byte output, where you can do regular taproot spend (sign with output key or proof MAST tweak), or if you can simply show the MAST root as the preimage of the output.

This obviously wouldn't work in the context of eliminating EC for quantum-resistance, but I have often wondered why taproot didn't include this option. Is such an idea broken in some way that I'm not seeing? It relies on both no one being able to sign for an arbitrary 32-byte SHA256 output and no one knowing the SHA256 preimage of a public key.

The privacy aspect I think would be identical for cases where a keyspend is not present.

-------------------------

sipa | 2025-07-15 15:35:18 UTC | #15

There are two privacy downsides to it:
* Unless you require that even in the MAST root case the output must look like a valid point on the curve, this creates outputs that are distinguishable (because only ~half of 32-byte values are valid X coordinates).
* In either case, spending a MAST root output reveals that no key path ever existed in the output.

-------------------------

dr-orlovsky | 2025-07-16 16:45:26 UTC | #16

[quote="EthanHeilman, post:13, topic:1811"]
I haven’t heard this before. I can only speak for myself, but that was not a consideration. Can you provide more details about this?
[/quote]

Well, I meant "taproot version of segwit". These 0-based numerations are driving me crazy.

If you carefully read BIP-341 you will see that you can use taproot segwit version (whatever it is, 1 or 2, I do not remember) with a different length of payload which should be not 256 bits, and it can be a future taproot version.

But you have 256 bits, meaning you can't go that route.

-------------------------

murch | 2025-07-16 17:53:01 UTC | #17

[quote="dr-orlovsky, post:12, topic:1811"]
From my understanding the main reason of using version 3 is the fact that in order to re-use version 2 you need a different payload size, while your size is exactly 256 bits (hash), and it is better to use a newer version than extend it with some unneeded byte(s), consuming blockchain space forever.
[/quote]

I think there is a misunderstanding here. P2TR is version 1, and Ethan was discussing using either version 2 or version 3, having already decided not to use the same version as P2TR.

-------------------------

dr-orlovsky | 2025-07-17 18:29:51 UTC | #18

Ok, so what I have meant is the fact that segwit v1 can also be used for post-taproot, if the payload is not 256 bit

-------------------------

conduition | 2025-07-18 23:11:43 UTC | #19

Hey @EthanHeilman, this is looking great, I like the new approach. I don't think the name really matters, but toss my hat in the "Pay to TapScript Hash" (P2TSH) bucket. I don't think you should invoke the term "Quantum resistant" in this BIP, since it no longer specifies any PQ signature algorithm opcodes to use inside the MAST.

The rest of this comment is possibly tangential and more in the realm of exploratory ideas but here goes. 

Are you familiar with how SPHINCS (SLH-DSA) works internally? [I wrote a more detailed summary here](https://conduition.io/cryptography/quantum-hbs/#SPINCS), but basically the authors of SPHINCS wanted to decouple the SPHINCS public key - which is just a merkle root hash - from the vast number of child signing keys (the leafs of the merkle tree) . They did this by using intermediate layers of "certification keys", which themselves sign other trees of certification keys, till eventually we reach the actual signing keys. Kinda like how TLS certificate authorities work.

We can borrow this idea for BIP360 to build tapscript leaves which can be defined dynamically at spending time. This would let users start moving funds to quantum-safe addresses even though we don't have PQ opcodes defined yet.

Imagine a P2QRH (or whatever) address, where one of its leaves commits to a hash-based one-time signature public key (such as WOTS or Lamport) instead of directly committing to a script. 

This leaf can be used to spend the UTXO if the spender provides *a locking script signed by the OTS pubkey,* and a witness stack to unlock the "dynamically endorsed script".

Why is this better than just defining OP_WINTERNITZ or some other OTS opcode to use inside tapscript?

- It will enable gentle transition towards new PQ signature opcodes defined in a future BIP. Users can start moving money over to P2QRH and be assured of quantum safety, even though the full-strength PQ signature opcodes and their public key formats aren't defined yet.
- We maintain forward-looking soft-fork compatibility as long as any PQ opcode is just a rename of OP_SUCCESS. 
- If a quantum attacker learns our wallet descriptor, they cannot exploit EC pubkeys inside the endorsed script to steal money until the owner has published an OTS endorsement signature, even if the adversary knows and cracks the EC pubkey(s) inside the to-be-endorsed script. 
- Users with funds in a P2QRH address can safely HODL until PQ opcodes are released, and then endorse a script which uses PQ opcodes instead of EC opcodes. They incur no short-term exposure to the quantum attacker at all. When the time comes, switching to PQ signatures will then be as easy as upgrading your wallet software.
- If a classical attacker learns our wallet descriptor, they can't sweep our money with an OP_SUCCESS opcode because they can't sign with the OTS key.
- Reusing the leaf doesn't leak the OTS secret key provided you always endorse the same script pubkey on every spend.
- Enables a new behavior never before possible on bitcoin: Dynamically choosing the script pubkey of an address *after* it is created, granted you can only do it once per leaf due to OTS limitations. Maybe there are new use cases to be explored?

It'd mean fully defining an OTS scheme for script endorsement, so maybe that'd be best as a new script leaf version, in a new BIP. What do you think?

EDIT: I still think if possible we should try to get ML-DSA and SLH-DSA standardized and packaged with BIP360, even if they are separate BIPs. However if that's not practical this "dynamic script endorsement" thing might give us a transition plan.

-------------------------

sipa | 2025-07-19 19:24:15 UTC | #20

Hi @EthanHeilman 

Thank you for the write-up to discuss the changes you're making to the proposal.

I believe there may be an implicit understanding behind the proposal and how it is to be deployed in conjunction with other changes, because I do not understand the motivation for this proposal if it does not disable(**) DL-based opcodes along with removing the key path spend from the start.

The proposal talks about allowing users to move to outputs that are not vulnerable to long-range attacks. However, given that there are already millions of BTC stored in outputs with exposed public keys, a significant amount of which should be expected to remain there (and that is just counting known ones, not including ones revealed selectively through second-layer protocols, payment protocols, wallet servers, other infrastructure, ...), it seems to me that any realistic hope(*) of long-range protection must inevitably disable spending through DL opcodes anyway. It does not matter that individual holders can move their coins to quantum safety if a significant portion of the supply is expected to remain vulnerable; those holders (if they perceive the threat of CRQC to be real, because if not, why are they moving their coins ar all?) are much better off exchanging their coins for another asset that does not risk being flooded by millions of stolen coins.

Given that, the current proposal achieves nothing in my mind. No quantum protection is achieved until DL spending is disabled through a separate consensus change, and it can disable that for P2TR just as well as inside BIP360 scripts.

I *could* see an advantage to having a separate output type **if** it also disabled the existing checksig opcodes inside the scripts. That would enable things like the introduction of a protocol rule to only allow sending to definitely-quantum-safe output types at some stage in the future to encourage migration. I'm very hesitant about the idea of using outputs (which for privacy reasons really shouldn't reveal anything about the receiver's conditions), but ignoring that, it seems like a defensible position. However, given that BIP360 as discussed here doesn't even disable DL spending, it does not even permit this. With that, the only benefit this proposal seems to give is to those who do not wish to pay the key-path reveal cost/complication in protocols that cannot or don't want to use a cooperate internal key, which I've argued above is not a positive evolution in my view.

(*) To be clear, I do not intend to take a position here about the viability of a CRQC, or the timelines involved. Treat everything in this post as a consideration to be made if/when whatever milestone you consider relevant is met, whether you think that is today or something hypothetical that likely never happens.

(**) I am not commenting on whether a DL-disabling consensus change should be made. I am merely pointing out that without one, I do not see how this proposal achieves its self-described goal.

-------------------------

EthanHeilman | 2025-07-21 17:39:38 UTC | #21

[quote="sipa, post:20, topic:1811"]
I believe there may be an implicit understanding behind the proposal and how it is to be deployed in conjunction with other changes, because I do not understand the motivation for this proposal if it does not disable(**) DL-based opcodes along with removing the key path spend from the start.
[/quote]

The purpose of BIP-360 and the future PQ signature BIP is to specify how to softfork Bitcoin so that funds can be securely spent in a post-quantum world. It does not need to have opinion on what to do about quantum vulnerable outputs.

It makes sense to propose such solutions in separate BIPs. This lets us make forward progress in some areas without getting consensus on the freeze/don't freeze proposals.

[quote="sipa, post:20, topic:1811"]
I *could* see an advantage to having a separate output type **if** it also disabled the existing checksig opcodes inside the scripts. That would enable things like the introduction of a protocol rule to only allow sending to definitely-quantum-safe output types at some stage in the future to encourage migration.
[/quote]

Users are unlikely to move to spending with quantum resistant signatures due to the high fee cost until quantum computers are a serious threat. We want to avoid everyone having to move outputs all at once at a moment of crisis.

With P2QRH, you can spend in a quantum-secure manner, but also spend using Schnorr so you don't hit high fees. This means users, exchanges, etc... can switch to P2QRH without any serious adoption cost and gain the benefits of post-quantum security. If, at some future time, the community decides to freeze quantum vulnerable spends, all funds in P2QRH outputs with PQ signature leafs are safe and don't need to move.

If P2QRH disabled Schnorr, then no one would move their coin to it until absolutely necessary because fees would be 6 times higher. This means that wallet, exchange and lightning network support will probably not exist (why support an output type no one uses). A rapid project to add support and switch everyone over at the last minute will increase the risk of disaster.

[quote="sipa, post:20, topic:1811"]
With that, the only benefit this proposal seems to give is to those who do not wish to pay the key-path reveal cost/complication in protocols that cannot or don’t want to use a cooperate internal key, which I’ve argued above is not a positive evolution in my view.
[/quote]

In protocols with a cooperative key spend and uncooperative script spend the P2QRH Merkle tree looks like:

1. Cooperative (key spend): A tiny script that does a simple 1 pubkey, 1 signature. 
2. Uncooperative (Script spend): A bigger script that does the more complex script spend with at least 2 pubkeys, 2 signatures.

It is at least 32+64 bytes cheaper to do cooperative (so cooperative is still incentivized). Yes, you leak the fact that another spending condition may exist.

Is there so much demand for script-only tapscript outputs with no cooperative path that making uncooperative 32 bytes cheaper would actually be problem?

You could, if you wanted, have P2TR where the script path spend replaces OP_CHECKSIG with OP_CHECKSIG_PQ and then keep the key path spend as Schnorr. Then you could disable key path spends when a CRQC becomes relevant. The downside here is the uncooperative spends would be much larger which could reduce the security if used in the Lightning Network and then once key path spends are disabled we are wasting an additional 32 bytes. It is workable but P2QRH just seems slightly better (smaller, easier to adopt, less complex)

[quote="sipa, post:20, topic:1811"]
it can disable that for P2TR just as well as inside BIP360 scripts.
[/quote]

We could take the path where there is a belief that P2TR is safe to use because P2TR key path spends are promised to be disabled if a CRQC arises. Such a promise could be made credible by having a quantum bounty that automatically disables P2TR key spends. Once disabled P2TR just becomes P2QRH but 32-bytes larger.

I think such a strategy is workable but not as nice as P2QRH.  Softforking a system to disable key spend via a quantum bounty is more complex and risker than P2QRH. Without such a automatic disable switch the promise alone seems insufficient.

-------------------------

jesseposner | 2025-07-20 18:46:55 UTC | #22

[quote="sipa, post:20, topic:1811"]
It does not matter that individual holders can move their coins to quantum safety if a significant portion of the supply is expected to remain vulnerable; those holders (if they perceive the threat of CRQC to be real, because if not, why are they moving their coins ar all?) are much better off exchanging their coins for another asset that does not risk being flooded by millions of stolen coins.
[/quote]

This assumes the existence of another asset that provides equivalent or superior properties. For example, a fork of bitcoin that disables DL opcodes could be perceived as inferior due to the precedent it sets with respect to property rights. In addition, the effect of millions of stolen coins is speculative. Bitcoin has survived many extreme drawdowns and it can likely survive them again, and we don't know what buyers might step in if a flood of coins hits the market. We also don't know if CRQC attackers would choose to flood the market.

-------------------------

EthanHeilman | 2025-07-21 15:21:36 UTC | #23

This is a cool idea, though I rather make as few changes in P2QRH as possible.

[quote="conduition, post:19, topic:1811"]
If a quantum attacker learns our wallet descriptor, they cannot exploit EC pubkeys inside the endorsed script to steal money until the owner has published an OTS endorsement signature, even if the adversary knows and cracks the EC pubkey(s) inside the to-be-endorsed script.
[/quote]

If I understand you correctly this would prevent long exposure attacks on reused public keys, but not solve short exposure attacks.

You don't need to change P2QRH to get this. Simply require a preimage and a signature (H(x) == y AND VER(pk, sig, m) == 1) to spend and have a different preimage per leaf. This prevents long exposure attacks even if the public key is reused since the attacker doesn't know the preimage x. This seems like not reusing a public key but with extra steps.

In P2SH you can do slightly better using the techniques from [Signing a Bitcoin Transaction with Lamport Signatures (no changes needed)](https://groups.google.com/g/bitcoindev/c/mR53go5gHIk). Though as pointed out by [Adam Borcany](https://groups.google.com/g/bitcoindev/c/mR53go5gHIk/m/HdLvltBIAAAJ) this scheme functions more like a speed bump since due to the 201 opcode limit in pre-tapscript the key space is small enough to be vulnerable to bruteforce.

[quote="conduition, post:19, topic:1811"]
It’d mean fully defining an OTS scheme for script endorsement, so maybe that’d be best as a new script leaf version, in a new BIP. What do you think?
[/quote]

You should write this up more concretely. It is an interesting idea.

-------------------------

conduition | 2025-07-23 14:40:25 UTC | #24

> You don’t need to change P2QRH to get this. Simply require a preimage and a signature (H(x) == y AND VER(pk, sig, m) == 1) to spend and have a different preimage per leaf.

Fair point, more efficient that way, I hadn't considered that. The other benefits of dynamic script endorsement still stand. I'll try to write up a draft BIP when I have some free time

-------------------------

conduition | 2025-07-23 15:10:42 UTC | #25

Whether or not DL opcodes are disabled, users will need somewhere to move their funds to once they become aware and concerned about quantum attack - Whether that occurs in a year or a decade or longer, we need an option ready.

Disabling key-path spending on P2TR is just as technically viable an option as BIP360, but if our long-term plan is P2TR without key-paths, this comes with some UX and DX problems:

- Once QCs are around, how do wallets prevent users from sending funds to quantum-vulnerable addresses, if they look exactly the same as quantum-resistant addresses?
- Before QCs are around, how do users know if their P2TR wallet actually has quantum-resistant leafs attached? A legacy P2TR wallet would mostly look and act exactly the same as a P2TR wallet which has post-quantum leafs. Users would need assurance from the wallet maintainers or to read the source code themselves. 
- Naive users who have funds in P2TR may hear "P2TR is now quantum secure" and falsely believe their UTXOs are safe and that no action is needed.
- Bitcoin client authors must now maintain secp256k1 code indefinitely - tap-tweaking the internal key becomes a required step to spend bitcoin even in a post-quantum world.

As for disabling EC opcodes, I am personally in favor of eventually placing a temporary restriction on them, pending a ZK-STARK soft fork which re-unlocks them. [See this mailing list thread for info](https://groups.google.com/g/bitcoindev/c/uEaf4bj07rE). But certainly that kind of change should not be concurrent with BIP360's deployment, as it would prevent people from migrating to P2QRH, and as Ethan hinted, it'd also needlessly embroil a perfectly valid BIP in a contentious debate. 

Whatever your stance on freeze/not-freeze/zk-starks, we can all agree we need addresses people can migrate towards which are definitively quantum resistant.

-------------------------

cryptoquick | 2025-07-24 21:26:47 UTC | #26

"Given that, the current proposal achieves nothing in my mind. No quantum protection is achieved until DL spending is disabled through a separate consensus change, and it can disable that for P2TR just as well as inside BIP360 scripts."

That's not quite true though, right? P2TR is vulnerable to long exposure attack via the key path spend. P2QRH disables that, making TapScript safe to use for committing to PQ PKHs. This then gives wallet users the *option* to spend their coins with either ECC or PQC, depending on which script path they choose. P2QRH makes Taproot useful in a PQ context. Right now it simply isn't. And if CRQCs never manifest, it's good that we still have the ECC path.

The alternative is to disable key path spends in P2TR, which could confiscate funds. Personally I'd be very against breaking people's existing usage of bitcoin even if a quantum threat were active or imminent. There's not as many coins using P2TR yet anyway in comparison to coins held in P2PKs or reused addresses... 150K vs. 1.7M.

-------------------------

sipa | 2025-07-27 21:51:45 UTC | #27

[quote="cryptoquick, post:26, topic:1811"]
That’s not quite true though, right? P2TR is vulnerable to long exposure attack via the key path spend
[/quote]

Yes. And I don't think it's relevant that individual users can *choose* to migrate their coins if there is no expectation that everyone will (all assuming a CRQC actalually appears). And without widespread agreement that EC spending will be disabled (for P2TR, but much more for the many other types of outputs whose pubkeys are known), I cannot image that enough coin owners (many of whom are dormant) will.

Millions of BTC likely will remain in EC-protected outputs even if the option of migrating to something else is possible. In such a situation, nobody's (value of their) holdings are safe until such time that *all* EC spending is disabled.

-------------------------

