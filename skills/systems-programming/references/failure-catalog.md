# Failure Catalog

A catalog of named failure modes for code that calls the operating system. The content comes
from *Advanced Programming in the UNIX Environment* (Stevens and Rago) and *Systems
Programming* (Donovan).

Each entry has three parts:

- **Signature** — what the code looks like. This is the text that you search for.
- **Consequence** — what goes wrong, and when.
- **Fix** — the remedy.

**How to use it.** `code-review` runs the applicable sections against a difference.
`verify` runs them against an implemented phase. `planner` and `detail-planning` run them
against a design. Report only the failures that apply. "Not applicable. This change touches
no file, no process, and no signal." is a valid result. That result is better than an
invented finding.

**Severity.** 🔴 causes data loss, data corruption, or a privilege defect. It blocks the
merge. 🟡 causes an outage, a wrong result under load, or an unbounded cost. 🔵 is a risk to
maintenance or to operation.

**A high-level language does not remove these failures.** `../SKILL.md`, Section 6 gives the
table of what each runtime still exposes. The entries marked "every language" apply to
Python, Go, Java, Rust, and Node.js without change.

---

## A. File descriptors and I/O

### 🔴 S-01 — Unchecked `write`
**Signature:** a call to `write`, `send`, or a language write method with no test of the
return value.
**Consequence:** the call moved fewer bytes than the code requested. The file holds a record
that stops in the middle. The defect appears only when the buffer fills, so a test with small
data never finds it.
**Fix:** write a loop until the count reaches the total. Use the `writen` function in
`01-file-descriptors-and-io.md`, Section 4. Applies to **every language** for sockets and
pipes.

### 🔴 S-02 — Unchecked `close`
**Signature:** `close(fd);` with no test. A `finally` block that closes and discards the
error.
**Consequence:** the kernel found a write error after the last `write` call returned. `close`
is the only call that can report it. The program reports success. The data is lost.
**Fix:** check the result of `close` on every file that the code wrote. Report the error to
the caller. Applies to **every language**.

### 🔴 S-03 — Short `read` treated as the whole file
**Signature:** one `read` call, and the code then uses the buffer as complete content.
**Consequence:** the code processes part of a record. A parser reports invalid data for a
file that is valid. A hash does not match.
**Fix:** loop until `read` returns 0, or until the count reaches the length that the code
needs.

### 🟡 S-04 — No handling of `EINTR`
**Signature:** a slow call with no retry loop, and `sigaction` without `SA_RESTART`.
**Consequence:** the call fails with `EINTR` when a signal arrives. The program reports an
I/O error that no I/O caused. The defect appears only when the program handles a signal.
**Fix:** write a retry loop, or set `SA_RESTART`. `SA_RESTART` does not cover `select`,
`poll`, or a call with a timeout.

### 🔴 S-05 — `lseek` to the end, then `write`
**Signature:** `lseek(fd, 0, SEEK_END)` and a `write` call after it.
**Consequence:** two processes both seek to the same end. The second `write` overwrites the
record that the first `write` produced. Records disappear from a log file.
**Fix:** open the file with `O_APPEND`. Then the kernel performs the seek and the write as
one atomic step.

### 🔴 S-06 — `stat` or `access` before `open`
**Signature:** `stat(path, ...)` or `access(path, ...)`, then `open(path, ...)`.
**Consequence:** the path names a different file at the second call. An attacker replaces the
name between the two calls. A privileged program then acts on a file that the user cannot
reach. Stevens and Rago name this pattern as a race.
**Fix:** call `open` first. Then call `fstat` on the descriptor. Use `O_NOFOLLOW` and
`openat`.

### 🔴 S-07 — A durable write with no `fsync`
**Signature:** a write, then a `close`, and the code reports success to a user.
**Consequence:** the kernel holds the data in a buffer. A power loss discards it. The file
exists with zero bytes or with old content.
**Fix:** call `fsync` on the descriptor before you report success. Call `fsync` on the
directory after a rename. Applies to **every language**.

