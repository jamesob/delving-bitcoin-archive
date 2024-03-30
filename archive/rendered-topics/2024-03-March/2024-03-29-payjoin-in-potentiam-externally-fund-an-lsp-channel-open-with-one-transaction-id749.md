# Payjoin-in-Potentiam: Externally fund an LSP channel open with one transaction

bitgould | 2024-03-30 17:16:39 UTC | #1

# Payjoin-in-Potentiam: Externally fund an LSP channel open with one transaction

As stated by Jesse Posner and ZmnSCPxj, "moving funds from an onchain-only address to Lightning Network is slow, especially if you desire trust-minimization," opening their [Swap-in-Potentiam proposal](https://lists.linuxfoundation.org/pipermail/lightning-dev/2023-January/003810.html).

That proposal involves a contract between two parties, Alice (the funds owner) and Bob (a potential swap partner), with the contract having two branches: one for onchain/channel transactions and another with a time lock for Alice. The protocol requires cooperation from a Lightning Service Provider (LSP) and is particularly useful for mobile wallet users who want to efficiently transfer funds from the blockchain layer to the Lightning layer without waiting for additional confirmations. For a detailed explanation and sequence, please refer to the original discussion on the Lightning-dev mailing list​​.

Swap-in-potentiam execution requires two transactions: 

1. Funds a swap address
2. Opens the lightning channel from funds in that address

Payjoin interaction enables the external funds to directly fund a channel opening from the LSP without a swap output address ever being posted on chain. Instead, the first transaction funding a swap address is replaced using payjoin output substitution so only the combined result is ever posted to the blockchain.

## Payjoin in Potentiam

The swap-in-potentiam proposal requires a commitment transaction to the swap-in-potentiam LSP address, after which the LSP has unilateral ability to open the channel or the funder can claw back after a timeout. If instead of broadcasting a transaction to such an address a funder first sends the transaction as a PSBT to the LSP, the LSP replaces the swap-in-potentiam address in the original PSBT for a channel opening address in a payjoin PSBT via [payment output substitution](https://github.com/bitcoin/bips/blob/b3701faef2bdb98a0d7ace4eedbeefa2da4c89ed/bip-0078.mediawiki#user-content-span_idoutputsubstitutionspanPayment_output_substitution)* to actualize the swap-in in a single transaction. The payjoin PSBT containing the funder input and the channel open output is returned to the funder, who then signs and broadcasts the transaction.

In the case the transaction is not sign and broadcast, the LSP can still broadcast the original PSBT's transaction with the swap-in-potentiam address output and proceed in that protocol to open a channel, or otherwise Alice could claw back funds after a timeout. This shares approximately the same flow with an earlier [Lightning payjoin idea](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-November/021180.html) but has an LSP open the channel rather than an edge node itself.

\* BIP 78 payjoin output substitution recommends a match between input and output script types, but I see this as an undesirable fingerprint that should be avoided and have campaigned for this change in the spec.

## Swap-in-Potentiam Sequence Diagram

![image|690x375](upload://bbkAqu3S4NTCVCiwvi0tYWjCRN0.png)

[details="Swap-in-Potentiam Diagram ASCII"]
<pre>
Source of Funds           Alice (Node)              Bob (LSP)      Blockchain
|                             |                         |                   |
|                        (Swap Address derived from Bob's Pubkey)           |
|                             |                         |                   |
| <-- 1. Swap Address ------- |                         |                   |
|                             |                         |                   |
| --- 2. Tx to Swap Address-----------------------------------------------> |
|                             |                         |                   |
|                             |                         |  3. Confirmation  |
|                             |                         |                   |
|                             | <- 4. Negotiate Open -> |                   |
|                             |                         |                   |
|                             |                 5. Sign and Broadcast Tx -> |
|                             |                         |                   |
|                             |                         |  6. Confirmation  |
|                             |                         |                   |</pre>
[/details]


1. Alice sends a swap-in-potentiam address derived from Alice and Bob keys to source of funds.
2. Bob funds the Swap Address.
3. The transaction funding the Swap Address is confirmed.
4. Alice and Bob derive a lightning channel open address.
5. Bob signs and broadcast the channel opening tx spending the swap address.
6. The transaction is confirmed by the blockchain, completing the channel open between Alice and Bob

## Payjoin-in-Potentiam Sequence Diagram

![image|690x334](upload://pbpjE83zOrvufWgDzFkw6TCgFIb.png)


[details="Payjoin-in-Potentiam Diagram ASCII"]
<pre>
Source of Funds         Alice (Node)                         Bob (LSP)             Blockchain
|                               |                                |                          |
|                             (Swap Address derived from Bob's Pubkey)                      |
|                               |                                |                          |
| <-- 1. Swap Address URI------ |                                |                          |
|                               |                                |                          |
| --- 2. Original PSBT ----------------------------------------> |                          |
|                               |                                |                          |
|                               | <---- 3. Negotiate Open -----> |                          |
|                               |                                |                          |
| <------------------------------------ 4. Output Substitution payjoin PSBT                 |
|                                         (Swap Address to Lightning Channel Address)       |
|                               |                                |                          |
| --- 5. Signed payjoin to channel -------------------------------------------------------> |
|                               |                                |                          |
|                               |                                |      6. Confirmation     |
|                               |                                |                          |
</pre>
[/details]


1. Alice sends a swap-in-potentiam address derived from Alice and Bob keys to source of funds in a bip21 payjoin URI.
2. The Source funds the swap-in-potentiam address in a finalized original PSBT which he sends to Bob.
3. Alice and Bob derive a lightning channel open address.
4. Bob performs output substitution, replacing the swap-in-potentiam address with a Lightning channel address in the PSBT and sending the result to the Source.
5. The Source of Funds checks that the payjoin PSBT doesn't overspend its inputs, signs, and broadcasts the payjoin.
6. The transaction is confirmed by the blockchain, completing the channel open between Alice and Bob.

In the optimistic case that both nodes are able to communicate in a timely fashion, only one transaction is broadcast to the same effect as swap-in-potentiam. If not, there is a safe fallback.

Swap-in-potentiam already assumes a cooperating LSP. In the case they're not cooperative, funds are locked for some time. If the sender goes offline or otherwise does not broadcast the payjoin the original PSBT can be broadcast to open a channel via swap-in-potentiam.

-------------------------

doglegs | 2024-03-29 22:53:14 UTC | #2

This is a good solution to move funds from the blockchain layer to the Lightning Network layer while maintaining trust-minimization.

You could consider introducing a trustless intermediary to facilitate the output substitution process and mediate disputes between the sender and the Lightning Service Provider, reducing the reliance on a single trusted party.

-------------------------

ZmnSCPxj | 2024-03-30 00:06:01 UTC | #3

Some clarifications:

* There is no need for steps 1 and 2.  Alice knows a public key controlled by Bob, namely, the node ID of Bob.  Alice can generate its own keypair.  The Alice pubkey is expected to be made by some form of BIP32 (HD derivation).  The reason for not asking for a swap address is that it allows Alice to implement a watch-only onchain wallet using only BIP32 HD derivation.
* Not announced yet, but the current swap-in-pointentiam detailed draft spec already uses channel opening flow instead of an onchain-to-offchain swap: https://github.com/ZmnSCPxj-jr/swap-in-potentiam/blob/cf7b11476ee5ad7a85edb5482d05f17ff625feb4/doc/swap-in-potentiam.md#opening-0-conf-lightning-channels-backed-by-swap-in-potentiam This spec is based on [LSPS0](https://github.com/BitcoinAndLightningLayerSpecs/lsp/blob/68e498ebc278dcd4ab6188d12e93ea2e91c5d882/LSPS0/README.md), but is an extension that is not yet in discussion with the LSPS standards group.
  * IMPORTANT: all inputs to the funding transaction MUST be those with the same LSP-as-Bob.  The Alice keys can differ.  For this reason, PSBT is specifically not used, instead this sub-protocol sends the inputs and the order of the outputs in a bespoke format.  This also allows this sub-protocol to use MuSig2 path as current PSBT has no MuSig2 support yet.
  * For general onchain-to-onchain spends, PSBT is used.  This allows swap-in-potentiam wallets to implement PayJoin, which uses PSBT.  The hope is that at some point in the future we can get MuSig2 support on PSBT, but the explicit 2-of-2 tapleaf path still exists so that we can uses no-MuSig2 PSBTs as they exist today.
* The whole point of swap-in-potentiam is that it is an onchain address, with onchain address rules (including miniimum confirmation depth), that can be magically spent in Lightning (*after* minimum confirmation depth).  The inputs to the transaction should be swap-in-potentiam addresses, not outputs.
  * The use-case is that the client generates its swap-in-potentiam address, then sends the address to somebody else (e.g. post it in the footer of their website, or in their platform-formerly-known-as-Twitter bio, or in an email with somebody, or in their forum sig), then the client goes to sleep.  Then somebody else funds the swap-in-potentiam.  When the client wakes up, hopefully the received funds are already confirmed to minimum confirmation depth. then the client can spend the funds to Lightning immediately, ***without*** incurring an additional wait to move from their onchain to offchain.
  * i.e. ***somebody else*** (*NOT* Alice!) funds the swap-in-potentiam address.  That somebody else might not even know who the Bob is.

The secret sauce of swap-in-potentiam addresses  is that sending an ordinary onchain transaction to it is actually a (non-Lightning!) channel open, specifically a modified `OP_CLTV`/`OP_CSV` Spilman channel (`OP_CSV` in this case).  This is unlike LN channel opens where the participants MUST participate during setup to exchange initial signatures --- the channel open can be initiated by either party without participation of the other party, or even by a third party making what it thinks is an ordinary onchain-to-onchain transaction.  Onchain confirmation is still required --- but it is between the third-party and Alice, no **additional** waiting is needed between Alice and Bob.

-------------------------

bitgould | 2024-03-30 17:02:46 UTC | #4

Thanks very much for the prompt response and corrections. I understand this nuance and see how the steps I've described failed to describe it accurately. It is confusing to have combined Alice's node and the Source of Funds in the flows. I've edited the original post to make a distinction.

## The problem payjoin-in-potentiam solves

> The whole point of swap-in-potentiam is that it is an onchain address, with onchain address rules (including miniimum confirmation depth), that can be magically spent in Lightning (after minimum confirmation depth). The inputs to the transaction should be swap-in-potentiam addresses, not outputs.

> i.e. somebody else (NOT Alice!) funds the swap-in-potentiam address. That somebody else might not even know who the Bob is.

Swap-in-potentiam always posts two transactions. Funds are be confirmed into a swap address before a channel is opened. Payjoin-in-potentiam adds an optimistic case to the protocol. Alice generates a payjoin bip21 URI containing a swap-in-potentiam address, tells the source of funds about it and the source of funds starts a payjoin sending to the swap address with Bob. *If alice is found NOT to be asleep* before the payjoin protocol expires, Bob takes advantage of that and combines a channel open with proposed external input from the source of funds in a single transaction rather than confirming funds into a swap-in contract first and then following that confirmation with a channel opening transaction. If alice is asleep for the payjoin protocol (e.g. after 1 minute payjoin expiration), Bob broadcasts the original PSBT transaction funding the swap-in-potentiam address and can open a channel with the swap address funding after the fact. This cuts the on-chain footprint of swap-in-potentiam channel opens by about half in the best case and maintains the magic of swap-in-potentiam.

## Fetching the public key from the LSP

> There is no need for steps 1 and 2. Alice knows a public key controlled by Bob, namely, the node ID of Bob. Alice can generate its own keypair. The Alice pubkey is expected to be made by some form of BIP32 (HD derivation). The reason for not asking for a swap address is that it allows Alice to implement a watch-only onchain wallet using only BIP32 HD derivation.

At some point Alice does need to fetch Bob's public key after which it can be derived. Neat. I've changed the post to reflect.

## MuSig2

> all inputs to the funding transaction MUST be those with the same LSP-as-Bob. The Alice keys can differ. For this reason, PSBT is specifically not used, instead this sub-protocol sends the inputs and the order of the outputs in a bespoke format. This also allows this sub-protocol to use MuSig2 path as current PSBT has no MuSig2 support yet.

Payjoin-in-potentiam removes this hard input requirement in the case that both Alice comes online before the payjoin window expires by using interaction. If Alice comes online before the payjoin expires, Alice can MuSig2 with Bob to create the channel output and create a transaction with external funds as input in a single transaction before funds land in a swap address. If the payjoin fails, funds still end up in the swap address and the swap-in-potentiam protocol is executed.

If I were designing a bespoke MuSig2 protocol, I'd still consider PSBT instead of something totally bespoke so that once the MuSig2 support is added to PSBT, software supporting PSBT already can quickly support swap-in-potentiam. Have you considered using the [MuSig2 PSBT extension draft by Sanket](https://gist.github.com/sanket1729/4b525c6049f4d9e034d27368c49f28a6) and [deployed by BitGo](https://bitcoinops.org/en/bitgo-musig2/) in your bespoke protocol?

I'm sure you've already thought deeply about how the bespoke sub-protocol operates, so please excuse my suggestions as naive if you've already given them thought. I know how hard protocol design can be and have only recently explored swap-in-potentiam as an avenue to make lightning onboarding cheaper and easier.

-------------------------

