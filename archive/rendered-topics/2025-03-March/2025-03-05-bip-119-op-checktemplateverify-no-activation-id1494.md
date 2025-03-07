# BIP-119 (OP_CHECKTEMPLATEVERIFY) (no activation)

marathon-gary | 2025-03-05 17:21:57 UTC | #1

A thread for general/conceptual discussion for Bitcoin Core PR #31989 outside of the PR review thread.

-------------------------

ariard | 2025-03-05 22:59:10 UTC | #2

See comments on the mailing list (pending) for any combination of CTV + another primitive, as it proposed by Bitcoin Core PR #31989, afaict.

If we go to discuss the merits of BIP-119, we should discuss of its *technical* merits alone. By design, it was very restraint to avoid OP_EVAL style of issue, again.

-------------------------

ariard | 2025-03-06 22:19:10 UTC | #3

I think the purpose of this thread asks for clarification, if the advocates of OP_CTV are only eager to discuss the technical merits of OP_CTV, or if it's for a combination with multiple primitives (as actually said so in #31989's OP).

As a historical reminder, there has been few previous iterations with OP_CHECKOUTPUTHASHVERIFY, OP_SECURETHEBAG, until we got OP_CHECKTEMPLATEVERIFY, where a reduced expressivity was actually one of the argument made by its original author, Jeremy Rubin, to progressively bring covenants in the Bitcoin space. If my memory is correct, there has been even change few times of the templates to make it committed to all inputs or to have a fixed template size, to avoid wider disruptive side-effects.

By bringing back on the table the combination with other primitives for a common activation, I believe one can say it's a regression on the idea to progressively bring covenants back in bitcoin. Now, one can argue there is no cost in this regression, problem it's not the case.

As explained on the mailing list, there is the old idea of [TxWithhold](https://blog.bitmex.com/txwithhold-smart-contracts/) by Gleb Naumenko, i.e costless bribes to mess up with time-sensitive protocols (i.e I believe all bitcoin L2s I've seen so far as even federated side-chains have timelocks for emergency paths) or even in the hypothetical worst-case miners’s `COINBASE_MATURITY.`

The reasoning in ***TxWithhold*** have come from surveying the academic literature at the time on the ill scientifically-defined idea of MEVs in the Ethereum space, before it was actually cool to do so, and in some ways transposing the analysis for the specificities of the Bitcoin space, to see what was similar and what was not.

As far as I can tell, though "prove me wrong" very welcome, OP_CHECKTEMPLATEVERIFY
alone do not facilitate or extend ***TxWithhold*** attacks against time-sensitive protocols, or even major components of the Bitcoin network as block mining and its corresponding rewards. I think this is not true, as soon as you combine it with other primitives, such as OP_CSFS.

Of course, one can point out that one can do evil ***TxWithhold*** construction today with *Bitvm* or *ColliderScript* style of techniques. Though as far as I understand the former assume significant witness size in terms of feerate cost and the latter assume non-existent hardware designs and significant energy ressources. So I reasonably believe they shouldn't be that much a concern for the covenant toolchain built on top of either bitvm or colliderscript to be used for ***TxWithhold*** attacks.

If ***TxWithhold***-style of attacks are plausible, in the meantime it's questionnable to go for bitcoin script primitives extension, at the very least without thinking more if mitigations for lightning or bitcoin mining shouldn't be devised or deployed, *first*. What if tomorrow if one can design a OP_CSFS-powered ***TxWithhold*** on-chain contract to make bitcoin script native trust-minimized bribe payouts to the majority of miners to prevent the commitment tx of its chan counterparties to confirm…

If this thread is only to talk about the merits of OP_CTV, IMHO this is a far lesser set of technical issues and long-term risks to be concerned of.

Personally, I'm an investor in bitcoin the hard currency, though only a reasonable one and with limited exposure as it's a novel class of asset with historically limited track record (-- this is not financial advice). Unforeseen MEVs due to ill-designed script primitives could start to irrevocably mess up bitcoin the network of miners and their corresponding incentives until leading to a slow decline. As we can observe in other cryptocurrencies playing the casino slot machine with their consensus changes. If that happens, that’s fine I'll go back to my pre-bitcoin career, i.e having fun with kernel security.

-------------------------

