---
title: Swift Toolchain Build / Install / Test
---
# {{ page.title }}

*Valid June 2017 with swift:master*

## Setup

Instructions per [project readme](https://github.com/apple/swift).

Select the right Xcode version BEFORE doing any building, it gets cached
somewhere and if wrong the build fails a long way through.

## Build

*I gratefully followed [Russ Bishop's path](https://news.realm.io/news/slug-russ-bishop-contributing-open-source-swift-proposal/).*

The Swift build process is enormously complicated.  Like most build processes,
the default settings are at best imperfect for most users and figuring out what
you need is a rite of passage.

I want to be able to change code in the Swift compiler & tools, use those tools
on code, and debug them when they don't work, ideally using Xcode's `lldb`.

I want to be able to build my existing Swift projects using my changed
compiler.  These projects use both `xcodebuild` and `swift build`.

I want to do this just on Darwin for Darwin.

Settled on the following options to `build-script`:
<dl>
<dt>--release</dt>
<dd>To make the build as fast as possible.</dd>
<dt>--llbuild --swiftpm</dt>
<dd>To get Swift PM built so I can test that way.  <code>llbuild</code> is the
back-end for Swift PM.</dd>
</dl>

Things I tried and discarded:
<dl>
<dt>--debug-swift</dt>
<dd>Makes the build way too slow (specifically building the Swift code using
the freshly built swiftc).  Need to investigate further here, must be
possible to rebuild the compiler in debug mode.</dd>
<dt>--xcode</dt>
<dd>Besides <i>creating an Xcode project for Swift</i> this uses
<code>xcodebuild</code> instead of <code>ninja</code> to run the build.  I saw
one hang during build; the eventual project crashed Xcode twice while indexing;
most people seem to use ninja.
</dd>
</dl>

Wrap these options up in a preset called `jfdev` in
`~/.swift-build-presets`:
```
[preset: jfdev]
release
llbuild
swiftpm
build-subdir=jfdev
```
That last line puts the build products in `swift-source/build/jfdev` just in
case the preset name doesn't do that automatically.

Now I can build with `build-script --preset=jfdev`.

Build from scratch takes 50 minutes on my elderly 2011 Macbook Pro.  Adding
`--debug-swift` bumps that up to over four hours.  Big nope there.

## Incremental Build

Default dependencies are conservative meaning running the build script after
code edits can take a while.

Use `ninja <target>` from `swift-source/build/jfdev/swift-macosx-x86_64` to
build just that target like good ol' `make` used to, using the build settings
established by the most recent buildscript run in the tree.

For example if I am fixing SourceKit things and using `sourcekitd-test` to run
tests then `ninja sourcekitd-test` does the minimal to rebuild it without, for
example, rebuilding parts of the standard library.

## Toolchain

*I gratefully followed [Sam Symon's path](https://samsymons.com/blog/exploring-swift-part-2-installing-custom-toolchains/).*

Here, a *toolchain* is the collection of useful build products bundled up in a
way that Xcode understands.

The `utils/build-toolchain` script does an entire rebuild and tests.  It's
probably what the Apple Jenkins uses to produce the nightly toolchains.

Sam mentions a community script that assembles a toolchain from an existing
build directory.  I use [@CodaFi's version](https://gist.github.com/CodaFi/e5a72d8c08bc4bc5df577ef18b3ac130).
This is one of those out-of-process things that infuriates build teams: I'm
sure this can be done using `build-script` options but life is too short.

This does leave `lldb` not working but that's fine: I used Xcode's `clang` to
build the swift compiler so should use Xcode's `lldb` to debug it.

Now I can use my version of the tools:
```shell
; TOOLCHAINS=swift-dev swift --version
Swift version 4.0-dev (LLVM 2b37177ffc, Clang cc5d6deb6e, Swift e7a0ba8433)
Target: x86_64-apple-macosx10.9
```

## Testing

Simplest way to run the compiler unit tests is via `ninja` directly,
`ninja check-swift` from `build/preset/swift-macosx-x86_64`.  About 15m.

See [the swift repo testing document](https://github.com/apple/swift/blob/master/docs/Testing.md)
for other target names.

Stuff to run `lit` directly to run just one test (file of tests), adapted from
Russ's notes, in `.bashrc`:
```shell
SWIFTSRC=~/project/swift-source
LIT=${SWIFTSRC}/llvm/utils/lit/lit.py
LITCFGDIR=${SWIFTSRC}/build/jfdev/swift-macosx-x86_64/test-macosx-x86_64
swiftlit() {
    ${LIT} --param swift_site_config=${LITCFGDIR}/lit.site.cfg $*
}
```
Then `swiftlit -sv rdar_31758709.swift` for example, in the directory where
that file is.
