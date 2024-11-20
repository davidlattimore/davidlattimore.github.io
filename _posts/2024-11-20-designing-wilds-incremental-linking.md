---
layout: default
title:  "Designing Wild's incremental linking"
date:   2024-11-20 00:00:00 +1000
categories: posts
---
# Designing Wild's incremental linking

Whenever I’m about to embark on implementing something even slightly non-trivial, I typically write
out a plan for what I’m about to do. Writing it down helps me to uncover things that I hadn’t
thought about. Usually, I write these designs only for myself, however this time, I thought I’d try
something different, writing a design and sharing it with anyone who might be interested. My hope is
that some people might have interesting ideas for variations on this design that I hadn’t
considered. If nothing else, hopefully this document will be an interesting read for someone.

If you’ve read my [previous posts](https://davidlattimore.github.io/), you’ll know that I’ve been
writing a linker, called Wild, with the goal of being a very fast incremental linker. I didn’t make
the linker incremental from the start because I wanted to get it reasonably correct and reasonably
fast before adding incremental linking to the mix. Wild is now working well enough that I’ve
switched to using it as my default linker. That’s not to say there aren’t things it doesn’t do
correctly, e.g. large model stuff, which is needed for very large executables, but it works well
enough for compiling Rust code with bits of C or C++ mixed in. It’s also relatively fast - on my
laptop, Wild can link itself 48% faster than Mold. If there’s lots of debug info however, Wild is
currently often slower. I hope to work on improving the performance of linking debug info
eventually.

So I feel that while there’s plenty more that could be done to improve Wild’s non-incremental
linking, now is probably a good time to start work on incremental. To that end, this document is my
plan for how I intend to go about that.

# The end-goal

First, let’s discuss the reason for wanting incremental linking. Mostly it’s to make linking as fast
as possible. I’d like it if when I’m making edits to a test case, I could see the pass / fail status
of that test within say 10 ms of hitting save. Incremental linking alone isn’t sufficient to reach
this goal, but it is necessary. There is lots of work needed to get to that point, including lots of
big changes to how the Rust compiler works. Let’s leave those changes for separate discussions,
since this document is about incremental linking.

In order to get that kind of speed, we can’t afford to reprocess all the inputs, rewrite the entire
output etc. We need to make minimal edits to update the existing binary on disk. For example, if
we’re editing our test case, then we’d like to just be rewriting the part of the executable that
contains the compiled code for that test, plus possibly a table of line numbers used by panics.  

A further goal of incremental linking is as a step towards hot code reloading - i.e. updating a
binary while it is running. That too, however, deserves its own document, so I won’t go into it in
detail now. It does however influence the design. For example, one idea for fast incremental linking
is to do an initial link with all your dependencies, then the final link just tacks on your code.
This might be fine for the goal of fast linking, however it doesn’t help with the eventual goal of
hot code reloading.

# Out-of-scope for first implementation

Linkers are pretty complex bits of software even without incremental linking. Adding incremental
updates into the mix adds significantly to this. However we can make things a little easier on
ourselves by reducing the scope somewhat.

## Archive semantics

One area of complexity in linkers is related to archive entries - so called “archive semantics”.
Linkers ignore entries in archives unless the archive entry defines at least one symbol that a
previous input object left undefined. This is used to avoid unnecessary initialisation of subsystems
that aren’t in use. Changing which archive entries are active in an incremental link would add
substantial complexity. Our target use case is incremental linking of Rust code, and Rust code,
while it uses archives for rlibs, doesn’t make use of archive semantics. So supporting incremental
updates of archive semantics would be a lot of work for very questionable benefit towards our use
case. For that reason, the plan is to punt on it for the first implementation.

## Unused section garbage collection

Most linkers support a flag `--gc-sections`, which causes them to get rid of sections that aren’t
reachable from a root or marked as must-keep. Wild supports this flag, and in fact does it by
default. However supporting this in conjunction with incremental linking would add extra complexity,
so we’ll skip this for now. The main downsides of this are that the binary will end up a bit bigger
and we might spend a bit of extra time copying data into the output file

## Removal of old merged strings

Strings in string-merge sections may be removed in subsequent links. Removing them from the output
would require reference counting each string. Besides taking up a little extra space in the binary,
there doesn’t seem to be much downside to keeping them around, so for now, we’ll do that.

## Strictly ordered sections

Our approach to incremental linking depends on not moving stuff that hasn't changed. If we need to
put input sections in a particular order in the output, then we might need to relocate unchanged in
order to make space. This would hurt performance. Most output sections are fine to put input
sections in any order. However a few aren't. For example `.init` is a section that contains a single
function and parts of that function come from different object files. The return instruction for
this function comes from `crtn.o` and it's essential that this goes at the end of the output
section.

For now, we'll just fall back to a full initial-incremental link if any sections that have strict
ordering get changed. Fortunately these aren't the kinds of sections that you tend to edit when
iterating on code unless you're doing something very obscure. Modern code tends not to use these
sections anyway, but rather uses `.init_array` which we discuss later in this document.

# Configuring incremental linking

To enable incremental linking, I’ll add a flag `--incremental`. I’ll probably also support setting
an environment variable - `WILD_INCREMENTAL=1`, since in many cases that may be easier for a user to
set than a flag.

I'll likely add additional configuration, in particular a flag for what percentage of additional
space to add to each output section to allow for growth.

# Object diffing

Ideally, when doing an incremental link, the compiler would pass only the bits that have changed to
the linker. This would be in the form of a list of updated, added and maybe deleted sections. Each
section would generally contain exactly one symbol that would point to the start of the section,
however we also need to handle sections with 0 or more than 1 symbols.

Unfortunately, for the time being, the compiler isn’t going to give us this list of updated
sections, so we’ll have to compute it ourselves. By separating incremental linking into two phases -
(1) computing a diff then (2) applying a diff - it’ll be easier to take advantage of future
compilers that can just supply us the diff directly. This separation may also open up extra options
for debugging and testing of incremental linking - e.g. testing each part separately.

The first stage of diffing will be determining which files have changed. We could hash the entirety
of each file, however, with lots of input files, that would be expensive, so instead the plan is to
just check to see if the modification timestamp has changed.

Once we have a list of changed files to look at, we can open each changed file and determine which
sections have changed.

Matching sections between the old and new versions of the object file is slightly tricky. For
sections containing code, the section should have a name that includes the mangled symbol name of
the function. e.g. `.text._ZN4core3fmt9Formatter9write_fmt` These are easy to match, since the name
should remain unchanged. However, sections containing anonymous data are harder. They have names
like `.rodata..L__unnamed_75`, which will likely change when edits are made. My plan at this stage
is to match these sections by looking at what references them. So for example if
`.text._ZN4core3fmt9Formatter9write_fmt` in the old object file references `.rodata..L__unnamed_75`
then in the new object file it references `.rodata..L__unnamed_78`, we’d match those two sections
for the purposes of diffing.

In order to diff the old object file against the new object file, we need to keep a copy of the old
object file. This can be done relatively quickly by making a hard link for each input file. Files to
which we don’t have write access, or which are located on a different filesystem than our
incremental state directory, would be skipped. These are likely system libraries and are unlikely to
change. If they do end up changing, then we’d need to link from scratch.

Rust writes each crate that we depend on as an .rlib file. These are archives which will often
contain multiple object files - one for each codegen unit. When such a crate is edited, one or more
of the codegen units within the archive will be updated. Unlike with bare object files on disk, we
can’t rely on timestamps to determine whether a file within the archive has changed. At least we
can’t at the moment because rustc doesn’t set the timestamp field. We can probably just compare the
bytes of the files directly for now.

# Persistent state

Wild will need to write various bits of state to disk in order to support making incremental updates
to the output file. My plan at this stage is to put these into a directory with a name based on the
output file. For example, if the output file is `target/debug/ripgrep`, then the incremental
directory would be `target/debug/ripgrep.incr`.

When accessing state files during an incremental link, we’ll often want to avoid reading the entire
file. In most cases, this will be done by using mmap to access the file. This means that the on-disk
and in-memory format will need to be the same. We don’t need to worry about things like endianness
of the data, since moving the incremental link state between machines isn’t a use-case we intend to
support.

## Previous input files and other metadata

As mentioned above with regard to diffing, we’ll likely need to store copies of the old input files.
We can put these in a subdirectory of our state directory.

We’ll also need an index file that contains information about all of the input files and arguments
for the previous link. This file shouldn’t be large, so we can probably afford to serialise and
deserialise it each time.  

We can store additional information here such as:

* The size and capacity of each output section.  
* The version of Wild used.
* Any additional small bits of information such as sizes of various tables.

## Symbol name to symbol ID map

When linking the updated code, we need to be able to quickly look up symbols by name and we don’t
want to have to rebuild the map from symbol names to symbol IDs every time we do an incremental
link. This means that we’ll need to persist our map from symbol names to symbol IDs to disk.
Currently, this is a hashmap stored in memory. The keys of this map are currently `&[u8]` - i.e.
slices of bytes. These slices reference data from the original input objects. This means that when
building this hashmap, we don’t currently copy the bytes of the symbol names, we just use them
in-place. Persisting this is somewhat tricky.

In the short term, the easiest option is probably to just accept that when incremental linking is
enabled, we’ll need to copy the symbol names into our map. If we’re doing that, we can use some
existing crate like `sled` to store our map. Besides needing to copy our symbol names, sled does
other things that we don’t really need like transactions. But it’ll get us going quickly and we can
iterate from there.

Longer term, I think what will give the best performance will be something like `odht` (an on-disk
hash table), but where the keys are external to the table. So hashing or comparing a key would
involve an external lookup to fetch the bytes of the actual key.

It’s likely that whatever we do here, it’ll be slower than what we’re doing now. We don’t want to
slow down the linker if incremental linking is disabled, so we’ll need to keep the existing
in-memory hashmap implementation around. We should be able to switch between the in-memory and the
on-disk maps by making code that does name lookups generic over some trait.

## Symbol resolution table

We currently store a table in which we can look up the address, GOT address, PLT address etc of each
symbol. This is stored in memory as a `Vec`. This can probably just be changed to an mmapped file
instead. Accessing this table should be pretty similar whether it’s backed by a Vec or by a file.
Initialising it will be different and a bit slower for file-based storage, since we’d need to
zero-initialise the file when we create it before we could mmap it, whereas currently we don’t
zero-initialise and just write the resolutions concurrently from multiple threads. We could
experiment with not using mmap for the initial write of this file. i.e. just create the Vec, then
write the bytes of the Vec to a file.

## Relocation reverse index

When a symbol gets moved, e.g. because it’s in a section that got updated and the new version of
that section didn’t fit in the old spot, we need to update all references to that symbol to point to
the new location.

In order to do this efficiently, we need to store all relocations indexed by the symbol to which
they refer. Doing this efficiently from multiple threads without ending up with non-deterministic
results is somewhat tricky. Certainly creating a Vec for each symbol to hold all the references to
that symbol would likely be too expensive.

My current plan is, for each symbol, to store the index of the first relocation that references that
symbol. Then for each relocation, store the index of the next relocation for the same symbol. This
would mean that the list of relocations for a symbol is stored effectively as an index-based linked
list within the list of relocations.

This approach to storage can be done with 2 or 3 flat files that we can treat as mutable slices of
their respective data types. We can build these in-place from multiple threads provided we use
atomic compare-exchange operations to update the list heads.

## Dynamic relocation reverse index

Input sections may contain relocations that refer to symbols provided by shared objects. Such
relocations cannot be resolved at link time and must instead be resolved at runtime. This is done by
emitting dynamic relocations. Executable code will generally not make direct use of such
relocations, but instead use the global offset table (GOT) which will then have the dynamic
relocation. Data sections however often contain vtables which will need dynamic relocations. If such
a data section gets removed or updated, then we need to make sure we remove or update any dynamic
relocations associated with the old version of that section.

Wild doesn’t currently have a concept of a global section ID. All storage of information about
sections is currently on a per-input-file basis. This is inconvenient for the purposes of storing a
reverse index for dynamic relocations, so probably, I’ll introduce a global input section ID, then
store a table from input section ID to the first dynamic relocation for that input section. All the
dynamic relocations for a section should be adjacent, so we can also just store the count of
relocations.

## Exception frames

Exception frames are needed in order for backtraces and panics to work. Information about all
executable sections is put in the `.eh_frame` section. The linker splits this input section up by
locating the individual frame description entries (FDEs) then recombining them into the output
`.eh_frame` section. The linker also needs to write a `.eh_frame_hdr` section which is a sorted
index of frame addresses and is used at runtime to do a binary search in order to locate the frame
information for a particular address.

When an executable section is updated or removed, we need to update or remove the corresponding
FDEs. Any change to the FDEs will require a corresponding change to the sorted `.eh_frame_hdr`
section.

Similar to dynamic relocations, all FDEs for an input section will be adjacent in the output file,
so a start index and a count should be sufficient to identify which FDEs belong to a particular
input section.

## String merge index

Input sections that have the M (merge) and S (string) bits set are string-merge sections. At link
time, we locate each string, by looking for its null terminator, then deduplicate the string with
other strings that are destined for the same output section.

When we incrementally link, some of the strings in a string-merge section may have changed. Even if
none have changed, we still need a way to look up the address for a particular string. This means
that we need to persist, for each output string-merge section, an index of where each string is
located. In some ways this is similar to our symbol-name to symbol-ID map. As with that map, we’ll
initially use a third-party on-disk database like `sled`, then later look at more optimised options
to avoid copying the actual strings.

# Logging

A log of links will by default be written to the user’s [state
directory](https://docs.rs/directories/5.0.1/directories/struct.ProjectDirs.html#method.state_dir).
This will be able to be displayed by running `wild log` and will show a line per linker invocation
with information about whether we did a full link or an incremental link and if we did a full link,
the reason why. The intention here is to provide a way for a user to be able to diagnose why
incremental linking isn’t behaving as expected.

# Algorithm

Once incremental linking is implemented, the linker will have three modes of operation.

* Non-incremental. In this mode, it’ll behave much like it does now.
* Initial-incremental. It’ll link from scratch but prepare for subsequent incremental linking.
  Output sections will have additional space allocated so that they can grow and various state files
  will be written.
* Incremental-update. Update the output file by making minimal changes and leaving the rest in
  place. Will also need to update the state files to reflect changes that were made.

The following is a rough outline of the proposed algorithm for an incremental-update. If any stage
fails, then it’ll fall back to doing initial-incremental.

* Check changes in flags.
* Check if a previous attempt to incrementally link was interrupted or didn’t complete for some reason.
* Identify changed files.
* Diff changed files to produce section update list.
* Determine how much additional space needs to be used in each output section. This includes
  generated sections such as the global offset table (GOT), dynamic relocations etc.
* Allocate addresses for each changed / added section. A section that has run out of space will
  result in failure (fallback to initial-incremental), however this may be relaxed in future for
  cases where we can safely create an additional section of the same type.
* Update symbol resolutions and record which symbols have changed their resolution.
* Write updated / added sections to the output file.
* Rewrite relocations for symbols with changed resolutions.
* Add / remove / update dynamic relocations
* Add / remove / update exception frame information
* Update .eh_frame_hdr by performing insertions and removals corresponding to FDEs that we added /
  removed. We can either do this by sorting the list of additions / removals, then doing a single
  pass over .eh_frame_hdr to merge in the added / removed index entries, or we could rebuild and
  resort the entire index.
* Update other state files.

# Sections that can't contain gaps

Sections containing code or data are generally fine to have gaps within them. However there are some
sections that cannot contain gaps or where if there are gaps, they need special handling. For
example `.init_array` is a list of pointers to initialiser functions that get run on startup. An
uninitialised element of this array would lead to undefined behaviour (likely a crash). For a
section like this that we know contains function pointers, we could fill gaps with pointers to a
no-op function. However, custom sections can also be declared where the linker generates symbols
that point to the start and end of the section. For such custom sections, we don't have any
reasonable filler value to put in gaps.

Where gaps would be left, we can probably relocate input sections from the end of the output section
to fill the gaps. This should work provided all the input sections are the same size - generally the
case when these sections actually just contain pointers. If we have input sections with different
sizes, then we might need to rewrite the whole output section, although initially, we'll probably
just fail the incremental update and fall back to a full initial-link.

# Testing

Most of Wild’s tests are small programs written in C, assembly, Rust etc. These programs get
compiled then linked with both GNU ld and Wild. They then get executed to make sure they produce the
expected result. We also compare the outputs using linker-diff (part of the Wild repository) which
helps by making it more obvious what we’re getting wrong and also picks up some kinds of bugs that
just executing our test binaries might not detect.

In order to test incremental linking, we can extend this system by compiling multiple versions of
each input file. For C code, we could predefine some macro, for example `-D WILD_INC=1` that the
code can then use to switch between different definitions of some function or data.

In addition to diffing the resulting binaries against the output of GNU ld, we can also diff the
incrementally linked output from Wild against a non-incremental output of Wild for the same inputs.

# Feedback

Hopefully most, or at least some of that made sense. If you have any thoughts or questions, please
do reach out. My contact details can be found on my [about page](/about) or you can comment on the
Reddit thread that I'll link below.

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
* pmarks
* coastalwhite
* mstange
* twilco
* binarybana
* willstott101
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
* gendx
* teh
* nazar-pc
* yerke
* drmason13
* NobodyXu
* jplatte
* ymgyt
* Pratyush
* +2 anonymous

# Discussion threads

* Reddit: TODO
