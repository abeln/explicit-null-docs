# Takeaways from the bootstrapping

We managed to bootstrap dotty with new explicit nulls type system.

Takeaways:
  1. Dotty uses multiple java libraries, which are currently unannotated. We need to support nullability annotations in Java code (currently unsupported), and annotate the java dependencies of dotty.
  2. Flow inference should support pattern matching: e.g.
  ```scala
  val x: String|Null = ???
  x match {
    case null => "dummy"
    case _ => x // should be x: String
  }
  ```
  and/or
  ```scala
  val x: String|Null = ???
  x match {
    case null => "dummy"
    case x1 => x1 // should be x1: String
  ```
  3. It'd be nice to redefine certain classes in the compiler (e.g. `BackendInterface`, `WeahHashSet`) to take type parameters that don't have a `Null` lower bound (maybe not possible).
  4. The case where a class has state (a `var` field) which is non-null most of the time, but where correctness is ensured through some non-trivial invariant is pretty common (I don't know how to get around this, since flow inference doesn't support `var`s).
  5. Could we support flow inference for _local_ `vars`? This would also be useful, and is easier (example from `WeakHashSet`):
  ```scala
    def fullyValidate(): Unit = {
      var computedCount = 0
      var bucket = 0
      while (bucket < table.size) {
        var entry = table(bucket)
        while (entry != null) {
          assert(entry.nn.get != null, s"$entry had a null value indicated that gc activity was happening during diagnostic validation or that a null value was inserted")
          computedCount += 1
          val cachedHash = entry.nn.hash
          val realHash = entry.nn.get.nn.hashCode
          assert(cachedHash == realHash, s"for $entry cached hash was $cachedHash but should have been $realHash")
          val computedBucket = bucketFor(realHash)
          assert(computedBucket == bucket, s"for $entry the computed bucket was $computedBucket but should have been $bucket")

          entry = entry.nn.tail
        }

        bucket += 1
      }

      assert(computedCount == count, s"The computed count was $computedCount but should have been $count")
    }
  ```

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

A bunch of the classes in the file declare state that starts out as `null` (so needs to be declared as nullable).
e.g. in `MatchType`
```scala
-    private[this] var myReduced: Type = null
-    private[this] var reductionContext: mutable.Map[Type, Type] = null
+    private[this] var myReduced: Nullable[Type] = null
+    private[this] var reductionContext: Nullable[mutable.Map[Type, Type]] = null
```

This state is not immediately initialized, and is sometimes used depending on complicated logic ():
```scala
      record("MatchType.reduce called")
      if (!Config.cacheMatchReduced || myReduced == null || !upToDate) {
        record("MatchType.reduce computed")
        if (myReduced != null) record("MatchType.reduce cache miss")
        myReduced =
          trace(i"reduce match type $this $hashCode", typr, show = true) {
            try
              if (defn.isBottomType(scrutinee)) defn.NothingType
              else if (reduceInParallel) reduceParallel(trackingCtx)
              else reduceSequential(cases)(trackingCtx)
            catch {
              case ex: Throwable =>
                handleRecursive("reduce type ", i"$scrutinee match ...", ex)
            }
```

### 3. tools/backend/jvm/BCodeSkelBuilder.scala (148/726)

This class exhibits another common pattern: class state that starts out as null, and where the class logic relies on "non-obvious" invariants to make sure things are initialized when needed.

e.g. declared at the top level in the class:
```scala
-    var cnode: asm.tree.ClassNode  = null
-    var thisName: String           = null // the internal name of the class being emitted
+    var cnode: Nullable[asm.tree.ClassNode]  = null
+    var thisName: Nullable[String]           = null // the internal name of the class being emitted
```

Later, `cnode.nn` is used 13 times throughout the file and `thisName.nn` has 3 matches.

There are other fields declared in this way:
```scala
-    var mnode: asm.tree.MethodNode = null
-    var jMethodName: String        = null
+    var mnode: Nullable[asm.tree.MethodNode] = null
+    var jMethodName: Nullable[String]        = null
```

There are 14 usages of `mnode.nn`.
Additionally, because these fields are `var`s, flow inference can't infer a more-precise for them even when checks happen.

Often, the fields are used without any checks:
```scala
    // helpers around program-points.
    def lastInsn: Nullable[asm.tree.AbstractInsnNode] = mnode.nn.instructions.getLast
```

### 4. dotc/core/Scopes.scala (138/480)

#### 4.1

There are multiple subclasses of `Scope`. `EmptyScope` forces some of the methods in Scope to have nullable return types:
```scala
   object EmptyScope extends Scope {
-    override private[dotc] def lastEntry: ScopeEntry = null
+    override private[dotc] def lastEntry: Nullable[ScopeEntry] = null
     override def size: Int = 0
     override def nestingLevel: Int = 0
     override def toList(implicit ctx: Context): List[Symbol] = Nil
     override def cloneScope(implicit ctx: Context): MutableScope = unsupported("cloneScope")
-    override def lookupEntry(name: Name)(implicit ctx: Context): ScopeEntry = null
-    override def lookupNextEntry(entry: ScopeEntry)(implicit ctx: Context): ScopeEntry = null
+    override def lookupEntry(name: Name)(implicit ctx: Context): Nullable[ScopeEntry] = null
+    override def lookupNextEntry(entry: ScopeEntry)(implicit ctx: Context): Nullable[ScopeEntry] = null
   }
```

The other subclasses _also_ use null; it's not just `EmptyScope`.

