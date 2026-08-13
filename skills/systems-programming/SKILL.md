---
name: systems-programming
description: Rules for code that calls the operating system directly. This skill covers file descriptors, file management, durable writes, processes, signals, threads, and interprocess communication. It also covers the toolchain that assembles, links, loads, and runs a program. Use this skill when you open, read, write, copy, move, delete, lock, or scan files. Use it when you start a process, handle a signal, or share memory. Use it when you debug a fault that appears only under load or only sometimes. This skill is not about software architecture. For architecture, use the system-design skill. The content comes from Advanced Programming in the UNIX Environment (Stevens and Rago) and Systems Programming (Donovan).
---

# SYSTEMS PROGRAMMING

**ROLE:** You are a systems engineer. You write code that calls the operating system.

**CORE FUNCTION:** You receive a task that touches files, processes, or memory. You produce
code that stays correct when the system is busy, when a call stops early, and when the
machine loses power.

Code at this level fails differently from application code. It fails only under load. It
fails only on one machine. It fails only after a power loss. The defect is in the code from
the first day. The test suite does not find it.

**This skill exists to make those defects visible before the code runs.**

## How this skill relates to the other skills

Four skills use the word "system". They answer different questions. Read this table before
you choose one.

| Skill | Question it answers | Level |
|---|---|---|
| `system-design` | What must we build? | Architecture |
| `data-systems-design` | Does the design stay correct across machines? | Distributed data |
| `systems-programming` (this skill) | Does the code stay correct against the kernel? | One machine |
| `security-engineering` | Can an attacker break it? | Threat |

Use this skill when the code calls the kernel. Use `system-design` when the code does not
exist yet.

---

# 1. WHEN TO USE THIS SKILL

Use this skill for these tasks:

- You open, read, write, copy, move, delete, or truncate a file.
- You scan a directory tree. You ingest documents. You build an index.
- You must not lose data after a crash or a power loss.
- You start a process. You run a command. You wait for a child process.
- You write a signal handler. You handle Ctrl-C. You reload a configuration file.
- You write a daemon or a long-lived worker.
- You share data between threads or between processes.
- You use a pipe, a FIFO, shared memory, or a socket.
- You debug an error that appears only sometimes.
- You see one of these error names: `EINTR`, `EAGAIN`, `EPIPE`, `ENOSPC`, `EMFILE`, `ETXTBSY`.
- You read or write a memory-mapped file.
- You debug a build error at link time or at load time.

Do not use this skill for these tasks:

- A change to business logic that touches no file and no process.
- A change to a web handler that only calls a library.
- A design question about which service owns which data. Use `system-design`.

**A high-level language does not remove these rules.** Python, Go, Java, Rust, and Node.js
call the same system calls. The runtime hides some faults. The runtime does not hide all of
them. Section 6 lists the faults that each runtime still exposes.

---

# 2. THE FIVE RULES OF THE SYSTEM CALL BOUNDARY

Learn these five rules. They cause most defects in low-level code. Check every rule on every
review.

### Rule 1. Every system call can fail. Check every return value.

A system call returns an error for reasons that your code did not cause. The disk becomes
full. The process reaches its file descriptor limit. Another user removes the directory.

Check the return value of every call. This includes `close`. A failed `close` can report a
write error that the kernel found after your last `write` call returned.

```c
if (close(fd) < 0)
    /* the data may be lost. Report this error. */
```

**Never ignore the result of `close` on a file that you wrote.**

### Rule 2. `read` and `write` can transfer fewer bytes than you request.

A short count is not an error. It is normal behavior for pipes, sockets, terminals, and
large transfers. A signal can also stop a transfer after some bytes move.

Call `read` and `write` in a loop until the count reaches the total that you need.

```c
ssize_t writen(int fd, const void *buf, size_t n);   /* loop until n bytes move */
ssize_t readn(int fd, void *buf, size_t n);          /* loop until n bytes or end of file */
```

Stevens and Rago give these two functions in Section 14.7 for this reason.

**A single `write` call that you do not check is the most frequent defect in file code.**

### Rule 3. A system call can stop early with `EINTR`.

A signal that arrives during a slow system call makes that call return −1. `errno` becomes
`EINTR`. The call did no work, or it did partial work.

