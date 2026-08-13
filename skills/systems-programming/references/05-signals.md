# Signals

Source: *Advanced Programming in the UNIX Environment*, 3rd Edition, Chapter 10.

A signal interrupts your program at a point that you cannot predict. Every rule in this
reference comes from that one fact.

---

## 1. What a signal is

A signal is a message to a process. The kernel sends it, another process sends it, or the
process sends it to itself. A signal has a number and a name. Use the name.

A process gives each signal one of three dispositions:

| Disposition | Behavior |
|---|---|
| Default | The action that the system defines. It usually stops the process. |
| Ignore | The kernel discards the signal |
| Catch | The kernel calls your handler function |

**Two signals accept none of these choices.** `SIGKILL` and `SIGSTOP` always act. A process
cannot catch them, block them, or ignore them. This rule gives the administrator a way to
stop any process.

---

## 2. The signals that matter most

| Signal | Cause | Default action |
|---|---|---|
| `SIGINT` | The user pressed the interrupt key | Stop the process |
| `SIGTERM` | A request to stop. `kill` sends it by default. | Stop the process |
| `SIGKILL` | A forced stop | Stop the process. No handler runs. |
| `SIGHUP` | The terminal closed. Many daemons use it to reload a configuration file. | Stop the process |
| `SIGCHLD` | A child process stopped or exited | Ignore |
| `SIGPIPE` | A write to a pipe or a socket that has no reader | Stop the process |
| `SIGSEGV` | An invalid memory access | Stop the process. Write a core file. |
| `SIGBUS` | A memory access that the hardware refused. A truncated `mmap` region causes it. | Stop the process. Write a core file. |
| `SIGALRM` | The timer from `alarm` expired | Stop the process |
| `SIGUSR1`, `SIGUSR2` | Your program defines the meaning | Stop the process |
| `SIGXFSZ` | A write past `RLIMIT_FSIZE` | Stop the process |

**`SIGPIPE` stops a program that writes to a closed connection.** This behavior surprises
engineers who write network code. Section 8 gives the two responses.

---

## 3. Use `sigaction`. Do not use `signal`.

The `signal` function has behavior that changes between systems. Stevens and Rago call the
older behavior **unreliable signals**.

| Question | Old System V `signal` | BSD and `sigaction` |
|---|---|---|
| Does the disposition return to the default after one signal? | Yes | No |
| Does the kernel block the signal during the handler? | No | Yes |
| Does the kernel restart an interrupted system call? | No | You choose with `SA_RESTART` |

The old behavior creates a window. The disposition returns to the default before your handler
runs. A second signal in that window stops the process.

```c
struct sigaction sa;
sa.sa_handler = my_handler;
sigemptyset(&sa.sa_mask);
sa.sa_flags = SA_RESTART;
sigaction(SIGTERM, &sa, NULL);
```

**Always call `sigemptyset` on `sa_mask`.** A `struct sigaction` on the stack holds values
that you did not set.

Useful flags:

| Flag | Effect |
|---|---|
| `SA_RESTART` | The kernel restarts some interrupted system calls |
| `SA_SIGINFO` | The handler receives a `siginfo_t` with the cause |
| `SA_NOCLDWAIT` | A child does not become a zombie |
| `SA_NOCLDSTOP` | Do not send `SIGCHLD` when a child stops |
| `SA_ONSTACK` | Run the handler on a separate stack. Use it for `SIGSEGV`. |

---

## 4. Reentrancy: what a handler can call

A signal handler runs at an unpredictable point. The main program can hold a lock. The main
program can sit in the middle of a change to a data structure.

Stevens and Rago give the example. The main program calls `malloc`. `malloc` changes a linked
list of blocks. A signal arrives. The handler also calls `malloc`. The list becomes corrupt.

The Single UNIX Specification names the functions that are safe. They are **async-signal
safe**. Figure 10.4 lists them.

### Functions that a handler may call

The list includes these common functions:

