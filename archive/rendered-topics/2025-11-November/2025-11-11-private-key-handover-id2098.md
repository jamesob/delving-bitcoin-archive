# Private Key Handover

ZmnSCPxj | 2025-11-11 04:06:55 UTC | #1

Subject: Private Key Handover

Introduction
============

There are protocols where, at the end of the protocol, semantically,
some lump fund which was previously shared by two or more
participants, is now owned by only one of the participants.

For example, an onchain HTLC starts with two participants: the offeror
and the acceptor.
Assuming the "happy path" where the preimage is learned by the offeror,
at the end of the protocol, the offeror semantically releases its claim
on the HTLC funds and the fund (should) now be singularly controlled by
the acceptor.

When implementing such a protocol onchain, on a blockchain that supports
Taproot, a possible optimization is for the protocol to always require
ephemeral public keys in the keyspend path, and for participant(s) that
want to release their claim on the fund to simply hand over the ephemeral
private key to the final single beneficiary, which would allow the
beneficiary party to use the keyspend path to claim the funds without
further participation of any other party.

In our HTLC motivating example, the keyspend path of the HTLC would be
the MuSig2 of the ephemeral keys of the offeror and acceptor, and in the
happy path, the acceptor gives the offeror the preimage directly (instead
of onchain) and the offeror then hands over the ephemeral private key.
The acceptor, now in possession of both halves of the ephemeral private
key of the keyspend path, can now use the keyspend path unilaterally.

Benefits And Limitations
------------------------

To be very clear, in a world where MuSig2 exists and is implemented, the
advantage is:

* The beneficiary party can RBF the transaction to claim the lump fund,
  even if the protocol has no support for RBF.
  - In particular, the transaction to claim the lump fund does ***not***
    need to have anchor outputs (which require extra blockspace weight)
    in order to freely (CPFP-)RBF after the termination of the protocol.
  - This can simplify the design of the protocol, while allowing for
    high-quality implementations to RBF the transaction that claims the
    fund (simple proof-of-concept implementations need not implement
    RBF, and are simplified by the absence of RBF support in the
    protocol; the RBF support can be added later without changing the
    protocol or requiring that the simple initial implementation has
    RBF support).
  - Without private key handover, we can still use MuSig2 to save
    blockspace, but must select one of these options:
    - Not have RBF, so you run the risk of sudden fee spikes after
      completing the MuSig2, possibly allowing timeout branches to
      become valid and risking funds loss.
    - Support RBF in the protocol, so that all participants (not just
      the beneficiary) need to stay online until the fund spend is
      deeply confirmed onchain, with additional protocol messages and
      state-tracking of multiple possible RBF candidates, needing to
      be implemented by all participants.
    - Support RBF-CPFP by adding an anchor output to the transaction
      signed using MuSig2 in the protocol, which increases your
      block space usage by that anchor output if RBF turns out to be
      unnecessary.
      This also ***requires*** exogenous fees and additional
      blockspace usage if RBF ***does*** turn out to be necessary.
* The beneficiary party can batch the transaction that claims the
  lump fund with ***any*** other operations.
  - The "other operations" here can be completely different
    protocols that do not themselves have any additional support for
    private key handover!

We should note the limitation as well:

* The fund must be a lump fund, i.e. a single UTXO onchain, and the
  fund must be transferred as a whole, at the end of the protocol, to
  unilateral control of some single beneficiary party.
  - In particular, this cannot be used to remove RBF support in the
    Lightning splicing protocol, as after the end of the protocol, the
    fund in the protocol --- the channel output --- must still be in
    bilateral control of both parties.
  - In particular, this cannot be used to create RBF support in the
    Lightning cooperative close protocol, as after the end of the
    cooperative close, the likelihood is very high that the single
    fund --- the channel output --- must be split into ***two***
    unilaterally-controlled funds; this optimization can only work on
    single funds.

Private Key Handover
====================

The "private key handover" is a building block for Bitcoin protocols:

* At the start of the protocol, each participant provides two public
  keys:
  - An ephemeral public key.
    - This should be a fresh keypair from entropy.
    - If using a keypair derived from some master private key,
      should use the equivalent of hardened derivation (e.g.
      derive using HMAC, where the key is the master private key,
      and the message is some nonce tweak).
    - Participants do not need to store the corresponding private
      key in persistent storage, and it would probably be
      undesirable to store it (or its derivation path) in
      persistent storage.
  - A permanent public key.
    - This should be recoverable across restarts of the software.
