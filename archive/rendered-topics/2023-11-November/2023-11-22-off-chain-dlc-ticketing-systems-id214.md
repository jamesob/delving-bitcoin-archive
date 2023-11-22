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

