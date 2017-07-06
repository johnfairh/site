---
title: SourceKit -- source.request.editor.open
---
# {{ page.title }}

*Valid July 2017 with swift:master*

Quest: find out why subscript + typealias do not show up.

## Flow

* Requests.cpp:handleSemanticRequest()
    * Requests.cpp:editorOpen()
        * Create SKEditorConsumer
        * SwiftEditor.cpp:SwiftLangSupport::editorOpen()
            * Create SwiftEditorDocument
            * SwiftEditorDocument::parse()
            * SwiftEditorDocument::readSyntaxInfo(Consumer)
            * SwiftEditorDocument::readSemanticInfo(Consumer)

## SKEditorConsumer

ABC `EditorConsumer` in SourceKit/include/SourceKit/core/LangSupport.h.

Passes through data unchanged into the dictionary.  Tells us what fields occur
though in this view, represented by `DocStructureArray.cpp:Node`:
* Offset
* Length
* Kind
* NameOffset
* NameLength 

Then optionally:
* AccessLevel
* SetterAccessLevel
* BodyOffset (always occurs with BodyLength)
* BodyLength (always occurs with BodyOffset)
* DisplayName
* TypeName
* RuntimeName
    * Only set for non-generic classes + protocols at the top level, see
      `SwiftDocumentStructureWalker`.
* SelectorName
    * Only set if `@IBAction`, see `SwiftDocumentStructureWalker`.
* InheritedTypes
* Attrs
* Elements
    * *not sure what this is yet, crops up with inherited types at least,
      comes from `handleDocumentSubStructureElement()`*
* Substructure

Still not sure what a 'semantic annotation' is - thing times out when I request
semantic processing.

=> Not responsible for discarding subscripts and typealias.

## SwiftEditorDocument

Important to realize this is the relatively-long-lived data structure that is
held for Xcode for each file.  The one-shot usage I care about is not the main
case.

Defined in SwiftLangSupport.h, implementation over in SwiftEditor.cpp.
```c++
class SwiftEditorDocument :
        public ThreadSafeRefCountedBase<SwiftEditorDocument>
```
Members in `SwiftEditorDocument::Implementation` struct, nothing worrying.

The `SwiftDocumentStructureWalker` is the thing that pushes structure nodes
back to the client.  There are no early 'return's from `walkToSubStructurePre()`
so no blame here.

This walker is invoked in a slightly roundabout way by
`SwiftEditorDocument::readSyntaxInfo()`:
* Create `ide::SyntaxModelContext` from the source file.
* Create a `SwiftEditorSyntaxWalker` against our sourcekitd consumer, and apply
  it to the context.
* `SwiftEditorSyntaxWalker` has its own `SwiftDocumentStructureWalker` and
  proxies appropriate 'walk' methods on to it.

Lots of opportunity for dropping the ball here but the relevant code is simple
and looks to be correct.

So we move on to this `ide::SyntaxModelContext::walk()` chappy.

It's crystallizing a bit why this request might be missing things compared to
the others: it is working directly on the syntax of the file without running the
actual Swift parser over it, whereas the doc-info and cursor-info APIs both run
against the AST.

## SyntaxModelContext

<i>(Now we are in swift/lib/IDE/SyntaxModel.cpp)</i>

TBC!
