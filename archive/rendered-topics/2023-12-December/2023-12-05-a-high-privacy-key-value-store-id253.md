# A High-Privacy Key-Value Store

ZmnSCPxj | 2023-12-05 14:25:53 UTC | #1

This is some shit I have been thinking about that is probably too insane to implement.  Tagging lightning here also since I started thinking about this crap in the context of dynamic channel state backups for Lightning nodes.  However the concept is general enough that in theory, you could use this for a remote filesystem or a database.

The goal here is to provide a remote store that is run "professionally" on e.g. cloud hardware, while ensuring that as little data about clients of the remote store is learned by the service.

Another goal is that obviously the client can recover their data later even if the client loses all its data.  The only requirement is that the client is able to retain the root private key corresponding to their data.

## The Store API

The store is a client-server setup, with a single server.

The primary store is a key-value store.  Keys are SECP256K1 points, a.k.a. public keys.  Values are fixed-size blobs, encrypted by the client (it might be best to do something like a round amount, such as 16384 bytes, plus enough extra bytes to encode a decent AEAD header, such as 33 bytes for a public key and 16 bytes for a MAC tag = 16384 + 33 + 16).

The primary interface is:

* insert - Add a key-value pair.
  * If the key already exists, fail, otherwise just insert the key-value pair.
* delete - Give a key and a signature of the corresponding value.
  * If the signature validates against the hash of the value / blob for the public key (the key), delete, otherwise fail the operation.
* lookup - Give a key and acquire the corresponding blob.
  * Clients could "fish" for the blobs of other clients, but if those other clients use decent cryptography to encrypt their blobs, it would be pointless, as the data would be indistinguishable from entropy.

The above is a very high-level view, I will show some nuance later.

### Paying For Storage

Obviously, a server providing this kind of service cannot be free, otherwise people will just dump all their crap into the server.
The server can monetize by various means, such as the archaic give-me-your-data-and-I-will-spy-on-you of Google, however, to ensure privacy, it is strongly encouraged to use a pay-for-service model.

The idea is that there will be some kind of Chaumian-bank solution, with the server getting paid and then issuing tokens in exchange for payment.
Clients then use these tokens to insert data into the store, and can recover a smaller fraction of the tokens when deleting data from the store.
The concrete proposal is to use WabiSabi credentials to store the numeric value of the number of tokens the client has, and to expose interfaces to transfer such values between credentials.
Insertion can then require that tokens be spent, and deletion could refund a smaller number of tokens.

(classic Chaumian blinded signatures, or the newfangled Wagner blinded Diffie-Helman key exchanges, have the issue that you need to send N tokens to represent a value of N, you could do the sensible thing and have tokens equal to powers of 2 so that you send log N tokens, but that reduces privacy since the different token denominations are not ACTUALLY fungible with one another,.  The costs of different operations vary (deletion is negative cost, insertions are higher cost than lookups) so variable N is more appropriate here, thus the WabiSabi credentials are better for privacy and usability.)

Any such token system needs to have "epochs" where old issued tokens are made invalid.
This is because spending a token requires the server to store some data indicating "this credential was already presented and we should reject this credential in the future".
If the token system is never "reset" then this is an infinitely-growing structure.
Epochs let the server delete data about spent older tokens, while clients can use slightly-old tokens and have them reissued to the latest epoch (e.g. the server might save data for the latest epoch and up to N epochs ago, and delete epochs older than that, letting clients have tokens from up to N epochs ago reissued to the latest epoch).

Insertions would require some N tokens, while deletions would refund tokens less than N.
This incentivizes deletion of unneeded data, while the time during which the data was stored will effectively paid for by the difference between the insertion cost and the deletion refund amount.

### Breakglass Storage

Now, we might argue that lookup, at least, should be free.
However, this is infeasible on the open Internet.
Attackers may attempt to bring down such a service by overloading its `lookup` interface.
Traditionally, such attacks are blocked by blocking certain IP ranges from which such attacks come from, however, this does not consider the case where third-world countries happen to have tiny IP ranges allocated to them, and most people end up with ISPs sharing IP addresses between their clients dynamically.
Attacks mounted on such IP ranges then cause innocent legitimate clients on the same IP range to also be blocked.
Thus, lookup must also require tokens to be spent.

Further, ideally the server does not learn anything about the client, including its IP address.  Thus, it must use a mechanism that allows the client to hide its identity, but this also lets potential DoS attackers to hide their identity.
Again: lookup must not be free.

However, again, remember that the goal is for the client to be able to recover all data it has.
If it needs tokens in order to read from the store, but the client has lost its cache of tokens because it lost ALL of its data except its root private key (represented by 12/24 words), how can the client recover anything?

The answer is to provide a second, "break glass" storage.
This is a much smaller key-value store, where keys are again SECP256K1 points (public keys), but values are tiny blobs, just enough to fit the encrypted form of a WabiSabi credential.

The breakglass storage has this interface:

* insert-or-replace - costs tokens, inserts a public key and inserts or replaces the value.
  * Requires a signature from the public key.
* lookup-and-deferred-delete - give a public key, get value.
  * Requires a signature from the public key, but does NOT cost tokens (this is breakglass).
  * On success, a timer starts ticking and the entry is deleted after the timer runs out.
  * The lookup operation can be repeated until the timer runs out.
  * The timer should be relatively long, unit of hours or days.

