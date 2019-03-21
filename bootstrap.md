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
    
## Per-file change stats (sorted)
`git diff --stat HEAD HEAD^|awk '{ print $3 " "$4 " " $1}'| sort -n -r|less | cat`

See stats here: https://gist.github.com/abeln/d0d2979efbf469501923c7d73341e145
