# A Proposal for Trustless Custody

ynniv | 2025-12-22 00:28:12 UTC | #1

A while back I proposed an idea for a layer 3 protocol that might address some fundamental problems Bitcoin faces. A work in progress, proposed by someone with little visible work in the community, riding the AI tsunami. Not surprising that it collected many pageviews but little engagement. FWIW, everything that was posted to delving was written solely by yours truly, banging on cherry blue keys.

Since then AI has become even more controvertial. Contributions appearing to be generated are in some cases automatically closed. There is trepedation that AI can provide incremental value at this level at all. In response, I am doubling down. All of my designs, documentation, and 50k lines of prototype code have been written by Claude, full stop. Unless otherwise stated, assume AI "slop". Out of respect, everything posted to delving remains written by my own fingers smashing keys.

This isn't due to laziness or naivete. I'm not here to waste anyone's time. I've spent over two decades writing software, including mission critical systems at fintechs that millions of people use every day. My work may be invisible, but it's out there. Employing AI is my intentional commitment to understanding what has rapidly progressed from possibility to undeniable force. Learning how to use it effectively in our work is essential to staying relevant in the world. To be buoyant as the water rises.

## Bitcoin Deposits: A Layer 3 Protocol for Trustless Custody

Bitcoin is full of compromises. Transactions take much longer than card payments. Small amounts are buried in transaction fees. Functionality is limited by what the game theory supports. There are many proposed solutions, each making different shapes of compromise. Most require various amounts of trust, some require availability, give up privacy, or require opcodes that haven't found consensus. I'd like to focus on one that's generally overlooked: custodial Lightning.

Custodial Lightning has great UX. It's cheap, it's fast, it works today. Everyone seems to agree that it's a beautiful solution ... except for the trust aspect. You have to trust the custodian fully. The only defenses against dishonesty are reputational harm and the court system. Most people agree that this is so fundamentally counter to the value of Bitcoin that they don't see it as a solution at all. Paper Bitcoin, at best.

At this point some propose incremntally improving the situation by adding privacy in the form of blinded tokens. Ecash undeniably improves the privacy of custodial systems, but as it doesn't also address unavoidable trust it's easy to see why victory hasn't been declared. There are other potential improvements for reducing this trust, but fundamentally the mechanism of ecash prevents a fully trustless system from ever being achieved. One point that has been top of mind for me is that the privacy of ecash prevents trustless transferrability, thus requiring honest mints to operate forever. Over time the expected value of all ecash tokens trends to zero.

Taking a step back, what are the sources of trust in custodial Lightning, and can we do anything about them?

### Transparency

Always the first thing that comes to mind, a trustless custodian must be auditable. Cryptographic ledgers are a well trodden solution to this: if every update is well formed and signed by the custodian, others can verify that the ledger itself is honest. We're all familiar with nodes validating the base chain updates published by miners.  The second half is proving reserves. Without sufficient funds, even a well formed ledger is dishonest. We commit funds together with an update hash and have all the information necessary to verify instead of trusting the honesty of a ledger.

### Accountability

To incentivize custodians to be honest, we need some way to hold them accountable. Bitcoin does this by holding their block rewards hostage: if a miner produces a block that isn't consensus valid, someone else gets the reward. For Deposits I propose that custodians post collateral exceeding the ledger reserves across many channels with different partners. By putting this collateral at risk of confiscation, the protocol increases the number of parties that must collude in order to make theft profitable. And since collution should also be visible and proveable, the funds at risk grows linear to the number of parties involved. With appropriate collateral ratios and reasonable honest-participant levels, profitable theft in a well connected graph can be extremely difficult.

### Recovery

Perhaps the most novel, and thus surprising, aspect of the protocol is recovery. Almost every level 2 protocol has made it a point to provide unilateral exit to wallets, to the point of some making it part of the definition of being a layer 2. Deposits was explicitly designed to avoid unilateral exit, promising only that funds will remain available in the system. This and the reliance on Lightning are why I am calling it a layer 3.

As part of holding custodians accountable, partners are able to confiscate funds. This is done by moving reserves to a separate output on the commitment transaction that spends to a multisig of partners with which a custodian is keeping collateral. In the event of unilateral close, the multisig will vote (visibily and undeniably) on whether the funds landing on the chain are correct for the ledger hash that accompanies them. Voters synchronize ledger updates amongst themselves and validate whether the chain ending in this hash is conforming or not. If yes, they vote to restore the funds and ledger to the custodian. If not, they vote to reassign both ledger and reserves. Vote honestly, and everyone agrees. Vote dishonestly, and a voter finds their own channels closed and collateral put up for judgement.

It is therefore critical that we have transparent accounts. The new custodian must be able to serve existing wallets without any additional information. But, this is no different from the base chain, and there are proposals to improve the privacy of such a system.

## Conclusion

It seems possible to provide no-maintenance, zero UTXO Lightning wallets that scale Bitcoin by creating a web of proportionally bonded custodians on the Lightning network. Honest participants benefit from routing and maintenance fees, as well as funds forfeited by dishonest participants. Wallets trade fees for convenience, knowing that their funds will remain available even if their custodian disappears or is dishonest. And we can do these things with the Bitcoin that we already have.

## A Request

Once again I have found myself building something that no one has asked for. It's risky, complex, and unexpected. Yet it may be a viable solution to some existential problems that Bitcoin is struggling with. What would the world look like if we didn't need to trust custodial Lightning? How many custodians would there be? Where would they live, and how might that influence the way they execute the bits of discretion we haven't removed yet?

For the last eight months it's been me, Claude, and the occasional, often anonymous, review. I'm currently experimenting with an ldk-node based implementation with MCP NWC wallets on mutinynet, but I still feel far from something that should be holding real wealth. It's time to find out whether this is something that people really need, or just something they didn't ask for. If you have the time, and can look past the "slop", please take a look. I'd really love for someone to point out why this whole thing won't work so that I can get back to a day job.

ynniv / npub12akj8hpakgzk6gygf9rzlm343nulpue3pgkx8jmvyeayh86cfrus4x6fdh

winter solstice, 2025

https://github.com/ynniv/deposits/blob/60593299f2bd638712b2c9858d6d3e99c9c0c0bc/WHITEPAPER.md

-------------------------

