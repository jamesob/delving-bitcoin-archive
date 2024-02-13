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

