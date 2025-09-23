---
layout: default
title:  "Wild Linker Update - 0.6.0"
date:   2025-09-03 00:00:00 +1100
categories: posts
---
Today, we've released [Wild version
0.6.0](https://github.com/davidlattimore/wild/releases/tag/0.6.0). There were many changes and we
were probably overdue for a release, having last released in May.

This release saw contributions from many people:

* davidlattimore: 90
* marxin: 69
* lapla-cogito: 41
* mati865: 28
* RossSmyth: 6
* daniel-levin: 3
* Noratrieb: 2
* lqd: 1
* m-hugo: 1
* dawnofmidnight: 1

That's the number of commits, which isn't a great measure, but it's something. Importantly, more
than half of the commits made were made by people other than me. It's awesome to see wild growing
into a team project. If you'd like to contribute, come along and have a chat on the [Wild
Zulip](https://wild.zulipchat.com/join/bbopdeg6howwjpaiyowngyde/) or have a look through the issues
for something you think you'd like to try implementing / fixing.

My work on the project has been reduced a bit the last couple of months due to me speaking at first
RustForge, then RustChinaConf. Conference preparation takes me a lot of time and I need to get
better at managing that preparation work while still getting other stuff done. In any case, the
conferences are over now and I'm looking forward to getting some solid work done.

The last few months we had Kei (lapla-cogito) join us for Google Summer of Code (GSoC). It was
awesome having Kei work with us. As you can see above, a lot of work got done. The project focused
on setting things up to run Mold's test suite with wild. This is now running in CI and helps fill
some gaps in our own tests as well as highlight things that we haven't yet implemented. Kei also did
a lot of other fixes and improvements. One of the more notable ones was implementing `--help`, which
was something we'd wanted for a while. I look forward to continuing to work with Kei going forward.

Martin (marxin) added initial RISC-V support to this release. There's still probably a little bit
more that could be done on this, but it basically works. Kei is working on adding RISC-V support to
linker-diff, which will help with further work in this area.

With this release, we now do release builds of Wild with Wild. i.e., we're using it in "production".
As such, we've removed the language from our README that used to say not to use it in production.
That's not to say you should do production builds with it and just put them out there. We definitely
recommend thorough testing. Wild is still intended primarily for fast development, but if you'd like
to use it for other things, who are we to stop you. As always, be sure to let us know if you hit any
problems.

With 0.6.0, we can now link the Chromium web browser. This is an interesting stress test for linkers
because it's really big. The binary, even without debug info is 1.4 GiB.

![Benchmark of time to link Chromium](/images/0.6.0/chromium.svg)

It's worth noting that the relative difference between lld and mold is very different to what's seen
in the benchmarks on the [mold repo](https://github.com/rui314/mold). This is likely due to the
benchmark machine being very different. i.e., my laptop has a lot less cores.

The following benchmark is for librustc-driver, which is where most of the code in the Rust compiler
goes.

![Benchmark of time to link librustc-driver](/images/0.6.0/librustc-driver.svg)

Our final benchmark is the bevy dylib. This is an interesting benchmark since it has a very large
version script and produces a shared object with more than half a million dynamic symbols.

![Benchmark of time to link bevy dylib](/images/0.6.0/bevy-dylib.svg)

My laptop has 4 cores and 8 threads. All my development work to date has been on this machine and on
it, Wild performs really well, often beating other linkers by a factor of 2 or sometimes more.
However, on machines with more cores, the performance isn't so great. We've started to look into
this to see what we can do about it. One area that particularly stands out is string merging. This
is where there's a section containing null-terminated strings that need to be deduplicated with
similar sections in other object files. Sounds easy, but getting it to perform well with multiple
threads is hard. We've gone through several different implementations of string merging in an
attempt to get good performance. Our current implementation is probably too complex. It's also
performing badly in some cases. In particular when there aren't many input sections but there are
lots of threads. In this case it's actually getting slower the more threads it has.

As such, I'm considering doing another rewrite of string merging. One option I'm considering here is
to change the way string-merge sections are represented in ELF files. I suspect that with a few
tweaks to how they're represented, we could get much better performance.

At a high level, the idea would be to store an additional section containing an index of the strings
to be merged. This index, similar to the symbol table, would contain the start offset of each
string. Where string relocations (references) are currently by section number + section offset, we'd
change them to be section number + string number. That means that we'd need a new relocation type.
Additionally, the string index would also store a hash of each string and the strings would be
sorted by hash.

I'll probably do some experiments on this front to see what's possible. If it performs well, then we
can talk to other linker authors and compiler writers to see if there's interest in the new
representation. I'll try to write a blog post about the outcome, even if it doesn't work out.

While string-merging is the worst offender in terms of scaling the number of threads, it looks like
other areas are also not ideal. This needs more investigation. I suspect at least part of the issue
might be rayon.

One area where we know we have a problem with rayon is its `try_for_each_init` API. We use this to
allocate a per-thread arena in a couple of cases. Unfortunately, rayon runs the init block for
pretty much every work item rather than just running it once per thread. This means that we end up
generating many times more arenas that we need, which is pretty wasteful. This is a known issue in
rayon, but I think it's perhaps not clear how to fix it with rayon's architecture.

I'm keen to try alternatives to rayon to see what difference they make. In particular, I've been
looking at [orx-parallel](https://github.com/orxfun/orx-parallel/). Once it has thread-pool support
and some way to handle graph algorithms (e.g. task spawning), I'll definitely be giving it a try.

Trying [chili](https://github.com/dragostis/chili) would also be interesting, but it's pretty low
level, so we'd need quite a few abstractions built on top of it (e.g. par_iter) before we could
reasonably use it.

If you've been following this project for a while, you might be wondering what's happening with
incremental linking. I thought that I was ready to start on this about a year ago, but it turns out
that I underestimated how much more there was to get a solid linker, so fixing bugs and adding
missing features has occupied most of my time. When I started the linker, I wasn't expecting to get
such good performance with non-incremental linking. Seeing the performance that we've gotten has
changed the equation a bit in terms of what seems important to work on. Anyway, I still intend to
get to incremental eventually, but I won't promise when.

There are lots of other things that we may or may not work on in the coming months. Possibilities
are:

* Improving linker-script support
* Linker-plugin LTO. Not needed for Rust LTO, but is needed for LTO of other languages.
* Improved symbol version support (Martin might be looking at this)
* Garbage collection of redundant / unused debug info. This one is a bit daunting, so we probably
  won't do it, but it'd be cool if we did, since it's something that none of the other linkers do.
* Putting ELF-specific stuff behind a trait to make porting to Windows / Mac easier.

Thanks to everyone who has been [sponsoring me](https://github.com/sponsors/davidlattimore), in
particular the following people who have sponsored at least $30 since the last release:

* CodeursenLiberte
* pmarks
* mati865
* repi
* Urgau
* teburd
* flba-eb
* tommythorn
* binarybana
* bcmyers
* Kobzol
* HadrienG2
* bes
* twilco
* mstange
* marxin
* joshtriplett
* jonhoo
* +1 anonymous
