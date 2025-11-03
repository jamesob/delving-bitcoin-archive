# Delving Simplicity Part Ⅳ:Two Side Effects

roconnor-blockstream | 2025-11-03 18:58:48 UTC | #1

In [Part Ⅲ](https://delvingbitcoin.org/t/delving-simplicity-part-building-data-types/1956) of this series, we showed how to build some data structures and computations using Simplicity’s core set of combinators. As we noted in [Part Ⅱ](https://delvingbitcoin.org/t/delving-simplicity-part-combinator-completeness-of-simplicity/1935), the core combinators are enough to implement any finite pure computation. This raises the question: what more can be achieved? We can add additional side effects to our expressions.

There are various kinds of possible side effects for expressions: state update, writing to a log, throwing an exception, reading from an environment, calling a continuation, etc.  The side effects available in Simplicity will depend on the application.

For Bitcoin and Liquid applications, we currently have two side effects: the Failure effect, which is an exception effect where the exception has type `𝟙`, and the Reader effect which allows data from the transaction environment to be accessed. Our core combinators are “pure”; they have no side effects. However, jets can introduce new primitives that do have side effects.

# Jets with Effects

We will talk more about jets later in this series, but here we introduce a few example jets to illustrate their side effects.

## Bip0340-verify

`bip0340-verify : (𝟚²⁵⁶ × 𝟚²⁵⁶) × 𝟚⁵¹² ⊢ 𝟙` is a jet for an expression that takes an x-only pubkey, a 256-bit message, and a Schnorr signature, and returns nothing! According to its type, it ought to behave the same as a `unit`.  The difference lies in the jet’s side effect: if the signature validation fails, then the entire computation is aborted by throwing an exception (of unit type). This is the Failure effect.

## Verify

`verify : 𝟚 ⊢ 𝟙` is a barebones jet for expressing the Failure effect.  If `verify`’s input is `false`, the entire computation is aborted, by throwing an exception. If the input is `true`, nothing is returned, but the computation can continue.

## Transaction Hashes

`sig-all-hash : 𝟙 ⊢ 𝟚²⁵⁶` appears to be a constant function, since there is only one possible input value: the empty tuple.  However, this jet reads from the transaction environment and produces a hash of transaction data that is analogous to the `SIGHASH_ALL` message digest used in Bitcoin Script’s signature verification. This is an example of the Reader effect: the value returned depends on the transaction environment that the jet is executed within.  There are several other hashing jets that hash various subsets of the transaction environment data to help build custom message digests for signatures.

## Introspection Jets

`input-sequence : 𝟚³² ⊢ 𝟚³²?` is a function that takes an input index and returns the transaction’s sequence number for that input, optionally returning nothing if the index is out of bounds.  Again, the output value is not a pure function of the input index, but rather, the operation uses the Reader effect to access the transaction environment in order to determine the output value.  There are several other introspection jets that return various fragments of the transaction environment data.

# Classifying Effects

Not all side effects are created equal. Some side effects behave nicer than others. We can classify effects by how amenable they are to program transformations.

## Commutative Effects

A commutative effect is one where, if you swap the outputs of two expressions, you can safely swap the expressions themselves without changing the expression’s effect. Consider `swap = I H ▵ O H : A × B ⊢ B × A`. If `f ▵ g ⨾ swap = g ▵ f` for every expression `f` and `g` with side effects, then the effects are commutative.

Reading transaction data from the environment is a commutative effect because the result of reading from the environment is the same, no matter what order we execute the reading in.

In general, throwing an exception is not a commutative effect. If `f` throws some exception `e₁` and `g` throws some other exception `e₂`, then which exception is thrown from the pair of `f` and `g` depends on the order they are executed in.

However, in the special case of the Failure effect, in which only a unit typed exception can be thrown, the effect is commutative.  No matter which of `f` or `g` throws an exception, the resulting exception will be the same, because there is only one possible exception value.

## Idempotent Effects

An idempotent effect is one where, if you duplicate the output of an expression, you can safely duplicate the expression itself without changing the expression’s effect.  Consider `dup = iden ▵ iden : A ⊢ A x A`. If `f ⨾ dup = dup ⨾ f ▵ f` for every `f` with side effects, then the effects are idempotent.

Reading transaction data from the environment is an idempotent effect. Throwing an exception is also an idempotent effect. Even though only one of the two duplicated expressions will be executed, any exception thrown by `dup ⨾ f ▵ f` will be the same as the exception thrown by `f ⨾ dup`.

However, writing to a log may not be idempotent, as duplicating the effect would cause the log message to appear twice. However, if the log consists of a *set* of messages instead of a *list* of messages, then the effect would be idempotent (and commutative) because set insertion is itself an idempotent operation.

## Unitary Effects

A unitary effect is one where, if you discard the output of an expression, you can safely discard the expression itself without changing the expression’s effects. If it is always the case that `f ⨾ unit = unit` for every `f` with side effects, then your effects are unitary.

Reading data from the environment is one of the few types of unitary effects. If the result of reading transaction data from the environment is discarded, the whole expression performing the read may be discarded.

The failure effect isn’t unitary. If `f` throws an exception then so will `f ⨾ unit`; execution will not even make it to the `unit` combinator before the computation is aborted.  On the other hand, `unit` obviously would not throw any exception, so the effects of `f ⨾ unit` and `unit` would be different.

# Effects Allowed in Simplicity

The more well-behaved properties that a type of effect has, the more room a Simplicity optimizer has for transforming programs that use those effects.  Ideally we would only allow effects that have all three properties: commutative, idempotent, and unitary.  This would allow an optimizer to perform any sort of program transformation it would like.  However, reading from an environment is the only effect that satisfies all three properties.

Instead we demand that Simplicity effects are commutative and idempotent. Both the effects we use in Simplicity, the Failure effect and the Reader effect, are commutative and idempotent. This allows a large class of optimizations to be performed on Simplicity code.

However, the “discard” transformation described above, attempting to replace `f ⨾ unit` with `unit`, or any similar transformation is not allowed if `f` may produce a Failure effect. Indeed, imagine if `f` contained a `bip0340-verify` assertion. It would be disastrous to attempt to optimize that check away.

# Why Allow Side Effects At All?

Why does Simplicity even allow side effects at all?  Wouldn’t it be better if every program took the entire transaction as input and returned a Boolean output that decides if a transaction is valid or not?

## Batch Verification

One reason we have the Failure effect is to support [batch verification](https://github.com/bitcoin/bips/blob/c9a6ca6297eb8de850f6b64dafb8e60ee9b64d66/bip-0340.mediawiki#batch-verification) of Schnorr signatures.  In batch verification, many individual Schnorr signature checks are pooled together in such a way that if any single signature check fails, then the entire batch fails.

This batching procedure improves efficiency over individually verifying each signature. The downside is that if the batch verification fails, then we do not learn which specific signature check or checks failed.

By using the failure side effect, `bip0340-verify` ensures that if a signature check fails, the whole transaction fails.  If `bip0340-verify` were instead to return `𝟚`, a Boolean type, for success or failure, then a failing signature check could still lead to a branch where the script succeeds. In such a case we would need to know if the particular signature is valid or not, and thus we wouldn’t be able to take advantage of batch verification.

## Precomputed Transaction Data

A problem in early Bitcoin Script was that the hashing function used to create message digests for signatures was linear in the size of the transaction.  Typically every input creates at least one message digest for signature verification, so overall the amount of hashing was quadratic in the transaction size.

This problem was fixed in Segwit and later iterations of Bitcoin Script by redefining the message digests so that they could be computed in constant time per signature check. This relies on having `PrecomputedTransactionData`, which precomputes hashes of transaction data once and is then shared by each input’s sighash computations. Simplicity’s transaction hashing jets rely on the same kind of precomputed transaction data in order to ensure the jets run in constant time.

Suppose `sig-all-hash` didn’t use the Reader effect.  Suppose we somehow managed to build a Simplicity type for the transaction environment.  Let’s call it `TxEnv`, so that `sig-all-hash : TxEnv ⊢ 𝟚²⁵⁶` was the jet’s type. Such a definition would require the `sig-all-hash` jet to be able to compute the hash of any transaction, not just the transaction it is involved with. Simplicity programs could copy the given \`TxEnv\` and pass a modified copy of it to `sig-all-hash`. In such a case `sig-all-hash` couldn't rely on `PrecomputedTransactionData`, and we would be back to requiring linear time in whatever transaction data was passed into this version of `sig-all-hash`.

Because `sig-all-hash : 𝟙 ⊢ 𝟚²⁵⁶` uses the Reader effect to access the transaction data, it *only* gets access to a fixed  transaction environment. For that reason, the jet’s implementation can safely use `PrecomputedTransactionData` and operate in constant time.

# Cross-Input Signature Aggregation

While neither Liquid nor Bitcoin support [cross-input signature aggregation](https://hrf.org/latest/cisa-research-paper/) at this point in time, we would like to check that Simplicity can be compatible with it when the time comes.

While details haven’t been worked out, we imagine half-aggregation being implemented using a Writer effect.  That is, a new jet with a type such as `half-agg-verify : (𝟚²⁵⁶ × 𝟚²⁵⁶) × 𝟚²⁵⁶ ⊢ 𝟙` would take a public key, message digest, and the `r`-component of a Schnorr signature (a Schnorr signature consists of an `r`-component and an `s`-component) and write it to a transaction log before continuing on with execution. Then, elsewhere in the transaction or with the transaction, an aggregate `s`-component for all half-aggregated Schnorr signatures would be provided.  The transaction would only be valid when such an aggregate `s`-component is provided for all the logged keys, messages, and `r`-components.

To meet Simplicity’s requirements, this Writer effect needs to be idempotent and commutative.  This can be ensured by treating the writer log as a set of key, message, `r`-component tuples. This works because set operations are idempotent and commutative. Treating the log as a set of values would be compatible with the half-aggregation verification algorithm.

# Conclusion

In this part we looked at adding side effects to the computations that Simplicity can do.  We classified various kinds of effects according to how well-behaved they are with respect to various kinds of program transformation.  We decided to restrict Simplicity’s effects to those that are commutative and idempotent.

The two effects we use for Bitcoin and Liquid applications are the Reader effect, for accessing the transaction environment, and the Failure effect, for aborting and failing the program. Some jets make use of primitive operations where these sorts of side effects can occur.

The Failure effect determines the output of a Simplicity program: the program either fails, making the transaction invalid, or the program succeeds. The Reader effect provides one sort of input to a Simplicity program: the environment containing transaction data. But we also need to provide other inputs, such as digital signatures, to Simplicity programs.

In the next part we will look at what Simplicity programs are, how they are turned into addresses, and how we add other inputs, such as signatures, to Simplicity programs.

-------------------------

