---
layout: post
title: "What Happens When You Build an Inode-Style Vector in Rust"
description: "I built an inode-inspired paged array in Rust, benchmarked it against Vec and SmallVec, and found exactly where the idea falls apart."
image: https://sot.dev/assets/images/inode-style-vector-in-rust.png
date: 2026-04-22
---

While taking some lectures on the Linux filesystem, I got particularly interested in how the inode stores metadata and points to data blocks. I find that whole design very elegant. It solves a real storage problem with a layout that is clearly optimized for the constraints of the filesystem rather than for raw contiguous access. After staring at it for a while, I had the obvious bad idea: what if I translated the same storage model into a custom Rust array? Would it be faster than `SmallVec`? Faster than native `Vec`? Or would it be the exact kind of cache-unfriendly pointer-chasing machine I already suspected it would be?

The higher-level answer was already obvious to me before I wrote any code. A contiguous array exists for a reason. CPU caches exist for a reason. Filesystem storage layouts and in-memory array layouts are solving very different problems. But some ideas are worth implementing precisely because you know they are probably wrong. They force you to stop hand-waving and actually measure where the machine starts getting annoyed.

## The Basic Idea

The shape I wanted was:

```rust
pub struct PagedSmallVec<
    T,
    const INLINE: usize,
    const DIRECT: usize,
    const CHUNK: usize,
> {
    direct: [Option<Box<[MaybeUninit<T>; CHUNK]>>; DIRECT],
    inline: [MaybeUninit<T>; INLINE],
    len: usize,
}
```

The plan was simple. Keep a small inline buffer for the first few elements, once that fills up, spill into fixed-size direct chunks instead of one large contiguous heap allocation. Later, if the experiment felt promising, I could extend it into double-indirect and triple-indirect tiers like an inode table.

At a conceptual level, it sounds reasonable: small data stays inline, growth past inline avoids large contiguous reallocations, and the design can scale to multiple indirection levels. It is exactly the kind of idea that sounds smarter in theory than it turns out to be on a CPU.

## Stage One: Just Inline + Direct Chunks

I intentionally kept the first version simple. So there's no double-indirect and triple-indirect. Just enough to see whether the first stage had any chance at all.

The hot indexing path after the inline region is basically:

```rust
let paged_index = index - INLINE;
let chunk_index = paged_index / CHUNK;
let offset = paged_index % CHUNK;
```

Then read from:

```rust
self.direct[chunk_index][offset]
```

So stage one is not conceptually complicated. It is just an inline array plus a direct chunk table. That means if performance is bad, it is not because the logic is too intellectually deep. It is because every extra indirection, branch, and lookup is costing something real.

## The First Real Problem: Unsafe Code

The moment you back a container with `MaybeUninit<T>`, the project stops being "write a cute data structure" and becomes "maintain invariants without lying to yourself."

The early versions had all the predictable issues:

```rust
pub fn push(&mut self, val: T) {
    self.write_slot(self.len, val);
    self.len += 1;
}

fn write_slot(&mut self, index: usize, val: T) {
    // ...
    self.len += 1; // bug
}
```

That kind of thing immediately corrupts logical state.

I also had the usual problems around:

```rust
fn take_slot(&mut self, index: usize) -> T {
    unsafe { self.inline[index].assume_init_read() }
}
```

That is only sound if `index < len`, the slot is initialized, and length tracking is correct.

And drop is another place where this gets real very quickly:

```rust
impl<T, const INLINE: usize, const DIRECT: usize, const CHUNK: usize> Drop
    for PagedSmallVec<T, INLINE, DIRECT, CHUNK>
{
    fn drop(&mut self) {
        for i in 0..self.len {
            // drop initialized slots only
        }
    }
}
```

Once I started fixing those bugs, it became obvious that the unsafe part of this project was not some tiny implementation detail. It was the foundation. If length tracking is wrong by even one slot, everything after that becomes suspect.

## The Initial API Looked Like a Normal Vec

The first public API looked like exactly what you would expect:

```rust
impl<T, const INLINE: usize, const DIRECT: usize, const CHUNK: usize>
    PagedSmallVec<T, INLINE, DIRECT, CHUNK>
{
    pub fn push(&mut self, val: T) { /* ... */ }
    pub fn pop(&mut self) -> Option<T> { /* ... */ }
    pub fn get(&self, index: usize) -> Option<&T> { /* ... */ }
    pub fn remove(&mut self, index: usize) -> Option<T> { /* ... */ }
    pub fn swap_remove(&mut self, index: usize) -> Option<T> { /* ... */ }
}
```

That was fine for usability, but it hid the real issue: this structure is not naturally contiguous. So whenever I used it exactly like `Vec`, I was forcing it into an access pattern that favored the standard containers.

## Benchmarks: Vec Usually Won, SmallVec Usually Came Second

I benchmarked the structure against `Vec<T>`, `SmallVec<[T; N]>`, and `PagedSmallVec<T, INLINE, DIRECT, CHUNK>` across `append_only`, `append_pop`, `full_iteration`, `random_indexing`, `remove`, and `swap_remove`, using `u32`, `[u8; 64]`, and `String` as the test element types. The benchmark harnesses are in the repo too, mainly [`benches/compare.rs`](https://github.com/borngraced/paged-small-vec/blob/master/benches/compare.rs) and [`benches/push_u32.rs`](https://github.com/borngraced/paged-small-vec/blob/master/benches/push_u32.rs).

The broad result was not subtle.

For normal vector-shaped workloads, `Vec` usually came first, `SmallVec` usually came second, and the paged structure usually came third. For cheap scalar types like `u32`, the gap was especially ugly. That was not surprising once I thought about it properly. A contiguous array is doing exactly what the CPU wants: predictable memory access, fewer pointer hops, fewer branches, fewer cache misses. My paged structure was paying for chunk math and chunk lookups just to reach data that `Vec` could reach by simple pointer arithmetic.

## Random Access Was Exactly As Bad As It Should Have Been

The indexed access path looked like this:

```rust
pub unsafe fn get_unchecked(&self, index: usize) -> &T {
    if index < INLINE {
        return self.inline.get_unchecked(index).assume_init_ref();
    }

    let paged_index = index - INLINE;
    let chunk_index = paged_index / CHUNK;
    let offset = paged_index % CHUNK;

    self.direct
        .get_unchecked(chunk_index)
        .as_ref()
        .unwrap_unchecked()
        .get_unchecked(offset)
        .assume_init_ref()
}
```

There is nothing shocking about why this loses to `Vec` on random indexing. Even the "simple" stage-one version still has an inline-vs-paged branch, subtraction, division, modulo, a chunk lookup, and a slot lookup. That is simply more work than contiguous memory.

## Ordered Remove Did Not Magically Get Better

This was one place where I think it is easy to fool yourself.

You might look at paged storage and think, "maybe middle removal is cheaper because I am moving chunks, not a whole buffer."

Not really.

If you want order-preserving `remove`, you still conceptually shift the whole tail:

```rust
pub fn remove(&mut self, index: usize) -> Option<T> {
    let removed = self.take_slot(index);

    for i in (index + 1)..self.len {
        let val = self.take_slot(i);
        self.write_slot(i - 1, val);
    }

    self.len -= 1;
    Some(removed)
}
```

That is still O(n), and in my case it was slower than `Vec` and `SmallVec` because the contiguous containers get to do their movement in a layout that is much friendlier to the CPU.

`swap_remove` was less terrible, but again the paged structure did not become some surprise winner. It just became less disadvantaged.

## The First Thing That Actually Helped: Chunk-Native Iteration

The first genuinely good result came when I stopped traversing the structure through repeated `get(i)`.

My original full iteration benchmark looked like this:

```rust
for i in 0..vec.len() {
    sum += vec.get(i).unwrap().score();
}
```

That was a bad fit for the layout. Sequential traversal through a paged structure should not pretend to be repeated random indexing.

So I added:

```rust
pub fn for_each_chunk(&self, mut f: impl FnMut(&[T])) {
    for chunk in self.chunks() {
        f(chunk);
    }
}
```

and:

```rust
pub fn for_each_ref(&self, mut f: impl FnMut(&T)) {
    self.for_each_chunk(|chunk| {
        for value in chunk {
            f(value);
        }
    });
}
```

That changed the picture a lot. Once full scans were expressed chunk-by-chunk, the structure became much more competitive for sequential traversal. That was the point where the design stopped feeling like "a bad Vec" and started feeling like "a different structure with a different natural API."

## I Added Chunk Iterators and Batch Append Too

Once chunk traversal proved useful, it made sense to expose it directly:

```rust
pub struct ChunkIter<'a, T, const INLINE: usize, const DIRECT: usize, const CHUNK: usize> {
    vec: &'a PagedSmallVec<T, INLINE, DIRECT, CHUNK>,
    remaining: usize,
    next_chunk_index: usize,
    yielded_inline: bool,
}
```

And for append-heavy work, I added:

```rust
pub fn extend_from_slice(&mut self, values: &[T])
where
    T: Clone,
{
    // fill inline first, then write chunk by chunk
}
```

This fit the layout better than repeating `push()` in a loop, even when it was not yet a dramatic performance win.

That was another important lesson: a good API does not need to beat the standard one immediately to still be the right API for the structure.

## Then I Optimized the Hot Append Path

Append was an obvious place to focus next, so I added:

```rust
tail_chunk_index: usize,
tail_offset: usize,
current_chunk_ptr: *mut MaybeUninit<T>,
```

The idea was to cache the logical tail position, cache a raw pointer to the current chunk, only allocate the next chunk when entering it, and make steady-state append look like this:

```rust
unsafe { (*self.current_chunk_ptr.add(self.tail_offset)).write(val) };
```

That helped append-heavy paths. Not enough to beat `Vec`, but enough to remove some of the overhead from repeatedly going through more abstract chunk lookup logic on every write.

## Chunk Size Was Not a One-Size-Fits-All Story

I also turned chunk size into a const generic and benchmarked `64`, `128`, `256`, and `512`.

What I found was not "bigger is always better." For `u32`, bigger chunks often helped batch append and sometimes helped append-heavy workloads. In [`extend_micro/u32`](https://github.com/borngraced/paged-small-vec/blob/master/benches/push_u32.rs), for example, `256` reached about `576 Melem/s` while `512` reached about `609 Melem/s`. But for pure `push` and `push+pop`, `256` came out as the better compromise: in [`push_only_micro/u32`](https://github.com/borngraced/paged-small-vec/blob/master/benches/push_u32.rs), `256` was about `382 Melem/s` while `512` was about `342 Melem/s`, and in [`push_pop_micro/u32`](https://github.com/borngraced/paged-small-vec/blob/master/benches/push_u32.rs), `256` was about `326 Melem/s` while `512` was about `314 Melem/s`.

For `[u8; 64]`, larger chunks did not help nearly as much, and for some paths they were worse. For `String`, the differences were smaller and more workload-dependent.

So I settled on `256` as the default chunk size because the current container is still more push/pop-shaped than purely bulk-append-shaped.

## The Real Lesson

At this point I think the biggest mistake would be judging the structure only by how well it mimics `Vec`.

Its natural APIs are not primarily `get`, `remove`, and random indexing. They are closer to `for_each_chunk`, chunk iterators, batch append, and chunk-wise transforms. That is where the layout starts making sense. Once I stopped forcing the structure through access patterns optimized for contiguous memory, the experiment became much more informative.

## My Verdict

If your goal is to beat `Vec`, beat `SmallVec`, win scalar `push` and `pop`, or win random indexing, inode-style array storage in Rust is a bad idea.

If your goal is to understand low-level container invariants, see exactly where pointer-heavy layouts lose, experiment with chunk-native APIs, and learn where contiguous storage wins and why, then it is a very good experiment.

So my final verdict is this: judged as a traditional vector, it did not do well but judged as an experiment, it went very well, because it made the trade-offs impossible to ignore. And honestly, that is usually the most useful outcome from a bad idea that was worth building anyway.

If you want to read through the full implementation, here is the code on GitHub: [borngraced/paged-small-vec](https://github.com/borngraced/paged-small-vec).
