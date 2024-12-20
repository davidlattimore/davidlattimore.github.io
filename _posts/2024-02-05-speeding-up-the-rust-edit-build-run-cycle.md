---
layout: default
title:  "Speeding up the Rust edit-build-run cycle"
date:   2024-02-05 00:00:00 +1100
categories: posts
redirect_from:
  - /working-on-rust-iteration-time.html
---
There are two main aspects to compile times that matter to developers. Cold build times, when
building from scratch and warm build times when you've already built and you're rebuilding following
an edit. This article focuses on warm build times, which for rapid iteration during development is
what generally matters most.

We start with some tips for speeding up your Rust development cycle, then talk about work that I'm
doing in this space to make it even faster.

For projects with minimal dependencies, you might find that your development cycle is already fast
enough. In that case, great. This article will therefore use a crate with some heavyweight
dependencies such that warm compilation times are a problem.

## Benchmark setup

* The benchmarked crate, benchmarking scripts can be found in [this
  repository](https://github.com/davidlattimore/warm-build-benchmark/commit/2ab40002dbbc3e6b0b8aa42b28a22894349695e6).
* All benchmarks were run on my few-year-old System 76 laptop. It's got an i7-10510U CPU, 42GB of
  RAM and is running Pop-OS (a Linux distribution).
* A couple of warmup builds are done first.
* Before each build, a trivial edit is made. This is similar to adding or removing a print
  statement, which is what we'd like to emulate.

To start, we benchmark the `cargo run` time for a default configuration debug build using the
default system linker (GNU ld). This gives us a time of 20.202 s ± 0.256 s which is slow enough to
be fairly annoying.

## Use a faster linker like Mold

Switching to a faster linker like `mold` (or `sold` for Mac) can make a very big difference. lld is
also pretty fast - see the timings for it later.

With mold version 2.3.3 installed, we add the following to our `.cargo/config.toml`:

```toml
[target.x86_64-unknown-linux-gnu]
linker = "/usr/bin/clang-15"
rustflags = ["-C", "link-arg=--ld-path=/usr/local/bin/mold"]
```

That reduces our `cargo run` time to 7.539 s ± 1.691 s.

## Avoid linking debug info

Debug information tends to be large and linking it slows down linking quite considerably. If you're
like many developers and you generally use println for debugging and rarely or never use an actual
debugger, then this is wasted time.

There are two ways you can do avoid linking debug information:

* Skip compiling the debug information in the first place by setting `debug=0` in your profile.
* Skip linking it, even it it was compiled by setting `strip="debuginfo"` in your profile.

Using just the second option, should mean in theory that if you need debug information, that you can
just need to relink. Unfortunately Cargo currently rebuilds everything from scratch when this option
is changed. Also, as far as I know, this option won't help on Mac, since rustc uses an external
`strip` command to remove the debug information after linking is complete, presumably because the
Mac linker doesn't support the `--strip-debug` flag.

One limitation of setting `debug=0` is that it doesn't affect compilation of the Rust standard
library because that comes precompiled with debug info. That means that with just `debug=0` set,
we'll still be linking a bit of debug info.

```toml
[profile.dev]
debug = 0
strip = "debuginfo"
```

With just `strip="debuginfo"` our time comes down to 1.507 s ± 0.301 s.

With just `debug=0` we're slightly slower at 1.611 s ± 0.329 s

With both we get 1.576 s ± 0.355 s.

All three are pretty similar and well within 1 standard deviation of each other, so they're
effectively the same.

If developing on Linux, I'd probably set both, since setting `debug=0` also improves the cold build
times (77 seconds down from 107). If developing on Mac, probably just set `debug=0`. If developing
on Windows, it's probably a good idea to try both and measure the warm build time to see what
combination has the best effect.

One downside of no debug info is that your backtraces will only have function names, with no line
numbers. Personally, I don't mind this and despite having `debug=0` in my profile for a long time,
didn't even notice until matthieum (on Reddit) pointed it out. Note that you'll still get the line
number where the panic occurred, just not line numbers for the functions that called it. I find this
to be an acceptable trade-off for faster warm builds.

If you do need debug info, setting `split-debuginfo="unpacked"` isn't quite as fast as no debug
info, but it's a lot faster than actually linking the debug info. Thanks VorpalWay on Reddit for
suggesting this option. Note however that this is already the default on Mac and it isn't supported
on Windows, so you'll probably only see a difference when you set this on Linux. Whatever you do,
it's a good idea to measure the effect of your settings change on your warm build times.

You might be wondering about the effect of setting `strip="symbols"` which strips not just debug
information, but also symbol tables. This can potentially speed up warm builds a little more, but
has the significant downside that you won't be able to get backtraces at all when you set
`RUST_BACKTRACE=1`, so I wouldn't recommend it for development builds.

## Building a non-relocatable executable

Rust by default compiles relocatable executables. This means that each time the binary gets run, the
operating system loads it at a random address. This is called ASLR (address space layout
randomisation) and is an important mitigation against security vulnerabilities. However it's
generally not something we need during development and it turns out that we can save a little more
time by turning it off. This step should be skipped if you're building or using shared objects (e.g.
.so files on Linux).

