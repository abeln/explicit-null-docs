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