### 🟡 S-08 — `F_SETFL` without `F_GETFL`
**Signature:** `fcntl(fd, F_SETFL, O_NONBLOCK)`.
**Consequence:** the call erases every other status flag, including the access mode. A later
write to a read-write descriptor fails.
**Fix:** read the flags with `F_GETFL`, add your bit, then set the result.

### 🟡 S-09 — `select` above `FD_SETSIZE`
**Signature:** `FD_SET(fd, &set)` with no test of `fd` against `FD_SETSIZE`.
**Consequence:** a descriptor number of 1024 or higher writes outside the `fd_set` structure.
This corrupts memory near it. The failure appears only when the server holds many
connections.
**Fix:** use `poll`, `epoll`, or `kqueue`. When `select` must stay, test every descriptor
number against `FD_SETSIZE` first.

### 🟡 S-10 — `mmap` on a file that another process can shorten
**Signature:** `mmap` with `MAP_SHARED` on a file that another process writes.
**Consequence:** a `truncate` from the other process makes your access to the removed pages
raise `SIGBUS`. The default action stops the process.
**Fix:** hold a lock that prevents the truncation, or handle `SIGBUS` with `sigsetjmp`. Do not
use `mmap` on a file that a machine on the network can change.

---

## B. Files, names, and directories

### 🔴 S-11 — A write over a file in place
**Signature:** `open` with `O_TRUNC` on a file that holds data, then a write of the new
content.
**Consequence:** a crash between the truncate and the end of the write leaves a file that
holds part of the old content and part of the new content. The name is valid. No reader can
detect the damage.
**Fix:** use the six steps in `02-files-and-directories.md`, Section 7. Write a temporary
file, call `fsync`, then call `rename`. Applies to **every language**.

### 🔴 S-12 — `rename` without an `fsync` on the directory
**Signature:** `rename(tmp, target)` and no `fsync` on the directory that holds the target.
**Consequence:** a power loss can leave a directory that holds neither the old name nor the
new name. The file disappears.
**Fix:** open the directory. Call `fsync` on it. Then close it.

### 🔴 S-13 — The data reaches the disk after the rename
**Signature:** `write`, then `rename`, and the `fsync` call comes after the `rename` call or
does not exist.
**Consequence:** the rename reaches the disk first. A power loss leaves a file with the
correct name and zero bytes. The old content is gone. This is the worst order of the steps.
**Fix:** call `fsync` on the temporary file **before** the `rename` call.

### 🟡 S-14 — A temporary file in a different file system
**Signature:** `mkstemp("/tmp/...")` and a `rename` onto a target in another directory.
**Consequence:** `rename` fails with `EXDEV`. The update never happens. Code that ignores the
error reports success.
**Fix:** create the temporary file in the same directory as the target.

### 🔴 S-15 — `mktemp` or `tmpnam`
**Signature:** `mktemp`, `tmpnam`, or `tempnam`, and an `open` call after it.
**Consequence:** the function returns a name, not a file. Another process creates that name
first. A privileged program then writes to a file that an attacker chose, or follows a link
that an attacker made.
**Fix:** use `mkstemp` or `mkdtemp`. They create the object and return a descriptor as one
step.

### 🔴 S-16 — A path from a user with no check
**Signature:** a path that comes from a request, an archive, or a configuration file, and a
`open` call with no check.
**Consequence:** a component of `..` escapes the target directory. A leading `/` writes to an
absolute location. A symbolic link points to a file outside the tree.
**Fix:** apply the table in `02-files-and-directories.md`, Section 11. Combine the string
check with `O_NOFOLLOW` and `openat`. A string check alone is not enough.

### 🟡 S-17 — A recursive walk that follows a symbolic link
**Signature:** `stat` inside a directory walk, or a walk with no test for a repeat of
`st_dev` and `st_ino`.
**Consequence:** a link that points to a parent directory makes a walk that never stops. The
program uses the processor and the memory until it fails.
**Fix:** use `lstat` or `fstatat` with `AT_SYMLINK_NOFOLLOW`. Record the pair `st_dev` and
`st_ino` for each directory. Set a maximum depth.

