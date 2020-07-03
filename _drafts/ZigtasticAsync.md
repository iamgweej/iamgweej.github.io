> **TODO** Explanation about Zig

> **TODO** Explanation about Godbolt?

> **TODO** Environment: Zig 0.6.0, x86_64

## A simple program

Let's start with simplest async program imaginable:

```zig
fn afoo() void {
    suspend;
}

export fn caller() void {
    var frame = async afoo();
}
```

We'll compile it with `--single-threaded`, since there is a slight difference I saw when compiling without this flag, which might complicate things. Don't worry, we'll get back to that!

Stick this sucker in Godbolt, and let's look at the [output](https://godbolt.org/z/8NuCF2)... Well. That's a lot of code. We'll go through it step by step.

It starts with the standard prolog:

```assembly
caller:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 32
```

which means the `frame` variable is 32 bytes wide. We'll write that down, and update it as we go.

```cpp
struct afoo_frame {
    uint8_t unknown[32];
};
```

Godbolt tells us which line generates which instructions. So we see that the line:

```zig
var frame = async afoo();
```

generates:

```assembly
        movabs  rax, offset afoo
        mov     qword ptr [rbp - 32], rax
        mov     qword ptr [rbp - 24], 0
        mov     qword ptr [rbp - 16], 0
        lea     rdi, [rbp - 32]
        mov     rsi, -3
        call    afoo
```

There are quite a few things going on here. We put the offset of `afoo()` in the first 8 bytes of the `afoo_frame` struct. We also put zeros in the 8th and 16th offsets of the structure. This makes us suspect that the structure is actually 24 bytes wide, and `sub rsp, 32` was emitted to keep the stack 16-bytes aligned (or something of that sort). This makes us guess it looks a bit like this:

```cpp
struct afoo_frame {
    fptr_t      func;       // The async function the frame holds
    uint64_t    unknown2;   // Initialized to 0
    uint64_t    unknown3;   // Initialized to 0
};
```

What's the point of putting the function pointer in the frame? Remember, when we use `async`, we can continue a frame without knowing which function we're calling. We're continuing a _frame_, not a specific _function_. This means the frame has to hold, in some manner, the function we're about to call.

Now, the actual call to `afoo()` looks like this:

```c
afoo(
    &frame  // passed in %rdi%
    -3,     // passed in %rsi%
);
```

That's odd. `afoo()` is supposed to be a `void (*)(void)` function, why is taking two parameters?

Well, the `&frame` parameter actually makes sense. Remember a _function frame_ holds all of its parameters and arguments. Since coroutines are invoked _asynchronously_, they can't push that data on the stack, cause stack might change when they are called again. They _have_ to recieve a pointer to their frame, and trust their caller that this pointer is valid throughout all of their execution. My guess is that in a "real" asynchrounous program those frames will be _heap allocated_, to ensure the frame's lifetime throughout their program. In our case, the compiler knows that this frame will only exist when `caller()` is running, so it's making it stack allocated instead. We'll come back to this topic later to check our assumption.

Now, the `rsi` parameter is a bit peculiar. I think it's related to Zig's _safety guarentees_ or the fact we're in debug mode. I think `rsi` is used as a parameter for the `panic()` function. This assumption is based on the fact that the only reference to it is in the `panic` assembly generated:

```assembly
panic:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     qword ptr [rbp - 8], rsi
        call    zig_panic
```

With that assumption in mind, I'm going to rudely ignore it throughout this article (please forgive me).

Now, let's take a look at the start of the `afoo()` function:

```assembly
afoo:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 32
        mov     rax, rdi
        add     rax, 16
        mov     rcx, rdi
        add     rcx, 8
        mov     rdx, qword ptr [rdi + 8]
        test    rdx, rdx
        mov     qword ptr [rbp - 8], rax
        mov     qword ptr [rbp - 16], rcx
        mov     qword ptr [rbp - 24], rdx
        je      .LBB2_1
        jmp     .LBB2_8
```

I trust you all to know your basic x86_64, so let's zoom through this. We have the standard prolog, allocating 32 bytes on the stack. Remembering that `&frame` as passed in `%rdi%`, we understand that this code:

```assmebly
        mov     rax, rdi
        add     rax, 16
        mov     rcx, rdi
        add     rcx, 8
        mov     rdx, qword ptr [rdi + 8]
        ; -- snip --
        mov     qword ptr [rbp - 8], rax
        mov     qword ptr [rbp - 16], rcx
        mov     qword ptr [rbp - 24], rdx
```

Translates to something of this fashion:

```c
uint64_t *unknown3_ptr = &frame->unknown3;
uint64_t *unknown2_ptr = &frame->unknown2;
uint64_t unknown2 = frame->frame2;
```

Cool. Now, we have a have a branching:

```assembly
        mov     rcx, rdi
        ; -- snip --
        mov     rdx, qword ptr [rdi + 8]
        test    rdx, rdx
        ; -- snip -- 
        je      .LBB2_1
        jmp     .LBB2_8
```

This suggests that `frame.unknown2` is actually a _branch identifier_. I like to think of coroutines like this: every _suspension point_ "splits" the function to two code blocks. The code following the suspension point and the code preceeding it. When `resume`ing the function, it has to know which block it has to continue from. Our current assumption is that Zig acheives this using a _branch identifier_ saved in the frame, telling it where to resume next.

Lets rewrite our structure according to this assumption:

```c
struct afoo_frame {
    fptr_t      func;       // The async function the frame holds
    uint64_t    branch_id;  // The branch to execute the next time the frame gets resumed
    uint64_t    unknown3;
};
```

Let's keep going! First, let's look at what happens in our flow, that is, `frame.branch_id` is 0. In that case, we jump to `.LBB2_1`:

```assembly
.LBB2_1:
        mov     rax, qword ptr [rbp - 16]
        mov     qword ptr [rax], 1
        mov     qword ptr [rax], 2
        add     rsp, 32
        pop     rbp
        ret
```

Ok, there's something odd happening here. We set `frame->branch_id` to 1, and then immediatly set it to 2. My guess is that this is a result of us using the `--single-threaded` flag. My assumption that this is some sort of [spinlock](https://en.wikipedia.org/wiki/Spinlock) or _condition variable_, to prevent the running of the same frame in several threads. Let's check that assumption later! For now, we'll ignore it.

We can now "split" our `afoo()` function to two branches:

```zig
fn afoo() void {
    // branch_id 0
    suspend;
    // branch_id 2
}
```

Now, we clean up the stack frame and `ret`. This is important: A `suspend` translates to a `ret`. It doesnt "suspend" anything. It just returns to the caller, like any "normal" function!

Let's check out the other branch:

```
.LBB2_8:
        mov     rax, qword ptr [rbp - 24]
        sub     rax, 1
        je      .LBB2_3
        jmp     .LBB2_9
```

Hmm. It checks that `frame->branch_id` is not 1. Our assumption that `frame->branch_id == 1` implies that _another thread is running this frame currently_. Indeed, if we follow the case `frame->branch_id == 1` we see that:

```
.LBB2_3:
        xor     eax, eax
        mov     esi, eax
        movabs  rdi, offset __unnamed_2
        call    panic

__unnamed_5:
        .asciz  "resumed a non-suspended function"

__unnamed_2:
        .quad   __unnamed_5
        .quad   32
```

We see that in that case, `panic()` is called with `%esi% = 0` and `rdi` pointing to the _Pascal String_ containing the error message `"resumed a non-suspended function"`. This confirms our suspicion: in multithreaded build, `branch_id == 1` implies that the frame is _currently running_.

This brings up another thing that have been bothering me. Why both `&frame->branch_id` and `frame->branch_id` are stored in the local stack frame of `afoo()`? My guess is that the storing and loading of `frame->branch_id` in multithreaded builds happens _atomically_, to prevent race conditions. Let's add that assumption to our ever-growing list of stuff to check.

Well, lets continue. If no one else is running the frame, we arrive at another check:

```
.LBB2_8:
        mov     rax, qword ptr [rbp - 24]
        sub     rax, 1
        je      .LBB2_3
        jmp     .LBB2_9
```