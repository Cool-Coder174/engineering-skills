# File Descriptors and I/O

Source: *Advanced Programming in the UNIX Environment*, 3rd Edition, Chapters 3 and 14.

This reference covers unbuffered I/O. Unbuffered means that every call reaches the kernel.
The C library adds no buffer. For buffered streams, read `03-standard-io-and-buffering.md`.

---

## 1. What a file descriptor is

A file descriptor is a small non-negative integer. The kernel uses it as an index. A process
receives a file descriptor from `open`, `creat`, `dup`, `pipe`, `socket`, or `accept`.

Three numbers have a fixed meaning by convention:

| Number | Constant | Use |
|---|---|---|
| 0 | `STDIN_FILENO` | Standard input |
| 1 | `STDOUT_FILENO` | Standard output |
| 2 | `STDERR_FILENO` | Standard error |

**The kernel always returns the lowest number that is free.** This behavior makes redirection
work. Close descriptor 1, then open a file. The file receives descriptor 1.

This behavior also creates a defect. Code that closes a descriptor and then opens a file can
receive a number that another part of the program still uses. Use `dup2` when you need a
specific number.

---

## 2. The three kernel tables

The kernel holds open files in three levels. Learn this structure. It explains every sharing
rule below.

| Level | One per | It holds |
|---|---|---|
| Descriptor table | Process | The descriptor flags, and a pointer to a file table entry |
| File table | Each `open` call | The status flags, the current offset, and a pointer to a v-node |
| V-node or i-node | File | The file type, the size, and the functions that act on the file |

Three consequences follow:

1. **Two `open` calls on one file give two offsets.** The two file table entries are separate.
2. **`dup` and `fork` share one offset.** Both descriptors point to the same file table entry.
   A `read` by one moves the offset for the other.
3. **The descriptor flags do not transfer.** `FD_CLOEXEC` lives in the descriptor table. It
   belongs to one process and one descriptor.

Rule 3 is the source of a frequent defect. `dup` does not copy `FD_CLOEXEC`. The new
descriptor stays open across `exec`.

---

## 3. `open` and `openat`

```c
int open(const char *path, int oflag, ... /* mode_t mode */);
int openat(int fd, const char *path, int oflag, ... /* mode_t mode */);
```

Exactly one of these three flags must appear:

| Flag | Meaning |
|---|---|
| `O_RDONLY` | Read only |
| `O_WRONLY` | Write only |
| `O_RDWR` | Read and write |

These flags change the behavior. Learn what each one prevents:

| Flag | Effect | Use it to prevent |
|---|---|---|
| `O_APPEND` | The kernel moves the offset to the end before every write | Lost records when many processes append |
| `O_CREAT` | Create the file when it does not exist | — |
| `O_EXCL` | With `O_CREAT`, fail when the file exists | A race between two creators |
| `O_TRUNC` | Set the length to 0 | — |
| `O_NONBLOCK` | Return at once instead of waiting | A stop that has no end |
| `O_NOFOLLOW` | Fail when the last part of the path is a symbolic link | A symbolic link attack |
| `O_CLOEXEC` | Close this descriptor at `exec` | A descriptor leak into a child process |
| `O_SYNC` | Every write waits for the disk | Data loss after a power loss |
| `O_DIRECTORY` | Fail when the path is not a directory | An open of the wrong object type |

**Set `O_CLOEXEC` at `open` time.** The other method needs two calls, `open` and `fcntl`. A
thread can call `fork` in the gap between them. The child then holds the descriptor.

**`openat` removes a race.** It resolves the path against an open directory descriptor. The
directory cannot change between the two steps. Use `openat` in every recursive walk.

### The `mode` argument

Give the `mode` argument only with `O_CREAT`. The kernel applies the file mode creation mask
to it. The result is `mode & ~umask`. Read `02-files-and-directories.md` for the mask.

---

## 4. `read`, `write`, and the short count