### 🟡 S-18 — A fixed buffer for a path
**Signature:** `char path[PATH_MAX]`, `char path[256]`, or `strcat` on a path.
**Consequence:** a long path overflows the buffer, or the code cuts the path. A cut path names
a directory instead of a file. Some systems do not define `PATH_MAX`.
**Fix:** use `openat` with one component at a time. Or allocate the buffer at run time and
grow it when a call returns `ERANGE`.

### 🟡 S-19 — A modification time used to detect a change
**Signature:** a comparison of `st_mtime` against a stored value, to decide whether to process
a file again.
**Consequence:** the pipeline skips a file that changed. The resolution can be one second. A
copy keeps the old time. Any program can set the time to any value.
**Fix:** use a content hash. Or combine the size, the time, and `st_ctime`. Write down which
method you chose and what it cannot detect.

---

## C. Buffered streams

### 🟡 S-20 — Output that a pipeline never shows
**Signature:** `printf` or a language print function for progress, and no `fflush`.
**Consequence:** the stream is line buffered at a terminal and fully buffered into a pipe or a
file. The program appears to do nothing for a long time. A crash loses every message.
**Fix:** call `fflush` after each message that a user must see. Or call `setvbuf` with
`_IOLBF` at the start. Applies to **every language**.

### 🔴 S-21 — Output that appears two times after `fork`
**Signature:** a buffered write, then a `fork` call, with no `fflush` between them.
**Consequence:** the child receives a copy of the buffer. Both processes write the same bytes.
A log file holds each record two times.
**Fix:** call `fflush(NULL)` before every `fork`. Applies to **every language**.

### 🟡 S-22 — A stream and a descriptor for one file
**Signature:** `fprintf(fp, ...)` and `write(fileno(fp), ...)` on the same file. In Python, a
file object and `os.write` on its descriptor.
**Consequence:** the two paths hold separate positions and separate buffers. The bytes reach
the file in an order that the code did not intend.
**Fix:** choose one method for each file. Call `fflush` before you change methods.

### 🟡 S-23 — A stream inside an event loop
**Signature:** a `FILE` object or a buffered reader on a descriptor that `select`, `poll`, or
`epoll` also watches.
**Consequence:** the library buffer already holds a full record. The kernel holds no more
data. The wait call reports that nothing is ready. The program waits without end.
**Fix:** use descriptors alone in an event loop. Manage the buffer in your own code.

---

## D. Processes

### 🟡 S-24 — An unchecked `fork`
**Signature:** `fork()` with a test for 0 only, and no test for −1.
**Consequence:** a failed `fork` returns −1. The code treats −1 as the parent branch. The
parent then waits for a child that does not exist.
**Fix:** test all three results: −1, 0, and a positive number.

### 🟡 S-25 — No `wait` for a child
**Signature:** a `fork` call with no `waitpid` and no `SIGCHLD` handler.
**Consequence:** each child becomes a zombie. It holds a process table entry. A long-lived
parent fills the table. No process on the machine can then start.
**Fix:** call `waitpid` for every child. Or handle `SIGCHLD` with a `waitpid` loop that uses
`WNOHANG`.

### 🟡 S-26 — A `SIGCHLD` handler with one `waitpid` call
**Signature:** a `SIGCHLD` handler that calls `waitpid` one time.
**Consequence:** standard signals do not queue. Several children that exit together deliver
one signal. The other children stay as zombies.
**Fix:** call `waitpid` with `WNOHANG` in a loop until it returns 0 or −1.

### 🟡 S-27 — Only the exit code is checked
**Signature:** `WEXITSTATUS(status)` with no `WIFEXITED` test. A comparison of `returncode`
against 0 with no test for a signal.
**Consequence:** a child that a signal stopped has no exit code. The macro returns a value
that has no meaning. A worker that `SIGKILL` stopped reports success. A crash stays hidden.
**Fix:** test `WIFEXITED` first. Test `WIFSIGNALED` and report the signal number.

