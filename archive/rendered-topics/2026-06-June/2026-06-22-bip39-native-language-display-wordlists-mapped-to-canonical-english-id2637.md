# BIP39 native-language display wordlists mapped to canonical English

DaniB | 2026-06-22 12:20:35 UTC | #1

Hi everyone,

I would like to share an early-stage proposal for native-language display and input wordlists for BIP39 recovery phrases, while keeping the existing English BIP39 mnemonic as the canonical form.

A discussion has already started on the Bitcoin Development mailing list here:

[https://groups.google.com/g/bitcoindev/c/Rwo7P5pTA0c](https://groups.google.com/g/bitcoindev/c/Rwo7P5pTA0c?utm_source=chatgpt.com)

Draft implementation and wordlists are available here:

[https://github.com/osem23/bip39-wordlists-tzur](https://github.com/osem23/bip39-wordlists-tzur?utm_source=chatgpt.com)

The proposal does not change BIP39 seed generation, checksum validation, PBKDF2, BIP32, BIP84, or any cryptographic behavior.

The localized words are not passed directly into PBKDF2. They are only used as a deterministic display/input layer over the canonical English BIP39 wordlist.

The intended flow is:

Native display words
-> fixed word indexes
-> canonical English BIP39 words
-> standard BIP39 validation
-> standard PBKDF2
-> standard wallet derivation

The motivation is practical recovery UX. Many users are still expected to back up and restore Bitcoin wallets using English recovery words, even when English is not their native language. This creates unnecessary friction and can lead to spelling mistakes, misunderstanding, and lower confidence during backup or recovery.

The goal is not to replace BIP39 or introduce a new seed format. The goal is to define an optional convention that wallet developers can implement while preserving compatibility with the canonical English BIP39 flow.

A wallet implementing this approach should always treat English BIP39 as the canonical recovery form. Localized wordlists should be treated as separate display/input lists, not as independent BIP39 mnemonic languages. The wallet should clearly label the recovery mode, avoid silent reinterpretation of existing non-English BIP39 mnemonics, and always allow the user to view and export the canonical English phrase.

One of the main concerns raised so far is ambiguity. For example, if a user enters a French 12-word phrase, the wallet must not silently guess whether it is a legacy French BIP39 mnemonic or a mapped display mnemonic. That distinction needs to be explicit in the UI and in any import/export behavior.

To reduce ambiguity, the current direction is to include clear metadata around each display wordlist, such as language code, version, and hash of the exact wordlist file. The canonical English phrase remains the compatibility layer for existing wallets.

I am posting this here because Delving Bitcoin is a better place for deeper technical review before deciding whether this should remain a wallet-level convention or be developed further as an informational BIP.

The areas where I would especially appreciate review are:

Whether this index-mapped display-layer model introduces any hidden recovery or compatibility risks.

Whether explicit mode selection is sufficient to avoid confusion with existing non-English BIP39 wordlists.

Whether this belongs in the BIP process at all, or should remain documentation and implementation guidance for wallets.

The intent is to improve native-language recovery UX without changing the underlying standards that existing Bitcoin wallets rely on.

Feedback and criticism are welcome.

Best,
Daniel

-------------------------

