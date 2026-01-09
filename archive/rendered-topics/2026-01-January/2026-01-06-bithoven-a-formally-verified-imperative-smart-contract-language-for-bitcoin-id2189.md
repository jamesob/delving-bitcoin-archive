# Bithoven: A Formally Verified, Imperative Smart Contract Language for Bitcoin

ChrisCho-H | 2026-01-06 18:23:50 UTC | #1

Hi everyone,

I’ve been working on a new smart contract language called **Bithoven**, and I wanted to share the v0.0.1 release along with an implementation, Web IDE, and the formal verification paper we just published on arXiv.

While Miniscript excels at policy analysis and structural correctness, its grammar and syntax can feel difficult for developers used to imperative control flow. I wanted to see if we could have a C-like imperative syntax (`if/else`, explicit variables) that still compiles down to safe, optimized Bitcoin Script.

### The Approach

Bithoven uses an `LR(1)` parser and static analysis to enforce type safety and prevent some known security bug(e.g. no sig required to spend) before compilation. The compiler handles the stack management(e.g. `OP_SWAP`), opcode optimization(e.g. `OP_EQUAL + OP_VERIFY = OP_EQUALVERIFY`) and target separation(e.g. `legacy`, `segwit` and `taproot`). To ensure we aren't introducing undefined behavior during that translation, we formally defined the language's semantics and proved its safety properties.

### Example (HTLC)

Here is a quick look at how it handles conditional paths. The stack inputs are defined explicitly at the top to guide the compiler:

```solidity
pragma bithoven version 0.0.1;
pragma bithoven target segwit;

/* * Stack Input Definitions
 * Each line defines a valid input stack configuration for a spending path.
 */
(condition: bool, sig_alice: signature)
(condition: bool, preimage: string, sig_bob: signature)

{
    // If 'condition' is true, we enter the Refund Path (Alice)
    if condition {
        // Enforce relative timelock of 1000 blocks
        older 1000;

        // If timelock is satisfied, Alice can spend with her signature
        return checksig(sig_alice, "0245a6b3f8eeab8e88501a9a25391318dce9bf35e24c377ee82799543606bf5212");

    } else {
        // Redeem Path (Bob)
        // Bob must reveal the secret preimage that hashes to the expected value
        verify sha256(sha256(preimage)) == "53de742e2e323e3290234052a702458589c30d2c813bf9f866bef1b651c4e45f";

        // If hash matches, Bob can spend with his signature
        return checksig(sig_bob, "0345a6b3f8eeab8e88501a9a25391318dce9bf35e24c377ee82799543606bf5212");
    }
}
```

### Design Philosophy (Syntax and Grammar) 

To maintain safety, Bithoven enforces a strict distinction between Statements and Expressions:

Statements must start with followings
- `verify`: This compiles down to `OP_VERIFY`. I designed statements to consume stack items without leaving leftovers. This simplifies stack management, ensuring that after every statement, the stack is clean. As every statement starts with `verify`, any expression argument of `verify` is available, which is evaluated to value(pushing the item on the stack, but removed by `verify` at the end).
- `older` and `after`: This is same with miniscript policy. I allowed it as statement as it doesn't leave any on the stack(if with `OP_DROP`), similar to `verify`.
- `return`: The terminal statement for each branch. It ensures the final top stack item is not empty, making the UTXO spendable.

Expressions can express most of bitcoin opcodes except stack managment opcode(excluded intentionally as it's a high level language), with intuitive syntax.

| Category | Bithoven Syntax | Bitcoin Opcode |
| :--- | :--- | :--- |
| **Mathematics** | `+` | `OP_ADD` |
| | `-` | `OP_SUB` |
| | `++` | `OP_1ADD` |
| | `--` | `OP_1SUB` |
| | `max` | `OP_MAX` |
| | `min` | `OP_MIN` |
| | `negate` | `OP_NEGATE` |
| | `abs` | `OP_ABS` |
| **Logic & Comparison** | `&&` | `OP_BOOLAND` |
| | `\|\|` | `OP_BOOLOR` |
| | `==` | `OP_(NUM)EQUAL` |
| | `!=` | `OP_(NUM)NOTEQUAL` |
| | `<` | `OP_LESSTHAN` |
| | `>` | `OP_GREATERTHAN` |
| | `<=` | `OP_LESSTHANOREQUAL` |
| | `>=` | `OP_GREATERTHANOREQUAL` |
| | `!` | `OP_NOT` |
| **Cryptography** | `sha256` | `OP_SHA256` |
| | `ripemd160` | `OP_RIPEMD160` |
| **Byte Operations** | `len` | `OP_SIZE` |


### Resources

* **Repo:** [https://github.com/ChrisCho-H/bithoven](https://github.com/ChrisCho-H/bithoven)
* **Web IDE:** [https://bithoven-lang.github.io/bithoven/ide/](https://bithoven-lang.github.io/bithoven/ide/)
* **Docs:** [https://bithoven-lang.github.io/bithoven/docs/](https://bithoven-lang.github.io/bithoven/docs/)
* **Paper (arXiv):** [https://arxiv.org/abs/2601.01436](https://arxiv.org/abs/2601.01436) (Contains the operational semantics and correctness proofs)


I would appreciate any feedback regarding the implementation, compiler design, and the formal proofs in the paper. As this is very early stage beta implementation, there could be specific edge cases that we've missed. 

Thanks!

-------------------------

Nuh | 2026-01-07 17:00:27 UTC | #2

This is an interesting alternative to Simplicity, at least less disruptive.

I wonder how would the very hacky things necessary for STARK proofs using OP_CAT look like using this compiler instead, same for transaction introspection with OP_CAT with Schnorr tricks.

-------------------------

ChrisCho-H | 2026-01-09 03:04:28 UTC | #3

Currently, Bithoven v0.0.1 targets strict mainnet safety (SegWit/Taproot), so experimental opcodes like `OP_CAT` are disabled.

**On Merkle Paths:** I plan to overload the `+` operator so that it compiles to `OP_ADD` for integers but switches to `OP_CAT` when operands are bytes/strings(if `OP_CAT` enabled on mainnet). This would allow users to verify Merkle paths naturally (e.g., `sha256(a + b)`) without managing the stack manually.

**On Schnorr Introspection:** While I haven't implemented that specific introspection pattern yet, it would likely be handled similarly—using the compiler to manage the byte concatenation and stack cleanup. If you have a raw script snippet for the introspection logic you are using, I’d love to see it! It would be a great test case for the compiler's experimental branch.

-------------------------

ftw2100 | 2026-01-09 13:28:04 UTC | #4

Hello, 

As it was said, very interesting to have a simplicity alternative to formalise covenant. 
My question is: how do you provide introspection? Do you avoid some access (as the witness in simplicity)?

-------------------------

