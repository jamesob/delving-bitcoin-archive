# Eddy: Free Cooperative Circular Rebalancing

wactario | 2026-03-21 22:49:44 UTC | #1

# The Problem

Circular rebalancing has a free rider problem. The initiator pays the routing fees, so everyone else waits for someone else to go first. A cycle might exist that benefits every node along the path, but nobody wants to be the one paying for it. And when someone does initiate a rebalance, they might be shifting other nodes’ balances in directions those nodes don’t want.

# The Solution

Eddy is a daemon that runs alongside LND. Each node gossips their channels’ surplus outbound to peers using LND’s custom P2P messages. Simultaneously, each node watches the gossip it receives and looks for cycles of channels where every node would benefit from liquidity moving in the same direction.
When a node detects a cycle and it has the lowest pubkey in that cycle (tiebreaker to prevent multiple nodes from proposing the same cycle), it proposes the rebalance to the other nodes. They agree or reject. If everyone agrees, the initiator executes the rebalance. If anything fails, the payment fails atomically. No central coordinator. No trust. No additional risk introduced.

# ResumeModified

This project went through three iterations. The first used a central coordinator, which required trust. The second removed the coordinator but used HODL invoices to lock HTLCs at each hop which is potentially dangerous if a node goes offline mid-circulation. The current version uses an LND feature I didn’t know existed: the `RESUME_MODIFIED` action on the HTLC interceptor. It was added in the v0.18.x timeframe as plumbing for Taproot Assets custom channels, but it turns out to be exactly what cooperative rebalancing needs.

`RESUME_MODIFIED` lets a forwarding node override the outgoing HTLC amount before forwarding. So if a node receives 1,000,000 sats and the onion says “forward 999,999 and keep 1 sat as your fee,” the interceptor can say “actually, forward the full 1,000,000.”
I [verified](https://github.com/wactario/free-rebalance-experiment) this behavior on regtest (Polar, LND v0.20, 3-node ring, non-zero fee per hop). The initiator pays a zero-fee route to itself, each intermediate node’s interceptor forwards the full incoming amount, and the initiator receives back exactly what it sent. If any intermediate node doesn’t waive its fee, the payment fails. Participation is all-or-nothing.

# Protocol

The protocol is best-effort and intentionally simple:

1. **Propose** — Initiator builds the route, creates an invoice, and sends a proposal (including the payment hash) hop-by-hop around the ring. Each node validates that the push would improve its balance and forwards the proposal to the next hop. If any node rejects, the proposal aborts.
2. **Execute** — When the proposal completes the ring back to the initiator, the initiator executes the payment along the pre-built route. Each intermediate node’s HTLC interceptor modifies the forwarded HTLC to waive its routing fee, so the circulation is free. The initiator holds the preimage, so settlement propagates back from it.

# Status and feedback

This is an MVP. The whole thing was vibecoded. I know it can be improved, that’s why I’m posting it here. I’m looking for code review and feedback on the design. Here is the [repo](https://github.com/wactario/eddy). Please tear it apart.

-------------------------

