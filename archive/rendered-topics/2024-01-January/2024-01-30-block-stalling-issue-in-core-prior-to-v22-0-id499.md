# Block-stalling issue in Core prior to v22.0

Crypt-iQ | 2024-01-30 16:51:22 UTC | #1

In May 2021, I reported a block-stalling issue to the Bitcoin Core security email. Master at the
time of writing was `6b254814c076054eedc4311698d16c8971937814`. While the majority of the network has upgraded to at least v22.0, there are still a few thousand stragglers so hopefully this disclosure will motivate those running lightning nodes to upgrade to a safer version. Additionally, I think it's informative to let the broader community know about historical attacks on lightning.

## Background ##
Some background on bitcoind (circa May 2021):

* bitcoind selects three of its peers to relay compact blocks to it. A peer is chosen if it is the first
  to relay a non-compact block. An attacker was able to replace the victim's three compact-block slots
  with attacker-controlled peers by providing blocks before the honest nodes could. This logic was in the
  [PeerManagerImpl::MaybeSetPeerAsAnnouncingHeaderAndIDs](https://github.com/bitcoin/bitcoin/blob/6b254814c076054eedc4311698d16c8971937814/src/net_processing.cpp#L818)
  function.
* When bitcoind requests a block, an entry is added to `mapBlocksInFlight` and the relevant node has 10
  or more minutes to reply with the requested block: https://github.com/bitcoin/bitcoin/blob/6b254814c076054eedc4311698d16c8971937814/src/net_processing.cpp#L4715
* bitcoind will evict peers via the `AttemptToEvictConnection` logic. If this function is called during
  the attack, it will make it harder to pull off. However, the attacker may be able to counteract this by
  simply adding more connections in the setup phase.

## The Attack ##

The attacker performs the following steps:
* Replace victim node's compact block connections by providing blocks before the honest connections can.
  The attacker should then have 3 compact block connections with the victim.
* Connect to the victim node with N different connections. If the CLTV delta is 40, then let N be 50. These
  connections should be sequential in the connection manager's `vNodes` vector.
* When a new block is mined, the first connection of the N races with the victim's honest connections to
  announce via headers-first announcement. If successful, the victim should add an entry to `mapBlocksInFlight`
  for this connection. The connection then has a lower bound of 10 minutes to announce the block via a
  `NetMsgType::BLOCK` message.
* The 3 attacker-controlled compact block connections do not announce the block via compact block.
* The other N-1 connections can announce the block via headers-first relay, and they will not be selected to
  download the block from by the victim. This ensures that the victim later sees these connections as viable
  for requesting the delayed block from.
* Just before the timeout for the first connection elapses (say at the 9m45s mark), the first connection sends
  an invalid `NetMsgType::BLOCK` which deletes the entry from `mapBlocksInFlight` via `MarkBlockAsReceived`. This
  can be done by mutating the block without invalidating the hash for example. The first connection should get
  disconnected due to the invalid block. I believe it's also possible to simply disconnect instead of sending
  a mutated block, but I did not test this.
* At this point, the 3 attacker-controlled compact block connections can be replaced by honest ones. They don't
  matter anymore. Stale blocks should not be announced via compact block connections unless explicitly requested.
* The connection manager will choose the next connection in `vNodes` and call `SendMessages` on it. Since the
  attacker has stacked up 49 more connections, this chosen connection should be attacker controlled.
* `SendMessages` logic will select the chosen connection to request the delayed block via `FindNextBlocksToDownload`.
* Before the relay timer is activated at the 10-minute mark, the connection relays a bad `NetMsgType::BLOCK`, to
  delete the entry from `mapBlocksInFlight`, and trigger the next connection in `vNodes` being selected to relay
  the block.
* This process completes until all sequential attacker-controlled connections are used up.  This should result in
  relay for this specific block being delayed for the time it takes to mine roughly 50 blocks.

## Implications for LN ##

Let's assume the topology M1 <-> B <-> M2 where each node is using a CLTV delta of 40. M1 and M2 are both
malicious.

* M1 routes an HTLC through B to M2. It will time out on the B <-> M2 channel at t+40 and time out on the
  M1 <-> B channel at t+80.
* At t+38 (or earlier for higher chances of success), M1/M2 start delaying blocks to B's bitcoind node.
* M2 broadcasts a force close tx (if B hasn't already) and it confirms at, say, t+39.
* M2 broadcasts a tx that spends via the preimage-claim path and it confirms at t+40.
  * B can learn of the preimage by watching the mempool, but this doesn't help.
* M1 force closes their channel at t+40. It confirms at t+41.
* At t+80, M1 relays the 2nd-level timeout claim tx. At t+81 it confirms.
* M1/M2 can stop delaying blocks as they have stolen the value of an HTLC.

In this example with a CLTV delta of 40, the attacker needs to delay for the CLTV delta + X where X is small.
A larger X value gives better chances of attack success. By default, bitcoind has an upper bound of 117
inbound connections. Implementations may not be affected if the minimum allowable CLTV value is high.

## Patch ##

This was fixed with two PR's that landed in 22.0:
* https://github.com/bitcoin/bitcoin/pull/22144
  * This randomized the message processing order and pretty much guarantees that an honest peer will be
    sandwiched in between two malicious block-delaying peers.

* https://github.com/bitcoin/bitcoin/pull/22147
  * This prevents the last outbound high-bandwidth compact-block relaying peer from being demoted by an
    inbound attacker.

-------------------------

instagibbs | 2024-01-30 21:10:59 UTC | #2

To be extra conservative running a non-listening node makes this, and other of these kinds of attacks, much more difficult.

-------------------------

