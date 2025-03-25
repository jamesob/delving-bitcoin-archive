# Is there a social approach to the quantum problem?

josh | 2025-03-24 23:47:15 UTC | #1

I saw [this tweet](https://x.com/lynaldencontact/status/1904262504638783832?s=46&t=5QZh3FLO-zdzeZNNX9xX0w) suggesting “quantum” was the greatest threat to Bitcoin, from an investment risk perspective.

While Bitcoin generates no cash flows, it is valued like an investment, whether we like it or not. And investments are discounted by the perceived riskiness of the investment.

Therefore, it is reasonable to conclude that market participants withhold capital allocation to Bitcoin on the basis of this perceived risk. This depresses Bitcoin’s price, not in USD terms but in terms of its real purchasing power.

Now, I am not someone who believes quantum is a threat to Bitcoin. I understand that hashed public keys are quantum safe and that Bitcoin is a social experiment. Not a technical one.

But do others? I think not. Most approach “Bitcoin” no different from “blockchain.” All they see is the tech.

So I ask, should the community present a social solution, to the quantum problem?

Perhaps there is no immediate need for a quantum soft fork. People don’t need that. What they need is certainty their coins will be safe. If the worst happens and a breakthrough out of left field happens tomorrow.

How might the community proceed, if that happened?

One way would be to give up, and start again from a quantum-resistant beginning.

But why throw away the existing ledger? Why not publicly release a transition plan, that any user can opt into?

This brings me to my point. Why not start a movement around signing quantum-resistant public keys?

Why not ask users to provide proof of ownership?

No one has to, and there is no enforcement mechanism, but if there existed a large repository of timestamped public key signatures, using something like [OpenTimestamps](https://opentimestamps.org) and [BIP322](https://github.com/bitcoin/bips/blob/02ad0e01c2a9189124e05a52afe97ef90a3b7f1f/bip-0322.mediawiki) proofs of ownership, it would be trivial to reconstruct the ledger, if the worst happened.*

Is this needed? Technically, probably not. Owners of UTXOs that do not expose the public key can probably safely move to a quantum-proof UTXO if they need to, by sending their coin through a private miner who has a legal obligation to not try to steal their coin.

But the point isn’t to fix the quantum problem. It’s to reassure users, from the smallest village to the biggest bank, that Bitcoin is safe to seriously adopt.

Has anyone pursued this approach before? Would there be interest / support from the community?

*the proofs themselves wouldn’t even need to be published, unless a quantum breakthrough occurs

-------------------------

