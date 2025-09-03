# Delving Simplicity Part Ⅲ: Building Data Types

roconnor-blockstream | 2025-09-02 23:52:41 UTC | #1

In [Part Ⅱ](https://delvingbitcoin.org/t/delving-simplicity-part-combinator-completeness-of-simplicity/1935) of this series, we introduced Simplicity’s type system and its core set of computational combinators. However, with only three kinds of type formers and nine core combinators, the language may seem too meager for practical computations. Can we really build any computation using only the three fundamental forms of composition we described in Part Ⅰ of this series? In the same way that computers are built out of logic gates, we start with the humble bit and build up abstractions from it. Let’s delve in.

# Boolean Logic

In Part Ⅱ, we introduced the Boolean type, which is `𝟙 + 𝟙`. We use the notation `𝟚` for this type. It has two values: `σᴸ⟨⟩` and `σᴿ⟨⟩`. By convention, we use the notation `0` or `false` for `σᴸ⟨⟩`, and the notation `1` or `true` for `σᴿ⟨⟩`. Using our computational combinators, we can build Boolean logic operators.

## And operation

The logical `and : 𝟚 × 𝟚 ⊢ 𝟚` operation takes two bits as input and returns one bit as output. Since Simplicity expressions have only a single input type, we use the product of the two input arguments as the input type. There are several ways of implementing this function. One way is to branch on the first bit: if the first bit is `false`, we return `false`; otherwise, we return the second bit.

```none
and ≔ case (injl unit) (drop iden) : 𝟚 × 𝟚 ⊢ 𝟚
```

We can check this definition on the input `⟨false, false⟩` by using equational reasoning:

```none
⟦and⟧⟨false, false⟩
 = {expand the notation for false}
⟦and⟧⟨σᴸ⟨⟩, σᴸ⟨⟩⟩
 = {expand the definition of and}
⟦case (injl unit) (drop iden)⟧⟨σᴸ⟨⟩, σᴸ⟨⟩⟩
 = {evaluate case for σᴸ}
⟦injl unit⟧⟨⟨⟩, σᴸ⟨⟩⟩
 = {evaluate injl}
σᴸ(⟦unit⟧⟨⟨⟩, σᴸ⟨⟩⟩)
 = {evaluate unit}
σᴸ⟨⟩
 = {by the notation for false}
false
```

We get the expected result of `false`. 

Let’s try it again on the input `⟨true, true⟩`.

```none
⟦and⟧⟨true, true⟩
 = {expand the notation for true and the definition of and}
⟦case (injl unit) (drop iden)⟧⟨σᴿ⟨⟩, σᴿ⟨⟩⟩
 = {evaluate case for σᴿ}
⟦drop iden⟧⟨⟨⟩, σᴿ⟨⟩⟩
 = {evaluate drop}
⟦iden⟧(σᴿ⟨⟩)
 = {evaluate iden}
σᴿ⟨⟩
 = {by the notation for true}
true
```

Again, we get the expected result of `true`. We will leave trying the remaining inputs for the curious reader.

## Other logic operations

The definition of the `not` operation requires a helper combinator:

```none
                   f : A ⊢ C    g : B ⊢ C
-------------------------------------------------------------
copair f g ≔ iden ▵ unit ⨾ case (take f) (take g) : A + B ⊢ C
```

The initial `iden ▵ unit : A ⊢ A × 𝟙`  adds an empty “environment” to the input, enabling the `case` combinator to apply. The use of `take` in the two branches drops this empty environment to execute `f` or `g`.

Now we can define all other Boolean logical operations:

* `or ≔ case (drop iden) (injr unit) : 𝟚 × 𝟚 ⊢ 𝟚`
* `not ≔ copair (injr unit) (injl unit) : 𝟚 ⊢ 𝟚`
* `xor ≔ case (drop iden) (drop not) : 𝟚 × 𝟚 ⊢ 𝟚`

## Bit Adders

In digital logic, a “half-adder” is a circuit that takes two bits and adds them, producing a two-bit output: a carry bit and the sum bit. We can implement this in Simplicity by pairing up the `and` and `xor` operations:

```none
half-adder ≔ and ▵ xor : 𝟚 × 𝟚 ⊢ 𝟚 × 𝟚 
```

This function has two output bits, so we return a pair of the two output types.

A “full-adder” takes an additional third “carry-in” bit and adds them all together, producing the same two-bit output. We must use a nested tuple to represent three inputs. We have two choices: `𝟚 × (𝟚 × 𝟚)` or `(𝟚 × 𝟚) × 𝟚`. These types are not quite the same, and we must choose one. For this example, we will choose the latter.

Types in Simplicity are often nested tuples, representing an “environment” of parameters. It is common to use sequences of `take`, `drop`, and `iden` to access specific parameters from such an environment. For sequences of these combinators, a compact notation will be used:

* `O f`  is notation for `take f`.
* `I f` is notation for `drop f`.
* `H` is notation for `iden`.

For example, `I O H` means `drop (take iden) : A × (B × C) ⊢ B`, which extracts the middle value out of its input.


🛈
The use of `O` and `I` for `take` and `drop` is meant to evoke a sense of binary digits. If we think of a nested tuple as a binary tree and label the tree’s positions with positive binary numbers appropriately, then our notation becomes the reversed binary digits (thinking of `H` as the leading 1 digit) of a location within that tree from which we are retrieving data. Such expressions form a kind of De Bruijn indices for Simplicity

⚠
The `I`, `O`, and `H` notation is only used for subexpressions that consist only of `take`, `drop`, and `iden`. They are not used for other occurrences of `take` or `drop`.

A full-adder is the composition of two half-adders, taking the logical `or` of the two carry bits:

```none
full-adder ≔ take half-adder ▵ I H
           ⨾ O O H ▵ (O I H ▵ I H ⨾ half-adder)
           ⨾ (O H ▵ I O H ⨾ or) ▵ I I H
           : (𝟚 × 𝟚) × 𝟚 ⊢ 𝟚 × 𝟚
```

In the first line, `take half-adder ▵ I H : (𝟚 × 𝟚) × 𝟚 ⊢ (𝟚 × 𝟚) × 𝟚`, we run the half-adder on the first two bits, and we save the last bit. In the second line, `O O H ▵ (O I H ▵ I H ⨾ half-adder) : (𝟚 × 𝟚) × 𝟚 ⊢ 𝟚 × (𝟚 × 𝟚)`, we save the first bit (which is the carry-out bit of the first half-adder) and run the half-adder on the last two bits. In the last line, `(O H ▵ I O H ⨾ or) ▵ I I H: 𝟚 × (𝟚 × 𝟚) ⊢ 𝟚 × 𝟚`, we take the logical *or* of the first two bits (which are the carry-outs of the two half-adders) and also return the sum-out bit of the second half-adder.

This code illustrates how one programs in Simplicity. We use the `I`, `O`, and `H` notation to reference bits of data from the subexpression’s input type, which is then formed into a suitable “environment” that is used to “call” other functions using sequential composition.

Simplicity users do not have to define such low-level operations themselves. Later in this series, we will discuss our standard library of jets that implement common functions.

Furthermore, end users are not expected to directly program in Simplicity, in the same way that end users are not expected to directly program with Bitcoin Script. Instead, we expect end users will use higher-level languages, such as SimplicityHL, to generate Simplicity. For example, SimplicityHL manages the “environment” of each subexpression, letting one use named variables, which are compiled into appropriate sequence of `take`s and `drop`s.

# Vectors

Given a Simplicity type `A`, we can define fixed-length vectors by forming iterated products of `A`:

* `A² ≔ A × A`
* `A⁴ ≔ A² × A²`
* `A⁸ ≔ A⁴ × A⁴`
* `…`

These types may be alternatively written as `A^2`, `A^4`, `A^8`, etc.

We will define vectors only for lengths that are powers of two. One can define notation for other powers, but a convention needs to be chosen for how to bracket the product types.

Given an expression `f : A ⊢ B`, we can repeatedly pair it with itself in order to “map” it over a vector of any fixed length:

* `f² ≔ f ▵ f : A² ⊢ B²`
* `f⁴ ≔ f² ▵ f² : A⁴ ⊢ B⁴`
* `f⁸ ≔ f⁴ ▵ f⁴ : A⁸ ⊢ B⁸`

Given a function `f : A × B ⊢ B`, we can iterate or “fold” it over a vector of any fixed length:

* `fold-right-2 f ≔ O O H ▵ (O I H ▵ I H ⨾ f) ⨾ f : A² × B ⊢ B`
* `fold-right-4 f ≔ fold-right-2 (fold-right-2 f) : A⁴ × B ⊢ B`
* `fold-right-8 f ≔ fold-right-2 (fold-right-4 f) : A⁸ × B ⊢ B`

There are many variations of these constructions we can implement. Given `f : A × B ⊢ C`, we can “zip” it over a pair of vectors with `zip-n f : (Aⁿ × Bⁿ) ⊢ Cⁿ`. Given `f : (A × B) × C ⊢ C`, we can fold it over a pair of vectors with `bifold-right-n f : (Aⁿ × Bⁿ) ⊢ C`. We can combine `map` and `fold-right` into an accumulating combinator that takes `f : A × C ⊢ C × B` and results in `map-accum-right-n f : Aⁿ × C ⊢ C × Bⁿ`. Many more variants are possible.

## Multi-bit Words

A vector of bits gives us multi-bit integers. For example, `𝟚³²` is a 32-bit word type. `𝟚²⁵⁶` is a 256-bit word type, which is suitable for hashes and other cryptographic operations.

Given our “full-adder” on bits, we can use a variant of the vector operations above to define a “ripple carry adder” over multi-bit words:

```none
full-adder-n ≔ zip-accum-right-n full-adder : (𝟚ⁿ × 𝟚ⁿ) × 𝟚 ⊢ 𝟚 × 𝟚ⁿ
```

`full-adder-n` takes two n-bit binary numbers and a one-bit carry-input value and returns a one-bit carry-out flag and an n-bit sum.

### SHA-256

In this manner, we can recursively define all our arithmetic operations on multi-bit words: subtraction, multiplication, division, etc. We can define bit-wise logical operations such as logical *and*, logical *or*, logical *xor*, etc. By repeatedly combining these operations, we can even build SHA-256’s block compression function: 

`sha256-hash-block ≔ … : 𝟚²⁵⁶ × 𝟚⁵¹² ⊢ 𝟚²⁵⁶`

But you don’t have to trust us. [The SHA-256 compression is formally defined using Simplicity](https://github.com/BlockstreamResearch/simplicity/blob/a619ab60d1c1e4c67b7e8680237ec811215f7cbc/Coq/Simplicity/SHA256.v#L93-L141) within the Rocq proof assistant (formerly named the Coq proof assistant), and it comes with [a formal proof that the `sha256-hash-block` implementation is correct](https://github.com/BlockstreamResearch/simplicity/blob/a619ab60d1c1e4c67b7e8680237ec811215f7cbc/Coq/Simplicity/SHA256.v#L235-L236).

Of course, the compression runs too slowly to be practical when executing it as raw Simplicity. We will introduce jets later in this series, which are used to execute common functions such as the SHA-256 compression function natively. However, we will still make use of such pure Simplicity implementations as formal specifications for our jets.

# Option Types

Option types are made by taking a sum with the unit type:

`Option A ≔ 𝟙 + A`

The type `Option A` may also be written as `A?` or `𝕊 A` (where `𝕊` stands for “successor”). As with vectors, functions can be “mapped” over the option type:

```none
                 f : A ⊢ B
------------------------------------------
f? ≔ copair (injl unit) (injr f) : A? ⊢ B?
```

We can also define monadic combinators such as bind:

```none
              f : A ⊢ B?
---------------------------------------
bind f ≔ copair (injl unit) f : A? ⊢ B?
```

# Variable Length Buffers

We can also define what we call “buffers,” which are types for partially filled vectors.

* `Aᑉ² ≔ A?`
* `Aᑉ⁴ ≔ A²? × Aᑉ²`
* `Aᑉ⁸ ≔ A⁴? × Aᑉ⁴`
* `…`


🛈
The type `Xᑉ⁸` expands to `(1 + X⁴) × ((1 + X²) × (1 + X))`. If we pretend this is a polynomial and expand it, we get `1 + X + X² + X³ + X⁴ + X⁵ + X⁶ + X⁷`. Interpreting this polynomial as a type, it is the sum of all possible tuples of X up to 7, including the empty tuple. In other words, it is exactly the type of lists whose length is strictly less than 8.

As with vectors, we can define various mapping and folding operations over buffers. We can also define stack operations such as `push-<n : Aᑉⁿ × A ⊢ Aⁿ + Aᑉⁿ` and `pop-<n : Aᑉⁿ ⊢ (Aᑉⁿ × A)?`. `push-<n` takes a buffer and a new item and appends it to the buffer, returning a full vector in case the buffer overflows. `pop-<n` takes a buffer and removes an item, returning the smaller buffer and the item removed, optionally returning nothing in case the original buffer was empty.

For example, we can define `push-<n` recursively:

* ```
  push-<2 ≔ case (drop (injr (injr iden))) (injl iden)
* ```
  push-<4 ≔ ((O I H ▵ IH) ⨾ push-<2) ▵ O O H
          ⨾ case (I H ▵ O H ⨾ case (injr (injr (I H) ▵ injl unit)) (injl iden))
                 (injr (I H ▵ O H))
* ```
  push-<8 ≔ ((O I H ▵ IH) ⨾ push-<4) ▵ O O H
          ⨾ case (I H ▵ O H ⨾ case (injr (injr (I H) ▵ (injl unit ▵ injl unit))) (injl iden))
                 (injr (I H ▵ O H))
* `…`

As you can see, raw Simplicity becomes inscrutable once one reaches a certain level of complexity. Again, we expect end users to utilize higher-level languages, such as SimplicityHL, that can generate these idiomatic expressions.

# Conclusion

In this part, we illustrated how to build logical operations starting from bits. From these logic operations, we were able to build bit-level arithmetic and illustrate how to reason about their execution. After developing vector types, we showed how to iterate over multi-bit words to define arithmetic. Continuing in this way, we can define cryptographic operations, such as SHA-256 and Schnorr signature validation, using just our computational Simplicity combinators. All of which we have actually defined using Simplicity.

This part is not meant as a comprehensive guide to the various possible data types and operations that we can build in Simplicity, but instead, it serves to illustrate how it is possible to build practical functionality within Simplicity’s constraints. Even though Simplicity’s types are all finitely bounded, we can still define useful vectors and buffer types and operations that iterate over values within these structures.

The actual specifications of operations in our standard library are a little different from the definitions used here. For instance, the full-adder in our standard library uses a 3-way *xor* and a “majority” logic function in its implementation, rather than using two half-adders as shown here.

Later, we will see that, in practice, Simplicity programs use jets for arithmetic and cryptographic operations. However, jets are limited to replacing expressions. The combinators for iterating over buffers and vectors we have defined cannot be replaced with jets. These defined combinators will appear in actual Simplicity programs. Although, rather than directly using these combinators, we expect end users will use higher-level languages, such as SimplicityHL, to generate such Simplicity expressions.

A keen-eyed reader may have noticed that our recursively defined combinators appear to grow exponentially in expression size. However, this is not a problem. Later, when we talk about serialization, we will see that expressions are encoded as a DAG (directed acyclic graph) rather than as a tree. So the actual presentation of these expressions will only grow linearly in size.

So far, we have only been considering pure computations. However, we need some way to interact with transaction data for tasks like signing transactions, and we need to be able to have the program fail somehow if the signature is not valid. In the next part, we will discuss side-effects in Simplicity.

-------------------------

