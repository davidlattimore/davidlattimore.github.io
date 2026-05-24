---
layout: default
title:  "Wild Linker Update - 0.9.0"
date:   2026-05-24 00:00:00
categories: posts
---

We've just released Wild version 0.9.0. It was a bit overdue, with our last release being back in
January. It's way too tempting to try to get just one more change in before the release, but you've
got to cut it somewhere.

Lots of progress has been made on linker script support. This work was done by a number of different
people in the leadup to GSoC (Google Summer of Code) 2026. Vishruth Thimmaiah has been selected to
work on linker script support through GSoC. The aim of the project is to get enough linker script
support to allow linking of Linux kernel modules and possibly even the kernel itself using Wild.

There were lots of bug fixes and new features as well as lots of work on porting. The ports
currently under way are to support output of Mac and Wasm.

Kei Akiyama is leading the Wasm port as a GSoC project. This is Kei's second year doing GSoC on the
Wild project.

Martin Liška is leading the Mac (Mach-O) porting effort.

Linker plugin LTO is a way to allow the compiler to run as part of the linker process, performing
cross-module optimisations. This isn't needed for projects that are primarily Rust code, since the
Rust compiler can do LTO itself without involving the linker. However, for C, C++ or mixed language
projects, linker plugin LTO is needed if you want these kinds of optimisations. Wild now has basic
support for this, with a few known issues still to be worked out. If your goal is fast link times,
then Wild with LTO won't be any faster than the other linkers and might even be a litte slower. LTO
link time is dominated by the work the compiler does. You're effectively doing most of the
optimisation and machine code generation at link time every time you link, so it's going to be slow.
The motivation for supporting linker plugins is that it can allow people who use linker plugin LTO
occasionally to use Wild as their linker without needing to switch to a different linker every time
they need to use LTO.

Another frequently requested feature that we now support is outputting compressed debug info. Prior
to 0.9.0 we supported compressed input debug info, but we didn't support compressing the output
debug info. Now we support both input and output, with options for zlib and zstd compression. This
generally slows down linking, however if your filesystem / disk is slow enough, then it might
actually be a performance win, since we end up writing less data to disk.

Wild now supports range-extension thunks on aarch64. This is necessary when the executable code part
of your binary grows beyond about 128 MiB. For example, the chromium binary is well over this size,
especially in non-optimised builds. This allowed us to include a [benchmark for
chrome-arm64](https://github.com/wild-linker/wild/blob/main/benchmarks/ryzen-9955hx.md#chrome-arm64---time)
for the first time.

Performance-wise, not much has changed with this release. While we didn't focus on improving
performance, since our attention was elsewhere, it's still a good result. It takes work just to keep
the performance from regressing, especially with so many bug fixes and new features.

For more info, you can read the full release notes for [Wild version
0.9.0](https://github.com/wild-linker/wild/releases/tag/0.9.0).

Thanks to everyone who has been [sponsoring me](https://github.com/sponsors/davidlattimore), in
particular the following people who have sponsored at least $30 since the last release:

* wasmerio
* repi
* pmarks
* mati865
* marxin
* Urgau
* Tudyx
* flba-eb
* rerun-io
* binarybana
* HadrienG2
* appcove
* ChezBunch
* bes
* twilco
* sourcefrog
* simonlindholm
* petersimonsson
* mstange
* joshtriplett
* belzael
* bcmyers
* +4 anonymous
* +37 others

# Discussions