```
_Exit  _exit  abort  accept  access  alarm  bind  chdir  chmod  chown  close  connect
creat  dup  dup2  execl  execle  execv  execve  fchmod  fchown  fcntl  fdatasync  fork
fstat  fsync  ftruncate  getpid  getppid  getuid  kill  link  listen  lseek  lstat  mkdir
open  openat  pause  pipe  poll  pselect  raise  read  readlink  recv  rename  rmdir
select  sem_post  send  sendmsg  setsid  shutdown  sigaction  sigaddset  sigemptyset
sigprocmask  sigsuspend  sleep  socket  stat  symlink  time  times  umask  uname  unlink
utimes  wait  waitpid  write
```

### Functions that a handler must not call

| Function group | Reason |
|---|---|
| `malloc`, `free`, `realloc`, `calloc` | They change a shared list of blocks |
| Every standard I/O function, including `printf` | They use global data in a way that is not reentrant |
| `getpwnam`, `getgrnam`, `strtok`, `localtime`, `inet_ntoa` | They return a pointer to static memory |
| `exit` | It calls the functions that `atexit` registered, and it flushes streams |
| `syslog` | It is not on the list |
| `longjmp`, `siglongjmp` | They can leave a data structure half changed |
| Any mutex lock in your own code | The main program may already hold that mutex. This deadlocks. |

**`printf` in a handler is a defect.** It appears in many examples, including some in APUE.
The authors state that it "is not guaranteed to produce the expected results". Use `write` on
descriptor 2 instead.

### Save and restore `errno`

A handler that calls any function on the safe list can change `errno`. The main program then
reads a value that belongs to the handler.

```c
void handler(int signo) {
    int saved = errno;
    /* ... */
    errno = saved;
}
```

This applies most often to `SIGCHLD`. Every `wait` function changes `errno`.

---

## 5. The only safe handler pattern

**Do the smallest possible work in the handler. Do the real work in the main program.**

Set a flag. The type must be `volatile sig_atomic_t`. That type has two properties: the
compiler does not cache it in a register, and a write to it is one indivisible operation.

```c
static volatile sig_atomic_t got_term = 0;

void handler(int signo) { got_term = 1; }

/* in the main loop */
while (!got_term) {
    /* work */
}
```

Three faults that this pattern prevents:

| Fault | Cause |
|---|---|
| The loop never sees the flag | The compiler kept the variable in a register. `volatile` prevents it. |
| The flag holds a partial value | The type was larger than one word. `sig_atomic_t` prevents it. |
| The handler corrupted the heap | The handler called `malloc`. This pattern calls nothing. |

**A flag alone does not wake a blocked call.** A program that waits inside `read` or `select`
does not test the flag. Section 6 gives the method for that case.

---

## 6. The self-pipe method

Use this method to make a signal wake an event loop.

1. Call `pipe` at the start of the program. Set both ends to nonblocking.
2. In the handler, call `write` with one byte on the write end. `write` is on the safe list.
3. Add the read end to your `select`, `poll`, or `epoll` set.
4. When the loop reports the read end as ready, read the byte. Then handle the signal in your
   main code.

This method turns a signal into normal I/O. The handler stays at one safe call.

Linux gives `signalfd` and `eventfd` for the same purpose. They are not portable.

**Ignore the error from the `write` in the handler.** A full pipe already holds a byte. The
loop wakes for that byte.

---

## 7. `EINTR` and interrupted system calls

A signal that arrives during a slow system call makes that call return −1 with `errno` set to
`EINTR`.

These calls are slow:

- A read or a write on a pipe, a terminal, a socket, or a device
- An `open` on a FIFO
- `wait` and `waitpid`
- `pause`, `sleep`, and `nanosleep`
- `select`, `poll`, and a record lock with `F_SETLKW`

Two responses:

| Method | Limit |
|---|---|
| Set `SA_RESTART` | The kernel does not restart `select`, `poll`, or a socket call with a timeout |
| Write a retry loop | You must write it at every call site |

```c
while ((n = read(fd, buf, len)) < 0 && errno == EINTR)
    ;
```

**A restart is not always correct.** A `read` that already moved some bytes returns the
partial count, not `EINTR`. Your loop must handle the partial count. Read
`01-file-descriptors-and-io.md`, Section 4.

