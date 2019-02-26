# Notes
This document contains rough notes of what we learn while we migrate
different libraries to the explicit null type system for Scala.

## Inlined Methods and Flow Inference
While migrating `tests/pos/rbtree.scala` (a Red-Black tree implementation),
we sometimes want to do a null check that's more "complicated"; e.g. because
it's implied by a type test.

See https://github.com/abeln/dotty/blob/explicit-null/tests/pos/rbtree.scala#L127

```
private[this] def isRedTree(tree: Tree[_, _]) = tree.isInstanceOf[RedTree[_, _]]
...
private[this] def balanceLeft[A, B, B1 >: B](isBlack: Boolean, z: A, zv: B, l: Tree[A, B1], d: Tree[A, B1]): Tree[A, B1] = {
  if (isRedTree(l) && isRedTree(l.left))
```

In the above, `isRedTree(l)` means that it's safe to do `l.left`.
However, since our flow inference is only _intraprocedural_, we can't infer
that `l` is non-null across method calls.

The solution was to mark `isRedTree` as `inline`, so that the (modified) test
is inlined:

```
private[this] inline def isRedTree(tree: Tree[_, _]) = (tree ne null) && tree.isInstanceOf[RedTree[_, _]]
```

## Arrays

The problem with (reference) arrays is that, depending on how they're constructed, they can start
out with its elements being all `null`.
```
val x = new Array[String](3)
x(0).length // NPE
```

This is a source of unsoundness for our explicit nulls type system. However, we can't simply make
_all_ arrays nullable by marking the return type of `apply` and the argument to `update` as such:
```
final class Array[T](_length: Int) extends java.io.Serializable with java.lang.Cloneable {

  def apply(i: Int): T|Null = throw new Error()

  def update(i: Int, x: T|Null) { throw new Error() }
}
```

The problem is that `T` can be instantiated with `Bool`, which means we'd lose the ability to create primitive
arrays:
```
val x = new Array[String](10)
x(1): String|Null // ok

val y = new Array[Bool](10)
y(1): Bool|Null // no longer a primitive array
```

This is too inefficient. The reason the array type parameter can remain unconstrained in Scala
is that the compiler requires an instance of a `ClassTag[T]` on every instantiation:
```
class Foo[T] {
  val x = new Array[T](10)
}

4 |  val x = new Array[T](10)
  |                          ^
  |                          No ClassTag available for T
```

The `ClassTag` allows the compiler backend to instantiate the right kind of Java array at runtime,
but we (the Typer) can't use it to decide whether to type the `Array[T]` as `Array[T]` (primitive type)
or `Array[T|Null]` (reference type).

### Proposed Solution

Keep the unsoundness: it's up to the user to decide whether their arrays are nullable or not.

However, we will add additional ways to create arrays _soundly_.

In total, there are currently >= 5 ways to create arrays
```
new Array[String](10) 		// unsound
Array.ofDim[String](10)		// unsound
Array.apply("hello", "world") 	// sound
Array.fill(10)(_ => "default")	// sound
```

We propose adding two additional ways to create arrays soundly (names are tentative):
```
object Array {
  def ofNulls[T >: Null <: AnyRef|Null](dim: Int): Array[T|Null]
  def ofZeros[T <: AnyVal](dim: Int): Array[T]  
}

Array.ofNulls[String](10): Array[String|Null]	// sound
Array.ofZeros[Boolean](10): Array[Boolean]	// sound
```

The above is sound because in the `AnyVal` case we know that `T` will erase to a value
type.

This is similar to `arrayOfNulls` in Kotlin: https://kotlinlang.org/docs/reference/basic-types.html#arrays

## Java Imports

A list of Java imports in the Dotty community build, sorted by frequency: https://gist.github.com/abeln/e7db55878014f27c90d1dfd2f5dc984e

## Ignore Nulls in Overrides

After we added a flag to gate the explicit nulls feature, we noticed that some updated tests no longer pass when the feature
is off. There are a couple of kinds of tests changes that weren't backwards-compatible, but one large class is due to overriding.

The problem is that currently `Null` is meaningful when doing override checks:
```scala
class Foo {
  String foo() {}
}

class Bar extends Foo {
  override def foo(): String|Null // error: can't override String with String|Null
}
```

This will be a problem because if library `A` depends on `B` through overrides, then 
`A` can't be updated to explicit nulls until `B` is.

### Proposed Solution

Ignore `| Null` when doing override checks. Instead, if the override is unsound (e.g. 
`String|Null` overrides `String` in a covariant position, or the other way around for contravariant positions),
issue a warning.

## Uninitialized Fields and Local Variables (the Underscore)

For now we decided to allow `_` both in fields and local variables, to indicate that they're uninitialized.

e.g.
```
class Foo {
  var x: String = _ 		// allowed but unsound
  def bar(): Unit = {
    var loc: String = _ 	// allowed but unsound
  }
}
```

TODO: bring up examples that motivate this from https://github.com/abeln/collection-strawman/pull/1

## Non local control flow and blocks

We sometimes see code that looks like

```scala
def foo(x: String|Null): Boolean = {
  if (x == null) return false
  // do stuff with x knowing that it's non-null
  val len = x.length  
}
```

For an example, see https://github.com/abeln/dotty/blob/explicit-null/tests/pos/rbtree.scala#L250

The general pattern is handling the null case early on in the function, followed by a non-local jump
of control: e.g. a `return` or throwing an exception.

### Proposed Solution (UNSOUND)

*NOTE*: the approach below is unsound, as shown by

