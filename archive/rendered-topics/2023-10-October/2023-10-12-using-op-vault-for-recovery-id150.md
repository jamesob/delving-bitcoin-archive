# Using OP_VAULT for recovery

AntoineP | 2023-10-12 15:06:22 UTC | #1

I was invited to give a talk about James' OP_VAULT in Amsterdam. Since he's done pretty extensive
communication around the vault usecase already, and since i've been working on Liana for the past
year, i decided to make the talk about (ab)using OP_VAULT for recovery. Figured a more in depth
inquiry than permitted by a talk would be worth sharing with people on this platform.


## What's the value proposition of Liana?

[Liana](https://github.com/wizardsardine/liana) is a Bitcoin wallet which lets one specify
timelocked spending path(s) to be used for recovery purposes.

Here is a list of usecases:
  - Third party safety net. The user has a non-timelocked key like for a regular wallet. But in
    addition a third party has a timelocked key that's only available after -say- one year.
    - As long as they don't let the timelock expire, the user is entirely in control of their
      funds. The third party cannot steal their coins or censor their transactions.
    - In case of a boating accident, they have the option of trusting the third party to send them
      back their coins. (TTP can be an uncle Jim for instance.)
  - Inheritance. Essentially the same as the safety net, with possibly more than one heir. For
    instance, the user has their key, non-timelocked, like a regular wallet. Spouse, child and
    notary each have one key. Spouse and child can spend together after 1y. 2 of spourse, child and
    notary can spend after 1y3mo in case they couldn't come to an agreement.
  - "More secure" backups. The typical Bitcoin user would have one extended key, generated from
    mnemonics. The key would be used through a signing device and the mnemonic backed up somehow.
    There is a tradeoff between accessibility and security for the backup. You don't want to put
    them in your kitchen's drawer but still want to be sure you can access them when shit hits the
    fan. The issue here obviously is that one access to your backups and your funds are gone. No
    time to react.
    Instead you could use a non-timelocked key that you *don't backup* in your signing device, and
    backup the timelocked key. If the backup is tamper-evident, regularly checking backups lets you
    time to react to a theft without risking your funds. It's not perfect but seems to be a strict
    improvement: worst case if your coins expire you're back to the current situation where the backup
    lets you spend the coins immediately.
  - Decaying multisig. As time passes, threshold for the required number of signature decreases, or
    more keys are introduced:
      - Instead of having to bake recovery in the primary, immediately available, spending path as
        today's multisigs you can aim for a larger threshold and only decrease it as time passes (ie
        only if you lose access to one of the keys).
      - You can reduce the threshold and introduce more keys as time passes, to make sure funds are
        never lost.


## How can it be improved by leveraging OP_VAULT?

Liana is an existing wallet for today's Bitcoin. As such it doesn't use any form of covenant, just a
simple `OP_CSV`. This means time starts ticking for the recovery path when you receive a coin. This
in turn means user would have to use their wallet regularly or "refresh" their coins every so often
if they don't want (some of) the recovery path(s) to become available.

Not a deal breaker, but it certainly does add some friction to the UX. It seems it would be better
to be able to trigger a "recovery process" and only then have the time start ticking. That's what
OP_VAULT permit. Roughly speaking just use an OP_VAULT in the script to receive coin with the "leaf
script" (in OP_VAULT BIP terminology) being the recovery script.

Another improvement over today's Liana is how the OP_VAULT recovery would presumably be triggered on
all coins at the same time and the recovery path would become available at the same time for all
coins to be recovered. That's not the case in today's Liana since not all coins were received at the
same time.

It's not a silver bullet, since you still need to check every so often whether a recovery wasn't
triggered. But you don't need to make a transaction when doing so. That's less burden, less cost and
less potential privacy footgun.

Let's go through the use cases listed above to see how we would implement them using OP_VAULT.

## Safety net / "More secure" backups

This is the simplest case. 1 primary key (the owner of the coins), 1 recovery key (the backed up key
or the third party key).

Today's Taproot tree:
  - internal key is owner's
  - `<recovery key> CHECKSIGVERIFY <365 * 144> CSV`

OP_VAULT Taproot tree:
  - internal key is owner's
  - `<recovery key> CHECKSIGVERIFY 0 0 <<recovery key> CHECKSIGVERIFY <365 * 144> CSV> OP_VAULT`

Note the owner of the coins can always clawback any triggered recovery attempt (within 1y of it
being triggered) since the internal key is retained.

## Inheritance

This one's still pretty simple, but it's got a multisig and 2 different timelocks which raise
interesting questions.

Today's Taproot tree:
  - internal key is owner's
  - `<spouse key> CHECKSIGVERIFY <child key> CHECKSIGVERIFY <365 * 144> CSV`
  - `<child key> CHECKSIGVERIFY <notary key> CHECKSIGVERIFY <455 * 144> CSV`
  - `<notary key> CHECKSIGVERIFY <spouse key> CHECKSIGVERIFY <455 * 144> CSV`

There is two ways this can be translated to using OP_VAULT. The first, simplest way would be to use
3 `OP_VAULT` branches like so:
  - internal key is owner's
  - `<spouse key> CHECKSIGVERIFY <child key> CHECKSIGVERIFY 0 0 <<spouse key> CHECKSIGVERIFY <child key> CHECKSIGVERIFY <355 * 144> CSV> OP_VAULT`
  - `<child key> CHECKSIGVERIFY <notary key> CHECKSIGVERIFY 0 0 <<child key> CHECKSIGVERIFY <notary key> CHECKSIGVERIFY <455 * 144> CSV> OP_VAULT`
  - `<notary key> CHECKSIGVERIFY <spouse key> CHECKSIGVERIFY 0 0 <<notary key> CHECKSIGVERIFY <spouse key> CHECKSIGVERIFY <455 * 144> CSV> OP_VAULT`

