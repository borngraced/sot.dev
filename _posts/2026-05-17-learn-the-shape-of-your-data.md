---
layout: post
title: "Learn the Shape of Your Data"
description: "Most programming tutorials start with syntax, but the real language starts with data layout, ownership, allocation, and what values actually look like in memory."
date: 2026-05-19
---

Most programming tutorials start in the same place: variables, loops, functions, modules, package managers, then a tiny CLI or todo app, and that is fine if the goal is to get someone to print something on the screen, but if the goal is to really understand a language, especially Rust, Go, Zig, C, C++, or even Swift, syntax is not enough.

If I could go back and teach myself again, I would start with one question: what does this value actually look like in memory? Not in a scary compiler-engineer way, just enough to know what is inline, what is a pointer, what allocates, what gets copied, what moves, who owns the memory, and when that memory goes away.

A little about my background: I started coding, I would not even call it programming then, when I was around 12 or 13, I was modding Java-based Nokia apps like `2go`, building HTML/JavaScript pages and forums on platforms like `Wapka.Mobi` and `JohnCMS`, also writing small `php` scripts and doing web stuff, all for fun in high school, never thinking it would become my actual work because I wanted to be an `Astronaut` xDDDD.

After high school, I tried to learn programming properly through online courses, PDFs, YouTube videos, the usual route, and I could write logic, run it, and feel happy when it worked, but I did not really understand what was happening under the hood until I was about to learn `Rust`. It was hard for me so I stopped trying to memorize language rules and started asking better questions like, how does the computer execute my program? What does my English looking code become? Where does the data live? What does the runtime do for me?

Those questions led me into a big research and I started reading books like, `Introduction to Computer Organization`, `Introduction to Computer Systems: From Bits & Gates to C/C++ & Beyond`, `Hacker's Delight`, `Thinking Low Level, Writing High Level`, ARM assembly, compiler books like `Writing a C Compiler` and `Modern Compiler Implementation in C`, plus a lot of blogs, PDFs, and Reddit threads. After writing a lot of `C` and some worthy amount of `arm assembly`, I picked up `Rust` again and suddenly things started clicking and I didn't need to stress my brain or think too much since I now have the right mental model. I can keep talking more on this but I don't want this article to get longer that I want it to be so I'll stop now.

*Fun fact:* during high school, I did most of that modding and web stuff on my `Nokia Symbian` and `Nokia Asha 200`. It was really fun xD.

I do not mean every beginner needs to read a compiler book before writing `hello world` instead, I mean the basic questions that explain why the language behaves the way it behaves:

- What does this value look like in memory?
- What lives inline?
- What is just a pointer?
- What allocates?
- What gets copied?
- What gets moved?
- Who owns the allocation?
- When does the allocation go away?
- What is a function?

These questions matter more than most tutorials admit.

## A String Is Not Just Text

Take Rust:

```rust
let name = String::from("khoomi");
```

A lot of tutorials will say this creates a string, and that is true, but is that enough?

The useful explanation is that `name` itself is a small value, usually a `pointer`, a `length`, and a `capacity`; that small value lives wherever the variable lives, while the actual bytes for `"khoomi"` live somewhere else, usually on the heap. So the `String` value is not the text, it is more like a handle to the text.

Once that clicks, Rust ownership starts feeling much less mystical. Moving a `String` does not copy all the bytes. It copies the small header and transfers responsibility for freeing the heap allocation. Borrowing a `&str` is not "some lifetime magic." It is a view into bytes owned by something else. This one mental model explains more than a dozen abstract examples.

It explains why this is different:

```rust
let a = String::from("hello");
let b = a;
```

from this:

```rust
let a = "hello";
let b = a;
```

In the first case, ownership of a heap allocation moves, while in the second case you are copying a reference to static data, so even though the syntax looks simple in both cases, the memory story is different.

That memory story is the language.

## Go Slices Have the Same Problem

Go has a similar problem with slices.

People are taught:

```go
items := []int{1, 2, 3}
```

Then they learn `append`, `range`, and function parameters, but many beginners are not told early enough that a slice is not the array.

A slice is a small descriptor: pointer, length, capacity. The backing array is somewhere else, and passing a slice to a function copies the descriptor, not the entire array, which explains why a function can mutate the underlying elements:

```go
func change(xs []int) {
    xs[0] = 99
}
```

But it also explains why `append` can become confusing:

```go
func add(xs []int) {
    xs = append(xs, 4)
}
```

Depending on capacity, `append` may reuse the same backing array or allocate a new one, and if you do not understand the descriptor/backing-array split, this feels like Go being inconsistent. It is not inconsistent, you are just missing the shape of the data, and again, the memory model explains the behavior better than memorizing rules.

