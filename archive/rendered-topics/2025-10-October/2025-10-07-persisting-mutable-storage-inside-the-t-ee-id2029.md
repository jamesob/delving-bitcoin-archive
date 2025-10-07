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