I found it easier to do this if we also statically link our binary with musl libc. The switch to
musl and to static linking didn't have any significant effect on the benchmark time.

You can install the musl toolchain with:

```sh
rustup target add x86_64-unknown-linux-musl
```

Then add the following to your `.cargo/config.toml`.

```toml
[build]
target = "x86_64-unknown-linux-musl"

[target.x86_64-unknown-linux-musl]
linker = "/usr/bin/clang-15"
rustflags = [ "-C", "relocation-model=static", "-C", "link-arg=--ld-path=mold" ]
```

This reduces our `cargo run` time to 1.156 s ± 0.039 s.

If you want to confirm that the executable is no longer relocatable, you can run `file` command on
it. It should say `ELF 64-bit LSB executable` instead of `ELF 64-bit LSB pie executable`. The "pie"
stands for Position Independent Executable.

## Alternatives to Mold

lld is also a pretty fast linker. It's not quite as fast as mold, but it's pretty fast. Switching to
lld increases our `cargo run` time to 1.602 s ± 0.011 s, which is still pretty good.

## Summary of improvements

At this point we've reduced the warm build (and run) time from 20 seconds to 1.2 seconds, a 16x
improvement! The main takeaways are:

* Use the fastest linker you can. 20 s -> 7.5 s
* Don't link debug info. 7.5 s -> 1.6 s
* Build a non-position-independent executable. 1.6 s -> 1.2 s

I'd suggest you experiment with what configuration works best for your project and platform. You can
also play with other configuration options. For example, if you're happy to abort if you get a panic
(at least during development), you might set `panic="abort"`, which from my measurements gives
another small reduction in warm build time. The key is to measure. Here's a one-liner to help with
that:

```sh
hyperfine --warmup 2 --prepare 'touch src/main.rs' 'cargo build'
```

If you don't have hyperfine, see the [hyperfine](https://github.com/sharkdp/hyperfine) repository
for installation instructions.

## Diagnosing unexpected rebuilds of dependencies

If you're frequently seeing cargo rebuild your dependencies when you've only changed your crate, it
can take ages and really sap your productivity. It's worth taking some time to figure out why. The
main tool to help diagnose this is building with the `-v` flag. e.g.

```sh
cargo build -v
```

You're looking for lines like the following:

```
Dirty rayon v1.8.1: the rustflags changed
Dirty regex-automata v0.4.5: the config settings changed
```

