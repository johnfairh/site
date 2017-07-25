---
title: Testing SourceKit
---
# {{ page.title }}

*July 2017*

Some spells for testing SourceKit.

## sourcekitd-test

Setup:
```shell
alias st=/Users/johnf/project/swift-source/build/jfdev/swift-macosx-x86_64/bin/sourcekitd-test
```

Simple tests against files, no imports:
```shell
st -req=structure ${PWD}/test.swift -- ${PWD}/test.swift`
```

Add Foundation etc:
```shell
st -req=structure -using-swift-args ${PWD}/test.swift -- \
        ${PWD}/test.swift \
        -sdk `xcrun --sdk macosx --show-sdk-path`
```

CursorInfo:
```shell
st -req=cursor -offset=XXX -print-response-as-json \
        ${PWD}/test.swift -- ${PWD}/test.swift
```

## swift-ide-test
Setup:
```shell
alias si=/Users/johnf/project/swift-source/build/jfdev/swift-macosx-x86_64/bin/swift-ide-test
```

Structure query:
```shell
si -structure -source-filename test.swift
```

## Xcode

To get some idea of what Xcode is doing and how happy it feels, after switching
to custom toolchain:

```shell
SOURCEKIT_LOGGING=3 /Applications/Xcode-beta.app/Contents/MacOS/Xcode 2>&1 | tee log
```