## This Is Why People Struggle With "Advanced" Features

A lot of supposedly advanced language features become less scary once the data layout is clear. Rust lifetimes are easier when you understand references as borrowed views into data owned elsewhere, `Box<T>` is easier when you see it as a pointer-sized owner of heap storage, and `Rc<T>` or `Arc<T>` starts making sense once reference counts and shared heap ownership stop being abstract words.

The same thing happens with trait objects when you understand fat pointers and dynamic dispatch(vtables), with Go interfaces when you understand that an interface value carries type information and a value pointer, with Go maps when you realize they are runtime-managed hash tables, not ordinary values copied around like structs, with closures when you understand what gets captured and whether it escapes, and with async when you understand that local state may be transformed into a state machine that has to live across suspension points.

None of this means you need to be a compiler engineer, it just means you need to know what kind of thing you are holding.

## Stack vs Heap Is Not Trivia

I think treating stack vs heap as a low-level detail you can learn later is backward. You do not need to obsess over it, but you should understand the basic tradeoff early.

`Stack allocation` is usually cheap, scoped, and straightforward, while `Heap allocation` is more flexible but involves the allocator, indirection, and eventual cleanup. In garbage-collected languages, cleanup is handled differently, but the allocation is still real, because the GC does not make allocations free, it just changes who pays the bill and when.

This is especially important in Go because escape analysis decides whether some values can stay on the stack or must move to the heap.

You can write perfectly normal-looking Go code and accidentally allocate much more than expected:

```go
func ptr() *int {
    x := 10
    return &x
}
```

The compiler is smart enough to make this safe because `x` escapes; it cannot live on the stack frame that is about to disappear, so it must live somewhere else.

That is good, but if you never learn escape analysis, you end up surprised by allocations, latency, and GC pressure.

The same applies to closures, interface boxing, goroutines, and large values passed around carelessly. You can still write the code, but you do not really understand the cost model.

## Syntax Can Lie

One reason this matters is that syntax often hides real cost, so what looks simple on the page may not be simple at runtime.

This looks small, but it allocates:

```rust
let xs = vec![1, 2, 3, 4, 5];
```

This looks harmless, but depending on `value`, it may involve interface conversion, reflection, allocation, formatting cost, and I/O:

```go
fmt.Println(value)
```

This looks like just passing a value, but maybe you are copying more than you think:

```go
func handle(v BigStruct) {}
```

This looks like just adding to a list, but maybe the vector has to grow, allocate a new buffer, and move the existing elements:

```rust
items.push(x);
```

Syntax is the surface and the runtime behavior is underneath. Good programmers do not need to think about the underneath every second, I mean that would be exhausting, but they should at least have a rough mental model, so when something gets slow, confusing, or unsafe, they know where to look.

## Tutorials Should Teach Data Layout Earlier

I am not saying every language course should start with a dump of ABI rules, CPU cache lines, pointer provenance, and allocator internals, that would be its own kind of bad teaching, but I do think tutorials should introduce data layout much earlier than they usually do.

When teaching Rust `String`, show the pointer/length/capacity model; when teaching `Vec<T>`, show that the buffer is separate from the small vector header; when teaching `&str`, explain that it is a borrowed view, not an owned string.

When teaching Go slices, show the slice header and backing array; when teaching Go maps, explain that maps are references to runtime-managed hash tables; when teaching interfaces, explain the type/value pair.

When teaching structs, explain padding and alignment at least once, and when teaching async, explain that state has to survive across awaits.

This is the difference between "I know the syntax" and "I understand what my program is doing."

## The Real Learning Path

For systems-ish languages, I think the path should look more like this:

1. Values and memory layout.
2. Ownership or lifetime rules.
3. Allocation and deallocation.
4. References, pointers, and views.
5. Collections and their internal shape.
6. Error handling.
7. Modules and packages.
8. Concurrency and async.
9. Performance tools.

Most tutorials do something closer to:

1. Variables.
2. Functions.
3. Loops.
4. Structs.
5. Collections.
6. Maybe memory later if things get confusing.

The problem is that memory is not a side quest but the thing making the rest of the language make sense.

## The Point

I think this is why some people can write Rust for months and still feel like they are negotiating with the borrow checker instead of understanding it, and it is also why some people write Go for years and still get surprised by slices, interfaces, and allocation spikes. They were taught the grammar, but not the shape of the values.

Learning a language properly means learning what the code means to the machine, not every detail and not all at once, but enough that the behavior stops feeling random.

Syntax tells you what you are allowed to write and the memory model tells you what you actually wrote.

I personally recommend [https://rust-book.cs.brown.edu](https://rust-book.cs.brown.edu) for learning Rust as a beginner or someone who only `knows` but don't really understand it.