Handle `EINTR` in one of two ways:

1. Repeat the call in a loop.
2. Set `SA_RESTART` in `sigaction`. Then the kernel repeats some calls for you.

`SA_RESTART` does not cover every call. It does not cover `select`, `poll`, or a socket call
that has a timeout. Write the retry loop for those calls.

### Rule 4. An operation that needs two system calls is not atomic.

The kernel can stop your process between any two calls. Another process runs in that gap.

Stevens and Rago state the rule directly:

> "Any operation that requires more than one function call cannot be atomic, as there is
> always the possibility that the kernel might temporarily suspend the process between the
> two function calls."

This rule creates a whole class of defects:

| Two calls | The defect | The atomic form |
|---|---|---|
| `lseek` to end, then `write` | Two processes overwrite each other's records | Open the file with `O_APPEND` |
| `lseek`, then `read` | The offset changes between the calls | `pread` |
| `stat`, then `open` | The path points to a different file at `open` time | `open`, then `fstat` |
| `access`, then `open` | An attacker replaces the path. This is a privilege defect. | `open`, then check with `fstat` |
| `open` to test, then `creat` | Another process creates the file and writes to it. That data is lost. | `open` with `O_CREAT` and `O_EXCL` |
| `close`, then `fcntl` | A signal handler changes the file descriptor in the gap | `dup2` |

**Find every pair of calls that forms one logical operation. Replace the pair with the
atomic call.**

### Rule 5. The kernel holds your data in memory until you call `fsync`.

A `write` call that returns success did not put the data on the disk. The kernel copied the
data into a buffer. The kernel writes that buffer later. Stevens and Rago call this a
**delayed write**.

A power loss between the `write` call and the disk write loses the data.

| Function | What it does | When it returns |
|---|---|---|
| `sync` | Puts every changed buffer in the write queue | At once. It does not wait for the disk. |
| `fsync(fd)` | Writes the data and the attributes of one file | After the disk write finishes |
| `fdatasync(fd)` | Writes the data of one file. It skips some attributes. | After the disk write finishes |

Call `fsync` when the data must survive a crash. Section 3 gives the full recipe.

**`fsync` on the file is not enough.** A new file name lives in its directory. You must also
call `fsync` on the directory that holds the file.

---

# 3. FILE MANAGEMENT RECIPES

These recipes solve the file tasks that appear most often. Each one is correct against
concurrent processes and against a power loss. Use them as written.

These recipes apply to document ingestion, index building, cache files, and any pipeline
that reads many files and writes derived files.

## Recipe 1 — Replace a file, and lose nothing after a crash

**Never write over a file in place.** A crash in the middle leaves a file that holds half of
the old content and half of the new content. Readers cannot detect that state.

Use these six steps:

1. Create a temporary file **in the same directory** as the target file. Use `mkstemp`.
2. Write the full new content to the temporary file. Use the loop from Rule 2.
3. Call `fsync` on the temporary file.
4. Call `close` on the temporary file. Check the result.
5. Call `rename` to move the temporary file onto the target name.
6. Call `fsync` on the directory that holds the target file.

`rename` replaces the target in one atomic step. A reader sees the old file or the new file.
A reader never sees a mixed state.

Two rules control step 1:

- The temporary file must be on the **same file system** as the target. `rename` fails
  across file systems. A different directory is often a different file system.
- Set the permissions of the temporary file before step 5. `mkstemp` creates the file with
  mode 0600.

Step 6 is the step that engineers omit most often. Without step 6, a power loss can leave a
directory that does not hold the new name.

## Recipe 2 — Create a file that must not exist yet

Call `open` with `O_CREAT` and `O_EXCL` together. The kernel tests for the file and creates
the file in one atomic step.

```c
fd = open(path, O_WRONLY | O_CREAT | O_EXCL, 0644);
if (fd < 0 && errno == EEXIST)
    /* another process owns this name */
```

This is the correct way to make a lock file. Do not test with `stat` first. Rule 4 explains
why.

## Recipe 3 — Append records from many processes

Open the file with `O_APPEND`. The kernel moves the offset to the end of the file before
every `write` call, as one atomic step.

Two limits apply:

- Write each record with **one** `write` call. Two calls can interleave with another process.
- A record that is larger than the pipe buffer or the block size can still split. Keep log
  records small.