### 🔴 S-28 — A command built from input that a user controls
**Signature:** `system`, `popen`, or `shell=True` with a string that holds a value from a
user.
**Consequence:** the shell interprets `;`, `|`, `&`, and `$( )`. The value becomes a command.
This is a privilege defect.
**Fix:** use `fork` and `execve` with an argument array. Then no shell reads the text. Never
call `system` from a program that holds a privilege.

### 🟡 S-29 — A descriptor that leaks into a child
**Signature:** `open` with no `O_CLOEXEC`, and an `exec` call after it.
**Consequence:** the child holds a descriptor that it must not hold. A child that holds a pipe
open stops the reader from seeing an end of file. A child that holds a log file open keeps
the disk space after a delete. A child can read data that it must not read.
**Fix:** set `O_CLOEXEC` at `open` time. Close each unneeded descriptor between `fork` and
`exec`.

### 🟡 S-30 — A `sleep` call used to order two processes
**Signature:** `sleep` or a delay between a `fork` call and the work that depends on the
child.
**Consequence:** the delay is not a guarantee. A load spike defeats any value. The program
fails on a busy machine and passes every test on an idle machine.
**Fix:** use a pipe. The child reads one byte. The parent writes it when the child may
continue. Or use `waitpid`.

---

## E. Signals

### 🔴 S-31 — A handler that calls a function that is not safe
**Signature:** `printf`, `malloc`, `free`, `syslog`, `exit`, or a mutex lock inside a signal
handler.
**Consequence:** the handler interrupts the same function in the main program. The heap
becomes corrupt, or the program deadlocks. The failure is rare and appears under load only.
**Fix:** set a flag of type `volatile sig_atomic_t`. Do the work in the main loop. The safe
list is in `05-signals.md`, Section 4.

### 🟡 S-32 — A shared variable that is not `volatile sig_atomic_t`
**Signature:** `static int flag;` that a handler sets and a loop reads.
**Consequence:** the compiler keeps the variable in a register. The loop never sees the
change. The program does not stop. An optimized build fails where a debug build works.
**Fix:** declare it `volatile sig_atomic_t`.

### 🟡 S-33 — A handler that does not save `errno`
**Signature:** a handler that calls `waitpid`, `read`, `write`, or any other system call.
**Consequence:** the call changes `errno`. The main program then reads a value that belongs to
the handler. The program reports an error that did not happen.
**Fix:** save `errno` at the start of the handler. Restore it at the end.

### 🟡 S-34 — `signal` instead of `sigaction`
**Signature:** `signal(SIGTERM, handler)`.
**Consequence:** the behavior differs between systems. On some systems the disposition returns
to the default before the handler runs. A second signal in that window stops the process.
**Fix:** use `sigaction`. Call `sigemptyset` on `sa_mask`. Choose `SA_RESTART` on purpose.

### 🔴 S-35 — `SIGPIPE` with no handling
**Signature:** a write to a socket or a pipe, with no `SIG_IGN` for `SIGPIPE` and no test for
`EPIPE`.
**Consequence:** a client that disconnects stops the server. The process exits with no
message. The log shows nothing.
**Fix:** set `SIGPIPE` to `SIG_IGN`. Then handle `EPIPE` at every write site. Or use
`MSG_NOSIGNAL`.

### 🟡 S-36 — An unblock, then a `pause`
**Signature:** `sigprocmask` to unblock, then `pause`.
**Consequence:** the signal arrives between the two calls. `pause` then waits for a signal
that already came. The program does not continue.
**Fix:** use `sigsuspend`. It sets the mask and waits as one atomic step.

---

## F. Threads

### 🔴 S-37 — `pthread_cond_wait` inside an `if`
**Signature:** `if (condition) pthread_cond_wait(...)`.
**Consequence:** a spurious wakeup, a broadcast, or a third thread that took the item makes
the condition false after the return. The code then acts on data that does not exist. The
failure appears rarely and under load only.
**Fix:** use a `while` loop that tests the predicate again after every return.

