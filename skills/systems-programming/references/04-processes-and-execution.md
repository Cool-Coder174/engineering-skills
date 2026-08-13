# Processes and Execution

Source: *Advanced Programming in the UNIX Environment*, 3rd Edition, Chapters 7, 8, 9, and 13.

---

## 1. The memory layout of a process

| Part | Content | It grows |
|---|---|---|
| Text | The machine instructions | No. It is read-only and shared. |
| Initialized data | A variable with a value at compile time | No |
| Uninitialized data (bss) | A variable that starts at zero | No |
| Heap | Memory from `malloc` | Toward a higher address |
| Stack | Frames, local variables, and return addresses | Toward a lower address |

The heap and the stack grow toward each other. The text part is read-only. A write to it
raises `SIGSEGV`.

**A local array is on the stack.** A large local array causes a stack overflow. Use `malloc`
for a buffer that is larger than a few kilobytes. A thread stack is smaller than the main
stack. The default is often 8 MB for the main stack and 512 KB or less for a thread.

### Memory allocation

| Function | Behavior |
|---|---|
| `malloc(n)` | The content is not defined |
| `calloc(n, size)` | The content is zero. It checks the product for overflow. |
| `realloc(p, n)` | It can move the block. Every old pointer becomes invalid. |
| `free(p)` | The pointer becomes invalid. A second `free` corrupts the heap. |

**Use `calloc` when you compute a size from two numbers.** `malloc(count * size)` can wrap
around. The result is a buffer that is too small. `calloc` detects the overflow.

**Never use the result of `realloc` without a test.** A failed `realloc` returns a null
pointer and keeps the old block. This assignment loses the old block:

```c
p = realloc(p, n);   /* wrong. A failure loses the pointer to the old block. */
```

---

## 2. Environment variables

`getenv`, `setenv`, `putenv`, and `unsetenv` act on the environment of the process.

Four rules:

1. **The environment comes from the caller.** A program cannot trust any value in it.
2. **`getenv` is not thread-safe against `setenv`.** One thread that sets a variable can
   invalidate a pointer that another thread holds. Read the environment one time at the
   start.
3. **`putenv` stores your pointer.** It does not copy the string. A `putenv` call with a
   local array creates a pointer to memory that no longer exists. Use `setenv`, which copies.
4. **A privileged program must clear the environment.** `PATH`, `LD_PRELOAD`,
   `LD_LIBRARY_PATH`, and `IFS` change what the program runs.

---

## 3. `fork`

```c
pid_t fork(void);
```

`fork` returns two times. Test all three results:

| Return value | Meaning |
|---|---|
| −1 | The call failed. Read `errno`. Do not assume that it succeeds. |
| 0 | This is the child process |
| A positive number | This is the parent. The value is the process ID of the child. |

The child receives a copy of the memory of the parent. The two processes then run
independently. A modern kernel uses copy-on-write. It copies a page only when one process
writes to it.

### What the child inherits and what it does not

| The child inherits | The child does not inherit |
|---|---|
| Every open file descriptor | The process ID |
| The file offsets, because the descriptors are shared | Pending alarms |
| The working directory and the root directory | Pending signals. The set is empty. |
| `umask`, the environment, and the signal mask | File locks |
| The memory, including the I/O buffers | The times, which start at zero |
| Every attached shared memory segment | Other threads. Only the calling thread continues. |

**Three defects come from this table:**

1. **The buffers copy.** Call `fflush(NULL)` before `fork`. Read
   `03-standard-io-and-buffering.md`, Section 2.
2. **Every descriptor copies.** The child holds descriptors that it must not hold. A child
   that holds a pipe open stops the reader from seeing an end of file. Set `O_CLOEXEC`, or
   close each descriptor in the child.
3. **Only the calling thread continues.** A mutex that another thread held stays locked
   forever in the child. Section 9 of `06-threads-and-concurrency.md` gives the full rule.

---

## 4. Race conditions after `fork`

Stevens and Rago define the fault:

> "a race condition occurs when multiple processes are trying to do something with shared
> data and the final outcome depends on the order in which the processes run."

