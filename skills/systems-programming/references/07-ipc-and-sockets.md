# Interprocess Communication and Sockets

Source: *Advanced Programming in the UNIX Environment*, 3rd Edition, Chapters 15, 16, and 17.

---

## 1. Choose the mechanism

| Mechanism | Between | Direction | Persists after the last close |
|---|---|---|---|
| Pipe | A parent and a child, or related processes | One way | No |
| FIFO (named pipe) | Any process on one machine | One way | The name persists. The data does not. |
| Unix domain socket | Any process on one machine | Two ways | The name persists |
| TCP socket | Any process on any machine | Two ways | No |
| UDP socket | Any process on any machine | Two ways, no order | No |
| Shared memory | Any process on one machine | Both read and write | Yes, until removal |
| Message queue | Any process on one machine | One way, with records | Yes, until removal |
| Semaphore | Any process on one machine | It signals only | Yes, until removal |
| A file with a lock | Any process on one machine | Both read and write | Yes |

**Choose a Unix domain socket for a new program on one machine.** It gives two directions,
it gives connections, it carries record boundaries with `SOCK_SEQPACKET` or `SOCK_DGRAM`, and
it can pass a file descriptor. The XSI mechanisms in Section 5 have an older interface.

**Choose a pipe when a parent starts a child.** It is the smallest correct method.

---

## 2. Pipes

```c
int pipe(int fd[2]);
```

`fd[0]` is the read end. `fd[1]` is the write end. Data moves in one direction.

The standard method for a parent and a child:

1. Call `pipe` before `fork`.
2. Call `fork`.
3. **The reader closes `fd[1]`. The writer closes `fd[0]`.** Both processes must do this.
4. Use `dup2` to connect the end to descriptor 0 or 1 when the child calls `exec`.

**Step 3 is the step that engineers omit. It causes two defects:**

| Omission | Result |
|---|---|
| The reader keeps the write end open | `read` never returns 0. The reader waits without end. |
| The writer keeps the read end open | The writer fills the pipe and waits, instead of receiving `SIGPIPE` |

A `read` on a pipe returns 0 only when **every** descriptor for the write end is closed. Your
own copy counts.

### Behavior at the ends

| Event | Result |
|---|---|
| Read from an empty pipe that has a writer | The call waits |
| Read from an empty pipe that has no writer | It returns 0, which means end of file |
| Write to a pipe that has no reader | `SIGPIPE`. With the signal ignored, `write` returns `EPIPE`. |
| Write to a full pipe | The call waits |

### Atomic writes and the pipe size

`PIPE_BUF` is the largest write that the kernel performs as one atomic step. POSIX sets a
minimum of 512 bytes. Linux uses 4096.

**A write that is larger than `PIPE_BUF` can interleave with a write from another process.**
Keep each record smaller than `PIPE_BUF` when several writers share one pipe.

**A deadlock appears when both processes write first.** Process A fills the pipe to B and
waits. Process B fills the pipe to A and waits. Neither reads. Use `select` or `poll` on both
descriptors, or set both to nonblocking.

### `popen` and `pclose`

`popen` runs a command through a shell and gives you a stream. It has the same defect as
`system`. **Never pass a string that holds input from a user.** Read
`04-processes-and-execution.md`, Section 7.

`pclose` returns the exit status of the shell. Read it with the macros in
`04-processes-and-execution.md`, Section 6.

---

## 3. FIFOs

```c
int mkfifo(const char *path, mode_t mode);
```

A FIFO has a name in the file system. Unrelated processes can then open it.

Rules:

- An `open` for reading waits until a writer opens the FIFO. An `open` for writing waits
  until a reader opens it. Set `O_NONBLOCK` to change this behavior.
- The data does not persist. A reader that arrives later receives nothing.
- The `PIPE_BUF` rule from Section 2 also applies.
- Remove the name with `unlink` when the program ends.

**A FIFO is not a queue that survives a restart.** Use a file or a message broker when the
data must survive.

---

## 4. Message framing

A stream carries bytes. It does not carry records. TCP and a pipe both give a stream.

**A `read` call does not match a `write` call.** One `write` of 100 bytes can arrive as three
`read` calls. Three `write` calls can arrive as one `read` call.

Choose one framing method and write it down:

| Method | Format | Notes |
|---|---|---|
| Length prefix | A fixed-size length, then the body | The best choice for binary data. Set a maximum length. |
| A delimiter | The body, then a byte such as a newline | You must escape the delimiter inside the body |
| A fixed size | Every record has the same size | Only for records that never change |
| A datagram socket | The kernel keeps the boundary | Use `SOCK_SEQPACKET` on a Unix domain socket |

**Check the length against a maximum before you allocate memory.** A length field that a peer
controls becomes an allocation that a peer controls.

---

## 5. XSI IPC: message queues, semaphores, and shared memory

These three mechanisms come from System V. They share one design and three faults.

| Mechanism | Create | Use |
|---|---|---|
| Message queue | `msgget` | `msgsnd`, `msgrcv` |
| Semaphore | `semget` | `semop` |
| Shared memory | `shmget` | `shmat`, `shmdt` |

**The three faults:**

1. **They persist after every process exits.** A crash leaves the object in the kernel. It
   uses memory until an administrator removes it with `ipcrm`. A program that restarts finds
   the old object with the old data.
2. **They use a key, not a path.** `ftok` builds a key from a path and a number. Two different
   paths can give one key. The identifier is not a file descriptor, so `select` and `poll`
   cannot wait for it.
