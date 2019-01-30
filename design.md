![project logo](https://i.imgur.com/587FGRs.png)
# Scala with Explicit Nulls

## Contents
  - [What is This?](#what-is-this)
  - [New Type Hierarchy](#new-type-hierarchy)
  - [Unsoundness](#unsoundness)
  - [Equality](#equality)
  - [Working with Null](#working-with-null)
  - [Java Interop](#java-interop)
    - [Nullification Function](#nullification-function)
    - [JavaNull](#javanull)
    - [Improving Precision](#improving-precision)
      - [toString](#toString)
  - [Binary Compatibility](#binary-compatibility)
  - [Flow-Sensitive Type Inference](#flow-sensitive-type-inference)
    - [Non-Stable Paths](#non-stable-paths)
    - [Logical Operators](#logical-operators)
    - [Inside Conditions](#inside-conditions)
    - [Algorithm](#algorithm)
    - [Unsupported Idioms](#unsupported-idioms)
  - [Local Type Inference](#local-type-inference)

## What is This?

This proposal and the accompanying [pull request](https://github.com/lampepfl/dotty/pull/5747) describe a modification to the Scala type system
that makes reference types (anything that extends `AnyRef`) _non-nullable_.

This means the following code will no longer typecheck
```scala
val x: String = null // error: found `Null`,  but required `String`
```

Instead, to mark a type as nullable we use a [type union](https://dotty.epfl.ch/docs/reference/new-types/union-types.html)
```
val x: String|Null = null // ok
```

Read on for details.

## New Type Hierarchy

There are two type hierarchies with respect to `Null`, depending on whether
we're _before_ or _after_ erasure.

Before erasure, `Null` is no longer a subtype of all reference types.
Additionally, `Null` is a subtype of `Any` directly, as opposed to `AnyRef`.

![type hierarchy before erasure](https://user-images.githubusercontent.com/1782179/51210362-9bf86900-18e0-11e9-9485-f40dc9061527.png)

After erasure, `Null` remains a subtype of all reference types (as forced
by the JVM).

![type hierarchy after erasure](https://user-images.githubusercontent.com/1782179/51210554-175a1a80-18e1-11e9-91a1-ca2188108e30.png)

## Unsoundness

The new type system is unsound with respect to `null`. This means there are still instances where an expressions has a
non-nullable type like `String`, but its value is `null`.

The unsoundness happens because uninitialized fields in a class start out as `null`:
```scala
class C {
  val f: String = foo(f)
  def foo(f2: String): String = if (f2 == null) "field is null" else f2 
}
val c = new C()
// c.f == "field is null"
```

Enforcing sound initialization is a non-goal of this proposal. However, once we have a type system where nullability is explicit,
we can use a sound initialization scheme like the one proposed by @liufengyun and @biboudis in [https://github.com/lampepfl/dotty/pull/4543](https://github.com/lampepfl/dotty/pull/4543)
to eliminate this particular source of unsoundness. 


## Equality

Because of the unsoundness, we need to allow comparisons of the form `x == null` or `x != null` even when `x` has a non-nullable reference
type (but not a value type). This is so we have an "escape hatch" for when we know `x` is nullable even when the type says
it shouldn't be.
```scala
val x: String|Null = null
x == null       // ok: x is a nullable string
"hello" == null // ok: String is a reference type
1 == null       // error: Int is a value type
```

### Reference Equality

Recall that `Null` is now a direct subtype of `Any`, as opposed to `AnyRef`. However, we also need to allow
reference equality comparisons:
```scala
val x: String = null
x eq null // ok: could return `true` because of unsoundness
```

We implement this by moving the `eq` and `ne` methods out of `AnyRef` and into a new trait `RefEq` that is extended
by both `AnyRef` and `Null`:
```scala
trait RefEq {
  def eq(that: RefEq): Boolean
  def ne(that: RefEq): Boolean
}

class AnyRef extends Any with RefEq
class Null extends Any with RefEq
```

## Working with Null

To make working with nullable values easier, we propose adding a few utilities to the standard library.
So far, we have found the following useful:
  - An extension method `.nn` to "cast away" nullability
    ```scala
    implicit class NonNull[T](x: T|Null) extends AnyVal {
      def nn: T = if (x == null) {
        throw new NullPointerException("tried to cast away nullability, but value is null")
      } else {
        x.asInstanceOf[T]
      }
    }
    ```
    This means that given `x: String|Null`, `x.nn` has type `String`, so we can call all the usual methods
    on it. Of course, `x.nn` will throw a NPE if `x` is `null`.
  - Implicit conversions from/to nullable arrays
    ```scala
    implicit def fromNullable1[T](a: Array[T|Null]): Array[T] = a.asInstanceOf[Array[T]]
    implicit def toNullable1[T](a: Array[T]): Array[T|Null] = a.asInstanceOf[Array[T|Null]]
    // similar methods for matrices and higher dimensions: e.g. `toNullable{2, 3, ...}`
    ```
    These are useful because Java APIs often return nullable arrays. Additionally, because `Array` is
    invariant neither `Array[T] <: Array[T|Null]` nor the other way, so to go from one to the other we need
    casts. For example, suppose we want to write a function that sorts arrays
    ```scala
    def sort[T <: AnyRef : Ordering](a: Array[T]): Array[T] = {
      // error: `copyOf` expects an `Array[T|Null]` and returns an `Array[T|Null]`
      val a2: Array[T] = java.util.Arrays.copyOf(a, a.length).nn
      scala.util.Sorting.quickSort(a2)
      a2
    }
    ```    
    To fix the error we use the implicit conversions above, which turn the third line into
    ```scala
    val a2: Array[T] = fromNullable1(java.util.Arrays.copyOf(toNullable1(a), a.length)).nn
    ```
    Of course, this is also unsound and `a2` could end up with a `null` value in it.

## Java Interop

The compiler can load Java classes in two ways: from source or from bytecode. In either case, when a Java class is loaded, we "patch" the type of its members to reflect that Java types remain implicitly nullable.

Specifically, we patch
  * the type of fields
  * the argument type and return type of methods

### Nullification Function

We do the patching with a "nullification" function `nf` on types:
```
1. nf(R)      = R|JavaNull          if R is a reference type (a subtype of AnyRef)
2. nf(R)      = R                   if R is a value type (a subtype of AnyVal)
3. nf(T)      = T|JavaNull          if T is a type parameter
4. nf(C[R])   = C[R]|JavaNull       if C is Java-defined
5. nf(C[R])   = C[nf(R)]|JavaNull   if C isn't Java-defined
6. nf(A => B) = nf(A) => nf(B)    
7. nf(A & B)  = nf(A) & nf(B) 
8. nf(T)      = T                   otherwise (T is any other type)
```

`JavaNull` is an alias for `Null` with magic properties (see below). We illustrate the rules for `nf` below with examples.

  * The first two rules are easy: we nullify reference types but not value types.
    ```scala
    class C {
      String s;
      int x;
    }
    ==>
    class C {
      val s: String|Null
      val x: Int
    }
    ```

  * In rule 3 we nullify type parameters because in Java a type parameter is always nullable, so the following code compiles.
    ```scala
    class C<T> { T foo() { return null; } }
    ==>
    class C[T] { def foo(): T|Null } 
    ```

    Notice this is rule is sometimes too conservative, as witnessed by
    ```scala
    class InScala {
      val c: C[Bool] = ???  // C as above
      val b: Bool = c.foo() // no longer typechecks, since foo now returns Bool|Null
    }
    ```
    
  * Rule 4 reduces the number of redundant nullable types we need to add. Consider
    ```scala
    class Box<T> { T get(); }
    class BoxFactory<T> { Box<T> makeBox(); }
    ==>
    class Box[T] { def get(): T|JavaNull }
    class BoxFactory[T] { def makeBox(): Box[T]|JavaNull }
    ```
    
    Suppose we have a `BoxFactory[String]`. Notice that calling `makeBox()` on it returns a `Box[String]|JavaNull`, not
    a `Box[String|JavaNull]|JavaNull`, because of rule 4. This seems at first glance unsound ("What if the box itself
    has `null` inside?"), but is sound because calling `get()` on a `Box[String]` returns a `String|JavaNull`, as per
    rule 3.
    
    Notice that for rule 4 to be correct we need to patch _all_ Java-defined classes that transitively appear in the
    argument or return type of a field or method accessible from the Scala code being compiled. Absent crazy reflection
    magic, we think that all such Java classes _must_ be visible to the Typer in the first place, so they will be patched.
        
  * Rule 5 is needed because Java code might use a generic that's defined in Scala as opposed to Java.
    ```scala
    class BoxFactory<T> { Box<T> makeBox(); } // Box is Scala defined
    ==>
    class BoxFactory[T] { def makeBox(): Box[T|JavaNull]|JavaNull }
    ```
    
    In this case, since `Box` is Scala-defined, `nf` is applied to the type argument `T`, so rule 3 applies and we get
    `Box[T|JavaNull]|JavaNull`. This is needed because our nullability function is only applied (modularly) to the Java
    classes, but not to the Scala ones, so we need a way to tell `Box` that it contains a nullable value.

  * Rules 6 and 7 just recurse structurally on the components of the type.
    The implementation of rule 7 n the compiler are a bit more involved than the presentation above.
    Specifically, the implementation makes sure to add `| Null` only at the top level of a type:
    e.g. `nf(A & B) = (A & B) | JavaNull`, as opposed to `(A | JavaNull) & (B | JavaNull)`.
  
### JavaNull
To enable method chaining on Java-returned values, we have a special `JavaNull` alias
```scala
type JavaNull = Null @FromJava
```

`JavaNull` behaves just like `Null`, except it allows (unsound) member selections:
```scala
// Assume someJavaMethod()'s original Java signature is
// String someJavaMethod() {}
val s2: String = someJavaMethod().trim().substring(2).toLowerCase() // unsound
```

Here, all of `trim`, `substring` and `toLowerCase` return a `String|JavaNull`.
The Typer notices the `JavaNull` and allows the member selection to go through.
However, if `someJavaMethod` were to return `null`, then the first member selection
would throw a `NPE`.

Without `JavaNull`, the chaining becomes too cumbersome
```scala
val ret = someJavaMethod()
val s2 = if (ret != null) {
  val tmp = ret.trim()
  if (tmp != null) {
    val tmp2 = tmp.substring(2)
    if (tmp2 != null) {
      tmp2.toLowerCase()
    }
  }
}
// Additionally, we need to handle the `else` branches.
```

`JavaNull` is also special in a few other ways
  - users cannot directly write the `JavaNull` type (in this sense, `JavaNull` is similar
    to Kotlin's [_platform types_](https://kotlinlang.org/docs/reference/java-interop.html#null-safety-and-platform-types)
    which are too _non-denotable_). Instead, users can only write `Null`, which is more
    restrictive.
    ```
    val s: String|Null = someJavaMethod()
    s.trim().substring(2) // error: `trim` is not a member of `String|Null`
    ```
  - `JavaNull` can, however, be inferred
    ```
    val s = someJavaMethod() // s: String|JavaNull inferred
    s.trim().substring(2) // ok
    ```
  - if the user wants to use `JavaNull` "non-locally" (e.g. by passing it to another
    method), they'll need to either cast away the nullability or use vanilla `Null`
    (this is a consequence of the first two bullets)
    ```
    val s = someJavaMethod() // s: String|JavaNull inferred
    def foo(s: String|Null) = ???
    foo(s) // allowed, but within `foo` we have to work with the strict `Null`
    ```
  - When looking for an implicit conversion from `A|JavaNull` to `B`, if we
    can't find it, then also look for an implicit conversion from `A` to `B`.
    This is done for ease of interop with Java.
  
### Improving Precision

Our nullification function is sometimes too conservative. For example, consider the `trim`
method in `java.lang.String`:
```java
/** Returns a copy of the string, with leading and trailing whitespace omitted. */
public String trim()
```

A vanilla application of nullification will turn the above into `def trim(): String|JavaNull`.
However, inspecting the function contract in the javadoc reveals that if `s` is a non-null string
to begin with, then `s.trim()` should in fact never return `null` (if `s` is null, then `s.trim()` should throw).
From the point of view of Scala it is then preferable to have a more precise signature
for `trim` where the return type is `String`.

In general, this is a problem for _many_ methods in the Java standard library: in particular, methods
in commonly-used classes like `String` and `Class`. We address this by maintaining a _whitelist_ of
methods that need "special" treatment during nullification (e.g. nullify only the arguments, or only the return type).

Currently, the whitelist is hard-coded in Dotty, but the plan is to use an annotated [version](https://github.com/typetools/checker-framework/tree/master/checker/jdk/nullness)
of the Java JDK with nullability annotations produced by the [Checker Framework](https://checkerframework.org/)
(hat tip to @liufengyun and Werner Dietl for suggesting this).

### toString

The `toString` method is a special case where we chose to trade away soundness for precision and usability.
`toString` is special because even though it lives in `scala.Any`, it actually comes from
`java.lang.Object` and it could, in principle, be overriden to return `null`.

However, changing `toString`'s signature to return `String|JavaNull` would break too much
existing Scala code. Additionally, our assumption is that `toString` is unlikely to return null.

This means the signature of `toString` remains `def toString(): String`.
   
## Binary Compatibility

Our strategy for binary compatibility with Scala binaries that predate explicit nulls
is to leave the types unchanged and be compatible but unsound.

Concretely, the problem is how to interpret the return type of `foo` below
```scala
// As compiled by e.g. Scala 2.12
class Old {
  def foo(): String = ???
}
```
There are two options:
  - `def foo(): String`
  - `def foo(): String|Null`

The first option is unsound. The second option matches how we handle Java methods.

However, this approach is too-conservative in the presence of generics
```scala
class Old[T] {
  def id(x: T): T = x
}
==>
class Old[T] {
  def id(x: T|Null): T|Null = x
}
```

If we instantiate `Old[T]` with a value type, then `id` now returns a nullable value,
even though it shouldn't:
```scala
val o: Old[Boolean] = ???
val b = o.id(true) // b: Boolean|Null
```

So really the options are between being unsound and being too conservative.
The unsoundness only kicks in if the Scala code being used returns a `null` value.
We hypothesize that `null` is used infrequently in Scala libraries, so we go with
the first option.

If a using an unported Scala library that _produces_ `null`, the user can wrap the
(hopefully rare) API in a type-safe wrapper:
```scala
// Unported library
class Old {
  def foo(): String = null
}

// User code in explicit-null world
def fooWrapper(o: Old): String|Null = o.foo() // ok: String <: String|Null

val o: Old = ???
val s = fooWrapper(o)
```

If the offending API _consumes_ `null`, then the user can cast the null literal to
the right type (the cast will succeed, since at runtime `Null` _is_ a subtype of
any reference type).
```scala
// Unported library
class Old() {
  /** Pass a String, or null to signal a special case */
  def foo(s: String): Unit = ???
}

// User code in explicit-null world
val o: Old = ???
o.foo(null.asInstanceOf[String]) // ok: cast will succeed at runtime
```

## Flow-Sensitive Type Inference

We added a simple form of flow-sensitive type inference. The idea is that if `p` is a stable path, then we can know
that `p` is non-null if it's "compared" with the `null` literal. This information can then be propagated to the `then` and `else` branches of an if-statement (among other places).

Example:
```scala
val s: String|Null = ???
if (s != null) {
  // s: String
}
// s: String|Null
```
A similar inference can be made for the `else` case if the test is `p == null`
```scala
if (s == null) {
  // s: String|Null
} else {
  // s: String
}
```

What exactly is considered a comparison for the purposes of the flow inference:
  - `==` and `!=`
  - `eq` and `ne`
  - `isInstanceOf[Null]`. Only kicks in if the type test is against `Null`:
     ```scala
     val s: String|Null
     if (!s.isInstanceOf[Null]) {
       // s: String
     }
     ```
     If the test had been (`if (s.isInstanceOf[String])`), we currently don't infer non-nullness.

### Non-Stable Paths
If `p` isn't stable, then inferring non-nullness is potentially unsound:
```scala
var s: String|Null = "hello"
if (s != null && {s = null; true}) {
  // s == null
}
```

### Logical Operators
We also support logical operators (`&&`, `||`, and `!`):
```scala
val s: String|Null = ???
val s2: String|Null = ???
if (s != null && s2 != null) {
  // s: String
  // s2: String
}

if (s == null || s2 == null) {
  // s: String|Null
  // s2: String|Null
} else {
  // s: String
  // s2: String
}
```

### Inside Conditions
We also support type specialization _within_ the condition, taking into account that `&&` and `||` are short-circuiting:
```scala
val s: String|Null
if (s != null && s.length > 0) { // s: String in `s.length > 0`
  // s: String
}

if (s == null || s.length > 0) // s: String in `s.length > 0` {
  // s: String|Null
} else {
  // s: String|Null
}
```

### Algorithm

Let `NN(cond, true/false)` be the set of paths (`TermRefs` in the compiler) that we can infer to be non-null if `cond` is `true/false`, respectively.

Then define `NN` by (basically De Morgan's laws)
```
NN(p == null, true)  = {}                     
NN(p == null, false) = {p} if p is stable     
NN(p != null, true)  = {p} if p is stable   
NN(p != null, false) = {}                    
NN(A && B,    true)  = ∪(NN(A, true),  NN(B, true))
NN(A && B,    false) = ∩(NN(A, false), NN(B, false))
NN(A || B,    true)  = ∩(NN(A, true),  NN(B, true))
NN(A || B,    false) = ∪(NN(A, false), NN(B, false))
NN(!A,        true)  = NN(A, false)
NN(!A,        false) = NN(A, true)
NN(cond,      _)     = {} otherwise
```

To type `If(cond, then, else)`
   1. compute `NN(cond, true)` and `NN(cond, false)`
   2. type the `then` branch with the knowledge that the paths in `NN(cond, true)`
     are non-nullable
   3. ditto for the `else` branch and the paths in `NN(cond, false)`

To propagate nullability facts _within_ a condition, when typing `A && B`
  1. type `A`
  2. compute `NN(A, true)`
  3. type `B` augmented context with the facts in `NN(A, true)`

Similarly, when typing `A || B`
  1. type `A`
  2. compute `NN(A, false)`
  3. type `B` in an augmented context with the facts in `NN(A, false)`

### Unsupported Idioms
We don't support
  - reasoning about non-stable paths
  - flow facts not related to nullability (`if (x == 0) { // x: 0.type not inferred }`)
  - tracking aliasing between non-nullable paths
    ```scala
    val s: String|Null = ???
    val s2: String|Null = ???
    if (s != null && s == s2) {
      // s:  String inferred
      // s2: String not inferred
    }
    ```

## Local Type Inference
Explicit nullability interacts with local type inference in a few ways:
  - Suppor for SAM types: if `S` is a SAM type, then we consider `S|Null` as a SAM type as well. This allows
    ```scala
    trait S {
      def foo(x: Int): Int
    }
    val s: S|Null = (x: Int) => x
    ``` 
  - Inferred nullable unions: Dotty currently never infers a union type; instead, the union is eagerly widened.
    (see [#4687](https://github.com/lampepfl/dotty/issues/4867)).
    ```scala
    def foo(): Int|String = 42
    val x = foo() // x: Any inferred
    ```
    We changed the inference rules so that unions of the form `T|Null` are inferred:
    ```scala
    def foo2(): String|Null = "hello"
    val x = foo2() // x: String|Null inferred
    ```
    Otherwise, the nullable type `String|Null` would be collapsed to `Any`, which loses too much information.
  
  - Nullable unions in prototypes: the compiler currently doesn't allow unions in prototypes
    ```scala
    def bar(a: Array[String|Int]|Int) = ()
    bar(Array(42))
    
    4 |  bar(Array(42))
      |      ^^^^^^^^^
      |      found:    Array[Int]
      |      required: Array[String | Int] | Int
    ```
    Notice that the type of `Array(42)` is synthesized as `Array[Int]`, even though the 
    (correct) type `Array[String|Int]` could be inherited. 
    The problem is that the compiler sees a prototype of `Array[String|Int]|Int`, and because
    the outer type is a union it discards the prototype and switches to synthesis.
    
    See [https://github.com/lampepfl/dotty/commit/8067b952875426d640968be865773f6ef3783f3c](https://github.com/lampepfl/dotty/commit/8067b952875426d640968be865773f6ef3783f3c) for
    why this was done.
    
    We've changed this behaviour so that nullable unions can be used in prototypes
    ```scala
    def bar(a: Array[String|Null]|Null) = ()
    bar(Array("hello")) // ok: inferred type parameter via prototype is String|Null`
    ```

## Other Languages

### Kotlin

TODO

## TODOs
  - Bootstrap Dotty with the new type system
  - Port the standard library to the new type system
  - Figure out how we'll present `JavaNull` in e.g. type errors
  - Support flow-sensitive type inference in TASTy
  - Flow-sensitive type inference for pattern-matching
    ```scala
    val x: String|Null = ???
    x match {
      case null => ???
      case _ => // x: String inferred
    }
    ```
  - Use the Checkers Framework to whitelist non-nullable Java methods
  - Recognize @NonNull annotation
           