They add that `fork` "is a lively breeding ground for race conditions".

**The kernel gives no guarantee about which process runs first.** Any code that depends on
the order is a defect. The defect appears under load, or on a machine with a different
processor count.

Never do these things:

- Call `sleep` to make the other process run first. A load spike defeats any delay.
- Assume that the child sees a write that the parent made after the `fork` call.
- Assume that the parent sees the child exit before the parent continues.

Use one of these methods instead:

| Need | Method |
|---|---|
| The parent waits for the child to finish | `waitpid` |
| The child waits for the parent to signal it | A pipe. The child reads. The parent writes one byte. |
| Both processes wait for each other | Two pipes, or a semaphore |
| The parent needs the exit status | `waitpid` with the status macros |

**The pipe method is the reliable one.** A `read` on a pipe returns when the other process
writes or closes. It needs no delay and no signal.

---

## 5. `exec`

The `exec` family replaces the program in the current process. The process ID does not
change. `exec` returns only on failure.

| Function | Argument style | Path search | Environment |
|---|---|---|---|
| `execl` | A list | No | Inherited |
| `execv` | An array | No | Inherited |
| `execle` | A list | No | You give it |
| `execve` | An array | No | You give it |
| `execlp` | A list | Yes, with `PATH` | Inherited |
| `execvp` | An array | Yes, with `PATH` | Inherited |

Remember the letters: `l` means a list, `v` means an array, `p` means a `PATH` search, and
`e` means an environment that you give.

**A privileged program must not use `execlp` or `execvp`.** They search `PATH`. A user who
controls `PATH` chooses the program that runs. Use `execve` with a full path.

### What survives `exec`

| It survives | It does not survive |
|---|---|
| The process ID and the parent process ID | The memory. The new program starts fresh. |
| Descriptors without `FD_CLOEXEC` | Descriptors with `FD_CLOEXEC` |
| The working directory and `umask` | A signal handler. It returns to the default. |
| The signal mask | Shared memory attachments |
| File locks | A signal that the process ignored stays ignored |

**A signal handler does not survive `exec`.** The disposition returns to the default. A
signal that the program ignored stays ignored. Reset every disposition to the default before
`exec`.

---

## 6. `wait`, `waitpid`, and the exit status

```c
pid_t wait(int *statloc);
pid_t waitpid(pid_t pid, int *statloc, int options);
```

**A parent must wait for every child.** A child that exits without a `wait` becomes a
**zombie**. It holds a process table entry until the parent waits or exits.

| Case | Result |
|---|---|
| The child exits. The parent waits. | Correct |
| The child exits. The parent does not wait. | A zombie. The table fills. |
| The parent exits first | `init` adopts the child and waits for it |

### Read the status with the macros

Do not compare the status against a number. The status holds the exit code and the signal
number in different bits.

| Macro | Question it answers |
|---|---|
| `WIFEXITED(status)` | Did the program call `exit`? |
| `WEXITSTATUS(status)` | Which value did it give to `exit`? Use it only after `WIFEXITED`. |
| `WIFSIGNALED(status)` | Did a signal stop the program? |
| `WTERMSIG(status)` | Which signal? |
| `WIFSTOPPED(status)` | Did the program stop? |

**A program that a signal stopped has no exit code.** Code that reads only `WEXITSTATUS`
reports success for a program that `SIGKILL` stopped. This defect hides a crash in a worker.

Only the low 8 bits of the value from `exit` reach the parent. `exit(256)` looks like
`exit(0)`.

### `waitpid` options

| Option | Effect |
|---|---|
| `WNOHANG` | Return 0 at once when no child has exited |
| `WUNTRACED` | Report a child that stopped |
| `WCONTINUED` | Report a child that continued |

Use `WNOHANG` in a loop inside a `SIGCHLD` handler. Signals do not queue. One `SIGCHLD` can
represent several children.

```c
while (waitpid(-1, &status, WNOHANG) > 0)
    ;
```

---

## 7. `system` and the shell

`system` runs a command through `/bin/sh`. It calls `fork`, `exec`, and `waitpid`.