Do not call `lseek` before `write`. That pair loses records. Rule 4 gives the reason.

## Recipe 4 — Read a whole file

1. Open the file.
2. Call `fstat` on the **file descriptor**, not `stat` on the path. The path can change.
3. Read in a loop until `read` returns 0.

Do not trust the size from `fstat` as the exact byte count. The file can grow between the
`fstat` call and the `read` call. Use the size to reserve memory. Use the loop to find the
end.

Set a maximum size before you read. A file on disk can be larger than your memory. A named
pipe has no end until the writer closes it.

## Recipe 5 — Walk a directory tree

Use `opendir` and `readdir`. Use `closedir` at the end.

Four rules apply:

- **Skip `.` and `..`.** Without this test, the walk does not stop.
- **Do not follow a symbolic link during a recursive walk.** A link that points to a parent
  directory makes an endless loop. Use `lstat`, not `stat`. `lstat` reports the link itself.
- **Do not join path strings and then call `open`.** Use `openat` and `fstatat` with the
  file descriptor of the parent directory. This method removes the race in Rule 4. It also
  removes any limit on path length.
- **Do not trust `PATH_MAX`.** Some file systems permit a longer path. Some systems do not
  define the constant.

`readdir` returns the entries in no defined order. Sort the entries when the order matters
to your output.

## Recipe 6 — Read a large file for an index

Choose the method by the size of the file and the access pattern.

| Method | Use it when | Cost |
|---|---|---|
| `read` into a buffer of 64 KB or more | You read the file from start to end, one time | Copies the data one time |
| `mmap` | You read the file many times, or you jump between offsets | The kernel maps pages. It skips the copy. |
| `pread` | Many threads read one file at different offsets | No shared offset. No lock. |

Rules for `mmap`:

- A `mmap` region ends at the size of the file at map time. Another process that truncates
  the file makes your program receive `SIGBUS`.
- `mmap` does not help for a file that you read one time from start to end. The page faults
  cost more than the copy.
- A 32-bit process cannot map a file that is larger than its address space.

For the buffer size in the `read` method: measure. Stevens and Rago show that the time
falls until the buffer reaches the block size of the file system. A larger buffer gives no
further gain.

## Recipe 7 — Lock a file between processes

Use `fcntl` with `F_SETLK` or `F_SETLKW`. This is POSIX record locking.

**POSIX record locks have two behaviors that surprise engineers. Both cause defects.**

1. The locks belong to the **process**, not to the file descriptor. Two threads in one
   process do not block each other.
2. A `close` on **any** file descriptor for that file releases **every** lock that the
   process holds on the file. A library that opens and closes the same file releases your
   lock.

Use `flock` when you need a lock that belongs to the open file description, and when your
platform supports it. Use a lock file from Recipe 2 when you need a lock on a whole file and
no byte ranges.

**A lock on a network file system is not reliable.** Do not depend on it.

---

# 4. THE DESIGN METHOD FOR A SYSTEM PROGRAM

Donovan gives a general design procedure. It applies to an assembler, a loader, a macro
processor, and a compiler. It applies equally to a parser, an indexer, a log processor, or
any program that changes one form of data into another form.

Follow these five steps in order:

| Step | Action | Output |
|---|---|---|
| 1 | Write the statement of the problem | What goes in. What comes out. What the program must not do. |
| 2 | Choose the data structures | The tables that the program needs, and what each table holds |
| 3 | Fix the format of the data bases | The exact layout of every table and every file |
| 4 | Write the algorithm | The passes, and the work in each pass |
| 5 | Look for modularity | The parts that you can separate and test alone |

**Step 3 is the step that engineers skip.** Donovan separates the data structure from its
format for a reason. The structure says what the program holds. The format says how the
bytes sit in memory or on disk. A format that you did not write down becomes a defect at
the boundary between two passes.

**The number of passes comes from the data, not from taste.** An assembler needs two passes
because a symbol can appear before its definition. Pass 1 builds the symbol table. Pass 2
generates the code. When your program must resolve a forward reference, it needs a second
pass or a patch list.

Use the same question for any tool that you write. Ask this question: *does any output
depend on input that the program has not read yet?* When the answer is yes, plan two passes.

---

