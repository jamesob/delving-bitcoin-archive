# Payjoin-in-Potentiam: Externally fund an LSP channel open with one transaction

bitgould | 2024-03-29 20:54:21 UTC | #1

# Payjoin-in-Potentiam: Externally fund an LSP channel open with one transaction

As stated by Jesse Posner and ZmnSCPxj, "moving funds from an onchain-only address to Lightning Network is slow, especially if you desire trust-minimization," opening their [Swap-in-Potentiam proposal](https://lists.linuxfoundation.org/pipermail/lightning-dev/2023-January/003810.html).

That proposal involves a contract between two parties, Alice (the funds owner) and Bob (a potential swap partner), with the contract having two branches: one for onchain/channel transactions and another with a time lock for Alice. The protocol requires cooperation from a Lightning Service Provider (LSP) and is particularly useful for mobile wallet users who want to efficiently transfer funds from the blockchain layer to the Lightning layer without waiting for additional confirmations. For a detailed explanation and sequence, please refer to the original discussion on the Lightning-dev mailing list​​.

Swap-in-potentiam execution requiries two transactions: 

1. Funds a swap address
2. Opens the lightning channel from funds in that address

Payjoin interaction and enables the external funds to directly fund a channel opening from the LSP without a swap output address ever being posted on chain. Instead, the first transaction funding a swap address is replaced using payjoin output substitution so only the combined result is ever posted to the blockchain.

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

