# Delving Simplicity Part Ⅱ: Combinator Completeness of Simplicity

roconnor-blockstream | 2025-08-18 14:26:20 UTC | #1

In [Part Ⅰ](https://delvingbitcoin.org/t/delving-simplicity-part-three-fundamental-ways-of-combining-computations/1902) of this series, we described the three methods of composition in programming: sequential, parallel, and conditional. In this part, we will see how these methods of composition are directly realized within Simplicity.

# Simplicity Types

Parallel compositions produce an output that is a product of two types. We write this product type as `A × B` or `A * B`. Conditional compositions consume an input that is a sum (also known as a tagged union) of two types `A + B`. In order to directly realize these forms of composition, Simplicity’s type system includes these two type formers. Simplicity has only one other type: the unit type.

## Unit Type

Simplicity’s unit type, written `𝟙` or `ONE`, is a type with only one value. This single value is typically denoted by the empty tuple, written `⟨⟩` or `()`. Since this type consists of only a single value, values of this type contain no information; it is essentially a zero-bit data type.

We use the `:` to denote a value’s type, so `⟨⟩ : 𝟙` means the empty tuple has unit type.

## Sum Type

The sum of two types, `A + B`, is the tagged union of two types. The tag is either “left” or “right,” indicating whether a given value corresponds to the left-hand type of the sum or the right-hand type. If `a : A`, then we write `σᴸ(a)` or `inl(a)` for the left-tagged value of type `A + B` (i.e., `σᴸ(a) : A + B`). Similarly, if `b : B`, then we write `σᴿ(b)` or `inr(b)` for the right-tagged value (i.e., `σᴿ(b) : A + B`).

Values of a sum type always contains a tag, even when taking the sum of two identical types. In particular, if `a : A`, then `σᴸ(a)` and `σᴿ(a)` are two distinct values of type `A + A` because they have different tags.

### Boolean Type

We will discuss building data structures in more detail in the next part of this series, but as a preview, let’s look at our first non-trivial type: `𝟙 + 𝟙`. We typically denote this type using the shorthand `𝟚` or `TWO`. Keep in mind that this is purely notational convenience. This type is the tagged union of two unit types and therefore has two values: `σᴸ⟨⟩ : 𝟚` and `σᴿ⟨⟩ : 𝟚`. This is a one-bit data type. By convention, we consider `σᴸ⟨⟩` to be a zero or false value, which we may write as `0` or `false`. Similarly, we consider `σᴿ⟨⟩` to be a one or true value, which we may write as `1` or `true`. Again, this is just notational convenience.

## Product Type

The product of two types, `A × B`, contains pairs of values. If `a : A` and `b : B` are values of types `A` and `B`, then the pair, written as `⟨a, b⟩` or `(a, b)`, has type `A × B` (i.e., `⟨a, b⟩ : A × B`).


🛈
Simplicity only has finite types. You can read off the number of values a type has by interpreting the type using arithmetic. For example, the type `𝟚 × 𝟚` has four values. The type `𝟚 + 𝟚` also has four values; however, they are different values from those of type `𝟚 × 𝟚`.

# Core Simplicity Expressions

In Part Ⅰ, we presumed we had some set of basic operations, and each operation had an input type and an output type. We will write `f : A ⊢ B` or `f : A |- B` to mean that `f` has input type `A` and output type `B`. 


⚠
While Simplicity expressions have input and output types, Simplicity’s types do not include function types. Simplicity is a “first-order” programming language.

## Two Basic Operations

Simplicity provides several basic operations, most of which we will see later in this series when we talk about Simplicity jets. The core language provides expressions for only two basic operations: `iden : A ⊢ A` and `unit : A ⊢ 𝟙`. Technically, these are two families of operations, with one of each operation per Simplicity type `A`.

The `iden` operation passes its input along to its output. The `unit` operation discards its input and returns the empty tuple, `⟨⟩`.

To be more precise, we need to distinguish between programming language syntax and semantics. `iden` and `unit` are Simplicity *expressions*, which constitute the language’s syntax. Simplicity expressions *denote* operations, which are functions from an input type to an output type. We use fancy square brackets to indicate what a Simplicity expression denotes:

* `⟦iden⟧(a) = a` (alternatively written`|[iden]|(a) = a`)


* `⟦unit⟧(a) = ⟨⟩`

## Three Composition Combinators

The sequential and parallel composition methods are directly implemented by Simplicity *combinators*. Recall that Simplicity expressions denote functions. A combinator is a function that takes functions and returns a function. We follow the usual programming language practice of defining these Simplicity expressions using typing rules:

* ```none
  f : A ⊢ B    g : B ⊢ C
  ----------------------
     comp f g : A ⊢ C
  ```
* ```none
  f : A ⊢ B    g : A ⊢ C
  ----------------------
   pair f g : A ⊢ B × C
  ```

The first rule states that if `f` is a Simplicity expression with input type `A` and output type `B`, and `g` is an expression with input type `B` and output type `C`, then `comp f g` is a Simplicity expression with input type `A` and output type `C`. The second rule defines the `pair f g` expression similarly.

We may also write `comp f g` as `f ⨾ g` or `f >>> g` and we may write `pair f g` as `f ▵ g` or `f &&& g`.

These combinators form expressions of exactly the right type to implement sequential and parallel composition. Accordingly, these expressions have semantics for these forms of composition.

* `⟦f ⨾ g⟧(a) = ⟦g⟧(⟦f⟧(a))`
* `⟦f ▵ g⟧(a) = ⟨⟦f⟧(a), ⟦g⟧(a)⟩`


⚠
The version of sequential composition used in mathematics is backwards with respect to our syntax.

The other method of composition is conditional composition. The natural approach would be, given `f : A ⊢ C` and `g : B ⊢ C`, to define `copair f g : A + B ⊢ C`. However, `copair` doesn’t allow branches to access shared data. In particular, the distribution function,

 `dist : (A + B) × C ⊢ A × C + B × C`,

cannot be defined in terms of `copair` alone.

Instead of `copair`, we use the following `case` combinator instead.

* ```none
  f : A × C ⊢ D    g : B × C ⊢ D
  ------------------------------
    case f g : (A + B) × C ⊢ D
  ```

This `case` combinator does conditional composition and distribution all in one. It’s helpful to think of the type `C` as the type of a shared environment that both branches of the `case` expression get access to. Accordingly, the `case` combinator has the following semantics.

* `⟦case f g⟧⟨σᴸ(a), c⟩ = ⟦f⟧⟨a, c⟩`
* `⟦case f g⟧⟨σᴿ(b), c⟩ = ⟦g⟧⟨b, c⟩`

## Four more Combinators

We are still missing a few fundamental operations on our types. For instance, we can use `pair` to produce product types, but we have no way yet to consume product types. Similarly, the `case` combinator consumes sum types, but we have no way yet to produce sum types. Four additional combinators enable the consumption of product types and production of sum types. The first two combinators extract one of the values of a pair, and execute a function on it:

* ```none
       f : A ⊢ C
  ------------------
  take f : A × B ⊢ C
  ```


* ```none
       f : B ⊢ C
  ------------------
  drop f : A × B ⊢ C
  ```

The last two combinators execute a function and wrap the result with either a left tag or a right tag.

* ```none
       f : A ⊢ B
  ------------------
  injl f : A ⊢ B + C
  ```


* ```none
       f : A ⊢ C
  ------------------
  injr f : A ⊢ B + C
  ```

These combinators have the required semantics:

* `⟦take f⟧⟨a, b⟩ = ⟦f⟧(a)`
* `⟦drop f⟧⟨a, b⟩ = ⟦f⟧(b)`
* `⟦injl f⟧(a) = σᴸ(⟦f⟧(a))`
* `⟦injr f⟧(a) = σᴿ(⟦f⟧(a))`

# Simplicity and the Sequent Calculus

Readers familiar with formal logic will find a resemblance between our set of nine rules and the conjunctive-disjunctive fragment of Gentzen’s sequent calculus. The sequent calculus inspired Simplicity’s language design. One can think of Simplicity as a slightly tweaked variant of the “functional interpretation” of Gentzen’s sequent calculus, which is analogous to the Curry-Howard correspondence between natural deduction and the lambda calculus.

Notice that the premises in the combinator rules of the core Simplicity language are typically “smaller” than the types in the conclusion. Later in this series we will describe “the Bit Machine”, an abstract stack machine for interpreting Simplicity expressions. The Bit Machine takes advantage of this phenomenon of “smaller types in the premises” to minimize the amount of data copied during execution. Only the `iden` and `comp` core combinators involve moving data around, and the rest of the core combinators are implemented with just some bookkeeping.

# Values are not Expressions

Above we gave notation for the values of the various types that Simplicity supports. It is important to note that Simplicity expressions only denote operations, and operations are functions, not values. We can construct values by starting with the `unit` function and composing it with `injl`, `injr` and `pair` combinators. To avoid writing out such tedious constructions, we introduce the following notation.

Given a value of some Simplicity type, `b : B`, we write `scribe b : A ⊢ B` for the unique Simplicity expression that always returns the value `b`. For example, `scribe ⟨σᴸ⟨⟩, σᴿ⟨⟩⟩` is shorthand for `injl unit ▵ injr unit : A ⊢ 𝟚 × 𝟚`. Keep in mind that, `scribe` isn’t a Simplicity combinator; it is a macro-like notional convenience.

Bitcoin Script has a similar property; Bitcoin Script only contains operations. For example `OP_1` is the operation that pushes the value `1` onto the stack, but there is no Bitcoin Script expression for the value `1` itself.

# Simplicity’s Completeness Theorem

Earlier we saw that the naïve realization of conditional composition would have left us unable to define the “distribution” function `dist : (A + B) × C ⊢ A × C + B × C`. How do we know that we aren’t missing something else?

The answer is that the “Simplicity Completeness theorem” proves that for any function between two Simplicity types there exists some Simplicity expression that denotes it.

We already saw `scribe`, which is a special case of the Simplicity Completeness theorem for constant functions. For other functions, we can build an nested set of `case` expressions to fully decompose any input of any type and compose that with a `scribe` expression for every output value. This procedure constructs what is effectively a giant lookup table to implement any function.

But you don’t have to trust us. The core Simplicity language is formally specified in the Rocq proof assistant (formerly named the Coq proof assistant), and the [Simplicity Completeness theorem](https://github.com/BlockstreamResearch/simplicity/blob/8adafc9b55d694111c99fa11f90c7e48376ed1e5/Coq/Simplicity/Core.v#L92-L93) is one of the theorems we have formally verified.

# Conclusion

We described Simplicity’s type system, combinators, and basic expressions that make up Simplicity’s core computational language. Later in this series, we will describe how Simplicity interacts with transactions and introduce a few more Simplicity combinators.

But before delving into that, in Part Ⅲ, we will look at building data structures and computation from this seemingly meager language. While we do have the Simplicity Completeness theorem, the expressions that the theorem constructs are astronomical in size, making it only useful as a theoretical tool. The practice of generating succinct Simplicity expressions could be viewed as an exercise in compression by exploiting structure within computations.

-------------------------

