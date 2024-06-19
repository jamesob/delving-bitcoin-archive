# DoS Disclosure: LND Onion Bomb

morehouse | 2024-06-18 17:48:35 UTC | #1

LND versions prior to 0.17.0 are vulnerable to a DoS attack where malicious onion packets cause the node to instantly run out of memory (OOM) and crash. If you are running an LND release older than this, your funds are at risk! Update to at least [0.17.0](https://github.com/lightningnetwork/lnd/releases/tag/v0.17.0-beta) to protect your node.

## The Vulnerability

When decoding onion payloads, LND did not do proper bounds checking on the decoded length of the payload.  The sender of the onion payload could set a length of up to 4 GB, and LND would allocate that much memory to decode the rest of the payload.

By sending multiple malicious onion payloads, an attacker could cause LND nodes to OOM crash within seconds.  Due to onion routing, the source of the attack would be trivially concealed.

## Protecting Your Node

Update to at least LND [0.17.0](https://github.com/lightningnetwork/lnd/releases/tag/v0.17.0-beta), which added a bounds check on onion payload lengths.

## More Details

For more details about the vulnerability, root causes, and prevention, see my [blog post](https://morehouse.github.io/lightning/lnd-onion-bomb/).

-------------------------

ariard | 2024-06-18 23:27:16 UTC | #2

I don’t know if the vulnerability you’re describing can be exploited in real-world conditions against LND nodes pre 0.17.0 or post 0.17.0

The lightning specification (BOLT8) is already setting the maximum lightning message size at 65535 bytes (cf. https://github.com/lightning/bolts/blob/c562d91ace0e95bec3c6f8758969eaf3627f23c8/08-transport.md#lightning-message-specification). LND Onion Bomb (onion paylaod >= 4 GB) have to be carried on to the LND node either through a `update_add_htlc` (BOLT2) or `onion_message` (BOLT4). All those messages are encrypted under Noise_XK encrypted and authenticated transport.

The protocol does not support messages fragmented over multiple transport frames so far. I don’t know if the fuzz target has been written directly as a half-peer lightning connection stack.

-------------------------

roasbeef | 2024-06-18 23:27:07 UTC | #3

@ariard the issue was that a buffer was _preallocated_ based on the encoded length. So it's not that they can send a message larger than the max size (protocol prevents that at the wire level), instead eager preallocation caused lnd to allocate a large amount of memory. 

A `BigSize` varint is used in the wire encoding, so the length prefix can be a value larger than `65535`.

-------------------------

ariard | 2024-06-18 23:38:56 UTC | #4

@roasbeef I see, reasons rust-lightning has fixed-size allocation buffers throughout the codebase. BOLT8 has already a recommendation in that sense about basing node memory management on max input size.

-------------------------

