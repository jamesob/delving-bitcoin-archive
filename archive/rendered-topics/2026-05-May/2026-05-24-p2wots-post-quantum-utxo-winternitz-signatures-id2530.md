# P2WOTS: Post Quantum UTXO Winternitz Signatures

opus-lux | 2026-05-24 23:08:14 UTC | #1

Hello Bitcoin Community!

This is the revised proposal for post quantum bitcoin using lightweight Winternitz signatures.

I would like to thank Murch for correctly pointing out the flaws in my original proposal.

Read the revised proposal essay along with the source code at the following links below!

Proposal:  block_opuslux.ar.io/#/bitcoin-proposal

Code: https://github.com/opus-lux/bitcoin-proposal

WOTS-39 essay:  block_opuslux.ar.io/#/essay

Let's start by explaining what was wrong with my original proposal and why Winternitz + Lamport works on the EVM but not for Bitcoin! I originally developed WOTS-39 on the EVM which has a fundamentally different blockchain structure than Bitcoin. Since Ethereum based chains are account based you can easily and cheaply use the Lamport authorization chain to verify the upload of each Winternitz public key for each message signature. Ethereum is built for smart contract execution so tracking this data is trivial on the ethereum virtual machine. Bitcoin is fundamentally different than Ethereum. Bitcoin treats each UTXO as an individual object so your coins are not as easily tracked under a single account address, you query the chain to see which UTXO's you can actually spend to get your full Bitcoin balance. So we have to take a fundamentally different approach to solve Winternitz signatures on Bitcoin because we must solve it at the UTXO level instead of the account object level. The good thing is that the core discovery of WOTS-39 still works without the account object. The core insight of WOTS-39 is that by using the message TXID as a unique anchor you can reliably use Winternitz signatures without ever repeating a public key. So the new construction for my proposal introduces a new native bitcoin output type using witness version 2. Each time the user wants to receive post quantum bitcoin they generate a new receive address, they generate a fresh wots_pk which gets included in their receive address and allows the sender to set the scriptPubKey. The scriptPubKey controls who can spend the UTXO. Since P2WOTS is an additive new output type all existing taproot and multi-sig functionality remains intact. The lightning network continues to function as well. All of this is possible without a hard fork thanks to BIP-141. This proposal is for a single sender only, post quantum multi-sig is not possible with P2WOTS, but the existing Schnorr multi-sig will continue to function as is. I look forward to the feedback on this idea from the bitcoin community! Thanks for taking the time to consider this option!

Best, Opus Lux!

-------------------------

