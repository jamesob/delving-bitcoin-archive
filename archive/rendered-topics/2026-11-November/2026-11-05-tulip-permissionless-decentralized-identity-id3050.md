# TULIP: Permissionless Decentralized Identity

tidwell | 2026-11-05 16:07:21 UTC | #1

I believe the world would benefit from an identity system that isn't centrally controlled. Nostr, afaik, is one of the few systems that has leaned into pseudonymous identities built on secp256k1 keys and Schnorr (BIP 340) signatures, which sign every event a user publishes. However, key rotation, the hard part, has never been solved in a widely adopted way because of the complexity involved. I don't find an identity system with central coordinators deciding what "truth" is interesting, because they can omit data and users aren't actually in control of their identities.

I've been ideating on an identity system that uses the Bitcoin blockchain as a timestamping and ordering authority for identity key state changes. I believe there is an interesting tradeoff between everything being offchain to a small onchain footprint allowing full reproducibility of state changes. This pseudonymous identity system proposal doesn't rely on any data besides the blockchain itself to know the most current status of a signing key. The scaling aspect to this is the assumption that most participants will never need to actually make an onchain transactions. When a key is compromised and that identity is worth saving, then an onchain native taproot transaction is required to rotate. An identity rotates via a precomitted "taproot chain". See the picture below.

![Screenshot 2026-11-05 at 9.57.54 AM|690x175](upload://h8T2abSYGMXHNsOXHHuBMUzUcgg.jpeg)

Here we are showing multiple rotations using a script-spend for the same identity. These are rotating to tweaked taproot outputs (serving as the new signing key and identity policy). The internal key is tweaked and used as the new signing key.

These special taproot addresses embeded with a minimum set protocol spend paths I call a "tulip".

This means you'd need to precommit to all rotations and those keys ahead of time when creating an identity. If/when you want to rotate, you'd bring the next key from cold storage online, sign and broadcast the rotation transaction, and now the identity's everyday signing key has changed. A recovery path, which should have a separate set of conditions, starts a new Taproot chain for an identity. An attacker who can satisfy your recovery script and has a descriptor of your current tulip could take over your identity (your main recourse at this point is WoT ideas and revocation of the previous identity that was stolen).

The interpretation of the value of an identity using this system is up to the service/application consumers that want to use it. Some services may find certain attributes more effective than others when filtering or giving access to pseudonymous requests. Examples Think: (testnet faucet amounts, account creation for some game, opening a git issue or PR, rate limits for some LLM thing, or putting reputation behind some media content persauding others it isn't fake).

I have a whitepaper link here:

https://tuliproot.com/

The goal would be eventual WoT applications and consumers being able to use the tuliproot protocol to help with verification and auth. Consumers of this idea would either need to run a node with some additional software to scan/index the bitcoin blockchain or would need to use a service. I'm proposing a 4 byte witness header be used to help people index on transactions of interest (i.e. when an important state change happens).

When interacting with a consumer, if there is no onchain footprint, identities can reveal off-chain proofs to backdate an identities inception. Think OTS, however in the paper I'm recommending KZG commitments since they are a constant size (48 byte commitment) and there is a chance a user would want to publish this backdated proof onchain.

I would be very interested in feedback.

-------------------------

