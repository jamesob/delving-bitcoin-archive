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