#### 4.2 Limitation of flow-inference

In the `filter` method of `Scope`:
```scala
       var syms: PreDenotation = NoDenotation
       var e = lookupEntry(name)
       while (e != null) {
-        val d = e.sym.denot
+        val d = e.nn.sym.denot
         if (select(d)) syms = syms union d
-        e = lookupNextEntry(e)
+        e = lookupNextEntry(e.nn)
       }
       syms
     }
```

Because the flow inference can't handle `vars`, then we need the `.nn`.

These changes now need to be propagated upstream to `Scope`, and then downstream to all implementers of `Scope`.

There are more examples of this:
```scala
-      ensureCapacity(if (hashTable ne null) hashTable.length else MinHashedScopeSize)
+      ensureCapacity(if (hashTable ne null) hashTable.nn.length else MinHashedScopeSize)

-        e = hashTable(name.hashCode & (hashTable.length - 1))
-        while ((e ne null) && e.name != name) {
-          e = e.tail
+        e = hashTable.nn(name.hashCode & (hashTable.nn.length - 1))
+        while ((e ne null) && e.nn.name != name) {
+          e = e.nn.tail
         }
```

#### 5.0 dotc/sbt/ExtractAPI.scala (86/635)

Around 40 usages of `.nn` in this file come from interactions with Java classes in package `xsbti.api`.
Examples:
```scala
   private def withMarker(tp: api.Type, marker: api.Annotation) =
-    api.Annotated.of(tp, Array(marker))
+    api.Annotated.of(tp, Array(marker)).nn
   private def marker(name: String) =
-    api.Annotation.of(api.Constant.of(Constants.emptyType, name), Array())
+    api.Annotation.of(api.Constant.of(Constants.emptyType, name), Array()).nn
...
   def combineApiTypes(apiTps: api.Type*): api.Type = {
     api.Structure.of(api.SafeLazy.strict(apiTps.toArray),
-      api.SafeLazy.strict(Array()), api.SafeLazy.strict(Array()))
+      api.SafeLazy.strict(Array()), api.SafeLazy.strict(Array())).nn
```

All these _could_ be avoided if we could recognize Java annotations _and_ the Java library were annotated.

#### 6 dotc/util/WeakHashSet.scala (71/406)

The top-level signature of `WeakHashSet` changed:
```scala
-final class WeakHashSet[A <: AnyRef](initialCapacity: Int, loadFactor: Double) extends mutable.Set[A] {
+final class WeakHashSet[A >: Null <: Nullable[AnyRef]](initialCapacity: Int, loadFactor: Double) extends m
utable.Set[A] {
```

Since `WeakHashSet` is used in the backend, this further requires that changes be propagated through the backend:
```scala
backend/jvm/BackendInterface.scala:    def newWeakSet[K >: Null <: AnyRef](): dotty.tools.dotc.util.WeakHashSet[K]
backend/jvm/DottyBackendInterface.scala:import dotty.tools.dotc.util.WeakHashSet
backend/jvm/DottyBackendInterface.scala:    def newWeakSet[K >: Null <: AnyRef](): WeakHashSet[K] = new WeakHashSet[K]()
```

The requirement that `A >: Null` comes from a bunch of places (see `elem match null` and `null.asInstanceOf[A]` below):
```scala
  def get(elem: A): Option[A] = Option(findEntry(elem))

  // from scala.reflect.internal.Set, find an element or null if it isn't contained
  def findEntry(elem: A): A = elem match {
    case null => throw new NullPointerException("WeakHashSet cannot hold nulls")
    case _    => {
      removeStaleEntries()
      val hash = elem.hashCode
      val bucket = bucketFor(hash)

      @tailrec
      def linkedListLoop(entry: Nullable[Entry[A]]): A = entry match {
        case null                    => null.asInstanceOf[A]
        case _                       => {
          val entryElem = entry.nn.get
          if (elem.equals(entryElem)) entryElem
          else linkedListLoop(entry.nn.tail)
        }
      }

      linkedListLoop(table(bucket))
    }
  }
```

Additional changes are required because flow inference is not smart enough when handling pattern matching:
```scala
-      def linkedListLoop(entry: Entry[A]): Unit = entry match {
+      def linkedListLoop(entry: Nullable[Entry[A]]): Unit = entry match {
         case null                        => add()
-        case _ if elem.equals(entry.get) => ()
-        case _                           => linkedListLoop(entry.tail)
+        case _ if elem.equals(entry.nn.get) => ()
+        case _                           => linkedListLoop(entry.nn.tail)
       }
```

Notice the `entry.nn` which is required, even though we already handled the `null` case.

Plus, the table entries themselves are nullable:
```scala
-  private[this] var table = new Array[Entry[A]](computeCapacity)
+  private[this] var table = new Array[Nullable[Entry[A]]](computeCapacity)
```

### 7 backend/jvm/GenBCode.scala (68/574)

Here again we have class state that starts out as nullable.
```scala
-    private[this] var bytecodeWriter  : BytecodeWriter   = null
-    private[this] var mirrorCodeGen   : JMirrorBuilder   = null
+    private[this] var bytecodeWriter  : Nullable[BytecodeWriter]   = null
+    private[this] var mirrorCodeGen   : Nullable[JMirrorBuilder]   = null
```

Since these are vars, later references always require `.nn`. e.g. `bytecodeWriter` is referenced 9 times later on.

### 9 core/SymDenotations.scala (63/2405)



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