### 🔴 S-38 — No lock order
**Signature:** two mutexes that two functions lock in a different order.
**Consequence:** a deadlock. Two threads each hold one mutex and wait for the other. The
program stops. The failure needs an exact timing, so it appears in production only.
**Fix:** define one lock order for the whole program. Write it down. Use
`pthread_mutex_trylock` and a release when the order cannot hold.

### 🟡 S-39 — `errno` after a pthread call
**Signature:** `if (pthread_create(...) != 0) perror(...)`.
**Consequence:** the pthread functions return the error number. They do not set `errno`. The
message reports an error from a much earlier call.
**Fix:** store the return value. Use `strerror` on that value.

### 🟡 S-40 — A thread that is neither joined nor detached
**Signature:** `pthread_create` in a loop, with no `pthread_join` and no `pthread_detach`.
**Consequence:** each finished thread holds its resources. A server that creates one thread
for each request runs out of memory.
**Fix:** call `pthread_detach`, or call `pthread_join` for every thread. Better, use a pool
with a fixed thread count.

### 🔴 S-41 — A function that returns static memory, called from a thread
**Signature:** `strtok`, `localtime`, `asctime`, `getpwnam`, `inet_ntoa`, or `rand` in code
that runs in more than one thread.
**Consequence:** two threads overwrite one buffer. Each thread reads the data of the other
thread. The result is wrong, and no error appears.
**Fix:** use the `_r` version, or `inet_ntop`. The full table is in
`06-threads-and-concurrency.md`, Section 6.

### 🔴 S-42 — `fork` in a process that has threads
**Signature:** `fork` in a program that called `pthread_create`, and any call other than an
async-signal-safe call before the `exec` call.
**Consequence:** only the calling thread continues. A mutex inside the library stays locked.
No thread in the child can unlock it. The child stops at the first `malloc` or `printf` call.
**Fix:** call `exec` at once after `fork`. Use `posix_spawn`. Or create every child process
before you create any thread.

### 🟡 S-43 — A shared variable with no lock on the read path
**Signature:** a write under a mutex, and a read of the same variable with no mutex.
**Consequence:** the compiler and the processor reorder the access. The reader sees an old
value or a partial value. A "small optimization" of this kind causes a defect that no test
finds.
**Fix:** lock for every access, including every read. Or use an atomic type with a memory
order that you chose on purpose.

---

## G. Interprocess communication and sockets

### 🟡 S-44 — A pipe end that stays open
**Signature:** a `pipe` call and a `fork` call, and one process does not close the end that
it does not use.
**Consequence:** the reader never receives an end of file, because its own copy of the write
end stays open. The program waits without end.
**Fix:** close the unused end in both processes at once after the `fork` call.

### 🔴 S-45 — A stream that has no framing
**Signature:** one `read` call on a TCP socket or a pipe, and the code treats the result as
one message.
**Consequence:** TCP gives a stream, not records. One message can arrive in several parts. Two
messages can arrive in one part. The parser fails, or it accepts a mixed record.
**Fix:** choose a framing method. Use a length prefix, a delimiter, or a datagram socket.
Write the method down.

### 🔴 S-46 — A length from a peer used as an allocation size
**Signature:** a length field that the code reads from the network, and a `malloc` call with
that value.
**Consequence:** a peer that sends a large length makes your process request all of the
memory. The process stops, or the machine stops.
**Fix:** check the length against a maximum before you allocate. Reject the connection when it
is larger.

### 🟡 S-47 — A network call with no timeout
**Signature:** `connect`, `read`, or `write` on a socket, with no `SO_RCVTIMEO` and no
`select`.
**Consequence:** a machine that lost power sends no reset. The call waits for many minutes.
Every worker becomes busy. The service stops.
**Fix:** set a timeout on every network call. Read
`data-systems-design/references/08-distributed-systems-faults.md`.