**Never call `system` with a string that holds input from a user.** The shell interprets
`;`, `|`, `&`, `` ` ``, `$( )`, and `>`. A value that a user controls becomes a command.

```c
sprintf(cmd, "convert %s out.png", filename);   /* wrong */
system(cmd);
```

Use `fork` and `execve` with an argument array instead. Then no shell reads the text.

**Never call `system` from a set-user-ID program.** Stevens and Rago state this rule
directly. The shell reads `PATH` and `IFS` from the environment.

`system` has three return cases. Check all three:

| Return value | Meaning |
|---|---|
| −1 | `fork` or `waitpid` failed |
| 127 in the exit status | The shell could not run the command |
| Any other value | The exit status of the shell. Read it with the macros. |

`popen` has the same defects. It also uses a shell.

---

## 8. Process groups, sessions, and job control

| Object | It holds | Purpose |
|---|---|---|
| Process group | A set of processes | The kernel sends a signal to the whole group |
| Session | A set of process groups | It connects to one controlling terminal |
| Controlling terminal | One terminal for the session | It sends `SIGINT`, `SIGQUIT`, and `SIGHUP` |

`setsid` makes a new session. The caller becomes the leader of the session and of a new
process group. The caller loses its controlling terminal.

**The caller of `setsid` must not be a process group leader.** This is why a daemon calls
`fork` first and lets the parent exit.

A signal from the terminal reaches every process in the foreground process group. `Ctrl-C`
sends `SIGINT` to the group, not to one process.

---

## 9. Daemon coding rules

Stevens and Rago give these rules in Section 13.3. Follow them in order.

1. **Call `umask(0)`.** The inherited mask can remove permissions that the daemon needs.
2. **Call `fork`. The parent exits.** The shell then treats the command as finished. The
   child is not a process group leader, which `setsid` requires.
3. **Call `setsid`.** The process becomes a session leader and a process group leader. It
   loses its controlling terminal.
4. **Change the working directory to `/`,** or to the data directory of the daemon. A working
   directory on a mounted file system stops that file system from unmounting.
5. **Close every descriptor that the daemon does not need.** An inherited descriptor holds
   resources and can leak data.
6. **Open descriptors 0, 1, and 2 onto `/dev/null`.** A library that writes to standard
   output then writes nowhere instead of writing to a file that the daemon opened later.
7. **Write log messages with `syslog`.** A daemon has no terminal.

Some systems recommend a second `fork` after step 3. The daemon is then not a session leader.
It cannot acquire a controlling terminal under the System V rules.

### Single instance

Use a lock file with `O_CREAT` and `O_EXCL`, or a record lock on a file that the daemon
holds open for its whole life. A lock that the kernel holds is better than a file that holds
a process ID. The kernel releases the lock when the process exits, including after a crash.

A file that only holds a process ID is not enough. A crash leaves the file. The number can
belong to a different process later.

---

## 10. Resource limits

`getrlimit` and `setrlimit` read and change the limits of the process.

| Limit | It controls |
|---|---|
| `RLIMIT_NOFILE` | The count of open descriptors. `EMFILE` comes from it. |
| `RLIMIT_NPROC` | The count of processes for the user |
| `RLIMIT_FSIZE` | The largest file. A write past it raises `SIGXFSZ`. |
| `RLIMIT_AS` | The size of the address space |
| `RLIMIT_CORE` | The size of a core file. Zero stops core files. |
| `RLIMIT_STACK` | The size of the stack |

Each limit has a soft value and a hard value. A process can raise the soft value up to the
hard value. Only a privileged process can raise the hard value. A lowered hard value cannot
rise again.

**Set `RLIMIT_NOFILE` in a program that opens many files.** The default is often 1024. A
server that holds many connections reaches it.

---

## Cross-references

- Descriptors across `fork` and `exec`: `01-file-descriptors-and-io.md`
- Buffers across `fork`: `03-standard-io-and-buffering.md`
- `SIGCHLD` and handler rules: `05-signals.md`
- `fork` in a process with threads: `06-threads-and-concurrency.md`, Section 9
- Command injection and `PATH`: `security-engineering/SKILL.md`
- Named failures: `failure-catalog.md`, section D
