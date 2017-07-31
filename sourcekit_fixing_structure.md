---
title: Fixing SourceKit Structure
---
# {{ page.title }}

*July 2017, swift:master*

## Part 1 - typealias

Typealias is missing from `editoropen`.  Fix plan:
1. `SyntaxModel.h` - new `SyntaxStructureKind.TypeAlias`
    It's OK to add to the middle of this enum because it does not escape the
    executable.
2. `SwiftLangSupport.cpp` - handle new kind in `getUIDForSyntaxStructureKind()`,
   map to existing `KindDeclTypeAlias`.
3. `SyntaxModel.cpp` - add another `else` to `walkToDeclPre()`.
4. Fix up tests.

Along the way:
1. Discovered `swift/tools/swift-ide-test/swift-ide-test.cpp` another dependency
   on SyntaxWalker.  Just added the handler, nothing special.
2. Need to decide what to put in `BodyLength/Offset` - go with nothing, like
   `var f = 2`.
3. Decided not to add associatedtype or anything else easy for this first PR,
   don't overcomplicate things.

[PR 11143](https://github.com/apple/swift/pull/11143)

Should have added a test for `swift-ide-test` change.

## Part 2 - everything else

Sit back + leave to Marcelo - LOL.
