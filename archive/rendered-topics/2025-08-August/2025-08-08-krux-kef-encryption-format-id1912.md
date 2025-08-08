# Krux: KEF Encryption Format

jdlcdl | 2025-08-08 16:55:50 UTC | #1

Good evening,

At project krux, work to extend an existing encryption scheme is near completion.  Before getting further along (release is near, this is already in beta), I'm sharing KEF_Specifications.

More eyes and any feedback would be much appreciated.

To avoid a long wall of text in this forum, I'll include only the "Motivation" section and a link below.

Enjoy your weekend.

---

KEF Encryption Format – Specification

…`The K stands for "KEF"` --anon

## 1. Motivation

In the autumn of 2023, during the lead-up to [krux release 23.09.0](https://github.com/selfcustody/krux/releases/tag/v23.09.0), contributors proposed a method of encrypting bip39 mnemonics that could be stored in SPI-flash, on sdcard, and/or exported to QR as specified in [krux documentation](https://selfcustody.github.io/krux/getting-started/features/encrypted-mnemonics/#encrypted-qr-codes-data-and-parsing).  Regarding the encrypted-mnemonic QR format: the layout proposed was interesting as an extensible, lite-weight, self-describing envelope that has been appreciated by users ever since.

…`"Wen passphrases, output descriptors, PSBTs, and notes?"` --plebs

This specification, and its accompanying implementation and test-suite are the result of months of exploration into improvements meant to better define, test, and extend the original encryption format that we’ll refer to as KEF.  It proposes ten new versions, extending its usefulness to more than mnemonics, targeting variable-length strings up to moderately sized PSBTs, flexibility to choose among four AES modes of operation, with-or-without compression, and versions optimized to result in a smaller envelope.

Above all, this specification aims to be supported by as many projects as would consider adopting it, so that users are not “locked” into a particular project when recovering their secrets.  Corrections and refinement to, and scrutiny of this specification are appreciated. Proposals for more `versions` are welcome, provided they offer “value” to the user and fit within the scope of this system.  Once released, because it cannot be known how many KEF envelopes may exist in-the-wild, changes to any particular version must remain backwards compatible for decryption.  Adopting implementations are free to support any KEF versions they wish to support, for decryption-only or for both encryption and decryption – with the expectation that claims-of-support made are clear and precise about what is supported.

[External link to KEF_Specifications on github](https://github.com/selfcustody/krux/blob/733483665b09506099921c0e61d75d965295c759/docs/getting-started/features/KEF_Specifications.md)

-------------------------

