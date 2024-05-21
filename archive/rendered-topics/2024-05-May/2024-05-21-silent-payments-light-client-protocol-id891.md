# Silent Payments: Light Client Protocol

setavenger | 2024-05-21 09:15:50 UTC | #1

During the development of a couple light clients I jotted down some notes and created a very early draft for a [Light Client Specification](https://github.com/setavenger/BIP0352-light-client-specification). The appendix of the SP BIP was used as a basis for this. The specification was designed with two goals in mind. (1) Reduce the computational burden for light clients, and (2) Minimise the bandwidth requirements. Both goals have to be achieved without compromising privacy for the user. The protocol is designed such that a light client can connect to any public indexing server without giving out more information than "I'm interested in this block".

There have been a couple short discussions around this Light Client Protocol. Now that an early draft exists I would like to get more people involved in the process. The draft can be found here: https://github.com/setavenger/BIP0352-light-client-specification

The computational burden is reduced by generating a tweak index. To further reduce the required computations (and also bandwidth) we can apply cut-through to the transactions. So pruning tweaks for transactions where all taproot UTXOs are spent. Bandwidth is reduced through various measures which mainly revolve around only providing the data necessary for a light client to find and spend a UTXO.

The basic flow for a client to receive is as follows:

Per block:

1. Fetch the tweaks (possibly filtered for dust limit)
2. Compute the possible pubKeys for n = 0
3. Fetch taproot-only filter (BIP 158)
4. Compare the pubKeys against a taproot-only filter
    - If no match: go to 1. with block_height + 1
    - Else: continue with 5.
5. Fetch simplified UTXOs
6. Scan according to the BIP (one could reuse pubKeys from 2. here)
7. Collect all matched UTXOs and add to wallet
8. Go to 1. with block_height + 1

As mentioned in the specification a scriptPubKey that has already received funds should be tracked as well. The reasoning can be found in the specification.

-------------------------

