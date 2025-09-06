---
layout: default
title:  "Wild Performance Tricks"
date:   2025-09-03 00:00:00 +1100
categories: posts
---

Last week I had the pleasure of attending RustForge in Wellington, New Zealand. I gave a talk titled
"Wild performance tricks". You can watch a [recording of my
talk](https://www.youtube.com/live/6Scgq9fBZQM?t=9246s). If you'd prefer to read rather than watch,
the rest of this post will cover more or less the same material. The talk shows some linker
benchmarks, which I'll skip here and focus instead on the optimisations, which I think are the more
interesting part of the talk.

The tricks here are a few of my favourites that I've used in the [Wild
linker](https://github.com/davidlattimore/wild).

## Mutable slicing for sharing between threads

In the linker, we have a type `SymbolId` defined as:

```rust
struct SymbolId(u32);
```

We need a way to store resolutions, where one `SymbolId` resolves (maps) to another `SymbolId`. If
we need to look up which symbol `SymbolId(5)` maps to, we then look at index `5` in the `Vec`.
Because every symbol maps to some other symbol (possibly itself), this means that we make use of the
entire `Vec`. i.e. it's dense, not sparse. For a sparse mapping, a `HashMap` might be preferable.

The Wild linker is very multi-threaded, so we want to be able to process symbols for our input
objects in parallel. To achieve this, we make sure that all symbols for a given object get allocated
adjacent to each other. i.e. each object has `SymbolId`s in a contiguous range. This is good for
cache locality because when a thread is working with an object, all its symbols will be nearby in
memory, so more likely to be in cache. It also lets us do things like this:

```rust
fn parallel_process_resolutions(mut resolutions: &mut [SymbolId], objects: &[Object]) {
   objects
       .iter()
       .map(|obj| (obj, resolutions.split_off_mut(..obj.num_symbols).unwrap()))
       .par_bridge()
       .for_each(|(obj, object_resolutions)| {
           obj.process_resolutions(object_resolutions);
       });
}
```

Here, we're using the Rayon crate to process the resolutions for all our objects in parallel from
multiple threads. We start by iterating over our objects, then for each object, we use
`split_off_mut` to split off a mutable slice of `resolutions` that contains the resolutions for that
object. `par_bridge` converts this regular Rust iterator into a Rayon parallel iterator. The closure
passed to `for_each` then runs in parallel on multiple threads, with each thread getting access to
the object and a mutable slice of that object's resolutions.

## Parallel initialisation of the Vec

The previous technique of using `split_off_mut` to get multiple non-overlapping mutable slices of
our Vec relies on the Vec having already been initialised. We'd like to initialise our Vec in
parallel, otherwise we'd have to wait for the main thread to fill the entire Vec with a placeholder
value only to then have our threads overwrite those placeholder values. To do this, we can use the
`sharded-vec-writer` crate, which was created for use in Wild, but which can be used for similar
purposes elsewhere.

First, we create a Vec with sufficient capacity to store the resolutions for all our symbols:

```rust
let mut resolutions: Vec<SymbolId> = Vec::with_capacity(total_num_symbols);
```

At this point, we've allocated space on the heap for the Vec, but that space is still uninitialised.
i.e. the length is still zero.

Next, we create a `VecWriter`, which mutably borrows the Vec, then split that writer into shards,
with each shard having a size equal to the number of symbols in the corresponding object.

```rust
let mut writer = VecWriter::new(&mut resolutions);
let mut shards = writer.take_shards(objects.iter().map(|o| o.num_symbols));
```

We can now, in parallel, iterate through our objects and their corresponding shards and initialise
the shards.

```rust
objects
   .par_iter()
   .zip_eq(&mut shards)
   .for_each(|(obj, shard)| {
      for symbol in obj.symbols() {
         shard.push(...);
      }
   });
```

Lastly, we return the shards to the writer, which verifies that all the shards were fully
initialised, thus resizing the Vec, after which it can be used normally.

```rust
writer.return_shards(shards);
```

## Atomic - non-atomic in-place conversion

Most parts of the linker can make do with either exclusive access to part of the `resolutions` Vec,
or shared access to the entire Vec. However, there's one part of the linker where we need to perform
random writes to the `resolutions` Vec. This is done when we have multiple symbol definitions with
the same name. Originally, I just did this work from the main thread, since I figured most of the
time there would only be a small number of symbols that had the same name. This was mostly true,
however for large C++ binaries like Chromium, it turns out that there are actually a lot of symbols
with the same names, presumably due to C++'s use of header files, which create lots of identical
definitions.

To allow random writes to `resolutions`, we introduce a new type:

```rust
struct AtomicSymbolId(AtomicU32);
```

Being an atomic, we can write to an `AtomicSymbolId` using only a shared (non-exclusive) reference.
However, we need a way to temporarily view our `Vec<SymbolId>` as a `&[AtomicSymbolId]`.

The standard library has something that might help - `AtomicU32::from_mut_slice`:

```rust
fn from_mut_slice(v: &mut [u32]) -> &mut [AtomicU32]
```

However, it's unstable (nightly only). Even if it were stable, it only works with slices of
primitive types, so we'd have to lose our newtypes (SymbolId etc).

Another option would be to always use atomics, however that would quite possibly hurt performance of
the rest of the linker, which doesn't need atomics. It'd also hurt ergonomics, since currently our
`SymbolId`s implement `Copy`, but if they wrapped an `AtomicU32`, then they wouldn't be able to.

A reasonable option at this point would be to resort to unsafe and use something like
`core::mem::transmute`. We'd need to check all the rules and make sure that we were meeting all the
requirements. This is not a bad option, but I personally like the challenge of doing things without
unsafe if I can, especially if I can do so without loss of performance.

Indeed, it turns out that we can, as follows:

```rust
fn into_atomic(symbols: Vec<SymbolId>) -> Vec<AtomicSymbolId> {
   symbols
       .into_iter()
       .map(|s| AtomicSymbolId(AtomicU32::new(s.0)))
       .collect()
}
```

It'd be reasonable to think that this will have a runtime cost, however it doesn't. The reason is
that the Rust standard library has a nice optimisation in it that when we consume a Vec and collect
the result into a new Vec, in many circumstances, the heap allocation of the original Vec can be
reused. This applies in this case. But what even with the heap allocation being reused, we're still
looping over all the elements to transform them right? Because the in-memory representation of an
`AtomicSymbolId` is identical to that of a `SymbolId`, our loop becomes a no-op and is optimised
away.

We can verify this by looking at the assembly produced for this function:

```nasm
movups  xmm0, xmmword, ptr, [rsi]
mov     rax, qword, ptr, [rsi, +, 16]
movups  xmmword, ptr, [rdi], xmm0
mov     qword, ptr, [rdi, +, 16], rax
ret
```

The main takeaway from this assembly is that there's no branching, no looping, just a few moves and
a return. If we allowed this function to be inlined into the caller, it would likely vanish to
nothing.

For conversion back to the non-atomic form, we can do much the same:

```rust
fn into_non_atomic(atomic_symbols: Vec<AtomicSymbolId>) -> Vec<SymbolId> {
   atomic_symbols
       .into_iter()
       .map(|s| SymbolId(s.0.into_inner()))
       .collect()
}
```

The main thing to note here is that we avoid doing an atomic load from the atomic and instead
consume the atomic with `into_inner`. This is easier for the compiler to optimise and if we look at
the assembly produced it's identical to what we got for `into_atomic`.

To actually use these functions, we first need to get ownership of our Vec using `core::mem::take`.
This puts an empty Vec in its place. Empty Vecs don't heap allocate, so this is very cheap. We then
call `into_atomic` to convert the Vec into the form we need.

```rust
let atomic_resolutions = into_atomic(core::mem::take(&mut self.resolutions));
```

We can then do whatever parallel processing we need with the Vec in its atomic form.

```rust
process_resolutions_in_parallel(&atomic_resolutions);
```

Finally, we convert back to the original non-atomic form and store back where we got it from,
overwriting the empty Vec that we temporarily put in its place.

```rust
self.resolutions = into_non_atomic(atomic_resolutions);
```

One thing worth noting here is that if we panic (or do an early return), we might leave
`self.resolutions` as the empty Vec. This isn't a problem in the linker, since if we're returning an
error or have hit a panic, then we don't care at that point about resolutions. It would be possible
to ensure that the proper Vec was restored for use-cases where that was important, however it would
add extra complexity and might be enough to convince me that it'd be better to just use transmute.

## Buffer reuse

Doing too much heap allocation tends to hurt performance. A common trick is to move heap allocations
outside of loops. For example, rather than this:

```rust
loop {
    let mut buffer = Vec::new();
    // Do work with `buffer`.
}
```

We might prefer to allocate buffer before the loop, then just clear it inside the loop:

```rust
let mut buffer = Vec::new();
loop {
    buffer.clear();
    // Do work with `buffer`.
}
```

However, if we're storing something into a Vec that has a non-static lifetime, then we can run into
problems. Here, we have a variable `text`, which holds a `String`. We then split that string and
store the resulting string-slices into `buffer`. Even though we clear `buffer` at the end of the
loop, the compiler is unhappy. It wants `text` to outlive `buffer` because we're storing references
to `text` into `buffer`.

```rust
let mut buffer = Vec::new();
loop {
    let text = get_text();
    buffer.extend(text.split(","));
    // Do work with `buffer`.
    buffer.clear();
}
```

We could at this point give up and just move our Vec creation back inside the loop. However, it
turns out that there's another solution.

```rust
fn reuse_vec<T, U>(mut v: Vec<T>) -> Vec<U> {
   v.clear();
   v.into_iter().map(|x| unreachable!()).collect()
}
```

The idea of this function is to convert from a Vec of some time to an empty Vec of another type,
reusing the heap allocation. This works in a very similar way to how we converted between atomic and
non-atomic `SymbolId`s, except this time because we first clear the Vec, the body of our `map`
function is unreachable.

The optimisation in the Rust standard library that allows reuse of the heap allocation will only
actually work if the size and alignment of `T` and `U` are the same, so let's verify that that's the
case. We can do the check at compile time, so if we accidentally call this function with
incompatible `T` and `U`, we'll get a compilation error at the call site.

```rust
fn reuse_vec<T, U>(mut v: Vec<T>) -> Vec<U> {
   const {
       assert!(size_of::<T>() == size_of::<U>());
       assert!(align_of::<T>() == align_of::<U>());
   }
   v.clear();
   v.into_iter().map(|_| unreachable!()).collect()
}
```

Let's verify that this optimises as we expect:

```nasm
mov     qword, ptr, [rsi, +, 16], 0
movups  xmm0, xmmword, ptr, [rsi]
movups  xmmword, ptr, [rdi], xmm0
mov     qword, ptr, [rdi, +, 16], 0
ret
```

More or less the same assembly as before, except that we're now setting the length of the Vec to 0.
Note, that the loop and the panic from the use of `unreachable!` are gone.

We can now integrate this into our previous code as follows:

```rust
let mut buffer_store: Vec<&str> = Vec::new();
loop {
    let mut buffer = reuse_vec(buffer_store);
    let text = get_text();
    buffer.extend(text.split(","));
    // Do work with `buffer`.
    buffer_store = reuse_vec(buffer);
}
```

Effectively, each time around the loop we move out of `buffer_store`, converting the type of the
`Vec`, use it for a bit, then convert it back and store it again in `buffer_store`. The only time
we'll need a new heap allocation is when our `Vec` needs to grow. The types of `buffer_store` and
`buffer` are both `Vec<&str>`, however the lifetime of the references is different.

## Deallocation on a separate thread

Freeing memory is generally a lot slower than allocating it. If we've done a very large allocation,
it can sometimes be worthwhile passing it to another thread to free it, so that we can get on with
other work.

For example, if using rayon, we might use `rayon::spawn` to spawn a task that drops our buffer:

```rust
fn process_buffer(buffer: Vec<u8>) {
   // Do some work with `buffer`.

   rayon::spawn(|| drop(buffer));
}
```

Note, that `rayon::spawn` itself does a heap allocation, so this would only be worthwhile if
`buffer` was potentially very large. This is definitely something you'd want to benchmark to see if
it actually improves the runtime for your use-case. There is at least one place in the Wild linker
where we did this and it did give a measurable reduction in runtime.

Similar to buffer reuse, if our heap allocation has non-static lifetimes associated with it, we can
get rid of them using `reuse_vec`.

```rust
fn process_buffer(names: Vec<&[u8]>) {
   // Do some work with `names`.

   let names: Vec<&[u8]> = reuse_vec(names);
   rayon::spawn(|| drop(names));
}
```

In this case, we're converting the `Vec` from a `Vec<&[u8]>` to  `Vec<&'static [u8]>`.

## Bonus: Strip lifetime with non-trivial Drop

This is a bonus tip that wasn't included in the talk and builds on the previous tip and is in
response to a question by VorpalWay on Reddit. If you want to drop a `Vec<T>` and `T` has both a
non-static lifetime and a non-trivial `Drop`, then things get slightly more tricky. The trick here
is to convert to a struct that is the same as `T`, but has non-static references replaced with
`MaybeUninit`.

For example, suppose we have the following struct:

```rust
pub struct Foo<'a> {
    owned: String,
    borrowed: &'a str,
}
```

We can define a new struct:

```rust
struct StaticFoo {
    owned: String,
    borrowed: MaybeUninit<&'static str>,
}
```

We can then convert our Vec to the new type with zero cost and no unsafe:

```rust
fn without_lifetime(foos: Vec<Foo>) -> Vec<StaticFoo> {
    foos.into_iter()
        .map(|f| StaticFoo {
            owned: f.owned,
            borrowed: MaybeUninit::uninit(),
        })
        .collect()
}
```

The presence of `MaybeUnit::uninit()` tells the compiler that it's OK to have anything there, so it
can choose to leave whatever `&str` was in the original `Foo` struct. This means that it's valid to
produce a `StaticFoo` with the same in-memory representation as the `Foo` that it replaces, allowing
it to eliminate the loop. The asm for this function is:

```nasm
 movups  xmm0, xmmword, ptr, [rsi]
 mov     rax, qword, ptr, [rsi, +, 16]
 movups  xmmword, ptr, [rdi], xmm0
 mov     qword, ptr, [rdi, +, 16], rax
 ret
```

i.e. the loop was indeed eliminated.

Now that we have a Vec with no non-static lifetimes, we can safely move it to another thread.

# Thanks

Thanks to my [github sponsors](https://github.com/sponsors/davidlattimore). Your contributions help
to make it possible for me to continue to work on this kind of stuff rather than going and getting a
"real job".

* CodeursenLiberte
* Urgau
* pmarks
* repi
* embark-studios
* mati865
* bes
* joshtriplett
* mstange
* bcmyers
* Rafferty97
* acshi
* Kobzol
* flba-eb
* jonhoo
* marxin
* tommythorn
* binarybana
* teburd
* bearcove
* yerke
* teh
* twilco
* Shnatsel
* coastalwhite
* wezm
* davidcornu
* gendx
* rrbutani
* nazar-pc
* willstott101
* tatsuya6502
* teohhanhui
* jkendall327
* EdorianDark
* drmason13
* HadrienG2
* jplatte
* rukai
* ymgyt
* dream-dasher
* alexkirsz
* Pratyush
* Tudyx
* coreyja
* dralley
* irfanghat
* mvolfik
* simtheverse

## Discussion

* [Reddit](https://www.reddit.com/r/rust/comments/1n7814i/wild_performance_tricks/)
