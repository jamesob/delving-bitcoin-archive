# Bitcoin Core v31.1rc1 is now available

fanquake | 2026-06-29 14:22:43 UTC | #1

Binaries for Bitcoin Core version v31.1rc1 are available from:

    https://bitcoincore.org/bin/bitcoin-core-31.1/test.rc1/

Source code can be found in git under the signed tag

    https://github.com/bitcoin/bitcoin/tree/v31.1rc1

This is a release candidate for a new minor version release.

Preliminary release notes for the release can be found here:

    https://github.com/bitcoin/bitcoin/blob/efde623463cf194dda8407271f4dd136d054bc9f/doc/release-notes.md

Release candidates are test versions for releases. When no critical problems are found, this release candidate will be tagged as v31.1.

Please test! If you run into issues, please report them to https://github.com/bitcoin/bitcoin/issues.

-------------------------

mercie-ux | 2026-07-02 19:30:34 UTC | #2

I have tested v31.1rc1 on my pc ubuntu 22.04(jammy), kernel 6.8.0.124. Environment : Intel Core i5-7300U @ 2.60GHz (4 cores), 8GB RAM.

Ran all the functional test suites(284/284 passed), manually verified (#35316) musig2 empty pubkey rejection, return a clean rpc error as expected. Therefore no issue found.

-------------------------

