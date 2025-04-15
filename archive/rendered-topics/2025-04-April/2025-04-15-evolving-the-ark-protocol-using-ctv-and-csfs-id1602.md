# Evolving the Ark protocol using CTV and CSFS

stevenroose | 2025-04-15 11:54:40 UTC | #1

Last weekend at OP_NEXT I outlined a novel variation of Ark that removes all need for user interacty in rounds, dubbed Erk. In this post, Erik De Smedt and I want to formalize Erk and some other improvements on the protocol that leverage CTV ([`OP_CHECKTEMPLATEVERIFY`](https://github.com/bitcoin/bips/blob/c5220f8c3b43821efa3841a6c5e90af9ce5519e8/bip-0119.mediawiki)) and CFSF ([`OP_CHECKSIGFROMSTACK`](https://github.com/bitcoin/bips/blob/c5220f8c3b43821efa3841a6c5e90af9ce5519e8/bip-0348.md)).

As a TL;DR, we came up with ***two*** alternative protocols for Ark, which have the following properties:

* "Erk":
  - no user interactivity for rounds
  - offline refresh (server can refresh for a user)
  - perpetual offline refresh (server can keep refreshing for a user indefinitely)
  - works best with single input and single output vtxo
  - requires CTV + CSFS
* "hArk":
  - no user interactivity for rounds
  - very efficient, also for multiple inputs
  - no offline refresh possible (so far)
  - requires CTV only

# A primer on Ark

Below we will briefly summarize the existing ctv-based Ark and MuSig-based clArk protocols, followed by a detailed explanation of the new Erk and hArk variants.

First, we summarize some of the core concepts that we will use in the subsequent explanations.

We denote the Ark server's public key with $S$ and users' public keys with $A$, $B$, ...

The basic building block is a transaction tree that allows users to share an onchain UTXO. The sketch below shows a transaction tree where Alice ($A$) owns 4 bitcoin, Bob $B$ owns 3 bitcoin, Caroll $C$ owns 2 bitcoin and Dave $D$ owns 1 bitcoin. Ark is an off-chain protocol, so in the happy path only the funding transaction will ever be confirmed on chain.


```text
(each box is a bitcoin transaction)

                 +------------+
                 |  FUNDING   |
                 +------------+
                       | 10₿
                 +-----+------+
                 |   NODE 1   | 
                 +-----+------+
             +---------┴--------------+
          7₿ |                        | 3₿
        +----+----+              +----+----+
        | NODE 2  |              | NODE 3  |
        +----+----+              +----+----+
             |                        |
       +-----+-----+            +-----+-----+
    4₿ |        3₿ |       2₿   |           | 1₿
    +---+----+  +---+----+   +---+----+  +---+----+
    | EXIT A |  | EXIT B |   | EXIT C |  | EXIT D |
    +--------+  +--------+   +--------+  +--------+
```

In this setup, Alice can perform a unilateral exit by broadcasting her branch of the tree to claim her funds. Her branch here consists of the txs _FUNDING_, _NODE 1_, _NODE 2_ and _EXIT A_. We say that Alice owns a vtxo when she knows these transactions (and required witness data) and can spend for pubkey $A$.

Ark vtxos have an expiry time, before which each vtxo has to be either spent or refreshed. Refreshing a vtxo essentially means swapping it for a new one with a fresh expiry time. This happens during Ark rounds, an interactive process between the Ark server and its users.

After expiry, the server can sweep the tree. In the ideal case, the server can directly sweep the funding tx, but if some users have performed a unilateral exit, it can sweep all the remaining nodes of the tree that have not been exited.


### Arkoor

If Alice holds a vtxo and wants to pay Bob, she can create a new off-chain tx, called an _arkoor tx_, that spends her vtxo, i.e. the output of her exit tx, and creates a new vtxo for Bob. Bob can now store all the txs that made up Alice's vtxo, plus the arkoor tx, as his new _arkoor vtxo_.


### Output policies

We identify three different output policies used in Ark protocols:

* ***node policy***: the policy used in all node txs, including the funding tx
* ***leaf policy***: the policy of the final node output, paying into an exit tx
* ***exit policy***: the policy used in the exit tx, which is the tx at the leaf of the tree and is used by the user to perform a unilateral exit

Logically, one can think of these three policies following each other. Below is a diagram indicating the output policies used in some of the different txs used above. The exit tx can be followed by either arkoor (out-of-round) spends or forfeit spends.

```text
FUNDING
+-------------+-----+
| node policy | 10₿ |
+-------------+-----+

NODE 1
+-------------+-----+
| node policy |  7₿ |
+-------------+-----+
| node policy |  3₿ |
+-------------+-----+

NODE 2
+-------------+-----+
| leaf policy |  4₿ |
+-------------+-----+
| leaf policy |  3₿ |
+-------------+-----+

EXIT A
+-------------+-----+
| exit policy |  4₿ |
+-------------+-----+
```


All variants have the following properties in common:

* Every round has an ***expiry height $T_{exp}$***, after which the funds can be swept by the Ark servers key.
 
* Both the _node policy_ and _leaf policy_ function to continue the tree branch of txs and have an alternative spend path that allows the server to sweep after the expiry.
 
* The liveness requirement for a user is that they should come online a certain time before $T_{exp}$, either to refresh their vtxos, or to perform a unilateral exit if the server or any previous owner of the arkoor vtxo misbehaved.

* Once the exit tx is confirmed, the expiry is averted and the funds enter the _exit policy_. The exit policy has two main functions and corresponding _clauses_:

  - the ***spend clause*** provides a way for the user and the server to collaboratively create immediate spends (used for arkoor txs and forfeit txs)
  - the ***exit clause*** provides a way for the user to access the funds if no immediate spends have been made.
  
  Initial designs of the _exit policy_ used a relative timelock in the _exit clause_, but all our newer designs combine both an absolute timeout $T_{exp}$ and a relative timelock $\Delta t$.
  - The relative timelock ensures that the _spend clause_ takes precedence over the _exit clause_.
  - The absolute timelock ensures that any subsequent owner of this vtxo (who might have received the vtxo through an arkoor tx using the _spend clause_) has no further liveness requirement than to come online some time before $T_{exp}$.



# Some variants in more detail

## Ark

The initial design of Ark was based on CTV. Our latest iteration on the idea looks as follows.

Both the _node policy_ and the _leave policy_ use `OP_CTV` with a commitment to the next tx.

* _node policy_ and _leaf policy_: $CTV \space or \space (S + T_{exp})$
* _exit policy_: $(A + S) \space or \space (A + T_{exp} + \Delta t)$

During rounds, users will sign a _forfeit tx_ which forfeits their vtxo to the server and is conditional on the correct termination of the next round. A _connector_ is a special-purpose descendant of the next round's funding tx, so when used as an input to the _forfeit tx_, it ensures the server can only enforce the forfeit if the next round succesfully confirmed. Once all users have signed forfeit txs for their input vtxos, the server can sign the funding txs and broadcast it to the network.

Forfeit txs take two inputs:
- the output of the exit tx, using the _spend clause_ of the _exit policy_
- the connector input which is an output resulting from a series of descendant txs from the new round's funding tx.

Forfeit txs have one output, sending all the money to the server.

```text
forfeit tx:
| inputs    | outputs | 
+===========+=========+
| exit tx   |       S | 
+-----------+---------+
| connector |         | 
+-----------+---------+
```

## clArk, "covenant-less Ark"

clArk is very similar to Ark, but uses recursive multisigs in the tree. Each _node policy_ contains a multisig with all the public keys of all the leaves below it.

* _node policy_: $(A + B + C + ... + S) \space or \space (S + T_{exp})$
* _leaf policy_: $(A + S) \space or \space (S + T_{exp})$
* _exit policy_: $(A + S) \space or \space (A + T_{exp} + \Delta t)$

The round mechanism is identical and uses _connectors_ as well. The only difference is that the coordination requires an extra phase in which all clients sign (their branch of) the tree.

## Erk, "Ark, by Erik"

We made some additional improvements to Erk after I presented it at OP_NEXT.

The policies in Erk are identical to the policies in Ark, but we make the signatures ***rebindable*** (rebindable signatures are signatures with APO semantics, meaning they don't commit to the input utxos so that they can be used to spend from any output containing the same pubkey). We can use CTV+CSFS to implement rebindable signatures.

* _node policy_ and _leaf policy_: $CTV \space or \space (S + T_{exp})$
* _exit policy_: $(A + S) \space or \space (A + T_{exp} + \Delta t)$

It is the $A+S$ part in the _exit policy_ that we will make rebindable.

The core and fundamental principle of Erk rests on the following tx, which we call a _refund tx_.

```text
refund tx:
| inputs             |        outputs | 
+====================+================+
| old exit tx for A  | exit policy A' | 
+--------------------+----------------+
| new exit tx for A' |              S | 
+--------------------+----------------+
```

Erk rounds have no user interaction (in the sense that users have to take actions synchronously together). Users sign up for a round by sending a new pubkey ($A'$) and a signed refund tx to the server. Note that one input of the refund tx is signed with the vtxo's current pubkey $A$ and the other input with the newly provided pubkey $A'$.

At that point, the server can safely create a new vtxo tree, issuing the same vtxo (minus some fee) for the user. This new vtxo has an _exit policy_ with the user's new key $A'$.

Users can safely sign this refund tx at any time, because they still receive their money in one of its outputs. So if these txs would be used by the server maliciously, users won't lose any money.

The server can safely create the round after having received signed refund txs for all inputs because in the case the user tries to (maliciously) exit the original old vtxo, the server can unroll the new vtxo and spend both vtxos together, taking the part that the user has no right to.

### Arkoor

Care needs to be taken to implement arkoor txs in the Erk variant. Imagine if user Alice sends their new vtxo to Bob using an arkoor tx, and then maliciously tries to exit the old vtxo. The server will be forced to broadcast the new vtxo and use the refund tx, invalidating Bob's arkoor vtxo.

However, since both the tree leaf exit tx and the refund tx have the same output policy for $A'$, the rebindable arkoor signatures that Bob received from Alice can be applied to both possible outputs. (Note that always only one of them can possibly exist at the same time, because the refund tx is a child of the exit tx.)

This design for arkoor txs for Erk works well in cases where each new vtxo comes from a single input vtxo. When we try to generalize this to work for multiple inputs being consolidated into a single output vtxo, the arkoor constructions become unworkable.

### Offline refresh

Now, imagine if the user already pre-signs a refund tx for his vtxo right after a round, hands the signatures to the server and then goes offline. This means that the server can at any time safely re-issue the vtxo, holding the refund tx that guarantees the user can never claim both. This means that if a user forgets to manually refresh, the server can do it automatically. It is also possible for watchtowers to monitor the Ark and notify an Ark user in case the server is not refreshing their vtxo.

Furthermore, imagine if one refresh changes the key $A$ to $A'$, Alice can _also_ already pre-sign a refund tx for the future vtxo $A'$, which refreshes into $A''$! In fact, Alice can already pre-sign any number of future refreshes, each into a new key (and probably each subtracting a small amount from the vtxo's value as fee). This way a server can perpetually refresh a vtxo without the user ever having to come online unless a watchtower warns it that the server is not acting as promised.

Possibly a malicious server could try to apply all the refund txs in rapid succession in order to claim the fees (note that this is unlikely to be economical because he has to pay mining fees for each tx). This can be avoided if each refund tx has an absolute locktime that ensures the server can only use the refund txs at points in the future that match the predicted round times.


## hArk, "hash-lock Ark"

Erk seems quite awesome (though we discarded aArk as a possible codename), but still has two main drawbacks:

- In order for the server to "retaliate" a malicious exit, he has to unroll the entire new vtxo's tx tree branch. This is particularly problematic for onboard vtxos (vtxos that are created by users themselves when moving new money into the Ark). Onboard vtxos consist of only a funding tx that directly has the _leaf policy_ and an exit tx on top. This means that the cost for a user to attempt a malicious exit is low, while the cost for the server to retaliate is high.
- Erk is only workable for single-input refreshes.

hArk is not a variation of Erk, but an entirely different variant of Ark. It also doesn't require any user interaction for rounds, it's footprint-minimal and works perfectly for multiple inputs.

In hArk, the _leaf policy_ is different from the _node policy_, in that it requires a secret to be known to use.

* _node policy_: $CTV \space or \space (S + T_{exp})$
* _leaf policy_: conceptually $(CTV + \text{secret}) \space or \space (S + T_{exp})$, but alternatively $(A + S + \text{secret tweak}) \space or \space (S + T_{exp})$ to be able to use keyspend which is more efficient
* _exit policy_: $(A + S) \space or \space (A + T_{exp} + \Delta t)$

During rounds in hArk, users submit the input vtxos they want to refresh. The server generates a secret for each new vtxo in the tree. Since only the server knows the secret initially, it can safely fund the round. Initially none of the new vtxo will be accessible by its users because all secrets are still secret.

After the round funding tx has confirmed, users sign a _forfeit tx_. The forfeit tx looks different than Ark and clArk forfeit txs. It allows the server to claim the input vtxo by revealing the secret.

The output policy of the forfeit tx is $(S + \text{secret}) \space or \space (A + \Delta t)$. Naively the revealing of the secret can be implemented using a simple hash preimage, but possibly an adaptor signature can be used in the keyspend path to save some bytes.

```text
forfeit tx:
| inputs  |    outputs | 
+=========+============+
| exit tx | S + secret |
|         | or  A + Δt |
+---------+------------+
```

After the user has signed this forfeit tx, the server tells the user the secret. This way, the user has access to its new vtxo.

If the server refuses to hand over the secret after the user signed the forfeit tx, the user is required to perform a unilateral exit attempt, which either gives it its money back or forces the server to reveal the secret onchain.


## Conclusion

Having CTV available allows us to fully eliminate all user interactivity during Ark rounds. Refreshing becomes an entirely asynchronous process where users can sign up, the server can independently issue new vtxos and the users later come back to finish the process.

Having CSFS available on top of CTV, allows for a form of offline refresh that can even be used recursively. This allows the server to refresh without the user's presence and it allows the user to delegate monitoring for correct behavior for its vtxos to a third party (watchtower).

Both Erk and hArk can be used together. Users can always pre-emptively sign Erk refund txs just in case they don't show up, but still manually perform rounds using hArk, for example to consolidate multiple vtxos into a single one, or simply because they prefer to refresh sooner.

We believe the above schemes greatly enhance the user experience for Ark, its viability in the mobile setting and the functionality that can be offered by Ark servers.

-------------------------

instagibbs | 2025-04-15 14:46:45 UTC | #2

This is getting extremely close to the classic Timeout Trees, though I've always said they were already similar. I think the analogue here is that you could build a Timeout Tree where instead of routing funds through the LN to move to a new tree, you can "refresh" the vUTXOs directly, transporting the channel to the new Ark. 

All without connector or control inputs.

Loving the interactivity reductions perhaps the most here.

-------------------------