* The keyspend branch of the Taproot output is computed as the
  MuSig2 of the ***ephemeral*** public keys of all participants.
  - The participants MUST store the MuSig2 sum of the public keys
    in persistent storage.
  - Public keys are ***public*** and exfiltration of the public
    keys only reduces privacy, but does not lead to loss of funds.
* Individual leaves of the Taproot output use the ***permanent***
  public keys as appropriate.
* At the end of the protocol where a single beneficiary party is
  selected, the other participants send the ephemeral private key
  to the beneficiary.
  - In case one or more of the other participants do not
    cooperate by giving their ephemeral private key, the single
    beneficiary can use the appropriate Taproot leaf.
* On receiving all the ephemeral private keys of the other
  parties, the beneficiary party can then repeat the MuSig2
  calculation using the private keys, to generate the combined
  private key.

For an HTLC protocol to be used as a building block for a swap
protocol:

* When setting up the HTLC, the HTLC offeror and HTLC acceptor
  exchange their public keys:
  - Offeror ephemeral public key.
  - Offeror permanent public key.
  - Acceptor ephemeral public key.
  - Acceptor permanent public key.
* The HTLC Taproot output would have a keyspend branch that
  is composed of the MuSig2 of the offeror ***ephemeral***
  public key and the acceptor ***ephemeral*** public key.
  - The participants MUST store the MuSig2 sum of the public
    keys in persistent storage.
* The HTLC Tapleaves would then be:
  - Timeout branch: Signature from offeror ***permanent***
    public key after a `CLTV` or `CSV` timeout.
  - Preimage branch: Signature from acceptor ***permanent***
    public key, and a preimage of a given hash.
* At the end of the overall swap protocol, the acceptor then
  provides the preimage to the offeror.
  In response, the offeror hands over the offeror
  ***ephemeral*** private key.
* On receiving the offeror ephemeral private key, the acceptor
  derives the combined private key for the MuSig2 sum of the
  ephemeral public keys.
  The acceptor then checks that the resulting sum private key
  matches the stored sum public key, and if so, can then spend
  the fund unilaterally.
  - As mentioned, this allows the acceptor to RBF any spends of
    the HTLC output without the protocol having to add special
    messages for RBF, and without requiring CPFP-RBF support via
    an anchor output.

The following fallback cases need to be handled:

- In case the offeror does not hand over the offeror ephemeral
  private key after a few seconds, or the offeror sends an private
  key that causes the resulting combined private key to not match,
  the acceptor can fall back by spending using the Tapleaf branch
  where it has to sign and show the preimage of the hash.
  - This requires that the acceptor know the public key of the
    keyspend path.
  - The acceptor can recover this public key as long as it can
    recover the MuSig2 sum of the ephemeral public keys.
- In case the acceptor software is restarted after it has sent the
  preimage, but before it can receive the offeror ephemeral
  private key, the acceptor can fall back by spending the Tapleaf
  branch where it ahs to sign and show the preimage of the hash.
  - The recommendation is to only store the ephemeral private keys
    in memory, so if the acceptor software is stopped and
    restarted, the acceptor ephemeral private key is lost and it
    cannot calculate the sum of the ephemeral private keys.
  - Thus, the requirement that the participants MUST store the
    MuSig2 sum of the ephemeral public keys, which lets the
    participants spend the funds using the Tapleaf branch by
    showing the keyspend branch public key and the Merkle tree
    proof of the corresponding Tapleaf.

Of note is that, as the keys in the keyspend are ephemeral, and
by "ephemeral" we mean "will only be used for this run of the
protocol", this gets forward security once the spend of the fund
is deeply confirmed.

Graceful Implementation
-----------------------

Initial implementations of a protocol that uses private key
handover need not ***immediately*** support RBF or batching.

Thus, initial implementations of the protocol can be simple,
naive implementations that ***just*** spend the lump sum
output to an address with unilateral control of the
beneficiary, in a simple one-input-one-output unbatched
transaction with a feerate slightly higher than prevailing
feerates.

The ability to add RBF or batching can be added later once the
implementors have enough confidence in their work for the
simple use-case, to pursue more ambitious sophistication.

