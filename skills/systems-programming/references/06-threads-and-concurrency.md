# Threads and Concurrency

Source: *Advanced Programming in the UNIX Environment*, 3rd Edition, Chapters 11 and 12.

Threads in one process share the address space. That sharing gives speed. That sharing also
removes every protection that the kernel gives between processes.

---

## 1. What threads share and what they do not

| Threads share | Each thread has its own |
|---|---|
| The address space, the heap, and global variables | A stack |
| File descriptors and the file offsets | A thread ID |
| The working directory and `umask` | A signal mask |
| The user ID and the group ID | `errno` |
| Signal dispositions | The scheduling priority |
| Resource limits | Thread-specific data |

**`errno` is separate for each thread.** The implementation defines `errno` as a macro that
finds the value for the calling thread.

**The working directory is shared.** A `chdir` call in one thread changes the path resolution
for every thread. Never call `chdir` in a thread. Use `openat`. Read
`02-files-and-directories.md`, Section 13.

---

## 2. Creating and stopping a thread

```c
int pthread_create(pthread_t *tid, const pthread_attr_t *attr,
                   void *(*start)(void *), void *arg);
int pthread_join(pthread_t tid, void **rval);
int pthread_detach(pthread_t tid);
void pthread_exit(void *rval);
```

**The pthread functions return an error number. They do not set `errno`.**

```c
if ((err = pthread_create(&tid, NULL, worker, arg)) != 0)
    /* err holds the error. errno does not. */
```

Code that tests `errno` after a pthread call reads a value from an older call. This defect
appears in a large amount of thread code.

A thread stops in one of four ways:

| Method | Effect |
|---|---|
| It returns from its start function | Normal |
| It calls `pthread_exit` | Normal |
| Another thread calls `pthread_cancel` | It stops at the next cancellation point |
| Any thread calls `exit`, or the main function returns | **Every thread stops at once** |

**A return from `main` stops every thread.** Call `pthread_join` for each thread, or call
`pthread_exit` in `main` to let the other threads continue.

### Join or detach. Choose one.

A thread that ends holds its resources until another thread joins it. This is the thread
version of a zombie process.

- Call `pthread_join` when you need the result or the completion.
- Call `pthread_detach` when you do not. Then the system frees the resources at once.

**A thread that you neither join nor detach leaks memory.** A server that creates one thread
for each request and does neither runs out of memory.

### Do not return a pointer to your stack

```c
void *worker(void *arg) {
    struct result r;       /* on the stack of this thread */
    return &r;             /* wrong. The stack is gone after the return. */
}
```

Allocate the result with `malloc`, or write into memory that the caller gives you.

---

## 3. Mutexes

```c
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

A mutex protects data. It does not protect code. **Write down which mutex protects which
data.** Put that statement in a comment next to the declaration of the data.

Rules:

- Lock the mutex before every access to the data, including every read.
- Unlock the mutex on every path, including every error path and every early return.
- Hold the mutex for the shortest possible time.
- **Never call an unknown function while you hold a mutex.** That function can lock the same
  mutex or wait for I/O.

**A read without a lock is a defect, not an optimization.** The compiler and the processor can
reorder the access. Another thread can see a value that no thread wrote.

### Deadlock

A deadlock needs two threads and two mutexes. Thread A holds mutex 1 and waits for mutex 2.
Thread B holds mutex 2 and waits for mutex 1. Neither thread continues.

**The remedy is a lock order. Define one order for every mutex in the program. Every thread
locks in that order.** Write the order in a document. A common order uses the memory address
of the mutex.

`pthread_mutex_trylock` gives a second method. When the second lock fails, release the first
lock and start again.

A mutex that a thread locks two times also deadlocks. The default mutex type is not
recursive.

### Mutex types

| Type | Behavior on a second lock by the same thread |
|---|---|
| `PTHREAD_MUTEX_NORMAL` | Deadlock. The behavior is not defined. |
| `PTHREAD_MUTEX_ERRORCHECK` | It returns an error |
| `PTHREAD_MUTEX_RECURSIVE` | It counts the locks. The same thread may lock again. |

Use `PTHREAD_MUTEX_ERRORCHECK` during development. It finds the defect.

**A recursive mutex hides a design fault.** It usually means that two functions lock the same
data and one calls the other. Separate the locking function from the working function
instead.

---

## 4. Condition variables

A condition variable makes a thread wait until another thread reports a change.

```c
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
int pthread_cond_timedwait(pthread_cond_t *cond, pthread_mutex_t *mutex,
                           const struct timespec *timeout);
