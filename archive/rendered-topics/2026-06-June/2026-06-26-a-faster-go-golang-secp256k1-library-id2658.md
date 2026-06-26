# A faster Go (golang) secp256k1 library

allocz | 2026-06-26 18:28:03 UTC | #1

With the support of [vinteum](https://vinteum.org) I created a Go [secp256k1](https://github.com/allocz/secp256k1) library that works both with and without CGO (which is the way Go does foreign function interfaces). When CGO is enabled, the implementation used is [libsecp256k1](https://github.com/bitcoin-core/secp256k1), when CGO is not enabled (or the system does not have a C compiler) the secp256k1 implementation used is [dcrec](https://github.com/decred/dcrd/tree/master/dcrec), a pure Go secp256k1 library that is used by [btcd](https://github.com/btcsuite/btcd).

The [benchmarks](https://github.com/allocz/secpbench) show that when using CGO, ECDSA and Schnorr signature verification time per call drop at least 70%, which is a speedup of \~350%.

It is common for several Go projects to not allow code that interfaces with C, because it hurts portability, since cross compilation setting `GOOS`and `GOARCH`stop working. This implementation avoids this issue by using the slower and portable implementation when CGO is not used, so cross compilation will always work when setting the environment variable `CGO_ENABLED=0`.

Would be nice to get feedback from other devs. Thanks for reading!

-------------------------

