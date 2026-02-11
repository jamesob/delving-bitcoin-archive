# Hourglass V2 Update

theblackmarble | 2026-02-10 20:54:22 UTC | #1

In response to feedback, the Hourglass proposal to mitigate against potential mass liquidation of P2PK funds has been enhanced to further limit spend amounts from such outputs to only 1 bitcoin per block.

https://github.com/cryptoquick/bips/blob/4587b5c5af5e1f76c0848c7b352cd92e628b0fef/bip-hourglass-v2.mediawiki

Prior discussion of the original Hourglass proposal:

https://groups.google.com/g/bitcoindev/c/zmg3U117aNc/m/lDCMs9j7EAAJ

Thoughts & feedback welcome!

-------------------------

orangesurf | 2026-02-10 21:35:23 UTC | #2

1.7M btc in P2PK outputs restricted to 1 btc per block means it would take 32 years for all these coins to migrate.

Even if as much as 75% of these coins are lost, it would still take 8 years.

Even if the value of lost coins is far greater, say 99%, it would still take 118 days.

In that time, there would be massive competition between legitimate holders to try and regain unthrottled access to their coins - leading to a large amount of funds being transferred from holders to miners.

-------------------------

theblackmarble | 2026-02-10 22:15:36 UTC | #3

(post deleted by author)

-------------------------

theblackmarble | 2026-02-10 22:17:00 UTC | #4


Hourglass would ideally be activated as a flag day soft fork at a certain block height announced well in advance.  Anyone having access to their keys and awareness of activation would be highly incentivized to move them prior to it (though likely no more incentivized to liquidate).  This is the same with any confiscation proposal.  If 25% or greater of P2PK coins are not lost, I would expect the vast majority of them to move prior to activation.

Instead of freezing/burning these coins, with Hourglass there is still a chance to reclaim them after activation.

It is reasonable to assume that if somebody leaves their coins in P2PK outputs through activation, they have an extremely low time preference.  While nobody can directly differentiate between a quantum attack and original keyholders moving their coins, actors with such a low time preference are less likely to bid up fees than quantum attackers, so long as they are unconvinced of active quantum threat.  Once whatever period of activity due to original keyholders moving coins ends, the lack of P2PK activity will be noticeable on chain, prompting those with a lower time preference to move their coins.

-------------------------

