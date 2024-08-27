---
layout: default
title:  "Rust dylib rabbit holes"
date:   2024-08-28 00:00:00 +1000
categories: posts
---
**I'll be giving a talk about this at the Rust Sydney meetup later today. It'll be about 9am UTC,
although it could be early or late depending on other talks. The livestream link will be posted to
Discord at https://discord.gg/pW35BNSBeV (see syd-2024-aug-28 channel) where you can also ask
questions. I hope that the talk will also be posted to YouTube later, although it's not me that does
that, so I can't make any promises.**

Bevy is a popular game engine for Rust. It's pretty large and compilation times can be an issue. To
help with this, Bevy provides an optional feature that when enabled, compiles most of Bevy as a
dynamic library. This allows for faster iteration as you don't need to relink all the Bevy internals
each time you rebuild.

```sh
cargo run --features bevy/dynamic_linking
```

I was experimenting with this from the perspective of testing and profiling the linker I'm writing,
Wild (see [previous posts](https://davidlattimore.github.io/)).

With that in mind, I was mostly looking at (a) how long it takes to link and (b) how well the
resulting .so file works.

Initially, I was only looking at debug builds. To speed up the build, I turned off debug info.

```toml
[profile.dev]
debug = false
```

So this was perhaps more accurately described as a non-optimised build. Having optimisations off
should make the build faster right? Probably it does, but it doesn't necessarily make linking
faster. Here's the times for linking this shared object:

| Linker        | Time (ms)
|---------------|----------
| lld (18)      | 1975
| mold (2.32.1) | 1763
| wild          | 895

I'll not include GNU ld because it's more than 10 seconds, making it painful to benchmark.

If we now set `opt-level = 2`, then the link time drops quite dramatically:

| Linker        | Time (ms)
|---------------|----------
| lld (18)      | 545
| mold (2.32.1) | 287
| wild          | 183

I sometimes wonder if Rust (or more accurately Cargo) needs a third default profile "fastbuild" that
doesn't have debug info and is optimised for building fast. I'm sure there are a bunch of tradeoffs
between compilation speed and debuggability that currently favour the latter. I bet there are
optimisations that, if applied, would speed up the build, especially a warm build, but which are
disabled in debug builds because they might make it harder to use a debugger on the code.

But what really drew my attention with the non-optimised build was what it's getting the linker to
do. We're creating a shared object (.so file on Linux). Rustc gives instructions to the linker to
tell it which symbols need to be exported. If a symbol is exported from the shared object, then an
executable or another shared object that depends on our shared object can make use of those symbols.
If a symbol isn't exported then it cannot be directly referenced from outside the shared object.

In order to control which symbols get exported from the shared object, the linker is passed a
version script which specifies which symbols should be global and then downgrades the rest to local.

```
{
  global:
    rust_metadata_bevy_dylib_2f311168f6c5d4f8;
    _ZN9hashbrown3set24HashSet$LT$T$C$S$C$A$GT$6insert17hcb8b576667efe889E;
    _ZN9hashbrown3set24HashSet$LT$T$C$S$C$A$GT$6remove17h53654c4e42de8b15E;
....

  local:
    *;
};
```

For a non-optimised build, this version script lists more than 300k symbols to export! Contrast this
with the optimised build, where it lists only 18k symbols. Looking into this a bit, the majority of
the extra symbols happen because non-optimised builds enable `-Z share-generics` by default. These
shared generics not only get exported from the crates that monomorphise them, they also get exported
from the dylib. The remainder of the extra symbols look to be functions that would have been inlined
in an optimised build. This seems somewhat surprising that a public function would be exported from
a dylib only if it didn't get inlined.

But let us for the moment assume that we actually for some reason need all 300k symbols to be
exported.

When a dynamically linked executable or a shared object gets loaded, the runtime can look up symbols
that are provided by other shared objects. On Linux, symbol lookups can either be eager, meaning
that they happen when the binary is loaded, or lazy, meaning that the symbol is only looked up when
the function is first called. For security reasons, lazy binding is less popular these days and rust
indeed sets linker flags to bind symbols at load time.

For shared objects produced by rustc, most of these non-lazy symbol lookups are done with `GLOB_DAT`
relocations. These relocations are instructions to the runtime to put the address of a symbol with a
particular name at a particular location in memory. For example, the following relocation says to
look up the symbol `__rust_alloc`, then put the address of that symbol at address `0x93ec698`.

```
00000000093ec698  0000000700000006 R_X86_64_GLOB_DAT      0000000000000000 __rust_alloc + 0
```

If we check how many `GLOB_DAT` relocations are in our bevy shared object, we get a bit of a
surprise.

```sh
readelf -W -r libbevy_dylib.so | grep GLOB_DAT | wc -l
291185
```

But `GLOB_DAT` is for resolving references to symbols that the shared object depends on, so why is
the number of outgoing references so similar to the number of symbols that the shared object
exports?

Indeed, it turns out that this isn't a coincidence. The majority of the symbols with `GLOB_DAT`
relocations are for symbols that are defined by the dylib itself.

But why would the dylib request runtime resolution of a symbol that it itself defines? Dynamic
linking on Linux allows symbols defined by shared objects to be overridden (also known as
"interposing"). One use-case for this is to override the allocator provided by libc in order to
perform runtime checks.

But we don't really want to be able to override all these symbols, we just want them to be exported
so that they can be used by our binary that uses the shared object. When the compiler builds an
object file on Linux, symbols can be local or global. Locals are only accessible within that codegen
unit, while globals can be referenced from other codegen units. Global symbols can then be further
restricted by setting their visibility, which affects how they'll be treated when dynamic linking.

| Binding | Visibility | Accessible from other codegen units? | Accessible from other dynamic objects? | Can be overridden?
|---------|------------|--------------------------------------|----------------------------------------|-------------------
| Local   |            | No                                   | No                                     | No
| Global  | Hidden     | Yes                                  | No                                     | No
| Global  | Protected  | Yes                                  | Yes                                    | No
| Global  | Default    | Yes                                  | Yes                                    | Yes

The key difference here is between default visibility and protected visibility. The latter means
that the symbol cannot be interposed (overridden). A default visibility symbol however can be
interposed, which means that if another shared object earlier in the load order, or the executable
itself defines a symbol with the same name, that will take precidence.

OK, so we just need to set all our symbols to protected. That way they'll be exported from the
shared object, but won't be permitted to be overridden.

I found the code in rustc that sets symbol visibility and prototyped changing it to set symbols to
have protected visibility unless the symbol was marked as `#[no_mangle]`. This worked and
drastically reduced the number of `GLOB_DAT` relocations. To test how much of a difference this
makes, I tried loading shared objects with and without this change.

* Default visibility: Shared object took about 150ms to load.
* Protected visibility: Shared object took about 5ms to load.

OK, that's great. At that point I thought I should look for existing issues related to this and
indeed found one. The creator of the cranelift backend for rustc, bjorn3 had also attempted to
change symbols to use protected visibility, but had hit issues when linking with GNU ld.

GNU ld complains that direct references to protected symbols cannot be used when building a shared
object. I tried GNU ld and got the same problem.

But let's think about this for a moment, why can't a shared object have direct references to a
protected symbol - it cannot be overridden, so it should be fine to reference it directly. Right?

To understand what GNU ld's objection is here, we need to look at how GCC compiles C code. We'll
start by looking at what it does with C code that references data.

```c
extern int my_value; // Likely from a header file

int main() {
    return my_value;
}
```

First, let's look at what the clang compiler does with this.

```nasm
   0:	48 8b 05 00 00 00 00 	mov    0x0(%rip),%rax        # 7 <main+0x7>
			3: R_X86_64_REX_GOTPCRELX	my_value-0x4
   7:	8b 00                	mov    (%rax),%eax
```

The first instruction is reading a pointer to our variable `my_value` from the GOT (global offset
table). The GOT is a table of pointers. These pointers are generally populated by the runtime at
startup to point to functions and variables that come from different shared objects.

The second instruction then loads the value from that pointer. This instruction sequence will work
fine even if the variable `my_value` ends up coming from a shared object.

If the variable ends up being statically linked into our binary, then the linker will transform this
assembly to:

```nasm
    1130:       48 8d 05 f1 2e 00 00    lea    0x2ef1(%rip),%rax        # 4028 <my_value>
    1137:       8b 00                   mov    (%rax),%eax
```

The `lea` instruction here is loading the relative address of our variable, which is now known at
link time. That means that there's no access to the global offset table.

Now, let's look at what GCC does:

```nasm
   4:	8b 05 00 00 00 00    	mov    0x0(%rip),%eax        # a <main+0xa>
			6: R_X86_64_PC32	my_value-0x4
```

It's using a PC32 relocation to access the variable `my_value`. This is a direct reference, which
will only work if the address of the variable is known at link time. i.e. this won't (or shouldn't
IMO) work if `my_value` comes from a shared object. If we add the flag `-fPIC` to gcc, then it
produces the same code as clang.

So we have a trade-off. The code to directly access a variable that gets statically linked into our
executable is shorter and presumably more efficient, but doesn't really work if the variable ends up
coming from a shared object. The code that does work for accessing the variable from a shared object
is slightly longer and a bit less efficient, although with the linker optimising away the access to
the global offset table, the efficiency difference is pretty small - however it remains longer than
the direct access code.

I said that the direct access approach doesn't work if the variable ends up coming from a shared
object. Unfortunately that's not entirely true. Linkers apply a horrible hack called
copy-relocations in order to make it work. When they encounter a direct access to a variable that's
defined by a shared object, they allocate space for that variable in BSS (a zero-initialised section
that doesn't take up space in the file on disk), then at runtime the bytes of the variable get
copied from the shared object that defined it into that space. That copy then overrides the
definition provided by the shared object.

![Diagram of a copy relocation](/images/protected/copy-relocation.svg)

But what if the symbol definition in the shared object has protected visibility? That means it can't
be overridden right? GCC chose to interpret "can't be overridden" as "can only be overridden by a
copy relocation".

For a shared object to work correctly when one of its symbols is overridden, there can't be direct
references to the symbol within the shared object. Here we get to a point of incompatibility between
the GCC / GNU ld world and the LLVM / LLD world.

If we now look at the code that each compiler produces for putting into a shared object, we can see
the other side of this difference. Here's our C code:

```c
__attribute__((visibility("protected")))
int my_value = 42;

int get_my_value(void) {
	return my_value;
}
```

We tell both compilers that we might put this into a shared object by compiling with `-fPIC`.

GCC produces the following assembly for the variable access.

```nasm
  19:	48 8b 05 00 00 00 00 	mov    0x0(%rip),%rax        # 20 <get_my_value+0xf>
			1c: R_X86_64_REX_GOTPCRELX	my_value-0x4
  20:	8b 00                	mov    (%rax),%eax
```

i.e. even though the variable is protected, it still accesses it via the GOT.

Clang however produces a more efficient direct access to the variable.

```nasm
  14:	8b 05 00 00 00 00    	mov    0x0(%rip),%eax        # 1a <get_my_value+0xa>
			16: R_X86_64_PC32	my_value-0x4
```

So when building an executable, GCC ends up directly referencing all symbols, even those that might
be protected symbols from a shared object. In order to make that work, it then uses indirect
references when building shared objects.

Clang does the opposite, using indirect references when building an executable, but then allows
direct references to protected symbols when building a shared object.

Mixing these two different, and incompatible models for when it's OK to directly reference something
can lead to problems. If your shared object is built by LLVM with direct access to protected
variables, then your main binary is built by GCC with direct access to all variables, we end up with
two separate copies of our variable. If the variable is mutable, then a change made in the main
binary won't be seen by the shared object and vice versa.

In order to protect against this, GNU ld detects direct access to protected variables and refuses to
link the shared object. But the shared object would have worked fine so long as it was only used by
a binary compiled with LLVM (Clang).

This can be seen if we try to compile a shared object with Clang and link it with GNU ld:

```sh
clang -fPIC -shared b.c -o libb.so
/usr/bin/ld: /tmp/b-09dfbd.o: relocation R_X86_64_PC32 against protected symbol `my_value` can not be used when making a shared object
/usr/bin/ld: final link failed: bad value
```

The examples so far used protected symbols that were data, not functions, however the same problem
occurs with functions. The only real difference is that the linker won't do a copy relocation for a
function, instead it synthesises a PLT entry (a small bit of machine code that jumps to the actual
function) then uses that to override the function definition in the shared object.

```c
__attribute__((visibility("protected")))
int f1(void) {
	return 42;
}

typedef int (*int_fn_t)(void);

int_fn_t get_f1_ptr2(void) {
	return &f1;
}
```

Compiling this code with clang causes a link failure with GNU ld:

```sh
clang -shared -fPIC x.c
/usr/bin/ld: /tmp/x-f06305.o: relocation R_X86_64_PC32 against protected symbol `f1` can not be used when making a shared object
/usr/bin/ld: final link failed: bad value
```

This might seem like it's just a trade-off between optimising code in the executable (GCC) or
optimising code in the shared object (LLD), in which case we should presumably pick to optimise the
executable, since for many uses that's where the bulk of the code lives. However picking this relies
on copy relocations, which are in my opinion, a hack. Like many hacks, they have a number of
downsides.

* They make the size of a variable part of its ABI. i.e. a shared object that defines a symbol now
  cannot change the size of that symbol without breaking the ABI.
* They require that the variable gets copied into writable memory. If a shared library embeds a
  large bit of data, say a 100MiB machine learning model and a copy relocation occurs, then at
  startup, that 100MiB will need to be copied. Furthermore, if there are several copies of the
  binary running, we're now going to have several independent copies of that 100MiB in RAM, whereas
  without a copy relocation, that 100MiB could be shared read-only between all the running
  processes.

The Rust compiler by default, uses LLVM to perform codegen. So when we change rustc to emit all
rust-mangled symbols with protected visibility, LLVM does the same as Clang above and emits direct
relocations to those symbols. This is fine provided we stick in the LLVM / LLD world, however if we
try to link using GNU ld, it gets rejected because it doesn't fit GNU's model of relying on copy
relocations for shared-object variable access from the main binary.

All of this came about because of GCC trying to simultaneously produce optimal code for executables
while not knowing at compile time whether a symbol might come from a shared object. On Windows, a
different path was taken. There, symbols that might come from a shared object (DLL on Windows) must
be annotated in the source code with `__declspec(dllimport)`. This allows the compiler to emit
optimised, direct-access instructions for all other symbols.

An alternative to annotating the source to indicate whether a symbol will come from a shared object
or be linked statically is to give the compiler access to the things we're going to link against so
that it can find where the definition comes from and make an appropriate decision. This would never
fly in the C world where it's expected that you can compile code with only access to the header
file, but in most modern languages like Rust, it's exactly what happens. When you compile a Rust
crate, the compiler is given access to its dependencies and so it knows whether those dependencies
are dylibs (shared objects) or rlibs (statically linked). This means that it's possible for the Rust
compiler to always make the optimal choice between a direct or an indirect reference because it has
all the information it needs to make that decision.

Using default visibility for symbols in shared objects affects not only load time for those shared
objects (150ms vs 5ms), but it also likely affects runtime performance, since all those variables
now need to be accessed via the global offset table, which means an extra pointer hop to get to the
data. There's a good chance it also prevents LLVM from making various optimisations, since by using
default visibility, we're effectively telling it that any of these variables or functions might be
swapped out for alternative definitions at runtime.

# Some good news

I do my development on a system that's based on Ubuntu 22.04, which has binutils version 2.38. Only
after writing most of this blog post did I think to try checking the behaviour of more recent
versions of GNU ld. As it turns out, binutils 2.40 fixes this problem in GNU ld.

Linking shared objects that have direct references to protected symbols is no longer an error.
Kudos to LLD maintainer, Maskray for making this change!

Instead, building an executable that would require a copy relocation for a protected symbol is now
an error.

```
/usr/bin/ld: /tmp/cciOjHc4.o: copy relocation against non-copyable protected symbol `my_value' in libb.so
collect2: error: ld returned 1 exit status
```

The error is now reported where it should be - when trying to build a binary that uses a shared
object with protected symbols and the compiler emitted direct references to those symbols. The fix
for that error is to compile the executable with `-fPIC` or switch to clang.

GCC maintains its behaviour of emitting direct relocations to variables and functions unless you
compile with `-fPIC`, but that's much less of a problem for Rust and other languages than the
previous GNU ld behaviour.

# Where to from here?

The fix to GNU ld is in binutils 2.40, which is in Ubuntu version 23.04 and later. However systems
built on 22.04 will be around for a while, so I don't think we can just switch to protected symbols
and cause link errors on those older systems.

Work has been done to use lld by default for linking on Linux. This is currently on nightly versions
of rustc. If we add a flag to enable emitting of protected symbols, then we could enable that flag
when lld is being used as the linker.

It's reasonable to ask, might creating shared objects with protected symbols cause those shared
objects to be unusable from programs compiled with GCC? I believe the answer is no, since we'd only
be making Rust mangled symbols as protected and they shouldn't be getting referenced from code
compiled by GCC.

# Further resources

* LLD maintainer, Maskray has an excellent [blog
  post](https://maskray.me/blog/2021-01-09-copy-relocations-canonical-plt-entries-and-protected)
  about this topic.
* Removal of problematic error from GNU ld. Not sure what to link to, but you can search for "x86:
  Make protected symbols local for -shared".
* [Disallow invalid relocation against protected symbol](https://sourceware.org/bugzilla/show_bug.cgi?id=28875)
* Related rustc issues:
    * [Use protected visibility by default on ELF platforms](https://github.com/rust-lang/rust/issues/105518)
    * [stop exporting every symbol](https://github.com/rust-lang/rust/issues/37530)
    * [linking staticlib files into shared libraries exports all of std::](https://github.com/rust-lang/rust/issues/33221)

# Thanks

Thanks to my [github sponsors](https://github.com/sponsors/davidlattimore). Your contributions help
to make it possible for me to continue to work on this kind of stuff rather than going and getting a
"real job".

* bearcove
* repi
* marxin
* bes
* Urgau
* jonhoo
* Kobzol
* coastalwhite
* mstange
* bcmyers
* Shnatsel
* Rafferty97
* joshtriplett
* teburd
* wezm
* davidcornu
* tommythorn
* flba-eb
* acshi
* teh
* yerke
* alexkirsz
* NobodyXu
* jplatte
* ymgyt
* Pratyush
* ethanmsl
* +2 anonymous

# Discussion threads

* [Reddit](https://www.reddit.com/r/rust/comments/1f2s7ot/rust_dylib_rabbit_holes/)
