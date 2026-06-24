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

gmaxwell | 2025-08-25 21:45:56 UTC | #6

Cute example!   Though if I consider it practically rather than just as an example: how big is the resulting compiled bytecode?   If it’s not tiny the consequence is that you could just leave it out, and overpay the fee by the amount required for the auto-bumping code and come out ahead.

-------------------------

instagibbs | 2025-08-26 14:01:55 UTC | #7

Compiling that example from the SimplicityHL repo, it gives me what I think would amount to 447WU for the program, without witness data.

Not a SimplicityHL question per se but one question I’ve had is how much the serialization of the Simplicity code itself can be compressed without violating the actual costing of the underlying JET operations, since f.e. [output amount](https://docs.rs/simfony-as-rust/latest/simfony_as_rust/jet/transaction/fn.output_amount.html) is only 298mWU but the serialized version of that script will be a couple order of magnitudes larger (15WU if I’m doing this right).

-------------------------

schoen | 2026-06-23 22:10:44 UTC | #8

I'm happy to say that we now have a Discourse forum of our own for Simplicity. You can find it at https://community.simplicity-lang.org/.

Simplicity has come a long way since @sanket1729 posted this background last year. We have more developer tools, better documentation, and some more example contracts (including a lending contract by some of our colleagues which enables lending at interest against collateral, especially relevant for Liquid where we have multiple asset types, so the loan principal and collateral can be different kinds of asset).

One big technical addition since last summer is a state commitment mechanism, where the contract can store a state value in Taproot, thus affecting the contract's own on-chain address. The introspection mechanisms in Simplicity let a contract confirm that an assertion about its mutable state matches the committed state value in Taproot which in turn was used to derive the contract's address.

I wrote a demo of this in the form of a covenant that requires three separate transactions in order to withdraw locked funds (with the state commitment counting how many prior transactions have taken place).

```
/*
 * "Third Time's The Charm" covenant demonstrating Simplicity state management 
 *
 * This covenant requires three transactions in a row in order to release the
 * locked counts. It counts how many of these transactions have been seen
 * already using the witness::STATE value.
 *
 * State management:
 * Computes the "State Commitment" — the expected Script PubKey (address) 
 * for a specific state value.
 *
 * HOW IT WORKS:
 * In Simplicity/Liquid, state is not stored in a dedicated database. Instead, 
 * it is verified via a "Commitment Scheme" inside the Taproot tree of the UTXO.
 *
 * This function reconstructs the Taproot structure to validate that the provided 
 * witness data (state_data) was indeed cryptographically embedded into the 
 * transaction output that is currently being spent.
 *
 * LOGIC FLOW:
 * 1. Takes state_data (passed via witness at runtime).
 * 2. Hashes it as a non-executable TapData leaf.
 * 3. Combines it with the current program's CMR (tapleaf_hash).
 * 4. Derives the tweaked_key (Internal Key + Merkle Root).
 * 5. Returns the final SHA256 script hash (SegWit v1).
 *
 * USAGE:
 * - For load(), we verify: CalculatedHash(witness::STATE) == input_script_hash.
 * - For store(), we verify: CalculatedHash(updated_state) == output_script_hash.
 * - This assertion proves that the UTXO is "locked" not just by the code, 
 *   but specifically by THIS instance of the state data. So the on-chain address
 *   necessarily changes after each state-updating transaction in order to reflect
 *   a cryptographic commitment to the appropriate state data.
 *
 * Surrounding context tracks the state_data, interpreting its low 64 bits as
 * a counter. The counter can be updated by an "update" transaction sending
 * value back to the same contract. When the counter is equal to 2 or more,
 * the "withdraw" transaction is permitted, sending some or all value
 * to an arbitrary output address instead of back to the contract. Both the
 * "update" and "withdraw" transactions must include a valid signature by
 * the authorized key (here hardcoded in check_sig() for demonstration
 * purposes).
 *
 * When initially funding the contract, use witness::STATE = 0 and
 * calculate the contract's initial address with state equal to
 * 0000000000000000000000000000000000000000000000000000000000000000.
 *
 * For educational purposes, this implementation interprets the contents
 * of the state commitment u256 directly as a numeric integer value. A
 * more general best practice is to make the state commitment a SHA256
 * hash of one or more stored values in some predetermined sequence
 * or structure, including a hashed concatenated list or a Merkle
 * root (representing commitments to multiple values as a Merkle tree
 * is a ubiquitous practice in Bitcoin). The committed values needed
 * to reconstruct the state commitment hash would be provided in the
 * witness and the contract would perform whatever calculations are
 * needed to confirm that the hash is correct. This is in addition to
 * the hashing performed inside the script_hash_for_input_script(),
 * which is needed at a lower layer of the state commitment mechanism.
 */

fn check_sig(sig: Signature) {
    let authorized_key: Pubkey = 0x79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798; // 1 * G
    let msg: u256 = jet::sig_all_hash();
    jet::bip_0340_verify((authorized_key, msg), sig);
}

fn script_hash_for_input_script(state_data: u256) -> u256 {
    // This is the bulk of our "compute state commitment" logic from above.
    let tap_leaf: u256 = jet::tapleaf_hash();
    let state_ctx1: Ctx8 = jet::tapdata_init();
    let state_ctx2: Ctx8 = jet::sha_256_ctx_8_add_32(state_ctx1, state_data);
    let state_leaf: u256 = jet::sha_256_ctx_8_finalize(state_ctx2);
    let tap_node: u256 = jet::build_tapbranch(tap_leaf, state_leaf);

    // Compute a taptweak using this.
    let bip0341_key: u256 = 0x50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0;
    let tweaked_key: u256 = jet::build_taptweak(bip0341_key, tap_node);
    
    // Turn the taptweak into a script hash
    let hash_ctx1: Ctx8 = jet::sha_256_ctx_8_init();
    let hash_ctx2: Ctx8 = jet::sha_256_ctx_8_add_2(hash_ctx1, 0x5120); // Segwit v1, length 32
    let hash_ctx3: Ctx8 = jet::sha_256_ctx_8_add_32(hash_ctx2, tweaked_key);
    jet::sha_256_ctx_8_finalize(hash_ctx3)
}

fn load(state_data: u256) {
    // Assert that the input state is correct, i.e. "load".
    assert!(jet::eq_256(
        script_hash_for_input_script(state_data),
        unwrap(jet::input_script_hash(jet::current_index()))
    ));
}

fn store(new_state: u256) {
    // Assert that the output state is correct, i.e. "store".
    assert!(jet::eq_256(
        script_hash_for_input_script(new_state),
        unwrap(jet::output_script_hash(jet::current_index()))
    ));
}

fn update(sig: Signature, state_data: u256) {
   // In this case, if the signature is correct, we approve the transaction
   // if the destination is the same contract but with a correct internal
   // state update that increases the count by 1.

   // Check that the signature is correct.
   check_sig(sig);

   let (state1, state2, state3, count): (u64, u64, u64, u64) = <u256>::into(state_data);
   let (carry, new_count): (bool, u64) = jet::increment_64(count);

   // Check for overflow.
   assert!(jet::eq_1(<bool>::into(carry), 0));

   // Assert that the output is being sent to a correctly-updated copy of this
   // specific program, i.e. "store".
   let new_state: u256 = <(u64, u64, u64, u64)>::into((state1, state2, state3, new_count));
   store(new_state);

   // Assert that there are exactly two outputs in the
   // currently-proposed transaction (corresponding to the new contract
   // and the network fee payment). Without this logic, the updater
   // could cause coins to leak out of the covenant by sending some of
   // the input value to an uncontrolled output address that is not a
   // copy of this contract.

   assert!(jet::eq_32(jet::num_outputs(), 2));
   assert!(unwrap(jet::output_is_fee(1)));

   // (Even with this logic, the fee amount itself is not constrained
   // here, and the updater could choose to give away some or all of
   // the stored value to miners in the form of an excessive fee. It
   // would be preferable as a matter of caution to assert that the
   // output asset and amount of output 0 match the asset and amount of
   // the current input. That rule would implicitly require the user
   // constructing the spend transaction to pay the fee, instead of
   // permitting the fee to be paid out of the contract's assets.)
}

fn withdraw(sig: Signature, state_data: u256) {
   // In this case, if the signature is correct, and the count is
   // already at least 2, we approve the transaction (allowing the
   // destination(s) and amount(s) indicated by the proposer).

   // Check that the signature is correct.
   check_sig(sig);

   // Assert that the count from the provided state is already at least 2.
   let (_, _, _, count): (u64, u64, u64, u64) = <u256>::into(state_data);
   assert!(jet::le_64(2, count));
}

fn main() {
    let state_data: u256 = witness::STATE;

    // Assert that the provided state_data is correct according to this
    // program's cryptographic commitment.
    load(state_data);

    match witness::UPDATE_OR_WITHDRAW {
        Left(sig: Signature) => update(sig, state_data),
        Right(sig2: Signature) => withdraw(sig2, state_data),
    }
}

```

Anyway, I'm happy to talk about any aspect of Simplicity over on our new forum devoted to it!

-------------------------

