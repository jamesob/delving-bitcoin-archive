# Persisting Mutable Storage Inside The "T"EE

ZmnSCPxj | 2025-10-07 17:50:26 UTC | #1

Title: Persisting Mutable Storage Inside The "T"EE

# Introduction

A Trusted Execution Environment is a scam where a hardware manufacturer,
in coordination with a cloud service provider, convinces victims that
they can trust somebody else’s computer to actually run normal code,
without use of homomorphic encryption, exactly as the victim intends
the normal code to be run.
This is done by distracting the victim with a certificate authority
attesting to a secret key hidden in the hardware, which (1) the hardware
manufacturer claims it does not know, despite having placed it in the
hardware in the first place, and (2) the cloud service provider claims
not to have extracted the secret key from the hardware, despite having
direct physical possession of the hardware, **and** the ability to
virtualize anything, including execution of said hardware with its
supposedly-not-extracted secret key.

All such existing "T"EE scams have an additional flaw that the scammers
also distract from their victims: they do not have persistent mutable
storage.
Any persistent mutable storage ***MUST*** be external to any "T"EE, and
***MUST*** be trusted to not perform “rollback attacks”.

A “rollback attack” is simply that a persistent mutable storage,
running outside the "T"EE, can keep backup copies of old state, and
recover the old state.
When the "T"EE application is restarted, the persistent mutable storage
recovers from the backup old state.

Rollback attacks are disastrous for many Bitcoin applications:

1. If the "T"EE application is a Lightning node, if it is given old
   state, it can be convinced to publish a unilateral close on old
   state, which is indistuingishable from a theft attempt.
   The peer can then get all the funds in the channels, effectively
   stealing by claiming the victim *is* the real thief.
2. If the "T"EE application is a signer for a statechain scheme,
   old states can effectively retain supposedly-deleted keys.
   There is ***NO*** possible proof-of-ignorance that the statechain
   signer can present to prove that there are no backup copies of
   old “deleted” keys, because ***by itself*** it cannot even know
   if it is a victim of a rollback attack.

Note that encrypting the stored data “at rest” ***DOES NOT*** protect
against rollback attacks — the rollback simply requires that old
data be available to the attacker, not that the attacker can decrypt
it.

In this writeup, I will show how to synthesize a persistent mutable
storage that can be audited to not be rollback-capable, using
multiple "T"EEs running the same program *in addition to* the main
"T"EE application they serve.

# All Things Must Pass

The key insight is this:

* Even an HDD is ephemeral and not persistent.
  One day, it, too, shall die and all its stored data lost —
  forever.
* Yet a ZFS array containing that HDD is permanent.
  You simply hotswap the dead HDD with a fresh one, and let ZFS
  keep on going on, because ZFS is ***awesome***.

Thus, from ephemeral storage, persistence is achievable: this
HDD, too, shall come to pass, but the end of the ZFS array is not
yet.

## A Persistent Array Of "T"EE Ephemeral Memories

We can consider that “even if an HDD is so big and strong and
tough, one day it will stop being able to read anyway”, then
perhaps the dinky little ephemeral RAM of a "T"EE can serve
just as well in an array of "T"EE ephemeral RAMs?

The only requirement then is that we distribute the "T"EEs
widely, so that it is unlikely that so many of them go down
that permanent data loss is possible.
We then use suitable erasure coding so that we can recover
even if some number of "T"EEs go down and lose the contents
of their ephemeral RAMs.

Note that part of the "T"EE scam is to convince victims that
the actual RAM used is *also* encrypted with the hardware
attestation key.
Thus, there is no need to encrypt data at rest, as the "T"EE
hardware claims it *already* encrypts the data in memory, and
our “persistent” storage is in fact the RAM inside the "T"EE.
We do still need to encrypt data in transit between the main
"T"EE application and all of the "T"EE storage programs.

Thus, we have multiple "T"EE storage programs, which the main
"T"EE puts in a resilient array, achieving persistence on top
of ephemeral storages.

### A Digression On Terminology

Traditionally, discussions about redundant arrays of
inexpensive storage devices use the term “block devices” that
store and retrieve “blocks” of data.

Unfortunately for our context — cryptocurrency network
applications — “block” is much too overloaded a
word:

1. Bitcoin blocks of transactions.
2. Symmetric encryption schemes working on blocks of data.
3. Network communications getting blocked.
4. Using blocking vs nonblocking APIs on network sockets.
5. IP (Internet Protocol) blocks.
6. IP (Intellectual Property) blocks which prevent some
   erasure coding schemes from being adopted.