### 🟡 S-48 — `EMFILE` from `accept` with no plan
**Signature:** an `accept` loop with no case for `EMFILE` and `ENFILE`.
**Consequence:** the descriptor limit stops new connections. Code that treats the error as
fatal stops the server. Code that ignores the error runs a loop at full processor use.
**Fix:** hold one spare descriptor. Close it, accept the connection, close the connection,
then open the spare again. Raise `RLIMIT_NOFILE`.

### 🔴 S-49 — Shared memory with no lock, or with a pointer inside it
**Signature:** `shmat` or `shm_open` with no semaphore, or a pointer stored in the shared
region.
**Consequence:** two processes change one value and lose one change. A pointer that one
process stored has no meaning in the other process, because the segment can sit at a
different address.
**Fix:** protect the data with a semaphore, or with a mutex that has the
`PTHREAD_PROCESS_SHARED` attribute. Store an offset from the start of the segment, not a
pointer.

---

## H. Memory, storage classes, and the toolchain

### 🔴 S-50 — A pointer to a local variable that the function returned
**Signature:** `return &local;` or a return of a pointer to a local array. A thread start
function that returns the address of its own stack.
**Consequence:** the frame disappears at the return. The caller reads memory that another
call now uses. The value looks correct in a debug build and changes in an optimized build.
**Fix:** allocate the result with `malloc`, or write into a buffer that the caller gives you.

### 🔴 S-51 — A size computed by multiplication
**Signature:** `malloc(count * size)` where a value from input decides `count`.
**Consequence:** the product wraps around. `malloc` returns a small block. The code then
writes past its end. This is a privilege defect.
**Fix:** use `calloc`, which checks the product. Or check the product against a maximum before
the call.

### 🟡 S-52 — `realloc` assigned onto its own pointer
**Signature:** `p = realloc(p, n);`.
**Consequence:** a failed `realloc` returns a null pointer and keeps the old block. The
assignment loses the only pointer to that block. The code then leaks the memory and reads a
null pointer.
**Fix:** assign to a new pointer. Test it. Then assign to the original pointer.

### 🔵 S-53 — An allocation with no named owner
**Signature:** a `malloc` call whose block passes between several functions, and no comment
that names the owner.
**Consequence:** two functions free it, or no function frees it. A double free corrupts the
heap. A leak grows until the process fails.
**Fix:** name one owner for every allocation. Write the owner in a comment at the allocation
site. Free the block in the code of that owner only.

### 🟡 S-54 — A library before its user on the link command
**Signature:** `cc -lfoo main.o`.
**Consequence:** the linker takes members from a static library only for the symbols that are
open at that point. No symbol is open at `-lfoo`. The build reports "undefined reference" for
a library that is present.
**Fix:** put every library after the object files that use it. Read
`08-toolchain-and-machine-model.md`, Section 7.

### 🟡 S-55 — A `struct` written to a file or a socket
**Signature:** `fwrite(&s, sizeof(s), 1, fp)` or a `send` call with a pointer to a `struct`.
**Consequence:** the byte order, the padding, and the size of each type differ between
machines and between compilers. Another machine reads values that have no meaning. A new
compiler version breaks the format.
**Fix:** write each field with a defined size and a defined byte order. Read
`data-systems-design/references/04-encoding-and-evolution.md`.

---

## Index by symptom

| Symptom | Check these entries |
|---|---|
| The file holds zero bytes after a power loss | S-07, S-12, S-13 |
| The file holds old content and new content | S-11 |
| A record disappeared from a log file | S-01, S-05 |
| A record appears two times | S-21 |
| The program waits and does not continue | S-23, S-36, S-37, S-38, S-42, S-44, S-47 |
| The program prints nothing in a pipeline | S-20 |
| The program exits with no message | S-35, S-27 |
| The program fails under load only | S-04, S-09, S-30, S-31, S-37, S-38, S-43 |
| The program fails in an optimized build only | S-32, S-50 |
| "too many open files" | S-29, S-48 |
| The disk is full, but the files are small | S-14, S-29 |
| The build reports "undefined reference" | S-54 |
| The data is wrong on another machine | S-55 |
| An attacker reached a file outside the directory | S-06, S-15, S-16 |
