---
layout: post
title:  "Async Explorer"
date:   2020-06-19 16:24:08 +0300
categories: jekyll update
tags: async
---

Have you ever looked up into the stars and wondered, "How the fuck is that feature implemented"? In this series, I'll (hopefully) dive into the implementation of coroutines for several (compiled) programming languages.

A short disclaimer: I'm not too sharp on the details of some (or actually any) of these implementations. Most of this will just be me rambling and looking at compiler source code/the output of [the Godbolt Compiler Explorer](https://godbolt.org/). I'll try to validate every claim I'll post here, but some mistakes are sure to sneak their way into one of these. Feel free to point them up and I'll fix them as soon as I can.

## Purpose

The purpose of this post is to set a few definitions for my upcoming posts. I'll explain what I mean when I use words like "coroutine" or "asynchronous", which will hopefully allow me to use them freely in the following posts.

Afterwards, I'll give a few short examples of how to coroutines "look like" in several programming languages, like C++, Rust, Zig and the LLVM IR. 

I hope I'll also be able to give you a rough idea of what coroutines are usful for, but it's not the main goal of this post.

This is not an "asyncio" tutorial or anything like that, just me messing with some programming languages.

## What is async?

If you're like me, you probably heard terms like "async" and "coroutines" mentioned a lot, and maybe even read (or wrote) some asynchronious code, but didn't really understand what's going on behind the scenes. At least I didn't (and still don't). This series of posts will try to fix that!

Let's consult [Wikipedia](https://en.wikipedia.org/wiki/Coroutine)!

> Coroutines are computer program components that generalize subroutines for [non-preemptive multitasking](https://en.wikipedia.org/wiki/Cooperative_multitasking), by allowing execution to be suspended and resumed. 

Let's break it down a little.

For me, a subroutine is a piece of code that accepts parameters, and "executes". That is, processes that parameters, produces side effects, and returns a value. Now, according to Wikipedia, a coroutine is a subroutine that can be "suspended and resumed". The way I understand that, I'm getting a picture of a subroutine "taking a nap", which we can later wake up and continue. Notice that I'm explicitly choosing to not mention [multithreading](https://en.wikipedia.org/wiki/Multithreading_(computer_architecture)) or [scheduling](https://en.wikipedia.org/wiki/Scheduling_(computing)). These concepts, which can be similar (and maybe I'll explore that in a future post), are not in my scope currently.

Let's look at an example written in [Zig](https://ziglang.org/). I think the way Zig supports async is great for a first example, more so than Python or C++,  because it's very "low-level" - It shows pretty clearly what we're dealing with when we're using coroutines:

```zig
// This prints stuff
const warn = @import("std").debug.warn;

fn print_in_parts(x: i32, y: i32) i32 {
    warn("1) [print_in_parts] Hello there! Im taking a nap.\n", .{});
    suspend;

    warn("3) [print_in_parts] Why did you wake me up? Im going back to sleep.\n", .{});
    suspend;

    warn("5) [print_in_parts] Fine, here you go.\n", .{});
    return x+y;
}

fn async_main() void {
    var print_in_parts_frame = async print_in_parts(1, 2);
    
    warn("2) [async_main] Wake up!\n", .{});
    resume print_in_parts_frame;

    warn("4) [async_main] Grab a brush and put a little (makeup)!\n", .{});
    resume print_in_parts_frame;

    var result = await print_in_parts_frame;
    warn("6) [async_main] print_in_parts(1,2) == {}\n", .{result});
}
```

Let's review the flow of this program. We call `print_in_parts()` using the `async` keyword, and at that point, `print_in_parts()` begins executing. Afterwards, using the keyword `suspend`, `print_in_parts()` forfeits it's context, which gives `amain()` an opaque `frame`, which it can use to "wake up" `print_in_parts()`. As we can see `print_in_parts()` forfeits it's context again, and finally, it returns a return value, using the `return` keyword. At that point, `amain()` retrieves that value using the `await` keyword.

So the expected output is something like this:

```text
D:\Projects\AsyncExplorer\examples> .\simple_async.exe
1) [print_in_parts] Hello there! Im taking a nap.
2) [async_main] Wake up!
3) [print_in_parts] Why did you wake me up? Im going back to sleep.
4) [async_main] Grab a brush and put a little (makeup)!
5) [print_in_parts] Fine, here you go.
6) [async_main] print_in_parts(1,2) == 3
```

Also, it might be worth noting that this program is completely _single threaded_. There is no hidden thread creation or synchronization. Pretty cool in my opinion.

Phew. That was exhausting. But I think we have a pretty solid grip on what the basics of "async" means.

## Use cases

Now I hope we understand the _definition_ of a coroutine, but what are the uses of these lazy little gremlins? Are they in some way _stronger_ than our classical subroutines?

The short answer is _No_. Coroutines (and more generally, every "code construct" of that sort) doesn't allow us to calculate _more things_. But we are developers, not computer scientists. What we care about is having extensible, modular, well designed solutions to our problems. That means, we like _modelling_ our problems in such ways that let us interact with them conviniently through code. For example through Object-Oriented design, Design Patterns, and in our cases, coroutines.

Im not going to go into the use cases of coroutines too much, since it's not the point of this series. I will, though, give a quick run down of what our "break taking" functions can be useful for:

* Modelling [Generators](https://en.wikipedia.org/wiki/Generator_(computer_programming)).
* Implementing [Cooperative Multitasking](https://en.wikipedia.org/wiki/Cooperative_multitasking).
* Using [Asynchronous IO](https://en.wikipedia.org/wiki/Asynchronous_I/O).
* A lot of other cool stuff.

## Examples

So now I have a solid grasp of what "async" and "coroutines" mean. But I'm still missing the "feel" for how it looks like in some other programming languages. So let's take a look!

### Zig

We already saw an example of the basic usage of async in Zig, but let's take another one from the [official documentation](https://ziglang.org/documentation/0.6.0/):

```zig
const std = @import("std");
const assert = std.debug.assert;

var the_frame: anyframe = undefined;
var final_result: i32 = 0;

test "async function await" {
    seq('a');
    _ = async amain();
    seq('f');
    resume the_frame;
    seq('i');
    assert(final_result == 1234);
    assert(std.mem.eql(u8, &seq_points, "abcdefghi"));
}
fn amain() void {
    seq('b');
    var f = async another();
    seq('e');
    final_result = await f;
    seq('h');
}
fn another() i32 {
    seq('c');
    suspend {
        seq('d');
        the_frame = @frame();
    }
    seq('g');
    return 1234;
}

var seq_points = [_]u8{0} ** "abcdefghi".len;
var seq_index: usize = 0;

fn seq(c: u8) void {
    seq_points[seq_index] = c;
    seq_index += 1;
}
```

This example is a bit more compilicated, but try to follow the flow! We can see in the `test` section what the output sequence looks like, so it should be pretty simple following the state of the program.

### C++

In C++20, there are plans to add support for _coroutines_. Let's take a look at the examples given by [cppreference](https://en.cppreference.com/w/cpp/language/coroutines):

```c++
// uses the co_await operator to suspend execution until resumed 
task<> tcp_echo_server() {
  char data[1024];
  for (;;) {
    size_t n = co_await socket.async_read_some(buffer(data));
    co_await async_write(socket, buffer(data, n));
  }
}

// uses the keyword co_yield to suspend execution returning a value
generator<int> iota(int n = 0) {
  while(true)
    co_yield n++;
}

// uses the keyword co_return to complete execution returning a value
lazy<int> f() {
  co_return 7;
}
```

Well, this seems pretty similar to how we would use coroutines in Zig, with one added functionality: we can `co_yield` intermediate values during our execution. That wasn't in our description of coroutines earlier, but it's a feature a lot of languages and frameworks choose to support, so it's pretty interesting to see how they implement that. 

Another thing to note about coroutines in C++, is that C++ supports [_exceptions_](https://en.cppreference.com/w/cpp/language/throw). It is also a thing to keep in mind while investigating coroutines. How does these two features of the language work together? What are their interactions? 

Also, cppreference goes into a lot of details about the internals of the C++20 couroutines implementation, which is really interesting. When I'll be getting to trying to understand this particular implementation, that will be a great resource.

### Rust

Rust also allows builtin support for async programming, using the `async` and `.await` keywords. There is actually a lot going on under the hood of the Rust async implementation, but let's have a quick look. This example is taken from the [Asynchronous Programming in Rust book](https://rust-lang.github.io/async-book/01_getting_started/01_chapter.html):

```rust
async fn get_two_sites_async() {
    // Create two different "futures" which, when run to completion,
    // will asynchronously download the webpages.
    let future_one = download_async("https://www.foo.com");
    let future_two = download_async("https://www.bar.com");

    // Run both futures to completion at the same time.
    join!(future_one, future_two);
}
```

As you can see, this looks a bit more involved than the last example I took a look at. This example downloads the contents of two web-pages in a single-threaded, asynchronous, manner.

Rust's implementation of asynchronous programming relies on the [Future Trait](https://doc.rust-lang.org/beta/std/future/trait.Future.html). It's definition is a bit much for this short example, so I think I'll postpone diving into the internals of Rust async to a later post, dedicated soley for that. Actually, I really enjoyed reading into the details of this implementation and what's going on here "under the hood", and I really look forward to sticking this in a compiler and see what comes out.

### LLVM IR

This one is probably a bit weird, since LLVM IR is not really a "Programming Language" most people use. I won't go into the details of LLVM and it's structure (because I have no clue about that), but the important thing here is that a lot of modern languages uses LLVM for it's compilation and/or optimization. According to [the official documentation](https://llvm.org/docs/LangRef.html):

> LLVM is a Static Single Assignment (SSA) based representation that provides type safety, low-level operations, flexibility, and the capability of representing ‘all’ high-level languages cleanly. It is the common code representation used throughout all phases of the LLVM compilation strategy.

So yea, it's pretty cool. What's interesting for my purposes, is that LLVM [supports coroutines](https://llvm.org/docs/Coroutines.html) as a part of their IR. It's supposed to look a bit like this (don't be afraid if you don't understand everything piece of code here, neither do I):

```
define i32 @main() {
entry:
  %hdl = call i8* @f(i32 4)
  call void @llvm.coro.resume(i8* %hdl)
  call void @llvm.coro.resume(i8* %hdl)
  call void @llvm.coro.destroy(i8* %hdl)
  ret i32 0
}
```

This reminds me a bit of the Zig code we started with: we call an async function, recieve a handle (or a _frame_), resume it to our heart's content, and in the end, destroy it. Seems pretty harmless.

### Libraries and APIs

The examples Iv'e looked into so far are _compiled languages natively supporting async programming_. That's pretty cool, but there are also a lot of frameworks and operating system APIs that allow those fun shenanigans.

For example, there are [Boost.Coroutine](https://www.boost.org/doc/libs/1_57_0/libs/coroutine/doc/html/index.html) and [Boost.Coroutine2](https://www.boost.org/doc/libs/1_61_0/libs/coroutine2/doc/html/index.html) for C++'s [Boost](https://www.boost.org/), Rust's [tokio](https://docs.rs/tokio/0.2.21/tokio/), D's [Fiber](https://tour.dlang.org/tour/en/multithreading/fibers) and a lot of other [cool stuff](https://en.wikipedia.org/wiki/Coroutine#Implementations).

Operating systems like Windows support cooperative multitasking API's through [Fibers](https://docs.microsoft.com/en-us/windows/win32/procthread/fibers), and Linux's [ucontext](https://www.man7.org/linux/man-pages/man2/getcontext.2.html) can be used to implement coroutines as well. Actually, some cool guy wrapped both of those up to a cross platform [C++ library](https://github.com/tonbit/coroutine). I'll maybe cover those later, as its always fun to poke into those pesky little Windows DLLs.

## Conclusion

The point of this post was to "set the stage" for a couple of posts I'll publish here in the future. I hope I managed to give you the idea of "what" coroutines are, and maybe a touch of their use-case. In the next posts we'll be digging into some of that oh-so-sweet [x86 assembly](https://en.wikipedia.org/wiki/X86_assembly_language), taking a look of some the _implementations_ of coroutines provided by some compiled languages.