However this has the downside that any party may continuously re-trigger the recovery, not letting
any timelock expire. In addition, one timelock expiring does not lead to the following expiring `tl2- tl1`
blocks later. If the first timelock path is chosen, any attempt to use the second path needs to
restart all over again.

This can be fixed by using a clumsier Taproot tree:
  - internal key is user's
  - Let `recovery_script` be
  ```
  <spouse key> OP_CHECKSIG OP_SWAP <child key> OP_CHECKSIG OP_ADD OP_SWAP OP_IF
    0
  OP_ELSE
    <144 * 365> OP_CHECKSEQUENCEVERIFY OP_0NOTEQUAL
  OP_ENDIF
  OP_ADD OP_SWAP OP_SIZE OP_0NOTEQUAL OP_IF
    <notary> OP_CHECKSIGVERIFY <144 * 455> OP_CHECKSEQUENCEVERIFY OP_0NOTEQUAL
  OP_ENDIF
  OP_ADD 3 OP_EQUAL
  ```
  - Single tapleaf is `<spouse key> CHECKSIG <child key> CHECKSIGADD <notary key> CHECKSIGADD 2 EQUAL 0 0 <recovery_script> OP_VAULT`

Now there is a single shot so you want any two keys from the set of 3 to be able to trigger a
recovery. There is no way to keep restarting it, if the spouse and the child don't get along it will
automatically decay to including the notary as signatory.

In this specific example it's probably fine to use the nicer version with multiple `OP_VAULT`
leaves because there is no reason for any two parties to shoot themselves in the foot. It's not
always the case however.

## Decaying multisig

Now let's look at a decaying multisig for a 5-stakeholders organization. Start from a 3of5 multisig
between all the stakeholders. In case 1 of them loses their key, after 1 year it becomes a 3of6 with
a notary. After a year and 3 months, it becomes a 2of6 between these same persons.

The recovery script is then:
```
OP_IF
  <stk1> CHECKISG <stk2> CHECKISGADD <stk3> CHECKISGADD <stk4> CHECKISGADD <stk5> CHECKISGADD <notary> CHECKISGADD 3 OP_EQUALVERIFY <144 * 365> OP_CHECKSEQUENCEVERIFY
OP_ELSE
  <stk1> CHECKISG <stk2> CHECKISGADD <stk3> CHECKISGADD <stk4> CHECKISGADD <stk5> CHECKISGADD <notary> CHECKISGADD 2 OP_EQUALVERIFY <144 * 455> OP_CHECKSEQUENCEVERIFY
OP_ENDIF
```
(That's `or_i(and_v(v:multi(3,stk1,stk2,stk3,stk4,stk5,stk6),older(144 * 365)),and_v(v:multi(3,stk1,stk2,stk3,stk4,stk5,stk6),older(144 * 455)))`.)

Which can be optimized to (while keeping miniscript compatibility):
```
<stk1> CHECKISG SWAP <stk2> CHECKSIG ADD SWAP <stk3> CHECKSIG ADD SWAP <stk4> CHECKSIG ADD SWAP <stk5> CHECKSIG ADD SWAP <notary> CHECKSIG ADD SWAP
OP_CHECKSIG OP_ADD OP_SWAP OP_IF
  0
OP_ELSE
  <144 * 365> OP_CHECKSEQUENCEVERIFY OP_0NOTEQUAL
OP_ENDIF
OP_ADD OP_SWAP OP_IF
  0
OP_ELSE
  <144 * 455> OP_CHECKSEQUENCEVERIFY OP_0NOTEQUAL
OP_ENDIF
```
(That's `thresh(4,pk(stk1),s:pk(stk2),s:pk(stk3),s:pk(stk4),s:pk(stk5),s:pk(stk6),sln:older(144 * 365),sln:older(144 * 455))`.)

Taproot tree is:
  - key path spend is a NUMS
  - `<stk1> CHECKISG <stk2> CHECKISGADD <stk3> CHECKISGADD <stk4> CHECKISGADD <stk5> CHECKISGADD <notary> CHECKISGADD 2 OP_EQUALVERIFY 0 0 <recovery script> OP_VAULT`


## Discussion

OP_VAULT seems like a good fit for this usecase, although a more Taprooty mechanism to add/remove
branches from the tree would be preferable. It'd avoid having to use bulk scripts in a single leaf
to avoid having to re-trigger a recovery simply to use another path, or avoid being stuck with a
perpetually re-triggered recovery. The latter may be mitigated by using longer timelocks than what's
in any the recovery script in every OP_VAULT leaf.

-------------------------

stevenroose | 2023-10-13 10:47:37 UTC | #2

(Dude this post looks like it's been formatted with rustfmt. One word per line, really?)

> In this specific example it’s probably fine to use the nicer version with multiple `OP_VAULT` leaves because there is no reason for any two parties to shoot themselves in the foot. It’s not always the case however.

How about the spouse and the kid both paying the notary to keep resetting because they want all the money? :D Inheritance is a game theory nightmare. Jokes aside, I agree with the equivalence :) I thought at some point that the semantics were different in another way, that if spouse and kid trigger the unvault and then spouse dies that the money would be locked, but OP_VAULT retains a re-vault, I suppose.

-------------------------

