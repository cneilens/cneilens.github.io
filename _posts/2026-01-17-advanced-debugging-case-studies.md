---
layout: post
title: "Advanced Debugging Case Studies in C++"
date: 2026-01-17 10:00:00 -0000
categories: cpp debugging advanced
---


In this final installment of our C++ debugging series, we'll dive into
some truly gnarly bugs that even seasoned developers struggle with.
We'll tackle **heap corruption**, **buffer overflows**, **memory
alignment issues**, **sneaky undefined behavior** (like ODR violations
and integer overflow), and **multithreading nightmares** (race
conditions, data corruption, deadlocks). For each case study, we'll walk
through the senior debugging mindset step-by-step. You'll see how to use
**AddressSanitizer (ASan)** and **UndefinedBehaviorSanitizer (UBSan)**
to catch errors, and then leverage powerful debuggers -- **LLDB
(macOS)**, **WinDbg (Windows)**, **VS Code + CodeLLDB (macOS)**, and
**VS Code + CppDbg (Windows)** -- to trace problems down to the root
cause. Let's get started!

## 1. Heap Corruption Case Study

Heap corruption bugs occur when you misuse dynamic memory -- writing
outside allocated bounds, using memory after freeing it, or freeing
something twice. These bugs can manifest as mysterious crashes, data
corruption, or program aborts deep inside `malloc()`/`free()`. Let's
illustrate with a use-after-free scenario (a common heap corruption):

    // heap_corruption.cpp
    #include <iostream>
    #include <cstring>
    int main() {
        char* data = (char*)malloc(16);
        strcpy(data, "Hello");
        free(data);
        // Bug: use-after-free - writing to freed heap memory
        data[0] = 'h';                    // Writing to freed memory (heap corruption)
        std::cout << data << std::endl;   // Reading freed memory
        return 0;
    }