Importantly, implementations *with* RBF and/or batching
support can talk seamlessly with simpler implementations
*without* that support; the protocol remains the same, but
there is no need for the simple implementation to be aware of
additional messages or anything needed by the sophisticated
implementation.

This allows for "graceful implementation" where you can start
with a simple implementation and add features, without breaking
protocol compatibility, and also start off with a simpler
protocol that will work at both the "proof-of-concept" scale
to "mass adoption champagne problems" scale.

Particularly sophisticated implementations can even batch the
spending of the lump sum with completely unrelated, other
protocols, which themselves may or may not use private key
handover, saving even more blockspace.

Compare this to the complexity of Lightning Network splicing.
Even initial implementations that have no intention of supporting
RBF or batching, must be at least aware of the messages that are
needed if the counterparty has to RBF or batch.
Indeed, initial implementation are forced to implement RBF, as
multiple versions of the splice transaction need to be signed in
parallel until one of the versions is deeply confirmed (so you
might as well just go whole hog and be able to propose the RBF
yourelf, since you need to keep track of multiple exclusive
transaction versions either way).

Encrypted Handover
------------------

We should be cautious about sending private keys across the
network.

Of course, one layer of protection is to use BOLT8, or the less
advanced protocol TLS, to send messages containing the private
key.
This provides end-to-end encryption and message integrity.

However, this is only *one* layer of protection.
In particular, the encryption and integrity ***ends*** at the
point in the software where the BOLT8 decryption and integrity
validation is done.

Modern software can be very complex and may send various
information to various sub-components.
Plaintexts of the messages after the end-to-end encrypted
integrity tunnel would be sent around and possibly have
multiple copies in transit from one part of the software to
the part which actually implements the protocol that uses
private key handover.
Additional copies of private keys would be problematic as it
increases the surface area for private key exfiltration.

As a concrete example, the C-Lightning (also known as CLN or
Core Lightning) implementation will send out plaintexts of
odd-numbered messages, via pipes, to any plugins that register
for notification of odd-numbered messages.
Partial security breaches may allow hackers to get at in-pipe
data, and various ways of running CLN plugins remotely may
allow the odd-numbered messages to be sent via JSON-RPC
notifications, in plaintext, over the network.

To reduce this attack surface, our protocol can send over the
encryption of the ephemeral private key, as follows:

* Suppose that participant Alice (HTLC-offeror in the HTLC
  use-case) wants to hand over their ephemeral private key to
  participant Bob (HTLC-acceptor in the HTLC use-case).
* Alice takes the ECDH of the Alice ephemeral private key and
  the Bob ephemeral public key:
  - Multiply Alice ephemeral private key (scalar) by Bob
    ephemeral public key (point), resulting in a point.
  - Format the point into the 33-byte DER format.
  - Take the SHA256 hash of the above 33-byte DER format,
    resulting in 32-byte hash.
- Alice XORs the 32-byte ECDH output with the Alice ephemeral
  private key.
  - This is best done using an in-place XOR where the result
    overwrites the plaintext Alice ephemeral private key, to
    further reduce the time the plaintext copy is available
    in-memory.
  - After this step, Alice can safely free the memory for
    reuse in insecure code.
- Alice sends the XORed result to Bob (preferably in an
  encrypted integrity tunnel, such as BOLT8).
- Bob receives the XORed encrypted result.
- Bob takes the ECDH of the Bob ephemeral private key and the
  Alice ephemeral public key.
  - This should give the same mask as above.
- Bob XORs the 32-byte ECDH output with the encrypted private
  key from Alice.
  - This results in the original Alice private key.

This scheme reduces the scope of where the plaintext private
key is available, providing extra layers of protection in
contexts where it would be desirable.

(The proposed scheme is only safe under the random oracle
model.)

While many implementations would not need to have the
protection against internal breaches of data, putting it as
part of the protocol allows the rare implementation that
***does*** need it to be safer.

The encryption sub-protocol requires primitives (SECP256K1
scalar by point multiplication, 33-byte DER formatting, SHA256)
that are likely to be commonly needed in Bitcoin software anyway,
so the additional implementation complexity of using encrypted
private key handover is expected to be low.

Non-HTLC Examples
=================

Private key handover is a generic building block for protocols.
It can be used beyond HTLCs.

