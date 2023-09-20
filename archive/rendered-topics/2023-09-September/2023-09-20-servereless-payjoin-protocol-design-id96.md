# Servereless Payjoin Protocol Design

bitgould | 2023-09-20 16:51:34 UTC | #1

A draft BIP has been submitted for the design of a payjoin protocol that stands a better chance at adoption: https://github.com/bitcoin/bips/pull/1483. 

> This document proposes a backwards-compatible second version of the payjoin protocol described in BIP 78, allowing complete payjoin receiver functionality including payment output substitution without requiring them to host a secure public endpoint. This requirement is replaced with an untrusted third-party relay and streaming clients which communicate using an asynchronous protocol and authenticated, encrypted payloads.

The proceeding discussion has given rise to a number of considerations and changes that I'm keeping track of on Delving Bitcoin. 

## Proposed Design

### Metadata Security

Data security is paramount to achieving effective privacy through a widespread implementation. However, Tor and optional VPN support may undermine adoption and privacy respectively.

I have recently discovered Oblivious HTTP (OHTTP), i.e. one-hop onion routing, to separate IP addresses from requests. I have recruited independent server operators to run OHTTP proxies to support serverless payjoin. The network protocol needs to be carefully considered to prevent an OHTTP proxy from conducting timing attacks of its own. 

### Authenticated Encryption

Payjoin version 1 depends on third party public key infrastructure (TLS Certificate Authorities or Tor) to secure interactions. I propose payjoin v2 clients share ephemeral public keys to authenticate and encrypt payloads associated with particular payjoin requests peer-to-peer instead. OHTTP uses a simple Hybrid Public Key Encryption (HPKE) standard to this end which we can make use of inside the payjoin application as well.

However, standard HPKE does not support secp256k1. Since all bitcoin software must support this curve we ought to support secp256k1 HPKE. Since secp256k1 is secure for our purposes and all parties relying on HPKE agree on a cryptosystem without conferring with third parties, this should be feasible.

### Network Transport

We should use common network transport to be supported by as many clients as possible. By removing a TLS requirement such a protocol could even be supported by Bitcoin Core, since it would not introduce new upstream dependencies.

I have explored a number of transport mechanisms for serverless payjoin, going as far as PoC implementations for a number of the following:

- HTTP long polling
- STUN/TURN/ICE p2p NAT traversal
- WebRTC
- WebSockets / Nostr
- WebTransport
- Custom application transport on QUIC

Pointing at widespread support and ease of implementation leads me back to simple HTTP polling. Additionally, HTTP support allows IP metadata to be secured using Oblivious HTTP. Using plain HTTP has the additional benefit of seamless backwards compatibility.

### Backwards Compatibility

Version 2 payjoin relays may operate version 1 HTTPS ingress alongside v2 OHTTP to allow version 1 senders to attempt payjoin. Receiving version 1 payjoin over version 2 relay would require receivers to remain online, which may pose a UX challenge. Version 1 payloads are unencrypted and unauthenticated, so their senders would need to disable payjoin output substitution.

### PSBT Version 2

PSBT version 2 allows for simple multiparty transaction construction and mutation. New PSBT fields can also be introduced to carry payjoin parameters to forgo a dependency on an additional serialization format like JSON.


### BIP 21 Public Key Encoding

Since the latest proposed design shares two keys (OHTTP, Payjoin PK) in the BIP21 URI it is critical to be efficient in the way these are represented in text. base64 URL encoding and blockchain commons UR encoding are both viable, with UR encoding behaving a bit better when displayed in QR codes.

## What do you think?

I'm posting this to Delving Bitcoin to hear what you think of this design and its applicability to software you use and maintain. Let me know what you think.

Follow the community links found on https://payjoindevkit.org to get involved.

_New users can only post two links in a post, so I'll have to follow up with links to protocols referenced in this doc. Thanks for understanding._

-------------------------

bitgould | 2023-09-20 18:31:20 UTC | #2

## References

Oblivious HTTP ([OHTTP](https://ietf-wg-ohai.github.io/oblivious-http/draft-ietf-ohai-ohttp.html))

Hybrid Public Key Encryption ([HPKE](https://www.rfc-editor.org/rfc/rfc9180))

-------------------------

