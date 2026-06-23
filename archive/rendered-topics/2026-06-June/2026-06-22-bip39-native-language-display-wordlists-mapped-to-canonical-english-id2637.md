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

optout | 2026-06-23 08:25:22 UTC | #2

Sounds like a good idea, and it solves the non-English seedphrase issue while building on top of existing standards. It does introduce/keep some confusion, with non-English BIP39, but that confusion exists today as well. Fortunately non-English BIP39 support is marginal.

Some thoughts:

\- Any description of this idea (no matter how succint) should mention that BIP39 has non-English language support, but it's not widely implemented, and how it's broken (no possibility for easy translation, incomplete).

\- Having some of the same words as in the BIP39 lists is unfortunate, because a seedphrase can be composed of words that are all present in both lists, making it impossible to decide which scheme is used. (It would be an interesting exercise, to see how many combinations exists that produce a valid checksum in both schemes). I understand that the "direct translation of the English words" was a stronger design principle than the avoidance collisions with the BIP39 non-English lists.

\- The main advantage of this scheme is that a non-English seedphrase can be relatively easily turned into the matching English seedphrase, by having access to and using the wordlists (non-English & English), and matching the words manually. This way the scheme does not rely on any implementation existing for the given language, after translation a standard English BIP-39 implementation will yield the same wallet. I see this as the biggest danger of BIP39, as someone with a non-English BIP39 seedphrase backup may be in trouble if supporting implementations disappear.

\- Checksum. The checksum validity of a non-English seephrase can be determined before translation, as I understand. The entropy is derived from the word indices, and the checksum algorithm operates on the entropy. And obviously, the validity is preserved by translation (the indices are preserved).

\- BIP. If the goal/expectation is that more than one wallet supports this scheme in an interoperable way, than this should be submitted as a BIP. It should also mandate that BIP39 non-English support should not be used, or at most as import-only (but never display/export in that format). After submission as a BIP draft the wordlists should not be changed (maybe new ones added).

\- It would be interesting to gather prefix statistics on each wordlist: the most common 2-, 3-, and 4-letter prefixes and their frequency, the longest common prefix.

\- I think the topic of diacritics/accented characters should be also covered, and (1) validate that no two words exist that are differentiated only by diacritics, and (2) propose that implementations also accept similar characters (e.g. 'acido' instead of 'ácido' (Spanish) or "eglenmis" instead "eğlenmiş" (Turkish)). Often input devices may lack the required characters.

-------------------------

DaniB | 2026-06-23 09:18:58 UTC | #3

Hi optout,

Thank you, this was very useful feedback and it prompted a concrete audit/update pass on the reference repository.

A few points are now documented or enforced more explicitly:

1. Existing non-English BIP39 wordlists

The draft now makes the relationship to existing non-English BIP39 wordlists clearer.

BIP39 already includes non-English wordlists, but those lists are independent canonical BIP39 lists, not translations of the English list. Existing backups created with those lists remain valid and should be supported by wallets that choose to support them.

This proposal does not reuse those legacy non-English BIP39 lists as display lists. It defines separate display/input lists that map deterministically by index back to canonical English BIP39.

2. Manual recovery path

I added clearer documentation for the manual recovery path:

localized display word -> word index -> English BIP39 word at the same index -> canonical English BIP39 mnemonic -> standard BIP39 recovery

This is one of the main advantages of the approach. A user is not locked into a specific wallet implementation. With the localized display list and the English BIP39 list, the backup can be converted back to the canonical English phrase manually, and then restored in a standard English BIP39-compatible wallet, assuming the same derivation path and any BIP39 passphrase settings.

3. Checksum preservation

The draft now states explicitly that checksum validity is preserved because the index sequence is preserved.

The display phrase resolves to the same BIP39 indexes as the English mnemonic. Therefore the entropy and checksum bits are unchanged. The display layer does not recompute, relax, or replace the BIP39 checksum.

4. Prefix statistics

A new deterministic prefix statistics report was added for the display wordlists.

It reports, per language:

* most common 2-character prefixes

* most common 3-character prefixes

* most common 4-character prefixes

* whether 4-character uniqueness holds

* largest prefix collision group

The result confirms that 4-character uniqueness should not be assumed generally. It holds only for Korean in the current set. Wallets should therefore use full-word matching as the safe fallback.

5. Normalization and diacritics

I also added a normalization collision check to the validator.

The important result is that there are zero NFKD collisions across all display wordlists. Since BIP39 uses NFKD at the derivation boundary, this is now enforced as a hard validation error so it cannot regress.

There are, however, collisions under lossy wallet-side input handling such as accent stripping or case folding. For example, some languages can produce ambiguity if a wallet accepts user input without diacritics.

The documentation now says that if normalization or forgiving input creates ambiguity, the wallet must reject the token and ask the user to disambiguate. It must never silently choose one word.

6. Legacy non-English BIP39 export

I added an interoperability recommendation that wallets implementing this display/input convention should prefer canonical English BIP39, or this reversible display path, for new backups rather than creating new legacy non-English BIP39 backups.

This does not deprecate existing non-English BIP39 backups and does not affect wallets that already created them. It is only a recommendation for new wallet export/display behavior, to improve portability.

7. BIP direction

I agree with your point that if the goal is interoperable support across more than one wallet, this should be formalized as a BIP.

The updated draft keeps English BIP39 as the seed of record and the only PBKDF2 input. The display wordlists are fixed, deterministic mappings to the English BIP39 indexes.

The latest update added:

* docs/prefix-statistics.md

* validation/prefix_stats.py

* normalization collision validation in CI

* clearer manual recovery wording

* clearer checksum wording

* clearer interoperability guidance around legacy non-English BIP39 exports

No wordlist or mapping bytes were changed.

Thanks again. This was very helpful feedback and made the draft more precise.

Best,
Daniel

-------------------------

