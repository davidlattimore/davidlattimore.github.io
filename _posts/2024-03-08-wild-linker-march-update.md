---
layout: default
title:  "Wild linker - March update"
date:   2024-03-19 00:00:00 +1100
categories: posts
---
A month ago, I posted some [investigations into Rust warm build
times](/posts/2024/02/05/speeding-up-the-rust-edit-build-run-cycle.html) and I mentioned at the end
that I was writing a linker called [Wild](https://github.com/davidlattimore/wild) in Rust.

This post is an update with some of the work since then. I'll go into a little bit of technical
detail about some of the changes I've made, but I'll try to keep it accessible.

## eh_frame support

These are necessary in order for stack unwinding to work. Without eh_frames, panics and backtraces
won't work. Generally each function in your binary will have an entry in eh_frame that provides
information about the stack for any instruction in the function. This is used both for the simple
case of printing a stack trace and the more complicated case of unwinding, which requires running
`Drop` for variables in each stack frame.

eh_frames require a few special things from the linker.

* The linker needs to build an index of the frame entries so that the runtime can do a binary search
  to find the entry for a particular function.
* In order for the index to be valid and not confuse the runtime, it's a good idea to drop any frame
  entries for functions that we've decided not to link. i.e. we don't want frame entries for dead
  code that the linker removed. That means that eh_frame handling needs to be integrated into the
  garbage collection pass of the linker, which figures out which sections of the input files to link
  and which to discard.

Now that Wild supports eh_frames, panic and backtraces work in Rust binaries that we produce.

## ifuncs

I had actually already implemented ifuncs when I made my last post, but they're an interesting
feature and I mention them later in the post, so I thought I'd cover them here.

An ifunc is a type of function where the implementation is resolved at runtime during program
startup. When libc starts, it goes through a list of ifuncs created by the linker and calls a
"resolver function" for each ifunc. The resolver function gets passed information such as what CPU
features are supported. It can then return the most optimal implementation for the current CPU. libc
then stores the returned pointer for use whenever the function is called during program execution.

This is used for functions like memcpy which have faster implementations for some CPUs.

ifuncs are a GNU extension to ELF, but glibc uses them, so if you want to link against glibc, you
need to support them.

## Dynamic relocation

Earlier versions of Wild only supported non-relocatable static binaries. This meant that the binary
had to be loaded at a fixed address that was decided by the linker. This likely isn't a problem for
development, however as a step towards implementing dynamic linking, I thought it'd be good to add
support for statically linked, relocatable executables.

When a linker runs, it takes the sections of the input files and decides where to put them in the
output file. It then needs to apply the relocations required by each section. A relocation is an
instruction written by the compiler to tell the linker to put an address of a section or a symbol at
a particular location. There are many different types of relocations (43 in the ELF 64 bit spec).
Some of these relocations are relative, which means that we write the relative offset to the
requested symbol or section. However, some things like vtables (used when you have trait objects)
require absolute addresses.

When an absolute address is required and we're writing a relocatable binary, the linker doesn't yet
know the address of the target. This means that it needs to write a dynamic relocation, which is an
instruction to the runtime. These dynamic relocations are applied at startup. Sometimes there can be
quite a lot of them. e.g. rustc has over 32K dynamic relocations which get processed each time rustc
starts.

As an aside, this made me think a bit about whether it would be possible to have vtables that
contain only relative pointers. Indeed, there has been some [research done on
this](https://llvm.org/devmtg/2021-11/slides/2021-RelativeVTablesinC.pdf) in relation to LLVM.

Most of the work for dynamic relocation was finding all the places where we needed to apply it. One
place that came as a bit of a surprise was the relocations for ifuncs, which I mentioned above.
These ifunc relocations need to contain the absolute address of the resolver function and the
absolute address where the relocation is to be applied. This meant that I needed to apply dynamic
relocations to the ifunc relocation. Applying a relocation to a relocation felt a bit meta. It also
made me realise how many layers of complexity Linux ELF (including GNU extensions) has built up over
the years. Ideally the ifunc relocations would contain only relative pointers, then we wouldn't need
to apply dynamic relocations to them.

Another chunk of work for dynamic relocation was implementing some additional linker
micro-optimisations. These are not LTO, which we don't yet support, they're small optimisations that
the linker does when it's applying relocations to some compiled code. These optimisations are
supposed to be optional for the linker to implement, but as we'll see, this is not always the case.

To explain this example, I first need to explain the global offset table (GOT). It's a table, built
by the linker, that contains pointers to some functions. For functions that the compiler wants to
allow to be overridden at runtime by dynamic libraries, the compiler, instead of calling a function
directly, requests that the linker create a GOT entry containing a pointer to the function.

Because the global offset table contains the absolute addresses of functions, if we're building a
relocatable executable, we need to apply dynamic relocations to all the entries it contains.

Sometimes the linker knows, perhaps because we're statically linking, that the function cannot be
overridden at runtime, meaning that it's free to bypass the GOT entry by transforming indirect calls
to the function into direct calls to that same function. This means that we no longer need an entry
in the global offset table and, perhaps more importantly, we no longer need to apply a dynamic
relocation.

The following assembly code contains a call instruction (which calls a function). That call
instruction has a relocation put there by the compiler that requests a GOT entry for the function
`__libc_start_main` function.

```nasm
  1f:   ff 15 00 00 00 00       call   *0x0(%rip)
                        21: R_X86_64_GOTPCRELX  __libc_start_main-0x4
```

What's interesting here is that this assembly code is in the function `_start` provided by libc,
which is where the program starts executing. It's trying to use the GOT, but dynamic relocations
haven't been applied to the GOT yet, because that happens somewhere in `__libc_start_main` which is
the function we're trying to call! That means that if we take the compiler's instructions as
written, our binary will segfault.

So the "optional" optimisation to eliminate the GOT entry turns out to be not so optional after all.
This is perhaps not surprising. Once GNU ld implemented optimisations like this, libc would have at
some point come to depend on it.

## Comment section

I wanted a reliable way to identify whether Wild was used to produce a particular binary file. The
way this is generally done is by adding a text comment into the `.comment` section in the binary.
The compiler (e.g.) rustc also puts a comment stating its version.

You can view the comments in a binary using `readelf` as follows:

```sh
readelf -p .comment my-binary
```

```
String dump of section '.comment':
  [     1]  GCC: (GNU) 9.4.0
  [    12]  rustc version 1.76.0 (07dca489a 2024-02-04)
  [    3e]  GCC: (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0
  [    69]  Linker: Wild version 0.1.0
```

## String merging

This work was mostly motivated by the work on the comment section, since each object produced by the
compiler has a separate copy of the comment. If we didn't do string merging, then our output binary
would have the same comment repeated many times.

String merging is activated when a section of an input file is marked with the `merge` and `strings`
flags.

You can see these flags with `readelf` by running:

```sh
readelf --sections my-binary
```

The comment section might look like:

```
  [16] .comment          PROGBITS         0000000000000000  00003264
       000000000000002d  0000000000000001  MS       0     0     1
```

Here `MS` are the flags for the `.comment` section and mean `merge` and `strings`.

Some linkers like GNU ld take string merging to an extreme. For example if they find a string
"foobar" and another string with the same suffix such as "bar" they'll include only "foobar" then
make references to "bar" point to the last three bytes of "foobar". This is a neat optimisation, but
since we're trying to write a fast linker, we don't do this. Other fast linkers like `lld` and
`mold` also don't do this - at least not by default, so we're in good company.

## Performance

Wild doesn't support all the same features as the more mature linkers, so when comparing, it's
important to try to make the comparison as fair as possible by only exercising features in the other
linkers that Wild also supports.

* Linking debug info is very time consuming, so we pass `--strip-debug` to all linkers so that they
  don't need to spend time linking debug info.
* Wild doesn't yet support dynamic linking, so we ensure that we're building a statically linked
  executable.
* Wild doesn't yet support build IDs, so we omit those flags from the link command.

The program that I used to benchmark build speeds was the same one I used in my previous blog post,
which is a Rust program with a moderate number of dependencies.

| Linker        | Time (ms) | ± Standard deviation (ms) |  File size (MiB)
|---------------|-----------|---------------------------|-----------------
| GNU ld (2.38) | 12413     | 76                        |  80.3
| gold (1.16)   | 3418      | 58                        |  83.3
| lld (15)      | 898.9     | 8.1                       |  84.8
| mold (2.40)   | 425.8     | 6.9                       |  81.1
| wild (0.2.0)  | 346.6     | 7.6                       |  80.9

The size column is mostly there to check that none of the linkers are doing significantly more or
less work than the others. The larger size for Gold and LLD is mostly because they have twice as
much data in their `.gcc_except_table` sections compared with the other linkers. `.gcc_except_table`
is used by `.eh_frame` which I described above and contains stuff required for `Drop` to work when
Rust unwinds. I'm not sure why those two linkers both have exactly 3132732 bytes in this section,
while the other three have exactly 1459724. My guess would be that their garbage collection
algorithms don't extend to these sections.

For the multithreaded linkers (lld, mold and wild), it's also interesting to look at total CPU time
consumed. If we allow as many threads as there are CPUs, then hyperthreading inflates the CPU usage
time when two threads are running on the same core. So we restrict the linkers to 4 threads, which
is how many CPU cores my laptop has.

We also set `--no-fork` for Mold, otherwise it forks a subprocess to do the work, which means we
don't get to observe the CPU usage of the process doing the actual work.

| Linker   | Wall time (ms) | CPU time (ms) | Parallelism
|----------|----------------|---------------|------------
| lld      | 903.9          | 1090.3        | 1.21
| mold     | 510.3          | 1724.3        | 3.38
| wild     | 425.8          | 1170.7        | 2.75

Currently Wild has slightly higher CPU usage compared to LLD, but considerably lower than Mold's. It
has more parallelism than LLD's, but not as much as Mold's. I've got ideas for improving
performance, but I'm trying to avoid working on those until I've improved correctness and
completeness.

I want to stress that this is only one benchmark. Many unknowns remain:

* Will the results be significantly different for other benchmarks?
* How will Wild scale up when linking much larger binaries and/or on systems with many CPU cores?
* Will implementing the missing features require changes to Wild's design that might slow it down?

All we can really conclude from this benchmark is that Wild is currently reasonably efficient at
non-incremental linking and reasonable at taking advantage of a few threads. I don't think that
adding the missing features should change this benchmark significantly. i.e. adding support for
debug info really shouldn't change our speed when linking with no debug info. I can't be sure
however until I implement these missing features.

The goal of Wild isn't to be the fastest at a cold build, but rather to be very fast at incremental
linking, once that's implemented. But it's nice to see that it's currently faring pretty well at
linking from scratch.

## Next steps

The next big things that I'm looking to implement are:

* Building a tool to identify significant differences in output files
* Dynamic linking
* Debug info

We're already part way toward dynamic linking by having implemented relocatable binaries. I think
mostly what remains is tracking which symbols need to be resolved from shared objects (.so files) at
runtime, writing dynamic relocations for them and adding some extra info into the dynamic header to
say which shared objects are required.

Another thing likely to be needed is to correctly handle symbol visibility. While we're statically
linking, we know that symbols can't be overridden at runtime, since our static binary is all there
is. That means that we can apply micro-optimisations like the one I talked about above to remove GOT
entries. Once we're dynamically linking, I'll need to only apply such optimisations for symbols that
have been marked as allowing it.

Debug info I suspect will be somewhat similar to eh_frames, but with substantially more tables and
relationships between those tables.

## Funding

I'm currently doing the live-off-savings-and-github-sponsors thing. Whether I can continue to do
this long-term will depend on how much sponsorship I can attract. If you or your company appreciate
this work and would like to see it continue, please consider [sponsoring
me](https://github.com/sponsors/davidlattimore).
