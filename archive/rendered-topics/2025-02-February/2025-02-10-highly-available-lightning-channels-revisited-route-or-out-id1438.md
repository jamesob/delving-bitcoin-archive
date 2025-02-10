# Highly Available Lightning Channels Revisited – ROUTE OR OUT

joostjager | 2025-02-10 17:01:20 UTC | #1

I wanted to revisit the idea discussed here: https://www.mail-archive.com/lightning-dev@lists.linuxfoundation.org/msg03094.html

## The core idea

The goal is to move toward a Lightning Network with zero payment failures. Currently, this is addressed using complex probabilistic scoring models — difficult to design and even harder to validate. But if the future holds a 100% success rate, probabilistic scoring becomes unnecessary. Instead, nodes could simply be skipped for an extended (nearly indefinite?) period after a single failure on a single channel. ROUTE OR OUT.

## Transitioning

To adopt this approach, nodes could signal that, for specific peers, a 100% forwarding success rate can be assumed up to a certain amount. This would allow routing node operators to gradually improve their service levels rather than making abrupt, system-wide changes. They could start small—offering guaranteed success with a single peer—and expand from there.

Senders might prioritize connections with this high availability (HA) signal and penalize aggressively if such a connection still fails. They might even be willing to pay higher fees for HA channels.

## Concerns & counterpoints

In the linked discussion, various developers raise concerns about this concept. However, the actual impact remains unknown. Moreover, nothing prevents routing nodes from modifying their software to send this signal. One could argue it’s better to formalize and standardize this signaling rather than leaving it to unofficial modifications.

In fact, developers aren’t even in control here. Routing node operators using existing implementations can already signal high availability today. The `htlc_maximum_msat` channel update parameter still has unused bits that could be repurposed for this.

For instance, in the public graph, no node currently sets `htlc_maximum_msat` to a value ending in 555. Nodes could slightly adjust their `htlc_maximum_msat` to end in 555, signaling that the channel is highly available for amounts up to that value.

Interested to hear more thoughts on the concept of high availability signaling and the proposed deployment mechanism.

-------------------------

cguida | 2025-02-10 19:04:45 UTC | #2

I like this proposal.

LN needs to be both private and fast/cheap/reliable, but it seems to me that the two aspects are mutually exclusive. However, it should be possible for the user to select whether they want payments to be fast/cheap/reliable on one hand, or private on the other hand, and different routes on the same network could serve different purposes.

-------------------------

