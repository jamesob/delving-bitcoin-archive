# Hashing to remember forgotten seed words?

zawy | 2025-04-13 10:31:56 UTC | #1

Is it safe to use a miner to reduce the number of seed words you have to remember?  I want to use a miner because we known an attacker can't hash 10x more efficiently than we can at home.  

You would make half your nonce equal to your 12 seed words (132 bits). The other half of the nonce would be a random seed (124 bits) that you make very public so that you can't forget it.  "I can't forget" = "everyone knows". You would set the difficulty (number of hashes required) equal to the "entropy" of the seed words you might forget. If you want to be able to forget 6, you have to hash 2^66 times, requiring 8 days on a 100 TH/s miner at a cost of about $50. The target would be (2^256-1)/2^66.

To recover the seed words, you let the bits that are forgotten to be what you change in the nonce until a solution is found. There's something like a 50% chance the first solution won't be the correct once (the correct seed words).  

An attacker doesn't know any of the 12 words, so his search space is 2^132 instead of 2^66, so his cost $50 for each of the 2^65 wrong solutions. 

Right?  Is it safe?

-------------------------

