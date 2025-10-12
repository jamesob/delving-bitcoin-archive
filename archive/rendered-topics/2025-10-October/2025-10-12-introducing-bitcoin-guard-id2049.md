# Introducing Bitcoin Guard

guardian | 2025-10-12 07:48:15 UTC | #1

**What is Bitcoin Guard?**

Earlier this year I read an article about a person who was kidnapped and tortured for their bitcoin. It was the third such article I’d read in a couple of weeks, and the details & circumstances concerned me. I thought to myself there must be a technical solution to the growing number of physical attacks, and Bitcoin Guard is my effort to shift the balance of power from the attacker to the victim. The (draft) BIPs are available for anybody to implement and all code is open source. My motivation to build this is to make self-sovereign security available for everyone by changing game theory outcomes, adjusting the risk/reward for the attacker to increase risk and reduce reward. If adoption is large enough, this creates protective value since the attacker does not know if Guardians are configured, if there’s a physical response, and visually wallets are locked.

The goal of this project is to provide both technical enforcement that prevents bitcoin theft and enhance the physical security posture of the user.

A lightweight signalling protocol is implemented in a  bitcoin address which sets a boolean lock state. The subsequent lock state transition is a pre-signed transaction, which enables users to activate the emergency signal without carrying private key material.

This opens a wide range of activation methods as the implementation only has to `curl -X POST <sig> <rpc>` to lock the Guardian.

Since the protocol and implementation are (draft) BIPs, the wallet behaviour may be deployed by both self-custodial wallets and centralised exchanges, making this design suitable for a range of custody environments.

There are several components to this design:

* Bitcoin Guard Desktop Application
* Bitcoin Guard Mobile Application
* Guardian Address Signal Protocol Draft (GASPv1) BIP
* Guardian Wallet Implementation Draft BIP

From a UX perspective my goal is to make Guardian Addresses simple to use without knowing any protocol specifics. If the user knows how to use a hardware wallet they can use Bitcoin Guard effectively.

Please note all features are in active development and may be subject to change.

**Bitcoin Guard Mobile**

The mobile app is where many of the security features are configured and used.

![bitcoinguardmobile|288x500](upload://i5FabHyHOE0GNQPOVvFQLKTetA.jpeg)

An interactive prototype is available to try at https://bitcoinguardian.github.io/mobile-prototype

**Trusted Contact**

If a device is stolen by an attacker, such as in a street robbery, it might be difficult for the victim to broadcast the Guardian Lock.

The app supports multiple Locks ready to broadcast, so if a trusted contact has one of the victim’s pre-signed Locks, they can broadcast it on behalf of the victim.

Locks may only be used once, so if a user’s signature is broadcast maliciously the impact is limited to a single lock instance which is fully recoverable. See technical details for how this works.

**Countdown Timer**

Sometimes in attacks the victim is unable to manually broadcast a Guardian Lock. This is when the device has been taken or the victim is physically unable to broadcast.

Each lock has a configurable countdown timer which will automatically broadcast the signal and notify trusted contacts with the victim’s location (optional).

Optional notifications remind the user that the countdown is close to completion. Optional haptic feedback informs the user that the signal has been broadcast, providing reassurance that wallets are locked and their physical location has been shared (also optional).

The user sets the Countdown Timer before an anticipated period of risk, and the app broadcasts the Guardian Lock if not cancelled.

**Rage PIN**

Traditional decoy PINs are a solution implemented in many wallets to prevent theft of bitcoin. When a duress PIN is entered an imitation wallet holding a smaller amount of bitcoin than the main wallet is presented.

This still results in a loss of bitcoin and does not help the user to be physically safe.

Rage PIN is a feature of Bitcoin Guard that works together with a Countdown Timer as an evolution of traditional decoy PINs. If an attacker gains access to the device and sees the active Countdown Timer they will attempt to cancel. A PIN code is required to abort, and if it is entered incorrectly the Guardian Lock is broadcast, preventing spending from monitoring wallets and the duress physical location is sent to a trusted contact.

The victim may then offer the attacker a non-genuine PIN code that will make the attacker believe the countdown will be stopped. The outcome always favours the defender who has now both locked wallets and broadcasted their location to trusted contacts in either scenario.

This forces an attacker to either enter an incorrect PIN or wait for the Countdown Timer to expire and broadcast the emergency signal.

**Voice Activation**

In some attacks the victim is forced to open their device before being stolen. Voice Activation enables the user to lock their Guardian by using a voice command.

This allows the victim to put physical distance between themselves and the attacker, and still being able to activate without holding the device.

In another use case, during a home invasion a voice command could enable hands free activation of the Guardian Lock and intruder alert via a home assistant device.

**Third Party Integrations**

Third party services have been integrated as a notification layer for the user to alert a trusted contact that an attack is underway. The message is customisable by the user and may include location data for fast security/law enforcement response.

Currently Telegram and Twilio are planned, though I’d like to hear what the community would think beneficial here.

**Manual Activation**

There may be scenarios where a user decides to manually broadcast a Guardian Lock. Developing risks where a threat is present is one reactive use case.

Alternatively users may decide to manually activate their Guardian proactively before entering an unsafe juristiction.

**Managing Guardian Address**

An interactive prototype is available to try at https://bitcoinguardian.github.io/desktop-prototype

Bitcoin Guard Desktop is an application to abstract the protocol implementation and manage the Guardian Address. It runs locally on the user’s machine and supports hardware wallet connectivity.

![bitcoinguarddesktop|690x468](upload://eDfOSFdzBjv68acYG7PPTYvPLJ0.png)

Full Guardian lifecycle operations are supported, from setup to signals. Each operation has UI prompts that explain to the user what happens when each signal is sent. The user clicks the button, approves in the hardware wallet, and broadcasts the transaction.

Signal history is taken from chain data which makes it easy to evaluate both the current lock state and historical actions.

Both lock and unlock operations are supported from the application:

![bitcoinguardunlock|365x500](upload://4awASRrq3x7wnwoALMlzgK2KxDH.png)

When the user creates the pre-signed lock it is presented as a QR code for easy import by the Bitcoin Guard mobile application:

![bitcoinguardqr|382x500](upload://mdinwZmOcEATGtIzrDVPEeob1HS.png)

Users who wish to build their own custom activation methods may copy the transaction hex to use in their own implementation.

Please note that it is not connected on-chain and all actions are to demonstrate features while in development.

**Does it actually work?**

Yes, there’s a full implementation in my Electrum fork here: https://github.com/bitcoinguardian/electrum

I’ve broadcast signals on testnet and validated wallet behaviour which prevents spending when locked: https://github.com/bitcoinguardian/electrum/tree/master/demo

**How does it work?**

Technical details here: https://delvingbitcoin.org/t/proposal-guardian-address-gaspv1/2006

-------------------------

