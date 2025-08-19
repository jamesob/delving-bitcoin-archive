# Writing Simplicity Programs with SimplicityHL

sanket1729 | 2025-08-04 19:21:46 UTC | #1

## Background on Simplicity

Simplicity is a low-level programming language and computational model designed for blockchain smart contracts. Its design is minimal, simple enough that the core semantics fit on a T-shirt, and it's formally verifiable. But that simplicity does not make it easy to use. Simplicity is functional and low-level, and writing it feels more like writing assembly than something like Java or Python. It is powerful, but not ergonomic for everyday development.

Simplicity was recently activated on the Liquid mainnet, making it now possible to write and deploy contracts using this language in production environments.

## SimplicityHL

SimplicityHL is a high-level language for writing smart contracts on Elements, and Liquid. It looks and feels a lot like Rust. Just as Rust compiles down to machine code, SimplicityHL compiles down to Simplicity bytecode. You write SimplicityHL. Full nodes execute Simplicity.

I am not actively working on SimplicityHL at the moment, but I wanted to share an example to show what working with the language actually looks like.

The goal is to show how SimplicityHL makes Simplicity development accessible. This is not a language tour, a discussion about how Simplicity could be added to Bitcoin, or a dive into formal semantics. It is just a practical example of writing a real contract with real behavior.

## A Real Example: Non-Interactive Fee Bumping
 
Bitcoin fee estimation is hard. If your transaction fee is too low, it can sit unconfirmed in the mempool for hours or days. Solutions like RBF, CPFP, anchors, and proposed sponsor transactions all exist, but they require coordination from the sender or a third party to replace or attach another transaction that bumps the fee.

## A Different Approach

In this example, the fee bumping logic is embedded directly into the script. The longer a transaction remains unmined, the more fee it permits. A miner can reduce the change output or increase the fee output and include the transaction in a block. No user action or third-party help is needed.

This works because SimplicityHL gives full flexibility to express spend conditions as functions over the transaction’s data. In this case, we enforce a linear function: as nLockTime increases, the allowed fee increases proportionally. But the same framework can support more complex functions over inputs, outputs, values, and metadata. You are writing a program that evaluates whether a transaction is authorized, and you get full control over that logic.

## The Code

Below is a complete SimplicityHL program that enforces a base fee plus 1 satoshi per second after a fixed broadcast time.


```rust
/*
 * NON-INTERACTIVE FEE BUMPING
 *
 * Anyone, including miners, can increase the fee by reducing the change amount,
 * based on a rule that adds 1 satoshi per second after broadcast.
 *
 * Allowed changes:
 * - nLockTime can increase
 * - change/fee outputs can be modified
 *
 * No need for RBF, CPFP, anchors, or sponsor transactions.
 */

fn sighash_tx_nifb() -> u256 {
    let ctx: Ctx8 = jet::sha_256_ctx_8_init();
    let ctx: Ctx8 = jet::sha_256_ctx_8_add_4(ctx, jet::version());
    let ctx: Ctx8 = jet::sha_256_ctx_8_add_32(ctx, jet::inputs_hash());

    // Include the non-change output (index 0)
    let ctx: Ctx8 = match jet::output_hash(0) {
        Some(sighash: u256) => jet::sha_256_ctx_8_add_32(ctx, sighash),
        None => panic!(),
    };

    let ctx: Ctx8 = jet::sha_256_ctx_8_add_32(ctx, jet::output_scripts_hash());
    let ctx: Ctx8 = jet::sha_256_ctx_8_add_32(ctx, jet::input_utxos_hash());
    jet::sha_256_ctx_8_finalize(ctx)
}

fn sighash_nifb() -> u256 {
    let ctx: Ctx8 = jet::sha_256_ctx_8_init();
    let ctx: Ctx8 = jet::sha_256_ctx_8_add_32(ctx, jet::genesis_block_hash());
    let ctx: Ctx8 = jet::sha_256_ctx_8_add_32(ctx, sighash_tx_nifb());
    let ctx: Ctx8 = jet::sha_256_ctx_8_add_32(ctx, jet::tap_env_hash());
    let ctx: Ctx8 = jet::sha_256_ctx_8_add_4(ctx, jet::current_index());
    jet::sha_256_ctx_8_finalize(ctx)
}

fn check_neg(v: bool) {
    let v1: u1 = <bool>::into(v);
    let v2: u64 = <u1>::into(v1);
    assert!(jet::eq_8(v2, 0));
}

// Enforces a linear fee increase over time
fn total_fee_check() {
    let curr_time: u32 = jet::tx_lock_time();
    let fee_asset: ExplicitAsset = 0x0000000000000000000000000000000000000000000000000000000000000000;
    let fees: u64 = jet::total_fee(fee_asset);

    let time_at_broadcast: u32 = 1734967235; // Dec 23, ~8:33am PST
    let (carry, time_elapsed): (bool, u32) = jet::subtract_32(curr_time, time_at_broadcast);
    check_neg(carry);

    let base_fee: u64 = 1000;
    let (carry, max_fee): (bool, u64) =
        jet::add_64(base_fee, jet::left_pad_low_32_64(time_elapsed));
    check_neg(carry);

    assert!(jet::lt_64(fees, max_fee));
}

fn main() {
    let sighash: u256 = sighash_nifb();
    total_fee_check();
    let alice_pk: Pubkey = 0x9bef8d556d80e43ae7e0becb3a7e6838b95defe45896ed6075bb9035d06c9964;
    jet::bip_0340_verify((alice_pk, sighash), witness::ALICE_SIGNATURE);
}
```
## Final Thoughts

