---
title: Digging Into SourceKit Queries
---
# {{ page.title }}

*Valid July 2017 with swift:master*

## Background

SourceKit is the component of Swift that provides an API for understanding
and manipulating Swift source code.  Its conceptual model is higher-level
and simpler than the Swift compiler data structures: SourceKit itself builds
and processes those data structures to provide what it thinks is useful to
clients.

The main client of SourceKit is Xcode.  It is pretty clear that changes to
the API are made organically to meet Xcode's needs: the various parts of the
API do not hang together as a coherent designed whole. The SourceKit API is
not specified publically or much documented -- it is not covered by the
swift-evolution process. This probably gives us a better Xcode experience but
makes it tough/interesting to work with.

The point of this piece is to compare the various APIs and figure out what
best to change to improve SourceKitten & clients, particularly Jazzy.

## Query Request Types

<dl>
<dt>source.request.editor.open</dt>
<dd>
Returns the 'structure' of the code.  Issued by
<code>sourcekitd-test -req=structure</code> and
<code>sourcekitten structure</code>.
<ul>
<li>Bugs: Omits (at least) subscript and typealias declarations.</li>
<li>The only SourceKit API to return accessibility fields.</li>
<li>No USR, declaration annotation.  No XML doc comment.</li>
<li>Lists attributes by names but drops all content.</li>
<li>(Requires <code>key.syntactic_only: 1</code> otherwise times out waiting for
sema???)</li>
</ul>
</dd>
<dt>source.request.docinfo</dt>
<dd>
Intended to be a more 'useful' view of code?  Issued by
<code>sourcekitd-test -req=doc-info</code>.
<ul>
<li>Includes USR and fully-annotated declaration, XML doc comment.</li>
<li>Has great decoding of <code>@available</code> attributes but ignores all
others.</li>
</ul>
</dd>
<dt>source.request.cursorinfo</dt>
<dd>
Returns details of a single declaration.  SourceKitten uses this after
<code>structure</code> to fill in blanks.  Issued by
<code>sourcekitd-test -req=cursor</code>.
<ul>
<li>Includes USR, annotated declarations, XML doc comment.</li>
<li>Does not include ACL; does not include attribute info.</li>
<li>Offset + Length refer to the identifier only.</li>
</ul>
</dd>
<dt>source.request.indexsource</dt>
<dd>
Returns a thorough view of the file structure.  Issued by <code>sourcekitd-test -req=index</code> and <code>sourcekitten index</code>.
<ul>
<li>Includes detailed references (for 'follow-symbol' type use).</li>
<li>File positions based on column + line numbers.</li>
<li>Includes USR, no declaration annotation, no XML doc comment.</li>
</ul>
</dd>
</dl>

So this is a bit of a mess. It looks like <code>structure</code> and
<code>cursorinfo</code> should fit together, and this is SourceKitten's main
strategy, but that leads to problems:
1. Due to `structure` bugs, subscripts and typealiases are missing.
   SourceKitten valiantly tries to deal with this by sending `cursorinfo`s
   after doc-comments that don't crop up in the `structure`.  This means
   undocumented decls are omitted, and there is no ACL or attribute info for 
   the documented ones.
2. The rich decode of `@available` is ... unavailable.

Let's try to figure out why these problems exist.  Hope to add the missing
types to `structure` but am somewhat resigned to having to add an additional
`docinfo` query to SourceKitten in order to access the `@available` stuff.

## Code Investigation

Briefly:
* SourceKit requests start in swift/tools/SourceKit/tools/sourcekitd/lib/API/Requests.cpp
    * All string identifiers hidden away in swift/tools/SourceKit/include/SourceKit/Core/ProtocolUIDs.def
    * This is really just a protocol driver layer
* Implementations mostly in swift/tools/SourceKit/lib/SwiftLang
* Further parts swift/lib/IDE

### Queries

[Structure - source.request.editor.open](sourcekit_editoropen.html)  
[CursorInfo - source.request.cursorinfo](sourcekit_cursorinfo.html)  
[DocInfo - source.request.docinfo](sourcekit_docinfo.html)  

These APIs are almost entirely independent, each having their own AST-traversal
logic, their own intermediate data structure, and their own serialization logic.
Indexing is separate again. The `structure` returned by `editoropen` uses an AST
that has only been parsed: the others use a type-checked 'sema'ed AST.

Would be fascinating to know the development history here -- feels like the
classic "user interfaces are easy" plus drip-drip of requirements antipatterns.

The fields returned form a classic three-way venn diagram:

| Key | editoropen | cursorinfo | docinfo |
|---|---|---|---|
| kind[1] | Y | Y | Y |
| offset[2] | Y | Y | Y |
| length[2] | Y | Y | Y |
| name | Y | Y | Y |
| nameoffset | Y | | |
| namelength | Y | | |
| bodyoffset | Y | | |
| bodylength | Y | | |
| usr | | Y | Y |
| accesslevel | Y | | |
| setteraccesslevel | Y | | |
| typename | Y[3] | Y | |
| runtime_name | Y[4] | | |
| selector_name | Y[5] | | |
| attributes | Y[6] | | Y[7] |
| full_as_xml | | Y | Y[8] |
| annotated_decl | | Y | |
| fully_annotated_decl | | Y | Y |
| groupname, modulename | | Y | |
| localization_key | | Y | Y |
| filepath | | Y | |
| parent_loc | | Y | |
| is_system | | Y | |
| typeusr | | Y | |
| containertypeusr | | Y | |
| unavailable | | | Y |
| deprecated | | | Y |
| optional | | | Y |
| generic_requirements | | | Y |

Then more complex:

| Concept | editoropen | cursorinfo | docinfo |
|---|---|---|---|
| Inherited classes | inheritedtypes - names | | inherits - name/usr/kind
| Conformed protocols | inheritedtypes - names || conforms - name/usr/kind
| Overridden class members | | overrides - usr | inherits - name/usr/kind
| Overridden proto members | | overrides - usr | conforms - name/usr/kind
| Extended types | | | extends - name/usr/kind
| Overloaded functions | | related_decls |
| Generic parameters | | | generic_params - names; `generic_type_param` entity - name / usr/declaration
|---|---|---|---|

Notes:
1. Kind space is different
2. `editoropen` and `docinfo` disagree on parameters.  `editoropen` generates
   a `decl.var.parameter` for the argumen, with name correct, offset pointing to
   the name of the arg, and length covering the entire arg declaration.  But
   `docinfo` generates a `decl.var.local` with the correct name but with
   offset and length pointing at the *type* of the arg.  For `cursorinfo` the
   input `offset` must be of the identifier in question, ie. `nameoffset` from
   `editoropen`.
3. Only for parameters
4. Only for top-level non-generic classes and protocols.
5. Only for `@IBAction`s!
6. Decl attribute names only, mixing up `@attribute`s with stuff like `override`
   that users do not think of as attributes.
7. `@available` only, all parameters decoded
8. Omits doc comments that should be inherited from protocol conformances into
   nominal types.

## Fixes

[Fixing Structure/source.request.editor.open](sourcekit_fixing_structure.md)