7. Block, the company that funds Spiral, and which itself
   sells various Block devices such as Square terminals.

Thus, I will use old, obsolete terminology that nobody uses
today:

* We have “disks”, which are storage devices, instead of
  those newfangled “block devices”.
  * It does ***NOT*** matter whether it is spinning rust,
    trapped electrons in capacitors, or DRAM that we misuse
    as “persistent” storage in the disk array.
    I will call them all “disks” nevertheless.
* The storage device is divided into “sectors”, instead of
  those newfangled “blocks”.

# Sector Size

Traditionally, disks presented 512-byte sectors (even more
traditionally, 128-byte sectors in single-density floppies
— is my age showing yet).
However, modern disks now present 4096-byte sectors; some
flash-based disks even want to present larger sectors, up to
32768 bytes.

Unfortunately, as we intend to use multiple "T"EEs as disks,
we have to connect them via Internet Protocol, most likely
via TCP or TLS (itself running on TCP).

And generally, the common MSS for TCP is about \~1400 bytes.

Both the 512-byte and the 4096-byte options just do not
fit well with the MSS of \~1400 bytes.
512 bytes would require us to `TCP_NODELAY` to deliver the
sector data ASAP with a smaller-than-max packet, while 4096
bytes would fragment the data into multiple packets.

Ideally, our sector size would be 1024, as that just about
fits the MSS for TCP, and can fit a little more data, such
as MACs for transmission integrity as well as any IVs for
encryption for in-transit data.

Unfortunately, if we were emulating a disk in the main
application "T"EE (actually an array of multiple remote "T"EE
disks), that we then pass to a “real” filesystem like XFS (or
directly to some database engine that works on disks instead
of filesystem), then those will usually assume sector sizes
of 4096 bytes (and might be configurable to use sector sizes
of 512 bytes).
Even if we work with 1024-byte sectors when communicating
with the remote "T"EE disks, the overlying file system will
attempt to write 4096-byte sectors which would require
multiple packets anyway, or 512-byte sectors which would
require read-modify-write operations on the main application
"T"EE and add *more* overhead on each 512-byte write.

So, we should stick with normal 4096-byte sector sizes,
and just accept that it will take 3 packets in general to
transmit them plus any cryptography overhead for integrity
and encrypted-in-transit requirements.

# RAID5 Write Hole

The RAID5 write hole happens due to the fact that writing
to multiple disks in parallel is ***NOT*** an atomic
operation.

It’s possible that the application is able to send out
data to *some* disks, then crash, with only part of a
stripe of data being written.
This leaves the on-disk data, *together with the extra
parity data needed by erasure coding*, inconsistent,
with no real way to determine which data is “real”.
Partial stripe writes are particularly bad because
if a RAID5 write hole occurs and *then* a disk is lost,
the recovered data for the lost disk can be completely
different from *both* the “old” data and the “new” data!

This is called the “RAID5” write hole simply because
it was *already* observed when using erasure coding
schemes where at most one disk can be lost, i.e. RAID5
XOR parities.
However, it applies to *all* uses of erasure coding,
including RAID6 with 2 parity disks or “generalized
RAID6” where there are 3 or more parity disks.

Fortunately, since this is a novel array scheme
anyway, we can fix the RAID5 write hole by simply
changing the disk interface (this is not possible with
“real” spinning-rust or electrons-in-capacitor disks,
of course):

* There is no “write to sector” command.
* There are instead three commands:
  * “provisional write to sector”, where the disk
    stores both the old version of the sector, and
    the new data for the sector.
  * “commit provisional write” where the disk discards
    the old version of the sector and keeps the new
    data.
  * “rollback provisional write” where the disk
    discards the new version of the sector and keeps
    the old data.
* At most one provisional write can be in-flight.
  If the disk is already storing a provisional write
  for one sector, and another provisional write (to
  any sector, not just this one) is done, then the
  command errors.
  * This means we only need to sacrifice one sector of
    expensive RAM for the provisional write scheme.
* On reads, the disk reports both the old and new
  data for sectors with a pending provisional write.
* There is also a status command to check if the disk
  has a provisional write for a sector, as well as to
  report the sector index for the sector in provisional
  write.

When the array-management code wants to update *one*
sector in the array:

1. It reads all sectors in the same stripe (this can be
   cached).
2. It calculates the new parity data for the new data.
3. It sends out provisional write requests to the
   destination storage disk and each of the parity disks.
