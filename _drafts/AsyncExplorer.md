
Have you ever looked up into the stars and wondered, "How the fuck is that feature implemented"? In this series, I'll (hopefully) dive into the implementation of coroutines for several (compiled) programming languages.

A short disclaimer: I'm not too sharp on the details of some (or actually any) of these implementations. Most of this will just be me rambling and looking at compiler source code/the output of [godbolt](https://godbolt.org/). I try really hard to validate every claim I'll post here, but some mistakes are sure to sneak their way into one of these. Feel free to point them up and I'll fix them as soon as I can.

## Purpose

The purpose of this post is to set a few definitions for my upcoming posts. I'll explain what I mean when I use words like "coroutine" or "asynchronous", which will hopefully allow me to use them freely in the following posts.

Afterwards, I'll give a few short examples of how to coroutines "look like" in several programming languages, like C++, Rust, Zig and the LLVM IR. 

I hope I'll also be able to give you a rough idea of what coroutines are usful for, but it's not the main goal of this post.

This is not an "asyncio" tutorial or anything like that, just me spewing nonsense and messing with some programming languages.

## What is `async`?

If you're like me, you probably heard terms like "async" and "coroutines" mentioned a lot, and maybe even saw (or wrote) some asynchronious code, but didn't really understand what's going on behind the scenes. Atleast I didn't (and still don't).

Let's consult [Wikipedia](https://en.wikipedia.org/wiki/Coroutine)!

> Coroutines are computer program components that generalize subroutines for non-preemptive multitasking, by allowing execution to be suspended and resumed. 

Let's break it down a little.

For me, a subroutine is a piece of code that accepts parameters, and "executes". That is, processes that parameters, produces side effects, and returns a value. Now, according to Wikipedia, a coroutine is a subroutine that can be "suspended and resumed". The way I understand that, I'm getting a picture of a subroutine "taking a nap", which we can later wake up and continue. Notice that I'm explicitly choosing to not mention multithreading or scheduling. These concepts, which can be similar (and maybe I'll explore that in a future post), are not in my scope currently.

Let's look at an example written in Zig, which shows pretty clearly what we're dealing with:

```zig
const warn = @import("std").debug.warn;

fn foo(x: i32, y: i32) i32 {
    warn("1) Hello there! Im taking a nap.\n", .{});
    suspend;

    warn("3) Why did you wake me up? Im going back to sleep.\n", .{});
    suspend;

    warn("5) Fine, here you go.\n", .{});
    return x+y;
}

fn amain() void {
    var frame = async foo(1, 2);
    
    warn("2) Wake up!\n", .{});
    resume frame;

    warn("4) Grab a brush and put a little (makeup)!\n", .{});
    resume frame;

    var ret = await frame;
    warn("6) foo(1,2) == {}\n", .{ret});
}
```

Let's review the flow of this program. We call `foo()` using the `async` keyword, and at that point, `foo()` begins executing. Afterwards, using the keyword `suspend`, `foo()` forfeits it's context, which gives `amain()` an opaque `frame`, which it can use to "wake up" `foo()`. As we can see `foo()` forfeits it's context again, and finally, it returns a return value, using the `return` keyword. At that point, `amain()` retrieves that value using the `await` keyword.

So the expected output is something like this:

```text
D:\Projects\AsyncExplorer\examples> .\simple_async.exe
1) Hello there! Im taking a nap.
2) Wake up!
3) Why did you wake me up? Im going back to sleep.
4) Grab a brush and put a little (makeup)!
5) Fine, here you go.
6) foo(1,2) == 3
```

Phew. That was exhausting. But I think we have a pretty solid grip on what the basics of "async" means.

## Examples

## Use cases

Now I hope we understand the _definition_ of a coroutine, but what are the uses of these lazy little gremlins? Are they in some way _stronger_ than our classical subroutines?

The short answer is _No_. Coroutines (and more generally, every "code construct" of that sort) doesn't allow us to calculate _more things_. But we are developers, not computer scientists. What we care about is having extensible, modular, well designed solutions to our problems. That means, we like _modelling_ our problems in such ways that let us interact with them conviniently through code. For example through Object-Oriented design, Design Patterns, and in our cases, coroutines.

Im not going to go into the use cases of coroutines too much, since it's not the point of this series. I will, though, give a quick run down of what our "break taking" functions can be usful for:

* Modelling [Generators](https://en.wikipedia.org/wiki/Generator_(computer_programming)).
* Implementing [Cooperative Multitasking](https://en.wikipedia.org/wiki/Cooperative_multitasking).
* Using [Asynchronous IO](https://en.wikipedia.org/wiki/Asynchronous_I/O).
* A lot of other cool stuff.

## Conclusion

The point of this post was to "set the stage" for a couple of posts I'll publish here in the future. I hope I managed to give you the idea of "what" coroutines are, and maybe a touch of their use-case. In the next posts we'll be digging into some of that oh-so-sweet x86 assembly, taking a look of some the _implementations_ of coroutines provided by some compiled languages.
