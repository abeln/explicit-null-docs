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