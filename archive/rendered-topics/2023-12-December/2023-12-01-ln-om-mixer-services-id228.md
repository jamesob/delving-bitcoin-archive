# LN OM-mixer services

ZmnSCPxj | 2023-12-01 14:44:59 UTC | #1

Currently, Lightning Network onion messages are intended to be rate-limited, possibly with backpressure.

An issue is that OM is specified as being unreliable.  This means that it quite valid for a highly-pressured node to simply drop onion messages and never forward them. Now, OM has an advantage over Tor: individual messages are not necessarily correlatable with each other, whereas since Tor emulates a TCP tunnel, messages coming over the same TCP tunnel are correlatable by third parties (the entry and exit nodes) as belonging to the same underlying communications.  While you CAN reduce this with Tor, this requires that you open a new TCP-over-Tor tunnel, send a single message (and possibly wait for a single response) then close it, preferably changing circuits each time.

With OM, a node that is a full routing participant on the LN can have many multiple paths to send OMs to anything it wants to talk to, and to get responses back.  In particular, in protocols that are naturally request-response, the response circuit can be different from the request circuit.

However, consider the case where a node is a client of an LSP.  In this environment, usually the client only has channels (or promises of a channel, such as by a pending JIT channel open) with only that single LSP, because the whole point of the LSP/client divide is so that liquidity management is done by the LSP on behalf of the client.  But this implies that the client has limited access to send and receive OMs.  Yes, it could connect to a random node on the LN and send out an OM via that, but this is likely to get rate-limited or just dropped. and it certainly cannot receive over that random node.  If the client always sends out OMs via its sole LSP, then the LSP has all the information on the timing of OMs to and from that client, which the LSP may be able to correlate with other activity (for instance, if watchtowers are contacted over OM, the LSP might note if some client never sends out an OM and attempt to steal from them while they are offline).

So I was wondering if random routing nodes can also provide OM-mixer services.  This is basically just an additional paid service by which a client can send an OM to a different routing node, hopefully one which has good OM-bandwidth with its peers.

So an OM-mixer might provide tokens in a WabiSabi credential in exchange for money.  Then the client can connect to the OM-mixer (possibly with a different node ID and hopefully via a different IP(???)) and then spend some of the tokens in the WabiSabi credential, in order to send and receive messages (to receive messages, the client would register a short-lived synthetic SCID by which it can receive messages and which it can put in the `reply_path`).

Additionally the OM-mixer might give hints to the client on which outgoing/incoming SCIDs are currently not getting used for OMs recently. As noted, the expectation is that nodes will rate-limit incoming OMs to be forwarded and thus the OM-mixer might have been used to forward some message over some common path, and the OM-mixer could advise the client "actually use this path out of me instead" to improve the chances that the client OM will reach its destination.  Similarly for receive paths, the OM-mixer could hint if some incoming path is currently not near the rate-limit the OM-server has, so that the client can build a `reply_path` that does not increase the rate at which OMs are received by the OM-server on some nodes.

Currently, the OM system is intended for use with BOLT12.  I wonder if, with the help of such OM-mixer services, the OM system can be used for e.g. contacting watchtowers or dynamic channel state backup servers (CLN VLS, LDK VSS, et al.).  Those services are likely to have much higher rates of messages than BOLT12, I think.

-------------------------

