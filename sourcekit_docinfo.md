---
title: SourceKit -- source.request.docinfo
---
# {{ page.title }}

*Valid July 2017 with swift:master*

Quest: find out how this works.

## Flow

* Requests.cpp:handleSemanticRequest()
    * Requests.cpp:reportDocInfo()
        * Create SKDocConsumer
        * SwiftDocSupport.cpp:SwiftLangSupport::getDocInfo()
            * SwiftDocSupport.cpp:reportSourceDocInfo()
                * Create compiler and compile up to sema
                * Collect Decls from AST into SourceTextInfo
                * reportDocEntities() to push structure data back to consumer
                    * initDocEntityInfo()
                    * initDocGenericParams()
                    * reportRelated()
                    * reportAttributes()
                * reportSourceAnnotations() to push syntax data back to consumer

## initDocEntityInfo

Pull out `available` and `deprecated` from the 'real' attribute classes -
applies to current compiler flags used for platform.

Gets doc comment via `ide::getDocumentationCommentAsXML()` which does the
parent lookup thing for undocumented members.

Collect generic params (&lt;T&gt; variables being declared in entity name) and
generic requirements ("T: C" clauses).

Create the fully-annotated declaration using the same code as `cursorinfo`,
remarkable.

Releated entities - inherits (a class), conforms (some protocols), extends (an
extensible thing).  For members, *inherits* means overrides a superclass member
and *conforms* means implementing a protocol member.

Decode `@available` but no other attributes.  Swift language constraints end up
with missing `Platform` field.

## SKDocConsumer

So then these fields are returned in the docinfo element:

* Kind

And optionally:

* Name
* USR
* Offset + Length
* Unavailable (key missing if available)
* Deprecated (key missing if not deprecated)
* Optional (key missing if mandatory)
* XML doc comment
* Fully annotated declaration
* Localization key
* List of generic parameters (names)
* List of generic requirements (strings)
* Inherits-from (kind/name/usr)
* Conforms-to (kind/name/usr)
* Extension-of (kind/name/usr)
* Attributes (but only availability)

And never for source files (vs. modules):
* Argument (still ??)
* Original USR
* DefaultImplementationOf USR
* Module Name (submodule name)
