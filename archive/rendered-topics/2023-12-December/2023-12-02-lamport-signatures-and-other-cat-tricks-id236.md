# Lamport signatures and other CAT tricks

moonsettler | 2023-12-03 00:47:43 UTC | #1

Looking for some feedback on this contract. Is there a way to make it more efficient? What limitations on script might affect it? this would be fed about 4-5KB of "signature" data, including 140x 20 byte hashes merkle control bytes 20 preimages and sighash bytes. Would it be possible to run this on inquisition signet? Does it even make sense to attempt quantum resistant signatures in taproot given how the P2TR output is formed? Could we soft-fork taproot to enable script-only if needed?

```
// witness script
// *** 1 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 1 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 1 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 1 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 1 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 1 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 1 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 1 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 1 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 1 push state to alt-stack
OP_TOALTSTACK // <pubkey-build>

// *** 2 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 2 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 2 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 2 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 2 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 2 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 2 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 2 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 2 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 2 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** 3 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 3 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 3 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 3 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 3 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 3 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 3 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 3 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 3 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 3 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** 4 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 4 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 4 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 4 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 4 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 4 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 4 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 4 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 4 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 4 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** 5 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 5 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 5 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 5 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 5 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 5 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 5 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 5 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 5 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 5 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** 6 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 6 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 6 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 6 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 6 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 6 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 6 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 6 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 6 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 6 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** 7 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 7 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 7 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 7 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 7 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 7 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 7 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 7 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 7 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 7 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** 8 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 8 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 8 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 8 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 8 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 8 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 8 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 8 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 8 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 8 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** 9 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 9 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 9 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 9 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 9 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 9 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 9 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 9 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 9 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 9 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** 10 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 10 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 10 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 10 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 10 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 10 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 10 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 10 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 10 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 10 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** 11 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 11 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 11 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 11 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 11 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 11 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 11 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 11 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 11 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 11 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** 12 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 12 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 12 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 12 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 12 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 12 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 12 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 12 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 12 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 12 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** 13 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 13 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 13 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 13 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 13 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 13 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 13 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 13 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 13 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 13 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** 14 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 14 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 14 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 14 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 14 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 14 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 14 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 14 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 14 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 14 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** 15 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 15 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 15 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 15 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 15 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 15 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 15 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 15 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 15 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 15 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** 16 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 16 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 16 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 16 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 16 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 16 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 16 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 16 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 16 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 16 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** 17 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 17 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 17 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 17 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 17 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 17 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 17 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 17 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 17 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 17 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** 18 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 18 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 18 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 18 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 18 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 18 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 18 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 18 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 18 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 18 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** 19 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 19 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 19 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 19 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 19 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 19 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 19 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 19 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 19 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 19 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** 20 byte of <sighash>
OP_DUP
OP_TOALTSTACK  // <byte-value>
OP_CAT
OP_HASH160

// 1 of 20 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 2 of 20 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 3 of 20 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 4 of 20 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 5 of 20 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 6 of 20 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 7 of 20 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// 8 of 20 merkle verify step
OP_IF
OP_SWAP
OP_ENDIF
OP_CAT
OP_HASH160

// *** 20 update state on alt-stack
OP_FROMALTSTACK // <byte-value>

// stack: ... <merkle-root> <byte-value> | alt-stack: <sighash-build> <pubkey-build>

OP_FROMALTSTACK
OP_SWAP
OP_FROMALTSTACK
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_HASH160
OP_TOALTSTACK

// *** finished building the sighash and pubkey
OP_FROMALTSTACK // <pubkey>

<0x04396be4408229b5097d2092aa74bdf61e245828>
OP_EQUALVERIFY

OP_FROMALTSTACK // <sighash>
```

UX notes: The user would have a "private key" (256 bit) which can be generated with a BIP-32 HD wallet, the private key is used as seed for hardened BIP-32 derivation of the preimages for each position and byte value. Then a merkle tree is built for each "sighash" bytes and their roots are hash aggregated in a way that is compatible with this contract. This "root hash" acts as the "public key" as it commits to all of the merkle trees and is deterministically generated from the private key. Some introspection must follow this contract to check the transaction's conformity to the "sighash" which is the left on top of the stack at the end.

-------------------------

ajtowns | 2023-12-03 07:39:02 UTC | #2

This requires separate pushes of `0` or `1` for each level of each merkle tree to decide what branch you want. I don't think you can do better than that with just CAT, but if you had `DIVMOD` (takes `s1, s2` from the stack, pushes `int(s1/s2)` then `s1%s2` onto the stack; cf elements' `OP_DIV64`) you could provide a leaf number, and execute something like:

  `ROT ROT 2 DIVMOD 2SWAP ROT IF SWAP ENDIF CAT HASH160`

to combine a path towards the root (starting stack is: `[path, n, hash]`). It's 6B more script per level, vs 2 bytes less witness data per level, so unlikely to be interesting unless you're running into stack size limits, or also adding a looping construct

You could also consider an `OP_STR_LESSTHAN` that compares two byte vectors, and code it as `STR_LESSTHAN NOTIF SWAP ENDIF CAT HASH160`, and construct the merkle tree with an implicit order.

I think your script's idea is:
 * to create a single-use pubkey, you generate 20 byte commitments.
 * for each byte commitment, you create a merkle tree where the leaves are "nonce ++ byte"
 * the single-use pubkey is then the hash of the concatenation of its commitment roots

To verify a signature, you then:
 * for each byte of the message hash, you reveal the path to the nth commitment for the pubkey
 * you combine the commitment roots to produce the single use pubkey
 * you check the pubkey is what you expect
 * you're left with the msg hash on the stack, which you might then verify by constructing a signature verification with R=P=G, or could compare directly with the result of `<x> TXHASH HASH160` or similar.

There's a couple of stack optimisations you could do (you've got a TOALT immediately followed by a FROMALT, eg), but I don't think there's much else to be done, without some sort of looping construct.

-------------------------

moonsettler | 2023-12-03 10:33:10 UTC | #3

Thank you for the feedback @ajtowns!

What do you think about soft-fork restricting G from being used as internal pubkey for taproot (it's an obvious anyone can spend), to have a quantum resistant script-only P2TR output? Could also be a specific NUMS point used by convention in the meantime, with the promise that it gets protected as needed, if that makes more sense?

-------------------------

ajtowns | 2023-12-03 12:14:11 UTC | #4

That doesn't work -- if you can solve the discrete log, you just spend via the key path without ever revealing that the internal pubkey was G.

-------------------------

moonsettler | 2023-12-03 14:55:34 UTC | #5

Yes, that's what I would like to disable (opt-in).

My thinking was that for keyspend revealing the preimage `t` of the tweak is required. So then the clients could check if `Q = point_add(G, point_mul(G, t))` and then say NOPE! *But i guess an adversary could just calculate a different* `(a+b)G = (t+1)G` *, is what you meant.*

Is there a way to force script only and retain compatibility with the current address format?

-------------------------

