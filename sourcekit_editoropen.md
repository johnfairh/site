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
* DisplayName (ie. name)
* TypeName
* RuntimeName (for top-level non-generic classes and protocols)
* SelectorName (Only set if `@IBAction`, see `SwiftDocumentStructureWalker`.)
* InheritedTypes - list of names of inherited types/adopted protocols for
  nominal types and extensions.
* Attrs
* Elements - two uses:
    1. kind[ref]/offset/length for the inherited types
    2. enum element init expression (`case f = 2`).  Odd.

=> Not responsible for discarding subscripts and typealias.

### Semantic annotations

'Semantic annotations' here are an array of {kind/offset/length/system}
structures that come out between the syntaxmap and the substructure.  They are
generated from a walk of the AST looking for `visitDeclReference()` calls --
that is not stuff being declared, rather refs to other types.  The SourceKit
keys are the *source.lang.swift.ref.* family.

=> Even if I could make this work it would not help.

## SwiftEditorDocument

This is the relatively-long-lived data structure that is held for Xcode for
each file.  The one-shot usage I care about is not the main case.

Defined in SwiftLangSupport.h, implementation over in SwiftEditor.cpp.
```c++
class SwiftEditorDocument :
        public ThreadSafeRefCountedBase<SwiftEditorDocument>
```
Members in `SwiftEditorDocument::Implementation` struct.  The most important
part is the `SwiftDocumentSyntaxInfo` class that holds the file data and owns
the `ParserUnit` that holds and populates the actual `SourceFile`.

The call to `SwiftEditorDocument::parse()` invokes the Swift compiler and
creates the full AST in the `SourceFile` ready to be looked at.

### Walker

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

The routine that maps to SourceKit UIDs,
`SwiftLangSupport::getUIDForSyntaxStructureKind()`, does not expect to receive
calls about typealiases or subscripts, so this is all consistent with the
upstream code.

So move on to `ide::SyntaxModelContext::walk()`.

## SyntaxModelContext

<i>(Now we are in swift/lib/IDE/SyntaxModel.cpp)</i>

`SyntaxModelContext` constructor tokenizes the file (so this is the second pass
over the file, it has already been parsed once) and builds up an array of
`SyntaxNode`s.

Then the `walk` method builds a `ModelASTWalker` and runs it over the source
file.

The point of all this is to emit a merged, linear view of syntax and structure.
The walk is driven by the structure -- from the `ASTWalker` -- but, broadly,
when it it finds a structure node to emit, it first consumes the `SyntaxNode`s
that occur before the structure node and spits them out to the client.

*This is all a bit weird given in our client we do not care about the relation
between the two worlds -- brief look, can't find anyone who does.  Perhaps in
Xcode or some other closed-source component.*

On the face of it then, the problem is that `ModelASTWalker::walkToDeclPre()`
does a non-exhaustive check of Decl types that ignores typealiases at least.

Examining the `Decl` tree shows the following are missed by this routine:
* `ImportDecl`
* `TypeAliasDecl`
* `AssociatedTypeDecl`
* `SubscriptDecl`
* `ConstructorDecl` and `DestructorDecl` treated as regular functions - means
  they end up with the wrong kind.
* (`MissingMemberDecl` but don't care.)

This entirely explains the observed behaviour including why subscript parameters
show up in the wrong place - because the `SubscriptDecl` is missed, the tree
doesn't nest and the params that come in next just stick at the current level.

Xcode can't be relying on this!  Looks easy enough to fix these, though haven't
checked for any other consumers.  Suspect none.

Another bug: the accessibility key is missing from extension declarations.  This
is because `SwiftDocumentStructureWalker::walkToSubStructurePre()` only looks
for accessibility on `ValueDecl`s which does not cover `ExtensionDecl`.
