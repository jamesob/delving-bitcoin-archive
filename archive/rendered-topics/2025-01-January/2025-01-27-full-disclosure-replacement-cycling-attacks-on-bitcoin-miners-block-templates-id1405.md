# Full Disclosure: Replacement Cycling Attacks on Bitcoin Miners Block Templates

ariard | 2025-01-27 15:38:22 UTC | #1

Today is fully disclosed the variant of replacement cycling attacks enabling to censor transaction traffic from miners block template and as such for an attacker to achieve a dominant strategy in the distribution of the bitcoin fee reward for the generation of blocks in the longuest valid chain among all honest mining nodes.

A proof-of-concept of this miner-level attack was tested against a Bitcoin Core 26.0 branch before my previous disclosure of the 16 October 2023 informing the community on how the replace-by-fee mechanism could be used to break the security of the Lightning channels. During the last months, it was independently re-discovered and noticed by at least Peter Todd (and I guess few other smart observers...) that replacement cycling attacks can affect miners block templates.

Since then, the attack variants have been successfully re-tested on both the “classic” mempool (bitcoin core branch 26.0) and the cluster mempool (bitcoin core branch 29.0). The full disclosure has been shared to the bitcoin dev mailing list and there is a copy available here: https://ariard.github.io/minerjamming.html

For an in-depth discussion of this novel attack vector on Bitcoin miners, there is paper available here: https://github.com/ariard/mempool-research/blob/6de7d8bb15f28b7444328335231e6b6d33f2a0af/rca-bmbt.pdf

-------------------------

