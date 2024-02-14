# Lightning Hardware Wallet

pgrange | 2024-02-13 10:50:39 UTC | #1

This is a request for comments on a new open-source project I’m starting. The goal is to provide a desktop and/or mobile lightning wallet which interacts with a hardware secured device to secure the keys used in the Lightning protocol. There is no plan in developing any hardware here but rely on an already available one like Ledger, for instance.

# Why

Lightning wallets are currently designed to hold a very small part of someone’s bitcoins and for small payments. The dramatic increase in bitcoin use and the induced traffic makes it more and more desirable for individuals (to reduce fees) and the whole community (to reduce layer 1 traffic) to handle larger amounts of bitcoin with lightning but it is my belief that we need hardware to decently secure the process for the end user.

# What

The project is in its experimental phase during which I want to validate technical feasibility and UX experience for combining a Lightning wallet with a hardware wallet. My plan is to base the source code of the Lightning wallet on the work done by ACINQ with the [phoenix](https://github.com/ACINQ/phoenix) project. For the hardware wallet, I’m considering relying on a Ledger device as it’s one of the most deployed one and I already worked with it in a similar setup in a past experience. I do not plan to develop any code embedded in the hardware ledger for this iteration of the project but only rely on capabilities already exposed by the device.

A the end of this experimental phase I plan to deliver a prototype application running on a desktop machine and/or mobile device that one can use to pay or receive payments on the lightning network. The signing of transactions will be delegated to the hardware wallet.

# How

My current plan looks something like that:
* Adapting [lightning-kmp](https://github.com/ACINQ/lightning-kmp), the core lightning component of the phoenix wallet, so that it can delegate signatures to a hardware wallet;
* Adapting the phoenix wallet to use this version of lightning-kmp, including some quick and dirty UX changes to deal with the impact of the hardware interaction on the payment workflow.

# Other Initiatives

The lightning protocol has some specifics that can make it hard to rely on « traditional » hardware key protections. Hence, to the best of my knowledge, there is no Hardware protected lightning wallet available today nor any ongoing open source project to do so.

[Validating Lightning Signer](https://gitlab.com/lightning-signer) is an open source project aimed at integrating hardware security with lightning node. The goal is to help lightning node operators to secure their funds by segregating the node key from the rest of the system. It can be compared to what a KMS/HSM could bring in terms of key protection for regular cloud services. The keys are stored in what they call the « Secure Area » and this secure area allows to embed some politics to ensure that what it signs is actually legit, reducing the risk on the rest of the system.

What I have in mind is different from or at least complements Validating Lightning Signer as it does not address the same use case. Validating Lightning Signer targets lightning nodes that relay payments in the network and need to operate automatically 24/7 without human interactions. I want to target end users that want to pay or receive payments but do not intend to relay for others. They do not want their system to make any payment without them being conscious about it and feeling in control. It implies very different implementation strategies and UX challenges.

Bitbox sale a hardware wallet and they have [announced](https://bitbox.swiss/blog/bringing-lightning-to-the-bitboxapp/) that they plan to cover lightning payments with it but they explicitly exclude the signing of the lightning transactions by their hardware and « trades security for convenience »: « But as your Lightning wallet is not meant for large amounts, this is a preferable trade-off ».

On the contrary, I’d like to focus on « large amounts ». This is a very personal notion that only means large enough for the owner to want to protect it with hardware. So the wallet will need the hardware to validate lightning transactions, which is very different from bitbox proposal and introduces specific challenges to not degrade convenience too much.

Comparable to bitbox, Electrum lightning implementation can open channel from and close channel by returning the funds to hardware protected wallets but, once in the lightning channel, we're back to a hot wallet with in-memory keys.

# Challenges

Integrating hardware security to a lightning wallet while ensuring convenience has, to the best of my knowledge, never been achieved before. In particular, when paying with lightning, the wallet searches for several candidate payment paths and tries each of them in sequence until the payment succeeds. With current wallets, this is totally transparent for the user but since each attempt implies cryptographic signatures, it can become quite cumbersome for the user if they have to validate several times on their hardware wallet for one single payment.

Receiving payment can be more straightforward but might have implications on the htlc delays as the user will have to validate the payment before receiving it, introducing a human interaction inside the payment workflow, hence increasing payment process duration on the network.

I also anticipate issues with multi-path payments as one payment can be split into multiple, low level, lightning payments, each involving a cryptographic signature.

The best possible integration between the protocol as it is and human and hardware interaction has yet to be explored and constitutes, I think, the main technical and UX challenge here.

## Request for comments

The target audience for this project are people owning bitcoin in self-custody and who care enough to secure them with a hardware wallet. I’m scratching my own itch here but I’ve noticed some interest around me with people complaining about the same pain point: I want to rely more on layer 2 but don't trust current solutions enough to do so. I would appreciate to read your feedback on this.

I’m also aware that paying « large amounts » on lightning today can also be complicated for other reasons that have nothing to do with hardware security, like liquidity issues on the network for instance. My idea with this project is to focus only on the security aspect and how to help users feel safe and in control. Although there are other problems to solve, I believe this one has to be solved anyway.

-------------------------

t-bast | 2024-02-13 12:49:23 UTC | #2

We've thought about this a few times, and even prototyped integration between Phoenix and Ledger for the on-chain operations (funding and splicing). But this is different from what you want to achieve because we still let the channel keys be hot, instead of letting the Ledger device manage them.

There is a lot of complexity with the approach you want to take. The first one is that the wallet won't be able to run in the background while the user isn't monitoring their phone. That's an important drawback, because most payments are received while the app is not open and is instead woken up on-the-fly by the LSP (and runs in the background).

The second one is that you will still need the hardware device to be stateful and implement non-trivial policies, similar to what VLS does, because once the user authorizes a payment, there are a lot of different signing operations that may happen for that payment to complete, and you want the hardware device to make sure that a malicious app isn't trying to exfiltrate funds through those updates.

There are also a lot of "background" operations happening all the time that require signatures (on-the-fly splicing, commitment fee updates, etc). You'll need to implement a lot of the lightning channel state machine logic *inside* the hardware device to properly analyze and authorize those without user input. You may end up re-writing a whole lightning implementation inside the hardware wallet, which is a tedious and complex task.

I'm not trying to scare you, it can still be worthwhile to spend time prototyping that approach, but you shouldn't underestimate how big of a rabbit hole this will be :slight_smile:

-------------------------

pgrange | 2024-02-14 09:23:43 UTC | #3


Thank you so much for your feedback @t-bast and don’t worry, I’m scared just the appropriate amount ;) Your answer doesn’t make it worst yet. On the contrary, it really helps me thinking so it’s really appreciated :)

> The first one is that the wallet won’t be able to run in the background while the user isn’t monitoring their phone. That’s an important drawback, because most payments are received while the app is not open and is instead woken up on-the-fly by the LSP (and runs in the background).

Indeed, that is a strong difference with the current phoenix operation and probably any non-custodial mobile wallet in fact. Today, as you mention, the app runs in the background, is woken up by the LSP and the payment is received transparently for the user, except a notification to share the good news with them. With my approach, for a payment to be received, we will have to, first, notify the user that a payment is on the way and ask them to connect their ledger to accept it and, then, ask them to validate some transactions on it. I can see how this can make receiving a payment very slow and, maybe even painful for the user. They will have to have their hardware device at hand to be able to receive the payment.

So there’s at least two impacts on receiving payment: payment technical process itself and user experience.

For the payment technical process, I’m not sure how different this would be from the situation where the user’s phone is off when they receive a payment. How does the LSP deal with off phone now? Can we get inspiration from that to handle this hardware device complications?

For the user experience it can well be that it’s just super painful but maybe acceptable in some cases where the stakes are high enough for them to make the extra effort. Also, I have the idea that the user is already often involved today when receiving a Lightning payment. It’s the case when I’m creating an invoice, sharing the invoice with some payer and, the payment being fast with lightning, I just see it arrive on my phone. For that use case, « synchronous payment » I can imagine that the impact will be way smaller. The user already has their phone in their hand and they « just » need to plug their secure device to validate payment reception.

> The second one is that you will still need the hardware device to be stateful and implement non-trivial policies, similar to what VLS does […]
> There are also a lot of “background” operations happening all the time that require signatures (on-the-fly splicing, commitment fee updates, etc) […]

That’s a very good point. I believe VLS has strong value for such a wallet I want to create and that, at some point, there needs to be non trivial logic embedded in the hardware. As you noticed, I do not even plan to develop any code embedded in the hardware device for this iteration of the project but only rely on capabilities already exposed by the device. So I would expect quite some drawbacks with this first iteration. Will it be really more secure if the user validates transactions on their hardware device that just look gibberish? Will the user have to spend their time validating meaningless transactions on their device for any protocol update?

> you shouldn’t underestimate how big of a rabbit hole this will be 

Thank you for the kind warning. Coming from you, it's a confirmation to be humble, and have the smallest possible scope for the very first iteration of this project. In particular, focusing on integrating with a ledger as it is and see what happens. I can imagine this first application as a testbed upon which we can reflect, learn and build further. Maybe some advanced and motivated users will find some value in trying it already. At least, we’ll be able to observe it, play with it and realize where efforts should be put in priority. Is usability in practice as painful as we might fear now or are there some use cases that are ok-ish? Could we think of an ad-hoc LSP that would be more suited for this use case and ease some of the process? Would we be able to design and propose some BLIP to make this approach more practical?

-------------------------

