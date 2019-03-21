We managed to bootstrap dotty with new explicit nulls type system.

This document will contain some notes on the experience.

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
  * num files changed (via `git diff --stat HEAD HEAD^`): *148 (37%)*
  * num LOC modified: *3142 (3%)* (1743 insertions(+), 1399 deletions(-))
  * See per-file stats below
  * usages of `.nn`:
    ```
    src git:(explicit-null-bootstrap-big) ✗ git grep "\.nn" | wc -l
    914
    ```
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

```scala
>>> valueOf with type (x$0: Double): String
>>> valueOf with type (x$0: Float): String
>>> valueOf with type (x$0: Long): String
>>> valueOf with type (x$0: Int): String
>>> valueOf with type (x$0: Char): String
>>> valueOf with type (x$0: Boolean): String
>>> valueOf with type (x$0: Array[Char], x$1: Int, x$2: Int): String
>>> valueOf with type (x$0: Array[Char]): String
>>> valueOf with type (x$0: Any): String
>>> toGenericString with type (): String
>>> newInstance with type (): T
>>> isInstance with type (x$0: Any): Boolean
>>> isAssignableFrom with type (x$0: Class[_]): Boolean
>>> isInterface with type (): Boolean
>>> isArray with type (): Boolean
>>> isPrimitive with type (): Boolean
>>> isAnnotation with type (): Boolean
>>> isSynthetic with type (): Boolean
>>> getName with type (): String
>>> getClassLoader0 with type (): ClassLoader
>>> getTypeParameters with type (): Array[java.lang.reflect.TypeVariable[Class[T]]]
>>> getInterfaces with type (): Array[Class[_]]
>>> getGenericInterfaces with type (): Array[java.lang.reflect.Type]
>>> getModifiers with type (): Int
>>> getSimpleName with type (): String
>>> getTypeName with type (): String
>>> isAnonymousClass with type (): Boolean
>>> isLocalClass with type (): Boolean
>>> isMemberClass with type (): Boolean
>>> getClasses with type (): Array[Class[_]]
>>> getFields with type (): Array[java.lang.reflect.Field]
>>> getMethods with type (): Array[java.lang.reflect.Method]
>>> getConstructors with type (): Array[java.lang.reflect.Constructor[_]]
>>> getField with type (x$0: String): java.lang.reflect.Field
>>> getMethod with type (x$0: String, x$1: Class[_]*): java.lang.reflect.Method
>>> getConstructor with type (x$0: Class[_]*): java.lang.reflect.Constructor[T]
>>> getDeclaredClasses with type (): Array[Class[_]]
>>> getDeclaredFields with type (): Array[java.lang.reflect.Field]
>>> getDeclaredMethods with type (): Array[java.lang.reflect.Method]
>>> getDeclaredConstructors with type (): Array[java.lang.reflect.Constructor[_]]
>>> getDeclaredField with type (x$0: String): java.lang.reflect.Field
>>> getDeclaredMethod with type (x$0: String, x$1: Class[_]*): java.lang.reflect.Method
>>> getDeclaredConstructor with type (x$0: Class[_]*): java.lang.reflect.Constructor[T]
>>> getProtectionDomain with type (): java.security.ProtectionDomain
>>> getRawAnnotations with type (): Array[Byte]
>>> getRawTypeAnnotations with type (): Array[Byte]
>>> getConstantPool with type (): sun.reflect.ConstantPool
>>> desiredAssertionStatus with type (): Boolean
>>> isEnum with type (): Boolean
>>> getEnumConstantsShared with type (): Array[T & Object]
>>> enumConstantDirectory with type (): java.util.Map[String, T]
>>> asSubclass with type [U](x$0: Class[U]): Class[_ <: U]
>>> isAnnotationPresent with type (x$0: Class[_ <: java.lang.annotation.Annotation]): Boolean
>>> getAnnotationsByType with type [A <: java.lang.annotation.Annotation](x$0: Class[A]): Array[A]
>>> getAnnotations with type (): Array[java.lang.annotation.Annotation]
>>> getDeclaredAnnotationsByType with type [A <: java.lang.annotation.Annotation](x$0: Class[A]): Array[A]
>>> getDeclaredAnnotations with type (): Array[java.lang.annotation.Annotation]
>>> casAnnotationType with type (x$0: sun.reflect.annotation.AnnotationType, x$1:
  sun.reflect.annotation.AnnotationType
): Boolean
>>> getAnnotationType with type (): sun.reflect.annotation.AnnotationType
>>> getDeclaredAnnotationMap with type ():
  java.util.Map[Class[_ <: java.lang.annotation.Annotation],
    java.lang.annotation.Annotation
  ]
>>> getAnnotatedSuperclass with type (): java.lang.reflect.AnnotatedType
>>> getAnnotatedInterfaces with type (): Array[java.lang.reflect.AnnotatedType]
>>> length with type (): Int
>>> isEmpty with type (): Boolean
>>> charAt with type (x$0: Int): Char
>>> codePointAt with type (x$0: Int): Int
>>> codePointBefore with type (x$0: Int): Int
>>> codePointCount with type (x$0: Int, x$1: Int): Int
>>> offsetByCodePoints with type (x$0: Int, x$1: Int): Int
>>> getBytes with type (x$0: String): Array[Byte]
>>> getBytes with type (x$0: java.nio.charset.Charset): Array[Byte]
>>> getBytes with type (): Array[Byte]
>>> equals with type (x$0: Any): Boolean
>>> contentEquals with type (x$0: StringBuffer): Boolean
>>> contentEquals with type (x$0: CharSequence): Boolean
>>> equalsIgnoreCase with type (x$0: String): Boolean
>>> compareTo with type (x$0: String): Int
>>> compareToIgnoreCase with type (x$0: String): Int
>>> regionMatches with type (x$0: Int, x$1: String, x$2: Int, x$3: Int): Boolean
>>> regionMatches with type (x$0: Boolean, x$1: Int, x$2: String, x$3: Int, x$4: Int): Boolean
>>> startsWith with type (x$0: String, x$1: Int): Boolean
>>> startsWith with type (x$0: String): Boolean
>>> endsWith with type (x$0: String): Boolean
>>> hashCode with type (): Int
>>> indexOf with type (x$0: Int): Int
>>> indexOf with type (x$0: Int, x$1: Int): Int
>>> lastIndexOf with type (x$0: Int): Int
>>> lastIndexOf with type (x$0: Int, x$1: Int): Int
>>> indexOf with type (x$0: String): Int
>>> indexOf with type (x$0: String, x$1: Int): Int
>>> lastIndexOf with type (x$0: String): Int
>>> lastIndexOf with type (x$0: String, x$1: Int): Int
>>> substring with type (x$0: Int): String
>>> substring with type (x$0: Int, x$1: Int): String
>>> subSequence with type (x$0: Int, x$1: Int): CharSequence
>>> concat with type (x$0: String): String
>>> replace with type (x$0: Char, x$1: Char): String
>>> matches with type (x$0: String): Boolean
>>> contains with type (x$0: CharSequence): Boolean
>>> replaceFirst with type (x$0: String, x$1: String): String
>>> replaceAll with type (x$0: String, x$1: String): String
>>> replace with type (x$0: CharSequence, x$1: CharSequence): String
>>> toLowerCase with type (x$0: java.util.Locale): String
>>> toLowerCase with type (): String
>>> toUpperCase with type (x$0: java.util.Locale): String
>>> toUpperCase with type (): String
>>> trim with type (): String
>>> toCharArray with type (): Array[Char]
>>> intern with type (): String
>>> compareTo with type (x$0: Any): Int
>>> compareTo with type (x$0: T): Int
>>> length with type (): Int
>>> charAt with type (x$0: Int): Char
>>> subSequence with type (x$0: Int, x$1: Int): CharSequence
>>> isAnnotationPresent with type (x$0: Class[_ <: java.lang.annotation.Annotation]): Boolean
>>> getAnnotations with type (): Array[java.lang.annotation.Annotation]
>>> getAnnotationsByType with type [T <: java.lang.annotation.Annotation](x$0: Class[T]): Array[T]
>>> getDeclaredAnnotationsByType with type [T <: java.lang.annotation.Annotation](x$0: Class[T]): Array[T]
>>> getDeclaredAnnotations with type (): Array[java.lang.annotation.Annotation]
>>> forName with type (x$0: String): Class[_]
>>> forName with type (x$0: String, x$1: Boolean, x$2: ClassLoader): Class[_]
>>> getPrimitiveClass with type (x$0: String): Class[_]
>>> getExecutableTypeAnnotationBytes with type (x$0: java.lang.reflect.Executable): Array[Byte]
```
    
