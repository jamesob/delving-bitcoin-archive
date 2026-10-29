# Multiplication in Tapscript today: 371 bytes, no soft fork

Laz1m0v | 2026-10-29 19:20:36 UTC | #1

`OP_MUL` has been disabled since 2010. Under BIP-342, any product has to be built from `OP_ADD` and shifts, and every intermediate that is later consumed by another arithmetic opcode must still fit in a 4-byte `CScriptNum`. The comment in `src/script/script.h` is the usual one: operands must be in `[-2^31+1 … 2^31-1]`, results may overflow, but the next numeric opcode that reads them re-parses with `nDefaultMaxNumSize = 4` and throws if the magnitude exceeds \(2^{31}-1\).

The tightest symmetric bound that survives that rule is

\[
L_{\max} = \lfloor\sqrt{2^{31}-1}\rfloor = 46{,}340
\]

so \(46{,}340^2 = 2{,}147{,}395{,}600 < 2^{31}-1\) (margin 88,047). One more and the next opcode fails. The same core also accepts the asymmetric domain \((x, y) \in [0, X_{\max}] \times [0, 65{,}535]\) when \(X_{\max} \cdot 65{,}535 \le 2^{31}-1\).

I am releasing a standalone 371-byte unrolled Horner multiplication core that stays inside that bound for \(x, y \in [0, 46{,}340]\) (along with its 375-byte and 377-byte Euclidean division extensions, plus the 275-byte reduction that drops explicit bit guards under consensus `MINIMALIF`). The Gist includes a 793-line Rocq 9.3.0 proof, a single-file zero-crate Rust auditor, a zero-dependency Python reference, and a live `bitcoind -regtest` harness.

The construction is witness-assisted. The spending transaction supplies the 16 bits of \(y\) (LSB at the bottom). The 4-byte prologue (`6b 6b 00 00`) parks \(y\) and \(x\) on the altstack and pushes two zero accumulators, then the script processes the bits from MSB to LSB while maintaining the product \(P\) and a reconstructed multiplier \(Y\). On a 1-bit it adds \(x\) to \(P\) and 1 to \(Y\); on a 0-bit it adds nothing. After the conditional it doubles both (except on the final bit). The 5-byte epilogue (`6c 75 6c 7b 9d`) drops \(x\), checks that the reconstructed \(Y\) matches the parked \(y\) via `OP_NUMEQUALVERIFY`, and leaves the product on the main stack.

The 23-byte intermediate round is:

```text
7b 76 63 51 9d 6c 76 6b 93 7c 51 93 7c 67 00 9d 68 76 93 7c 76 93 7c
```

Fifteen of those (345 B) + one 17-byte final round omitting the 6-byte doubling suffix (`76 93 7c 76 93 7c`) + 4-byte prologue + 5-byte epilogue = 371 bytes (`SHA-256: 99f5d9766b0e5fb191ca1c6ea3b8052934246832b0dbe8ca3a0a526690cac8fe`). Only 13 active opcodes in the core. No `OP_PICK`, no disabled opcodes, no `OP_SUCCESSx`. Peak combined stack depth is 21 items. BIP-342 removed the old 201 non-push opcode limit, so the variable opcode count (\(211 + 8 \cdot \mathrm{HW}(y) \in [211, 339]\)) is not a consensus issue.

The 371-byte core keeps the explicit `OP_1 OP_NUMEQUALVERIFY` (`51 9d`) and `OP_0 OP_NUMEQUALVERIFY` (`00 9d`) guards inside each branch so the mechanized soundness proof does not depend on `SigVersion`. Under current BIP-342 those guards are redundant because consensus `MINIMALIF` already rejects any `OP_IF` input other than `[]` or `0x01`. Stripping `76`, `51 9d`, and `67 00 9d` from all 16 rounds yields a 275-byte variant (17 B per intermediate round: `7b 63 6c 76 6b 93 7c 51 93 7c 68 76 93 7c 76 93 7c`, 11 B final round). The difference is 96 WU (24 vB).

Two bugs only showed up when testing against actual `interpreter.cpp`. First, an early draft of the 10-byte bound check used `0xa4` (`OP_MAX`) instead of `0xa2` (`OP_GREATERTHANOREQUAL`); `OP_MAX(0, 0)` evaluates to zero, which is falsy and causes `OP_VERIFY` to reject the valid \((0, 0) \to 0\) input, and the same draft encoded 46,341 in Little-Endian as `03 f5 b4 00` (46,325) instead of `03 05 b5 00`. Both are fixed in the 10-byte `LOT_BOUND_VERIFY` prefix (`76 00 a2 69 03 05 b5 00 9f 69`). Second, because that guard consumes its argument (\(\Delta|\sigma| = -1\)), both operands must be pre-duplicated with `OP_DUP` (`76`) before being guarded if they are still needed for the multiplication, and the test script finishes with `OP_NUMEQUAL` (`9c`) to satisfy BIP-342 `CleanStack`. The complete self-contained TapLeaf spent in the regtest harness is therefore 396 bytes (with odd tweaked parity `0xc1` under the standard BIP-341 \(P_{\text{NUMS}}\) internal key `0x50929b74...`).

The Coq file (`TapscriptMul16Soundness.v`) has six closed theorems (`Closed under the global context`, zero `Admitted`, zero axioms). The key result is the linearity reduction: control flow depends only on the bits of \(y\), while the product accumulator is bilinear (\(P = x \cdot Y\)) and monotonically non-decreasing for \(x \ge 0\). Sweeping the 46,341 values of \(y \in [0, 46{,}340]\) at the ceiling \(x = 46{,}340\) is therefore sufficient to certify all \(46{,}341^2 = 2{,}147{,}488{,}281\) pairs in \([0, 46{,}340]^2\). The single-file Rust auditor runs that sweep in 54 ms and kills 455 mutants (371 single-byte bytecode mutations, 16 bit-flip desyncs, 48 non-binary witness values, and 20 domain/Euclidean boundary mutations). The regtest harness confirms the positive and negative vectors against Bitcoin Core `v31.1.0` via `testmempoolaccept`.

The two 377-byte artifacts in the Gist (`mul16_377b` appending `6c 93 6c 9d 55 75` for exact Euclidean verification \(Q \cdot D + R == N\) at \(D=5\), and `mul16_377b_divconst` appending `7b 93 7c 9f 91 69` for the floor inequality \(Q \cdot D + R \le N\)) are optional extensions that share the exact same 371-byte prefix.

What it does not do: no transaction introspection by itself, not arbitrary precision (requires multi-limb composition for wider integers), and executed opcode count depends on the Hamming weight of \(y\). On L1 the witness is public upon broadcast so the branch length is not a side-channel; off-chain use over private witnesses should still account for it.

Questions I would like people who know the interpreter better than I do to look at:

1. Is paying 96 WU for `SigVersion`-independent bit guards (371 B) worth it, or should production leaves drop them and ship the 275 B `MINIMALIF` version?
2. Can the 23-byte (or 17-byte) round, or the 10-byte bound check, be compressed further without `OP_PICK`?
3. Is there a signed-offset or alternative encoding that pushes the single-limb symmetric ceiling past 46,340 while staying inside 4-byte `CScriptNum`?
4. If a future soft fork brings 64-bit arithmetic or re-enables `OP_MUL`, does an unrolled gadget remain useful as a `SigVersion::TAPSCRIPT` fallback, or does it become pure dead weight?

 https://github.com/Laz1mov/tapscript-mul16-unrolled The README has the four reproduction commands. Any surviving mutant or proof gap will be credited and added to the suite.

-------------------------

