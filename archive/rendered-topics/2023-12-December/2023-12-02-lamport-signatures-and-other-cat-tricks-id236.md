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

sipa | 2023-12-03 15:09:21 UTC | #6

Key path spends don't reveal the tweak at all. They're just BIP340 signatures for the key in the output (which is typically the tweaked key), nothing more.

If you want to retain security if DL is no longer assumed to be hard, you need to disable key path spends, no way around it. As long as SHA256 remains preimage resistant, script path spends remain secure (obviously only under the assumption the script itself isn't vulnerable to a DL break).

-------------------------

moonsettler | 2023-12-03 15:24:03 UTC | #7

Yes, thank you!

We were discussing this with @reardencode and figured you could force revealing additional information in the annex maybe, like the script hash and enforcing the tweak and it's commitment to the pubkey in validation. In general there might be no point in allowing keyspends then. My thinking was, at first it might be prohibitively expensive to go after everyone. Large treasuries and old P2PK coinbase outputs would be the first targets. Of those, treasuries are the one that might prepare a quantum proof 'exit hatch' in advance especially if it costs them next to nothing.

-------------------------

moonsettler | 2023-12-03 15:52:18 UTC | #8

So basically this script is not implementable on bitcoin in any meaningful way, even if CAT was soft-forked in...  :smiling_face_with_tear:

-------------------------

ZmnSCPxj | 2023-12-05 12:33:44 UTC | #9

If we are willing to softfork something in that allows Lamport and other ttricks and re-enabling of `OP_CAT` (and converting all `OP_NOP` to `OP_SUCCESS`), then maybe we can use a different P2SH-like encoding?  i.e. instead of `OP_HASH160 <hash> OP_EQUAL` as in P2SH, create a new P2XSH with template `OP_HASH256 <hash> OP_EQUAL`.  If this template is recognized, then the softfork version expects the script top to be equivalent to a `0xC0`-version script as in tapleaf, including `OP_SUCCESS`.  Then you can execute any extensions supported inside `0xC0`-version tapleaves in the P2XSH script, including `OP_CAT` if that is enabled inside tapscript version `0xC0`.  This lets you avoid any quantum-non-resistant paths on the way to the script.

(Or maybe just make it into a tapleaf so include the version to the front of the script. sort of thing)

-------------------------

moonsettler | 2023-12-05 15:06:11 UTC | #10

It would be nice to have something like P2SH/P2WSH where the script is like tapscript in upgradeability and relaxed limitations.

-------------------------

moonsettler | 2024-02-08 22:19:48 UTC | #11

CAT can be used to solve the data availability problem for LN-symmetry

```text
# S = 500000000
# IK -> A+B
<sig> <state-n-recovery-data> <state-n-hash> | CTV SHA256 CAT IK CSFSV <S+1> CLTV
```
before funding sign first state template:
```text
# state-n-hash { nLockTime(S+n), out(contract, amount(A)+amount(B)) }
# settlement-n-hash { nSequence(2w), out(A, amount(A)), out(B, amount(B)) }
# state-n-recovery-data { settlement-n-hash or state-n-balance }

# contract for state n < m
IF
  <sig> <state-m-recovery-data> <state-m-hash> | CTV SHA256 CAT IK CSFSV <S+n+1> CLTV
ELSE
  <settlement-n-hash> CTV
ENDIF
```
`CLTV` ensures only a larger `nLockTime` transaction can spend the current on-chain state, the relative timelock for the last co-signed state's `CTV` distribution is committed to in the `settlement-hash`

to recover the script for state n in case it's not the final state, `settlement-n-hash` can be abbreviated to 7 bytes `state-n-balance` in cases where the state can be represented as a simple balance (no HTLCs)

signing a concatenated string ensures critical data availability, and therefore only the latest state needs to be kept.

**edit:**
as @JeremyRubin pointed out, the naive implementation is malleable and insecure, because the CTV operation is only defined for 32 byte parameters. by adding an extra SHA256 over the CTV template hash, and then concatenating it for signature check we make the template and the composite message immutable. (alternatively we could use several opcodes for a precise size check)

`message = state-m-recovery-data|hash(state-m-hash)`

**sidenote:**

*a 'neutered' version of `OP_CAT` could be limited to producing only 80 byte long strings maximum, which allows for a number of harmless use cases like lamport signatures, 0-conf bonds and other nonce point manipulation schemes, and also for a more vbyte efficient channel closure, without enabling detailed introspection and finite state machines and other possibly undesired or highly contentious MEVil related behaviors without explicit future authorization.*

*said restriction can even be temporary (up to a specific block height), and reinforced with a future soft fork if still desired, or left to expire if the concerns become moot.*

-------------------------

harding | 2024-02-09 03:13:23 UTC | #12

[quote="moonsettler, post:11, topic:236"]
CAT can be used to solve the data availability problem for LN-symmetry
[/quote]

What is the data availability problem for LN-symmetry?  (Sorry if it's something obvious and I'm just blanking on it.)

-------------------------

moonsettler | 2024-02-09 11:24:28 UTC | #13

no, perfectly fair question! my bad for not explaining.

so the issue is, when the channel coins are moved by your peer to a new address, and that is not the final state that you recognize, you need to have all the non-deterministic pieces of the script to reconstruct it and be able to spend with the latest state you kept.

a lot can be reconstructed from the nLockTime to which CTV commits to so it's not alterable. that gives you precisely which state are you looking at. but the final settlement hash depends on the balance between the peers and the HTLCs if there are any.

so you can boil it down, to having to know the time locked settlement CTV template hash.

one goal of LN-symmetry is to have leaner channel state and better backup and recovery assumptions. therefore the critical data for recovery needs to be retrievable from the transaction that spends to an intermediate state.

OP_RETURN output can be used for this, but that is 4x more expensive than SegWit space.

**other possible options are:**

- CTV committing to the taproot annex, and annex being enabled in relay policy (APO approach)
- VECTORCOMMIT opcode, hash(hash(m1)|hash(m2)|...|hash(m16)), where hash is taggedSha256 purposfully made to be incompatible with SIGHASHING to avoid 'sneaky CAT'
- nCSFS/V where you can optionally commit to multiple messages (integrated VECTORCOMMIT into CSFS/V)
...

-------------------------

harding | 2024-02-11 19:03:26 UTC | #14

Sorry, I'm still confused.

[quote="moonsettler, post:13, topic:236"]
when the channel coins are moved by your peer to a new address, and that is not the final state that you recognize, you need to have all the non-deterministic pieces of the script to reconstruct it and be able to spend with the latest state you kept.
[/quote]

Is this about your peer having a later state than you have, so you need to be able to put the settlement transaction onchain in order to be able to spend your outputs?  Or is it about you having a later state than your peer has published, so you need to be able to spend their onchain state to your state (i.e., rebinding your state to the state they published)?

For settlement tranactions where you've lost state, don't we just assume that the remote peer will put the transaction onchain and that you'll be able to identify your outputs?  For rebinding, isn't that what APO provides?

-------------------------

moonsettler | 2024-02-11 22:04:22 UTC | #15

[quote="harding, post:14, topic:236"]
is it about you having a later state than your peer has published, so you need to be able to spend their onchain state to your state (i.e., rebinding your state to the state they published)?
[/quote]

yes. it's about spending an earlier state to the latest you know. presumably you have the settlement transaction details for that.

-------------------------

cguida | 2024-02-11 22:49:38 UTC | #16

@harding See the part of the draft spec [here](https://github.com/instagibbs/bolts/blob/a17b60f42077a785c625430e8f6e8e2828d4d898/XX-eltoo-transactions.md#rationale-1) where instagibbs uses the annex to store the tapleaf for the previous settlement transaction, inside the witness for the invalidated update transaction. Without the tapleaf for the old settlement transaction, there's no way to spend the old update transaction into the new update transaction, without storing every old settlement transaction's output amounts and htlcs (or, tapleaves) in the database, which Symmetry tries to avoid.

-------------------------

moonsettler | 2024-02-15 10:44:20 UTC | #17

*copy pasta from the CAT github PR for those that don't follow:*

Some of us have been playing with the idea of neutered CAT: instead of the MAX_SCRIPT_ELEMENT_SIZE (520 bytes) the maximum output size would be 80 bytes.

**Rationale:**
It explicitly disables open ended second order effects by removing the ability to assemble a SIGHASH or CTV template on the stack for more detailed introspection than intended. If such introspection behavior is desired in the future it can be explicitly enabled by specialized and more efficient opcodes.

Combined with CSFS it can be used as a signed datacarrier, for which the 'standard' limit is 80 bytes, it also has to be smaller than 84 bytes which is required for building CTV templates on the stack, and larger than 64/65/72 bytes which are respectively needed for:
- Merkle inclusion: 2x 20/32 byte hashes
- LN-symmetry (LNhance): 2x 32 byte hashes
- Separate sig: 64/72 bytes (0-conf bonds, staking contracts)

Could be a livable compromise between the conservatives that want to preserve certain characteristics of bitcoin and the prometheans who want to give the developers more practical and useful tools to build with.

-------------------------

salvatoshi | 2024-02-16 10:27:36 UTC | #18

I [commented](https://github.com/bitcoin/bitcoin/pull/29247#issuecomment-1948016168) on github on why that does not prevent the CAT tricks.

-------------------------