## Per-file change stats (sorted)
`git diff --stat HEAD HEAD^|awk '{ print $3 " "$4 " " $1}'| sort -n -r|less | cat`

```
274 ++++++++++----------- .../tools/dotc/reporting/diagnostic/messages.scala
219 ++++++++-------- .../tools/backend/jvm/DottyBackendInterface.scala
207 ++++++++-------- compiler/src/dotty/tools/dotc/core/Types.scala
148 ++++++----- .../dotty/tools/backend/jvm/BCodeSkelBuilder.scala
138 +++++------ compiler/src/dotty/tools/dotc/core/Scopes.scala
90 +++---- .../dotty/tools/backend/jvm/BackendInterface.scala
86 ++++--- compiler/src/dotty/tools/dotc/sbt/ExtractAPI.scala
71 +++--- .../src/dotty/tools/dotc/util/WeakHashSet.scala
68 +++-- .../src/dotty/tools/backend/jvm/GenBCode.scala
66 +++-- compiler/src/dotty/tools/dotc/core/Contexts.scala
60 +++-- compiler/src/dotty/tools/dotc/sbt/ShowAPI.scala
52 ++-- .../src/dotty/tools/dotc/typer/Implicits.scala
51 ++-- .../src/dotty/tools/dotc/core/TypeComparer.scala
49 ++-- .../dotty/tools/backend/jvm/BCodeBodyBuilder.scala
47 ++-- .../dotty/tools/dotc/core/OrderingConstraint.scala
46 ++-- .../dotc/core/unpickleScala2/Scala2Unpickler.scala
42 ++-- compiler/src/dotty/tools/dotc/core/Names.scala
40 ++- .../src/dotty/tools/dotc/parsing/Parsers.scala
38 ++- compiler/src/dotty/tools/backend/jvm/BTypes.scala
38 ++- .../src/dotty/tools/dotc/typer/ImportInfo.scala
33 ++- compiler/src/dotty/tools/io/AbstractFile.scala
31 ++- compiler/src/dotty/tools/io/Jar.scala
30 +-- compiler/src/dotty/tools/repl/ScriptEngine.scala
29 +-- compiler/src/dotty/tools/io/ZipArchive.scala
29 ++- compiler/src/dotty/tools/io/Path.scala
29 ++- compiler/src/dotty/tools/dotc/core/Hashable.scala
28 +-- .../src/dotty/tools/dotc/typer/ConstFold.scala
28 +-- .../src/dotty/tools/backend/sjs/JSCodeGen.scala
27 +- compiler/src/dotty/tools/dotc/Run.scala
27 +- .../src/dotty/tools/dotc/config/Properties.scala
26 +- .../dotc/core/classfile/ClassfileParser.scala
24 +- .../src/dotty/tools/dotc/parsing/Scanners.scala
23 +- compiler/src/dotty/tools/dotc/core/Uniques.scala
23 +- .../tools/dotc/classpath/DirectoryClassPath.scala
22 +- .../src/dotty/tools/dotc/core/TyperState.scala
22 +- .../dotty/tools/backend/jvm/BCodeSyncAndTry.scala
20 +- compiler/src/dotty/tools/dotc/typer/Namer.scala
20 +- .../tools/dotc/transform/OverridingPairs.scala
19 +- .../src/dotty/tools/dotc/core/Substituters.scala
19 +- .../dotc/parsing/xml/SymbolicXMLBuilder.scala
17 +- .../src/dotty/tools/dotc/typer/ProtoTypes.scala
16 +- .../dotty/tools/backend/jvm/BytecodeWriters.scala
15 +- .../tools/dotc/core/tasty/TreeUnpickler.scala
15 +- .../src/dotty/tools/dotc/core/TypeErrors.scala
14 +- compiler/src/dotty/tools/dotc/ast/Trees.scala
14 +- .../tools/dotc/transform/PatternMatcher.scala
14 +- .../src/dotty/tools/dotc/util/Attachment.scala
14 +- .../src/dotty/tools/dotc/transform/LazyVals.scala
14 +- .../src/dotty/tools/dotc/parsing/xml/Utility.scala
14 +- .../reporting/diagnostic/MessageContainer.scala
14 +- .../dotty/tools/dotc/transform/ReifyQuotes.scala
14 +- .../dotc/classpath/VirtualDirectoryClassPath.scala
13 +- .../tools/dotc/classpath/ClassPathFactory.scala
13 +- .../src/dotty/tools/dotc/typer/Applications.scala
13 +- .../src/dotty/tools/backend/jvm/BCodeHelpers.scala
12 +- .../tools/dotc/interactive/InteractiveDriver.scala
12 +- .../src/dotty/tools/dotc/profile/Profiler.scala
12 +- .../dotty/tools/dotc/reporting/StoreReporter.scala
12 +- .../dotc/decompiler/DecompilationPrinter.scala
12 +- .../classpath/ZipAndJarFileLookupFactory.scala
11 +- compiler/src/dotty/tools/io/NoAbstractFile.scala
11 +- .../src/dotty/tools/dotc/core/Decorators.scala
11 +- .../dotty/tools/dotc/sbt/ExtractDependencies.scala
11 +- .../dotty/tools/dotc/core/ConstraintRunInfo.scala
11 +- .../dotty/tools/backend/jvm/BCodeIdiomatic.scala
10 - tests/pos/explicit-null-option-ornull.scala
10 +- compiler/src/dotty/tools/io/VirtualDirectory.scala
10 +- compiler/src/dotty/tools/dotc/core/Phases.scala
10 +- compiler/src/dotty/tools/dotc/ast/tpd.scala
10 +- compiler/src/dotty/tools/dotc/Driver.scala
10 +- .../tools/dotc/reporting/MessageRendering.scala
10 +- .../src/dotty/tools/dotc/typer/RefChecks.scala
10 +- .../src/dotty/tools/dotc/typer/Inferencing.scala
10 +- .../src/dotty/tools/dotc/core/SymbolLoaders.scala
10 +- .../src/dotty/tools/backend/sjs/JSPositions.scala
9 +- compiler/src/dotty/tools/io/File.scala
9 +- .../src/dotty/tools/dotc/core/Annotations.scala
8 +- library/src/dotty/runtime/LazyVals.scala
8 +- compiler/src/dotty/tools/dotc/util/HashSet.scala
8 +- compiler/src/dotty/tools/dotc/typer/Typer.scala
8 +- .../tools/dotc/config/WrappedProperties.scala
8 +- .../src/dotty/tools/dotc/transform/Splicer.scala
8 +- .../src/dotty/tools/dotc/transform/MegaPhase.scala
8 +- .../src/dotty/tools/dotc/quoted/QuoteDriver.scala
8 +- .../src/dotty/tools/dotc/fromtasty/Debug.scala
8 +- .../src/dotty/tools/dotc/core/Denotations.scala
8 +- .../dotty/tools/dotc/transform/MoveStatics.scala
8 +- .../dotc/classpath/ZipArchiveFileLookup.scala
7 +- compiler/src/dotty/tools/repl/JLineTerminal.scala
7 +- compiler/src/dotty/tools/io/Directory.scala
7 +- .../src/dotty/tools/dotc/core/Definitions.scala
6 +- compiler/src/dotty/tools/dotc/typer/Inliner.scala
6 +- compiler/src/dotty/tools/dotc/plugins/Plugin.scala
6 +- .../tools/dotc/printing/SyntaxHighlighting.scala
6 +- .../tools/dotc/core/tasty/DottyUnpickler.scala
6 +- .../tools/dotc/core/quoted/PickledQuotes.scala
6 +- .../src/dotty/tools/dotc/util/SourceFile.scala
6 +- .../src/dotty/tools/dotc/config/OutputDirs.scala
6 +- .../dotty/tools/dotc/transform/patmat/Space.scala
6 +- .../dotty/tools/dotc/transform/CapturedVars.scala
6 +- .../dotty/tools/dotc/interactive/Completion.scala
5 - .../src-non-bootstrapped/scala/ExplicitNulls.scala
5 +- library/src/scala/runtime/EnumValues.scala
5 +- library/src/scala/TupleXXL.scala
5 +- compiler/src/dotty/tools/repl/Rendering.scala
5 +- compiler/src/dotty/tools/io/VirtualFile.scala
5 +- compiler/src/dotty/tools/dotc/core/TypeOps.scala
5 +- .../src/dotty/tools/dotc/core/TypeErasure.scala
5 +- .../src/dotty/tools/dotc/config/PathResolver.scala
5 +- .../dotty/tools/repl/AbstractFileClassLoader.scala
5 +- .../dotty/tools/backend/jvm/BCodeAsmCommon.scala
4 - library/src-bootstrapped/scala/ExplicitNulls.scala
4 +- compiler/src/dotty/tools/io/ClassPath.scala
4 +- compiler/src/dotty/tools/dotc/util/DiffUtil.scala
4 +- compiler/src/dotty/tools/dotc/core/Constants.scala
4 +- compiler/src/dotty/tools/dotc/ast/untpd.scala
4 +- compiler/src/dotty/tools/dotc/ast/Positioned.scala
4 +- compiler/src/dotty/tools/dotc/ast/Desugar.scala
4 +- .../tools/dotc/transform/TypeTestsCasts.scala
4 +- .../tools/dotc/transform/SuperAccessors.scala
4 +- .../tools/dotc/transform/GenericSignatures.scala
4 +- .../tools/dotc/tastyreflect/PositionOpsImpl.scala
4 +- .../tools/dotc/tastyreflect/ContextOpsImpl.scala
4 +- .../tools/dotc/reporting/diagnostic/Message.scala
4 +- .../tools/dotc/classpath/AggregateClassPath.scala
4 +- .../src/dotty/tools/dotc/util/SourcePosition.scala
4 +- .../src/dotty/tools/dotc/transform/Flatten.scala
4 +- .../src/dotty/tools/dotc/transform/Erasure.scala
4 +- .../src/dotty/tools/dotc/sbt/ThunkHolder.scala
4 +- .../src/dotty/tools/dotc/printing/Formatting.scala
4 +- .../src/dotty/tools/dotc/parsing/JavaParsers.scala
4 +- .../src/dotty/tools/dotc/config/Settings.scala
4 +- .../src/dotty/tools/dotc/CompilationUnit.scala
4 +- .../src/dotty/tools/backend/sjs/ScopedVar.scala
4 +- .../dotty/tools/dotc/transform/AccessProxies.scala
4 +- .../dotty/tools/dotc/fromtasty/TastyFileUtil.scala
4 +- .../dotty/tools/dotc/core/tasty/TreePickler.scala
4 +- .../dotty/tools/dotc/core/tasty/TastyString.scala
4 +- .../dotc/decompiler/IDEDecompilerDriver.scala
3 +- compiler/src/dotty/tools/repl/ReplDriver.scala
3 +- compiler/src/dotty/tools/io/PlainFile.scala
3 +- compiler/src/dotty/tools/io/JarArchive.scala
3 +- .../tools/dotc/parsing/xml/MarkupParsers.scala
3 +- .../src/dotty/tools/dotc/plugins/Plugins.scala
3 +- .../src/dotty/tools/dotc/classpath/FileUtils.scala
3 +- .../dotc/transform/CollectNullableFields.scala
2 +- project/Build.scala
2 +- compiler/src/dotty/tools/dotc/core/Symbols.scala
```
