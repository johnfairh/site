---
title: Making libruby available through SwiftPM, redistributably
---
# {{ page.title }}

*Feb 2018*

Problem: make the Ruby C API visible to Swift using SPM in a way that is useful
for other users.

# The Ruby C API installation

(From a place of macOS by default with Linux as an exception...)

## Headers

Ruby can be installed in lots of different places.
I happen to have four versions installed under `rbenv` and one system version
from Apple.

My `RBENV_ROOT` (although that env var is not set) is `~/.rbenv`.  Other
people on the internet have a system-wide installation under `/usr/local`.

Other Ruby version managers exit. Presumably these do not install into paths
involving `rbenv`.

On Linux & macOS Homebrew, the headers end up in `prefix/ruby-x.y.z`.  Debian
puts the arch header directory under in `prefix/arch/ruby-x.y.z`.

All this means is that the Ruby umbrella header file and include directories
can be in various places depending on how the user's machine is set up and
what version of Ruby they want to run against.

For the system Ruby, the header files are shipped as part of Xcode rather than
macOS, in `$(xcrun --show-sdk-path --sdk macosx)/System/Library/Frameworks/Ruby.framework/Headers`.  Apple have munged the arch-specific include dir
(`ruby/config.h`) into the main one.

And that `xcrun` could give different answers on different machines.

### Linux

On Linux, `#include <ruby.h>` fails to compile due to a missing dependency on
`<termios.h>` for `pid_t` or something.  Maybe this is due a difference between
`clang` used by SPM and `gcc` that Ruby probably expects.  Easy enough to
work around.

### Clang modules and headers

When a header is given in a module map, and that header includes others, Clang
is smart enough to look for them relative to that first header.  The native
macOS Ruby structure is:
```
include/ruby-r.m.f
    ruby.h
    ruby/
        ruby.h
        <other stuff>
    x86_64-darwin17/    <or whatever>
        ruby/
            config.h
```
The module map contains the higher-up `ruby.h`.  This means all of its includes
of the form `#include <ruby/foo.h>` get resolved by clang no problems.  But it
also does `#include <ruby/config.h>` which will not get resolved without passing
an explicit `-I` option.

Apple's distribution puts `config.h` into the regular directory which means no
`-I` is needed once the module map points to the right `ruby.h`.

## Libraries and dependencies

The system Ruby is shipped as `/usr/lib/libruby.2.3.0.dylib` with regular
symlinks so `-lruby` works.  `otool -L` tells us this depends in turn on
`/System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/libruby.2.3.0.dylib` which depends on Core Foundation.

The `ruby`s built by default with `rbenv install` provide only a static library.
My earlier `rbenv` installations have called this `libruby-static.a`; latest
`rbenv` has adopted a less risky naming scheme of `libruby.2.5.0-static.a`.

(This 'no shared lib' thing is a [6.5 year-old open issue in ruby-build](https://github.com/rbenv/ruby-build/issues/35) which provides the command to install a
 version with the shared library.)

Trial and error reveals that the static library depends on `Foundation` (which
is reassuring given the dylib does.)

So this is a step more complicated than the headers: not only the location but
also the name of the library + its link options can vary from machine to
machine.

Installing via macOS Homebrew leaves a `libruby.dylib` as well as the expected
symlinks in `prefix/lib`.  Linux `-dev` packages do not do this, they leave
just the expected versioned symlinks.  Debian packages put the libs in
`/usr/lib/x86_64-linux-gnu`.

### More pain with static libraries

When linking with a static library, `clang` produces warnings if any of the
object files were compiled with a different target than the current setting.

SwiftPM defaults to a target of something-something-10.10.  But `rbenv install`
defaults to "whatever the current macOS is" - meaning, for latest Ruby 2.5,
something-something-10.13 (High Sierra).

To avoid the warnings - let's assume they're there for a reason - we have to
tell SwiftPM to use the more recent target.

Linking against this static library also produces the heart-stopping:
```
(kernel) Failed to query kext info (MAC policy error 0x1).
Failed to read loaded kext info from kernel - (libkern/kext) internal error.
```
Luckily `swift build` still has exit code 0 and seems to work fine.  So we'll
ignore that.

## pkg-config

So, `pkg-config` to the rescue?  Well, sort of: Ruby from `rbenv` etc. does
provide a `.pc` file.  However it is stored alongside that version of Ruby and
not added to `$PKG_CONFIG_PATH`.

Once found, `pkg-config --cflags` works but `pkg-config --libs` is a bit
broken: it doesn't support `--static` properly and if the Ruby installation
doesn't *have* a dylib then it doesn't bother to mention a Ruby library at all.

Ruby from `rvm` provides and does not install an amusingly broken
`pkg-config --libs`.  Linux `ruby-dev` and macOS Homebrew packages properly
install a working pkgconfig file.

# SwiftPM system packages

Wrapping an existing library requires an entire package, can't just be a target
of something else.

## Module map

The module map has to contain the absolute location of the umbrella header,
and the name of the library to link with.

## Package.swift

Ideally we would set compiler flags here.  SwiftPM does not allow this.  It does
allow a pkgconfig filename to be given which SwiftPM itself will parse and use
to find compiler/linker flags.  **However** if *any* of these flags are not on
a "whitelist" (basically -I / -F / -L/l) then the are *all* thrown away and the
pc file is ignored.  This makes the feature almost entirely useless.  I can
imagine the line of discussion that led here but still...

SwiftPM has a complex list of places to look for .pc files.  None of these
include anything useful like 'the package root directory' or 'the current
directory'.

# Compiler default paths

Clang default lib paths, from `clang -v -Xlinker` are (today):
* /usr/local/lib
* Xcode usr/lib
* Xcode SDK frameworks

Now, `/usr/local/lib` is where homebrew puts its stuff by default, so there
is definite chance of a mixup here: doing `clang -lruby` will always grab the
`/usr/local/lib` version.

But be careful, `swiftc` is not `clang`.  Libraries mentioned in module maps
passed to `swiftc -fmodule-map-file` do not appear to resolve in `/usr/local`
ahead of the SDK -- observation not proof, mind.

# Summary

Choices seem to be:
1. Apple default: dylib in `/usr/lib`, headers in a single directory inside
   Xcode.
2. Unix prefix (Homebrew, Linux, etc.): dylib in `prefix/lib`, headers in
   `prefix/include/ruby-2.X.Y`.  Prefix may be more complicated on Linux with
   separate arch path part.
3. Rbenv or similar: standard Ruby installation somewhere in the filesystem,
   may have static and/or dynamic libs.

Case (2) may be symlinks from a case (3) install but regular users will not
know the origin.

Linux package install & macOS Homebrew actually both provide and install a
decent `pkgconfig` file.  Nothing else does.  But, due to harmless linker flags
SwiftPM will refuse to use these `pkgconfig` files.  Because native Ruby splits
its include files over two directories, the **only** 'just works' option is to
target the macOS system default Ruby with the single-directory Xcode headers.

All other configs + platforms will need at minimum `-Xcc -I` passed to deal with
the `ruby/config.h` directory.

So we will:
1. Ship a default config using the Apple defaults.
2. Ship a script designed to be run from a `swift package edit` session that
   will update the various files for a Ruby of the user's choice including
   creation of a package-config file if necessary to wrap up the compile + link
   arguments.

This will have to be done for all down-stream consumers, eg. if `TMLRuby` is
a library package that uses `CRuby` and `TopazFancy` is an executable package
using `TMLRuby` then building `TopazFancy` will require this Ruby config
bootstrap step to use a non-system default Ruby (or work at all on Linux).

Never thought I would miss autoconf.