```c
ssize_t read(int fd, void *buf, size_t nbytes);
ssize_t write(int fd, const void *buf, size_t nbytes);
```

| Return value | Meaning |
|---|---|
| A positive number | That many bytes moved. **The number can be smaller than `nbytes`.** |
| 0 from `read` | End of file. For a socket, the peer closed its end. |
| −1 | An error. Read `errno`. |

A `read` call returns fewer bytes than you request in these cases:

- The file has fewer bytes left before the end.
- The source is a pipe, a FIFO, or a socket that holds fewer bytes.
- The source is a terminal that returned one line.
- A signal interrupted the call after some bytes moved.
- The kernel reached a record boundary on a device.

A `write` call moves fewer bytes than you request in these cases:

- The pipe or the socket buffer became full, and the descriptor is nonblocking.
- A signal interrupted the call after some bytes moved.
- The file system reached its size limit or its space limit.

**Always write the loop.** Stevens and Rago give `readn` and `writen` in Section 14.7:

```c
ssize_t writen(int fd, const void *ptr, size_t n)
{
    size_t nleft = n;
    ssize_t nwritten;
    while (nleft > 0) {
        if ((nwritten = write(fd, ptr, nleft)) < 0) {
            if (nleft == n) return -1;   /* error on the first call */
            break;                        /* report the partial count */
        } else if (nwritten == 0) {
            break;
        }
        nleft -= nwritten;
        ptr = (char *)ptr + nwritten;
    }
    return n - nleft;
}
```

### I/O size

Stevens and Rago measure the CPU time against the buffer size. The time falls as the buffer
grows. The time stops falling when the buffer reaches the block size of the file system.

**Choose a buffer of 32 KB or more for bulk transfer.** A larger buffer gives no further
gain. A buffer of one byte costs the most.

---

## 5. `lseek` and the file offset

```c
off_t lseek(int fd, off_t offset, int whence);
```

| `whence` | The offset becomes |
|---|---|
| `SEEK_SET` | `offset` bytes from the start |
| `SEEK_CUR` | The current value plus `offset` |
| `SEEK_END` | The size of the file plus `offset` |

Three rules:

1. **Compare the result against −1, not against a negative number.** Some devices permit a
   negative offset. Use `if (lseek(fd, 0, SEEK_CUR) == -1)`.
2. **`lseek` does no I/O.** It only changes a number in the file table entry.
3. **A seek past the end of the file creates a hole.** A `read` of the hole returns zero
   bytes. The hole uses no disk space. The size of the file grows.

A file with a hole reports a large size from `ls`. It reports a small block count from `du`.
A copy program that reads and writes every byte removes the hole. The copy uses more disk
space than the original file.

### `pread` and `pwrite`

```c
ssize_t pread(int fd, void *buf, size_t nbytes, off_t offset);
ssize_t pwrite(int fd, const void *buf, size_t nbytes, off_t offset);
```

These calls seek and transfer as one atomic step. They have two differences from the pair:

- No other call can run between the seek and the transfer.
- The current file offset does not change.

**Use `pread` when many threads read one file.** Each thread gives its own offset. The
threads need no lock and no separate descriptor.

---

## 6. Atomic operations

Section 2 of `../SKILL.md` gives Rule 4. This section lists the atomic calls that the kernel
provides.

| Logical operation | The atomic form |
|---|---|
| Move to the end, then write | `O_APPEND` at `open` time |
| Seek, then read or write | `pread` or `pwrite` |
| Test for a file, then create it | `O_CREAT` with `O_EXCL` |
| Close a descriptor, then reuse the number | `dup2` |
| Resolve a path, then open the file | `openat` |
| Replace the content of a file | Write a temporary file, then `rename` |

### Why `O_APPEND` matters

Stevens and Rago give this example. Process A and process B both open one log file without
`O_APPEND`. Both call `lseek` to the end. Both receive offset 1500.

