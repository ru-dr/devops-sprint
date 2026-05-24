## Process
A process is a running instance of a program or a task on an operating system.

It consists a chunk of memory which is allocated to process by a kernel, and it consists 4 main separate regions.

- The code (list of instructions) which process is following.
- Data (any global variables it requires during the task)
- heap (allocated at runtime for the task to run, grows up as code runs and save the processes' data)
- stack (required for function calls, local variables, grows down as code runs)

> Note: both stack and heap increase in size the "grows up" - "grows down" is just used to refer the direction of the increase, look below diagram for better visualization.

```
High addresses
┌─────────────────────┐
│       STACK         │  ← starts here, grows DOWN
│         ↓           │     (function calls push frames downward)
│                     │
│                     │
│      (unused        │
│       space)        │
│                     │
│                     │
│         ↑           │
│       HEAP          │  ← grows UP
│                     │     (malloc/new extends upward)
├─────────────────────┤
│       DATA          │  global variables
├─────────────────────┤
│       CODE          │  your program instructions
└─────────────────────┘
Low addresses
```

Additionally, it stores metadata such as PID, UID, PPID, and the [**file descriptor**](#file-descriptor) table — the list of open files (and other I/O streams) the process is using.

Those 4 separate regions are called **Virtual Memory** (*each process sees their own private address and can't directly access the RAM*) in the terms of operating system and since Heap should grow upward and stack should grow downward towards the memory, so in theory they will never crash into each other. But, even if they did due to any circumstances then the kernel will just kill that process.

### Threads

In the background each process will have one or more Thread as per the requirement. And each Thread will have its own stack. Everything else will be shared such as code, data, heap, open files, and metadata. 

A **Thread** is a lightweight part of a process which have a new stack and new execution context for its own task. While the process have other things one of which is an address space.

### Signals

When we press `CTRL + C` the kernel sends a `signal` (A **signal** is a small numbered message kernel deliver to a process).

Specifically `SIGINT` a signal interrupt, number - 2.

By default, it accepts it and terminates the execution.

But here is the other outcomes that are possible.

1. Default - accept and terminate
2. Catch - run custom code (such as any cleanup code)
3. Ignore

Also, these are some other signals which the kernel can pass to a running process. Each will have a number and kernel pass that number to a process not a word, the word is just for humans to read it and understand easily.

1. `SIGINT` (2) - `CTRL+C` polite and catchable.
2. `SIGTERM` (15) - polite and also catchable defaults to a `kill` command.
3. `SIGKILL` (9) - force kill command, uncatchable.
4. `SIGSEGV` (11) - segmentation fault, touched the memory you shouldn't have.
5. `SIGHUP` (1) - terminal hung up, often repurposed to mean **reload the config**.

### Pitfalls

- If a process is consisting a recursive function call which is not returning anything will cause a stack to overflow due to at runtime kernel allocate a specific amount of stack space (about **~8MB** in the case of Linux) at runtime and since the function call is not returning anything it will never be popped from the stack and each function call will occupy a new stack frame, and it consists of a detail of the function, local variables, etc. 

- If both Threads are updating the same value (in this case let's think of a global variable) this can cause in a data corruption. As updating a global variable might sound one operation but under the hood its 3 at the CPU level.

    - Read the value from the memory into a register.
    - Update the value in the register
    - Write the updated value from the register to the memory.

    > A **register** is a tiny piece of storage inside the CPU unit, it's not a RAM it's a part of a CPU chip. A Typical CPU might have a 16-32 general purpose registers.

    So if there are two Threads `A & B`. And if Thread `A` read the value from the memory (e.g. `5`) and at the same time Thread `B` completes the all 3 steps (read, add, write) then Thread `A` complete its step 2 and 3. then the value will be `6` even though we updated the value twice. This will result in a **race condition**.

    > A **race condition** is a phenomenon which says the correctness of the running program will depend on which thread will run first - and the timing is not always guaranteed.

    To fix the race condition we can use lock/mutex which makes those 3 steps **atomic**.
    
    > A **mutex** a.k.a. mutual exclusion is a memory flag which says "*I am currently in use*".

    mutex can be implemented by a special CPU instructions that are atomic at hardware level (e.g. `compare-and-swap`)

    So before Thread touches any shared value it has to acquire the mutex. And if another Thread wants to use it has to wait until the first Thread release it.

## File Descriptor

A **File Descriptor** is just a number which a process use to refer to an open file. In simple words it's just a connection ID used by a process to locate an open file. In the case of Linux a file descriptor is nothing but anything that is related to the I/O. Here are some of the examples of it sockets, pipes, terminal, devices.

These are the three standard file descriptors which every process begin with.

1. `STDIN` (0) - where the input come
2. `STDOUT` (1) - where the output come
3. `STDERR` (2) - where the error go

Even though both `STDOUT` and `STDERR` both end up in the same terminal window they are separated at OS level, it's a design choice.

Because when we save the output to a file we have to separate them normal logs go in one file and error log go in other one. If both of them are `STDOUT` they both will end up in the same file, and we can't separate them at the OS level, and we have to parse the text and take a guess to find which line is normal log and which is error log.

The reason that this split exists is that the tools (monitoring, logging, etc.) can route them independently without parsing.

### Pitfalls

- If we run `2> err.log` alone in `zsh`, the shell treats it as an incomplete command and enters multi-line input mode — the cursor moves to a new line, but no fresh prompt appears. It looks "stuck" but it isn't. An empty `err.log` file is created on the way in.

    To cancel and return to a fresh prompt, press `CTRL+C` (which sends `SIGINT` to the shell).

    > **Why this happens**: redirection (`>`, `2>`, `<`) is a modifier attached to a command, not a command itself. With no command to attach to, the shell waits for the rest of the input.

    The **redirection** does **not** persist to the next command — each command line is parsed independently.

    > A **Redirection** tell a shell to send a commands input or output somewhere other than the default.

    By Default:
    - A command read input from our KBD.
    - A command write output to our terminal (fd-1).
    - A command write error to our terminal (fd-2).

    Redirection is used to change these defaults.

    The Operators for redirect:
    - `>` — send stdout to a file
    - `2>` — send stderr to a file
    - `<` — read stdin from a file
    - `2>&1` — send stderr to wherever stdout is going

- Commands like this have a problem `./prog > out.log 2>&1 | grep DATABASE` - what problem exactly?

    Since we are writing the output of the `./prog ` to the out.log and combining the fd-2 to wherever the fd-1 is going then it will not work, because we can't save to a file and the same time do a pipe (`|`) at the same time because when we do the pipe the file is being written and empty in the current state so because of that the `grep` will get an empty file.