For example, in the theoretical SuperScalar design, a client
may cooperatively exit from the LSP, by offering the entire
amount inside the SuperScalar for some equivalent onchain
amount.
The client can use private key handover for the keys it used
inside the SuperScalar in the cooperative exit protocol; once
the server has given an onchain contract that the client can
use to atomically swap its in-SuperScalar funds for the
onchain funds.

In particular, with possession of the client key after the
client has exited, the LSP can now arbitrarily reallocate
liqudiity towards other clients that happen to share part
of the SuperScalar tree with the client that exited, giving
the LSP extra flexibility in its liquidity management.

The only requirement is that the client use some kind of
hardened derivation for the in-SuperScalar keys; because the
keys are handed over on exit, the client must ensure that
the handed-over private key cannot be used to derive their
master private key.
Obviously, as the private key would need to be used across
client restarts, the client would need to store the
derivation path for the in-SuperScalar key(s) in persistent
storage.

Even outside of SuperScalar, a bespoke LSP-client protocol
can allow for a form of cooperative exit where the LSP can
sign the channel output fund unilaterally, in exchange for
the equivalent amount of client funds in an onchain TXO.
This allows the LSP to batch multiple cooperative exits:

* The client that wants to exit offers the entire funds of
  the channel with the LSP, swapping it for onchain funds
  from the LSP.
  - The LSP can batch the onchain funds it offers into the
    swap with other operations.
  - After the swap, the client hands over their private key
    to the channel.
    The LSP can now unilaterally sign for that channel, and
    can batch the spend of the channel to recover its funds
    with other operations.

If the LSP has enough onchain churn (e.g. if it is also
operating other onchain businesses, such as a Lightning-onchain
swap service, or operating as an exchange at enough scale to
use onchain operations), the ability to freely batch the above
operations with others can provide actual blockspace savings.

Again, the only requirement is that the client use some kind
of hardened derivation for the channel signing keys.

Non-Taproot Usage
=================

Of note is that while ***true*** MuSig2 signing, where there
is no private key handover, is only possible with Schnorr
signatures, if private key handover is done, you can use a
MuSig2 sum of public keys with ECDSA.

After private key handover, the beneficiary can compute the
corresponding private key of the MuSig2 sum.
Thu, even if ECDSA is not linear, the beneficiary can still
compute the equivalent private key anyway.

The major drawback for ECDSA usage is that there is no ECDSA
address that *also* has Taproot.
Thus, you would need to expose an additional public key in
the single SCRIPT, which may very well cancel out any
blockspace savings.
Nevertheless, it is still quite possible to use private key
handover for additional flexibility and the ability to RBF
and batch arbitrarily, even without blockspace savings.

(It should be noted that the last example in the previous
section --- bespoke LSP-client cooperative exit protocol
where the LSP gets the flexibility to batch with other
operations --- would, today, be using ECDSA.)

-------------------------

ZmnSCPxj | 2025-11-12 00:15:13 UTC | #2

Shared-spending Inter-Protocol Framework
==

A high-quality ***general*** Bitcoin-management (a.k.a. "wallet") implementation can implement the following software interface as a framework by which additional code can implement protocols that are capable of sufficient flexibility to allow arbitrarily adding new transaction inputs and new transaction outputs.

As a concrete example, any protocol that uses Private Key Handover as described in the main text above can integrate with the below-described inter-protocol framework in order to create batched, RBFed transactions.

The framework maintains a (possibly empty) set of requested commands.  A "command" in this context is either (1) a request to spend an input before some timeout or (2) a request to fund some address before some timeout.

In particular, end-users can simply call `fund_output` one-at-a-time and the framework will, if feasible, batch the outputs into a single transaction, i.e. instead of requiring end-user code to call into a `send_multi`-equivalent command (so that end-user code has to decide to batch before calling into the Bitcoin-management framework), the end-user code calls multiple `fund_output` commands, and the framework automatically batches them, using RBF to replace any broadcasted transactions.

The framework implicitly assumes that all inputs are some SegWit version, and thus have a non-malleable `txid` independent of signing.  The framework simply bans non-SegWit inputs outright.

The framework has two entry points: `fund_output` and `spend_input`.  Both accept one or more callbacks, which the framework uses to coordinate with the actual protocols.

