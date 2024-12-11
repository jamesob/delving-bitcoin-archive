# Disclosure: irrevocable fees---stealing from LN using revoked commitment transactions

harding | 2024-12-11 00:18:59 UTC | #1

Old versions of all four major LN implementations would accept a series of  operations that could allow a miner to steal up to 98% of a channel's funds.  Additionally, honest users performing entirely normal operations could obtain states that could be used similarly (although the worst-case loss would almost always be less than 98%).

Eclair, LDK, and LND with default settings (and Core Lightning with non-default settings) were vulnerable to an immediately exploitable version of the attack.  All versions of all LN implementations, even today, are vulnerable to a theoretical version of the attack that depends on natural variations in estimated onchain transaction fees; however, current-generation LN implementations have tightened their bounds to limit the maximum vulnerable amount per channel.  Eliminating all variants of the vulnerability depends on changes to both the LN protocol and the Bitcoin P2P transaction relay protocol.

Pull requests and releases mitigating the most critical vulnerability are listed below:

- [Eclair #2815][] (merged 27 Feb 2024).  Released in [Eclair v0.10.0][] (29 Feb 2024).

- [LDK #3045][] (merged 7 May 2024).  Released in [LDK 0.0.123][] (8 May 2024).

- [LND #8824][] (merged 30 Jul 2024).  Released in [LND 0.18.3-beta][] (11 Sept 2024).

Core Lightning remains vulnerable to the worst-case scenario if the `--ignore-fee-limits` configuration setting is enabled.  The documentation for this setting currently contains a warning that implies to knowledgeable users that they may lose funds ("This may result in a channel which cannot be closed").  The lead maintainer plans to update the configuration documentation to describe the risk of this vulnerability after this disclosure is made public.

[eclair #2815]: https://github.com/ACINQ/eclair/pull/2815
[eclair v0.10.0]: https://github.com/ACINQ/eclair/releases/tag/v0.10.0
[ldk #3045]: https://github.com/lightningdevkit/rust-lightning/pull/3045
[ldk 0.0.123]: https://github.com/lightningdevkit/rust-lightning/releases/tag/v0.0.123
[lnd #8824]: https://github.com/LightningNetwork/lnd/pull/8824
[lnd 0.18.3-beta]: https://github.com/lightningnetwork/lnd/releases/tag/v0.18.3-beta

## Vulnerability description

This section assumes the reader is at least moderately familiar with the LN protocol.  Those needing extra context are recommended to read Appendix A to this post.

### Type 1 "and... it's gone" irrevocable fees

In a channel opened by Mallory with counterparty Bob, Mallory performs the following steps:

1. Moves 99% of the balance in a channel to her side, e.g. by rebalancing or receiving funds.

2. Updates the offchain LN commitment transaction to allocate 98% of the balance to onchain transaction fees.

3. Again updates the commitment transaction to restore fees to a minimum acceptable level.

4. Moves 99% of the balance to Bob's side of the channel.

5. Broadcasts or personally mines the version of the transaction that pays 98% of the balance to fees.

If Mallory's transaction is confirmed, Bob will be able to keep the 1% of the channel balance on his side in that transaction and may be able to recover up to 1% from Mallory but will lose the other 98% of the channel value.  Whoever mines the 98%-fee transaction will get to keep its fees.  Mallory does not need to broadcast the unconfirmed commitment transaction, so she can easily rent hashrate and personally mine it herself, taking as long as she wants (as long as the channel remains open).  Assuming a commitment transaction size of about 300 vbytes, Mallory can capture 98% of value from over 3,000 channels per block (each channel potentially stealing from a different victim).

After Mallory steals from potentially thousands of channels in her first block, many node operators will be alerted to the attack and might try to respond by shutting down their LN nodes.  However, that would not protect them if Mallory had already created the 98%-fee offchain transaction.  Other node operators may respond by attempting to force close their channels in the latest state where they receive 99% of the value.  That will also not necessarily protect them: Mallory can simply broadcast each 98%-fee transaction to all miners for them to take value from the victims (although this will cause Mallory to lose 1% of each channel's value without recompense).  No fee-maximizing miner will mine an honest transaction with minimal or modest fees when Mallory's 98%-fee transaction is available.

At the time of writing, it appeared possible to rent about 1% of network hashrate for long enough to have a high probability of mining at least one block at a cost of about $50,000 USD over the recent average block reward.  Mallory would also lose about 1% of value in each channel.  For 3,000 channels with an average value of $1,000 ($10 of which would be a cost and $10 of which would stay on Bob's side), Mallory could net $2.89 million USD within a day.  The $1,000 average value per channel is arbitrarily chosen for this example; a higher or lower value would increase or decrease Mallory's expected gain accordingly.  To create the channels and rent the hashrate, Mallory would need several hundred thousand dollars in capital and the technical knowledge to modify LN software and set up a private Stratum work server.

### Type 2.0 "dark side" irrevocable fees vulnerability

Honest users who correctly follow the LN protocol could find themselves holding a state that allows stealing from their counterparty.  For example, in a channel opened by Alice with counterparty Bob, the following actions could occur:

1. Some of the balance in a channel moves to Alice's side, e.g. by rebalancing or receiving funds.

2. As feerates naturally rise, her node updates the offchain LN commitment transaction to allocate many satoshis to onchain transaction fees.

3. As feerates naturally fall, her node again updates the commitment transaction to restore fees to a minimum acceptable level.

4. Alice discovers she has obtained a vulnerable state that allows her to steal from Bob.  She gives into the dark side and moves 99% of the balance to Bob's side of the channel.

5. She broadcasts or personally mines the version of the transaction that pays a large number of satoshis to fees.

In this version of the vulnerability, it's important to consider the effect of the attacker losing the 1% remaining on their side of the channel when the revoked state is broadcast, which can make the attack cost-prohibitive against high-value channels.  For example, assume a peak feerate of 333 sats/vbyte on a 300 vbyte commitment transaction allowing theft of about 100,000 sats.  On a channel of 1,000,000 sats, the victim recovers 10,000 sats (1%) from the attacker, reducing the loss to 90,000 sats; on a channel of 10,000,000 sats, the victim recovers 100,000 sats, making the attack unprofitable.

In practice, many LN implementations choose a feerate higher than current network estimates to ensure the commitment transaction will still be confirmed quickly even if feerates rise further.  So if the peak estimated feerate was 333 sats/vbyte, the actual commitment feerate might have been 3x higher (1,000 sats/vbyte)---in our example, this would make the attack profitable against channels up to about 30,000,000 sats in value.

In channels that use the original LN commitment format, the fees need to pay for the entire cost of rapid confirmation.  In channels that use anchor outputs, the fee only needs to be above the dynamic mempool minimum, which is often significantly lower than the cost of rapid confirmation.  Continuing the previous example, an original-style channel that is safe at 30,000,000 sats might be compared to an anchor-style channel with a commitment feerate 1/10th as high that becomes safe at just 3,000,000 sats.

As described above, the amount that can be stolen per channel in this version of the vulnerability is limited.  That amount can be multiplied by a user with multiple channels, but few honest users have large numbers of channels (especially not the roughly 3,000 channel maximum that can be closed in a single block).  The potential net gain from the attack must be weighed against the time and cost of modifying LN software and attempting to create a block, which may be significant for those who don't already control enough hashrate to mine a block within the lifespan of most channels (e.g., a few months).

Additionally, LN nodes do not need to store most details about revoked states, and most implementations discard unnecessary information.  That means honest users who do not regularly create backups (and retain them for weeks or months) may be unable to reconstruct the past revoked states necessary to perform this attack.

For all the reasons described above, I don't consider the type 2.0 irrevocable fees vulnerability to be a major risk.

### Type 2.5 "long con" irrevocable fees vulnerability

Each participant in a channel independently estimates the appropriate onchain feerate in a non-deterministic fashion, so even honest participants often arrive at different estimates.  Nodes deal with this expected divergence by accepting a broad range of proposed fees---including fees much higher than they calculate are necessary.  This can be exploited.  For example, in a channel opened by Mallory with counterparty Bob, the following steps are performed by Mallory:

1. Moves up to 99% of the balance in a channel to her side, e.g. by rebalancing or receiving funds.

2. During a period of high feerates, iteratively updates the offchain LN commitment transaction with increasingly higher fees until Bob rejects a proposal by disconnecting.

3. Reconnects to Bob and updates the commitment transaction to restore fees to a minimum acceptable level.

4. During a period of low feerates, moves 99% of the balance to Bob's side of the channel.

5. Broadcasts or personally mines the version of the transaction that pays the greatest amount to fees.

In this case, Mallory is attempting to increase fees as high as possible, she is likely performing the attack simultaneously against hundreds or thousands of channels, and she may already have set up the infrastructure she needs to mine a block once she obtains enough vulnerable states.

Multiple mitigations for type 2.0 and type 2.5 vulnerabilities have been implemented and proposed, but the likely ultimate solution depends on allowing commitment transactions to have a static feerate.  That depends on upgrades to the Bitcoin P2P relay protocol for reliable [package relay][] as well as upgrades to the LN protocol to take advantage of those features (and disable the `update_fee` mechanism).

To minimize the amount that can be stolen until upgrades are fully deployed, nodes should close older non-anchor channels and only allow new channels to be opened using anchors.  As noted above, the peak feerates on anchor channels are often an order of magnitude or more lower than non-anchor channels, limiting the amount that an attacker can steal.

[package relay]: https://bitcoinops.org/en/topics/package-relay/

## Theft versus vandalism

For Mallory or Alice to profitably steal from Bob, they must mine a block---either by themselves or in cooperation with a miner.  However, they may at any time simply broadcast a revoked state transaction that allows any miner to receive a portion of Bob's funds.  This is vandalism rather than theft.  Even honest miners are expected to confirm revoked states and receive victim funds.

If Mallory or Alice broadcast a revoked state, they will lose the 1% of the channel funds they have remaining in the channel's latest state.  This should serve as a disincentive against vandalism.  However, there are vandals in this world who are willing to pay at a 1:98 ratio to hurt other people, so this 1% disincentive does not represent strong security.

## Deployed mitigation and proposed solutions

Eclair, LDK, and LND now limit the maximum they will pay to transaction fees to a _tolerance_ multiple of their estimated transaction fee.  Core Lightning has always enforced this limit, except when the `--ignore-fee-limits` configuration option is set to true.  This limits the maximum amount of money a malicious counterparty can extract using this attack.  This mitigation was easy to deploy in relative stealth and completely fixes the type 1 vulnerability.  It also bounds the worst-case loss from the type 2.0 and 2.5 vulnerabilities.

As noted above, allowing static commitment fees would completely eliminate the vulnerability.  This requires changes to both the Bitcoin P2P transaction relay protocol and the LN protocol, and developers of both protocols have already made significant progress in that direction.

Other complete fixes were discussed during the private disclosure process, such as tracking a fee high watermark and not allowing funds beyond that amount to move to the other side of the channel.  For example, if Alice put 5% of the channel value into fees, Bob would never accept a later state where more than 94% of the channel value moved to his side (100% - 5% fees - 1% Alice's channel reserve).  However, this would result in a UX degradation and participants believed that developer time was better used to implement static-fee commitment transactions that wouldn't degrade UX and would come with other benefits for safety and convenience.

## Discovery

On 1 Jan 2024, during a discussion of pre-signed fee bumping, Bastien Teinturier [wrote][teinurier endo fees]: "[...] if you pre-sign commitment transactions at various feerates, the actual balance that you can use off-chain is the balance of the commitment with the highest feerate."  I [summarized][news284 endo] his point and related discussion as, "if Alice signs fee variants from 10 s/vb to 1,000 s/vb, she must make decisions based on the possibility that her counterparty Bob will put the 1,000 s/vb variant onchain, even if she wouldn’t pay that feerate herself. That means she can’t accept payments from Bob where he spends the money he would need for the 1,000 s/vb variant."

On 30 Jan 2024, Matt Corallo [posted][corallo cov fees] to Nostr: "[..] Allowing the transaction broadcaster to simply reduce their balance to pay fees at broadcast-time [using a covenant] would solve one of the biggest pain points for LN."

I began drafting a reply to Corallo to ask how that could ever be done securely given that (as Teinturier pointed out) each user has to assume their counterparty will close the channel in the state with the highest fees.  As I was slowly tapping out that message on my phone, I remembered the existing LN protocol allowed adjusting feerates and I wondered what protection it had against an old state paying more in fees than a party had in the most recent state.  I deleted my draft reply to Matt and made a note to investigate further when I got back to my computer.

The BOLT specification made it clear that feerates could be decreased and I remembered [summarizing][news170 eclair1980] an Eclair pull request that allowed one party to set feerates arbitrarily, as long as they were above the counterparty's minimum and resulted in a valid commitment transaction.  I immediately sent an email to Teinturier, who is an Eclair maintainer, and followed up later that day and the next with email to the maintainers of the other implementations as I discovered they were vulnerable too.

[teinurier endo fees]: https://github.com/bitcoin/bitcoin/pull/28948#issuecomment-1873793179
[news284 endo]: https://bitcoinops.org/en/newsletters/2024/01/10/#an-alternative-use-endogenous-fees-with-presigned-incremental-rbf-bumps
[corallo cov fees]: https://nostrapp.link/note1s4vgrkwgemars2szftt52367f0xs6kr4e9pl60t78yxly279gxksmwx94t
[news170 eclair1980]: https://bitcoinops.org/en/newsletters/2021/10/13/#eclair-1980

## Timeline

(All dates UTC-10 unless otherwise noted)

- 2024-01-31: discovered vulnerability in Eclair, CLN (with non-default settings), and LDK; reported to maintainers
- 2024-02-01: also discovered in LND; reported to maintainer
- 2024-02-01: maintainers began discussing solutions
- 2024-02-27: Eclair merges #2815; released two days later
- 2024-05-07: LDK merges #3045; released next day
- 2024-07-30: LND merges #8824; released forty days later
- 2024-11-08: disclosure scheduled for Dec 11 UTC (all maintainers notified; 3 ACKs; 0 NACKs)
- 2024-12-10: public disclosure

## Acknowledgments

This vulnerability is based upon a principle I learned from Bastien Teinturier.  He and Matt Corallo were especially active in discussing the implications of the vulnerability and both developed the patches for their respective implementations.  Eugene Siegel wrote the patch for LND; he and Olaoluwa Osuntokun also made multiple insightful contributions to the discussion.

Several of those named above, in addition to a large number of other contributors to Bitcoin and LN, have been working for years to make fee management for offchain contract protocols simpler and more robust.  Without their often underappreciated effort, we would not have a clear path to eliminating this class of vulnerability.  On the Bitcoin Core side, I especially thank Suhas Daftuar, Mark "Murch" Erhardt, Greg Sanders, Pieter Wuille, and Gloria Zhao.

Larry Ruane, Mike Schmidt, and Vojtěch Strnad kindly reviewed a draft of this disclosure barely 24 hours before it was due to be published.  Any remaining errors are entirely my fault.

## Appendix A: LN background

This appendix describes parts of the Bitcoin and LN protocols that are necessary to understand the vulnerability.  The last paragraph of this section also describes how I chose the name of the vulnerability.

### Commitment transactions and `update_fee`

BOLT2 specifies that the party who initiates opening a channel (e.g., who funds it in the case of single-funded channels) is responsible for paying any [endogenous][topic fee sourcing] onchain transaction fees associated with the channel.

Parties in an LN channel commit to their current balances in the channel by signing _commitment transactions_.  These are fully valid Bitcoin transactions that comply with the _standard transaction_ rules that allow them to be relayed by nearly all relaying full nodes and mined by all known miners.

Like any other Bitcoin transaction, commitment transactions may contain onchain transaction fees.  From the inception of a functional LN in 2017 until the present, all commitment transactions must include at least some fees if they want to be relayed using the P2P network.  Specifically:

- Non-anchor commitment transactions (added in the original protocol; being gradually phased out as of 2024) had to pay a high enough feerate to be confirmed within a few blocks (e.g., within 3 blocks).

- _Anchor_-style commitment transactions (added to the LN specification in August 2020) had to pay at least the dynamic mempool minimum feerate, which had a floor of 1 sat/vbyte but would frequently rise above that floor.

If a commitment transaction carrying payments (_HTLCs_) pays too low of a feerate, it may not be possible to get it relayed and confirmed before some of those HTLCs expire and one of the parties in the channel is at risk of losing money.  That makes it essential to choose an appropriate feerate.

The fee necessary to get a transaction into mempools or confirmed can vary over time.  For that reason, BOLT2 specifies that the party responsible for paying fees (and only that party) may send an `update_fee` message to propose a feerate to use for all future commitment transactions (until another `update_fee` message is accepted).

The `update_fee` message can propose either an increase or a decrease to the feerate to use.  The counterparty can accept or reject the proposal.  If they accept, they respond with a signature for an updated commitment transaction that uses the new feerate.  If they reject the proposal, they either reset the connection (continuing the use of the old feerate and allowing a new `update_fee` proposal to be sent) or they broadcast the latest commitment transaction (called _force closing_ the channel).

Because a too-low feerate may result in a later loss of money, all versions of all implementations will not accept `update_fee` proposals for a feerate below their own feerate estimates.

However, because commitment transactions may be created hours to weeks before they need to be broadcast and confirmed, a period during which feerates may increase, BOLT2 recommends overpaying commitment transaction feerates: "Given the variance in fees, and the fact that the transaction may be spent in the future, it's a good idea for the fee payer to keep a good margin (say 5x the expected fee requirement) for legacy commitment [transactions]; but, due to differing methods of fee estimation, an exact value is not specified."

For that reason, all versions of all implementations have accepted `update_fee` proposals that suggested paying higher fees than the minimum currently required.  Additionally, because the minimum required at one time may be greater than the minimum required at a later time, it was possible for entirely correct behavior to have created a commitment transaction in the past that used feerates far higher than the minimum required at present.

There are no LN protocol limitations on how often a node may use the `update_fee` mechanism to adjust the commitment transaction fees.  For the update to be accepted, the receiving counterparty must reply with a signed commitment transaction, which typically takes less than a second.  This allows a node to propose the highest feerate allowed, followed approximately one second later by a proposal for the lowest feerate allowed.

### Revoked commitment transactions and channel reserve

Bitcoin consensus rules ensure that any transaction that can be added to the blockchain at a point in time can also be added to the blockchain at any later time, barring large blockchain reorgs; this is known as _reorg safety_.  This means any version of the commitment transaction from an LN channel is eligible to be added to the blockchain (provided no other version has already been added).

A goal of the LN protocol is to incentivize users to prefer confirmation of the latest commitment transaction.  The LN-Penalty protocol (used for all deployments to date) accomplishes this by forcing a party who publishes an outdated state to give their counterparty an exclusive time window during which the counterparty can seize all funds in that outdated state (called a _revoked state_).  For example, if Mallory broadcasts a revoked state and it is confirmed into a block, her counterparty Bob can claim all of her funds from that channel.

Because this mechanism is based on economic incentives, BOLT2 recommends each party require their counterparty keep a _channel reserve_ of 1% of the total channel value.  That ensures Mallory will lose at least 1% of the total channel value if one of her revoked commitment transactions is confirmed onchain.

However, funds allocated to fees are given to the miner who includes a commitment transaction in a block regardless of whether it is the latest state or an older version.  Thus I think onchain fees in commitment transactions should be considered _irrevocable fees_.

[topic fee sourcing]: https://bitcoinops.org/en/topics/fee-sourcing/

## Appendix B: selected previous LN fee vulnerability disclosures

- [Fee blackmail][pickhardt blackmail] by René Pickhardt (2020): Bob opens a channel to Mallory (making Bob responsible for all endogenous onchain fees); Mallory sends large numbers of HTLCs to inflate the commitment transaction size, proportionally increasing the amount Bob must pay to fees if Mallory force closes.  Mallory blackmails Bob: pay her or she'll make him pay all those fees.  _Deployed solution:_ limit the maximum number of pending HTLCs in channels (e.g., reducing from the protocol-allowed maximum of 966 (483 in each direction) to a maximum of 20 (10 in each direction)); the default limit varies by implementation.

- [HTLC fee redirection][riard trim] by Antoine Riard (2020): Mallory opens an original [anchor][]-style channel to Bob and ensures 99% of the channel balance is on her side; she creates many low-value HTLCs with each second stage HTLC-X transaction paying an endogenous fee; Bob signs each HTLC-X transaction with `SIGHASH_SINGLE|ANYONE_CAN_PAY`.  Mallory then moves 99% of the balance to Bob's side and broadcasts the old (revoked) commitment transaction.  Mallory takes advantage of Bob's `SIGHASH_SINGLE|ANYONE_CAN_PAY` signatures to aggregate all of the HTLC-X transactions into a single transaction that only pays one of the endogenous fees, with the remaining fee value going to Mallory.  This allows Mallory to capture a significant amount of channel value without miner assistance.  _Deployed solution:_ updated anchor-style channels all use zero-fee HTLC-X transactions; whoever broadcasts an HTLC-X transaction must provide an input for purely [exogenous fee sourcing][topic fee sourcing].

- [Excessive trimmed HTLCs][riard trim] by Antoine Riard (2021): any two parties have a channel; a large number of [trimmed HTLCs][topic trimmed htlc] are routed across it; the channel closes and all of the value in the trimmed HTLCs is paid in fees to miners rather than returned to the channel participants.  This can be exploited for profit or used at minimal cost to cause third parties to lose money.  _Deployed solution:_ bound the maximum amount of channel value that can be placed in trimmed HTLCs.  Trimmed HTLCs are sometimes called _dust HTLCs_, so the limit is sometimes called _max dust exposure_.

In most implementations, the deployed mitigation for the irrevocable fees vulnerability builds on the solution to the excessive trimmed HTLCs vulnerability.  The maximum dust exposure limit is expanded to become an overall maximum fees tolerance.  This was a practical repurposing of existing code and aided in stealth deployment of the mitigation.  However, I think it's worth remembering that the problem of endogenous fee sourcing in an offchain state protocol is separate from the problem of uneconomical outputs.  Entirely exogenous fee sourcing may be able to solve the former problem (although introducing some tradeoffs); the latter problem requires either continuing to accept bounded risk or the development of new protocols for small-value payments.

[pickhardt blackmail]: https://diyhpl.us/~bryan/irc/bitcoin/bitcoin-dev/linuxfoundation-pipermail/lightning-dev/2020-June/002735.txt
[riard fee redirection]: https://diyhpl.us/~bryan/irc/bitcoin/bitcoin-dev/linuxfoundation-pipermail/lightning-dev/2020-September/002796.txt
[riard trim]: https://diyhpl.us/~bryan/irc/bitcoin/bitcoin-dev/linuxfoundation-pipermail/lightning-dev/2020-September/002796.txt
[anchor]: https://bitcoinops.org/en/topics/anchor-outputs/
[topic trimmed htlc]: https://bitcoinops.org/en/topics/trimmed-htlc/

-------------------------

