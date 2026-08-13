# Files and Directories

Source: *Advanced Programming in the UNIX Environment*, 3rd Edition, Chapters 4, 5.13, and 13.

Chapter 3 covers the content of a file. This reference covers the **name** of a file, its
**attributes**, and the **directory** that holds the name. Most file management defects live
here, not in `read` and `write`.

---

## 1. `stat`, `fstat`, `lstat`, and `fstatat`

```c
int stat(const char *path, struct stat *buf);
int fstat(int fd, struct stat *buf);
int lstat(const char *path, struct stat *buf);
int fstatat(int fd, const char *path, struct stat *buf, int flag);
```

| Function | It acts on | It follows a symbolic link |
|---|---|---|
| `stat` | A path | Yes |
| `lstat` | A path | **No.** It reports the link itself. |
| `fstat` | An open descriptor | The question does not apply |
| `fstatat` | A path, relative to a directory descriptor | Only without `AT_SYMLINK_NOFOLLOW` |

**Prefer `fstat`.** A path can name a different file at every call. A descriptor names one
file for its whole life.

**Use `lstat` in a recursive walk.** `stat` follows a link. A link that points to a parent
directory makes a walk that never stops.

### Useful members of `struct stat`

| Member | Content |
|---|---|
| `st_mode` | The file type and the permission bits |
| `st_ino` | The i-node number |
| `st_dev` | The device that holds the file |
| `st_nlink` | The count of hard links |
| `st_uid`, `st_gid` | The owner and the group |
| `st_size` | The length in bytes |
| `st_blocks` | The count of 512-byte blocks that the file uses |
| `st_atime`, `st_mtime`, `st_ctime` | The times. Section 8 explains them. |

The pair `st_dev` and `st_ino` names a file without ambiguity on one machine. Use the pair to
detect two names for one file. Use the pair to detect a loop in a walk.

`st_size` is meaningful for a regular file, a directory, and a symbolic link. It has no
meaning for a pipe or for most devices.

---

## 2. File types

Test the type with a macro. Do not compare `st_mode` against a number.

| Macro | Type |
|---|---|
| `S_ISREG` | Regular file |
| `S_ISDIR` | Directory |
| `S_ISLNK` | Symbolic link |
| `S_ISFIFO` | Pipe or FIFO |
| `S_ISSOCK` | Socket |
| `S_ISCHR` | Character special file |
| `S_ISBLK` | Block special file |

**Check the type before you read.** A program that expects a regular file and receives a FIFO
waits until a writer appears. A program that receives `/dev/zero` reads until it runs out of
memory.

**A document ingest task must test `S_ISREG`.** Skip every other type.

---

## 3. Permission bits and the access test

Nine bits control access. Three bits apply to the owner, three to the group, and three to
everyone else.

| Bits | Value | Meaning for a regular file | Meaning for a directory |
|---|---|---|---|
| Read | 4 | Read the content | List the names |
| Write | 2 | Change the content | Create, rename, and remove names |
| Execute | 1 | Run the file | Pass through the directory to reach a name inside |

**A directory needs the execute bit for every path that passes through it.** Read permission
on a directory lets you list the names. Execute permission lets you use a name. The two are
separate.

**Write permission on a directory controls deletion, not write permission on the file.** A
user who can write to a directory can remove any file in it. The permissions of the file do
not prevent this action. The sticky bit changes this rule.

### The sticky bit

Set the sticky bit (`S_ISVTX`) on a shared directory. Then only three users can remove or
rename a file in it: the owner of the file, the owner of the directory, and the superuser.
`/tmp` uses this bit.

**Any program that writes to a shared directory must expect the sticky bit rules.**

### Set-user-ID and set-group-ID

These bits make a program run with the identity of the file owner, not the identity of the
caller. They are the source of most privilege defects on UNIX.

- Never set these bits on a program that you did not design for them.
- Never set these bits on a shell script.
- A set-user-ID program must check every path, every environment variable, and every
  argument. Use `security-engineering` for that review.

### `access` and `faccessat`

`access` tests permission with the real user ID, not the effective user ID.

**Do not call `access` before `open`.** Stevens and Rago name this pattern as a race. An
attacker replaces the path between the two calls. The test passes for one file. The `open`
acts on another file.

The correct method has one step: call `open`, then handle the error. When a set-user-ID
program must test the real user, use `faccessat` with `AT_EACCESS`, or drop the privilege and
then call `open`.

---

## 4. `umask` and the mode of a new file

The file mode creation mask removes bits from the mode that a program requests.

```
final mode = requested mode AND NOT umask
```

A `umask` of 022 removes group write and other write. A request for 0666 becomes 0644.

Three rules:

1. **A daemon must call `umask`.** It inherits the mask of whatever started it. Stevens and
   Rago give this as coding rule 1 for a daemon.
2. **`umask` belongs to the process.** A change affects every thread and every later file.
   Set it one time at the start.
3. **`umask` does not apply to `chmod`.** Use `chmod` or `fchmod` to set an exact mode.

**Use `fchmod` on a descriptor, not `chmod` on a path.** The path can change between the
`open` call and the `chmod` call.