3. **They have no reference count that removes them.** Nothing frees the object when the last
   user exits.

**Prefer the POSIX versions for new code.** `shm_open`, `sem_open`, and `mq_open` use names
that look like paths. `shm_open` returns a file descriptor.

### Shared memory rules

Shared memory gives no synchronization. The kernel maps the same pages into two processes.
Nothing else.

- **Use a semaphore or a mutex with the process-shared attribute** to protect the data.
- Set `PTHREAD_PROCESS_SHARED` on a mutex that lives in shared memory. Without it, the
  behavior is not defined.
- **Store no pointer in shared memory.** The segment can sit at a different address in each
  process. Store an offset from the start of the segment.
- A process that stops while it holds the lock stops every other process. Set the
  `PTHREAD_MUTEX_ROBUST` attribute where your platform provides it. A later locker then
  receives `EOWNERDEAD` and can repair the data.

---

## 6. Sockets

```c
int socket(int domain, int type, int protocol);
```

| Domain | Use |
|---|---|
| `AF_INET`, `AF_INET6` | Between machines |
| `AF_UNIX` | On one machine |

| Type | Behavior |
|---|---|
| `SOCK_STREAM` | Ordered, reliable, two-way, a stream of bytes. TCP. |
| `SOCK_DGRAM` | Records. No order. Not reliable. UDP. |
| `SOCK_SEQPACKET` | Ordered, reliable, and it keeps the record boundary |

### The server sequence

1. `socket`
2. `setsockopt` with `SO_REUSEADDR`
3. `bind`
4. `listen`
5. `accept` in a loop

### The client sequence

1. `socket`
2. `connect`

### Rules that prevent the frequent defects

**Set `SO_REUSEADDR` before `bind`.** Without it, a restart fails with `EADDRINUSE`. The old
connections stay in the `TIME_WAIT` state for up to four minutes.

**Handle `EINTR` on `accept`.** `SA_RESTART` does not restart `accept` on every system.

**Handle a partial `send`.** `send` returns the count that it moved. That count can be smaller
than the length. Write the loop.

**Ignore `SIGPIPE`.** A write to a closed connection stops the process by default. Read
`05-signals.md`, Section 8.

**A `read` that returns 0 means that the peer closed its end.** It is not an error. Do not
treat it as one.

**Set a timeout.** Use `SO_RCVTIMEO` and `SO_SNDTIMEO`, or use `select` and `poll`. A TCP
connection to a machine that lost power can wait for many minutes. Read
`data-systems-design/references/08-distributed-systems-faults.md`.

**Set `TCP_NODELAY` for a request-and-response protocol.** The Nagle algorithm holds a small
write until the peer confirms the earlier data. This can add 40 milliseconds to each request.

### The descriptor limit

Each connection uses one descriptor. A server reaches `RLIMIT_NOFILE` and then `accept`
returns `EMFILE`.

**A loop that treats `EMFILE` as fatal stops the server.** A loop that ignores it runs
without end at full processor use. Hold one spare descriptor. Close it, accept the connection,
close the connection, and open the spare again.

---

## 7. Unix domain sockets

A Unix domain socket has a path in the file system. It gives the socket interface with no
network.

Three properties that no other mechanism gives:

1. **It is faster than a TCP socket on one machine.** No checksum and no protocol work.
2. **It carries permissions.** The permission bits of the path control who can connect. A TCP
   socket on the loopback address does not.
3. **It can pass a file descriptor and a credential.**

### Passing a file descriptor

Use `sendmsg` and `recvmsg` with a control message of type `SCM_RIGHTS`.

The receiving process receives a **new descriptor** for the same open file description. The
number differs. The offset is shared.

Use this method to give a privileged descriptor to a process that could not open the file
itself. A listener process can open a port below 1024 and then pass the descriptor to a
worker that holds no privilege.

`SO_PEERCRED` on Linux and `LOCAL_PEERCRED` on BSD report the user ID of the peer. Use it to
authenticate a local client.

### Rules

- Remove the path with `unlink` before `bind`. A leftover file makes `bind` fail with
  `EADDRINUSE`.
- Remove the path when the program ends.
- Put the socket in a directory with restrictive permissions. The permission check on the
  socket file itself is not portable.
- The path has a small maximum length, often 108 bytes. It is shorter than `PATH_MAX`.

---

## 8. The review scan for IPC

- [ ] Does each process close the pipe ends that it does not use?
- [ ] Does the code define a framing method for every stream?
- [ ] Does the code check a length from a peer against a maximum before it allocates memory?
- [ ] Does the code handle a partial `send` and a partial `recv`?
- [ ] Does the code ignore `SIGPIPE` and handle `EPIPE`?
- [ ] Does every network call have a timeout?
- [ ] Does the code set `SO_REUSEADDR` before `bind`?
- [ ] Does the code handle `EMFILE` from `accept` without a loop that never stops?
- [ ] Does shared memory have a lock, and does the lock have the process-shared attribute?
- [ ] Does shared memory hold a pointer?
- [ ] Does the code remove every XSI IPC object and every socket path at exit?

---

## Cross-references

- Nonblocking I/O, `select`, and `poll`: `01-file-descriptors-and-io.md`, Sections 9 and 10
- `SIGPIPE`: `05-signals.md`, Section 8
- Pipes across `fork`: `04-processes-and-execution.md`, Section 3
- Timeouts and network faults: `data-systems-design/references/08-distributed-systems-faults.md`
- Named failures: `failure-catalog.md`, section G
