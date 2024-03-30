# Payjoin-in-Potentiam: Externally fund an LSP channel open with one transaction

bitgould | 2024-03-29 20:57:53 UTC | #1

# Payjoin-in-Potentiam: Externally fund an LSP channel open with one transaction

As stated by Jesse Posner and ZmnSCPxj, "moving funds from an onchain-only address to Lightning Network is slow, especially if you desire trust-minimization," opening their [Swap-in-Potentiam proposal](https://lists.linuxfoundation.org/pipermail/lightning-dev/2023-January/003810.html).

That proposal involves a contract between two parties, Alice (the funds owner) and Bob (a potential swap partner), with the contract having two branches: one for onchain/channel transactions and another with a time lock for Alice. The protocol requires cooperation from a Lightning Service Provider (LSP) and is particularly useful for mobile wallet users who want to efficiently transfer funds from the blockchain layer to the Lightning layer without waiting for additional confirmations. For a detailed explanation and sequence, please refer to the original discussion on the Lightning-dev mailing list​​.

Swap-in-potentiam execution requiries two transactions: 

1. Funds a swap address
2. Opens the lightning channel from funds in that address

Payjoin interaction enables the external funds to directly fund a channel opening from the LSP without a swap output address ever being posted on chain. Instead, the first transaction funding a swap address is replaced using payjoin output substitution so only the combined result is ever posted to the blockchain.

## Payjoin in Potentiam

The swap-in-potentiam proposal requires a commitment transaction to the swap-in-potentiam LSP address, after which the LSP has unilateral ability to open the channel or the funder can claw back after a timeout. If instead of broadcasting a transaction to such an address a funder first sends the transaction as a PSBT to the LSP, the LSP replaces the swap-in-potentiam address in the original PSBT for a channel opening address in a payjoin PSBT via [payment output substitution](https://github.com/bitcoin/bips/blob/b3701faef2bdb98a0d7ace4eedbeefa2da4c89ed/bip-0078.mediawiki#user-content-span_idoutputsubstitutionspanPayment_output_substitution)* to actualize the swap-in in a single transaction. The payjoin PSBT containing the funder input and the channel open output is returned to the funder, who then signs and broadcasts the transaction.

In the case the transaction is not sign and broadcast, the LSP can still broadcast the original PSBT's transaction with the swap-in-potentiam address output and proceed in that protocol to open a channel, or otherwise Alice could claw back funds after a timeout. This shares approximately the same flow with an earlier [Lightning payjoin idea](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-November/021180.html) but has an LSP open the channel rather than an edge node itself.

\* BIP 78 payjoin output substitution recommends a match between input and output script types, but I see this as an undesirable fingerprint that should be avoided and have campaigned for this change in the spec.

## Swap in Potentiam Sequence Diagram

<pre>
Alice (Funds Owner)          Bob (Swap Partner/LSP)             Blockchain
       |                              |                              |
       | --- 1. Address Request ----> |                              |
       |                              |                              |
       | <--- 2. Swap Address ------- |                              |
       |                              |                              |
       |                              |                              |
       | --- 3. Tx to swap Address----+--- 4. Confirm Transaction -> |
       |                              |                              |
       |                              | <----------- 5. Confirmation |
       |                              |                              |
       | --- 6. Request Swap -------> |                              |
       |                              |                              |
       | <--- 7. Validate & Accept -- |                              |
       |                              |                              |
       |                              | --- 8. Create & Sign Tx ---- |
       |                              |                              |
       |                              | <----------- 9. Confirmation |
       |                              |                              |
</pre>

1. Alice requests a unique swap-in-poetentiam address from Bob to send her funds to.
2. Bob provides a specially constructed address that commits to a contract between Alice and Bob.
3. Alice sends the funds to the provided address on the blockchain.
4. Bob confirms the transaction is broadcast to the blockchain.
5. The blockchain confirms the transaction.
6. Once the transaction is confirmed, Alice requests a swap to move the funds to the Lightning Network.
7. Bob validates the request and agrees to the swap if the funds are confirmed.
8. Bob creates and signs a transaction according to the swap-in-potentiam contract.
9. The transaction is confirmed by the blockchain, completing the swap.

## Payjoin in Potentiam Sequence Diagram

<pre>
Alice (Sender)          Bob (Receiver/LSP)          Blockchain
     |                          |                       |
     | -- 1. Request Address -->|                       |
     |                          |                       |
     |<-- 2. Swap Address ------|                       |
     |                          |                       |
     | -- 3. Original PSBT ---->|                       |
     |                          |                       |
     |<--- 4. Output Substitution payjoin PSBT          |
     |  (Swap Address to Lightning Channel Address)     |
     |                          |                       |
     | -- 5. Signed payjoin to channel ---------------->|
     |                          |                       |
     |                          |<- 6. Confirm Channel -|
     |                          |                       |
</pre>

1. Alice requests a swap-in-potentiam address from Bob.
2. Bob provides a swap-in-potentiam address to Alice.
3. Alice constructs a Partially Signed Bitcoin Transaction (PSBT) paying to the provided address and sends the PSBT to Bob.
4. Bob performs output substitution, replacing the swap-in-potentiam address with a Lightning channel address in the payjoin PSBT.
5. Alice checks the substituted output to ensure it's acceptable signs and broadcasts the payjoin.
6. The blockchain confirms the transaction, effectively opening the Lightning channel between Alice and Bob.

In the optimistic case that both participants are able to communicate in a timely fashion, only one transaction is broadcast to the same effect as swap-in-potentiam. If not, there is a safe fallback.

Swap-in-potentiam already assumes a cooperating LSP. In the case they're not cooperative, funds are locked for some time. If the sender goes offline or otherwise does not broadcast the payjoin the original PSBT can be broadcast to open a channel via swap-in-potentiam.

-------------------------

doglegs | 2024-03-29 22:53:14 UTC | #2

This is a good solution to move funds from the blockchain layer to the Lightning Network layer while maintaining trust-minimization.

You could consider introducing a trustless intermediary to facilitate the output substitution process and mediate disputes between the sender and the Lightning Service Provider, reducing the reliance on a single trusted party.

-------------------------

ZmnSCPxj | 2024-03-30 00:05:02 UTC | #3

Some clarifications:

* There is no need for steps 1 and 2.  Alice knows a public key controlled by Bob, namely, the node ID of Bob.  Alice can generate its own keypair.  The Alice pubkey is expected to be made by some form of BIP32 (HD derivation).  The reason for not asking for a swap address is that it allows Alice to implement a watch-only onchain wallet using only BIP32 HD derivation.
* Not announced yet, but the current swap-in-pointentiam detailed draft spec already uses channel opening flow instead of an onchain-to-offchain swap: https://github.com/ZmnSCPxj-jr/swap-in-potentiam/blob/cf7b11476ee5ad7a85edb5482d05f17ff625feb4/doc/swap-in-potentiam.md#opening-0-conf-lightning-channels-backed-by-swap-in-potentiam This spec is based on [LSPS0](https://github.com/BitcoinAndLightningLayerSpecs/lsp/blob/68e498ebc278dcd4ab6188d12e93ea2e91c5d882/LSPS0/README.md), but is an extension that is not yet in discussion with the LSPS standards group.
  * IMPORTANT: all inputs to the funding transaction MUST be those with the same LSP-as-Bob.  The Alice keys can differ.  For this reason, PSBT is specifically not used, instead this sub-protocol sends the inputs and the order of the outputs in a bespoke format.  This also allows this sub-protocol to use MuSig2 path as current PSBT has no MuSig2 support yet.
  * For general onchain-to-onchain spends, PSBT is used.  This allows swap-in-potentiam wallets to implement PayJoin, which uses PSBT.  The hope is that at some point in the future we can get MuSig2 support on PSBT, but the explicit 2-of-2 tapleaf path still exists so that we can uses no-MuSig2 PSBTs as they exist today.
* The whole point of swap-in-potentiam is that it is an onchain address, with onchain address rules (including miniimum confirmation depth), that can be magically spent in Lightning (*after* minimum confirmation depth).  The inputs to the transaction should be swap-in-potentiam addresses, not outputs.
  * The use-case is that the client generates its swap-in-potentiam address, then sends the address to somebody else (e.g. post it in the footer of their website, or in their platform-formerly-known-as-Twitter bio, or in an email with somebody, or in their forum sig), then the client goes to sleep.  Then somebody else funds the swap-in-potentiam.  When the client wakes up, hopefully the received funds are already confirmed to minimum confirmation depth. then the client can spend the funds to Lightning immediately, ***without*** incurring an additional wait to move from their onchain to offchain.
  * i.e. ***somebody else*** (*NOT* Alice!) funds the swap-in-potentiam address.  That somebody else might not even know who the Bob is.

The secret sauce of swap-in-potentiam addresses  is that sending an ordinary onchain transaction to it is actually a (non-Lightning!) channel open, specifically a modified `OP_CLTV`/`OP_CSV` Spilman channel (`OP_CSV` in this case).  This is unlike LN channel opens where the participants MUST participate during setup to exchange initial signatures --- the channel open can be initiated by either party without participation of the other party, or even by a third party making what it thinks is an ordinary onchain-to-onchain transaction.

-------------------------

