---
layout: default
title:  "Speeding up rustc by being lazy"
date:   2024-06-06 00:00:00 +1100
categories: posts
---
I’ve been busy working on the Wild linker (see [previous posts](/)), but wanted to divert for a moment to
look at some other compilation speed things that I’ve been thinking about. This post discusses
various thoughts about moving Rust codegen, monomorphisation and inlining later in compilation and
some of the ways this might reduce both from-scratch and incremental build times.

# Dead code

Dead code is code that gets compiled, but isn’t needed for the final binary. This might come from
crates in our dependency tree where we’re only using part of the crate. It might also be from impls
that we’re not using - e.g. lots of Debug and Clone impls that aren’t actually used. The amount of
dead code that we compile varies quite a bit by crate.

In order to assess how much code is getting compiled then discarded during linking, I [added
support](https://github.com/davidlattimore/wild/blob/main/wild_lib/src/gc_stats.rs) to the Wild
linker to print garbage collection statistics. If I run this on ripgrep, which has a pretty lean and
well-tuned build, we find that 17% of the executable code compiled is discarded.

For a less well-tuned binary, let’s pick on one of my own crates, the evcxr REPL. It shows that 35%
of compiled code is discarded by the linker.

There’s already been work done in Rustc to support MIR-only rlibs. This would defer codegen to later
in compilation. A lot of that work has been motivated by wanting to support compiling libstd with
different options. Depending on how it’s done, we may be able to take advantage of it to make
codegen happen on-demand. If codegen is deferred until link time, we know what is and isn’t
referenced. e.g. we can start from main and see what is referenced. We can then perform codegen only
for those functions that are referenced.

# Repeated monomorphisations

Another source of wastage is duplicate monomorphisations. Generic code, such as
`std::Vec::<T>::push` can’t be compiled to native code until the type parameter T is substituted.
This means that it happens when building the crate that calls the function. But there could be
multiple crates or codegen units that make use of the same monomorphisation. Repeating codegen for
each of them is wasteful.

I did an investigation into duplicate functions by [creating a
tool](https://github.com/davidlattimore/duplicate-function-checker) that determines what percentage
of the executable bytes in a binary are excess due to duplicate functions. For many build
configurations, about 5-10% of the machine code going into your executable is likely excess copies
of duplicated functions and most of that is due to repeating the same monomorphisation. You can read
more in the tool’s README.

This is not only wasteful of compilation time, but also binary size. For release builds, various
options such as setting `codegen-units=1` and fat LTO reduce this duplication, however these options
also hurt build times, so we need another solution.

There are two different sources of repeated monomorphisations. The first between codegen units
within a crate. This seems to mostly be an issue for release builds because the monomorphisations
are put into codegen units they’re referenced from, in case LLVM wants to inline them.

The second source of repeated monomorphisations is between crates. If multiple crates all need the
same monomorphisations, then each crate produces it. These duplicates happen in both release and
debug builds.

When compiling C++ code, GCC and Clang emit such monomorphisations as weak symbols rather than local
symbols like rustc does. This lets the linker deduplicate them. This might be an option for reducing
binary sizes, although it’s complicated by Rust’s use of the archive format for rlibs, since if the
only symbols referenced in an archive entry are weak symbols, then the archive entry won’t be
loaded. bjorn3 points out that this could be fixed by passing `--whole-archive` to the linker. Like
setting `codegen-units=1`, this only helps the binary size-problem, not the wasted-compilation-time
problem.

Some work on this problem has already been done in the form of the unstable flag `-Zshare-generics`
which is on by default for non-optimised builds. This does reduce the number of duplicate
monomorphisations, but there’s still plenty of duplicates from different crates remaining.

Duplicates originating from the same monomorphisation in different crates are somewhat tricky to
solve, but one possibility to do something similar what the proposal above for dead code. i.e. defer
monomorphisation to link time. Doing this would mean that we could create just one copy of each
monomorphised function.

# Recompiling dependents on implementation changes

Another source of wastage happens when you have several crates and you’re making changes to a
library crate, then rebuilding some binary that depends on the library crate that you edited.
Currently cargo rebuilds all crates in the dependency tree between the crate that you edited and the
binary crate you’re building.

![Diagram of several crates in a workspace](/images/lazy/crate-graph.svg)

In the diagram above, if `A` is our binary (or a test crate) and we’re making edits to the
implementation of a function in `F`, say adding and removing print statements, then each time we
make a change, rustc needs to be invoked on all the crates with the dashed outlines. However
ideally, it should be possible to just recompile `F`, then relink `A`.

Currently when the rust compiler compiles a library crate, it emits an rmeta file, then later emits
an rlib containing the results of codegen. The rmeta file for a build (as opposed to a check)
currently includes the MIR of all the functions in the crate. This is currently necessary, since the
dependent crates might want to inline some of the functions.

If we’re delaying codegen to link time, then we can also delay inlining. This means that we don’t
need the MIR in order to compile the dependent crates. This would give us two advantages:

* Pipelined compilation can work better, since we don’t need to wait for the MIR to be ready before
  the dependent crates can be built.
* We don’t need to rerun rustc on the dependent crates when the rlibs change, only when the rmeta
  changes. That means that if you edit the implementation of a function in one of your library
  crates, you only need to rerun rustc for that one library crate and then relink. During relinking,
  any functions that changed as well as any functions that inlined changed functions would go
  through codegen.

# Parallelism

When doing work in parallel with multiple threads or processes, if some, or one unit of work
finishes significantly later than the rest, it can slow things down because we have CPU cores
sitting idle with nothing useful to do. I’ll call these late finishers “stragglers”.

Currently in the Rust compiler, codegen of one crate can happen at the same time as earlier
compilation stages of another crate. By deferring codegen until we’re building the final binary, we
introduce an extra wait-point where we can potentially get stragglers.

One mitigation that we already get with the changes proposed above, is that a normal build becomes
more like a `cargo check` in terms of pipelining. Rather than emitting .rmeta files containing MIR,
the compiler emits .rmeta files without MIR. This means that dependent crates can start being
compiled earlier because they don’t need to wait for the MIR of their dependencies.

However we still need to wait for the MIR for the last crate(s) to finish being written before we
can start codegen. One potential for increased parallelism here is that rustc could make the MIR for
a crate available before it has finished checking the crate. The compiler stages might look
something like this:

* Parse files, do everything that’s required to write a .rmeta file containing only what’s needed to
  check dependent crates. i.e. emit interface information, type information, exported macros etc.
  Once this finishes, dependent crates can start being compiled. This is similar to what currently
  happens with a cargo check. Emit MIR-only rlib. Once this finishes for all crates needed by a
  binary, codegen and linking of that binary can begin.
* Complete remaining error-checking of the crate. Cargo would wait for this to complete, but other
  steps including codegen and linking can run concurrently with this.

So this is a form of pipelined building, similar to the pipelined building that cargo and rustc currently do. For comparison, this is what currently happens during a build:

* Do everything that’s required to write a .rmeta file. Unlike above, this contains MIR, since
  subsequent crates might need the MIR in order to inline functions during codegen. Once this is
  completed, subsequent crates can start building.
* Codegen crate. Once this is completed, the final binary can be built.

# Finer-grained codegen units

One way to reduce stragglers is by having smaller units of work. Currently the Rust compiler is a
bit limited as to how small it can make codegen units. Some of the things that limit the Rust
compiler are affected by changes proposed above.

My ideal would be if we could codegen each function separately. That maximises parallelism and also
means that when doing incremental compilation we can avoid the need to repeat codegen for other
functions that just happened to be in the same codegen unit on the previous build. However we’d need
to make sure we’re not repeating any work in multiple codegen units.

If this were done today, one source of repeated work would be that any generic functions called by
the function that we were going to codegen might, depending on optimisation level, be included too.
Above, this post proposed that if we’re not inlining a generic function, that we codegen it only
once and make it global rather than local. That would allow us to not include it together with the
function that we’re compiling.

Apparently the cranelift backend already does codegen for each function independently.

Writing a separate object file for each function is unlikely to be practical or efficient. There are
potential limits on how many arguments can be passed to the linker. The linker also might not be
optimised for this. Having one function per object within an archive might be a possibility,
although experimentation would be needed to see how well different linkers handled that. The
alternative would be to pack multiple function definitions into a single object file even though
they went through codegen separately.

# Linker integration?

One option for deferring codegen would be to integrate codegen into the linker. This could take the
form of building a linker into rustc and then using rustc as the linker.

An alternative would be to do codegen just prior to linking.

Integrating a linker into rustc would have some advantages:

* The linker is already doing a graph traversal, taking advantage of that avoids the need to do a
  separate graph traversal in the compiler.
* If you have a mix of code from Rust and other languages (e.g. C or C++), then the linker has a
  view of all of this. If doing the graph traversal without the help of the linker, we’d need to
  assume that any function that could be called from another language is called.
* Caching is probably easier with tighter linker integration, since the linker can read entries
  directly from a cache and we’re not constrained to putting everything in object files.

However, the main disadvantage of such tight linker integration is that we then don’t get all the
benefits of this work unless we’re using the integrated linker. My linker, Wild, is still a way off
being ready for general use on Linux and I haven’t even started to look at porting to other
platforms. So I think it’s important to try to do deferred codegen without integrating the linker.

Dong codegen just prior to linking could be done as follows:

* Compile binary crates to rlibs rather than directly invoking the linker when the binary crate gets
  compiled.
* Have cargo invoke rustc to perform the linking step. This final rustc invocation would determine
  what codegen was needed, do it, then invoke the linker on the resulting object files.

# Caching

If codegen is deferred until we are building a binary, then we need to make sure that we avoid
repeating the same codegen more than once. This means that when doing a warm build, we need only do
codegen for new / changed code.

We also need to be careful if we’re building multiple binary crates. All binary crates need to be
able to share the codegen outputs where appropriate. One way to achieve this might be as follows.

* Keep an index file in which we record which functions are in which object files.
* When invoking rustc to do codegen / linking, lock the index file, figure out which functions we’re
  going to codegen, update the index file to indicate which object files those new functions will be
  in, create those files and lock them, then release the lock on the main index file.
* That should hold the main index lock for a relatively short time after which another rustc process
  can do the same.
* When we finish doing codegen, before we invoke the linker, make sure that none of the object files
  we are going to pass to the linker are still locked, which would indicate that the rustc process
  writing them was still working.

Caching is quite possibly the hardest part of all of this to get correct. Ideally we’d like to avoid
storing each compiled function twice (once in the cache and once in the object file to be linked),
but this does make things significantly more complicated, especially without causing
non-determinism.

# Keeping memory usage in check

With all codegen being done by the one rustc process, care needs to be taken to ensure memory usage
isn’t too high. Several strategies might help here:

* Store graph information (what references what) separately from the MIR so that we can do a graph
  traversal without loading all the MIR.
* Load the MIR for each function only when we’re ready to codegen it, write the resulting machine
  code into an object file then drop it and the MIR.

# Why not do all compilation on demand?

It would be pretty hard to retrofit fully on-demand compilation to a mature compiler like rustc.
It’s also unclear how much you could actually skip. At least a bit of processing of each file is
required in order to find all trait implementations so that method resolution can give correct
results.

Correctness checks within function bodies could potentially be done only for functions that were
reachable. But that raises lots of questions about whether you’d want to do that. I’ve heard that
Zig doesn’t report some errors for dead code.

At least for now, it’s better to still do all correctness checking even for dead code.

# Related work

I was somewhat inspired here by an excellent episode of the Software Unscripted podcast where
Richard interviewed matklad. The episode is called “Incremental Compilation with Alex Kladov”
([link](https://podcasts.apple.com/us/podcast/incremental-compilation-with-alex-kladov/id1602572955?i=1000647825248)).
On the same podcast, in a much earlier episode, Richard interviews Andrew Kelley, the creator of Zig
([link](https://podcasts.apple.com/us/podcast/open-source-with-zig-creator-andrew-kelley/id1602572955?i=1000554066581))
which does a lot of its compilation in a more on-demand way.

Various related previous discussions:

* [Laziness in the compiler](https://internals.rust-lang.org/t/laziness-in-the-compiler/19112) (July 2023)
* [Towards a second edition of the
  compiler](https://internals.rust-lang.org/t/towards-a-second-edition-of-the-compiler/5582) (July
  2017)
* MIR-only RLIBs (Discussions on github / Zulip from January 2017 and more in 2024!)
  * The motivations for MIR-only RLIBs are different than for this post but there’s substantial
    cross-over.
* `-Z share-generics`
  * [Explicit monomorphization for compilation time
    reduction](https://internals.rust-lang.org/t/explicit-monomorphization-for-compilation-time-reduction/15907)

I've only linked to discussions where they're archived, but you can find open issues etc on these
topics with a quick web search.

# Next steps

I’m busy making the [Wild linker](https://github.com/davidlattimore/wild), however I think I
potentially have some bandwidth to start working on at least one of the ideas here. I haven't yet
figured out which one. If you’ve got any comments or would like to discuss this, my contact details
are on my about page.

# Thanks

Thanks to bjorn3, simulacrum, Jakub Beránek, davidtwco and nora (Nilstrieb) for providing feedback
on an earlier draft of this post. Any errors or inaccuracies are mine.

Thanks also to my [github sponsors](https://github.com/sponsors/davidlattimore). Your contributions
help to make it possible for me to continue to work on this kind of stuff rather than going and
getting a "real job".

* repi
* bes
* Urgau
* coastalwhite
* mstange
* bcmyers
* Shnatsel
* Rafferty97
* joshtriplett
* acshi
* teh
* yerke
* alexkirsz
* Pratyush
* lexara-prime-ai
* ethanmsl
* +1 anonymous
