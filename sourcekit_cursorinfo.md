---
title: SourceKit -- source.request.cursorinfo
---
# {{ page.title }}

*Valid Sept 2017 with swift:master*

## Flow

* Requests.cpp:handleSemanticRequest()
    * SwiftSourceDocInfo.cpp:SwiftLangSupport::getCursorInfo()
        * SwiftSourceDocInfo.cpp:resolveCursor()
        * Create CursorInfoConsumer
        * ASTManager::processASTAsync()
            * CursorInfo::handlePrimaryAST()
                * SwiftSourceDocInfo.cpp:passCursorInfoForDecl()
    * Callback Requests.cpp::reportCursorInfo()

## ASTManager

This is a cache layer to avoid compiling the same file repeatedly.

Key difference here over `request.structure` -- the file is properly compiled
up to Sema and type-checked, not just parsed.  Presence of USRs is giveaway.

## CursorInfoConsumer

Pulls attributes from the `ValueDecl` for propagation.  Note this means we
cannot be talking about an extension per se because that is further up the
hierarchy.

Doc comment handling: uses `ide::getDocumentationCommentAsXML()` which does the
parent lookup for class inheritance and protcol extensions, then does some
custom code to look up protocol conformance.  This routine creates all the XML
for the `full_as_xml` field including the declaration:
`CommentToXMLConverter::visitCommentParts` uses a very customized declaration
printer based on `PrintOptions::printInterface()`.  In particular it shows most
decl attributes *except* @discardableResult.  Probably because that is on by
default on interfaces generated from ObjC.  Which makes no sense here.

Uses `PrintOptions::printQuickHelpDeclaration()` style for the annotated
declaration field, and also the fully-annotated declaration field.  In particular
this means 'include decl attributes *except* @available'.

Uses `ide::walkOverriddenDecls()` to find USRs of swift + objC things that have
been overridden (functions and properties not nominals).

Related declarations code is incomplete but should return declarations with
the same name in the same (or lower?) scope.

Group names.  This looks to be something to do with module definitions and
serialization.  Ignore for now.

## CursorInfo and reportCursorInfo

So then these fields are are returned in the cursorinfo:

* Kind
* Name

And optionally:
* USR
* Type name
* Doc comment as XML
* Annotated declaration
* Fully-annotated declaration
* Module name [only if symbol from a separate module]
* Group name [only if symbol from a separate module]
* Localization key (from doc comment)
* Offset and Length
* Filename
* Overridden USRs list
* Available actions [can't find any code that sets this though...]
* Parent name offset (only for `ParamDecl`s)
* Related declarations list
* IsSystem - either 'true' or not here at all
* Type USR
* Container Type USR 

And never:
* TypeInterface (always empty in `CursorInfo`)
