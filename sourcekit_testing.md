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
