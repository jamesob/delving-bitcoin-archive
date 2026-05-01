# QR-based signing flow payloads in Miniscript context

pyth | 2026-04-28 11:35:37 UTC | #1

This post follows discussions that have been happening on GitHub
([here](https://github.com/SeedSigner/seedsigner/issues/306) and
[here](https://github.com/wizardsardine/liana/issues/539)) and
conversations I’ve had with several signing device and wallet developers.
I’m trying to formalize all of this into something concrete we can iterate on.

## Motivation

QR-based signing flows already exist for standard multisig protocols, those schemes
already handle passing descriptors, xpubs, and PSBTs between software wallets and
air-gapped devices. This actually “works well” because descriptor follow a common
“template” (m-of-n), derivation paths follow BIP-48, and devices know what to expect.

Miniscript changes the picture. The spending policy is no longer a fixed template
, it can be user-defined, potentially complex (timelocked recovery, decaying
thresholds, etc… ). Existing QR protocols don’t cover this well.

This post proposes defining the **content** of a set of payloads to cover the full
Miniscript flow over QR. Payloads below are described as JSON purely for readability,
this is **not** a commitment to JSON as the “wire” format. A bytes/binary encoding
could be preferable in practice to save space and reduce the number of QR frames.
I’m leaving encoding/framing/chunking out of scope for now — the goal is to agree on
*what* data needs to travel between software wallet/coordinator and signing device
before discussing *how* to serialize it.

### What Miniscript adds on top of existing multisig QR flows

1. **Flexible xpub retrieval**
   Miniscript policies may need xpubs at arbitrary or multiple derivation paths, so the
   software wallet needs to explicitly request them. This could also be useful for
   wallet software willing to improve ux (fetching a set of xpub once in order to let
   the user choose the account without one more QR exchange).

2. **User-customizable descriptors and registration**
   In Miniscript, the policy is variable, the device *must* receive, display,
   and let the user verify a descriptor it has never seen before. On top of that,
   miniscript descriptors can be larger than simple multisig configs, so stateless or
   storage-limited devices could need a way to offload a cryptographic proof of
   registration to the software wallet and re-validate it later without persisting the
   descriptor.

3. **Descriptor–PSBT binding**
   At signing time, the device needs to know which descriptor(s) the transaction relates
   to. This way it can verify and label inputs and outputs that belong to known
   descriptors (e.g. showing “Bob wallet” instead of a bare address), which could be a
   significant UX and security improvement on the signing device screen.

---

## Proposed payloads

### 1. Get Xpub(s)

**Use cases:** Policy creation, encrypted backup decryption.

**Request** (software → device)

It’s important here that the software can request a *list* of xpubs, for 2 reasons:

* if the software want to allow the user to choose which account is used, it’s better
  UX to request xpub only once.
* Some backup encryption scheme like [this one](https://github.com/bitcoin/bips/pull/1951)
  can need to fetch several xpub at once.

```
derivation_paths: [
  "48'/0'/0'/2'",
  ...
  "48'/0'/9'/2'",
]
```

**Response** (device → software)

```
xpubs: [
  "xpub6Abc...",
  ...
  "xpub6Xyz...",
],
fingerprint: <string>, // mandatory
model: <string>,    // optional — device model identifier
version: <string>,  // optional — firmware version
```

Note: `model` & `version` can be important to know for the software wallet, for
instance knowing the device dot NOT support tapminiscript (Hi Jade!) in order to hint
the user if he wanted to craft a taproot descriptor.

Note2: order of returned xpubs must follow order of derivation paths

---

### 2. Register descriptor

**Use cases:** Miniscript policy creation or restoring a wallet on a replacement
device after failure.

**Request** (software → device)

```
wallet: "Liana-abcdefgh",          // descriptor alias (mandatory)
bip380: <BIP-380 descriptor>,      // optional
bip388: <BIP-388 wallet policy>,   // optional
bip392: <BIP-392 silent-payment descriptor>, // optional
```

Semantics:

* If **only** `wallet` is sent → the software is asking: *“Is this descriptor
  already registered on the device?”*
* If `wallet` **+** one `bip3xx` field is sent → the software is asking: *“Let the
  user verify this descriptor”*, and the device should either persist it or return a
  Proof of Registration.

> At most one of `bip380` / `bip388` / `bip392` should be present.

**Response** (optinal) (device → software)

```
wallet: "Liana-abcdefgh",
registered: <true|false>,
por: <proof_of_registration>,  // optional
error: <string>,               // optional
```

> `registered` may be omitted if `por` is returned (the proof itself implies
> registration).

**Open questions:**

* Should we support registering multiple descriptors in a single QR exchange?

---

### 3. Address verification

**Use case:** Trigger an address verification flow on the signing device — the
device independently derives the address and displays it for the user to confirm.

**Request** (software → device)

```
wallet: <descriptor_alias>,
address: <address>,             // optional but recommended if test network
deriv: <derivation_path>,       // which index to derive
descriptor: <full descriptor>,  // optional, for stateless devices
por: <proof_of_registration>,   // optional, for stateless devices
```

**Response** (device → software)

Typically nothing is needed (verification is visual on the device). Optionally:

```
uri: <BIP-21 payment URI>
```

---

### 4. Signing

**Use case:** Sign a PSBT.

> Note: It would be useful to add PSBT proprietary fields (or any other means)
> indicating which descriptor a given input/output belongs to, so the device can
> quickly match without trial-deriving. Referencing several descriptors could be useful
> in cases either we spend tx from several descriptors we own, or if the device can use
> a “contact list” so it could clearly labels each input/output.

**Request** (software → device)

```
wallets: [
  { alias: <descr_alias>, descr: <descriptor>, por: <proof of registration>},
  ...
  { alias: <descr_alias>, descr: <descriptor>, por: <proof of registration>},
],
psbt: <base64-encoded PSBT>
```

Note: desriptor and proof of registration can be optional when the signing device
stores registered descriptors.

**Response** (device → software)

**Option A, Return the full PSBT** (simplest)

```
psbt: <base64-encoded PSBT with partial sigs added>
```

**Option B, Return only signatures** (minimal, better for QR bandwidth, and better
than trim the psbt)

```
signatures: [
  { input: <index>, type: "segwit"|"tapkey"|"taptree", signature: <hex sig> },
  ...
]
```

> For `taptree` spends, the control block should also be included in the response,
> since the device knows which leaf was used.

**Open questions:**

* For taptree, should the response include the full script path (control block +
  leaf script), or just the control block?

---

## What this post is NOT about

* **Encoding/framing**
* **QR chunking**: How to split large payloads (PSBTs) across animated QR frames.
* **Transport layer**: Fountain codes, frame numbering, error correction.

---

Looking forward to feedback — especially from signing device and wallet
developers already working with Miniscript or planing to do so.

-------------------------

odudex | 2026-04-28 13:07:40 UTC | #2

[quote="pyth, post:1, topic:2464"]
Note: `model` & `version` can be important to know for the software wallet, for instance knowing the device dot NOT support tapminiscript (Hi Jade!) in order to hint the user if he wanted to craft a taproot descriptor.

[/quote]

To be vendor agnostic, how about compatibility flags instead of model and version?

-------------------------

pyth | 2026-04-28 13:13:37 UTC | #3

hum, I’d say it’ll be a pain to maintain

-------------------------

scarlin90 | 2026-05-01 15:46:58 UTC | #4

Hey @pyth, the structural flow here looks solid.

I am currently developing Signingroom.io, which acts purely as a coordinator rather than a full software wallet. My focus is entirely on allowing signers to easily gather, communicate, and merge their signatures. Right now, I’m working on implementing fountain QRs to make that air-gapped multisig experience as frictionless as possible.

While I haven’t dived deep into the Miniscript parsing side myself, I am very focused on the transport layer. From a coordinator’s perspective, where bandwidth is important, here is my feedback on the payloads:

1. I strongly prefer Option B (Signatures only). As a coordinator, I already hold the state of the full PSBT. Returning the full PSBT (Option A) forces the hardware device to transmit kilobytes of redundant data, requiring a long, dense animated QR. Returning just the signatures drops the payload to a few hundred bytes, which can often be displayed as a single, static QR frame. One note on Option B: Since the coordinator might be managing multiple sessions, it would be highly useful to include a lightweight identifier in this payload (like the unsigned txid, a hash of the PSBT, or a simple session_id). This allows the coordinator to instantly verify the signatures belong to the correct active PSBT before attempting to merge them.


2. I completely agree with @odudex regarding feature flags. As a coordinator, relying on model and version strings in the Get Xpubs payload forces maintenance of a centralized, constantly updating lookup table of every hardware device on the market. Swapping that out for a device-agnostic array (e.g., supported_features: \[“taproot”, “miniscript”\]) is much more scalable and future-proof.

3. Relying on an optional error field feels a bit brittle too. Moving toward a standardized set of integer error codes would allow coordinators to programmatically route users to specific UI fixes rather than just dumping raw text on the screen. (Something similar to JSON-RPC error codes could be a great model here).

Overall, this is a step in the right direction in my opinion. Once the proposal matures, I’d love to support this workflow and would be very open to collaborating on the transport/QR side of things once I get the traditional PSBT fountain QR codes implemented in signing room.

All the best,
Sean Carlin

-------------------------