4. It waits until all disks have either responded, or
   timed out.
   * In case of a time out, it permanently records that
     disk as “dead”.
     This record can be placed in a special sector of
     all disks, with a strict monotonic version number
     seen by the disk, and RAID1-replicated across all
     disks, which is always directly replaced if the
     replacement has a monotonically higher version
     number.
     If the highest-versioned death record marks a disk
     as dead, then the array code never reads or writes
     to that disk ever again, treating all writes as
     successes and all reads as recoverable using
     erasure code recovery of the remaining disks, and
     reporting the degraded state to the operator.
   * In particular, if there are N storage disks and P
     parity disks, and we are updating ONE sector on a
     storage disk, we are writing to P + 1 disks, and
     up to P of them can time out and be marked as
     dead, and the array will still consider itself as
     “degraded but hanging on”.
5. It sends out commit commands to the destination
   storage disk and each of the parity disks (at least
   the live ones that did not time out in step 4).
6. It waits until all disks have either responded, or
   timed out.

The array code may crash at any time, and on restart,
has to check for pending provisional writes.
It looks for the highest-versioned death record to
determine which disks it still considers alive, and
then checks for pending provisional writes.

* If the array crashed between steps 3 and 4, some
  writes may have been received by the disk, but some
  disks might not have received them.
* If the array crashed between steps 4 and 6, some
  commits may have been received by the disk, but
  some disks might not have received them.

The array thus needs to differentiate between the above
two cases, by checking if taking the new version as
true leads to a consistent parity.
If the new version leads to a consistent parity, it
means it got past step 4 and can commit everything.
If the new version does not lead to a consistent parity,
it means it was in between step 3 and 4 and has to
roll back everything.

# XOR Optimization

Some erasure coding schemes have an XOR optimization
property:

* In case of a partial stripe write, instead of reading
  the unwritten storage sectors of the stripe, we can
  read the *written* storage sector(s), XOR the old
  version and the new version, set all the *unwritten*
  storage sectors in-memory as zero, then run the
  normal parity-calculate code.
  The resulting parities can then be XORed with the
  existing parities-to-be-overwritten to result in a
  consistent stripe of the new version.

The XOR optimization above means we do not have to
read the entire stripe to write a few sectors;
assuming N storages and we want to write W sectors
of a stripe, if W < N - W then doing the XOR
optimization means reading fewer sectors.

We can thus add a special provisional-write command
that, instead of taking the new version of the data,
takes a “delta” sector.
This provisional-write-delta command then reads the
old version sector data, XORs the delta sector,
and the result is the “new” sector for purposes of
the provisional write.

The above provisional-write-delta command is used for
writing to parity sectors when using the XOR
optimization.

# Generalized RAID6

The free and open source world currently has a 2-parity
erasure code scheme implemented in ZFS, with similar
code written in the Linux kernel (for `md`), using
`PSHUFB` SIMD instruction.
The performance is generally pretty good ***if*** the
processor has `PSHUFB`.
However, for 3-parity and above, even the `PSHUFB`
SIMD instruction cannot be used to speed up the
parity calculation and erasure recovery code for the
third parity sector.
RaidZ3 simply accepts the efficiency loss, while the
Linux kernel `md` simply does not support more than 2
parities.

However, given how much more fragile our disks are in
this scheme, we really want to have a nice efficient
generalized erasure coding scheme that supports much
more than 2 parities.
Unfortunately, faster erasure coding schemes are
patent-encumbered, i.e. blocked by IP law.

We *do* have a generalized scheme, based on Reed
Solomon codes and Galois Field math.
This is *usually* very slow since Galois Field math
often requires moving individual bits around, and
also not very amenable to SIMD, but because of this,
it was never patent-encumbered.

