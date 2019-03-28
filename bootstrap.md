We managed to bootstrap dotty with new explicit nulls type system.

This document will contain some notes on the experience.

## Easy-to-count stats
  * num of files:
    ```
    src git:(explicit-null-bootstrap) ✗ pwd
    /Users/abeln/src/dotty2/dotty/compiler/src
     src git:(explicit-null-bootstrap) ✗ ls -lR | grep scala | wc -l
     403
    ```
  * num LOC in dotty:
    ```
    src git:(explicit-null-bootstrap) ✗ cat `ls -d -1 "$PWD/"**/* | grep ".scala"` | wc -l
    117010
    ```
  * num files changed (via `git diff --stat HEAD HEAD^`): *156 (39%)*
  * num LOC modified: *3188 (2.7%)* (1391 insertions(+), 1797 deletions(-))
  * See per-file stats below
  * usages of `.nn`:
    ```
    src git:(explicit-null-bootstrap-big) ✗ git grep "\.nn" | wc -l
    780
    ```
    
## Per-file change stats (sorted)
`git diff --stat HEAD <commit-before-migration>^|awk '{ print $3 " "$4 " " $1}'| sort -n -r|less | cat`

See stats here: https://gist.github.com/abeln/d0d2979efbf469501923c7d73341e145    

Here's a per-file description of what changed in the top 20 files:

### 1. tools/backend/jvm/{BackendInterface, DottyBackendInterface.scala} (370/1203)

#### 1.1

`BackendInterface` is (surprise) a trait that defined an abstract interface to the backend.
Specifically, it defined a bunch of abstract type members for each kinds of AST node. In the original code, most of these
members have `Null` as a lower bound. 

Since `Null` is no longer a subtype of `AnyRef`, this means the upper bound must change to `Nullable[AnyRef]` (this kind of change where we make the upper bound of an abstract type member nullable re-appears multiple times during the bootstrapping).

```scala
-  type Constant   >: Null <: AnyRef
-  type Symbol     >: Null <: AnyRef
-  type Type       >: Null <: AnyRef
+  type Constant   >: Null <: Nullable[AnyRef]
+  type Symbol     >: Null <: Nullable[AnyRef]
+  type Type       >: Null <: Nullable[AnyRef]
```

The corresponding concrete type in `DottyBackendInterface` needs to change as well, so that the lower bound is satisfied:
```scala
-  type Symbol          = Symbols.Symbol
-  type Type            = Types.Type
-  type Tree            = tpd.Tree
+  type Symbol          = Nullable[Symbols.Symbol]
+  type Type            = Nullable[Types.Type]
+  type Tree            = Nullable[tpd.Tree]
```

Further, these changes propagate through the backend, since every time we have a e.g. `Symbol`, we actually have a `Nullable[Symbols.Symbol]`, so we have to use `.nn`.

#### 1.2

Additionally, this file interacts with the `ASM` java library, which has no annotations (plus, we can't recognize Java annotations yet), so all java methods return nullable values, which then need `.nn`:
```scala
-      val pannVisitor: asm.AnnotationVisitor = jmethod.visitParameterAnnotation(idx, innerClasesStore.typeD
escriptor(typ.asInstanceOf[bcodeStore.int.Type]), isRuntimeVisible(annot))
+      val pannVisitor: asm.AnnotationVisitor = jmethod.visitParameterAnnotation(idx, innerClasesStore.typeD
escriptor(typ.asInstanceOf[bcodeStore.int.Type]), isRuntimeVisible(annot)).nn
       emitAssocs(pannVisitor, assocs, bcodeStore)(innerClasesStore)
```
   
### 2. dotc/core/Types.scala (208/5330)

#### 2.1

The `equals`, `iso`, and `computeHash` methods in the base `Type` class changed:
```scala
-    final def equals(that: Any, bs: BinderPairs): Boolean =
+    final def equals(that: Any, bs: Nullable[BinderPairs]): Boolean =

-    protected def iso(that: Any, bs: BinderPairs): Boolean = this.equals(that)
+    protected def iso(that: Any, bs: Nullable[BinderPairs]): Boolean = this.equals(that)

-    final def computeHash(bs: Binders): Int = NotCached
+    override final def computeHash(bs: Nullable[Binders]): Int = NotCached
```

These are methods that are overriden by many subclasses of types, so the method signature needs to be updated in those as well?

Why did the changes happen? Well, `NamedType` overrides the standard `equals` method (not the one above) as
```scala
override def equals(that: Any): Boolean = equals(that, null)
```

The `equals` on the r.h.s is the method in the base `Type` class. So we know the second argument to `equals` must be nullable. But then `equals` calls `iso` (again, in the base type):
```scala
final def equals(that: Any, bs: Nullable[BinderPairs]): Boolean =
  (this `eq` that.asInstanceOf[AnyRef]) || this.iso(that, bs)
```

So `iso` must also take a nullable second argument.

`computeHash` is called with null as argument
```scala
  abstract class CachedGroundType extends Type with CachedType {
    private[this] var myHash = HashUnknown
    final def hash: Int = {
      if (myHash == HashUnknown) {
        myHash = computeHash(null)
        assert(myHash != HashUnknown)
```

Similarly, `equalBinder` in BindingType takes a nullable second argument:
```scala
-    def equalBinder(that: BindingType, bs: BinderPairs): Boolean =
+    def equalBinder(that: BindingType, bs: Nullable[BinderPairs]): Boolean =
       (this `eq` that) || bs != null && bs.matches(this, that)
```

And `identityHash`, which coems from `Hashable`:
```scala
+    override def identityHash(bs: Nullable[Binders]): Int = {
+      def recur(n: Int, tp: BindingType, rest: Nullable[Binders]): Int =
         if (this `eq` tp) finishHash(hashing.mix(hashSeed, n), 1)
         else if (rest == null) System.identityHashCode(this)
         else recur(n + 1, rest.tp, rest.next)
```

#### 2.2


   
## JavaNull

I instrumented the compiler to log every time a member selection happens on a union with `JavaNull`: e.g. `x.foo` where `x: String|JavaNull`.

There were 896 such instances. See the types that showed up here: https://gist.github.com/abeln/3b709744ccfe13a0065d4f36db35b927 
   
## Using the checker framework info

From the checker framework we got nullability annotations exclusively about method and field return types.
We have annotations for ~ 6232 methods, 1712 fields in 850 classes.

See the annotations file here: https://github.com/abeln/explicit-null-docs/blob/master/explicit-nulls-stdlib.xml

The annotations don't seem to be very useful, however. Disabling the annotations produces just _4_ additional type errors (but see below for why this is probably misleading):
e.g.
```scala
[error] -- [E007] Type Mismatch Error: /Users/abeln/src/dotty2/dotty/library/src-bootstrapped/scala/reflect/GenericClass.scala:44:46
[error] 44 |    def addElem = elems += labelsStr.substring(start, cur)
[error]    |                           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
[error]    |                           Found:    String | JavaNull
[error]    |                           Required: String
```
    
However, there are 114 methods used while bootstrapping that we were able to translate more precisely because of the annotations. We speculate that these result in a small number of errors because of `JavaNull`.

It's also probably the case that I only saw 4 additional errors because the compiler sees the errors and gives up. Often fixing type errors would reveal additional ones.

https://gist.github.com/abeln/36a245f67a519b7318f749811d84e475

## Flow inference

There were 375 `TermRef`s whose type was refined by flow-sensitive type inference (of any kind: within ifs, conditionals, or blocks).

See the list here: https://gist.github.com/abeln/3cdde264692b2ee69ba29da8ee2133bd
   
## Statistics we want to gather:
  * total number of files
  * num files changed
  * num LOC in dotty
  * num LOC modified
  * num of `.nn` used
  * num of usages of `JavaNull`
  * num of usages of flow inference broken down by type
    - body of `if` expressions
    - within conditions
    - within blocks
  * num of times we used checker framework info
  * num of nullable fields/arguments/return types