# 5. THE MEMORY AND TOOLCHAIN MODEL

An engineer who does not know these steps cannot read a link error or a load error.

Donovan separates three time periods. Keep them apart when you debug.

| Period | Name | What happens |
|---|---|---|
| Translation | Assembly time or compile time | The source becomes an object file |
| Loading | Load time | The loader puts the program in memory and fixes the addresses |
| Running | Execution time | The processor runs the instructions |

Donovan states that a relocating loader performs four functions:

1. **Allocation.** It reserves memory space for the program.
2. **Linking.** It resolves the symbolic references between object files.
3. **Relocation.** It adjusts every address-dependent location to match the allocated space.
4. **Loading.** It places the instructions and the data in memory.

Every loader type does these four functions. The types differ only in **when** each function
happens. A direct-linking loader does all four at load time. A dynamic linker delays linking
and relocation until the program calls the symbol.

This model explains the errors that you see:

| Error | Which function failed |
|---|---|
| "undefined symbol" at build time | Linking. No object file defines the name. |
| "undefined symbol" at run time | Linking, delayed to run time by the dynamic linker |
| "cannot open shared object file" | Allocation. The loader cannot find the library file. |
| "text file busy" (`ETXTBSY`) | Loading. You wrote to a file that a running program uses. |
| Wrong version of a function runs | Linking. Two libraries define the same name. |

For the memory layout of a running process, read `references/08-toolchain-and-machine-model.md`.

---

# 6. WHAT YOUR LANGUAGE STILL EXPOSES

A runtime hides some of the five rules. It does not hide all of them. Check this table
before you decide that a rule does not apply.

| Rule | C | Go | Rust | Python | Java | Node.js |
|---|---|---|---|---|---|---|
| Check every return value | You | Runtime raises an error value | Runtime returns `Result` | Runtime raises | Runtime raises | Runtime raises |
| Short `read` and `write` | You | You, unless you call `io.ReadFull` | You, unless you call `read_exact` | Hidden for files. **You** for sockets and pipes. | Hidden by streams. **You** for `SocketChannel`. | **You**. A stream can send partial data. |
| `EINTR` | You | Hidden since Go 1.14 | Mostly hidden | Hidden since Python 3.5 | Hidden | Hidden |
| Two calls are not atomic | **You** | **You** | **You** | **You** | **You** | **You** |
| `fsync` | **You** | **You** | **You** | **You** | **You** | **You** |

**Rule 4 and Rule 5 are never hidden.** No runtime makes two calls atomic. No runtime calls
`fsync` for you when you close a file. A durable write in any language needs Recipe 1.

Two more items that no runtime hides:

- A signal handler in any language runs under the reentrancy rule. Read
  `references/05-signals.md`.
- A `fork` in a process that has threads copies only the calling thread. Read
  `references/06-threads-and-concurrency.md`.

---

# 7. THE REVIEW SCAN

Run this scan on any change that touches a file, a process, or a signal. Report only the
items that apply.

**Files and descriptors**

- [ ] Does the code check the result of every call, including `close`?
- [ ] Does the code handle a short `read` or a short `write`?
- [ ] Does any pair of calls form one logical operation? Rule 4 lists the pairs.
- [ ] Does an update over an existing file use Recipe 1?
- [ ] Does the durable path call `fsync` on the file **and** on the directory?
- [ ] Does the code close every file descriptor on every path, including the error paths?
- [ ] Does the code set `FD_CLOEXEC` on descriptors that a child process must not receive?
- [ ] Does the code call `open` on a path that it tested with `stat` or `access` earlier?

**Processes**

- [ ] Does the parent call `wait` or `waitpid` for every child?
- [ ] Does the code depend on whether the parent or the child runs first?
- [ ] Does the code test the return value of `fork` for all three cases: −1, 0, and positive?
- [ ] Does the code check the exit status, and not only the exit code?
- [ ] Does the code build a command string from input that a user controls?

**Signals**

- [ ] Does the handler call a function that is not async-signal-safe?
- [ ] Does the handler save and restore `errno`?
- [ ] Is every variable that the handler shares declared `volatile sig_atomic_t`?
- [ ] Does the code use `sigaction` instead of `signal`?
- [ ] Does the code handle `EPIPE` or block `SIGPIPE`?

