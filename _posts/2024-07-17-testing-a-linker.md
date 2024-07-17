---
layout: default
title:  "Testing a linker"
date:   2024-07-17 00:00:00 +0000
categories: posts
---
I’ve been writing a linker, called Wild (see [previous posts](https://davidlattimore.github.io/)).
Today, I’m going to talk about my approach to testing the linker. I think this is an interesting
case study in its own right, but also there’s aspects of the approach that can likely be applied to
other projects.

The properties that I like the tests for my projects to have are:

* I want to feel confident that they will pick up bugs if I introduce them when refactoring.
* They should be fast to run.
* They should be easy to diagnose what's wrong when they fail.
* They should be easy to maintain. When I refactor code, I should need to change tests as little as
  possible, or maybe not at all.

These priorities are sometimes in conflict with each other. For example merging several tests
together into a single test might make the test suite as a whole faster, but might also make
diagnosing what's wrong harder. Whether I choose to split or merge integration tests depends on
circumstances. Sometimes splitting is the right approach, especially if there's common work done by
each separate test than can be cached, thus regaining the speed. Often however I prefer to merge.
I'm more often running tests that pass than diagnosing tests that fail, so I'd prefer the speed.
Also, often with extra tooling, diagnosing what's wrong can be made easier, even in a large
integration test that is doing many things.

Unit tests can be very fast, however when you refactor your code, if you change an interface that is
unit tested, then the test needs updating or even rewriting. They can also very easily miss bugs
when interfaces don't change, but assumptions about who does what where in the code change.

I’ve been on projects that have relied entirely on unit tests and even with a high percentage of the
code covered by those unit tests, in the absence of good integration tests, the system has felt
incredibly fragile.

For these reasons, I generally focus first on integration tests, then resort to unit testing to fill
in gaps where I don’t think the integration tests are sufficient or would be too slow to cover all
the cases. I then build tooling in and around the integration tests to make them easier to diagnose
and maintain.

To provide some specific examples, I’ll now go into how the integration tests for the Wild linker
work.

When I started writing Wild, the first integration tests I wrote were of the form:

* Compile a small C program using GCC
* Link the program using GNU ld
* Link the program again using Wild
* Run the binaries produced by both linkers and make sure they both exit with the expected exit
  code.

Linking with GNU ld is important in order to ensure that the test itself is correct. We want the
program to behave the same when linked with both linkers.

Already here we can see some opportunity to speed up our test slightly with caching. Generally when
we rerun our test it’ll be because we made a change to the linker. However GCC and GNU ld are
unlikely to have changed. So if the C program and the argument we're passing didn’t change, then we
can skip rerunning GCC and GNU ld. This can be a significant saving, since GNU ld is really slow -
it often takes 10 to 30 times as long as Wild to link the same program.

Integration tests in Rust are typically put in a separate `tests` directory. Cargo will compile each
file in this directory as a separate binary. So if you have lots of completely separate integration
tests, this can get slow. For that reason, I generally only ever have a single integration test file
and do all my integration testing from that one file. It’s fine however to have multiple tests in
that file.

The Wild integration test compiles many small C, assembly and Rust programs, links them and runs
them. I include instructions for the test runner inline in the test in the form of specially
formatted comments.

```rust
//#Object:exit.c
//#ExpectSym: _start .text
//#ExpectSym: exit_syscall .text
//#EnableLinker:lld

#include "exit.h"

void _start(void) {
   exit_syscall(42);
}
```

In the example here, the first line tells the test runner to compile exit.c as an object file and
include that in the link. Then there’s a couple of assertions to check that some symbols are in the
correct output sections. The last instruction tells the test runner to enable linking with lld. This
is in addition to GNU ld and Wild that are always enabled for all tests.

```rust
//#AbstractConfig:default
//#DiffIgnore:section.tdata.alignment

//#Config:llvm-static:default
//#CompArgs:--target x86_64-unknown-linux-musl -C relocation-model=static -C target-feature=+crt-static -C debuginfo=2

//#Config:cranelift-static:default
//#CompArgs:-Zcodegen-backend=cranelift --target x86_64-unknown-linux-musl -C relocation-model=static -C target-feature=+crt-static -C debuginfo=2 --cfg cranelift

//#Config:llvm-dynamic:default
//#CompArgs:-C debuginfo=2
//#DiffIgnore:.dynamic.DT_JMPREL
```

In this more complex example, we’ve defined an abstract config in which we provide some default
settings. Then we have several configurations that inherit from that config and override various
properties. Each config has a unique name that is used for naming output files and when reporting
test failures. This test has a configuration that statically links with musl libc, one that uses the
cranelift backend and one that dynamically links.

Early on when developing the linker, if a test failed, it was generally necessary to step through
running the program in a debugger. I would step through both the output from GNU ld and the output
from my linker and see where they would diverge. The replay debugger `rr` was great for this as it
lets you step backwards in addition to forwards. However even with awesome tools like `rr`, this was
still a slow process. Fortunately it’s something I rarely need to do anymore.

The reason for that is that I now make extensive use of diffing against the output of GNU ld using a
tool I created called linker-diff. The binaries produced by different linkers are not byte-for-byte
identical and I wouldn’t want to try to make them so. However there’s lots of things we can diff,
even if the layout of the file is different. e.g.:

* Values of many of the header fields.
  * Even when the actual value of the header field is different, we can often interpret it in a way
    that can make it the same. e.g. when we look at the header field that contains the entry point
    for the program, the addresses will be different because the layout of the files is different,
    however if we look to see what symbol names point to that address, we’d expect them to be the
    same.
* We can disassemble global functions and check that the instructions match.
  * This is complicated somewhat because the instructions will often contain relative offsets to
    other functions, or absolute values that are expected to be different depending on how the
    linker laid out the binary. Similar to what we did with the entry point in the header, we can
    allow these instructions to match provided they point to a symbol with the same name.

Diffing linker outputs is non-trivial. Like linkers themselves, there are lots of corner cases. It
can be challenging to avoid false positives, while still detecting actual differences that we care
about. There’s still more than can be improved with the diff support, but already it has proved
incredibly valuable in diagnosing problems.

linker-diff is integrated into the integration tests. This means that generally now if I’m changing
how something works and I accidentally break something, rather than a mysterious and opaque test
failure when the binary produces the wrong result, I get a diff report showing where I did something
different to GNU ld.

One complication that arises, is where GNU ld is doing something that’s suboptimal. I observed this
with the linker not applying a particular optimisation if a symbol in our output binary was
referenced by a shared object that we were linking against. Trying to replicate GNU ld’s behaviour
here would have made our output binary link slower, run slower and added significant complexity to
our linker. Fortunately lld had better behaviour in this case. So what I ended up doing for my tests
was diffing Wild’s output against both the output of GNU ld and lld. For each thing we diff, e.g.
each instruction, header field etc, if Wild matches either GNU ld or lld’s output, then we accept it
as correct.

This is what typical output from linker-diff looks like:

```
wild: /wild/tests/build/libc-integration-0.clang-dynamic-b756cc1ceaeaa45d.wild.so
ld: /wild/tests/build/libc-integration-0.clang-dynamic-b756cc1ceaeaa45d.ld.so
lld: /wild/tests/build/libc-integration-0.clang-dynamic-b756cc1ceaeaa45d.lld.so
asm.get_weak_var
                  endbr64
                  push %rbp
                  mov %rsp,%rbp
  
  wild 0x00402429 48 8d 05 b0 10 00 00 lea 0x10BF,%rax  // weak_var
  ld   0x000011b2 48 8b 05 1f 2e 00 00 mov 0x2E2E,%rax  // DYNAMIC(weak_var)
  lld  0x00001a12 48 8b 05 3f 12 00 00 mov 0x124E,%rax  // DYNAMIC(weak_var)
  ORIG            48 8b 05 00 00 00 00 mov 7,%rax  // R_X86_64_REX_GOTPCRELX -> `weak_var`
  TRACE           relaxation=MovIndirectToLea value_flags=ADDRESS resolution_flags=DIRECT
  
                  mov (%rax),%eax
                  pop %rbp
                  ret
```

Here we can see the disassembly of the function `get_weak_var`. At the top and bottom are
instructions that are the same in the output of all three linkers.

In the middle is an instruction that is different. First we have a row for each of the three
linkers, wild, GNU ld and lld. We can see that GNU ld and lld both produced relative move
instructions that reference a dynamic relocation for a variable called `weak_var`. Wild however is
loading a relative address directly with no dynamic relocation. This may in fact still run
correctly, but only if this variable isn't overridden at runtime by the main executable or another
shared object. So this is, or rather was, a bug in Wild.

When diagnosing failures like this, it’s very helpful to be able to see what was in the input file.
I used to find this manually, however it's somewhat time consuming. So I added support to the linker
to write layout information to a .layout file. linker-diff then uses this to find where a particular
instruction came from in an input file and display that. That is shown on the line prefixed with
`ORIG`. The relocation type `GOTPCRELX` is especially useful in diagnosing what's happening.

It's often useful to be able to log the values of variables from the code in the linker. Matching
these log statements up to the output of the linker can be tricky. To help fix this, the linker can
associate tracing log statements with particular addresses in the output file. If linker-diff finds
any log messages associated with any of the bytes for an instruction that has a diff, then it'll
display them. This is shown on the `TRACE` line above. The code in the linker that emitted this,
then looks like this:

```rust
  let _span = tracing::span!(
      tracing::Level::TRACE, "relocation", address = place).entered();
  ...
  if let Some((relaxation, r_type)) =
      Relaxation::new(r_type, out, offset_in_section, value_flags, output_kind)
  {
      tracing::trace!(?relaxation, %value_flags, %resolution_flags);
      ...
  }
```

The first line creates the variable `_span`. Until this variable goes out of scope, all uses of
`tracing::trace!` will be associated with the address specified when we created the span.

When a test fails, it’s useful to be able to rerun the failing linker invocation outside of the
context of the test. If the bug is in linker-diff, then it’s useful to be able to rerun that. So
when a test fails, I print out the command lines to do both of these. I can then copy and paste
whichever I’d like to work on into my terminal.

```
...
Error: Validation failed.

WILD_WRITE_LAYOUT=1 WILD_WRITE_TRACE=1 OUT=/home/david/work/wild/wild/tests/build/libc-integration-0.clang-dynamic-b756cc1ceaeaa45d.wild.so /home/david/work/wild/wild/tests/build/libc-integration-0.clang-dynamic-b756cc1ceaeaa45d.wild.save/run-with cargo run --bin wild --

 To revalidate:

cargo run --bin linker-diff -- --wild-defaults --ignore '.got.plt,.dynamic.DT_PLTGOT,.dynamic.DT_JMPREL,.dynamic.DT_NEEDED,.dynamic.DT_PLTREL,.dynamic.DT_FLAGS,.dynamic.DT_FLAGS_1,section.plt.entsize,section.relro_padding' --ref /home/david/work/wild/wild/tests/build/libc-integration-0.clang-dynamic-b756cc1ceaeaa45d.ld.so --ref /home/david/work/wild/wild/tests/build/libc-integration-0.clang-dynamic-b756cc1ceaeaa45d.lld.so /home/david/work/wild/wild/tests/build/libc-integration-0.clang-dynamic-b756cc1ceaeaa45d.wild.so
```

When I find a program that misbehaves when linked with Wild, the first thing I want to do is try to
figure out what Wild is getting wrong. To help with that, I’ve integrated support for running linker
diff into Wild itself. This is done by setting the environment variable `WILD_REFERENCE_LINKER` to
the name of a reference linker to invoke.

```sh
WILD_REFERENCE_LINKER=ld RUSTFLAGS="-Clinker=clang -Clink-args=--ld-path=wild" cargo test
```

When set, Wild will run the reference linker (GNU ld) with the same arguments as those it was
invoked with, but change the output file. It’ll then invoke linker-diff to check for unexpected
differences, then fail the link if any are found.

Once I’ve identified the part that Wild is getting wrong, I can try to add something similar to one
of my existing test programs.

Wild’s tests still have lots more that needs doing. I’ve mostly focussed on the happy path so far,
since getting even that right is tricky. Soon I'll probably need to start looking at testing error
conditions. I’ll likely follow a somewhat similar approach of having some test programs and making
sure that both the reference linker and Wild reject them and that each linker includes some specific
string in the error output - e.g. the name of a symbol that was unresolved.

At some point in the future, I’m interested in trying fuzzing as a testing strategy. Profile-guided
fuzzing could find interesting inputs that hit corner cases in the linker not covered by regular
tests.

The eventual plan for Wild is to make it incremental. When it comes time to start working on this, I
think linker-diff will again be useful. My plan is test as follows:

* Link a test program with wild. Call this output A.
* Make a random change to the input objects (possibly via fuzzing), then link this with wild. Call
  this output B.
* Undo the random change we made and incrementally link. Call this output C.
* A and C should be semantically the same, so if we diff them with linker-diff, it should report no
  differences.

Another strategy I’m keen to employ is mutant testing (see [mutants.rs](https://mutants.rs/)). This
makes random changes to your code that should change behaviour - e.g. inverting a comparison - then
checks if any of your tests pick up the change. Not only does this have the potential to pick up
gaps in testing, but it may also help find bits of code that are unnecessary. I’d also be interested
in seeing if it could be used to rank tests by how many problems they detect that other tests miss.
Tests that only detect a subset of the bugs detected by other tests would be candidates for removal.

I hope this look into how I approach testing and in particular testing of the Wild linker has given
you some ideas for your own projects.

# Thanks

Thanks to my [github sponsors](https://github.com/sponsors/davidlattimore). Your contributions help
to make it possible for me to continue to work on this kind of stuff rather than going and getting a
"real job".

* bearcove
* repi
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
* tommythorn
* flba-eb
* acshi
* teh
* yerke
* alexkirsz
* NobodyXu
* Pratyush
* ethanmsl
* +2 anonymous

# Discussion threads

* [Reddit](https://www.reddit.com/r/rust/comments/1e54pml/testing_the_wild_linker/)