---

## 5. Hard links, `unlink`, and the true delete rule

```c
int link(const char *existingpath, const char *newpath);
int unlink(const char *path);
int remove(const char *path);
int rename(const char *oldname, const char *newname);
```

A directory entry holds a name and an i-node number. `st_nlink` counts these entries.

**`unlink` removes a name. It does not always remove the file.** The kernel removes the
content only when both counts reach zero:

1. The link count `st_nlink`
2. The count of processes that hold the file open

This rule produces three behaviors that engineers must know:

- **Disk space does not return until every process closes the file.** A daemon that holds a
  removed log file open keeps the space. `du` shows free space. `df` shows a full disk. The
  fix is to restart the process or to make it reopen the file.
- **An open file that you `unlink` becomes a private temporary file.** No other process can
  find the name. The kernel removes the content when your process exits, even after a crash.
  This is the safest temporary file method.
- **A `write` to a removed file still works.** The descriptor stays valid.

Two limits on `link`:

- A hard link cannot cross a file system. The i-node number has meaning only inside one file
  system.
- Only the superuser can make a hard link to a directory. Most systems refuse it. A link of
  this kind can make a loop in the file tree.

---

## 6. `rename` is the atomic replace

```c
int rename(const char *oldname, const char *newname);
```

`rename` performs the change as one atomic step. When `newname` exists, the kernel replaces
it. A reader sees the old file or the new file. A reader never sees a state between the two.

**This one call is the base of every safe file update.**

Three limits:

1. `rename` fails with `EXDEV` across file systems. Create the temporary file in the **same
   directory** as the target.
2. `rename` does not make the change durable. Call `fsync` on the directory after it.
3. `rename` on a directory needs the new name to be an empty directory or to not exist.

---

## 7. The durable file update

This is the standard method. Use it for a configuration file, a cache file, an index, a
checkpoint, and any file that a person can lose.

**Never open the target file with `O_TRUNC` and write over it.** A crash in the middle leaves
a file that holds part of the old content and part of the new content. The file has a valid
name and a valid size. No reader can detect the damage.

### The six steps

1. Build a temporary name in the **same directory** as the target. Call `mkstemp`.
2. Write the full content. Use a loop, because `write` can move fewer bytes.
3. Call `fchmod` and `fchown` to set the permissions that the target needs. `mkstemp` creates
   the file with mode 0600.
4. Call `fsync` on the temporary file. Then call `close`. **Check both results.**
5. Call `rename` to move the temporary name onto the target name.
6. Open the **directory** that holds the target. Call `fsync` on it. Then close it.

### What each step prevents

| Step | Without it |
|---|---|
| 1, same directory | `rename` fails with `EXDEV` |
| 2, the loop | The file holds a short count of bytes |
| 3 | The file has mode 0600. Other users lose access. |
| 4, `fsync` | The rename can complete before the data reaches the disk. The file holds zero bytes after a power loss. |
| 4, check `close` | A delayed write error goes unreported |
| 5, `rename` | A reader sees a file that is half written |
| 6, directory `fsync` | The new name can disappear after a power loss |

**Step 4 must happen before step 5.** A rename that reaches the disk before the data leaves a
file with the correct name and no content. This ordering is the single most important rule in
this reference.

### When you must remove the temporary file

Delete the temporary file on every error path. A failed write leaves a file that nobody
owns. A pipeline that runs many times fills the disk with these files.

---

## 8. Symbolic links

A symbolic link holds a path as its content. The kernel follows it during path resolution.

| Function | It follows the link |
|---|---|
| `stat`, `open`, `chmod`, `chown` | Yes |
| `lstat`, `readlink`, `symlink`, `unlink`, `rename` | No |

`readlink` reads the content of the link. It does not add a null byte. Add the null byte
yourself from the returned length.

**Four hazards:**

1. **A loop.** A link that points to a parent directory makes a walk that never stops. The
   kernel returns `ELOOP` after a limit for one path. A recursive walk in your code has no
   such limit. Use `lstat`.
2. **A link that points outside the tree.** An archive or an upload can hold a link to
   `/etc/passwd`. Resolve the target and compare it against your root directory before you
   use it.
3. **A dangling link.** The target does not exist. `stat` fails. `lstat` succeeds.
4. **A replacement race.** An attacker replaces a path element with a link between your test
   and your use. Use `O_NOFOLLOW` and `openat`.

---

## 9. File times

| Field | It changes when | `ls` shows it with |
|---|---|---|
| `st_atime` | Something reads the file | `-u` |
| `st_mtime` | Something changes the **content** | The default |
| `st_ctime` | Something changes the **i-node** | `-c` |

The i-node changes for a permission change, an owner change, a link count change, and a
content change. **You cannot set `st_ctime`.** The kernel controls it.

Use `futimens` or `utimensat` to set the access time and the modification time. These
functions give nanosecond precision.

**Do not use a modification time to detect a change in an ingest pipeline.** Three reasons:

1. The resolution can be one second on an old file system. Two changes in one second look
   like one change.
2. Any program can set the time to any value.
3. A file copy often keeps the old time.

