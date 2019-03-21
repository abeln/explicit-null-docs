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
The annotations don't seem to be very useful, however. Disabling the annotations produces just _4_ additional type errors:
e.g.
```scala
[error] -- [E007] Type Mismatch Error: /Users/abeln/src/dotty2/dotty/library/src-bootstrapped/scala/reflect/GenericClass.scala:44:46
[error] 44 |    def addElem = elems += labelsStr.substring(start, cur)
[error]    |                           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
[error]    |                           Found:    String | JavaNull
[error]    |                           Required: String
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
