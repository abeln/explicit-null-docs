We managed to bootstrap dotty with new explicit nulls type system.

This document will contain some notes on the experience.

Statistics we want to gather:
  * total number of files
  * # files changed
  * LOC in dotty
  * LOC modified
  * # of `.nn` used
  * # of usages of `JavaNull`
  * # of usages of flow inference broken down by type
    - body of `if` expressions
    - within conditions
    - within blocks
  * # of times we used checker framework info
  * # of nullable fields/arguments/return types
