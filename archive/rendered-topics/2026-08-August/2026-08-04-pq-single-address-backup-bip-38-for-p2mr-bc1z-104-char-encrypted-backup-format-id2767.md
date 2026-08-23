# PQ-single-address-backup - BIP-38 for P2MR (bc1z) - 104-char encrypted backup format

coldtest-berlin | 2026-08-06 11:41:35 UTC | #1

After discussion on P2MR / BIP-360 (bc1z), one open question remains: how to safely store a 32-byte SLH-DSA seed offline for cold storage / inheritance.

Storing 64 hex in plain text is not safe - one copy, fire/theft = loss. For long-term cold storage we need the BIP-38 UX: encrypted paper backup, many copies. 

Single-Address: One backup = One key = One `bc1z` address for hodl. Ready for sweep to P2MR operational wallet (after years, when you need your BTC)..

Proposal: PQ-single-address-backup(SAB).

Goal: companion to BIP-360, not a change to BIP-360. BIP-360 defines WHAT bc1z is. This defines HOW to store its seed.

Format v1:

• Payload: base58( salt:16 || iv:12 || ciphertext:32 || tag:16 ) = 76 bytes -> strictly 104 chars Base58 • Alphabet: Bitcoin Base58 123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz • KDF: PBKDF2-HMAC-SHA256 200k, ENC: AES-256-GCM • No version byte in seed itself per FIPS-205 / BIP-360 draft - seed is pure 32 bytes random. Algorithm is identified by descriptor p2mr_slhdsa_sha2_128s(seed)

File wrapper:

\-----BEGIN PQ SINGLE-ADDRESS BACKUP-----

<104 chars Base58>

ALG: SLH-DSA-SHA2-128s

KDF: PBKDF2-SHA256-200k

ENC: AES-GCM Base58

\-----END PQ SINGLE-ADDRESS BACKUP-----

ALG/KDF/ENC are outside encrypted bytes, human-readable, ignored by decoder. Ensures future wallets know which algorithm to use for bc1z.

Properties:

• Offline only: single HTML file, Wi-Fi OFF, ∼200 lines, no fetch/CDN, auditable • One file = one address = one bc1z • Password mandatory

Implementation: working offline index.html (encrypt/decrypt), spec.md attached. Loop ensures exactly 104 chars (re-rolls salt/iv).

Roadmap: v1 is audit build with PBKDF2 for easy WebCrypto audit. If the community accepts the concept, we need ONE canonical profile in a new BIP (e.g. Argon2id).

Question for the forum: Do we want a standardized 104-char encrypted single-address backup as a companion to BIP-360?

Looking for feedback and a reference implementation.

Reference code (offline HTML + spec) - see repository.

Repository: https://github.com/coldtest-berlin/pq-single-address-backup

Live https://coldtest-berlin.github.io/pq-single-address-backup/

-------------------------

coldtest-berlin | 2026-08-14 09:26:38 UTC | #2

UPDATE (Aug 14, 2026): bc1z reference removed. Format is now prefix-agnostic to stay compatible with BIP-360 evolution. Now described as BIP-360 (P2MR / P2QRH) compatible, no bc1z/bc1r hardcoded. Seed remains pure 32-byte SLH-DSA. See updated repo.

-------------------------

Anzus_GemWallet | 2026-08-14 12:20:14 UTC | #3

A self-describing format seems important here. If the binary payload has no version or algorithm identifier, but the human-readable ALG/KDF/ENC lines are ignored by decoders, how would a wallet reliably distinguish future profiles or reject a mismatched wrapper?

It may also be worth defining the recovery UX as part of the interoperability requirements: checksum or typo detection before running the KDF, an explicit wrong-password result, and test vectors covering corrupted backups. For long-lived paper backups, reliable failure behavior may matter as much as keeping the encoding compact.

Is the fixed 104-character requirement providing a concrete scanning or transcription advantage that would outweigh putting versioning inside the authenticated payload?

-------------------------

coldtest-berlin | 2026-08-14 17:12:11 UTC | #4

Thanks @Anzus_GemWallet — excellent points, exactly the feedback I was hoping for.

You are right about the wrapper. In v1 the ALG/KDF/ENC lines were human-only hints. That's fixed in today's UPDATE:

1\. Self-describing payload: Format is now described as BIP-360 (P2MR/P2QRH) compatible, no bc1z\`/bc1r\` hardcoded. Seed remains pure 32-byte SLH-DSA. For v2 I will move versioning inside the authenticated payload:

plaintext = \[1-byte version | 1-byte alg-id | 32-byte seed\] -> AEAD(scrypt(pwd))

Then decoder can reliably:

• reject mismatched wrapper (AEAD tag fail = wrong password) • reject future profile (version > supported) • ALG/KDF/ENC human lines stay as hint only, not trusted

2\. Recovery UX: Agree, for long-lived paper backup failure behavior matters as much as compactness. Plan:

• keep outer 104-char fixed length — concrete advantage for scanning / visual completeness check (one line = one key = sweep), like BIP-38 6P invariant • add checksum inside the 104-char encoding (pre-KDF typo detection) • flow: length check -> checksum check -> only then KDF -> AEAD -> explicit TYPO vs WRONG_PASSWORD vs UNSUPPORTED_VERSION • will add test vectors for corrupted / truncated / wrong-password backups

3\. On fixed 104 vs versioning inside: 104 stays for human factors, versioning moves inside authenticated payload, so they don't compete. The outer fixed size is the UX guarantee, the inner version is the interoperability guarantee.

Updated repo reflects the prefix-agnostic change. Will push v2 spec with the above.

If the community prefers to put version/KDF-params inside the outer wrapper instead of inside AEAD, I'm fine to bump the fixed size from 104 to e.g. 108-110 to keep a fixed-length invariant. The key property for me is fixed length + checksum + one-line = one-key sweep, not the exact number 104.

-------------------------

Anzus_GemWallet | 2026-08-15 06:44:34 UTC | #5

Thanks, this direction looks much safer.

One remaining consideration is that the decoder needs to know the KDF and its parameters before it can decrypt the authenticated payload. If those values exist only inside the AEAD plaintext, there is a bootstrapping problem; if they remain only in the human-readable wrapper, changing them could cause incorrect or unexpectedly expensive key derivation before authentication.

Perhaps a minimal outer header could contain the format version, KDF identifier, and bounded KDF parameters, with that exact header also supplied as AEAD associated data. The decoder could then select the derivation procedure while still detecting any header modification.

Also, an AEAD failure cannot by itself distinguish a wrong password from corrupted ciphertext. The pre-KDF checksum helps classify ordinary transcription errors, but it may be safest for the specification to describe the remaining result as “authentication failed” rather than guaranteeing “wrong password.”

-------------------------

coldtest-berlin | 2026-08-15 09:50:03 UTC | #6

@Anzus_GemWallet You are absolutely right on both points.

The bootstrapping problem is real. A minimal outer header (version | KDF id | bounded params) supplied as AEAD associated data is the correct way to do it — select KDF before decrypt, but still detect header tampering.

And yes, AEAD failure should be specified as authentication failed — the pre-KDF checksum is for catching transcription typos, it cannot prove wrong password vs corrupted ciphertext.

I think at this point the core problem is clear: we need an encrypted short backup for cold-storage, BIP-38 style, but for PQ SLH-DSA seeds, with a fixed-length, scannable, one-line = one-key sweep invariant.

I intentionally don't want to go deeper into the byte layout alone. This should become a shared BIP, not my personal format. My proposal (pure 32-byte seed, prefix-agnostic BIP-360 compat, 104-char visual invariant) can be the starting point, but KDF bounds, header as AAD, failure semantics — that should be decided together by wallets.

If there is interest, I am happy to co-author or hand it over.

-------------------------

coldtest-berlin | 2026-08-21 09:39:58 UTC | #7

Thanks for the feedback. Thinking about it further — we can also guarantee a fixed length 108 without padding tricks.

Since encrypt is done once, we can just do rejection sampling: if base58 length != target, generate new salt/iv and try again. On average ∼1.3 tries if we target the ceiling length. Cost is negligible compared to Argon2id.

That way for heirs the rule is trivial: grandpa's backup is always exactly 108 chars. If not — transcription error.

This keeps the format clean for a future shared BIP and gives instant pre-KDF check via length alone. Full checksum as AAD can be added later as part of the shared header.

-------------------------

Anzus_GemWallet | 2026-08-23 08:21:23 UTC | #8

A fixed length sounds helpful, especially for paper backups that may be stored for many years.

I would still include the typo check in the first version, though. A copied or mistyped character can leave the backup the same length, so length alone may not give people enough confidence before they try to recover it.

-------------------------

coldtest-berlin | 2026-08-23 08:59:41 UTC | #9

Thanks for the feedback — you're right, length alone isn't enough for a paper backup that lives for years. A single mistyped char keeps the length the same.

We will follow your advice and add a typo check in v2:

Fixed length will be 108 chars Base58 (79 bytes instead of 76), and we will add 2 bytes CRC16 in the header checked before any KDF.

Structure: \[1 ver | 2 CRC16(salt+iv+ct) | 16 salt | 12 iv | 48 ct\] = 79 bytes → 108 chars

Keeps audit simple, but gives confidence before recovery.

-------------------------