int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *cond);
```

`pthread_cond_wait` performs three steps as one atomic step. It unlocks the mutex, it waits,
and it locks the mutex again before it returns.

### Always wait inside a loop

```c
pthread_mutex_lock(&lock);
while (queue_is_empty(q))               /* while, not if */
    pthread_cond_wait(&cond, &lock);
item = take_from(q);
pthread_mutex_unlock(&lock);
```

**Use `while`, never `if`.** Three reasons:

1. **A spurious wakeup.** The implementation may wake a thread when no thread signalled. The
   standard permits this behavior.
2. **A stolen wakeup.** A third thread can take the item between the signal and your return.
3. **A broadcast.** `pthread_cond_broadcast` wakes every waiting thread. Only one thread can
   take the item.

A wait with `if` produces a program that takes from an empty queue. The defect appears rarely
and under load only.

### Signal or broadcast

| Call | Use it when |
|---|---|
| `pthread_cond_signal` | Every waiting thread waits for the same condition, and one item serves one thread |
| `pthread_cond_broadcast` | Threads wait for different conditions, or one change serves many threads |

**Use `broadcast` when you are not sure.** It costs processor time. It does not cause a
program that stops.

### Use an absolute time for a timed wait

`pthread_cond_timedwait` takes an absolute time, not a duration. Compute it from
`clock_gettime`. Use a monotonic clock when the attribute permits it. A change to the system
clock then does not change your timeout.

---

## 5. Reader-writer locks and spin locks

| Lock | Use it when |
|---|---|
| Mutex | The default choice |
| Reader-writer lock | Reads are far more frequent than writes, and the critical section is long |
| Spin lock | The critical section is a few instructions, and it never blocks |
| Barrier | Threads must all reach one point before any continues |

**A reader-writer lock costs more than a mutex for a short critical section.** Measure before
you choose it. Some implementations make a writer wait without end when readers keep
arriving.

**Never call a blocking function while you hold a spin lock.** The waiting thread uses the
processor for the whole time.

---

## 6. Thread safety and reentrancy

| Term | Meaning |
|---|---|
| Thread-safe | Several threads may call it at the same time |
| Reentrant | A second call is safe while the first call has not finished |
| Async-signal-safe | A signal handler may call it |

These three terms are not the same. A function that locks a mutex is thread-safe. It is not
async-signal-safe.

### Functions that return static memory

These functions are not thread-safe. Each one has a `_r` version that takes a buffer.

| Function | Safe version |
|---|---|
| `strtok` | `strtok_r` |
| `localtime`, `gmtime` | `localtime_r`, `gmtime_r` |
| `asctime`, `ctime` | `asctime_r`, `ctime_r` |
| `getpwnam`, `getpwuid` | `getpwnam_r`, `getpwuid_r` |
| `getgrnam`, `getgrgid` | `getgrnam_r`, `getgrgid_r` |
| `inet_ntoa` | `inet_ntop` |
| `rand` | `rand_r`, or a generator that you own |

**A signature that returns a pointer, and takes no buffer, is the signature of a function
that is not thread-safe.** Use this test when you read an unknown API.

---

## 7. Thread-specific data

`pthread_key_create`, `pthread_setspecific`, and `pthread_getspecific` give each thread its
own value under one key.

Use it to make an old interface thread-safe without a change to its signature. Give the
destructor function to `pthread_key_create`. The system calls it when the thread ends.

Create the key one time. Use `pthread_once`:

```c
static pthread_once_t once = PTHREAD_ONCE_INIT;
pthread_once(&once, create_the_key);
```

**Thread-specific data does not remove the need for locks.** It only removes the sharing.

---

## 8. Cancellation

`pthread_cancel` requests a stop. It does not force one. The target thread stops at a
**cancellation point**. `read`, `write`, `sleep`, and `pthread_cond_wait` are cancellation
points.

**A cancelled thread that holds a mutex never releases it.** Every other thread then waits
without end.

Push a cleanup handler for every resource:

```c
pthread_cleanup_push(unlock_it, &lock);
/* work */
pthread_cleanup_pop(1);
```

`pthread_cleanup_push` and `pthread_cleanup_pop` must appear in the same block. Some
implementations define them as macros that open and close a brace.

**Prefer a flag over cancellation.** Set a flag that the thread tests in its loop. Cancellation
is difficult to make correct. Most programs do not need it.

---

## 9. `fork` in a process with threads

**Only the calling thread continues in the child.** Every other thread disappears.

This creates the worst defect in this reference. The other threads disappear, but the memory
that they held does not change. A mutex that another thread locked stays locked. **No thread
in the child can unlock it.** The child stops at the first lock attempt.

Any function can lock a mutex inside the library. `malloc` locks one. `printf` locks one. So
the rule is strict:

> **Between `fork` and `exec` in a process that has threads, call only async-signal-safe
> functions.**

That list is in `05-signals.md`, Section 4. It excludes `malloc` and every standard I/O
function.

`pthread_atfork` registers three handlers. The parent locks every mutex before the `fork`
call, and both processes unlock them after it. This method works only when you know every
mutex. A library that you did not write breaks it.

**The reliable method is to call `exec` at once after `fork`.** Use `posix_spawn` where your
platform provides it. Or create every child process before you create any thread.

---

## 10. Threads and signals

Read `05-signals.md`, Section 10 for the full rule. The three facts:

1. A signal disposition belongs to the process. Every thread shares it.
2. A signal mask belongs to the thread. Use `pthread_sigmask`.
3. The kernel chooses one thread that does not block the signal. You cannot predict which one.

**The correct pattern:**

```c
/* in main, before any pthread_create */
sigfillset(&mask);
pthread_sigmask(SIG_BLOCK, &mask, NULL);
/* create the worker threads. They inherit the mask. */
/* create one signal thread that calls sigwait in a loop */
```

The signal thread runs normal code. It can call any function. It needs no handler.

---

## 11. Threads and I/O

Descriptors are shared. The file offset is shared. Two threads that call `read` on one
descriptor receive interleaved data.

| Need | Method |
|---|---|
| Many threads read one file | `pread` and `pwrite`. Each call gives its own offset. |
| Many threads write one log file | One writer thread with a queue, or `O_APPEND` with one `write` for each record |
| Each thread reads a different file | A separate `open` call for each thread |

**A descriptor number can return.** Thread A closes descriptor 7. Thread B opens a file and
receives descriptor 7. Thread C then writes to descriptor 7 and reaches the wrong file. Close
a descriptor only when no other thread can use it.

---

## 12. The review scan for threads

- [ ] Does every `pthread_cond_wait` sit inside a `while` loop that tests the predicate?
- [ ] Does the code define a lock order? Do all paths follow it?
- [ ] Does the code test the return value of every pthread call, and not `errno`?
- [ ] Does every thread receive a `pthread_join` or a `pthread_detach`?
- [ ] Does any thread return a pointer to its own stack?
- [ ] Does the code call a function from the static-memory list in Section 6?
- [ ] Does the code call `chdir`, `umask`, or `setenv` in a thread?
- [ ] Does the code call `fork` while other threads run? Does the child call only safe
      functions?
- [ ] Does the code hold a mutex while it calls a function that can block?
- [ ] Is every shared variable protected, including the ones that the code only reads?

---

## Cross-references

- Shared descriptors and offsets: `01-file-descriptors-and-io.md`, Section 2
- The working directory is shared: `02-files-and-directories.md`, Section 13
- Async-signal-safe functions: `05-signals.md`, Section 4
- `fork` semantics: `04-processes-and-execution.md`, Section 3
- Race conditions across machines: `data-systems-design/references/hazard-catalog.md`
- Named failures: `failure-catalog.md`, section F
