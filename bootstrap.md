We managed to bootstrap dotty with new explicit nulls type system.

This document will contain some notes on the experience.

Statistics we want to gather:
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

How we're counting:
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
  * num files changed (via `git diff --stat HEAD HEAD^`): 148
  * num LOC modified: 3142 1743 insertions(+), 1399 deletions(-)
     
    
  