**Symptoms:** Running this program *without* any tools might still print
"hello" (if the memory wasn't yet reused), or it might crash
unpredictably later. On Linux, a double free would often cause an
immediate abort with a message like "\*\*\* Error in `a.out`: double
free or corruption" printed to stderr. On Windows, a double free or heap
misuse might trigger a debug error or an access violation inside
`HeapFree`. These vague symptoms don't directly pinpoint the bug.

**Using AddressSanitizer:** To catch heap issues early, compile with
ASan enabled. For example, on Clang or GCC use `-fsanitize=address -g`
(and on MSVC use `/fsanitize=address` with Visual Studio 2019 or
later[\[1\]](https://learn.microsoft.com/en-us/cpp/sanitizers/error-heap-buffer-overflow?view=msvc-170#:~:text=To%20build%20and%20test%20this,or%20later%20developer%20command%20prompt)).

    $ clang++ -fsanitize=address -g heap_corruption.cpp -o heap_corruption
    $ ./heap_corruption
    =================================================================
    ==244157==ERROR: AddressSanitizer: heap-use-after-free on address 0x60b0000000f0 at pc 0x00000047a560 bp 0x7ffcdf0d59f0 sp 0x7ffcdf0d51a0
    WRITE of size 1 at 0x60b0000000f0 thread T0
        #0 0x47a55f in __interceptor_memcpy ... sanitizer_common_interceptors.inc:790
        #1 0x528403 in main heap_corruption.cpp:8[2]

ASan immediately halts at the invalid access and prints a detailed
error. The message **"heap-use-after-free"** tells us we wrote to memory
after freeing it. The output shows a stack trace: frame `#1` points at
line 8 in our code (the `data[0] = 'h'`
write)[\[2\]](https://www.osc.edu/resources/getting_started/howto/howto_use_address_sanitizer#:~:text=%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%20%3D%3D244157%3D%3DERROR%3A%20AddressSanitizer%3A%20heap,c%3A8).
It even shows the allocation and free history (not shown above for
brevity). **This is a huge help:** we now know exactly what went wrong
and where.

Next, let's use a debugger to inspect the state at the moment of
failure:

- **LLDB (macOS):** We can run the ASan-instrumented program under LLDB.
  Because ASan raises a signal when it detects the error, LLDB will
  break at the point of failure. For example:

<!-- -->

    $ lldb ./heap_corruption
    (lldb) run
    ...
    AddressSanitizer: heap-use-after-free ... (reports error)
    (lldb) bt  # backtrace

The backtrace (`bt`) in LLDB will show our code frames, confirming the
crash at the `data[0] = 'h'` line. We can also inspect variables or
memory if needed. In this simple case, ASan already told us the cause,
but the debugger lets us verify program state (e.g., check that `data`
pointer value is the same freed address ASan reported).

- **WinDbg (Windows):** On Windows, you can achieve similar insight
  using the **Page Heap** feature. By enabling *Page Heap* for your app
  (using the **GFlags** tool), every heap allocation will be surrounded
  by guard
  pages[\[3\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/example-12---using-page-heap-verification-to-find-a-bug#:~:text=Step%201%3A%20Enable%20standard%20page,heap%20verification)[\[4\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/example-12---using-page-heap-verification-to-find-a-bug#:~:text=%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%20VERIFIER%20STOP%2000000008%3A%20pid,0xAA0%3A%20corrupted%20suffix%20pattern).
  This means any out-of-bounds or use-after-free access triggers an
  immediate access violation. For example, run:

<!-- -->

    C:\> gflags /p /enable heap_corruption.exe /full
    C:\> windbg heap_corruption.exe

When the use-after-free write occurs, WinDbg breaks execution on the
exact instruction. In full Page Heap mode, WinDbg might show a
**"VERIFIER STOP 00000008: corrupted suffix pattern"**
message[\[4\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/example-12---using-page-heap-verification-to-find-a-bug#:~:text=%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%20VERIFIER%20STOP%2000000008%3A%20pid,0xAA0%3A%20corrupted%20suffix%20pattern)
-- indicating a buffer overflow or use-after-free was caught by the heap
verifier. You can then use WinDbg commands like `kb` (stack backtrace)
to see that the crash happened in our `main` at the offending line. The
**heap verifier** message "corrupted suffix pattern" specifically means
we wrote past the end of an allocated
block[\[5\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/example-12---using-page-heap-verification-to-find-a-bug#:~:text=The%20header%20information%20includes%20the,00000100%20%3A%20Block%20size),
which matches our use-after-free (the freed block's red-zone was
overwritten).

- **VS Code + Debuggers:** If you prefer a GUI, VS Code's debug
  environment can be used on both platforms. On macOS, use the CodeLLDB
  extension to launch the program with ASan; it will break on the error
  just like LLDB. On Windows, using the C++ (cppdbg) configuration, you
  can launch under the Visual Studio debugger -- the program will hit a
  fault when the heap corruption is detected (either by ASan or Page
  Heap). In VS Code, you'll see the exception notification, and you can
  open the *Call Stack* pane to find our code line. Essentially, VS Code
  with these extensions provides a front-end to LLDB or the MSVC
  debugger, so the information you get is similar.

**Diagnostic reasoning:** The key steps for a heap corruption bug are:

1.  **Reproduce with diagnostics enabled:** Turn on ASan (or Page Heap)
    to catch the violation as soon as it happens, rather than hunting
    down downstream
    effects[\[2\]](https://www.osc.edu/resources/getting_started/howto/howto_use_address_sanitizer#:~:text=%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%20%3D%3D244157%3D%3DERROR%3A%20AddressSanitizer%3A%20heap,c%3A8).
    This gives a clear starting point (the exact bad memory access).

2.  **Inspect the context:** Use the debugger's backtrace and variables
    to confirm what's being corrupted. In our case, we see the write to
    a freed pointer. We'd inspect `data` to confirm it's the same
    address that was freed. A quick mental check or debugger inquiry
    ("was this pointer freed already?") confirms the bug.

3.  **Additional tools:** If ASan hadn't been used, we might have only a
    crash in `free()` or an OS exception. In such a case, advanced heap
    debugging commands help. For example, WinDbg's `!heap` extension can
    detect heap corruptions. After a crash, running
    `!heap -p -a <address>` on the address in question can sometimes
    show heap block status and that it was freed. But these are advanced
    steps -- using sanitizers is often the faster path for memory
    errors.

4.  **Fix and retest:** After identifying the misuse (e.g., remove the
    stray `data[0]` write or ensure it happens before `free`), run again
    with ASan to confirm the error is gone.

*Side note:* Double-frees are another heap corruption. ASan catches
those too, typically with an error **"attempting double-free"** pointing
to the second `free`. On Windows, enabling Application Verifier (or Page
Heap full) will break on a double-free as well. Always pay attention to
these tool reports -- they precisely flag heap misuse that might
otherwise be heisenbugs.

## 2. Buffer Overflow Case Study

Buffer overflows are a classic C/C++ bug: writing outside the bounds of
an array. They can smash the stack or heap, leading to corrupted data or
security vulnerabilities. Let's consider a simple stack buffer overflow:

    // buffer_overflow.cpp
    #include <iostream>
    #include <cstring>
    int main() {
        char small[8];
        strcpy(small, "Overflow!");  // 9 characters into an 8-byte array (overflow)
        std::cout << "small = " << small << std::endl;
    }

Here, `small` has space for 7 characters + null terminator, but
"Overflow!" is 9 characters plus null. We overflow the stack buffer.
What happens? On many systems, this will overwrite the stack canary (a
guard value) and trigger a runtime check failure. For instance, on Linux
with default protections, you might see: **"** *stack smashing detected*
**: terminated"** and the program aborts. Without canary protection, it
might just corrupt adjacent data or the return address, causing bizarre
crashes.

**Using AddressSanitizer:** Again, ASan is extremely useful. Compile
with ASan and run:

    $ clang++ -fsanitize=address -g buffer_overflow.cpp -o buffer_overflow
    $ ./buffer_overflow
    =================================================================
    ==1==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7ffe6acc8e70 at pc 0x5591ad37f523 bp 0x7ffe6acc8e68 sp 0x7ffe6acc8e60
    READ of size 1 at 0x7ffe6acc8e70 thread T0
        #0 0x5591ad37f522 in main buffer_overflow.cpp:5:45
        #1 0x5591ad37f59a in main buffer_overflow.cpp:10:4
    ...[6]

ASan reports a **"stack-buffer-overflow"** (in this case it labels it
heap-buffer-overflow because `strcpy` tries to read beyond the allocated
chunk -- but the key is the **stack** address in the error and the line
number). The stack trace shows the error originates at the `strcpy` call
in
`main`[\[6\]](https://blog.trailofbits.com/2024/05/16/understanding-addresssanitizer-better-memory-safety-for-your-code/#:~:text=%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%20%3D%3D1%3D%3DERROR%3A%20AddressSanitizer%3A%20heap,gnu%2Flibc.so.6%2B0x29d8f%29%20%28BuildId).
ASan also typically explains where the overflow happened relative to the
buffer (e.g., "0 bytes to the right of 8-byte region"). This pinpoints
our mistake.

**Debugging the overflow:**

- **LLDB/VSCode (macOS):** Under LLDB, a stack overflow that triggers
  the canary will usually raise a SIGABRT. If you run the program in
  LLDB, it will break at the abort signal. By examining the backtrace
  (`bt`), you see the program died during or right after `strcpy`. You
  can inspect the memory around `small` using LLDB's memory commands
  (e.g., `x/16bx &small` to view bytes) to see that the string wasn't
  fully contained. With ASan, it breaks exactly at the point of
  overflow, so the process is similar to the heap case: read ASan's
  output and confirm in debugger if needed.

- **WinDbg (Windows):** In Visual C++ with security cookies (/GS), an
  overflow will trigger an exception on function return. Under WinDbg or
  VS, you'd see an access violation or an assert from the runtime. If
  you suspect an overflow, you can use WinDbg's `k` command to get a
  call stack at the point of failure. It often shows the failure
  occurred as `main` was returning (since the cookie check failed at end
  of scope). Alternatively, compile with `/fsanitize=address` in MSVC
  (available in VS2019+ as mentioned) to catch it instantly. The MSVC
  AddressSanitizer will output a similar message in the debugger console
  for the buffer
  overflow[\[1\]](https://learn.microsoft.com/en-us/cpp/sanitizers/error-heap-buffer-overflow?view=msvc-170#:~:text=To%20build%20and%20test%20this,or%20later%20developer%20command%20prompt).

- **Page Heap is for heap only:** Note that Page Heap doesn't help for
  stack overflows, since those happen in stack memory. Instead, on
  Windows, one might use Visual Studio's `/RTC` runtime checks or the
  sanitizer as discussed.

**Takeaway:** The strategy for buffer overflows is to **catch them where
they occur**. ASan's report gave us the file and line, so we know the
fix is to correct the `strcpy` (or better, use `strncpy` with correct
length, or increase the buffer). In more complex cases (e.g., buffer
overflow in a large input), you might set a breakpoint at a suspicious
loop and watch indices or use debugger watchpoints on array boundaries.
For example, if you have an array `arr[N]`, you can set a watchpoint on
`arr[N]` (the first byte past the end) to break when it's written. In
LLDB:

    (lldb) watchpoint set expression -w write -- (&arr[N])

This will halt when code writes beyond the valid range. In WinDbg, you
can do similarly with a data breakpoint (if you know the address). But
frankly, using a sanitizer or adding manual bounds checks is easier and
safer for catching overflows.

## 3. Memory Alignment Issues

Not all bugs crash immediately -- some just cause *weird behavior or
slowdowns*. One example is misaligned memory access. On many
architectures (ARM, for instance), reading a 4-byte integer from a
non-4-byte-aligned address can trigger a hardware fault or a performance
penalty. Consider this code:

    // alignment.cpp
    #include <iostream>
    #include <cstdint>
    int main() {
        alignas(1) struct Misalign { char c; int x; }; 
        Misalign m;
        char* ptr = reinterpret_cast<char*>(&m);
        // Intentionally misalign an int pointer by 1:
        int* misInt = reinterpret_cast<int*>(ptr + 1);
        *misInt = 42;  // write to an int at an unaligned address
        std::cout << "misInt=" << *misInt << std::endl;
    }

We force an `int` to be at an odd address (`ptr+1`). On x86, this
**won't crash** -- x86 can handle unaligned access (just slower). But on
stricter architectures (say, some ARM or SPARC systems), that line would
crash with a bus error. How can we debug something that doesn't
obviously crash on our machine? **UndefinedBehaviorSanitizer (UBSan)**
to the rescue.

Compile with UBSan enabled: `-fsanitize=undefined -g`. Running this on
x86 might produce:

    runtime error: store to misaligned address 0x7ffdba454351 for type 'uint16_t' (aka 'unsigned short'), which requires 2 byte alignment[7]  
    0x7ffdba454351: note: pointer points here ...  
    SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior alignment.cpp:12:5

UBSan caught the misaligned access and printed a runtime error. (In our
example, it might mention `uint16_t` if our types differ, but the idea
is the same -- it will complain that an address isn't properly aligned
for the target
type[\[7\]](https://github.com/iree-org/iree/issues/16608#:~:text=%5B%20RUN%20%5D%20MemoryStreamTest.Fill%20%2Fopt%2Fwork%2Firee%2Fgithub,behavior%20%2Fopt%2Fwork%2Firee%2Fgithub).)
The output also identifies the exact line in `alignment.cpp` where the
issue occurs.

**Using the debugger:** If you run this under LLDB on a system where it
doesn't crash, you won't get an automatic break -- the program might
happily print a value (potentially the wrong value if the hardware did
multiple accesses). In such cases, using UBSan is the best first step
because it flags an issue that the debugger wouldn't notice on its own
(since the program didn't crash). Once UBSan points it out, you use the
source info to fix the code (in this case, ensure proper alignment or
avoid such pointer casts).

If you *were* on a system where it crashed (say an ARM device or an
emulator that faults on misalignment), then the debugger would catch a
SIGBUS or similar. The backtrace would show the faulting instruction at
the `*misInt = 42` line. You could then examine the pointer value and
realize it's not divisible by 4, indicating misalignment.

**Diagnostic mindset:** Memory alignment bugs often hide as performance
issues or platform-specific crashes. The senior approach is to use UBSan
instrumentation proactively to detect undefined behaviors like this. As
the Clang documentation shows, UBSan can catch misaligned pointer use,
null dereferences, integer overflows, and
more[\[8\]](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html#:~:text=,bounds%20can%20be%20statically%20determined)[\[9\]](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html#:~:text=,be%20represented%20in%20type%20%27int).
The sanitizer is essentially your early warning system. Once a runtime
error is reported, you can use a debugger to inspect how that misaligned
address came to be (e.g. inspect the struct layout or pointer arithmetic
leading to it). In our example, seeing `ptr+1` in code is an obvious
bug, but in bigger codebases you might binary search with breakpoints or
print addresses to locate where the misalignment originates.

## 4. Nasty Undefined Behavior Cases (ODR Violations & Integer Overflow)

Undefined behavior (UB) in C++ can lead to *astonishingly weird* bugs
because the program's behavior is literally unpredictable by the
standard. Two advanced examples are: One-Definition Rule (ODR)
violations and signed integer overflow.

- **ODR Violation Scenario:** Imagine you have a header defining a
  struct or class, but two source files include slightly different
  versions (perhaps a compile-time macro causes the size to differ).
  This violates the ODR -- the program is ill-formed, but it might still
  compile and run... albeit incorrectly. You might see memory corruption
  (as one translation unit expects an object of one size while another
  writes beyond it). These are extremely hard to debug by guesswork.
  Tools can help: AddressSanitizer can detect some ODR violations at
  program startup (with a special flag) by comparing type
  metadata[\[10\]](https://github.com/google/sanitizers/wiki/AddressSanitizerOneDefinitionRuleViolation#:~:text=AddressSanitizer%2C%20however%2C%20is%20able%20to,time%20switch).
  UBSan also has checks for ODR violations. If enabled, it might print a
  runtime error when it detects two definitions of the same entity that
  don't match. For example, if class `Foo` is defined differently in two
  .cpp files, UBSan could report an error at runtime like *"ODR
  violation: Foo has different size in module A and B"*.

**Debugging approach:** If you suspect an ODR issue (e.g., weird
behavior only when certain object files are linked together), enabling
UBSan's ODR checker is step one. Once it flags something, you've
basically found the culprit (you'll need to reconcile the definitions in
your code). In a debugger, there's not a specific "ODR command," but a
clever trick is to compare `sizeof(Foo)` in different compilation units
if you can. For instance, in WinDbg or LLDB you could evaluate type
sizes or inspect object memory layouts to see a mismatch. This is
advanced (and usually not necessary thanks to sanitizers). The main
point: trust the sanitizer and then double-check your code for multiple
definitions.

- **Signed Integer Overflow:** By C++ rules, overflowing a signed
  integer (e.g., doing `INT_MAX + 1`) is undefined behavior. On typical
  hardware it will wrap around, but the compiler might assume it never
  happens and optimize accordingly, leading to logic bugs. Let's say we
  have:

<!-- -->

    int a = INT32_MAX;
    int b = a + 1;  // UB: signed overflow
    printf("b = %d\n", b);

Most likely, `b` will end up as `-2147483648` due to two's complement
wraparound. The program might appear to work, but imagine if this
overflow is controlling a loop or array index -- it could cause wrong
results or buffer access on the wrong side of an array. **UBSan**
catches this too. If compiled with `-fsanitize=undefined`, running might
print a message:

    runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'[11].

This tells us an overflow happened at that line. In a debugger, such an
overflow won't raise an exception (the CPU just wraps around), so it's
hard to notice. The senior approach upon suspecting overflow is to add
instrumentation or checks. With UBSan flagging it, we know to fix the
code (perhaps use larger types or check for overflow before adding).

If we didn't have UBSan, one could manually watch the value in the
debugger. For example, set a breakpoint at the addition and check the
values of `a` and `b`. Tools like LLDB support conditional breakpoints
-- e.g., break when `a` is near INT_MAX. But this requires you to
*suspect* the overflow in advance. That's why using sanitizers during
testing is invaluable for catching these invisible errors.

**Summary of UB debugging:** Use UBSan to **detect** subtle undefined
behaviors (alignment, overflows, null deref, etc.) at runtime with
informative
messages[\[9\]](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html#:~:text=,be%20represented%20in%20type%20%27int).
Once detected, use the debugger to further examine state if needed.
Often, the fix is straightforward once you know what happened (e.g.,
avoid relying on overflow, fix inconsistent definitions, etc.). The
combination of UBSan's diagnostics with a debugger's ability to inspect
program state gives you both the "what happened" and the "how we got
here."

## 5. Multithreading Bugs: Race Conditions, Data Corruption, and Deadlocks

Concurrency bugs are some of the trickiest to reproduce and debug. They
might only show up under specific timing, making them **heisenbugs**
(the act of debugging can change the timing and hide the bug!). We'll
look at two common classes: data races (leading to corrupted data) and
deadlocks.

### Race Conditions and Data Corruption

A **race condition** occurs when two or more threads access the same
data without proper synchronization, and at least one is a write. The
outcome can vary run-to-run, with incorrect results or corrupted state.
Consider:

    // race.cpp
    #include <thread>
    #include <iostream>
    static int counter = 0;
    void incr() {
        for(int i = 0; i < 1000000; ++i) {
            counter++;  // data race on counter
        }
    }
    int main() {
        std::thread t1(incr);
        std::thread t2(incr);
        t1.join();
        t2.join();
        std::cout << "Final counter = " << counter << std::endl;
    }

We forgot to protect `counter` with a mutex or atomic. The two threads
race to update it. If you run this, **sometimes** you'll get the correct
`2000000`, but often you'll get a smaller number (lost increments) due
to overlapped operations. There's no crash -- just wrong output.

**Diagnosing races:** Traditional debuggers don't automatically flag
data races. Here's how to tackle it:

- **ThreadSanitizer (TSan):** Although not asked in the prompt, it's
  worth mentioning: TSan is the sanitizer for data races. If you compile
  with `-fsanitize=thread`, running the program would spew a report the
  moment a data race is detected, showing the two threads' code paths
  that conflicted. For example, TSan might say "Data race on `counter`
  in function incr() \..." with stack traces for both threads. This is
  immensely helpful -- it tells you which variable and where the
  unsynchronized access occurred. (TSan reports look somewhat like
  ASan's, describing the memory address and threads; if you ever suspect
  a race, this tool is a go-to.)

- **Logging & Debugger Tricks:** Without TSan, you can insert logging:
  e.g., have each thread print when it updates `counter`. This will
  likely intermix output and show you that they are running
  concurrently. But logging can slow things and alter timing.

- **Watchpoints (Data Breakpoints):** A powerful debugger feature for
  data corruption is the **watchpoint** (also called data breakpoint).
  This lets you break execution when a specific memory location is
  accessed or changed. For instance, suppose `counter` was getting
  mysteriously set to a wrong value by some race or stray pointer. In
  LLDB you can do:

<!-- -->

    (lldb) watchpoint set variable -w write counter

This tells LLDB to pause whenever `counter` is
written[\[12\]](https://gist.github.com/rais38/6ecead686af9291da542#:~:text=Set%20a%20watchpoint%20on%20a,option%20terminator).
In GDB, the analogous command is `watch counter`. In WinDbg, you'd use
the `ba` (break on access) command (e.g., `ba w 4 &counter` to watch 4
bytes at counter's address). In VS Code, modern versions allow data
breakpoints too -- if you're stopped in debug, you can right-click a
variable in the *Watch* window and select "Break when value
changes"[\[13\]](https://devblogs.microsoft.com/cppblog/data-breakpoints-15-8-update/#:~:text=Now%2C%20data%20breakpoints%20can%20also,up%20notification%20about%20the%20change).
Once set, **resume execution**. The program will run and whenever *any
thread* modifies that variable, the debugger will pause and show you the
context.

How does this help with our `counter`? If we set a watchpoint on
`counter` and run the race program, the debugger will break on the first
modification. That's not super useful if we break on every increment
(we'd stop a million times!). But you can refine the strategy: for
example, set a condition to break only when `counter` becomes a specific
unexpected value or exceeds a threshold. Some debuggers allow
conditional watchpoints; in LLDB you could do
`watchpoint modify -c "counter > 100 && counter < 1000"` as a contrived
example. Or simply let it run and break at some random increment -- then
use the debugger's thread view to see what both threads are doing. If
you see two threads in `counter++` concurrently, you've confirmed the
race.

Another scenario: **memory corruption from races** -- suppose a data
structure gets corrupted occasionally. A senior debugger move is to
watch the memory address that keeps getting corrupted. When the
watchpoint fires, it reveals *which code* did the write. For instance,
if an integer turns to an unexpected value "42" and you suspect a race,
set a watchpoint. When it triggers, the call stack will show the culprit
thread and function that wrote to it, solving the mystery of "who the
hell writes 2 into my variable" (as one famous debugging story
goes)[\[14\]](https://khorbushko.github.io/article/2021/05/27/the-lldb-debugger-part3-watchpoint.html#:~:text=Often%2C%20,some%20cases%2C%20but%2C%20not%20always).
Watchpoints are gold for catching intermittent data corruption by
breaking at the exact moment of change.

- **Stepping and breakpoints:** You might attempt to step through both
  threads interleaved by setting breakpoints and using debugger controls
  to switch threads. This is cumbersome and not usually effective unless
  you *already* know where to look. Instead, a better mindset is to
  narrow down the race to a region of code (we know `counter++` is the
  spot here) and use tools or instrumentation to catch it.

In our `counter` example, the fix is to guard the increment with a mutex
or make `counter` an `std::atomic<int>`. But witnessing the
non-deterministic outcome (final values varying) or using
TSan/watchpoints to catch concurrent access gives you confidence about
the root cause.

### Deadlocks

A deadlock happens when two or more threads lock resources in an
inconsistent order, causing each to wait forever for the other. The
program just hangs -- no crash, no error message. Let's simulate a
deadlock:

    // deadlock.cpp
    #include <mutex>
    #include <thread>
    int main() {
        std::mutex m1, m2;
        auto task1 = [&]() {
            std::lock_guard<std::mutex> lock1(m1);
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            std::lock_guard<std::mutex> lock2(m2);
        };
        auto task2 = [&]() {
            std::lock_guard<std::mutex> lock2(m2);
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            std::lock_guard<std::mutex> lock1(m1);
        };
        std::thread t1(task1);
        std::thread t2(task2);
        t1.join(); t2.join();
    }

Thread 1 locks m1 then m2, while Thread 2 locks m2 then m1. The 100ms
sleep ensures they each get the first lock and then try for the second,
leading to a deadlock. The program will hang at join and never
terminate.

**Diagnosing a deadlock:** When your program seems stuck, a common
approach is to pause it in a debugger and examine thread states:

- **Visual Debuggers (VS, VS Code):** If you attach to or launch the
  hung program in VS Code or Visual Studio, you can hit the "pause"
  button (break all). The debugger will halt all threads wherever they
  are. Then open the *Threads* panel. You'll likely see one thread's
  call stack stopped in `lock_guard` (inside `pthread_mutex_lock` on
  Unix or `EnterCriticalSection` on Windows), and the other thread
  similarly paused trying to acquire the other mutex. Each owns one lock
  and waits on the other -- classic deadlock. By looking at the call
  stack, you see exactly which line in `task1` and `task2` they're
  waiting at. This tells you the locks are acquired in inconsistent
  order.

- **LLDB (CLI):** In LLDB, you can do `thread list` to see all threads,
  then `thread backtrace all` (or the shorthand `bt all`). This will
  print the backtrace of each thread. You would see something like:
  Thread 1's trace includes `lock_guard<std::mutex> lock1(m1)` and is
  now inside the mutex code waiting for m2, and Thread 2's trace
  includes `lock_guard<std::mutex> lock2(m2)` now waiting for m1. LLDB
  doesn't have a special deadlock detector, but this manual inspection
  reveals it. There's no "deadlock exception" -- the program is just
  waiting, so you use the debugger's ability to snapshot all threads.

- **WinDbg:** In WinDbg, you can break into the hung process (Debug -\>
  Break). Use `~*kp` to get all thread stacks. You'll see one thread in
  something like `ntdll!ZwWaitForSingleObject` or
  `KernelBase!WaitForCriticalSection` (if it's a CriticalSection), and
  the other thread in a similar wait. WinDbg has an extension `!locks`
  which can list owned critical sections and who owns
  them[\[15\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/displaying-a-critical-section#:~:text=0%3A000).
  If our std::mutex uses Win32 CriticalSections under the hood, running
  `!locks` might output each lock, showing one is owned by thread X and
  one by thread Y, each with one waiting thread. For example, `!locks`
  output might show "CritSec m1 ... OwningThread = thread1id,
  LockCount=1, EntryCount=1" and "CritSec m2 ... OwningThread =
  thread2id, LockCount=1...". That confirms the deadlock circle. (Note:
  `!locks` requires symbols and works for known synchronization
  primitives; it's very useful in complex apps to list all locks and
  owners[\[15\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/displaying-a-critical-section#:~:text=0%3A000).)

**Resolving deadlocks:** The fix is to enforce a consistent locking
order or use lock ordering utilities (like `std::scoped_lock(m1, m2)`
which locks both in a defined order). But from a debugging perspective,
the "Aha!" moment is realizing threads are stuck waiting on each other.
The debugger gives you that insight by frozen call stacks. A senior dev
also knows to **suspect deadlock** if an app is unresponsive and
especially if the threads involved hold locks -- then immediately go to
thread dump analysis as above.

**Tip:** Sometimes deadlocks only happen under certain conditions. If
you can't reproduce easily, you might force-break the program when it
hangs (e.g., sending a SIGINT to generate a core dump, or on Windows
using Task Manager to create a dump file). Then you can load that dump
in WinDbg or LLDB and perform the same thread stack inspection. It's
post-mortem debugging, but the thread traces will often show the
circular wait.

------------------------------------------------------------------------

## Conclusion

Debugging advanced C++ bugs requires a blend of the right tools and the
right mindset. We covered how **sanitizers** like ASan and UBSan act as
bug radar -- catching memory errors and undefined behavior *as soon as
they happen*, with helpful diagnostics. We saw how to compile and
interpret their reports, using them to zero in on the offending code
lines (for example, ASan pointing out a heap buffer
overflow[\[6\]](https://blog.trailofbits.com/2024/05/16/understanding-addresssanitizer-better-memory-safety-for-your-code/#:~:text=%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%20%3D%3D1%3D%3DERROR%3A%20AddressSanitizer%3A%20heap,gnu%2Flibc.so.6%2B0x29d8f%29%20%28BuildId)
or UBSan alerting us to an integer
overflow[\[11\]](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html#:~:text=%25%20clang%2B%2B%20,be%20represented%20in%20type%20%27int)).
We then paired that with systematic **debugger workflows**: setting
breakpoints/watchpoints, inspecting memory and threads, and using
platform-specific features (like Page Heap on
Windows[\[4\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/example-12---using-page-heap-verification-to-find-a-bug#:~:text=%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%20VERIFIER%20STOP%2000000008%3A%20pid,0xAA0%3A%20corrupted%20suffix%20pattern)
or LLDB's watchpoints) to unravel the bug completely.

For each case -- whether it's a rogue memory write, an overflow, or a
multithreading blunder -- the process was: **observe symptoms, use tools
to trap the bug in action, then investigate state with a debugger to
understand and confirm the cause.** By combining sanitizer output with
interactive debugging, we get both the high-level error report *and* the
low-level insight. This is the senior developer's approach: **leverage
every aid available**, and methodically narrow down the problem.

As you tackle your own tough bugs, remember these examples. If something
"impossible" is happening, think of undefined behavior -- and try UBSan.
If memory is mysteriously corrupt, run with ASan or set a watchpoint to
catch the culprit. If your app hangs, break in and examine thread
backtraces. By practicing these techniques, you'll not only fix the bug
at hand but also build an intuition for debugging that will serve you
throughout your C++ career. Happy debugging!

**Sources:** The techniques and outputs above were informed by sanitizer
documentation[\[9\]](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html#:~:text=,be%20represented%20in%20type%20%27int)[\[11\]](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html#:~:text=%25%20clang%2B%2B%20,be%20represented%20in%20type%20%27int),
official debugging guides, and real-world examples of using LLDB,
WinDbg, and VS Code. Notably, AddressSanitizer and
UndefinedBehaviorSanitizer are documented by Clang/LLVM and provide
clear error messages for issues like
heap-use-after-free[\[2\]](https://www.osc.edu/resources/getting_started/howto/howto_use_address_sanitizer#:~:text=%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%20%3D%3D244157%3D%3DERROR%3A%20AddressSanitizer%3A%20heap,c%3A8)
and misaligned memory
access[\[7\]](https://github.com/iree-org/iree/issues/16608#:~:text=%5B%20RUN%20%5D%20MemoryStreamTest.Fill%20%2Fopt%2Fwork%2Firee%2Fgithub,behavior%20%2Fopt%2Fwork%2Firee%2Fgithub).
Microsoft's documentation on Page Heap and debugger extensions
illustrates how WinDbg catches heap corruption (e.g., "corrupted suffix
pattern" on
overflow)[\[4\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/example-12---using-page-heap-verification-to-find-a-bug#:~:text=%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%20VERIFIER%20STOP%2000000008%3A%20pid,0xAA0%3A%20corrupted%20suffix%20pattern)
and how Visual Studio/VS Code support data breakpoints for tracking down
errant
writes[\[13\]](https://devblogs.microsoft.com/cppblog/data-breakpoints-15-8-update/#:~:text=Now%2C%20data%20breakpoints%20can%20also,up%20notification%20about%20the%20change)[\[16\]](https://learn.microsoft.com/en-us/shows/pure-virtual-cpp-2022/data-breakpoints-in-visual-studio-code#:~:text=Data%20breakpoints%20in%20GDB%20allow,be%20hard%20to%20track%20down).
By applying these tools as shown, you can conquer even the most
perplexing C++ bugs.
[\[2\]](https://www.osc.edu/resources/getting_started/howto/howto_use_address_sanitizer#:~:text=%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%20%3D%3D244157%3D%3DERROR%3A%20AddressSanitizer%3A%20heap,c%3A8)[\[7\]](https://github.com/iree-org/iree/issues/16608#:~:text=%5B%20RUN%20%5D%20MemoryStreamTest.Fill%20%2Fopt%2Fwork%2Firee%2Fgithub,behavior%20%2Fopt%2Fwork%2Firee%2Fgithub)[\[4\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/example-12---using-page-heap-verification-to-find-a-bug#:~:text=%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%20VERIFIER%20STOP%2000000008%3A%20pid,0xAA0%3A%20corrupted%20suffix%20pattern)[\[11\]](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html#:~:text=%25%20clang%2B%2B%20,be%20represented%20in%20type%20%27int)

------------------------------------------------------------------------

[\[1\]](https://learn.microsoft.com/en-us/cpp/sanitizers/error-heap-buffer-overflow?view=msvc-170#:~:text=To%20build%20and%20test%20this,or%20later%20developer%20command%20prompt)
Error: heap-buffer-overflow \| Microsoft Learn

<https://learn.microsoft.com/en-us/cpp/sanitizers/error-heap-buffer-overflow?view=msvc-170>

[\[2\]](https://www.osc.edu/resources/getting_started/howto/howto_use_address_sanitizer#:~:text=%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%20%3D%3D244157%3D%3DERROR%3A%20AddressSanitizer%3A%20heap,c%3A8)
HOWTO: Use Address Sanitizer \| Ohio Supercomputer Center

<https://www.osc.edu/resources/getting_started/howto/howto_use_address_sanitizer>

[\[3\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/example-12---using-page-heap-verification-to-find-a-bug#:~:text=Step%201%3A%20Enable%20standard%20page,heap%20verification)
[\[4\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/example-12---using-page-heap-verification-to-find-a-bug#:~:text=%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%20VERIFIER%20STOP%2000000008%3A%20pid,0xAA0%3A%20corrupted%20suffix%20pattern)
[\[5\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/example-12---using-page-heap-verification-to-find-a-bug#:~:text=The%20header%20information%20includes%20the,00000100%20%3A%20Block%20size)
Example 12 Using Page Heap Verification to Find a Bug - Windows drivers
\| Microsoft Learn

<https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/example-12---using-page-heap-verification-to-find-a-bug>

[\[6\]](https://blog.trailofbits.com/2024/05/16/understanding-addresssanitizer-better-memory-safety-for-your-code/#:~:text=%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%3D%20%3D%3D1%3D%3DERROR%3A%20AddressSanitizer%3A%20heap,gnu%2Flibc.so.6%2B0x29d8f%29%20%28BuildId)
Understanding AddressSanitizer: Better memory safety for your code - The
Trail of Bits Blog

<https://blog.trailofbits.com/2024/05/16/understanding-addresssanitizer-better-memory-safety-for-your-code/>

[\[7\]](https://github.com/iree-org/iree/issues/16608#:~:text=%5B%20RUN%20%5D%20MemoryStreamTest.Fill%20%2Fopt%2Fwork%2Firee%2Fgithub,behavior%20%2Fopt%2Fwork%2Firee%2Fgithub)
ubsan load/store misaligned address errors from memory_stream_test and
gguf_parser_test · Issue #16608 · iree-org/iree · GitHub

<https://github.com/iree-org/iree/issues/16608>

[\[8\]](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html#:~:text=,bounds%20can%20be%20statically%20determined)
[\[9\]](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html#:~:text=,be%20represented%20in%20type%20%27int)
[\[11\]](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html#:~:text=%25%20clang%2B%2B%20,be%20represented%20in%20type%20%27int)
UndefinedBehaviorSanitizer --- Clang 23.0.0git documentation

<https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html>

[\[10\]](https://github.com/google/sanitizers/wiki/AddressSanitizerOneDefinitionRuleViolation#:~:text=AddressSanitizer%2C%20however%2C%20is%20able%20to,time%20switch)
AddressSanitizerOneDefinitionR\...

<https://github.com/google/sanitizers/wiki/AddressSanitizerOneDefinitionRuleViolation>

[\[12\]](https://gist.github.com/rais38/6ecead686af9291da542#:~:text=Set%20a%20watchpoint%20on%20a,option%20terminator)
Watchpoint in LLDB · GitHub

<https://gist.github.com/rais38/6ecead686af9291da542>

[\[13\]](https://devblogs.microsoft.com/cppblog/data-breakpoints-15-8-update/#:~:text=Now%2C%20data%20breakpoints%20can%20also,up%20notification%20about%20the%20change)
Data Breakpoints - Visual Studio 2017 15.8 Update   - C++ Team Blog

<https://devblogs.microsoft.com/cppblog/data-breakpoints-15-8-update/>

[\[14\]](https://khorbushko.github.io/article/2021/05/27/the-lldb-debugger-part3-watchpoint.html#:~:text=Often%2C%20,some%20cases%2C%20but%2C%20not%20always)
The LLDB Debugger - Part 3: Watchpoint \| kyryl horbushko

<https://khorbushko.github.io/article/2021/05/27/the-lldb-debugger-part3-watchpoint.html>

[\[15\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/displaying-a-critical-section#:~:text=0%3A000)
Displaying a Critical Section - Windows drivers \| Microsoft Learn

<https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/displaying-a-critical-section>

[\[16\]](https://learn.microsoft.com/en-us/shows/pure-virtual-cpp-2022/data-breakpoints-in-visual-studio-code#:~:text=Data%20breakpoints%20in%20GDB%20allow,be%20hard%20to%20track%20down)
Data Breakpoints in Visual Studio Code \| Microsoft Learn

<https://learn.microsoft.com/en-us/shows/pure-virtual-cpp-2022/data-breakpoints-in-visual-studio-code>
