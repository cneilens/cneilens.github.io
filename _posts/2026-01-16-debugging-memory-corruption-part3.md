---
layout: post
title: "Debugging Memory Corruption and Crashes in C++ (Part 3)"
date: 2026-01-16 10:00:00 -0000
categories: cpp debugging case-studies
---


In this third part of our series, we'll walk through **multiple
debugging case studies** involving memory corruption, crashes, and
undefined behavior in C++ programs. We'll use **the same buggy code in
each case** and debug it step-by-step with four different
tools/environments:

- **LLDB on macOS** (command-line debugger)
- **WinDbg on Windows** (Windows debugger, using MSVC symbols)
- **VS Code with CodeLLDB** (on macOS)
- **VS Code with CppDbg** (on Windows, using GDB)

For each case study, we'll present a **realistic code example** (null
pointer dereference, use-after-free, stack overflow, race condition,
etc.) and then do a **complete walkthrough** of debugging that bug using
each tool. Along the way, we'll highlight the **senior engineer's
thought process**: how to compile with debug info, set breakpoints, run
the program under a debugger, interpret crashes or incorrect behavior,
inspect variables, examine call stacks and memory, use watchpoints (data
breakpoints), and ultimately identify the root cause of the bug. The
tone remains friendly and hands-on, guiding you through each step just
as in the earlier articles. Let's dive in!

## Case Study 1: Null Pointer Dereference Crash (Single-Threaded)

**The Bug:** Our first example is a classic *null pointer dereference*.
The program calls a function with a null pointer, and that function
tries to dereference it, leading to a crash. This is a simple scenario
to get started with debugging basics.

    #include <iostream>

    void foo(int* ptr) {
        // Attempt to write to the pointer
        *ptr = 42;  // BUG: ptr is null, this will crash
    }

    int main() {
        int *p = nullptr;
        foo(p);  // calling foo with a null pointer
        std::cout << "This will not execute.\n";
        return 0;
    }

If you compile and run this program normally, it will crash with a
**segmentation fault** (access violation) because we tried to write to
memory address 0 (null). The goal is to use each debugger to catch this
crash and confirm that the pointer was null.

### Debugging with LLDB (macOS)

**Step 1: Compile with debug symbols.** On macOS, we'll use `clang++` or
`g++`. It's crucial to include debug info so the debugger can show
source lines and variable names. For example:

    clang++ -g -O0 -std=c++17 -o nullptr_bug nullptr_bug.cpp

