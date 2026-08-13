# Standard I/O and Buffering

Source: *Advanced Programming in the UNIX Environment*, 3rd Edition, Chapter 5.

The standard I/O library adds a buffer between your program and the kernel. The goal is to
use the smallest number of `read` and `write` calls.

Stevens and Rago state that buffering is the part of this library that "generates the most
confusion". Most defects in this area come from a buffer that the program did not flush.

Every language has this layer. C uses `FILE`. Python uses a file object. Java uses
`BufferedWriter`. Node.js uses a stream. **The rules below apply to all of them.**

---

## 1. The three buffer modes

| Mode | The library writes when | Default for |
|---|---|---|
| Fully buffered | The buffer becomes full | A file on a disk |
| Line buffered | The library finds a newline | A terminal |
| Unbuffered | At once, on every call | Standard error |

The library allocates the buffer with `malloc` at the first I/O call on the stream.

**Standard error is never fully buffered.** This rule exists so that an error message reaches
the user before the program stops.

### The rule that surprises engineers

The mode depends on **where the output goes**, not on your code.

```
./myprogram              -> standard output goes to a terminal -> line buffered
./myprogram > out.txt    -> standard output goes to a file     -> fully buffered
./myprogram | grep x     -> standard output goes to a pipe     -> fully buffered
```

A program that prints progress works at a terminal. The same program in a pipeline prints
nothing for a long time. Then it prints everything at the end.

**This is not a defect in the pipeline. It is the buffer mode.**

Two responses:

- Call `fflush` after each message that a user must see at once.
- Call `setvbuf` with `_IOLBF` at the start of the program to force line buffering.

Two more caveats for line buffering. Stevens and Rago give both:

1. The buffer has a fixed size. A line that is longer than the buffer causes a write before
   the newline arrives.
2. A read from an unbuffered stream, or from a line-buffered stream that must reach the
   kernel, flushes **every** line-buffered output stream.

---

## 2. `fflush` and the loss of data at exit

`fflush(fp)` writes the buffer of one stream. `fflush(NULL)` writes the buffer of every
output stream.

**A buffer that you do not flush is lost in these cases:**

| Case | Result |
|---|---|
| The process calls `_exit` or `_Exit` | Every buffer is lost |
| The process receives a signal that stops it | Every buffer is lost |
| The process calls `abort` | Every buffer is lost |
| The process calls `exec` | Every buffer is lost |
| A child after `fork` also exits | The buffer is written **two times** |

`exit` flushes the buffers. `_exit` does not. A signal handler that calls `_exit` loses the
output. That behavior is correct for a handler, because `exit` is not async-signal-safe.
Read `05-signals.md`.

### The double output defect after `fork`

`fork` copies the memory of the parent. That copy includes the buffers of the standard I/O
library. The parent and the child then both write the same bytes.

```c
printf("start\n");     /* goes into the buffer when output is a file */
fork();                /* the child receives a copy of the buffer */
/* both processes write "start" */
```

**Call `fflush(NULL)` before every `fork`.** This rule applies in every language that
buffers output.

---

## 3. A buffer flush is not a durable write

These are three separate levels. A defect appears when an engineer treats them as one level.

| Level | The call that moves the data | The data is safe against |
|---|---|---|
| Library buffer to the kernel | `fflush` | The process stopping |
| Kernel buffer to the disk | `fsync` | The machine losing power |
| Name in the directory to the disk | `fsync` on the directory | The machine losing power |

**`fflush` alone gives no durability.** The full method is:

```c
fflush(fp);              /* library buffer -> kernel */
fsync(fileno(fp));       /* kernel -> disk */
fclose(fp);              /* check the result */
```

Read `02-files-and-directories.md`, Section 7 for the full durable update.

**`fclose` flushes and closes.** Check its return value. It reports a write error that no
earlier call could report.

---

## 4. Do not mix a stream with a descriptor

