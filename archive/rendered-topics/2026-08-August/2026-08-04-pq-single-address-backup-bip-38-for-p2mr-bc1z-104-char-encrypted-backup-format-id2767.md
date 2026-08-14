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