The tokens spent to insert into the breakglass storage effectively pays for both the insertion and long-term storage of the small amount of data, and the subsequent lookup.
Looking up deletes the data, as the tokens used to insert to the breakglass limit the number of bytes that can be retrieved later, and prevents this mechanism from being abused to store the data instead of the primary store.
Deletion is timered instead of immediate, as network is unreliable --- the server might receive the request, but the network (or the client) may fail before the client can receive the request and re-seed its token cache with the breakglass.
During the timeout, the client may recover or re-attempt the read, to still recover the data, but the data is "doomed" and WILL be deleted to prevent abuse of the breakglass store.

Further, storage, on a per-byte basis, must be more expensive for the breakglass compared to the primary storage, thus encouraging users to use the primary storage for actual mass data.

The breakglass store is effectively a persistent "identity" for a client.
However, the only activity on the breakglass is to initially put some tokens into it for emergency recovery purposes.
Any token system like this would have "epochs" where old tokens are made invalid and need to be "reissued" to the latest epoch, so the breakglass has to update that, but importantly this is NOT correlated to the client write or read activity in the primary store, only to epochs, thus is not a major privacy leak; it only leaks that "the client is still willing to pay for continued recoverable storage for the current epoch".

The client may use any fixed derivation from their root private key.
If used for LN state storage, the key used in the breakglass MUST NOT be the same as used as any node ID of the client.

### Client Operation

An important power of the client is that a client can get any number of public keys.
The space of keys is so vast that we expect that the server will run out of space faster than the number of possible public keys.

Let us focus on CLN.  In CLN, the `db_write` hook provides a counter.  A client using CLN that wants to save `db_write` querysets can simply HMAC the counter with the node private key and then use that to create the keypair used to insert data into the primary store.
The `db_write` queries can be divided into batches that fit into the blob size, and if it requires more than one blob, it can get some entropy, store that in the blob, and then get the node private key plus the entropy to generate a new keypair for the continuation of the data, creating a singly-linked list of blobs. (Implementation-wise, it really ought to insert the continuation blobs and ensure they are inserted before inserting the head blob for that version).

On recovery, the recovery process can simply iterate over versions (again, by just HMACing the counter value with the node private key) and look them up on the server, acquiring blobs of `db_write` querysets that can be applied to a starting database.  When the last version is found, the recovery process has completed.

For LDK, each channel has its own monotonic counter.  In such a case, the "main" sequence would contain openings and closings of channels.  This would require a separate counter for the sequence of channel opens and closes, which can be implemented outside of LDK.  A channel closing can allow deletion of data as well, to refund some amount of tokens.

On recovery on LDK, we go through the "main" sequence, looking at channel openings and closings, until we reach the final version and get a canonical list of currently-open channels.  Then the recovery process can start downloading individual updates of the currently-open channels --- for instance, they could HMAC the channel ID and the counter, with the node private key, to get the key to the blobs containing each `ChannelMonitorUpdate`.

Note that this use of counters can be used for any other store.  For example, it would be possible to write an immutable b-tree implementation that can be used to back a database or fileystem on such a service.  The client is free to implement any number of data storage schemes with any index, with the superpower of getting any number of public keys derived from its root private key, as we expect that the amount of data that any client can feasibly generate and store will be much smaller than the space of valid SECP256K1 keys.

Any equality index can be converted to a public key by just HMACing a canonical serialization of the index key with the client root private key.  Thus, any kind of key-value store can be built on top of this mechanism.

Actual datastores might want to "roll up" some of the old data, so that a sequence of updates is replaced with some "latest" version.  This is left as an exercise to the reader.  For instance, the "main sequence" might just contain a reference to the root of an immutable Merklized data structure (such as an immutable b-tree).  As data gets updated, older versions may be deleted over time once new latest anchors have been written to the store.  On recovery, the main sequence is traversed and the latest anchor can be recovered.  This lets clients delete some blobs and recover tokens for those, which also reduces the amount of data the service has to store.

### Transport Layer

Any privacy-focused mechanism needs to get around the fact that the public Internet is VERY VERY public.

Obviously, we cannot have clients make a TCP tunnel directly to the service.

We can consider the use of Tor.  However, Tor emulates a TCP tunnel.  If a client makes multiple requests over the same TCP-over-Tor tunnel, the service can correlate them as coming from the same client.  Thus, the client really needs to create a TCP-over-Tor tunnel, send a single request, then close it.  It is also strongly recommended that the client change circuits between requests, in case it turns out to be the only user of a specific exit node.

An alternative is something that we already built: the Lightning Network Onion Messages system.

Onion messages can include a `reply_path`, which makes it very nice for request-response type of operations.  Generally, onion messages can be sent only over actual published channels (otherwise the client has no idea whether two nodes on the network have a connection between them).  This limits the number of ways by which a service could potentially be contacted, reducing the ability of the service to "bin" clients.  A client with an up-to-date gossip map (using some kind of semi-trusted fast gossip sync) can generate multiple circuits to the service by which it can distribute its interactions with the service, further reducing the ability to "bin" clients.

An LN client that has only a single channel to a single LSP, and which uses this key-value store service on the same LSP, can send out OMs over that single channel, cycle it through some nodes, and then route it back to the LSP.

An issue is that LN OM are currently targeted only for BOLT12 invoice requests, which are expected to be low-bandwidth.  For large-scale data storage as proposed here, that assumption would be tested.  Nevertheless, it seems to me that the LN OM system is better here than OHTTP or Tor: OHTTP has only a single hop (laughably insufficient for privacy) while Tor has a persistent TCP tunnel instead of request-response messages (and a client can have the response sent over a different path than the request, via `reply_path`).

-------------------------