Process B writes 100 bytes. The file grows to 1600. Process A then writes at offset 1500.
**Process A overwrites the record that process B wrote.**

`O_APPEND` removes the gap. The kernel moves the offset and writes as one step.

---

## 7. `dup`, `dup2`, and `fcntl`

```c
int dup(int fd);            /* returns the lowest free number */
int dup2(int fd, int fd2);  /* returns fd2, after it closes fd2 */
```

`dup2` closes `fd2` and duplicates `fd` onto it as one atomic step. The two-call form is not
equal:

```c
close(fd2);          /* a signal handler or a thread can run here */
fcntl(fd, F_DUPFD, fd2);
```

**Use `dup2`.** Use it to connect a pipe to standard input or standard output before `exec`.

`fcntl` performs five groups of work:

| Command | Work |
|---|---|
| `F_DUPFD`, `F_DUPFD_CLOEXEC` | Duplicate a descriptor |
| `F_GETFD`, `F_SETFD` | Read or set the descriptor flags. `FD_CLOEXEC` is the only one. |
| `F_GETFL`, `F_SETFL` | Read or set the status flags |
| `F_GETOWN`, `F_SETOWN` | Choose the process that receives `SIGIO` |
| `F_GETLK`, `F_SETLK`, `F_SETLKW` | Record locking |

**A defect appears in almost every use of `F_SETFL`.** The status flags hold the access mode
in three bits. Read the flags first. Add your bit. Then set the result.

```c
val = fcntl(fd, F_GETFL, 0);
val |= O_NONBLOCK;
fcntl(fd, F_SETFL, val);
```

Code that calls `fcntl(fd, F_SETFL, O_NONBLOCK)` erases the access mode.

---

## 8. `sync`, `fsync`, and `fdatasync`

The kernel holds written data in a buffer cache. It writes the buffer to the disk later.
Stevens and Rago call this a **delayed write**.

```c
void sync(void);
int fsync(int fd);
int fdatasync(int fd);
```

| Function | Scope | It waits for the disk |
|---|---|---|
| `sync` | Every changed buffer | No |
| `fsync` | One file, data and attributes | Yes |
| `fdatasync` | One file, data only | Yes |

**Three rules for durable data:**

1. Call `fsync` on the file before you report success to a user.
2. Call `fsync` on the **directory** after you create, rename, or remove a name in it.
   A directory entry is data in the directory, not in the file.
3. Check the return value. `fsync` reports a write error that `write` could not report.

`O_SYNC` makes every `write` wait for the disk. It costs more than `fsync` at the end of a
group of writes. Use `O_SYNC` only when every single write must be durable.

**A disk can hold data in its own cache.** `fsync` asks the disk to flush that cache. Some
consumer disks report success before the flush finishes. A power loss can still lose the
data on that hardware.

---

## 9. Nonblocking I/O

A slow system call can wait without a limit. These calls are slow:

- A read from a pipe, a terminal, a socket, or a device that holds no data
- A write to a pipe, a terminal, a socket, or a device that is full
- An `open` on a FIFO that has no other user
- A record lock with `F_SETLKW`

Set `O_NONBLOCK` to make these calls return at once. The call returns −1 with `errno` set to
`EAGAIN` when it cannot proceed.

**Nonblocking I/O alone wastes the processor.** A loop that calls `read` until it returns
data is a busy wait. Use `select`, `poll`, or `epoll` to wait for readiness. Then read.

**Nonblocking mode belongs to the open file description, not to the descriptor.** A
nonblocking flag on standard input also applies to every process that shares it. A shell can
report an error after your program exits.

---

## 10. I/O multiplexing

| Function | Limit | Notes |
|---|---|---|
| `select` | `FD_SETSIZE`, often 1024 | POSIX. It changes its arguments. Set them again in every loop. |
| `poll` | No fixed limit | POSIX. It uses an array. It scales better than `select`. |
| `epoll`, `kqueue` | No fixed limit | Not POSIX. Linux uses `epoll`. BSD and macOS use `kqueue`. |