Here `-g` generates debug symbols (so we can see source and variables in
LLDB)[\[1\]](https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html#:~:text=match%20at%20L204%20The%20compiler,as%20opposed%20to%20deployment),
and `-O0` disables optimizations (making debugging easier by preserving
the code structure).

**Step 2: Launch LLDB and load the program.** In a Terminal, run:

    lldb nullptr_bug

This starts LLDB and drops you at the `(lldb)` prompt with our program
loaded. You can also do `lldb` then `target create "nullptr_bug"` --
both have the same
effect[\[2\]](https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html#:~:text=,lldb).

**Step 3: Set a breakpoint (optional).** In this simple case, we can
actually let the program crash and LLDB will break on the fault
automatically. But as a senior engineer, you might set a breakpoint to
inspect state before the crash. For example, set a breakpoint at the
start of `foo` to check the pointer value:

    (lldb) break set -f nullptr_bug.cpp -l 5 

This sets a breakpoint in `nullptr_bug.cpp` at line 5 (inside `foo`).
You could also do `(lldb) b foo` to break at function `foo()`. LLDB will
confirm the breakpoint is set.

**Step 4: Run the program in the debugger.** Type `run` (or `r`) at the
`(lldb)` prompt:

    (lldb) run
    Process 12345 launched: './nullptr_bug' (x86_64)

LLDB starts executing the program. If you set a breakpoint at `foo`,
execution will pause there before the crash. If not, it will run until
the crash occurs. In our case, if we didn't set any breakpoint, LLDB
will break at the crash point. Suppose we didn't break early --- we
would see something like:

    Process 12345 stopped
    * thread #1: hit 0x0000000100001145 nullptr_bug`foo(ptr=0x0) at nullptr_bug.cpp:6, stop reason = EXC_BAD_ACCESS (code=1, address=0x0)[3]

This message tells us the program stopped due to a bad memory access. It
even shows that in `foo(ptr=0x0)` at line 6, `ptr` was `0x0` (NULL)! We
have already a huge clue: `ptr` is null.

**Step 5: Examine the call stack.** In LLDB, use the `bt` (backtrace)
command to see the call stack:

    (lldb) bt
    * thread #1, stop reason = EXC_BAD_ACCESS (code=1, address=0x0)
      * frame #0: foo(int* ptr) at nullptr_bug.cpp:6
        frame #1: main() at nullptr_bug.cpp:12

This shows that we're in `foo`, called from
`main`[\[4\]](https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html#:~:text=,fib%28n%3D2%29%20%2B%2071%20at).
Only two frames -- it's a simple program. Frame #0 is `foo` where the
crash happened, frame #1 is `main`. The senior engineer notes *where* in
the code the crash occurred (line 6 in `foo`) and the call path.

**Step 6: Inspect variables and memory.** Let's confirm the pointer
value. We can switch to the crashing frame and print `ptr`:

    (lldb) frame variable ptr
    (int *) ptr = 0x0000000000000000

This confirms `ptr` is `0x0`. We've found the bug: `foo` was called with
a null pointer. We can also list local variables with `frame variable`
(or shorthand `fv`) -- here it's just `ptr`. If we had a more complex
scenario, we might use `p ptr` or `p *ptr`. (Attempting `p *ptr` now
would likely also trigger an error since that tries to read memory at
0x0.) The key insight is that `ptr` is null, which is why writing to
`*ptr` caused a crash.

For completeness, LLDB allows examining memory with commands like
`memory read <address>` if needed. Here, the address is 0, which is
invalid, so that's not useful.

**Step 7: Identify root cause.** The senior engineer's thought process:
*"Why was* `ptr` *null? Oh, in* `main` *we explicitly set* `p = nullptr`
*and still passed it to* `foo`*. That's our bug -- we dereferenced a
null pointer."* We've confirmed the cause. The fix would be to either
initialize `p` properly or add a null check in `foo`.

**Watchpoints in LLDB:** In this case, a watchpoint isn't needed (we
directly caught the crash). But as a note, LLDB can set *watchpoints* to
monitor memory reads/writes. For example, if we wanted to break whenever
a variable changes or a certain address is accessed, we could use:

    (lldb) watchpoint set variable -w write ptr   // break when 'ptr' is written

or watch an address:

    (lldb) watchpoint set expression -w write -- <address>[5] 

This is more useful in the next scenarios (e.g. catching *who* modifies
a pointer). Keep this in mind as a powerful technique.

LLDB summary: we compiled with symbols, ran the program, LLDB stopped at
the crash, and we used the stack trace and variable inspection to
determine that a null pointer caused the crash. The entire process was
quick -- much better than guessing from just a "Segmentation fault"
message!

### Debugging with WinDbg (Windows)

Next, let's see how a Windows expert would debug the **same null pointer
crash** using WinDbg. We assume we compiled the program with MSVC in
Debug mode (which produces a `.pdb` symbol file). For example, from the
Developer Command Prompt:

    cl /Zi /Od /Fe:nullptr_bug.exe nullptr_bug.cpp

`/Zi` generates debug symbols (PDB), and `/Od` disables optimizations.
Now we have `nullptr_bug.exe` and a PDB for debugging.

**Step 1: Launch WinDbg and load the program.** Open **WinDbg** (the
**Desktop** or **Preview** version). In the menu, choose **File -\>
Launch Executable** and select `nullptr_bug.exe`. WinDbg will start the
process in a paused state at the entry point (it typically breaks on an
initial breakpoint in `ntdll` or at program start). You'll see something
like:

    (0:000> break instruction exception - code 80000003)

This is the initial break. Before we continue, ensure WinDbg has loaded
symbols for our app. WinDbg might load the PDB automatically if it's in
the same directory. You can verify by the command: `.sympath` to check
symbol path (or use `.symfix` to set up Microsoft symbol servers for
system libraries, if needed) and `.reload` to reload
symbols[\[6\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/getting-started-with-windbg#:~:text=3,readable%20function%20and%20variable%20names)[\[7\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/getting-started-with-windbg#:~:text=Then%2C%20enter%20this%20command%3A).
For our own program, it should find the PDB locally.

**Step 2: Run the program (go).** At the `0:000>` prompt, type `g` and
hit Enter. This resumes the program. We haven't set any breakpoints yet,
so it will run until something noteworthy happens -- in this case, the
crash.

**Step 3: Observe the crash in WinDbg.** When the null pointer
dereference occurs, WinDbg breaks execution and prints an exception
message. We expect an **access violation (0xC0000005)** since we tried
to write to address 0. For example, WinDbg might show:

    (1234.5678): Access violation - code c0000005 (first chance)[8]

This means thread `1234.5678` threw an access violation. "First chance"
indicates the exception is caught at the point of occurrence (you get a
"second chance" message if it was unhandled and the program is about to
terminate). In our case it's unhandled, so WinDbg will catch it as a
second-chance exception too. The key part is `code c0000005`, the code
for access violation, and it likely says **reading or writing address
0x00000000**. WinDbg might also indicate if it was a read or write. If
you see
`Access violation - code c0000005 (first/second chance not available)`
that means it's breaking on an unhandled
exception[\[8\]](https://levelblue.com/blogs/spiderlabs-blog/windows-debugging-exploiting-part-3-windbg-time-travel-debugging#:~:text=0%3A000,first%2Fsecond%20chance%20not%20available).

Now the program is paused exactly at the faulting instruction. The
command prompt changes to something like `0:000>` (or maybe `0:001>` if
multiple threads). We're in break mode ready to investigate.

**Step 4: Check the call stack.** Use the `k` command to get a stack
trace. `k` (or `kb` for more detail) will list the stack frames.
Example:

    0:000> k
     # Child-SP          RetAddr           Call Site
    00 0019fed0 00401016 nullptr_bug!foo(int*) [nullptr_bug.cpp @ 6] 
    01 0019fee0 00401030 nullptr_bug!main [nullptr_bug.cpp @ 11] 
    02 0019ff20 76ab7bd4 nullptr_bug!__tmainCRTStartup ... 
    03 0019ff30 76c0ce51 KERNEL32!BaseThreadInitThunk ...
    04 0019ff60 76d8e4b1 ntdll!RtlUserThreadStart ...

Frames **0** and **1** correspond to our code (addresses in
`nullptr_bug!foo` and `nullptr_bug!main`). We see `foo(int*)` at line 6
and `main` at line 11. The rest are CRT and system startup frames. So
just like in LLDB, we know we crashed in `foo`, called from `main`.
Good.

**Step 5: Inspect variables in the crashing frame.** We want to see the
value of `ptr` in `foo`. Since we have debug symbols, we can use the
**Local Variables window** or the `dv` command. Let's try `dv` (Display
Variables):

    0:000> dv
                   ptr = 0x00000000

WinDbg shows that `ptr` is 0x00000000 (NULL). That confirms our
suspicion. We could also use the newer `.frame` syntax or the
NatVis/WinDbg Preview Locals pane, but `dv` is quick and shows the local
variables and arguments in the current
frame[\[9\]](https://medium.com/ax1al/introduction-to-windbg-1177487c77a4#:~:text=Medium%20medium,the%20type%20of%20the).
We see `ptr` is null. Alternatively, we could evaluate `ptr` with the
`?` or `dx` command (for instance, `? ptr` or `dx ptr`), but `dv`
already gave us the answer in a straightforward way.

**Step 6: (Optional) Breakpoints and watchpoints.** In this simple run,
we didn't set any breakpoints -- we just let the program crash. If we
wanted, we could have set a breakpoint at `nullptr_bug!foo` before
running. In WinDbg, you can set a breakpoint by function name or source
line. For example:

    0:000> bu nullptr_bug!foo    ; (bu = break on symbol, unresolved is okay)
    0:000> g

This would break at the start of `foo`. Once hit, we could inspect `ptr`
before it's used. In more complex scenarios, that's useful to catch
issues before the actual crash.

For *watchpoints* (called **data breakpoints** in WinDbg), WinDbg uses
the `ba` (Break on Access) command. In this case, a watchpoint isn't
necessary, but to illustrate: if we wanted to break when a certain
memory address is accessed, we could do `ba <access> <size> <address>`.
For example, `ba w 4 0x0` would break on any write of 4 bytes to address
0 (not very useful here since that always crashes anyway). A more
realistic use: we'll see in the next case how to break on access to
freed memory. Just remember that in WinDbg, `ba w 4 <addr>` means "break
on 4-byte writes to
\<addr\>"[\[10\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/ba--break-on-access-#:~:text=Option%20Action)[\[11\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/ba--break-on-access-#:~:text=This%20example%20sets%20a%20conditional,the%20debugger%20should%20break%20in).

**Step 7: Identify the root cause.** The call stack and variable state
make it clear: `ptr` was null when we tried to use it, leading to the
access violation. The senior engineer concludes: *"We passed a null
pointer into* `foo` *-- there's no check, so it crashed. The bug is the
null pointer usage."*

We'd fix it similarly (initialize `p` or avoid calling `foo` with null).
WinDbg successfully helped us confirm the issue.

### Debugging with VS Code + **CodeLLDB** (macOS)

Now let's see the same bug through the lens of VS Code on macOS, using
the **CodeLLDB** extension. This gives us a graphical interface but
still uses LLDB under the hood. The process will be familiar but with a
friendlier UI.

**Step 1: Compile the program.** We use the same compile command as in
the LLDB section (clang++ or g++ with `-g`). Suppose we have
`nullptr_bug.cpp` in a VS Code workspace. We compile it (via a VS Code
task or manually in Terminal):

    clang++ -g -O0 -o nullptr_bug nullptr_bug.cpp

**Step 2: Set up debugging in VS Code.** Open the folder in VS Code.
Install the **CodeLLDB** extension (if not already). Go to the Run and
Debug view (the Debug sidebar) and create a launch configuration. VS
Code might auto-detect and suggest a configuration for LLDB. For
example, your `.vscode/launch.json` might have:

    {
        "name": "Debug nullptr_bug",
        "type": "lldb",
        "request": "launch",
        "program": "${workspaceFolder}/nullptr_bug",
        "args": [],
        "cwd": "${workspaceFolder}"
    }

This tells VS Code to use the CodeLLDB debugger to launch our compiled
program. (The `"type": "lldb"` comes from the CodeLLDB extension.) Make
sure the path to the program is correct.

**Step 3: Set a breakpoint in the source.** Open `nullptr_bug.cpp` in
the editor. Click in the gutter next to the line where `foo`
dereferences the pointer (line 6) or at the call in main. As a senior
engineer, I might set a breakpoint at `foo(ptr=...)*ptr = 42` line to
pause just before the crash. A red dot will appear indicating the
breakpoint is set. (Alternatively, you could skip setting any breakpoint
-- the debugger will stop on the exception anyway. But let's do it to
demonstrate normal debugging.)

**Step 4: Start debugging.** Click the **Run and Debug** button (green
triangle or press F5). VS Code will launch the program under the
debugger. If you set a breakpoint, execution will pause at that line. If
not, it will run and then pause when the crash occurs.

Let's assume we set the breakpoint at `*ptr = 42`. The debugger stops
there, highlighting the line in the source code. In the **Debug
toolbar** you'll see controls (Continue, Step Over, Step Into, etc.),
and in the sidebar you'll see **Variables, Call Stack, Breakpoints,**
etc. The **Variables** pane shows local variables in the current
function. We should see something like:

- **Local**: `ptr = 0x0`

Yes, VS Code's Variables view will show that `ptr` is `0x0` (null) at
this breakpoint. We haven't even continued to the crash -- we caught it
right before. This is great: as a senior engineer, I suspected a null
pointer, and the breakpoint confirms it.

**Step 5: Step or continue to observe the crash.** If we hit the
**Continue** button (or press F5 again), the program will attempt to
execute the `*ptr = 42` line. Since `ptr` is 0, a crash will occur.
CodeLLDB will then break execution and likely show a popup or message
like "Program received signal SIGSEGV". VS Code might switch to a
disassembly view if it doesn't know where in source to show the crash
(since we continued past our last breakpoint into a crash). Don't worry
-- we can still debug. The **Call Stack** pane now likely shows an entry
for `foo` and `main`, similar to before. You can click on `foo` in the
call stack to open the source at that frame. The Variables pane still
shows `ptr = 0x0`.

**Step 6: Inspect variables and call stack.** In VS Code, much of this
is done via the UI: the call stack is already visible, and local
variables are listed. We can hover over `ptr` in the editor to see its
value (a tooltip shows 0x0). We can also open the **Debug Console** and
use LLDB commands or expressions. For example, we could type `print ptr`
or `p ptr` in the Debug Console. CodeLLDB will execute that via LLDB and
print the result. Since `ptr` is simple, the UI already told us it's
null, so we're good.

If we wanted the backtrace textually, we could use the Debug Console to
run the `bt` command (preface it with `-exec` if needed, e.g.
`-exec bt`). But visually we can see `foo -> main` in the stack list.

**Step 7: (Optional) Use watchpoints or advanced breakpoints.** VS
Code's UI (with CodeLLDB) doesn't have a dedicated "data breakpoint"
button like the MSVC/GDB version does, but we can still do it via the
LLDB console. For instance, in the Debug Console we could run:

    -exec watchpoint set variable ptr

However, in this scenario we already know `ptr` is null from the get-go.
We'll showcase watchpoints in the next case where it's more applicable.
Just know that in CodeLLDB you have the full power of LLDB commands
available in the console, and you can also set *conditional breakpoints*
in VS Code (right-click a breakpoint -\> Edit Condition) if you only
want to break when a certain condition is true (e.g., break when
`ptr == nullptr`).

**Step 8: Diagnose the issue.** The outcome in VS Code/CodeLLDB is the
same: we see that `ptr` is null and that caused a segmentation fault.
The senior dev confirms the bug is using a null pointer. We stop
debugging (Shift+F5) and go fix the code.

Using VS Code in this way can be more intuitive for those who prefer GUI
debugging: you click breakpoints and inspect variables without
memorizing commands. But under the hood, the process and the information
we gather are identical to the LLDB command-line session we did first.

### Debugging with VS Code + **CppDbg** (Windows / GDB)

Finally, let's reproduce the debugging process on Windows using VS Code
with the default C++ extension (cpptools), which we'll configure to use
GDB. This setup is common if you're using MinGW or WSL for C++
development in VS Code. (If you have Visual Studio's debugger, you might
use the `"cppvsdbg"` type instead, but here we'll assume a GCC/GDB
toolchain on Windows for variety.)

**Step 1: Compile the program with debug info.** If using MinGW-w64 g++,
you'd do:

    g++ -g -O0 -o nullptr_bug.exe nullptr_bug.cpp

This produces `nullptr_bug.exe` with DWARF debug symbols that GDB can
use. If you're instead using MSVC but still want to debug in VS Code,
you could compile with `/Zi` and configure VS Code for `"cppvsdbg"`
(Visual Studio debugger) -- but we'll stick to GDB for now.

**Step 2: Set up VS Code debugging (cppdbg).** In VS Code on Windows,
install the **C/C++ extension** by Microsoft (if not already). Open the
folder with the program. In the Run and Debug tab, create a launch
configuration and choose the C++ (GDB/LLDB) template. You'll get a
`launch.json` entry like:

    {
        "name": "Debug nullptr_bug (GDB)",
        "type": "cppdbg",
        "request": "launch",
        "program": "${workspaceFolder}/nullptr_bug.exe",
        "cwd": "${workspaceFolder}",
        "stopAtEntry": false,
        "console": "externalConsole",
        "MIMode": "gdb",
        "miDebuggerPath": "C:/mingw/bin/gdb.exe"
    }

Key points: `"type": "cppdbg"` and `"MIMode": "gdb"` tells VS Code to
use GDB. Make sure `miDebuggerPath` points to your GDB. Now we're set to
debug with GDB through VS Code.

**Step 3: Set a breakpoint.** Just like before, open the source file in
VS Code and click to set a breakpoint at the `*ptr = 42` line (or
anywhere you want to pause). Let's do the same line.

**Step 4: Start debugging.** Press F5 to launch. An external console
opens (since we set `"externalConsole": true` for MinGW), and VS Code
attaches GDB to our program. The program will run until the breakpoint
is hit. VS Code will highlight the line and show "Paused on breakpoint".

Look at the **Variables** pane -- with GDB it should show local
variables as well. You'll see `ptr = 0x0`. The **Call Stack** pane shows
`foo` and `main` similar to before. So far, so good.

**Step 5: Continue to crash.** If we continue execution, the program
will attempt the write and crash. GDB will stop with a SIGSEGV signal.
VS Code should display something like "Segmentation fault (SIGSEGV)" in
the Debug Console or as a notification. The stack and variables are
still available for inspection. You might see the console output say
\"**Program received signal SIGSEGV, Segmentation fault.**\" and
indicate the line of crash. GDB has broken at the point of failure.

**Step 6: Inspect the state.** We again confirm `ptr` is null (0x0) via
the Variables pane. We can also use the **Debug Console** to run GDB
commands if needed (for example, `info locals` or `print ptr`). The call
stack is visible; if you expand the stack frames, it may show
`foo(int*)` and `main` with line numbers. Everything confirms the null
pointer scenario.

**Step 7: Data breakpoints (watchpoints) in VS Code/GDB.** While not
needed here, it's worth noting that the VS Code C++ extension supports
**Data Breakpoints** with GDB. For instance, we could have set a data
breakpoint on `ptr` if it was a global or on an address. In VS Code, you
can right-click a variable in the Variables list and choose **"Break on
Value Change"** to create a data
breakpoint[\[12\]](https://devblogs.microsoft.com/cppblog/whats-new-for-c-debugging-in-visual-studio-code/#:~:text=To%20set%20a%20data%20breakpoint,select%20Break%20on%20Value%20Change).
(This sets a hardware watchpoint in GDB for that variable's address.) If
we had done that for a global pointer or some memory address, the
debugger would break when it changes. For a local like `ptr`, this
feature is more useful in later scenarios (like watching a global
pointer that might be freed by another thread). We'll use this in the
multi-threaded case.

**Step 8: Conclusion of analysis.** The VS Code + GDB run yields the
same conclusion: the pointer is null and that's the bug. The senior
engineer notes the fix as before. We stop debugging and resolve the
issue in code.

------------------------------------------------------------------------

We have now debugged the same null pointer crash with four different
tools/environments. The outcome was identical in each: by using
breakpoints or catching the exception, we inspected the call stack and
saw the null pointer value, pinpointing the root cause. Along the way,
we used each tool's features (like `bt` in LLDB/WinDbg, the VS Code UI,
etc.).

Before moving on, key transferable techniques from this case: always
compile with debug symbols for easier
debugging[\[1\]](https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html#:~:text=match%20at%20L204%20The%20compiler,as%20opposed%20to%20deployment),
let the program run under the debugger to catch crashes, use the call
stack to find where the crash happened, and inspect variable values to
diagnose the cause. Next, we'll apply these techniques to more complex
bugs.

## Case Study 2: Use-After-Free Memory Error (Heap Corruption)

**The Bug:** In this case, we have a program that allocates memory,
frees it, and then mistakenly uses the freed memory. This is a
*use-after-free* bug. Such issues can cause crashes, memory corruption,
or bizarre behavior. We'll use a simple example that may or may not
crash immediately, illustrating how debuggers can help catch the misuse
of freed memory.

    #include <iostream>
    #include <cstdlib>

    int main() {
        int *arr = (int*) std::malloc(5 * sizeof(int));
        for (int i = 0; i < 5; ++i) arr[i] = i;   // fill the array
        std::free(arr);
        // Bug: use-after-free
        std::cout << "Element[0] after free: " << arr[0] << std::endl;
        // Also a double free for demonstration
        std::free(arr);  // freeing again - double free
        return 0;
    }

In this code, we free `arr` and then **immediately try to read**
`arr[0]`. This is undefined behavior -- the memory is freed, so reading
it is an error. We even call `free(arr)` a second time, which is a
double free bug. Depending on the environment, this might crash or it
might appear to "work" while corrupting the heap. For example, on some
systems this program might print the old value for `arr[0]` and only
crash later (or terminate normally but poison the heap). On others, the
runtime might detect the double free and abort the program. We'll see
how to debug such issues by detecting the misuse of freed memory.

### Debugging with LLDB (macOS)

**Step 1: Compile with debug info.** As before, compile with `-g`:

    clang++ -g -O0 -o use_after_free use_after_free.cpp

**Step 2: Run under LLDB.** Launch LLDB with the program:

    lldb use_after_free
    (lldb) run

Now, observe what happens. Since this bug may not crash immediately, we
might see the program's output in LLDB:

    /*** Running ***/
    Element[0] after free: 0
    Process 18103 exited with status 0

Imagine LLDB shows that the program ran to completion printing
"Element\[0\] after free: 0". No obvious crash happened. That's actually
worse -- the bug didn't crash, so it could silently corrupt memory. The
senior engineer suspects something's wrong because reading memory after
free is illegal. How do we catch it? We use debugger tools smartly:

**Step 3: Break on the** `free` **and watch the memory.** A technique
for use-after-free: set a breakpoint at `free()` to capture the address
being freed, then set a *watchpoint* on that memory so we break if it's
accessed later.

In LLDB: we set a breakpoint on `free` (the C library function).

    (lldb) b free
    Breakpoint 1: where = libsystem_malloc.dylib`free, address = 0x7fff... 

Run the program again: `(lldb) run`. The program will break when it
calls `free(arr)`. We hit the breakpoint inside the `free` function
(LLDB will show a stop in the memory allocator code). We can get the
argument to `free` by inspecting registers or use a backtrace to our
code. An easier way: look at the call stack to find our `main` frame.
Use `bt` and find the frame in our code that called free. For example:

    (lldb) bt
    * thread #1, stop reason = breakpoint 1
      * frame #0: 0x7fff... libsystem_malloc.dylib`free
        frame #1: 0x0000000100000abc use_after_free`main + 60 at use_after_free.cpp:7

Frame #1 is our `main` at the line calling `free`. Let's select that:

    (lldb) frame select 1
    (lldb) print arr
    (int *) $0 = 0x0000000102b00510

We got the address of `arr` that's being freed (say it's `0x102b00510`).
Now we'll set a watchpoint on that memory. We know `arr` points to 5
ints, so 20 bytes. We can watch the first 4 bytes for any access
(read/write).

    (lldb) watchpoint set expression -w read_write -- arr
    Watchpoint created: Watchpoint 2: addr = 0x0000000102b00510 size = 8 state = enabled

We set a watchpoint on that address (LLDB chooses size 8 by default
here, which covers at least the first int and
more)[\[13\]](https://gist.github.com/rais38/6ecead686af9291da542#:~:text=Set%20a%20watchpoint%20on%20a,a%20condition%20on%20a%20watchpoint).
Now continue execution:

    (lldb) c   // continue

LLDB will continue until something accesses that memory. In our program,
the next line tries `arr[0]` which is a read of the just-freed address.
That should trigger our watchpoint. Indeed, LLDB should stop and report
the watchpoint:

    Process 18103 stopped
    * thread #1: stop reason = watchpoint 2
        frame #0: 0x0000000100000aeb use_after_free`main + 75 at use_after_free.cpp:8
       6    std::free(arr);
    -> 7    std::cout << "Element[0] after free: " << arr[0] << std::endl;

It breaks at line 7 (the use-after-free) because the watchpoint caught
our read of the freed memory. LLDB prints that it's a watchpoint and
we're in `main` at that
line[\[14\]](https://gist.github.com/rais38/6ecead686af9291da542#:~:text=...%20%28lldb%29%20bt%20,int32_t%29%20global%20%3D%205).
Now we caught the bug red-handed *before* any weird behavior.

**Step 4: Inspect variables and memory.** We know `arr` is the freed
pointer. We can confirm:

    (lldb) print arr
    (int *) $1 = 0x0000000102b00510

It's the same address freed earlier. If we want, we can check if maybe
the allocator has overwritten it with a pattern. Often, debug allocators
might fill freed memory with 0xDEADBEEF or 0xDD... in hex. We can
examine memory:

    (lldb) memory read 0x0000000102b00510 0x0000000102b00530

This would show the bytes at that address. (We might see some pattern or
the old data.) The specifics aren't crucial -- the main point is we
**caught a use-after-free**. The senior engineer's thought: *"The
program is reading memory that was freed. That's a bug. Also, it calls
free twice on the same pointer, which is another bug."*

**Step 5: Double-free detection.** After the watchpoint break, if we
continue, the program may hit the second `free(arr)`. Many allocators
will detect this and abort. If we hadn't caught the watchpoint, LLDB
likely would have stopped when the second free caused a crash or an
abort signal. For example, macOS's malloc might throw an error and halt
the program. In LLDB, you'd see a stop with a message about double free
or an `abort` signal. We can simulate continuing:

    (lldb) c
    Process 18103 stopped
    * thread #1: stop reason = signal SIGABRT
        frame #0: 0x7fff... libsystem_malloc.dylib`free

This indicates the program aborted (SIGABRT) in `free`, likely due to
heap corruption (double free). We could again inspect the call stack and
see it was the second call to free in `main`.

**Step 6: Diagnose the issue.** Using LLDB, we systematically caught the
misuse. The output "Element\[0\] after free: 0" was a red flag (why is
it printing a value after free?). By setting a watchpoint, we confirmed
`arr[0]` was accessed after free. The double free was caught by the
runtime abort. The root causes: use-after-free and double-free. A fix
would be to remove the extra `free` and ensure we don't use memory after
freeing (perhaps set `arr = nullptr` after free and check it).

The key takeaway here is how to use breakpoints and watchpoints to catch
memory errors *even when the program doesn't crash exactly at the point
of misuse*. We essentially forced a stop at the point of invalid access.

### Debugging with WinDbg (Windows)

On Windows, the C runtime has its own debug mechanisms for heap issues,
but let's see how to catch the use-after-free with WinDbg.

**Step 1: Compile for Windows (Debug mode).** If using MSVC, compile
with debug symbols (`/Zi`). If using MinGW, similar approach -- but
let's assume MSVC for WinDbg to leverage the CRT's checks. For instance:

    cl /Zi /Od /RTC1 /MDd use_after_free.cpp /Fe:use_after_free.exe

Here, `/RTC1` (Runtime Checks) and `/MDd` (Debug CRT) enable extra heap
checks. The Debug CRT will often catch double frees by crashing
immediately with an error message. So our program might actually crash
on the second `free` with an assertion. If that happens, WinDbg will
break on it. Let's proceed.

**Step 2: Launch under WinDbg and run.**

- Start WinDbg and open `use_after_free.exe` (File -\> Launch
  Executable).
- Set a breakpoint at the C runtime free or at our code after the first
  free. One strategy: break on `ntdll!RtlFreeHeap` which is underlying
  Windows heap free, or on the CRT `free`. For simplicity, let's break
  on `free` like we did in LLDB:

<!-- -->

    0:000> bu ucrtbased!free

(`ucrtbased` is the debug CRT library in MSVC; in release it would be
`ucrtbase` or MSVCRT.) WinDbg sets the breakpoint.

- Now `g` to run. The program breaks at the first `free(arr)`. We see a
  break in `ucrtbased!free`.

**Step 3: Find the address being freed.** Similar to LLDB, check the
call stack to find our frame:

    0:000> k
     # Child-SP    RetAddr     Call Site
    00 0019f7c0 74f3abc1 ucrtbased!free(void * block = 0x00a51020) 
    01 0019f7d0 01001347 use_after_free!main+0x47 [use_after_free.cpp @ 6]
    ...

Frame 0 is inside free, it shows the argument `block = 0x00a51020` (the
address freed). Frame 1 is our `main` at line 6. So the pointer value is
`0x00a51020`. We can note this or even copy it.

**Step 4: Set a watchpoint (break on access).** In WinDbg, we use the
`ba` command. We want to break on any access (read or write) to, say,
the first 4 bytes of that freed block. We take the address `0x00a51020`.

    0:000> ba r4 00a51020 

This sets a data breakpoint on a 4-byte read (actually `r` means
read/write) at that
address[\[10\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/ba--break-on-access-#:~:text=Option%20Action)[\[15\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/ba--break-on-access-#:~:text=Size%20Specifies%20the%20size%20of,e%2C%20Size%20must%20be%201).
Now continue:

    0:000> g

The program continues. Now when it executes `std::cout << arr[0]`, that
tries to read `arr[0]` at `0x00a51020`. Our watchpoint should trigger on
that memory access. WinDbg breaks and shows something like:

    (1234.5678): Break instruction exception - code 80000003 (first chance)
    Breakpoint 2 hit
    use_after_free!main+0x5A at use_after_free.cpp:7

It indicates our data breakpoint hit at line 7 of main. Great -- we
caught the use-after-free access. We can now inspect:

    0:000> dv
           arr = 0x00a51020

`arr` is still that address, now invalid. If we dump memory at that
address, the debug CRT might have filled it with a pattern (0xFEEE or
0xDD etc). We can try:

    0:000> dd 00a51020 L1
    00a51020  ???????? 

Sometimes in WinDbg, a freed block might show specific hex (in debug
mode, freed heap memory is usually filled with `0xFEEEFEEE` in Windows).
But even if not visible, we *know* it's freed.

**Step 5: Continue to catch double free.** If we continue again, the
second `free(arr)` will be called. The debug CRT should detect that this
block was freed already and likely trigger an assertion or throw a
safeguarded exception. WinDbg would then break (probably with an access
violation or an `_CrtIsValidHeapPointer` assertion). For example, you
might see:

    (1234.5678): Invalid address specified to RtlFreeHeap( 00A50000, 00A51020 )

and an access violation inside heap code. The exact message can vary,
but the bottom line: WinDbg will catch it (as a second-chance exception
if the CRT aborts). The call stack would show the second call to `free`
from main.

**Step 6: Analyze root cause.** We've used WinDbg to verify the program
freed memory and then accessed it (our watchpoint proved that), and
attempted a double free. The senior engineer notes: *"We have a
use-after-free (the program read memory after freeing it) and a double
free. These are serious memory corruption bugs."*

The debugging strategy here was crucial: because the program didn't
outright crash on the first invalid access, we had to proactively break
on suspicious behavior (like calling free, then watching that memory).
This is a common pattern in debugging heap corruption -- catch the
moment of misuse rather than the aftermath.

### Debugging with VS Code + CodeLLDB (macOS)

In VS Code with CodeLLDB, we don't have an out-of-the-box UI feature to
set a watchpoint on an arbitrary address, but we can still catch this
bug using a mix of breakpoints and the LLDB console.

**Step 1: Compile with -g** (as done before). Open in VS Code, ensure
CodeLLDB is configured.

**Step 2: Set a breakpoint on** `free`**.** This is tricky in VS Code's
UI because `free` is a C library function not in our code. But we can
use the debug console. Alternatively, you can add an entry in
`launch.json` for a function breakpoint (VS Code allows specifying
function names in the launch config's `"functionBreakpoint"` section or
using the UI's **Add Function Breakpoint** in the Breakpoints view).
Let's do that: in VS Code's Breakpoints pane, click **\"+\"** \>
**Function Breakpoint**, and enter `free`. VS Code (via LLDB) will set a
breakpoint on `free()`.

**Step 3: Start debugging.** Run under the debugger. The program will
break when `free` is called (just like our LLDB CLI session did). VS
Code will likely show assembly or indicate we're in `free` (which is
outside our source). Look at the call stack: one frame will be `free`,
and below it our `main`. Click the `main` frame to see our code. We're
at the line after free returned (or still on the free call). The
Variables pane shows `arr` and its value.

**Step 4: Use LLDB console to set a watchpoint.** Open the Debug Console
in VS Code. We'll issue an LLDB command to set a watchpoint on `arr`'s
memory. For example:

    -exec watchpoint set expression -w read_write -- (void*)arr

This tells LLDB to break on read/write to the address in `arr`. (We cast
to void\* to match expression context). LLDB should respond with a
watchpoint created message. Now continue execution:

    -exec continue

(This is equivalent to clicking Continue). When the program tries to
read `arr[0]`, LLDB triggers the watchpoint and VS Code will break
again. It may show a message like "Watchpoint 2 hit". We're at the line
of `std::cout << arr[0]`. Great. Now we know a use-after-free is
happening.

**Step 5: The Variables pane** will still show `arr` (address). At this
point, we know it's freed. We can optionally verify if CodeLLDB shows
any clue (in some cases, the memory might show as `<invalid address>` if
the debug runtime set protections, but likely it just appears as an
address). Regardless, we've confirmed an access to freed memory.

**Step 6: Continue to see double free reaction.** Continue again. If the
program aborts on double free, CodeLLDB will break on the abort signal
(SIGABRT). VS Code might pop up "Program terminated with signal
SIGABRT". The call stack will show it coming from `free` second time. We
gather the evidence.

**Step 7: Conclusion.** The VS Code approach required some console
commands for watchpoints, but it allowed us to catch the issue
similarly. The takeaway is that even in VS Code, you can combine UI and
debugger console to leverage advanced features like watchpoints.

*(If one were using AddressSanitizer or other tooling, it would catch
these errors too, but here we're focusing on manual debugging
techniques.)*

### Debugging with VS Code + CppDbg (Windows)

Using VS Code with the Microsoft C++ extension (and GDB on Windows), we
can attempt a similar strategy. Note: The cpptools extension has a UI
for data breakpoints (as mentioned). Let's exploit that.

**Step 1:** Compile with MinGW (or MSVC). If MinGW's GDB, same `-g -O0`.
Launch VS Code with GDB as configured.

**Step 2:** We can try setting a breakpoint on free by function name in
VS Code as well (similar to above, add a Function Breakpoint for
`free`). Alternatively, since the MS C++ extension supports data
breakpoints directly, we can do a simpler trick: after the first free,
set a data breakpoint on `arr`. But we need to pause after free to do
that.

Maybe simpler: set a normal breakpoint at the line *after* the first
`free(arr)` in our code (i.e., at the `std::cout << arr[0]` line). When
we hit that, `arr` is already freed, but we have its value. We can then
set a data breakpoint.

**Step 3:** So set a breakpoint at the `std::cout << ... arr[0] ...`
line (line 7). Start debugging. The program runs, frees arr (not caught
yet), and stops at line 7 due to our breakpoint. Now in the
**Variables** pane, we see `arr` and its value. Now right-click on `arr`
and choose **"Break on Value Change"** (this is the data breakpoint
feature)[\[12\]](https://devblogs.microsoft.com/cppblog/whats-new-for-c-debugging-in-visual-studio-code/#:~:text=To%20set%20a%20data%20breakpoint,select%20Break%20on%20Value%20Change).
However, "Break on Value Change" is intended to break when the *value of
a variable* changes, which isn't exactly our scenario -- we want to
break on access to the memory pointed by arr. The cpptools data
breakpoint actually sets a hardware watchpoint on that memory address
(it works for global/static variables or heap addresses, up to 4 or 8
bytes). If `arr` were global, it'd break when `arr` changes. But here,
we want to watch the pointed memory.

We might have to use the Debug Console with GDB commands: `watch *(arr)`
in gdb could watch the memory pointed by arr. Let's do that: in Debug
Console, type:

    -exec watch *(int*)arr

GDB will set a watchpoint on that address. Now continue (F5). The
watchpoint should trigger immediately because we are about to read it
(actually, we're at the line already, if we step or continue, it will
try to execute that read). If the breakpoint is set correctly, GDB will
break on the memory access. VS Code will indicate a break (maybe showing
the watchpoint in the Breakpoints pane as well).

Now we know the watchpoint hit, meaning an access to freed memory
happened. The output may not have even printed yet, and we caught it.

**Step 4:** Continue after that, and the double free likely causes a
crash which GDB will catch as well (maybe an abort or another watchpoint
if we set one for free).

Admittedly, data breakpoints are a bit clunky in VS Code with GDB for
this scenario. Another approach: If using WSL or Linux environment, you
could use Valgrind or AddressSanitizer, but we're focusing on the
debugger.

**Step 5:** Summarize findings. The VS Code + GDB approach (cppdbg)
confirms the same problem through manual watchpoint usage. The senior
engineer notes that using the debugger, we intercepted the invalid
memory usage which otherwise might have gone unnoticed or only shown up
as a crash much later.

------------------------------------------------------------------------

In this case study, we demonstrated how to debug a tricky memory
corruption where the program's failure isn't immediate. **Key
techniques:** using breakpoints on library calls (like `free`), and
using **watchpoints (data breakpoints) to catch any access to freed
memory**. All debuggers -- LLDB, WinDbg, GDB -- have this
capability[\[5\]](https://gist.github.com/rais38/6ecead686af9291da542#:~:text=Set%20a%20watchpoint%20on%20a,my_ptr)[\[11\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/ba--break-on-access-#:~:text=This%20example%20sets%20a%20conditional,the%20debugger%20should%20break%20in).
In VS Code, the UI provides data breakpoints for certain scenarios, or
you can invoke them via the console. The thought process is: *"I suspect
a use-after-free; let's catch the moment the freed memory is touched."*
This way, we caught the bug in action. The root cause analysis is
straightforward once you see that: freeing memory and then using it is a
bug, as is freeing it twice.

Next, we'll explore a different category of bug: **stack overflow due to
recursion**, to see how debuggers handle deep call stacks.

## Case Study 3: Stack Overflow (Infinite Recursion)

**The Bug:** Our third scenario is a function that inadvertently calls
itself infinitely (a recursion bug), eventually causing a **stack
overflow** crash. This will test our debuggers' ability to handle deep
call stacks. The code:

    #include <iostream>

    unsigned int fib(unsigned int n) {
        if (n == 0) return 0;
        if (n == 1) return 1;
        // Bug: wrong recursion, calls fib(n) instead of fib(n-2)
        return fib(n-1) + fib(n);
    }

    int main() {
        std::cout << "fib(2) = " << fib(2) << std::endl;
        return 0;
    }

This intends to compute Fibonacci numbers, but there's a **bug in the
recursion**: the function calls `fib(n)` again instead of `fib(n-2)`. So
`fib(2)` calls `fib(1)` and `fib(2)` again, leading to an infinite
recursion. Eventually the program will crash with a stack overflow (when
the stack memory is exhausted). The behavior: it will likely print
nothing and then segfault with a stack overflow error.

### Debugging with LLDB (macOS)

**Step 1: Compile with -g.**

    clang++ -g -O0 -o fib_bug fib_bug.cpp

**Step 2: Run under LLDB.**

    lldb fib_bug
    (lldb) run

This program will run and eventually crash. LLDB will stop the process
at the crash point. On macOS, a stack overflow typically appears as an
**EXC_BAD_ACCESS** (access violation) at some stack address or just an
abrupt stop. LLDB might say something like:

    Process 18200 stopped
    * thread #1: tid 0x12345, 0x0000000100000e?? fib_bug`fib(n=2) + 87 at fib_bug.cpp:..., stop reason = EXC_BAD_ACCESS (code=2, address=0x7fff5fbfffc8)[3]

It shows we stopped in `fib(n=2)` with a bad access at some address near
the stack top (the address 0x7fff5fbfffc8 is likely near the end of
stack). This indicates a stack overflow.

**Step 3: Examine the stack trace.** The big clue for a recursion bug is
in the backtrace. Type `bt` in LLDB:

    (lldb) bt
    * thread #1: stop reason = EXC_BAD_ACCESS
      * frame #0: 0x0000000100000add fib_bug`fib(n=2) + 61 at fib_bug.cpp:??
        frame #1: 0x0000000100000af7 fib_bug`fib(n=2) + 87 at fib_bug.cpp:??
        frame #2: 0x0000000100000af7 fib_bug`fib(n=2) + 87 at fib_bug.cpp:??
        frame #3: 0x0000000100000af7 fib_bug`fib(n=2) + 87 at fib_bug.cpp:??
        ... (many repeated frames) ...
        frame #50: 0x0000000100000af7 fib_bug`fib(n=2) + 87 at fib_bug.cpp:??
        frame #51: 0x0000000100000c9e fib_bug`main + 78 at fib_bug.cpp:12

LLDB will list a **lot** of frames (it might truncate after a large
number). We can see `fib(n=2)` repeated many
times[\[16\]](https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html#:~:text=,fib%28n%3D2%29%20%2B%2061%20at%20lldbtutorial%3A18)[\[17\]](https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html#:~:text=,%2B%2078%20at%20lldbtutorial%3A80).
Frame #51 shows `main`. The repeated pattern indicates that `fib(2)` is
calling itself recursively and never stopping. The senior engineer
observes: *"The stack is filled with calls to fib(2). This must be
infinite recursion."*

We can scroll the stack (`bt` might show partial, but it's enough to see
the pattern).

**Step 4: Inspect variables (if needed).** We might inspect the value of
`n` in the top frame:

    (lldb) frame variable n
    (unsigned int) n = 2

It's always 2 in these frames, meaning the function keeps recursing with
the same `n`. That confirms the bug: the code didn't decrease `n`
properly in one branch. We might check one frame further down to see if
any difference, but likely all show n=2.

**Step 5: Breakpoints for insight (optional).** Another debugging
approach: set a breakpoint at the start of `fib` and run, then observe
the flow. For example,

    (lldb) b fib
    (lldb) run

Then manually step through a couple of calls with `n=2` to see what
happens. You'd see it call `fib(2)`, then inside call `fib(1)` and
`fib(2)` again, etc. After a few steps, the pattern becomes clear. In
practice, the backtrace from the crash gave it away directly.

**Step 6: Conclusion in LLDB.** The program crashed due to stack
overflow (EXC_BAD_ACCESS on stack). The cause is the infinite recursion
(buggy `fib`). The fix is to change `fib(n)` to `fib(n-2)` in the
return. The senior engineer's thought: *"We have a recursion that never
ends, causing a stack overflow. The debugger showed an endlessly
repeating call stack."*

### Debugging with WinDbg (Windows)

On Windows, an infinite recursion typically triggers a
**StackOverflowException** (exception code 0xC00000FD). WinDbg will
catch this when the stack is exhausted.

**Step 1: Compile with MSVC debug.** (Or MinGW, but MSVC's output might
give a clearer call stack). E.g.:

    cl /Zi /Od fib_bug.cpp /Fe:fib_bug.exe

**Step 2: Launch in WinDbg.**

    windbg fib_bug.exe

Run (`g`). The program will run until the stack overflow occurs. WinDbg
will then break execution and output something like:

    (1234.5678): Stack overflow - code c00000fd (first chance)[18]
    First chance exceptions are reported before any exception handling...

This indicates a stack overflow exception (code `0xc00000fd`). Since
it's unhandled, a second chance will also occur, but WinDbg stops on
first chance by default here.

**Step 3: Get the stack trace.** Use `k`. This stack will be huge.
WinDbg might not show all frames by default. It could show something
like:

    0:000> k
      *** Stack trace may be truncated, frame count 1024 ***
    00 fib_bug!fib(unsigned int)+0x?? 
    01 fib_bug!fib(unsigned int)+0x?? 
    02 fib_bug!fib(unsigned int)+0x?? 
    ... (repeats many times) ...
    nn fib_bug!fib(unsigned int)+0x?? 
    nn+1 fib_bug!main+0x?? 

It might display the first few and last few frames, with "\..."
indicating repetition. If WinDbg truncates, we can try `kc 100` to show
100 frames or so. But the pattern will be clear: multiple entries of
`fib_bug!fib`. We know it's repetitive.

We can also use `!analyze -v` (a WinDbg extension) which might summarize
that there was a stack overflow and possibly show the last calls. But
the manual approach suffices.

**Step 4: Check a couple of frames to confirm parameter values.** We can
try to look at the top of stack. But note: in a stack overflow, the
system might not be able to unwind all symbols properly. Still, if
symbols are there, WinDbg knows the function name.

We might do:

    0:000> .frame 0 
    0:000> dv

However, in a stack overflow, `dv` might fail ("unable to enumerate
locals") because of corruption or because it's too deep. Alternatively,
we can inspect a register for the parameter. On x64 Windows, `n`
(unsigned int) would be passed in ECX for recursive calls. The register
state might be lost. But given the repetitive calls, it's likely the
same scenario: n=2 throughout.

**Step 5: Analyze the problem.** WinDbg clearly indicated a Stack
Overflow. The call stack shows recursive `fib`. The senior engineer
infers the recursion bug. The output of the program (if any) didn't even
get to print because it overflowed before returning.

**Step 6: Alternative debug: set a breakpoint in fib and watch
recursion.** We could have done:

    0:000> bp fib_bug!fib
    0:000> g

It would break at the first call to `fib(2)`. Then we could use `t`
(trace) or `p` (step) to step through recursion, but given it's
infinite, stepping would be laborious. Instead, after a few steps you'd
see the pattern of calls. The backtrace from the crash is usually
enough.

In summary, WinDbg confirms infinite recursion by the nature of the
exception and the repetitive call stack.

### Debugging with VS Code + CodeLLDB (macOS)

**Step 1:** Compile with debug symbols (`-g`). Open in VS Code.

**Step 2:** Set a breakpoint at the start of `fib` (line 5).

**Step 3:** Run the debugger. It will hit `fib(2)` at the beginning. You
can inspect `n` (it's 2). Step into the recursion: - Step over the
`if (n==0)` and `if (n==1)` conditions until the return. - Step into the
`return fib(n-1) + fib(n)` expression. VS Code will go into the next
call of `fib`. Now you're in `fib` again with `n=1` (for the `fib(n-1)`
part). Step through that -- it will eventually return 1. - Then it will
call `fib(n)` where n is still 2 for the second part of the addition.
Now you're calling `fib(2)` again, starting the cycle anew.

You'll notice the debugger hitting the breakpoint in `fib` repeatedly.
The **Call Stack** pane in VS Code will start showing multiple `fib`
frames one under another as you step deeper. Eventually, if you just
continue (F5) without breakpoints, the program will run until it
overflows. VS Code will then stop with a message (it might say "Program
received signal SIGSEGV" or just exit unexpectedly). If CodeLLDB catches
it, you might not get a nice message like WinDbg's, but you'll know
something went wrong (process exited with code 11, etc.).

However, an interactive way: after noticing the repeat, you would deduce
the recursion bug and stop. You might not let it overflow fully since
the pattern is obvious. But if you did, CodeLLDB would break on the
segfault. The Variables and Call Stack at that moment would be similar
to LLDB's view (lots of frames of `fib`). You can scroll in the Call
Stack panel to see the repeating pattern.

**Step 4:** Analyze and conclude. The VS Code debugger reveals the
infinite recursion by the repeating calls. The fix is apparent once you
inspect the code logic.

### Debugging with VS Code + CppDbg (Windows)

**Step 1:** Compile with MinGW `-g`. In VS Code, set up GDB.

**Step 2:** We can attempt the same approach: set a breakpoint in `fib`
and debug. The GDB backend will catch it similarly. If we let it run to
crash, GDB will stop with something like "Program received signal
SIGSEGV" when the stack overflows (GCC doesn't throw a structured
exception; it'll just segfault on overflow). The call stack in VS Code
will show many `fib` frames.

**Step 3:** We inspect the call stack in VS Code. The UI might not list
all thousands of frames, but it will list as many as it can, showing
`fib` repeated and eventually main at the bottom. That's evidence
enough.

**Step 4:** The analysis is the same: stack overflow due to runaway
recursion.

------------------------------------------------------------------------

In this case study, the **transferable techniques** are: using the
debugger to catch a crash due to stack overflow and using the backtrace
to identify a repeating pattern of calls. All tools gave us a clue: -
LLDB showed repeated `fib(n=2)`
frames[\[16\]](https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html#:~:text=,fib%28n%3D2%29%20%2B%2061%20at%20lldbtutorial%3A18). -
WinDbg explicitly flagged a stack overflow
exception[\[18\]](https://stackoverflow.com/questions/18721841/how-to-find-the-source-of-a-stackoverflowexception-in-my-application#:~:text=that%20raised%20the%20exception%2C%20and,will%20be%20something%20like%20this). -
The VS Code debug views also showed many repeated calls.

The senior engineer's process is to recognize that pattern as a logical
bug (incorrect recursion stopping condition). No fancy watchpoints
needed here, just careful reading of the stack trace.

## Case Study 4: Data Race and Multithreading Bug (Concurrent Access)

**The Bug:** Our final example involves threads. We have two threads
accessing a shared pointer without proper synchronization. This can lead
to a *race condition*, where one thread frees memory while another
thread is still using it. This is a classic multi-threaded bug that can
cause crashes or unpredictable results depending on timing.

Consider:

    #include <thread>
    #include <chrono>
    #include <iostream>

    int *sharedPtr = nullptr;

    void writer() {
        sharedPtr = new int(42);
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
        // Free the memory and set pointer to null
        delete sharedPtr;
        sharedPtr = nullptr;
    }

    void reader() {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        // Attempt to use sharedPtr after it may have been freed
        if (sharedPtr) { 
            *sharedPtr = 10;  // BUG: possible use-after-free if writer freed it
        }
    }

    int main() {
        std::thread t1(writer);
        std::thread t2(reader);
        t1.join();
        t2.join();
        return 0;
    }

In this code, the **writer thread** allocates an int and after 50ms
deletes it and sets `sharedPtr` to null. The **reader thread** waits
100ms (ensuring writer has likely freed the memory) and then checks
`sharedPtr` and if not null, writes to it. The intention might have been
to read something, but here we simulate a write. There's a race:
depending on timing, `sharedPtr` could be non-null when checked (if
reader's 100ms sleep isn't long enough or if writer was slower), or it
could already be null.

- If `sharedPtr` is null, the `if` will skip and nothing happens (no
  crash).
- If `sharedPtr` is non-null (i.e., the writer hasn't nullified it yet
  or hasn't run delete yet, which is unlikely with these timings), then
  no issue (the write happens before deletion).
- The worst case is if writer deletes and sets it to null exactly
  between the if-check and the write in reader -- that's a tiny window.
  In our example, we set `sharedPtr = nullptr` in writer, so reader will
  likely see null and do nothing. But imagine if we *didn't* set it to
  nullptr in writer after delete (a more common bug scenario), then
  reader would definitely attempt to write freed memory.

Let's tweak for a more demonstrable crash: say the writer **does not**
set it to nullptr after delete (simulate forgetting to clear the
pointer):

    // ... in writer()
    delete sharedPtr;
    // sharedPtr not cleared here, still points to freed memory

Now the reader will find `sharedPtr` still pointing to something
(dangling) and try `*sharedPtr = 10`, which is a use-after-free. This
likely causes a crash. We'll debug that scenario. (The original code
with setting to nullptr actually turns the bug into a null-dereference
if the reader thread runs late, which would also crash at
`*sharedPtr = 10` since sharedPtr is null. That's equally illustrative.)

So either we debug a *dangling pointer race* or a *null pointer race*.
Both are results of a race condition. We expect an access violation in
the reader thread.

This is a multi-thread bug, so we'll see how to inspect thread states
and coordinate breakpoints.

### Debugging with LLDB (macOS)

**Step 1: Compile with -g (and consider** `-fsanitize=thread` **if we
wanted TSAN, but here we stick to manual debugging).**

**Step 2: Run under LLDB.**

    lldb race_bug
    (lldb) run

The program might or might not crash every time, depending on
scheduling. With the timing we gave (writer frees at 50ms, reader writes
at 100ms), the reader will likely crash because by 100ms the pointer was
freed and left dangling (not null). So assume it crashes. LLDB will stop
with an access violation in the reader thread.

It might show something like:

    Process 18300 stopped
    * thread #2, stop reason = EXC_BAD_ACCESS (code=1, address=0xXXXXXXXX)
        frame #0: 0x0000000100001040 race_bug`reader() + 32 at race_bug.cpp:16
       14    std::this_thread::sleep_for(std::chrono::milliseconds(100));
       15    // Attempt to use sharedPtr after free
    -> 16    *sharedPtr = 10;  // crash here
       17 }

This indicates **thread #2** (likely the reader thread) crashed trying
to write to an invalid
address[\[19\]](https://levelblue.com/blogs/spiderlabs-blog/windows-debugging-exploiting-part-3-windbg-time-travel-debugging#:~:text=%281f20.fc0%29%3A%20Access%20violation%20,first%2Fsecond%20chance%20not%20available).
The address might be some freed heap address. We see the source line.

**Step 3: Examine threads.** We have multiple threads. In LLDB, use
`thread list` to list threads:

    (lldb) thread list
    Process 18300 stopped
    * thread #1: tid = 0x1234, 0x... race_bug`writer()  (running or finished)
    * thread #2: tid = 0x1235, 0x... race_bug`reader()  stop reason = EXC_BAD_ACCESS

It shows thread #2 stopped due to the exception. Thread #1 might have
finished (or maybe still in join). We can switch to thread 1 to see its
state:

    (lldb) thread select 1
    (lldb) bt

If writer thread finished and is waiting in join or ended, the backtrace
might show it in `pthread_join` or somewhere, or nothing if it's
terminated. If it's still joining, frame might be in `main` at join
call.

Anyway, the main issue is with thread #2. We can inspect `sharedPtr`:

    (lldb) frame variable sharedPtr
    (int *) sharedPtr = 0xXXXXXXXX

It will show a dangling address (the one freed). Possibly if guard
malloc is on, accessing it triggers crash. We know it's invalid.

**Step 4: Use watchpoints to catch the race proactively.** Another
approach the senior engineer might use: set a watchpoint on `sharedPtr`
(the global pointer) to see when it changes or when it's accessed. For
instance, before running, in LLDB:

    (lldb) watchpoint set variable sharedPtr

This will break whenever `sharedPtr` is written (by default, watchpoints
monitor writes). So as soon as writer does `sharedPtr = new int(42)` or
later `sharedPtr = nullptr`, LLDB will break. This might be a bit too
many breaks, but let's say we did that.

- It breaks when writer thread writes to `sharedPtr` (setting it to new
  int). We continue.
- It breaks again when writer sets `sharedPtr = nullptr` after delete.
  We continue.
- Now, crucially, we could also watch memory at the allocated int
  address to catch use after free. But since we now know the crash
  happens, it's okay.

**Step 5: Multi-thread inspection.** We might want to see what happened
in thread scheduling. We know thread1 freed and nullified the pointer at
t=50ms. Thread2 attempted use at t=100ms. Actually in this scenario with
nullification, thread2 would crash on null deref. In the scenario
without nullification, thread2 crashed on dangling pointer. In any case,
the debugger caught it.

LLDB can also show thread states: `thread list` we did, and `bt all` can
show backtrace for all threads. If we do `bt all`, we might see:

    (lldb) bt all
    * thread #2: ... reader() frame 0 at race_bug.cpp:16 stop reason=EXC_BAD_ACCESS
      thread #1: ... main() frame 0 at race_bug.cpp:... (probably in join or after)

So we confirm thread2 died in `reader()`.

**Step 6: Conclusion from LLDB.** The senior engineer sees that the
reader thread crashed trying to access `sharedPtr`. The value was
already freed by writer. The root cause is a race: no synchronization,
leading to use-after-free across threads. The fix is to protect
sharedPtr with locks or use atomic operations or join timing such that
reader doesn't run after free.

### Debugging with WinDbg (Windows)

In WinDbg, multi-thread debugging means we have to pay attention to
thread context.

**Step 1: Compile for Windows.** e.g. MSVC with /Zi.

**Step 2: Launch WinDbg and run.** The program might crash similarly
with an access violation in one of the threads. WinDbg will break and
show something like:

    (1db4.3f48): Access violation - code c0000005 (first chance)

We need to see which thread crashed. WinDbg's default thread is the one
that hit the exception. Use `~` to list threads:

    0:002> ~
       0  Id: 1db4.3f48 Suspend: 0 Teb: ... Unfrozen (this might be thread 0)
       1  Id: 1db4.5c70 Suspend: 0 Teb: ... Unfrozen 
    .  2  Id: 1db4.4a88 Suspend: 0 Teb: ... Unfrozen  <-- the dot indicates current thread (#2)

Thread 2 is current (the one with the exception). Let's see its stack:

    0:002> k
     # Child-SP     RetAddr      Call Site
    00 002ff5c0     01101567     race_bug!reader(void)+0x27 [race_bug.cpp @ 16]
    01 002ff5d0     75c4f5f8     race_bug!invoke_thread_procedure+0x...   // thread startup internals
    02 002ff60c     77a25efa     KERNEL32!BaseThreadInitThunk+0x18
    03 002ff61c     77a25ec8     ntdll!RtlUserThreadStart+0x2a

Frame 0 is `reader()` at line 16, offset 0x27 (likely at
`*sharedPtr = 10`). So thread 2 (ID 4a88) crashed there.

Switch to thread 1 if needed:

    0:002> ~1s
    1:001> k
     # Child-SP     RetAddr      Call Site
    00 00aff7c4     77a8b4a2     ntdll!NtWaitForSingleObject+0x12
    01 00aff83c     75c4fa29     KERNEL32!WaitForSingleObjectEx+0x92
    02 00aff858     75c4f9e8     KERNEL32!WaitForSingleObject+0x12
    03 00aff874     0110148a     race_bug!std::this_thread::sleep_for<...> // maybe writer sleeping or join
    ...

Thread 1 might be writer or main waiting. Possibly thread 1 is main
waiting for join, and writer might have been thread 0 or vice versa. We
can identify by looking at code context if symbols allow.

Anyway, to keep it simple: the thread that crashed is the reader thread.
We see it clearly in the call stack.

**Step 3: Inspect variables.** We know `sharedPtr` is global. We can try
to evaluate it:

    2:002> dx sharedPtr
    sharedPtr : 0x00ab1230  (int*)    ; or use ? sharedPtr

It shows some address. That's the freed address presumably. If we have
debug heap, maybe it's overwritten. But anyway.

Alternatively, since reader had a parameter or local? Actually
`reader()` has no args. The pointer is global, so we just had to inspect
global.

**Step 4: Use WinDbg breakpoints/watchpoints.** Another approach: set a
breakpoint on `reader` or on the line of `*sharedPtr = 10` to catch it
before crash. E.g., use `.printf` or `bu race_bug!reader+0x27` (if we
compute offset) -- but better is to use a **conditional breakpoint or
guard**: In WinDbg, you could do a memory breakpoint on the pointer
address if you knew it, but that's tricky since pointer value changes at
runtime.

Alternatively, set a breakpoint on `delete sharedPtr` in writer thread,
then step to see what happens and quickly switch to reader thread.
That's complicated.

However, WinDbg has a nice feature: **Stop on exceptions**. We can
configure to break on access violation as soon as it happens (first
chance). It already does first chance break by default. So we got it at
the moment of fault.

**Step 5: Synchronization debugging tools:** If the bug was subtler (not
always crashing), one might use tools like Application Verifier or Intel
Inspector or ThreadSanitizer. But with pure WinDbg, we rely on catching
the crash or manual inspection with breakpoints.

We have enough with the crash caught.

**Step 6: Conclusion from WinDbg.** We identified the crashing thread
and location. The fix is to add synchronization. The senior engineer's
reasoning: *"The reader thread accessed memory that the writer thread
freed. This is a race condition."* We saw this because the crash
happened in the reader, and we know writer did a delete earlier. So the
remedy is to ensure proper ordering or locking.

### Debugging with VS Code + CodeLLDB (macOS)

**Step 1:** Compile with -g, open in VS Code.

**Step 2:** Set breakpoints or let it run. Multi-thread debugging in VS
Code: If we run without breakpoints, the program may crash. VS Code will
break on the crash (SIGSEGV). We'll then need to inspect threads.

So perhaps set a breakpoint at the problematic line in `reader`
(`*sharedPtr = 10`). Run the debugger. Possibly the writer thread runs
and frees before reader hits that line, but since we have a breakpoint
at that line, when the reader thread is about to execute it, it will
pause *before* actually doing it. This is great -- we catch it right in
time. The program will freeze when thread 2 reaches that line, while
thread 1 might have already done its job.

VS Code will show we are paused at line 16 (inside reader). In the call
stack, it will show that this pause is in thread 2 (it usually labels
threads like Thread (tid:0x..)). We can use the threads dropdown in VS
Code's debug toolbar to switch between threads. We might see Thread 1 in
some other location (maybe finished or in join).

**Step 3:** Check the value of `sharedPtr` now. In the Variables pane,
since `sharedPtr` is global, it might appear under "Globals" or we can
evaluate it in the Debug Console: `print sharedPtr`. It likely shows
`0x...` (some address). Is it null or an address? If writer set it to
nullptr already, it will show 0x0. If we simulate the scenario where
writer didn't null it, it shows a non-null address that's actually
freed. Either way, it's not a valid allocated memory. If it's 0, that
means the code is about to dereference a null -- a guaranteed crash. If
it's a dangling address, also a crash.

Now we know if we continue, it will crash. But we have essentially
caught the bug in the act (the breakpoint sort of acted like a
watchpoint on that code line).

**Step 4:** We can inspect thread 1 as well. Switch to thread 1 in VS
Code's thread picker. Maybe it's at `delete sharedPtr;` or past it. If
writer hasn't finished, maybe it's sleeping or just did delete and about
to set null (if that code existed). Possibly thread 1 is already done
and waiting in join in main. Either way, nothing more needed from it.

**Step 5:** We know the issue now: at this breakpoint, `sharedPtr` was
freed. If it's null, the code is about to do `*0 = 10`. If it's not null
(dangling), it's even more insidious because the `if(sharedPtr)` check
passes and we attempt to write an invalid location. In either case, we
have a bug.

**Step 6:** If we hit continue, we expect a crash. VS Code would then
stop the program with a SIGSEGV notice.

**Step 7:** Data breakpoint method: Alternatively, we could have tried
using the data breakpoint feature on `sharedPtr`. In VS Code, we could
right-click `sharedPtr` in the Variables pane and \"Break on Value
Change\". That would break when writer thread changes `sharedPtr` (on
allocation or deletion). That might help us see the timing. For
instance, break when `sharedPtr` becomes null. Then resume and then
break in reader. This is a bit advanced, but doable. It would show the
interleaving of operations.

**Step 8:** Conclusion for VS Code + CodeLLDB. We effectively identified
the race by catching one thread at the point of misuse. The solution is
to synchronize access to `sharedPtr` (e.g., join threads in proper order
or use a mutex or atomic).

### Debugging with VS Code + CppDbg (Windows)

Similarly, with VS Code and GDB or the Visual Studio debug engine: - We
can set a breakpoint at the write line in reader. Run, it pauses in
thread 2. We inspect the pointer. See it's invalid. - Or let it crash
and then inspect threads.

Using the MSVC debugger in VS Code (cppvsdbg) would give similar info:
it would break on the exception and you could see the thread and call
stack.

One helpful UI feature: VS Code shows a **Threads** view where each
thread can be selected, and you can see where each thread is stopped.
This is useful to verify thread 1 maybe completed, thread 2 is at crash.

In GDB, if you want to catch the race earlier, you might use a
conditional breakpoint or watchpoint as earlier cases.

Anyway, the process is analogous.

------------------------------------------------------------------------

In this multi-thread case, the **key techniques** are: understanding
thread scheduling, using breakpoints to pause one thread, using
thread-specific commands to examine each thread's stack (`thread list`
in lldb, `~` in WinDbg, thread dropdown in VS Code), and using
watchpoints or break-on-value-change to monitor shared variables across
threads[\[12\]](https://devblogs.microsoft.com/cppblog/whats-new-for-c-debugging-in-visual-studio-code/#:~:text=To%20set%20a%20data%20breakpoint,select%20Break%20on%20Value%20Change).
The senior engineer tracks the sequence of events: writer did X, reader
did Y, leading to crash.

By seeing that the crash happened in reader and knowing writer freed
memory earlier, we pinpoint the race condition. This is a transferable
approach: when debugging a multithreading bug, always identify which
thread crashed and what the other threads were doing.

## Conclusion and Key Takeaways

We've walked through four debugging scenarios (null pointer,
use-after-free, stack overflow, and a multithreaded race) using four
different tools. Despite the differences in interface, the **core
debugging steps** are consistent:

- **Compile with debugging symbols** to get meaningful
  information[\[1\]](https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html#:~:text=match%20at%20L204%20The%20compiler,as%20opposed%20to%20deployment).
  (e.g., `-g` for GCC/Clang, `/Zi` for MSVC).
- **Run the program under the debugger** and reproduce the issue (crash
  or misbehavior).
- **Set breakpoints** at strategic locations (at the crash line, or
  before the bug triggers) to pause execution and inspect state.
- **Use the call stack (backtrace)** to see the chain of function calls
  leading to the bug. This often reveals infinite recursions or who
  called the faulty
  function[\[16\]](https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html#:~:text=,fib%28n%3D2%29%20%2B%2061%20at%20lldbtutorial%3A18).
- **Inspect variables and memory** at the point of failure. Check
  pointer values, indices, etc. All debuggers let you examine variables
  (`frame var`/`print` in LLDB, `dv`/`?` in WinDbg, the Variables pane
  or `print` in VS Code).
- **Use watchpoints (data breakpoints) for memory issues** to catch when
  a variable changes or when freed memory is
  accessed[\[5\]](https://gist.github.com/rais38/6ecead686af9291da542#:~:text=Set%20a%20watchpoint%20on%20a,my_ptr)[\[11\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/ba--break-on-access-#:~:text=This%20example%20sets%20a%20conditional,the%20debugger%20should%20break%20in).
  We saw this was invaluable for use-after-free bugs.
- **Consider thread interactions** in multi-threaded programs. Check the
  state of each thread (using thread lists or VS Code\'s thread view),
  and use breakpoints to freeze threads at critical moments. We
  essentially performed *post-mortem* analysis on the crashing thread
  and *live analysis* by pausing threads.
- **Interpret the debugger output**: For example, an **EXC_BAD_ACCESS /
  c0000005** means invalid memory access -- likely null deref or freed
  memory access. A **c00000fd** means stack
  overflow[\[18\]](https://stackoverflow.com/questions/18721841/how-to-find-the-source-of-a-stackoverflowexception-in-my-application#:~:text=that%20raised%20the%20exception%2C%20and,will%20be%20something%20like%20this).
  These codes and messages point to categories of bugs.

The senior engineer's thought process at each step is to formulate
hypotheses ("It might be a null pointer\...", "Maybe it's recursing
infinitely\...", "Could this be a race condition?") and then use the
debugger's features to confirm or refute those hypotheses. By setting
breakpoints or watchpoints and examining program state, we gather
evidence for the root cause.

For each tool:\
- **LLDB (macOS)** -- we used commands like `run`, `bt`, `print`,
`watchpoint set`, etc., to diagnose issues. LLDB gave us rich info like
function names and line numbers where things went
wrong[\[4\]](https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html#:~:text=,fib%28n%3D2%29%20%2B%2071%20at).\
- **WinDbg (Windows)** -- we learned to use `g`, `k`, `dv`, and thread
commands. It gave explicit exception codes (0xC0000005, 0xC00000FD)
which are clues, and its `ba` command allowed catching memory
accesses[\[10\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/ba--break-on-access-#:~:text=Option%20Action).
WinDbg is powerful for low-level analysis (like viewing heap state or
using `!analyze`), but even basic use helped us pinpoint the problem.\
- **VS Code with CodeLLDB** -- provided a user-friendly way to set
breakpoints and inspect variables in a GUI. We saw how to set function
breakpoints and that we could drop into the LLDB console for advanced
commands. It's a great way to leverage LLDB with an IDE feel.\
- **VS Code with CppDbg (GDB)** -- similarly, we used breakpoints and
the Variables/Call Stack panes. We also noted VS Code's **"Break on
Value Change"** feature for data
breakpoints[\[12\]](https://devblogs.microsoft.com/cppblog/whats-new-for-c-debugging-in-visual-studio-code/#:~:text=To%20set%20a%20data%20breakpoint,select%20Break%20on%20Value%20Change).
GDB's behavior was comparable to LLDB's in these cases.

In practice, the choice of tool might depend on your environment (Visual
Studio has its own debugger, etc.), but the skills transfer. Setting
breakpoints, understanding memory addresses, interpreting call stacks --
these are universal debugging skills.

**Final advice for junior developers:** Don't be intimidated by memory
bugs. Use your debugger as a flashlight: - If you get a crash, run under
a debugger and let it tell you *where* and *why* (e.g., null pointer at
this line). - If the program doesn't crash but behaves strangely (like
the use-after-free example that printed stale data), set traps
(watchpoints, breakpoints) at points where you suspect the bug (like
right after free, check if that memory gets touched). - For
multi-threaded issues, try to reproduce the issue with a debugger
attached, and inspect each thread's state to piece together the
timeline.

Each of the case studies showed a slightly different use of the tools,
but the underlying approach is systematic: **break, inspect, deduce,
confirm**. With these techniques, you can tackle many C++ bugs involving
memory and concurrency. Happy debugging!

**Sources:**

- Low-level debugger usage and watchpoint
  commands[\[5\]](https://gist.github.com/rais38/6ecead686af9291da542#:~:text=Set%20a%20watchpoint%20on%20a,my_ptr)[\[14\]](https://gist.github.com/rais38/6ecead686af9291da542#:~:text=...%20%28lldb%29%20bt%20,int32_t%29%20global%20%3D%205)[\[11\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/ba--break-on-access-#:~:text=This%20example%20sets%20a%20conditional,the%20debugger%20should%20break%20in)
- LLDB debugging example of a crash and
  backtrace[\[3\]](https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html#:~:text=match%20at%20L238%20,fib%28n%3D2%29%20%2B%2071%20at)[\[16\]](https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html#:~:text=,fib%28n%3D2%29%20%2B%2061%20at%20lldbtutorial%3A18)
- Microsoft documentation on data breakpoints in VS
  Code[\[12\]](https://devblogs.microsoft.com/cppblog/whats-new-for-c-debugging-in-visual-studio-code/#:~:text=To%20set%20a%20data%20breakpoint,select%20Break%20on%20Value%20Change)
- WinDbg output for stack overflow and access
  violation[\[18\]](https://stackoverflow.com/questions/18721841/how-to-find-the-source-of-a-stackoverflowexception-in-my-application#:~:text=that%20raised%20the%20exception%2C%20and,will%20be%20something%20like%20this)[\[19\]](https://levelblue.com/blogs/spiderlabs-blog/windows-debugging-exploiting-part-3-windbg-time-travel-debugging#:~:text=%281f20.fc0%29%3A%20Access%20violation%20,first%2Fsecond%20chance%20not%20available)

------------------------------------------------------------------------

[\[1\]](https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html#:~:text=match%20at%20L204%20The%20compiler,as%20opposed%20to%20deployment)
[\[2\]](https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html#:~:text=,lldb)
[\[3\]](https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html#:~:text=match%20at%20L238%20,fib%28n%3D2%29%20%2B%2071%20at)
[\[4\]](https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html#:~:text=,fib%28n%3D2%29%20%2B%2071%20at)
[\[16\]](https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html#:~:text=,fib%28n%3D2%29%20%2B%2061%20at%20lldbtutorial%3A18)
[\[17\]](https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html#:~:text=,%2B%2078%20at%20lldbtutorial%3A80)
LLDB Tutorial

<https://klasses.cs.uchicago.edu/archive/2016/spring/15200-1/assigns/week5/lldb.html>

[\[5\]](https://gist.github.com/rais38/6ecead686af9291da542#:~:text=Set%20a%20watchpoint%20on%20a,my_ptr)
[\[13\]](https://gist.github.com/rais38/6ecead686af9291da542#:~:text=Set%20a%20watchpoint%20on%20a,a%20condition%20on%20a%20watchpoint)
[\[14\]](https://gist.github.com/rais38/6ecead686af9291da542#:~:text=...%20%28lldb%29%20bt%20,int32_t%29%20global%20%3D%205)
Watchpoint in LLDB · GitHub

<https://gist.github.com/rais38/6ecead686af9291da542>

[\[6\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/getting-started-with-windbg#:~:text=3,readable%20function%20and%20variable%20names)
[\[7\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/getting-started-with-windbg#:~:text=Then%2C%20enter%20this%20command%3A)
Get Started with WinDbg User-Mode Debugger - Windows drivers \|
Microsoft Learn

<https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/getting-started-with-windbg>

[\[8\]](https://levelblue.com/blogs/spiderlabs-blog/windows-debugging-exploiting-part-3-windbg-time-travel-debugging#:~:text=0%3A000,first%2Fsecond%20chance%20not%20available)
[\[19\]](https://levelblue.com/blogs/spiderlabs-blog/windows-debugging-exploiting-part-3-windbg-time-travel-debugging#:~:text=%281f20.fc0%29%3A%20Access%20violation%20,first%2Fsecond%20chance%20not%20available)
Windows Debugging & Exploiting Part 3: WinDBG Time Travel Debugging

<https://levelblue.com/blogs/spiderlabs-blog/windows-debugging-exploiting-part-3-windbg-time-travel-debugging>

[\[9\]](https://medium.com/ax1al/introduction-to-windbg-1177487c77a4#:~:text=Medium%20medium,the%20type%20of%20the)
Quick introduction to WinDbg. Reverse Engineering Tutorial - Medium

<https://medium.com/ax1al/introduction-to-windbg-1177487c77a4>

[\[10\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/ba--break-on-access-#:~:text=Option%20Action)
[\[11\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/ba--break-on-access-#:~:text=This%20example%20sets%20a%20conditional,the%20debugger%20should%20break%20in)
[\[15\]](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/ba--break-on-access-#:~:text=Size%20Specifies%20the%20size%20of,e%2C%20Size%20must%20be%201)
ba (Break on Access) - Windows drivers \| Microsoft Learn

<https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/ba--break-on-access->

[\[12\]](https://devblogs.microsoft.com/cppblog/whats-new-for-c-debugging-in-visual-studio-code/#:~:text=To%20set%20a%20data%20breakpoint,select%20Break%20on%20Value%20Change)
What's new for C++ Debugging in Visual Studio Code - C++ Team Blog

<https://devblogs.microsoft.com/cppblog/whats-new-for-c-debugging-in-visual-studio-code/>

[\[18\]](https://stackoverflow.com/questions/18721841/how-to-find-the-source-of-a-stackoverflowexception-in-my-application#:~:text=that%20raised%20the%20exception%2C%20and,will%20be%20something%20like%20this)
c# - How to find the source of a StackOverflowException in my
application - Stack Overflow

<https://stackoverflow.com/questions/18721841/how-to-find-the-source-of-a-stackoverflowexception-in-my-application>
