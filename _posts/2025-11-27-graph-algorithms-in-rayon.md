---
layout: default
title:  "Graph Algorithms in Rayon"
date:   2025-11-27 00:00:00
categories: posts
---

The [Wild linker](https://github.com/davidlattimore/wild) makes very extensive use of
[rayon](https://docs.rs/rayon/latest/rayon/) for parallelism. Much of this parallelism is in the
form of
[`par_iter`](https://docs.rs/rayon/latest/rayon/iter/trait.IntoParallelRefIterator.html#tymethod.par_iter)
and friends. However, some parts of the linker don’t fit neatly because the amount of work isn’t
known in advance. For example, the linker has two places where it explores a graph. When we start,
we know some roots of that graph, but we don’t know all the nodes that we’ll need to visit. We’ve
gone through a few different approaches for how we implement such algorithms. This post covers those
approaches and what we’ve learned along the way.

## Spawn broadcast

Our first approach was to spawn a task for each thread (rayon’s
[spawn\_broadcast](https://docs.rs/rayon/latest/rayon/struct.Scope.html#method.spawn_broadcast))
then do our own work sharing and job control between those threads. By “our own job control” I mean
that each thread would pull work from a channel and if it found no work, it’d [park the
thread](https://doc.rust-lang.org/std/thread/fn.park.html). If new work came up, the thread that
produced the work would wake a parked thread.

This was complex. Worse, it didn’t allow us to use other rayon features while it was running. For
example, if we tried to do a par\_iter from one of the threads, it’d only have the current thread to
work with because all the others were doing their own thing, possibly parked, but in any case, not
available to rayon.

## Scoped spawning

Using rayon’s [`scope`](https://docs.rs/rayon/latest/rayon/fn.scope.html) or
[`in_place_scope`](https://docs.rs/rayon/latest/rayon/fn.in_place_scope.html), we can create a scope
into which we spawn tasks.

```rust
rayon::scope(|scope| {  
  for node in roots {  
    scope.spawn(|scope| {  
      explore_graph(node, scope);  
    });  
  }  
});  
```

The idea here is that we create a scope and spawn some initial tasks into that scope. Those tasks
then spawn additional tasks and so on until eventually there are no more tasks.

The rayon documentation warns that this is more expensive than other approaches, so should be
avoided if possible. The reason it’s more expensive is that it heap-allocates the task. Indeed, when
using this approach, we do see increased heap allocations.

## Channel \+ par\_bridge

Another approach that I’ve tried recently and which arose out of the desire to reduce heap
allocations is to put work into a [crossbeam
channel](https://docs.rs/crossbeam-channel/latest/crossbeam_channel/fn.unbounded.html). The work
items can be an enum if there are different kinds. Our work scope is then just something like the
following:

```rust
let (work_send, work_recv) = crossbeam_channel::unbounded();

// Add some initial work items.  
for node in roots {  
  work_send.send(WorkItem::ProcessNode(node, work_send.clone()));  
}

// Drop sender to ensure we can terminate. Each work item has a copy of the sender.  
drop(work_send);

work_recv.into_iter().par_bridge().for_each(|work_item| {  
   match work_item {  
      WorkItem::ProcessNode(node, work_send) => {  
        explore_graph(node, work_send);  
      }  
   }  
});  
```

The trick with this approach is that each work item needs to hold a copy of the send-end of the
channel. That means that when processing work items, we can add more work to the queue. Once the
last work item completes, the last copy of the sender is dropped and the channel closes.

This approach works OK. It does avoid the heap allocations associated with scoped spawning. It is a
little bit complex, although not as complex as doing all the job control ourselves. One downside is
that like doing job control ourselves, it doesn’t play nicely with using `par_iter` inside of worker
tasks. The reason why is kind of subtle and is due to the way rayon is implemented. What can happen
is that the `par_iter` doesn’t just process its own tasks. It can also steal work from other
threads. When it does this, it can end up blocking trying to pull another work item from the
channel. The trouble is that because the `par_iter` was called from a work item that holds a copy of
the send-end of the channel, we can end up deadlocked. The channel doesn’t close because we hold a
sender and we don’t drop the sender because we’re trying to read from the read-end of the channel.

Another problem with this approach that I’ve just come to realise is that it doesn’t compose well. I
had kind of imagined just getting more and more options in my `WorkItem` enum as the scope of the
work increased. The trouble is that working with this kind of work queue doesn’t play nicely with
the borrow checker. An example might help. Suppose we have some code written with rayon’s
[par\_chunks\_mut](https://docs.rs/rayon/latest/rayon/slice/trait.ParallelSliceMut.html#method.par_chunks_mut)
and we want to flatten that work into some other code that uses a channel with work items. First we
need to convert the `par_chunks_mut` code into a channel of work items.

```rust
let foo = create_foo();  
foo.par_chunks_mut(chunk_size).for_each(|chunk| {  
   // Do work with mutable slice `chunk`  
});  
```

If we want the creation of `foo` to be a work item and each bit of processing to also be work items,
there’s no way to do that and have the borrow checker be happy.

```rust
match work_item {  
   WorkItem::CreateAndProcessFoo => {  
      let foo = create_foo();  
      // Split `foo` into chunks and queue several `WorkItem::ProcessChunk`s….?  
   }  
   WorkItem::ProcessChunk(chunk) => {  
      // Do work with mutable slice `chunk`.  
   }  
}  
```

So that clearly doesn’t work. There’s no way for us to take our owned `foo` and split it into chunks
that can be processed as separate `WorkItem`s. The borrow checker won’t allow it.

Another problem arises if we’ve got two work-queue-based jobs and we’d like to combine them, but the
second job needs borrows that were taken by the first job to be released before it can run. This
runs into similar problems.

The kinds of code structures we end up with here feel a bit like we’re trying to write async code
without async/await. This makes me wonder if async/await could help here.

## Async/await

I don’t know exactly what this would look like because I haven't yet tried implementing it. But I
imagine it might look a lot like how the code is written with rayon’s scopes and spawning. Instead
of using rayon’s scopes, it’d use something like
[`async_scoped`](https://crates.io/crates/async-scoped).

One problem that I have with rayon currently is, I think, solved by using async/await. That problem,
which I briefly touched on above is described in more detail here. Suppose we have a `par_iter`
inside some other parallel work:

```rust
outer_work.par_iter().for_each(|foo| {
  let foo = inputs.par_iter().map(|i| ...).collect();

  // < Some other work with `foo` here, hence why we cannot merge the two par_iters >

  foo.par_iter().map(|i| ...).for_each(|i| ...);
});
```

If the thread that we're running this code on becomes idle during the first inner `par_iter`, that
thread will try to steal work from other threads. If it succeeds, then even though all the work of
the `par_iter` is complete, we can't continue to the second inner `par_iter` until the stolen work
also completes. However, with async/await, tasks are not tied to a specific thread once started.
Threads steal work, but tasks don't, so the task that's running the above code would become runnable
as soon as the `par_iter` completed even if the thread that had originally been running that task
had stolen work - the task could just be run on another thread.

It'd be very interesting to see what async/await could contribute to the parallel computation space.
I don’t have any plans to actually try this at this stage, but maybe in future.

## Return to scoped spawning and future work

In the meantime, I’m thinking I’ll return to scoped spawning. Using a channel works fine for simple
tasks and it avoids the heap allocations, but it really doesn’t compose at all well.

I am interested in other options for avoiding the heap allocations. Perhaps there’s options for
making small changes to rayon that might achieve this. e.g. adding support for spawning tasks
without boxing, provided the closure is less than or equal to say 32 bytes. I’ve yet to explore such
options though.

## Thanks

Thanks to everyone who has been [sponsoring](https://github.com/sponsors/davidlattimore) my work on
Wild, in particular the following, who have sponsored at least $15 in the last two months:

* CodeursenLiberte
* repi
* rrbutani
* Rafferty97
* wasmerio
* mati865
* Urgau
* mstange
* flba-eb
* bes
* Tudyx
* twilco
* sourcefrog
* simonlindholm
* petersimonsson
* marxin
* joshtriplett
* coreyja
* binarybana
* bcmyers
* Kobzol
* HadrienG2
* +3 anonymous

## Discussion threads

* [Reddit](https://www.reddit.com/r/rust/comments/1p7omoh/thoughts_on_graph_algorithms_in_rayon/)