If you are familiar with Rust, most of this will feel straightforward. You do not need to understand Simplicity’s internal encoding or read through its formal semantics to get started. You can inspect the available jets, write logic around them, and compile.

Simplicity gives you a strong foundation for secure and auditable smart contracts. SimplicityHL gives you a way to actually build them.

This post focuses only on writing logic in SimplicityHL. It does not cover how this might affect miners incentives for this fee bumping model, how wallets might index these scripts, how this integrates with descriptors, or how it could be supported across the stack. These are important questions but separate from the programming model itself.

The goal here is to highlight that writing these types of contracts is possible today.

### Resources:

- SimplicityHL: https://github.com/BlockstreamResearch/SimplicityHL
- rust-simplicity: https://github.com/BlockstreamResearch/rust-simplicity

-------------------------

niftynei | 2025-08-05 15:32:01 UTC | #2

Nice. A reverse dutch auction for fees on a transaction is such a neat idea. 

This is my first time really looking at a simplicityHL contract, and it's surprising me how much it resembles "normal code". 

Am I right in reading this as including a custom sighash implementation? That's a neat trick.

-------------------------

sanket1729 | 2025-08-06 22:37:56 UTC | #3

[quote="niftynei, post:2, topic:1900"]
Am I right in reading this as including a custom sighash implementation? That’s a neat trick.
[/quote]

The current version of SimplicityHL requires committing to programs at the time of address creation. However, it is also possible to implement this behavior using a sighash check, which allows the signer to make this choice at signing time instead of during address setup. This approach is enabled by a Simplicity extension called *delegation*. Currently, SimplicityHL programs do not utilize the [universal sighash](https://blog.blockstream.com/simplicity-taproot-and-universal-sighashes/) mode described below. While there are no technical barriers to implementing this as a more flexible sighash based check, it has not been implemented yet.


> The key insight is that sighash modes, unlike any other aspect of Bitcoin’s Script, allow the user to decide what gets signed *at signing time* rather than at *address generation time* . In Bitcoin, this signing-time ability is limited to setting the sighash mode, but with careful use of Simplicity’s disconnect combinator, we can go much further. We can enable the signer to do much more than fixing various parts of the transaction data. He could fix arbitrary transaction parts not only to specific values, but to certain ranges or subsets, and conditional these restrictions on timelocks being satisfied, external data being signed — or any arbitrary computation! Further, he could delegate these decisions to alternate (sets of) public keys.

-------------------------

niftynei | 2025-08-15 16:12:24 UTC | #4

I’m not sure I follow. 

The universal sighash mode seems to allow for rangeproof style signature commitments or key delegation. In the original code sample you simply create a ‘custom’ sighash based on selective commitment to the transaction contents. It seems like you could also require a commitment to arbitrary data (however the data would have to be committed to in the script at creation time)

Out of curiosity, what would be required for adding the universal sighash to simplicity?

-------------------------

hodlinator | 2025-08-19 19:59:33 UTC | #5

Great to see progress in this area!

Here follows first impressions from someone interested in programming languages but no expert by any means.

---

[quote="sanket1729, post:1, topic:1900"]
```
fn check_neg(v: bool) {
    let v1: u1 = <bool>::into(v);
    let v2: u64 = <u1>::into(v1);
    assert!(jet::eq_8(v2, 0));
}
```

[/quote]

What is happening here? Almost looks like the language has negative booleans that require a user defined function to check.

---

Why does one use `jet::eq_8()` above with a `u64` parameter when the less-than operator for the same types has the `_64` suffix?

[quote="sanket1729, post:1, topic:1900"]
`   assert!(jet::lt_64(fees, max_fee));`

[/quote]

---

The repeated `let`-binding of `ctx` seems like it could make provability easier, but this language is meant to *compile down* to a provable language. Is there any goal to increase mutability in syntax at this higher level to gain ergonomics, `mut` without `let`, for example:

`jet::sha_256_ctx_8_add_32(mut ctx, sighash_tx_nifb());`

or is there intent behind preferring the current syntax?

---

Again, I think it’s phenomenal that work is being done in this area. Hope this comes off more as curious than ungrateful, and somewhat mirrors first impressions of others.

-------------------------