It's possible that you'll see crates being recompiled and Cargo doesn't give you a reason. One
common reason in this case is that the features for that crate have changed. This typically happens
when you're using a cargo workspace. Different crates in your workspace might request different
features from your dependencies. When building the whole workspace with `cargo build`, the union of
those features is used, however if you then request to just build a single crate, e.g. `cargo build
-p foo`, only the features needed by the `foo` crate will be built, which means the dependency needs
to be rebuilt. One way that some people solve this is to create a `workspace-hack` package that
depends on all your dependencies with the union of all their features then have all your packages
depend on `workspace-hack`. [cargo-hakari](https://crates.io/crates/cargo-hakari) is a tool to help
automate this. Other options are to just not use workspaces or never use the `-p` flag.

## Investigating remaining time

For many people, 1.2 s might be fast enough. But what if our project has an order of magnitude more
dependencies, or if we're running on a slower computer. Perhaps even 1.2 seconds isn't fast enough
for some.

Now that we've gotten it as fast as we can, it's time to look at what's taking time.

We can look at the times when each process gets started using `strace`. e.g.:

```sh
./random-edit src/main.rs; strace -tt -f --trace=execve -o strace.out cargo run
```

This increases the build time reported by cargo from 1.07s to about 1.33s, but that's still close
enough that it should give us a pretty good idea of how long things are taking. Looking at the
resulting `strace.out`, we can note a few things:

* The time from executing `~/.cargo/bin/cargo` (the rustup wrapper) to executing the actual cargo
  binary is 37 ms. [Rustup issue](https://github.com/rust-lang/rustup/issues/2626) for improving
  this.
* The time from executing `cargo` to executing `rustc` is 230 ms. I wonder if there's scope for
  caching whatever expensive computations cargo is doing here. Update: epage has been [looking into
  this](https://github.com/rust-lang/cargo/pull/13399).
* The time from when `rustc` starts until it finishes is 1100 ms. Not surprisingly this is where we
  spend the majority of our time.
* There's 19 ms between when `rustc` exits and when `cargo` executes our binary. Looks like this is
  a few different things. Writing a fingerprint file, deleting the old binary and putting the new
  binary in its place then freeing memory.
* The time to actually execute the binary we built is 3 ms.

Lets look a little more closely at what rustc is doing. Using a nightly version of rustc, we can
build with the `-Ztime-passes` flag.

```sh
./random-edit src/main.rs; cargo rustc -- -Ztime-passes
```

The most interesting lines are the following:

```
time:   0.411; rss:  138MB ->  285MB ( +147MB)	codegen_crate
time:   0.454; rss:  145MB ->  146MB (   +1MB)	link
time:   0.939; rss:   26MB ->   75MB (  +48MB)	total
```

So of the 939 ms total, we're spending about 454 ms linking, 411 ms on codegen and 74 ms on other
stuff.

## Possible future changes

### Changing the defaults

We managed a 16x speedup in warm `cargo run` time relative to the defaults. This is great, but new
users won't necessarily know to do these things and will just be left with the impression that rust
projects are slower to build than what's actually necessary.

There has been talk about bundling lld with rust and using it by default. That would go a long way.

I also wonder what might be done about the debug information. In an earlier version of this article,
I suggested that maybe the default should be no debug info, however given that this makes backtraces
not have line numbers, I'm now not so sure. Probably the `split-debuginfo="unpacked"` should become
the default on Linux like it is on Mac.

### Incremental linking

What if we didn't need to redo linking every time we made a change? i.e. if adding a print statement
to a function could just result in that one function being updated in our binary. This could
potentially give us an even faster dev cycle, especially for projects with even more dependencies or
where we still need debug information.

The downside of incremental linking is that the resulting binary probably wouldn't be bit-for-bit
identical to what we'd get if we linked from scratch. But personally I don't see that as a problem
when I'm iterating on code making small edits.

Implementing linkers is hard, complex work and adding incremental linking to the mix makes it even
more challenging. The Mold author [has
said](https://www.reddit.com/r/cpp/comments/kxvw5c/mold_a_modern_linker/) that incremental linking
is too hard and has too many downsides. But my experience is that Rust makes hard, complex things
easier to implement, so let's build an incremental linker in Rust!

There's somewhat of a tradition that linkers end with the letters "ld" and it's intended to
eventually be an incremental linker, so that suggests it ends in "ild". A quick search for words
ending in "ild" yields "wild" as an interesting name. Let's go with that.

Over the last couple of months, I've [made a start on it](https://github.com/davidlattimore/wild).
There's still heaps to do, including actually making it incremental, but it can now link itself.

The performance of the wild linker can't yet be properly compared with other linkers, since I
haven't yet implemented support for eh_frames, which are required for unwinding to work properly.
They're the next thing that I intend to implement.

My plan for this year is to work full time on improving Rust warm build times, starting by writing
an incremental linker. I'll be living off savings, the goodwill of my partner and github sponsors.
How long I can do that for will depend on how much sponsorship I can get. So if you think your
company might benefit from that, perhaps you can convince them to sponsor me? Huge thanks to my
existing sponsors, especially Embark studios.

## Discussion threads

[Reddit](https://www.reddit.com/r/rust/comments/1ajdm1v/speeding_up_rust_editbuildrun_cycle/)