---

## 8. `SIGPIPE`

A write to a pipe or a socket that has no reader raises `SIGPIPE`. The default action stops
the process.

A server that writes to a client that disconnected exits without a message. The log shows
nothing.

Two responses:

1. **Ignore the signal.** Call `signal(SIGPIPE, SIG_IGN)`. Then `write` returns −1 with
   `errno` set to `EPIPE`. Handle `EPIPE` at every write site.
2. **Use `MSG_NOSIGNAL`** in `send` on Linux, or set `SO_NOSIGPIPE` on the socket on BSD and
   macOS.

**Choose method 1 for any network server.** Then check for `EPIPE` on every write.

---

## 9. Signal masks

| Function | Work |
|---|---|
| `sigemptyset`, `sigfillset`, `sigaddset`, `sigdelset` | Build a signal set |
| `sigprocmask` | Block or unblock signals for the process |
| `pthread_sigmask` | Block or unblock signals for one thread |
| `sigpending` | Report the signals that arrived while blocked |
| `sigsuspend` | Set the mask and wait, as one atomic step |

**A blocked signal waits. It does not disappear.** The kernel delivers it after you unblock
it.

**Standard signals do not queue.** Ten `SIGCHLD` signals that arrive while blocked deliver
one time. This is why a `SIGCHLD` handler must call `waitpid` in a loop with `WNOHANG`. Read
`04-processes-and-execution.md`, Section 6.

### The race that `sigsuspend` prevents

This code holds a defect:

```c
sigprocmask(SIG_UNBLOCK, &mask, NULL);   /* the signal can arrive here */
pause();                                  /* it waits forever */
```

The signal arrives between the two calls. `pause` then waits for a signal that already came.
`sigsuspend` performs both steps as one atomic step.

---

## 10. Signals with threads

- **A signal disposition belongs to the process.** Every thread shares one handler for each
  signal.
- **A signal mask belongs to the thread.** Use `pthread_sigmask`, not `sigprocmask`.
- **The kernel delivers a signal to one thread** that does not block it. You cannot predict
  which thread.

**The correct pattern:** block every signal in every thread. Start one thread that calls
`sigwait`. That thread handles the signals as normal code, not as a handler. It can then call
any function.

Read `06-threads-and-concurrency.md`, Section 10.

---

## 11. `kill`, `raise`, and `alarm`

```c
int kill(pid_t pid, int signo);
int raise(int signo);
unsigned int alarm(unsigned int seconds);
```

`kill` sends a signal. The `pid` argument has four meanings:

| Value | Target |
|---|---|
| A positive number | That process |
| 0 | Every process in the group of the sender |
| −1 | Every process that the sender may signal |
| A negative number | Every process in the group with that number |

**`kill(pid, 0)` tests whether a process exists.** It sends no signal. It returns an error
when no process has that number. This test has a race. The number can belong to a new process
by the time you act.

`alarm` sets one timer for each process. A second `alarm` call replaces the first one. A
library that uses `alarm` breaks your timer. Use `setitimer` or `timer_create` when you need
more than one timer.

---

## 12. Stopping a process in an orderly way

Follow this order for a service.

1. Handle `SIGTERM` and `SIGINT` with the flag pattern from Section 5.
2. The main loop sees the flag. It stops accepting new work.
3. Finish the work that is already in progress, up to a deadline.
4. Flush every buffer. Call `fsync` on every file that must be durable.
5. Release the lock file and remove any temporary file.
6. Exit with a status that reports the result.

**Set a deadline for step 3.** A process manager sends `SIGKILL` after a timeout. That signal
gives you no chance to finish. Finish before the deadline of your manager.

---

## Cross-references

- `EINTR` on I/O calls: `01-file-descriptors-and-io.md`, Sections 4 and 10
- Streams are not safe in a handler: `03-standard-io-and-buffering.md`, Section 8
- `SIGCHLD` and `waitpid`: `04-processes-and-execution.md`, Section 6
- Signals in a process with threads: `06-threads-and-concurrency.md`, Section 10
- Named failures: `failure-catalog.md`, section E