Use a content hash, a size with a time, or an i-node change time. State which one you chose.

---

## 10. Reading a directory

```c
DIR *opendir(const char *path);
DIR *fdopendir(int fd);
struct dirent *readdir(DIR *dp);
int closedir(DIR *dp);
```

`struct dirent` holds two members that POSIX guarantees: `d_ino` and `d_name`. Some systems
add `d_type`. **`d_type` is not portable.** Some file systems report `DT_UNKNOWN`. Call
`fstatat` when `d_type` is not available.

### Rules for a safe walk

1. **Skip `.` and `..` before anything else.** Without this test, the walk does not stop.
2. **Use `lstat` or `fstatat` with `AT_SYMLINK_NOFOLLOW`.** Do not follow links.
3. **Use `openat` and `fstatat` against the descriptor of the parent directory.** Do not join
   strings and then call `open`. This method removes the race, and it removes the path length
   limit.
4. **Record `st_dev` and `st_ino` for each directory that you enter.** A repeat of a pair
   means a loop. Stop.
5. **Limit the depth.** Set a maximum. Report the directories that reach it.
6. **`readdir` returns no defined order.** Sort the names when your output must be stable.
7. **A change during the walk gives no guarantee.** A name that another process adds during
   your walk may appear or may not appear.
8. **Count your open descriptors.** A recursive walk that keeps a descriptor for every level
   reaches `EMFILE`. Use `nftw` or close each descriptor as you leave the level.

`readdir` is not reentrant on some systems. It returns a pointer to static memory. Two
threads that walk two trees can corrupt each other. Use `readdir_r` only when your platform
documents it, because POSIX marks it as obsolete. The better method gives each thread its own
`DIR` object.

---

## 11. Path safety

**Do not trust `PATH_MAX`.** Some systems do not define it. Some file systems permit a longer
path. A fixed buffer for a path is a defect.

Use one of two methods:

- Use `openat` with one component at a time. Then no full path exists in your program.
- Allocate the buffer at run time. Grow it when the call returns `ERANGE`.

**Check every path that comes from a user or from an archive:**

| Check | It prevents |
|---|---|
| Reject a component that equals `..` | Escape from the target directory |
| Reject a path that starts with `/` | A write to an absolute location |
| Reject a name that holds a null byte | A truncated path in a C library |
| Resolve the final path and compare it against your root | A symbolic link that points outside |
| Reject an empty component | A path that resolves in a way that you did not expect |

A check on the string alone is not enough. A symbolic link changes the meaning after the
check. Combine the string check with `O_NOFOLLOW` and `openat`.

---

## 12. Temporary files

| Function | Status |
|---|---|
| `mkstemp` | **Use this one.** It creates the file and returns a descriptor. |
| `mkdtemp` | Use this one for a temporary directory |
| `tmpfile` | Acceptable. It creates a file and removes the name at once. |
| `tmpnam`, `tempnam` | **Never use these.** They return a name, not a file. |
| `mktemp` | **Never use this.** It is removed from the standard. |

`tmpnam` and `mktemp` create a race. They give you a name. Another process can create that
name before your `open` call. `mkstemp` creates the file and gives you the descriptor as one
step.

Two rules for `mkstemp`:

- The template must end with exactly six `X` characters. The function changes the string in
  place. The template cannot be a string constant.
- `mkstemp` does not remove the file. Delete it yourself on every path.

**For the durable update in Section 7, the temporary file must sit in the target directory.**
`/tmp` is often a different file system. `rename` fails across file systems.

---

## 13. `chdir`, `fchdir`, and `getcwd`

The working directory belongs to the **process**. A change affects every thread.

**Do not call `chdir` in a library or in a thread.** Another thread resolves a relative path
at the same time. That thread reaches the wrong file. Use `openat` with a directory
descriptor instead.

A daemon changes its working directory to `/` or to its own data directory. Stevens and Rago
give this as coding rule 4 for a daemon. A working directory on a mounted file system stops
that file system from unmounting.

---

## 14. File systems and the limits that break an ingest task

| Limit | Symptom | Response |
|---|---|---|
| Free blocks reach zero | `ENOSPC` from `write` or from `close` | Check the space before a large job. Report the error. |
| Free i-nodes reach zero | `ENOSPC` with free space present | Many small files caused it. Combine them. |
| Descriptor limit for the process | `EMFILE` | Close descriptors. Raise the limit with `setrlimit`. |
| Descriptor limit for the system | `ENFILE` | Reduce the concurrent job count |
| Name length limit | `ENAMETOOLONG` | Shorten the name. Use a hash of the name. |
| A file larger than the offset type | `EOVERFLOW` | Build with 64-bit file offsets |

**A full disk is a normal condition, not an exception.** Handle `ENOSPC` on every write path.
Check the result of `close` for it.

---

## Cross-references

- `open`, `read`, `write`, `fsync`, and locking: `01-file-descriptors-and-io.md`
- Buffered streams: `03-standard-io-and-buffering.md`
- Daemon rules and the working directory: `04-processes-and-execution.md`
- Path attacks and privilege: `security-engineering/SKILL.md`
- Named failures: `failure-catalog.md`, section B