And then there is this [XOR-based EC paper](https://www.researchgate.net/publication/2643899_An_XOR-Based_Erasure-Resilient_Coding_Scheme).

The key insight of this paper is:

* Traditionally, we have been using bytes as
  `GF(2^8)` field elements.
* However, we can rotate our view:
  * One byte is actually 8 bits from 8 different
    `GF(2^8)` field elements.
  * So for example, byte index 0 is actually the
    0th bit of 8 different `GF(2^8)` elements,
    byte index 1 is actually 1st bit of the same
    8 different `GF(2^8)` elements… byte index
    7 is actually 7th bit of the same 8 different
    `GF(2^8)`elements.
  * Thus, XORing a byte is actually an 8-wide
    SIMD instruction that works on individual bits
    of multiple `GF(2^8)` elements at a time.

The above rotation of our view means that even
\*\*\*non-\*\*\*SIMD reads, XORs, and writes of bytes,
words, doublewords, or quadwords, are actually
semantically the same as SIMD reads, XORs, and
writes of individual bits of multiple `GF(2^8)`
elements at a time.

This creates a *generalized* Reed Solomon code
using Galois Field math that **is not slow**, and
yet remains patent-unencumbered due to Reed Solomon
being traditionally viewed as based on really
slow-for-computers math.

Scouring the githubs, I found an [abandoned project](https://github.com/raid5atemyhomework/llrfs)
that implements the paper above.

While abandoned, it seems to have completed the code
that calculates parity and erasure recovery (including
tests), and that is what we really want anyway.

It claims that in the 2-parity case, the \*\*\*non-\*\*\*SIMD
version matches the `PSHUFB` SIMD implementation in
ZFS in performance.
The code apparently can be automatically vectorized by
GCC to use simple SIMD instructions, and wide XORs
inferred by GCC with `-ftree-vectorize` (and which are
very common in most processors these days, including
ARM) speed up the performance even more, beating the
ZFS `PSHUFB` implementation (which can be run on fewer
processors) handily.

(the two are incompatible in the sense that they
generate different parities for the 2nd parity sector,
so the same code cannot be used in ZFS without losing
back compatibility, but both can still recover the
data perfectly fine if 2 sectors in a stripe are lost;
they just use different schemes for 2nd parity sector
and beyond)

The same basic XOR code is also used for 3-parity and
beyond, meaning that 3-parity does not have the same
performance loss that RaidZ3 has, *and* it generalizes
all the way up to 128-parity.

The scheme also works with the aforementioned XOR
optimization in the previous section.

That combination of flexibility combined with raw
speed on parity calculation and erasure recovery are
*perfect* for this scheme, as we probably want more
than just 2, or even 3, parities, for this scheme.
Remember, number of parities are number of disks you
can tolerate failing, and since a "T"EE shutting down
is a total loss of all data in its RAM, you want to
have larger parities with this scheme compared to actual
HDDs.

# Auditing The Storage "T"EEs

The “disks” in this scheme are really "T"EEs running
very simple programs that allocate pretty much the
entire the available memory in the "T"EE for use as
the “persistent” storage for the array.

Because of the sheer simplicity, this can be easy to
audit; we only need to check that it handles the
in-memory storage correctly as per the rules:

* The special “death record” sector that has no
  provisional-write capability, but requires strict
  monotonic increase in a version counter stored in
  it, and which the array uses to record which storage
  "T"EEs it considers dead vs alive.
* The current provisional-write sector being held,
  if any.
* The operations to read sectors, set up provisional
  writes (overwrite and delta modes), and commit and
  rollback provisional writes.
* The actual sector memory.

(In practice you probably also want the host supplicant
program to use `TCP_NODELAY` and `TCP_QUICKACK` as well,
you do *not* want 40ms delays inserted by the combination
of Nagle and deferred ack)

The simplicity makes it easier to audit the code.

This allows auditing even of the persistent storage,
and rollbacks can be protected against by ensuring
that the storage "T"EE can only rollback a provisional
write and cannot rollback further.

From there, the array code running in the main
application "T"EE builds up an array from multiple
instances of the storage "T"EEs, simulating a disk
inside the main application "T"EE.
The main application can then use standard file
system or database engine on top of this simulated
disk.
For example, key erasure needed in the statechain
schemes can be assured by using a non-copy-on-write
filesystem, such as Ext4 without journaling, or by
storing keys in a dedicated partition of the
simulated disk, and overwriting them directly and
waiting for a sync from the array code, which would
then be propagated to all surviving storage "T"EEs,
which you have audited to be actually operating
correctly and overwriting memory backing that
sector containing the to-be-deleted keys.

-------------------------

ZmnSCPxj | 2025-10-08 11:29:24 UTC | #2

# RAID5 Write Hole, Really Fixed

This is an amendment.

Unfortunately, the proposed fix for the RAID5 write hole does
not in fact work.

To see why it does not work, let us review the RAID5 write hole.

For simplicity, let us consider minimal disks that can only
store one bit.
We consider a setup with 2 storage disks, A and B, and 1 parity
disk, P, initially with the following data stored:

```
+-----+  +-----+  +-----+
|  A  |  |  B  |  |  P  |
+-----+  +-----+  +-----+
|  0  |  |  0  |  |  0  |
+-----+  +-----+  +-----+
```

Now, we want to change the bit stored in B from 0 to a 1.
That involves writing to both B and P, so that `P = A ^ B`
as per the erasure coding we use.

However, suppose we were able to only write to B, but we
crashed before we could write to P.

```
+-----+  +-----+  +-----+
|  A  |  |  B  |  |  P  |
+-----+  +-----+  +-----+
|  0  |  |  1  |  |  0  |
+-----+  +-----+  +-----+
```

And on recovery, it turns out the reason *why* we crashed
was because of a major electrical fault which totally fried
the disk A:

```
+-----+  +-----+  +-----+
|XXXXX|  |  B  |  |  P  |
|XXXXX|  +-----+  +-----+
|XXXXX|  |  1  |  |  0  |
+-----+  +-----+  +-----+
```

No biggie, we can recover, right?
Except that in order to recover, we calculate `A = B ^ P`
as per the erasure coding we use, and end up with:

```
+-----+  +-----+  +-----+
|  A  |  |  B  |  |  P  |
+-----+  +-----+  +-----+
|  1  |  |  1  |  |  0  |
+-----+  +-----+  +-----+
```

The problem is that before the crash we only intended to
write a 1 to B, but not to A.
Thus, the RAID5 write hole.

The solution in the post, unfortunately, does *not* fix
the write hole above.

Let us re-run the same scenario with our "T"EE storage
disks.
We start with this:

```
+-----+  +-----+  +-----+
|  A  |  |  B  |  |  P  |
+-----+  +-----+  +-----+
|  0  |  |  0  |  |  0  |
+-----+  +-----+  +-----+
```

We intend to change B to 1, and send a provisional write
to B and P to change their bits to 1.
Unfortunately, we are able to send the provisional write
to B, but not to P, before we crash.

```
+-----+  +-----+  +-----+
|  A  |  |  B  |  |  P  |
+-----+  +-----+  +-----+
|  0  |  |  0  |  |  0  |
+-----+  +-----+  +-----+
         |  1  |
         |(new)|
         +-----+
```

And it turns out the *reason* we crashed is because there was
an AWS region outage, and it happens that A is on the same
region as we were, totally trashing A as well:

```
+-----+  +-----+  +-----+
|XXXXX|  |  B  |  |  P  |
|XXXXX|  +-----+  +-----+
|XXXXX|  |  0  |  |  0  |
+-----+  +-----+  +-----+
         |  1  |
         |(new)|
         +-----+
```

Both the new and old states for B are plausible, in the absence
of A, and thus this does not in fact completely close the RAID5
write hole.
But if we select the new state for B, then we would get an
incorrect recovery for A.

We cannot, in general, determine if the state is before or
after all writes were posted to all modified stores, and that
is the core problem of the RAID5 write hole.
In particular, we cannot simply always roll back the change in
B — because what if instead the original situation had been:

```
+-----+  +-----+  +-----+
|  A  |  |  B  |  |  P  |
+-----+  +-----+  +-----+
|  1  |  |  0  |  |  1  |
+-----+  +-----+  +-----+
```

Then we post provisional writes successfully to both B and P:

```
+-----+  +-----+  +-----+
|  A  |  |  B  |  |  P  |
+-----+  +-----+  +-----+
|  1  |  |  0  |  |  1  |
+-----+  +-----+  +-----+
         |  1  |  |  0  |
         |(new)|  |(new)|
         +-----+  +-----+
```

Then we tell B and P to commit, but only the commit command
to P gets through before we crash:

```
+-----+  +-----+  +-----+
|  A  |  |  B  |  |  P  |
+-----+  +-----+  +-----+
|  1  |  |  0  |  |  0  |
+-----+  +-----+  +-----+
         |  1  |
         |(new)|
         +-----+
```

And A is trashed because it was an AWS region outage:

```
+-----+  +-----+  +-----+
|XXXXX|  |  B  |  |  P  |
|XXXXX|  +-----+  +-----+
|XXXXX|  |  0  |  |  0  |
+-----+  +-----+  +-----+
         |  1  |
         |(new)|
         +-----+
```

What we need to be able to do is to figure out what the
original state actually was.

## The Real Write Hole Solution

The correct solution is to ensure true atomicity.

For example, we can use what in filesystems is traditionally
called a journal, and in databases is traditionally called a
write-ahead-log.
Both are actually the same thing.

The array-management code reserves a journal area on all the
disks, and stores the journal RAID1 replicated to all disks.

The journal area has a version number as well, so that if
multiple disks have different journals, the array treats the
one with highest version number as the winner.

In the imagined scenario, the array code would write to the
journal “I will write 1 to B and 1 to P”, together with the
next higher journal version, and replicate that across all
the disks.
Once at least `num_of_parities + 1` disks have responded to
the writes to the journal, the array code can report success
to higher layers, but would still need to apply the
journalled writes to the disks — this would delay future
writes until the previously-journalled writes were known to
be committed to the actual storage areas, but at least would
allow parallelism between the higher layers and the array
code, if the higher layers have other things it needs to do
(such as sending `revoke_and_ack` after synching to disk).

On restarts, the array code would simply read the journal
area and always apply the journal entry with highest
version before signalling that the array has been mounted.

With this, the "T"EE storage disk program is now simpler,
as it no longer has to maintain provisional writes — the
array-management code gets the extra complexity of
maintaining a RAID1 journal in an area of the disks that
it reserves.

An optimization we can have is to add a “copy sector”
command, where the "T"EE storage disk program simply copies
the contents of one stored sector over the contents of a
different stored sector.
Then, when applying the latest journal entry, the array
does not need to read the journalled sector and then
re-send it to the destination disk; if the disk has the
latest journal entry the array can simply tell the disk to
copy the journalled sector into the final destination
sector, saving network bandwidth.

Once applied, the journal entry does not need to be
deleted; on restart the array will see the journal
entry again and re-apply it, but its application is safely
idempotent.
This reduces the network bandwidth in the expected common
case where we are not crashing the array-management code
all the time.

Alternately, if this is being used for a statechain signer, we could have a “trim” command that sets the sector to all 0s, and use that to delete the journalled sectors after the journal entry has been applied. This would assure us that data in the journal can be removed after the array knows the journal entry has been applied, and removes the possibility that old “deleted” keys may have been stored in the journal area of one of the disks and not overwritten.

As the journal gets overwritten, and involves multiple
sectors of data (a journal entry stores the change in the
storage sector plus all the changes in the parity sectors),
we need to ensure atomicity of the journal entry getting
written.
The journal area size would then have one sector for each
disk in the array (so that the journal has space to store a
full stripe write) plus an additional sector that stores
the version number of the journal entry and the destinations
for the journalled sectors.
We can make the additional, versioned, sector, the atomicity
sector.
It would not only store the version number, as well as the
destinations for the journalled data sectors, but also
include *checksums* (MACs) of the journalled data sectors.

Then, when the array knows it has applied the previous
journal entry, and wants to make a new one for a new write,
the array first writes the atomicity sector with the new
version, destination, and checksums, and then writes the
journalled data sectors.
These processes can be done in parallel for each of the
disks (with the requirement that the journal first ensures
a response from the write to the atomicity sector before
sending out write commands for the journalled data sectors).

On restart, the array looks for the highest-versioned
atomicity sector.
Then it checks for whether the corresponding journalled
data sectors match the checksums.
If the data sectors do not match, then it means the
latest journal entry was not successfully completely
written, and the array should simply not apply the
journal entry (the fact that it had a half-written
journal entry at version V means that it already
knew that the journal entry at version V - 1 was
already fully applied, and thus there is no need to
apply anything; the mismatching checksums imply that
the journal entry at V - 1 was deleted because it was
already known to be applied).
If the data sectors DO match the checksums, the journal
entry was fully written and the array thus always
applies it (this is benign as the latest journal entry
getting reapplied is idempotent).

-------------------------

ZmnSCPxj | 2025-10-08 16:13:19 UTC | #3

An objection against this scheme is: why would you even do this?

For instance, instead of having n storage devices and building against failure of n - k devices, why not just have a k-of-n signer, with n “T”EEs running signer code and using ephemeral keys?

The reason for preferring this scheme is: What are the reasons for you to bring down a running “T”EE?

Now, the more things a piece of code does, the more likely we need to stop it in order to update it.

And in the case of a simple storage “T”EE program, there is very little it does:

1. Validate that request comes from an authorized main program (e.g. if it is a Lightning node, it could sign all requests with the node key; the model being that exfiltration of the node key would be total loss of funds anyway)
   1. This public key can be hardcoded so that it is part of the PCR attested to when the main program first mounts the array.
2. Read sector
3. Write sector
4. (if we optimize) copy sector
5. (if we are paranoid about key deletion) trim/`memset(buf, 0, sizeof(buf))` sector

The sheer simplicity of this means there is little reason to ever update the storage “T”EE program, and thus, to ever have to stop it and restart it.

In addition, the “T”EE program itself does not have any private keys of its own.  If the main application itself needs to store keys, and is paranoid about sidechannels in the remote storage “T”EE letting the keys be exfiltrated because it has been running for several years on old hardware with old microcode that lets sidechannels exist, it can encrypt the keys (using the same key it uses to authorize itself as a reader/writer to the “T”EE disks) while still able to restart and migrate to new hardware with fixed microcode itself, since the persistent layer is separate from the business logic layer.

Thus the model is:

* The main program is stateless except for the private key it holds (wave hands about how it gets into the enclave; see ACINQ setup if you want an example).
  * It can be restarted at any time, and the software updated.
  * Operator is responsible for ensuring the updated software is trustworthy enough to hand over the private keys to (e.g. see how it is done for ACINQ setup, they basically have a number of responsible developers sign off on each version that they allow to run in the “T”EE, and the loader component that is fixed validates the developer signatures before running the main program and giving it the private key via a `tmpfs` mount)
* The storage programs are always kept running at very high uptime, with no real expectation to be restarted.
  * Even if new sidechannel attacks on their old hardware arise, the storage programs hold NO private keys of their own.  They only validate the commands sent by the purported main program via the hardcoded public key.
  * Without their own private key (not even an ephemeral one) the “T”EE storage program cannot establish an encrypted tunnel with the main program!  But:
    * The main program array-managing code can encrypt the data it puts in the sectors, and place the MAC elsewhere such as in a validation tree of sectors (like how ZFS does its checksums, because ZFS is ***awesome***).  Then, as far as the storage programs are concerned, the data it receives from the main program in the “write sector” command, and thus stores in its “persistent” memory, is “plaintext”, even though the array-managing code is actually encrypting the data before handing over to the “T”EE storage program.  That way, there is no need to establish an encrypted tunnel (which would require that the “T”EE storage program hold an ephemeral private key for purposes of establishing encrypted tunnels via ECDH), because the array-managing code has already encrypted the data that will be put remotely, and the “plaintext” commands contain ciphertexts already.  Integrity is still preserved by the requirement that commands be signed by the authorized main program, thus any change in the data in-transit will be detectable by failing the signature.
    * If you think about it, persistent storage is just a messaging system between your past self and your future self, and thus, you only need to encrypt data from your past self to your future self, with the persistent storage not requiring its own dedicated encrypted tunnel; your encrypted tunnel goes from your past self to your future self.
    * What about rollback attacks, which do not require that attackers be able to decrypt, only that they have an old copy of the ciphertext? Well, on each “read sector” command, the “T”EE storage program also creates a remote hardware attestation, committing to the sector read plus a challenge nonce from the main program, and thus assuring the main program that the “T”EE storage program with ***no*** rollback code is the one responding with the latest data for that sector, and not old data inserted by a MITM (the challenge nonce means old responses cannot be replayed, and the attestation key is something you already trust to not be exfiltrated, ever, because that would allow the “T”EE to be virtualized sufficiently that the program giving attestations can be different from the program you signed and checked the PCRs of).
  * You could even use standard LUKS, with the Lightning node key, or the statechain operator signing key, as the password to the LUKS volume, so that the array-managing code is just emulating a disk and presenting it to LUKS as the mountable disk, which simplifies your array manager so that the necessary MACs are handled by the LUKS layer.  OR just use ZFS with encryption on top, though that may be problematic for the statechain signer case as ZFS naturally does copy-on-write and may retain old “deleted” keys — at least LUKS itself presents just a disk as well, and you can use a non-journalled EXT4 to avoid the issue of multiple copies of stored disk, or manage the deletable keys yourself in a separate partition and put a proper filesystem like ZFS on the rest of the storage.

-------------------------

ZmnSCPxj | 2025-10-10 06:21:59 UTC | #4

[quote="ZmnSCPxj, post:3, topic:2029"]
You could even use standard LUKS

[/quote]

After a bit of research, I found that LUKSv2 composes two subsystems: dm-crypt and dm-integrity. The combination is *effectively* equivalent to an AE scheme, with dm-integrity checking for data modification or corruption before dm-crypt decrypts (effectively equivalent to Encrypt-then-MAC).  dm-integrity however needs to ensure atomicity of both the tag (what it calls the MAC) update and the actual sector, and to do so, it uses… a journal!

The problem is that of a [log on a log](https://www.usenix.org/system/files/conference/inflow14/inflow14-yang.pdf).  Briefly, a log — including just a short write-ahead log or journal — is required for atomicity, but represent writing twice as much on every write: once to the log, once to the real location.  But if you have a ***logged layer on top of a logged layer***, then you are writing ***four times***: first the upper layer writes to its log, which on the lower layer translates to two writes (one to the lower-layer log, one to the lower-layer storage), and then the upper layer writes to it storage, which on the lower layer translates to two ***more*** writes (one to the lower-layer log, one to the lower-layer storage).

Such a thing can happen when our array-management code uses a journal to write out planned writes to ensure atomicity across devices and plug the RAID5 write hole, then have an additional layer on top that handles AE (i.e. dm-integrity+dm-crypt aka LUKS2) that has its own journal to write out planned writes to ensure atomicity of updating MACs and the actual sector storage.

The correct solution is to collapse the log layers into a single one, which is why ZFS is ***awesome***, it uses the same atomicity logging for both plugging the RAID5 write hole ***and*** ensuring it is a transactional filesystem ***and*** it is capable of using cryptography-quality checksums ***and*** has encryption ***and*** that is all on one log layer.  More broadly, XFS has been proposing a bunch of extensions as well to integrate database logs into its own logs so that it can provide the atomicity of its own log to upper layer databases running on XFS, and avoid the log-on-a-log problem.

The eventual evolution of this is that you have a lower layer that has a ridiculously large log / journal, because all the higher layers are relying on it for atomicity, and that means more and more data being pushed inside an atomic operation.  The end result is you have a copy-on-write filesystem, just like ZFS, where in essence the ***whole*** disk is a log and there is no separate “storage” to rewrite to — you just write the log on to whatever free space is available instead of overwriting existing storage, and then after you are ***sure*** the lying underlying disk has ***actually*** written the data out, you mark the existing storage whose data you replaced as “now this is free”, without having to double-write from the journal to the storage: the journal ***is*** the storage.

The problem with that scheme is for the “key deletion” problem of statechain signers.  Old journal entries are effectively backups of the data, and therefore at risk of exfiltration of supposedly-deleted keys.  So we actually have to ***avoid*** copy-on-write schemes for statechain signers; we want a short journal that we specifically destroy each time we have applied the latest journal entry.  However, we ***can*** at least merge the atomicity-providing log layers for the AE and the RAID-X.

We can have an array of IV+MAC.  Each IV+MAC covers on sector of encrypted storage, and is itself stored in some number of sectors.  Like the encrypted storage itself, the IV+MAC is also done with erasure coding — note that we do not need to IV+MAC the parity sectors, only the actual storage sectors; if we can recover using the parity sectors and the recovered data matches the IV+MAC, then the parity sectors are also correct, thus they are implicitly covered by the same IV+MAC.  The IV removes the problem of using a stream cipher with full disk encryption, as full disk encryption requires sector-addressible deciphering; each encrypted storage sector gets its own IV, and we can use an AEAD scheme where the AD is the encrypted storage sector index in the RAID array.  For ChaCha20, the IV is 12 bytes.  We should use HMAC instead of Poly1305, due to a minor weakness in Poly1305 that allows the tag to be malleated by attackers to match the same encrypted data to a different MAC encryption key, in that the Poly1305 MAC matches, but decryption results in garbage output — but the point of Encrypt-then-MAC is that the probability of MAC matching for a different key is so low that if the MAC matches then decryption will be for the same encryption key and result in the original plaintext and not garbage, thus this issue with Poly1305 violates that assumption.  This is the key-commitment problem, and HMAC naturally commits to the MAC key (but polynomial-based MACs like Poly1305 do not).  The output of HMAC-SHA-2-256 is 32 bytes but I understand that it can be truncated to 16 bytes, so a 28 byte IV(12 bytes)+MAC(16 bytes) for each 4096 sector is slightly above half a percent overhead.  The array of IV+MAC would also be grouped into 4096-byte sectors, and each such sector also needs its own IV+MAC covering the section of the array it contains — we can have 145 IV+MAC entries in a 4096-byte sector, plus a 146th entry covering itself.

The journal only needs to be expanded enough to be able to contain one stripe width (storage sectors + parity sectors) for the main data, plus two IV+MAC storage sectors and the IV+MAC parity sectors to update.  We only need two IV+MAC sectors in case the storage sectors of a stripe have to cross between two IV+MAC sectors.  Then we can atomically ensure not only against RAID5 write hole, but also atomic update of the encryption+integrity data.  At the same time, the (relatively) small journal means we can afford to ask the disk to delete the journal data and leave only the version counter in the atomicity sector, thus also preventing having natural backups of supposedly-deleted keys.  While typical modern filesystems and databases will actually have logs on top for atomicity as well, a statechain signer application can simply perform direct writes to a dedicated partition of the simulated persistent storage disk, to ensure that erasure of old keys goes through fewer layers that need to be audited against having accidental backup copies.

-------------------------

