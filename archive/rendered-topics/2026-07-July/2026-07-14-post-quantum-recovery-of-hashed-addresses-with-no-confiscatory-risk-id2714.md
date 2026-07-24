# Post Quantum Recovery Of Hashed Addresses With No Confiscatory Risk

shinobi | 2026-07-14 16:55:27 UTC | #1

Currently the discussion around how to address the potential risk of a viable quantum computer centers around two primary issues: the quantum-safe signature schemes (or schemes) to integrate for users to migrate to and use if need be, and what (if anything) to do about any coins that remain secured by vulnerable ECC based scripts, specifically should these coins be frozen, and what mechanisms are available to allow legitimate owners to recover those coins if possible without an attacker having the ability to do so.

This second issue is (and always has been) a very socially contentious one, as the common understanding is it is impossible to guarantee with certainty that no user is being left in a position where they are incapable of generating a recovery proof, and therefore are forever prevented from accessing their coins.

My motivation for writing this is to solve this contention (at least partially) in a manner that can hopefully move discussions forward in a productive direction rather than lead to two opposing plans of action eventually colliding in the real world with live implementations of conflicting rules.

The current solution landscape as it stands to my understanding is:

BIP 32 hierarchical proofs, as proposed a decade or so ago by Adam Back and recently implemented as a proof-of-concept by roasbeef.
Non-deterministic stateful proofs constructed and timestamped before a deadline, showing a vulnerable ECC key signing off on a quantum-safe authentication mechanism to be used for spending after a post-quantum spending restriction is activated
Tim Ruffing/Tadge Dryja's idea (I apologize, but I forget who was the actual originator of the proposal) of a commit-reveal migration scheme where a transaction spending vulnerable ECC inputs must have an encrypted commitment to that exact transaction confirmed in a block with a pre-determined number of confirmations prior to the decrypted plaintext transaction's confirmation, or the decrypted transaction is invalid.

All three of these proposals leave some subset of coins uncovered. HD proofs are useless for users who generated their keys any other way than BIP 32, stateful timestamped proofs are useless for any inactive user or someone who for any reason does not create them before the creation deadline, and the commit-reveal migration is useless for anyone with an address type that isn't hashed because any attacker would have access to the material needed to create a valid pre-commitment for a spend.

However, by layering them, and allowing any single one of them to be used, complete coverage of any conceivable key generation method can be achieved for hashed address types. Coverage of non-hashed address types is fundamentally impossible, because the requirement to allow use of commit-reveal migration would inherently leave such address types still exposed to a quantum attacker.

Hierarchical proofs cover any BIP 32 user
Stateful timestamped proofs cover any active/observant non-BIP 32 user
Commit-reveal covers any non-BIP 32 user who is inactive/not observant

The only case in which I can see a restriction of hashed address type spends using this layered approach of recovery would leave a user unable to recover their coins is if they have lost access to their private keys. That case is completely outside of the scope of any of these proposals, and inherently impossible to cover.

-------------------------

shinobi | 2026-07-24 00:40:51 UTC | #2

This is just a rewrite of the original message to address some of conduition’s remarks in his reply, and more concisely state the implicit assumptions of the overall idea. I just belted out the original post in a few minutes while I was buried under work, so hopefully this gets across the scope and assumptions more clearly.

There are a number of proposed mechanisms for facilitating the recovery of coins by their legitimate owner by applying new additive encumbrances specifying a new proof of secret knowledge in addition to a signature by a private key, as well as commit-reveal schemes to allow for committing to a transaction spending vulnerable coins and requiring such a commitment to attain a certain number of blocks built upon it before the committed transaction is consensus valid.

None of these schemes is capable of providing complete recovery coverage for all coins whose owners still possess a private key if applied blanketly to a given address type. I believe that by layering multiple recovery schemes together additively, and allowing any single one of them to meet the threshold required for spending, 100% coverage can be achieved by taking on only one new security assumption: the requirement to keep your public key/internal script paths secret.

It should also be possible to account for the fact that most users not running their own full node are leaking master public keys to a third party backend server for balance querying.

To deal with the xpub problem, a new set of derivation paths can be defined for each address type which is specced to only use the Electrum protocol for balancing fetching. By only using individual address queries, you avoid the disclosure of the xpub for these sets of addresses and create a path where for end users a transition can happen potentially even without their awareness. New wallet updates can simply start generating addresses using the new derivation paths and new protocol for balance fetching.

The only complications I can see with this is out-of-box middle-ware that is being used to connect wallets to users’ nodes are built around the assumption of xpubs and using Core’s internal wallet. This is almost certain to be a non-issue in the vast majority of cases as it will be a user connecting to their own node, but even in the small number of cases where a user is connecting to a third party node with such software, it can be adapted to use something like Electrum protocol. I don’t see this being a major show stopper.

The xpub issue mitigated, now the interaction of the of different recovery mechanisms.

## Xpub derivation based recovery:

This will cover any BIP 32 based wallet, regardless of what derivation path (?) is used. So this recovery path should encompass both any address generated using legacy derivation paths with exposed public keys, as well as newer derivation paths securing them through the use of the Electrum protocol.

For any hashed-address type (P2PKH, P2SH, P2WPKH, P2WSH) the derivation proof alone is sufficient, and as conduition pointed out in response to my initial post the internal key can additively be used to make this workable for P2TR addresses that do not use the NUMS point.

This specifically ensures that any user with coins using these address types can produce a recovery proof without having to take any kind of proactive action before the activation of a fork implementing additive encumbrances on address types. (It’s worth noting however these proofs will be pretty big, which is relevant for stateful proofs).

## Stateful timestamped proofs

This has been pitched multiple times before I can recall but never really described in detail. Simply the basic components of a signature from the existing encumbrance condition over a new authentication mechanism (a public key for a quantum safe scheme), and a timestamp to prove that this attestation was produced before some pre-defined deadline chosen to expire before a viable quantum computer exists.

This could be pretty simply boiled down to the 1) the signature over the new public key/authentication commitment, 2) the timestamp. Given the potential size of ZKPs, and the fact that hash-based signatures can be optimized to \~580 bytes, I think these are still worth considering looking at how much smaller and more efficient with blockspace stateful proofs can be compared to ZKPs.

They do require proactive action, but in the event of failing to do so or losing the proofs after the deadline, the possibility of HD recovery and commit-reveal migration would cover users in that case.

## Commit-reveal migration

For any hashed address type, the old commit-reveal migration scheme requiring an encrypted commitment to a transaction be confirmed in the blockchain for a pre-defined number of blocks before the plain-text transaction can be considered consensus valid. The secrecy of public keys can be maintained using the new derivation specification, but this will still be consensus valid for coins in legacy derived addresses.

Again, as conduition pointed out in his reply to my original post, the internal key forms the basis for a secret inaccessible to an attacker, and can be applied as a requirement for P2TR keyspends as an additional validity requirement while using commit-reveal migration. Any tapscript spends using commit-reveal should work as long as those tapscript paths have never been reused with the same internal variables like public keys.

## Wrap up

So I think I’ve covered all my bases in terms of implicit assumptions and responding to conduitions comments.

Unless I am fundamentally missing something, or have overlooked important implementation details like those conduition corrected in his replies from my initial post, I believe this combination of these recovery mechanisms can achieve 100% recovery coverage for every address type live on the network except P2PK outputs and custom bare scripts with the singular new security assumption of maintaining the secrecy of public keys (which as noted above can be accomplished with a rather painless migration behind the scenes for users and without the need to go through the process of generating a new master key and migrating funds across seeds).

So I guess, yeah…what am I overlooking here?

-------------------------

