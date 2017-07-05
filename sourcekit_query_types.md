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
Returns the 'structure' of the code.  Issued by `sourcekitd-test -req=structure`
and `sourcekitten structure`.
<ul>
<li>Omits (at least) subscript and typealias declarations.</li>
<li>The only SourceKit API to return accessibility fields.</li>
<li>No USR, declaration annotation.  No XML doc comment.</li>
<li>Lists attributes by names but drops all content.</li>
<li>(Requires `key.syntactic_only: 1` otherwise times out waiting for sema???)</li>
</ul>
</dd>
<dt>source.request.docinfo</dt>
<dd>
Intended to be a more 'useful' view of code?  Issued by
`sourcekitd-test -req=doc-info`.
<ul>
<li>Includes USR and fully-annotated declaration, XML doc comment.</li>
<li>Includes subscript and typealias declarations, although is still a bit
confused about subscripts.</li>
<li>Has a slightly different opinion about declaration lengths than
`structure` but offsets are in sync.</li>
<li>Has great decoding of `@available` attributes but ignores all(?) others.
</li>
</ul>
</dd>
<dt>source.request.cursorinfo</dt>
<dd>
Returns details of a single declaration.  SourceKitten uses this after
'structure' to fill in blanks.  Issued by `sourcekitd-test -req=cursor`.
<ul>
<li>Includes USR, annotated declarations, XML doc comment.</li>
<li>Does not include ACL; does not include attribute info.</li>
<li>Offset + Length refer to the identifier -- `nameoffset` and `namelength`
from `structure`.</li>
</ul>
</dd>
</dl>

So this is a bit of a mess. It looks like `structure` and `cursorinfo` should
fit together, and indeed this is SourceKitten's main strategy, but that leads
to the problems:
1. Subscripts & typealiases are missing.  SourceKitten valiantly tries to deal
   with this by sending `cursorinfo`s after doc-comments that don't crop up in
   the `structure`.  This means undocumented decls are omitted, and there is no
   ACL or attribute info for the documented ones.
2. The rich decode of `@available` is ... unavailable.

## Code Investigation

* outline code
* linky per query
