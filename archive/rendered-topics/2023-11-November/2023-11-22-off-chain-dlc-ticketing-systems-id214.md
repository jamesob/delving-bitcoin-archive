# Off Chain DLC Ticketing Systems

conduition | 2023-11-22 01:40:54 UTC | #1

Hey plebs. I recently had an idea which I believe enables simple off-chain participation in [discreet log contracts (DLCs)](https://bitcoinops.org/en/topics/discreet-log-contracts/). 

[I wrote about it on the dlc-dev mailing list here.](https://mailmanlists.org/pipermail/dlc-dev/2023-November/000182.html)



The email above describes the protocol in more detail, but the TLDR is:
- One person called the _market maker_ can commit all the on-chain capital into a single multisig contract UTXO, sharing custody with the other DLC participants. 
- By cooperatively adaptor-signing a specific set of outcome transactions in advance, the _market maker_ can create _tickets_ (preimages, or discrete logs), which they sell to their peers using off-chain HTLCs or PTLCs. 
- A ticket buyer gains certain rights to be paid depending on the outcome of the DLC, as signed by the oracle.
- Once the outcome is published by the oracle, ticketholders who stand to gain from the outcome can receive their payout from the _market maker_ off-chain.  

The benefits:
- By removing on-chain buy-ins and enabling off-chain payouts, we gain huge fee savings where the number of players is large, and also improve privacy of all players. It is especially effective for contracts with many players, but few winners, such as lotteries.
- New business opportunities for those with capital available to lease.
- DLCs can now have microtransactions. Imagine a lottery where 1 million people can all buy in with a single satoshi each. Imagine a video game where the players all buy in with a few dozen satoshis, and the best performing players can take home a few thousand sats. 
- The market maker never has custody of funds. As with any DLC, only the oracle can decide the outcome.

I'm creating this thread for two reasons:
1. I want to know whether I'm insane, and if this is indeed possible. Can anyone spot a bug? 
2. What use cases can you think of where this concept could be best applied? 


(Small aside, I did make one error in my email to the dlc-dev list. I said that the market maker is the one who encrypts the signatures. In fact, the players must be the ones to encrypt their own partial signatures _before_ sending them to the market maker, otherwise collusion is possible)

-------------------------

harding | 2023-11-23 02:35:32 UTC | #2

I think the major efficiency gains that this depends on requires that each participant be a member of the multisig (so the contract is trustless) and then also be available after all tickets have been settled in order to sign a non-contracted payment back to the market maker.  The more signers that are required, and the more time that passes between setup and teardown, the less likely it is that all signers will be available, requiring all of the individual outcome transactions to go on chain.

To support the above claim, here's how I understand your proposal:

- A maker agrees to work with Alice and Bob.  The maker proposes (but does not sign) a transaction paying the n-of-n multisig {A, B, M}.  Alice and Bob sign a timelocked refund from that transaction back to the maker, ensuring the maker can reclaim their funds even if Alice or Bob later become unavailable.  The maker signs and broadcasts the funding transaction.

- The maker then sells tickets to Alice and Bob over LN.  I think this likely requires PTLCs given that DLCs are working with signatures.  If you have more details on how that would work, I'd appreciate them.

- Alice wins.  Her winning ticket is an output that can be spent by her unilaterally after a certain delay, or which can be spent by the maker at any time if they learn a preimage from Alice (or PTLC scalar).  Alice uses LN or ecash to sell the preimage to the maker.

- In the meantime, Alice and Bob have bought and settled 99 other tickets with the maker in the same way.

- The maker now has enough information to claim all 100 outputs onchain, but that would be just as inefficient as Alice and Bob claiming them individually.  If Alice and Bob cooperate, they and the maker can sign an alternative transaction with a single output paying back to the maker---but this requires Alice and Bob to be available.  As the number of participants increases, the likelihood of everyone being available decreases.

- The maker needs to settle all of the claims before the timelocks on the unilateral spends expires, or the participants can take back the funds they've swapped over LN or ecash.  This deadline for the participants also needs to be earlier than the timelocked refund to the maker (or the maker would've been able to steal from the participants), so the maker won't be able to just wait and sweep all unclaimed funds.

I'm not sure I fully understood your idea, so please let me know if I missed something.

-------------------------

conduition | 2023-11-23 20:35:12 UTC | #3

Thanks for the questions @harding. If you'd like to understand the protocol in more detail, i think you'd find my original email to the dlc-dev mailing list helpful: https://mailmanlists.org/pipermail/dlc-dev/2023-November/000182.html

I think that should clear up most of your confusion, but let me try to answer a few things directly:

> I think the major efficiency gains that this depends on requires that each participant be a member of the multisig (so the contract is trustless) and then also be available after all tickets have been settled in order to sign a non-contracted payment back to the market maker. 

Only the DLC winners would need to be online and available to sell the on-chain contract TXO back to the market maker. 

Once the DLC oracle publishes the outcome signature, the winners of the DLC can prove they have sole custody of the contract output. If they cooperate during that time, the winners can sell the whole contract back to the market maker, in exchange for off-chain payments therefrom. 

The trick is in using multiple stages of transactions, not just a single n-of-n multisig address. 
- The first stage TXO (of $\text{TX}_{\text{init}}$) is owned by all players and the market maker. 
- The second stage TXO (of $\text{TX}_{\text{outcome } i}$) is owned by ticketholding winners, or the market maker as a fallback. 
- Adding a separate third stage ($\text{TX}_{\text{split }i}$) which splits the winnings would enable an 'all winners cooperate' path, where the DLC contract output doesn't need to be split into separate per-winner outputs. This would be useful for DLCs which have lots of winners.

I think a diagram might be helpful to visualize the TX flow. This diagram shows a contract with 3 players, Alice, Bob and Carol, plus a market maker. I added 'split' transactions (not described in my original email) to enable the 'all winners cooperate' path. 


<img src="upload://xq35HbL1frk44gngSHJJPOhIYgX.png">

[Mathcha diagram link](https://www.mathcha.io/editor/dxr6ncP8twWsGkYVZxHP52E1PhdgEoQMiwZlDV0)

- The big solid-lined boxes are transactions.
- The small boxes inside them labeled with $v$ are transaction outputs.
- The arrows indicate transactions which spend those outputs. 
- Locking conditions are written above and below the spending path arrows.
- $\Delta$ is a relative block delay timelock.


This diagram uses adaptor signatures and PTLCs, but you could easily swap that logic for hashlocks instead. 

> The maker then sells tickets to Alice and Bob over LN. I think this likely requires PTLCs given that DLCs are working with signatures. If you have more details on how that would work, I’d appreciate them.
>
> Alice wins. Her winning ticket is an output that can be spent by her unilaterally after a certain delay, 

Tickets are actually just preimages - or scalars depending on the implementation. In the above diagram, i represent them as locking conditions with $T_A$, $T_B$, and $T_C$. The ticket locking conditions are one stage _after_ the DLC oracle signature is needed, and i'm assuming the players all learn the outcome signature directly from the oracle for free (no PTLC needed).

You're correct about the winner being able to unilaterally spend her winnings output after a delay. But to spend this output, the winner needs her ticket secret/preimage. Without it, after a longer timelock, the output returns to the control of the market maker, via $\text{TX}_{\text{reclaim}}$.

> In the meantime, Alice and Bob have bought and settled 99 other tickets with the maker in the same way.

I think the protocol you're picturing diverged from mine beyond this point. Each player can only buy one ticket per DLC. The purchase of the ticket secret grants the player access to a specific set of payouts, contingent on the outcome signed by the oracle.

Also the protocol as i've described is only for a single DLC contract; i haven't yet considered how this might be applied to multiple DLCs.

-------------------------