**Threads**

- [ ] Does every wait on a condition variable sit inside a loop that tests the predicate?
- [ ] Do all threads lock the mutexes in the same order?
- [ ] Does the code call a function that returns a pointer to static memory?
- [ ] Does the code call `fork` while other threads run?

**Memory and resources**

- [ ] Does the code free every allocation on the error paths?
- [ ] Does the code size a buffer from a length that came from input?
- [ ] Does the code use a fixed buffer for a path?

The full list, with the consequence and the fix for each item, is in
`references/failure-catalog.md`.

---

# 8. HOW MUCH OUTPUT EACH TASK NEEDS

| Type of change | Output |
|---|---|
| A read of one small file, through a library | Rule 1 only |
| A write to a file that a person can lose | Recipe 1. Say that you applied it. |
| A directory walk or a document ingest | Recipe 5 and Recipe 6. Run the file section of the scan. |
| A new process, a daemon, or a signal handler | The full scan for that section |
| Shared memory, a lock, or a thread | The full scan, plus a written statement of the ordering rules |
| A change to a build, a link step, or a load path | Section 5 |

---

# 9. GLOSSARY

This skill uses one word for one meaning. Use these words in your output.

| Word | Meaning in this skill |
|---|---|
| **system call** | A function that moves control into the kernel |
| **file descriptor** | The small integer that a process uses to name an open file |
| **open file description** | The kernel record that holds the offset and the status flags |
| **atomic** | The kernel performs every step, or the kernel performs no step |
| **durable** | The data stays on the disk after a power loss |
| **reentrant** | A second call is safe while the first call has not finished |
| **async-signal-safe** | A signal handler can call this function |
| **short count** | `read` or `write` moved fewer bytes than the caller requested |
| **choose** | To select one option. The word `select` always means the system call. |
| **record** | To write information into a document |

---

# 10. REFERENCE INDEX

| Reference | Content | Read it when |
|---|---|---|
| `references/01-file-descriptors-and-io.md` | File descriptors, `open`, `read`, `write`, `lseek`, sharing, atomic calls, `dup`, `fcntl`, nonblocking I/O, `select` and `poll`, `readv`, `mmap` | Any file or socket I/O |
| `references/02-files-and-directories.md` | `stat`, file types, permissions, `umask`, hard links, symbolic links, `rename`, directory reading, `openat`, durable update, path safety | Any file management task |
| `references/03-standard-io-and-buffering.md` | Streams, the three buffer modes, `fflush`, mixed stream and descriptor I/O, temporary files, when not to use a stream | You use `FILE`, `printf`, or a language stream |
| `references/04-processes-and-execution.md` | Memory layout, environment, `fork`, `exec`, `wait`, exit status, races, process groups, sessions, daemon rules | You start or manage a process |
| `references/05-signals.md` | Signal concepts, `sigaction`, reentrancy, the async-signal-safe list, `EINTR`, signal masks, the self-pipe method, job control signals | You write or review a signal handler |
| `references/06-threads-and-concurrency.md` | Thread creation and exit, mutexes, condition variables, reader-writer locks, deadlock order, thread-local data, threads with `fork`, threads with signals | Any thread work |
| `references/07-ipc-and-sockets.md` | Pipes, FIFOs, `SIGPIPE`, shared memory, semaphores, sockets, Unix domain sockets, passing a file descriptor, message framing | Two processes exchange data |
| `references/08-toolchain-and-machine-model.md` | The machine model, assembly and load and run time, the four loader functions, static and dynamic linking, symbol resolution, the process address space | You debug a build, link, or load error |
| `references/failure-catalog.md` | 55 named failure modes with a signature, a consequence, and a fix. It also has an index by symptom. | Any review. Any defect that appears only sometimes. |

**Related skills.** Use `security-engineering` when the code holds a privilege, or when it
handles a path from a user. Many defects in this skill are also privilege defects. Use
`data-systems-design` when the data moves between machines.

---

**Sources.** The rules and the numbers come from these books:

- *Advanced Programming in the UNIX Environment*, 3rd Edition, W. Richard Stevens and
  Stephen A. Rago, Addison-Wesley, 2013
- *Systems Programming*, John J. Donovan, McGraw-Hill, 1972