**`select` fails silently above `FD_SETSIZE`.** A descriptor number of 1024 or higher writes
outside the `fd_set`. This defect corrupts memory. Use `poll` when the descriptor count can
grow.

Four rules for every multiplexing loop:

1. **Readiness is not a promise.** A `read` after `select` can still return `EAGAIN`. Keep
   the descriptor nonblocking.
2. **Handle `EINTR` on the wait call.** `SA_RESTART` does not restart `select` or `poll`.
3. **Read until `EAGAIN`** when you use edge-triggered mode in `epoll`. One read is not
   enough.
4. **Remove a descriptor from the set before you close it.** A closed number can return as a
   new descriptor.

---

## 11. `readv`, `writev`, and `mmap`

### Scatter and gather

```c
ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
```

`writev` writes several buffers as one atomic call. Use it to write a header and a body
without a copy and without a second call. This method keeps a record whole on a pipe.

### Memory-mapped I/O

```c
void *mmap(void *addr, size_t len, int prot, int flag, int fd, off_t off);
int munmap(void *addr, size_t len);
int msync(void *addr, size_t len, int flags);
```

| Flag | Effect |
|---|---|
| `MAP_SHARED` | Changes reach the file and other processes |
| `MAP_PRIVATE` | Changes stay in this process. The file does not change. |

Use `mmap` when you read one file many times, or when you jump between offsets. Do not use
`mmap` for a single pass from start to end.

**Four hazards:**

1. **`SIGBUS` after a truncation.** Another process shortens the file. Your access to the
   removed pages raises `SIGBUS`.
2. **The map size is fixed at map time.** A file that grows does not extend the map.
   Call `mmap` again.
3. **`msync` is not `fsync`.** Call `msync` with `MS_SYNC` to make the changes durable.
4. **A write error appears as a signal.** A full disk can raise `SIGBUS` instead of an error
   return.

**Do not use `mmap` on a file that another machine can change.** A network file system gives
no guarantee for shared pages.

---

## 12. Record locking

```c
int fcntl(int fd, int cmd, struct flock *lock);
```

| Command | Behavior when another process holds the lock |
|---|---|
| `F_GETLK` | Report the holder |
| `F_SETLK` | Return −1 at once, with `errno` set to `EACCES` or `EAGAIN` |
| `F_SETLKW` | Wait |

Set `l_len` to 0 to lock from `l_start` to the end of the file, including any future growth.

**Two behaviors cause defects. Both come from the POSIX definition.**

1. **A lock belongs to the process.** Two threads in one process do not block each other.
   POSIX record locks do not work between threads.
2. **Any `close` releases every lock on that file.** A library function that opens the same
   file and closes it releases the locks that your code holds.

Consequences to check on review:

- A `F_SETLKW` lock can deadlock. The kernel detects some cases and returns `EDEADLK`. It
  does not detect every case.
- A lock does not stop a process that never asks for the lock. POSIX locks are advisory.
- A lock on a network file system depends on a lock daemon. Do not depend on it.

Use `flock` when you need a lock that follows the open file description. `flock` is not in
POSIX. Linux and BSD support it.

---

## 13. `/dev/fd`

`/dev/fd/n` names the file that descriptor `n` holds open. An `open` on `/dev/fd/0` gives a
new descriptor for standard input.

Use it to give a file name to a program that accepts only a name, when your data is on a
descriptor or in a pipe. The shell syntax `<(command)` uses this method.

---

## Cross-references

- File names, permissions, and directories: `02-files-and-directories.md`
- Buffered streams and `FILE`: `03-standard-io-and-buffering.md`
- `EINTR` and signal handlers: `05-signals.md`
- Descriptors across `fork` and `exec`: `04-processes-and-execution.md`
- Named failures: `failure-catalog.md`, section A