* `fund_output(amount, timeout, requesterAddressCallback, requesterReadyCallback, requesterDoneCallback)` - Tell the framework that it has to make a transaction that sends out an `amount`, targeting the transaction to be confirmed before the `timeout` blockheight.
  * The `requesterAddressCallback` is called whenever the framework has decided on some number of owned UTXOs, plus any pending spend-an-input requests, and a feerate to use.  It is given the feerate the framework has decided on.  It should return either a destination address for the amount, or a request to cancel itself and remove its request to fund the output.
    * For example, consider the LN BOLT `openv1` protocol, where this might be used to initiate a new `open_channel`, and thus generate a new address from the replied `accept_channel` (i.e. this callback would send `open_channel` and wait for a corresponding `accept_channel`).
    * The framework may call this at any time, as long as the request has not been cancelled, for example if it decides to RBF.
  * The `requesterReadyCallback` is called whenever the framework has already called `requesterAddressCallback` on all `fund_output` commands, and has created, but not signed, the transaction.  It is given the stable `txid` of the transaction and the output index of that transaction where it is created, as well as the corresponding address.  It returns either an `ok`, or a request to cancel the transaction and remove its request to fund the output.
    * For example, consider the LN BOLT `openv1` protocol, where this would be used to send `funding_created` and wait for a `funding_signed` (and if it times out waiting for `funding_signed`, to cancel the request instead of continuing).
    * The framework will not sign and broadcast the transaction until after all `requesterReadyCallback`s return `ok`.  If any do not return `ok`, the framework will remove that output and then restart its loop starting with calling `requesterAddressCallback` on all unremoved output requests.
  *  The `requesterDoneCallback` is called when some transaction that has funded the output has been confirmed (possibly to some depth determined by policy).  This is given the specific txout and address.
     * For example, consider the LN BOLT `openv1` protocol, where this would be used to send `funding_locked` / `channel_ready`.
* `spend_input(txin, amount, timeout, requesterSignCallback)` - Tell the framework that some specific transaction `txin` (a stable `txid` + `outnum`) must be spent before the given `timeout` blockheight.  `amount` is given here but depending on implementation it may be possible to remove it from the interface (in principle a full UTXO-map would store the `amount` and spending conditions, but that is not trivially exposed by e.g. bitcoin-core).
  * If the framework currently has no `fund_output` requests, the funds (minus fees) should go to an address that the framework wallet unilaterally controls.
  * The `requesterSignCallback` is called whenever the framework has already decided to sign and broadcast some transaction that includes this input.  It is given a PSBT, and it must return a PSBT with the corresponding input having the required witness filled in.
    * For example, consider the LN BOLT `revocation` protocol, where the only requirement is to sign with the `revocationpubkey`, and the transaction can spend to anything.

For a simple "send to an address" base usecase, you can call `fund_output` with the amount to be sent, with trivial callbacks where `requesterAddressCallback` returns the target address, `requesterReadyCallback` trivially returns `ok`, and `requesterDoneCallback` notifies the user UI.  The magic here is that if the human operator performs multiple "send to an address" commands in sequence, the wallet framework automatically batches them (it can delay its decision loop to wait for new commands to reduce the number of times it RBFs, but it can always RBF newer transactions for each new "send to an address" request).  In addition, in case of a sudden surge in fees, the wallet framework can automatically RBF without further human operator input; further, the human operator can specify some maximum feerate, and then `requesterAddressCallback` can fail the request automatically if the feerate is higher than the maximum set by the human operator.

For the proposed "Private Key Handover", any protocol that implements Private Key Handover proposed in the original post can call into `spend_input`, and provide a `requesterSignCallback` that uses the handed-over private key to sign the input it is claiming.

Of note is that obviously the framework needs to continue operating across restarts.  Thus, the actual callbacks would not be some kind of in-memory representation of a function; instead, you would need to pre-register some set of protocols that can use this framework, with some kind of per-protocol name or identifier that is stable across restarts (for example, it can be a versioned string, such as `simple_send_to_addr_v1`).  This registration for protocols that would call into `fund_output` could provide the function addresses for all the callbacks needed, so that the ***actual*** `fund_output` call accepts only the registered name.

The framework would store the pending requests on persistent storage, and on restart, would load and look for any pending requests, and check the stored protocol names against the pre-registered protocols (and abort the restart if the in-storage string does not match any pre-registered protocols).  This make the framework a little harder to use, but makes it robust against restarts; the framework can record exactly where in its processing it has completed (e.g. if it has called some callback for some particular request).  This may require that individual requests also have an identifier (such as a UUID) across restarts.

-------------------------