A `FILE` object and a file descriptor for the same file hold two separate positions. The
stream has a buffer. The descriptor does not.

```c
fprintf(fp, "a");        /* stays in the library buffer */
write(fileno(fp), "b", 1);   /* reaches the kernel now */
fclose(fp);              /* writes "a" now */
/* the file holds "ba" */
```

**Choose one method for each file.** When you must change methods, call `fflush` first, then
call `fileno`.

The same rule applies in a high-level language. A Python program that writes with a file
object and also writes with `os.write` on the same descriptor produces mixed output.

---

## 5. Line input

| Function | Status |
|---|---|
| `getline` | **Use this one.** It allocates the buffer and grows it. |
| `fgets` | Acceptable. You must check for the newline to detect a long line. |
| `gets` | **Never use this.** It has no limit. The standard removed it. |

`fgets` reads at most `n − 1` bytes. When the line is longer, the buffer holds no newline.
The next call returns the rest of the same line. Code that treats each call as one line
splits the long line into two records.

**Check for the newline after every `fgets` call.** Or use `getline`.

Every line reader has one more limit: a line has no maximum length. A file that holds one
line of 4 GB defeats a program that reads a whole line into memory. Set a limit and report
the lines that reach it.

---

## 6. Binary data

`fread` and `fwrite` move whole objects. They return the count of **objects**, not the count
of bytes.

```c
size_t fread(void *ptr, size_t size, size_t nobj, FILE *fp);
size_t fwrite(const void *ptr, size_t size, size_t nobj, FILE *fp);
```

A short count is normal. Call `ferror` and `feof` to tell an error from an end of file.

**Do not write a `struct` with `fwrite` and then read it on another machine.** Three
properties change between machines:

1. The byte order of an integer
2. The padding that the compiler inserts between members
3. The size of a type such as `long`

Use a defined format instead. Read `data-systems-design/references/04-encoding-and-evolution.md`.

---

## 7. Formatted output

| Function | Safety |
|---|---|
| `snprintf` | **Use this one.** It takes the buffer size. |
| `sprintf` | **Never use this.** It has no limit. |
| `vsnprintf` | Use this one inside your own message function |

`snprintf` returns the length that the full output **needs**. That number can be larger than
the buffer. **Do not use the return value as the length of the text that it wrote.** Compare
it against the buffer size to detect a cut.

**Never pass data from a user as the format string.** A format string that a user controls
reads the stack and writes to memory. This is a privilege defect, not a formatting defect.

```c
printf(user_text);            /* wrong */
printf("%s", user_text);      /* correct */
```

---

## 8. When not to use a stream

Use a file descriptor instead of a stream in these cases:

| Case | Reason |
|---|---|
| You need a durable write | A stream adds a buffer that hides the timing |
| You need an atomic append from many processes | The library buffer breaks the record boundary |
| You use `select`, `poll`, or `epoll` | Data in the library buffer makes the descriptor look empty |
| You write from a signal handler | No stream function is async-signal-safe |
| You need `pread` or `pwrite` | A stream has one offset |
| You move large blocks | The extra copy gives no gain |

**The multiplexing case causes a defect that is hard to find.** `select` reports that a
descriptor holds no data. Your stream buffer already holds a full line. The program waits
without end. Use descriptors alone in an event loop.

---

## 9. Memory streams

`fmemopen`, `open_memstream`, and `open_wmemstream` make a stream that writes into memory.
Use them to build a string with the `printf` functions, without a temporary file.

`open_memstream` allocates the buffer and grows it. The pointer and the length become valid
only after `fflush` or `fclose`.

---

## Cross-references

- Descriptors, `fsync`, and short counts: `01-file-descriptors-and-io.md`
- The durable update: `02-files-and-directories.md`, Section 7
- Buffers across `fork`: `04-processes-and-execution.md`
- Async-signal-safe functions: `05-signals.md`
- Named failures: `failure-catalog.md`, section C