```scala
class Foo {
  def foo(): Unit = {
    val x: String|Null = ???
    g.length
    if (x == null) return ()
    def g: String = x // executes _before_ the test above is evaluated
    ()
  }
}
```

The problem is that if s1 and s2 are consecutive statements in a block, it is _not_
always the case that s1 must evaluate _before_ s2. For example, if s2 is a `def`, or a `lazy val`
then a statement s0 that appears earlier in the block can cause s2 to execute without s1 executing
(as in the example above).

---
Add support for the idiom above in the flow-sensitive type inference.

More formally, let `s1, s2, ..., sn` be a block of consecutive statements. Now suppose statement
`si` is of the form:
  - `If(cond, then, else)`

Then we can compute `(ifTrue, ifFalse)`, the set of non-nullability facts that hold if `cond` is
true/false, respectively.

Now define a notion of `non-local` expression:
  - a return is a non-local expression
  - an expression with type `Nothing` is non-local (this includes applications of the `throw` builtin) 
  - a block is non-local if its last expression is
  - no other expressions are non-local

Consider what happens when `nonLocal(then)` holds. Then we know that for `s_{i + 1}` to execute, the
condition must have been false, for if the condition were true, then the `then` expression would execute.
But `then` is non-local, so `s_{i + 1}` would not execute (a contradiction).

This means that if `nonLocal(then)` holds, then we can propagate the `ifFalse` facts to the context
under which `s_{i + 1}` is typed.

The algorithm for blocks is thus:
  - if `then` and `else` are both non-local, propagate `ifTrue ++ ifFalse`
  - if only `then` is non-local, propagate `ifFalse`
  - if only `else` is non-local, propagate `ifTrue`
  - if neither branch is non-local, propagate the empty set of facts (no information)

See https://github.com/abeln/dotty/commit/f0bc0e5b6d5276dbc4ed7489cac5124bcbfcd5f1 for supported cases.

## A sound solution for blocks

We can modify the inference for blocks described in the section above to get back soundness.

The idea is to not propagate flow facts to `defs` and `lazy vals`, but propagate them to `vals`.

So this works
```scala
val x: String|Null = ???
if (x == null) throw new NPE()
val y: String = x // x: String inferred
```
but this doesn't
```scala
val x: String|Null = ???
if (x == null) throw new NPE()
def y: String = x // x: String|Null
```

Notice, though, that we only restrict facts coming from the _inner_ block.
Facts from outer blocks are still propagated.

e.g.
```scala
val x: String|Null = ???
if (x != null) {
  def y: String = x // ok; x: String inferred
}
```

## A new type of completer

Related to the topic of flow inference within blocks, we have the following problem,
 caused by definitions generating completers, and completers using the creation context
for completion.

```scala
val x: String|Null = ???
if (x == null) throw NPE
val y = x // y: String|Null inferred
```

The reason is that if `y` is completed with its creation context, then it won't know
that `x` is non-null (because the creation context is built _before_ we type the if statement.

The solution is to create a new kind of Completer that uses the current context.
This new Completer is used for local (inside a method) definitions that are `vals`,
but not `defs` or `lazy vals`.

It's safe to use this new kind of Completer because local definitions will be forced as soon as the corresponding block is typed.

## Implicit Nulls (backwards compat mode)

Sometimes migrating a file to use explicit nulls is not straightforward. See `TrieMap` in collections-strawman for an example
https://github.com/abeln/collection-strawman/blob/a970e293551b2fdd4111e56f1b272d8276cce769/collections/src/main/scala/strawman/collection/concurrent/TrieMap.scala

TrieMap is a 1000 line file that interacts with Java, uses null for efficiency, and has concurrent logic.
Determining what should or shouldn't be null within it requires domain-specific knowledge (e.g. what fields did the author intend to make non-null?)
that we lack.

In general, when porting their project to explicit nulls, a user might want to migrate only part of the project initially,
and then gradually migrate others. Or a user might want to avoid migrating a specific file because it's tricky, but not
want to block the _entire_ migration on that one file.

### Proposed Solution

We created a "backward compatibily" mode for our feature. This mode is invoked via a language import,
and can be used to reduce (significantly, but not entirely) the burden of migrating to explicit nulls.

Of course, the backwards compat mode is unsound in that it allows certain non-nullable types to contain null.

So really this is something that should be used during a migration, but eventually removed.
And it's not intended to be used with new code.

Example
```
import scala.language.implicitNulls
val x: String|Null = ???
val y: String = x // turns into `val y: String = x.asInstanceOf[String]`
val y2: String = null // turns into `val y2: String = null.asInstanceOf[String]`
```

As the example shows, the import enables the following two implicit conversions (these are special conversions
implemented in the compiler, not real implicit conversions):

1) if `tree` has type `Null` and the prototype has a reference type S, then `tree` becomes `tree.asInstanceOf[S]`
2) if `tree` has type `S|Null` and the prototype has type `S`, then `tree` becomes `tree.asInstanceOf[S]`

_Note_: this will not fix every error triggered by explicit nulls. For example, if we have
```
val x: String|Null = ???
x.length // error: selection on nullable union
```
and
```
val x: Array[String] = (??? : Array[String|Null]) // error: Array is invariant
```

See https://github.com/abeln/collection-strawman/commit/a970e293551b2fdd4111e56f1b272d8276cce769
for an example of how we migrated `TrieMap` with minimal changes after the import.

## Array.copyOf

We added a new copyOf method that can handle non-nullable Arrays.
Otherwise, the user needs to use java.util.arrays.copyOf, which takes and returns an `Array[T|Null]`.

This is useful in e.g. HashMap and HashSet from collection-strawman.
