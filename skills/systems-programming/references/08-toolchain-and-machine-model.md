# The Machine Model and the Toolchain

Source: *Systems Programming*, John J. Donovan, McGraw-Hill, 1972, Chapters 1 to 5 and 8.
The APUE sections give the modern form.

Donovan wrote about an IBM 360. The machine changed. **The structure of the toolchain did
not.** An engineer who knows these steps can read a link error. An engineer who does not know
them can only try changes until one works.

---

## 1. Approach an unknown machine with four questions

Donovan gives a general method to understand any new machine. Ask these four questions:

1. **Memory.** How much? What is the smallest addressable unit? How does the machine form an
   address?
2. **Registers.** How many? How wide? Which ones have a special purpose?
3. **Data.** Which types does the hardware support? What is the byte order? What alignment
   does each type need?
4. **Instructions.** Which operations exist? Which addressing modes exist?

The method still applies. Use it for a new processor, a new virtual machine, or a new
bytecode format.

**The answers explain the defects that you will meet:**

| Answer | The defect that it explains |
|---|---|
| The byte order | A binary file that one machine writes and another machine cannot read |
| The alignment rules | A `struct` that changes size between compilers, and a `SIGBUS` on some processors |
| The register width | An integer that wraps around at a different value |
| The address form | A pointer that does not fit in an `int` |

---

## 2. The three time periods

Donovan separates three periods. Keep them apart when you debug. A symptom in one period has
a cause in that period.

| Period | Name | What happens | The tool |
|---|---|---|---|
| 1 | Assembly time or compile time | The source becomes an object file | The compiler and the assembler |
| 2 | Load time | The program enters memory and the addresses become correct | The linker and the loader |
| 3 | Execution time | The processor runs the instructions | The kernel and the dynamic linker |

**Name the period before you look for the cause.** An "undefined symbol" at period 1 and an
"undefined symbol" at period 3 have different causes and different fixes.

---

## 3. The components of a programming system

Donovan lists the programs that turn source text into a running program. Each one solves one
problem.

| Component | Input | Output | The problem it solves |
|---|---|---|---|
| Macro processor | Source with macro calls | Source with the text expanded | A programmer repeats identical text |
| Assembler | Assembly source | An object file | A programmer must not write numeric opcodes and addresses |
| Compiler | High-level source | Assembly source or an object file | A programmer must not write for one machine |
| Loader and linker | Object files | A program in memory | A program uses parts that separate translations produced |

Every modern toolchain still has these four parts. `cpp`, `cc1`, `as`, and `ld` map onto
them.

---

## 4. The assembler and the two-pass rule

Donovan gives a general design procedure for a system program. `../SKILL.md`, Section 4 states
the five steps. This section shows why the assembler needs two passes.

**The problem is the forward reference.** An instruction can name a symbol before the source
defines that symbol.

```
        B    LOOP        <- the assembler does not know the address of LOOP yet
        ...
LOOP    L    2,DATA      <- the definition appears here
```

Pass 1 reads the whole source. It assigns an address to each statement, and it records each
label in the **symbol table**. Pass 2 reads the source again. Every symbol now has a value,
so pass 2 can generate the code.

**The general rule applies to any tool that you write:**

> Ask this question. Does any output depend on input that the program has not read yet?
> When the answer is yes, the program needs a second pass or a patch list.

A patch list is the other method. The program writes a placeholder, records the location, and
fills the value later. A single-pass assembler uses this method. So does a JSON writer that
must put a length before a body that it has not built.

### The tables that an assembler holds

| Table | Content |
|---|---|
| Machine operation table | The mnemonic, the length, and the binary opcode |
| Pseudo-operation table | The directives that control the assembler, such as `START` and `END` |
| Symbol table | Each label and its address |
| Literal table | Each constant that the source wrote in place |
| Base register table | Which register holds which base address |

Donovan separates the **data structure** from the **format of the data base**. The structure
says what the table holds. The format says how the bytes sit. **Write both down.** A format
that you did not write down becomes a defect at the boundary between pass 1 and pass 2.

---

## 5. The macro processor

A macro processor works on text. It expands a call into the body before the assembler or the
compiler reads the text.

Donovan lists four features. Each one adds a requirement to the implementation:

| Feature | What the processor must hold |
|---|---|
| Arguments | A table that maps each parameter to its value |
| Conditional expansion | A test, and a way to skip text |
| A macro call inside a macro | A stack of expansions |
| A macro that defines a macro | A definition step at expansion time |

**A macro expands before the language sees it.** This is the reason for the defects that the C
preprocessor causes:

```c
#define SQUARE(x) x * x
SQUARE(a + b)          /* becomes a + b * a + b, not (a+b)*(a+b) */
```

Two rules follow. Put parentheses around every parameter and around the whole body. Do not
name an argument two times in the body, because the caller can pass an expression that has a
side effect.

**A macro is not a function.** It has no type check, no scope, and no address. Use a function
when a function works. This rule applies to a C macro, to a build template, and to a code
generator.

---

## 6. The four functions of a loader

Donovan states that a relocating loader performs four functions. **Every loader performs
these four. The types differ only in when each one happens.**

| Function | Work |
|---|---|
| **Allocation** | Reserve the memory space for the program |
| **Linking** | Resolve the symbolic references between object files |
| **Relocation** | Adjust every address-dependent location to match the allocated space |
| **Loading** | Place the instructions and the data into memory |

### The loader types

| Type | When the four functions happen | Cost |
|---|---|---|
| Compile-and-go | The translator writes the code into memory at once | The translator uses the memory. No object file exists. |
| Absolute | The programmer chooses the address. The assembler does the rest. | The programmer must place every part |
| Relocating | The loader relocates at load time | The object file must record every address to adjust |
| Direct-linking | The loader links and relocates at load time, with a full symbol table | The standard method for a static build |
| Dynamic | Linking and relocation wait until the program calls the symbol | The load is faster. Some errors move to run time. |

**This table explains a modern build.** A static link performs all four functions before the
program runs. A shared library moves linking and relocation into period 3. That change is why
a missing library appears when you start the program, not when you build it.

Donovan gives the reason that relocation exists at all. Programmers wanted to name subroutines
by symbol. They did not want to know the address of any part of their program.

---

## 7. The modern form: symbols, sections, and the dynamic linker

An object file today holds sections and a symbol table.

| Section | Content | Permissions |
|---|---|---|
| `.text` | The instructions | Read and run |
| `.rodata` | Constants and string literals | Read |
| `.data` | A variable with a value at compile time | Read and write |
| `.bss` | A variable that starts at zero. It uses no space in the file. | Read and write |
| `.symtab` | The symbols that this file defines and needs | — |
| `.rela.*` | The relocation records | — |

### How the linker resolves a symbol

1. It reads each object file in the order that you gave.
2. It records each symbol that a file defines.
3. It records each symbol that a file needs and no file defines.
4. For a static library, it takes only the members that resolve a symbol that is still open.
5. At the end, an open symbol is an "undefined reference".

**The order of the arguments matters for a static library.** The linker takes members from a
library only for the symbols that are open at that point. Put the libraries after the object
files that use them.

```
cc main.o -lfoo      correct
cc -lfoo main.o      "undefined reference", because no symbol was open at -lfoo
```

### The errors and the function that failed

| Message | The function from Section 6 | Cause |
|---|---|---|
| "undefined reference to X" at build time | Linking | No object file defines X, or the library came before its user |
| "multiple definition of X" | Linking | Two files define X. A variable in a header often causes it. |
| "undefined symbol: X" at start time | Linking, delayed to run time | The shared library changed after the build |
| "cannot open shared object file" | Allocation | The loader cannot find the library. Check `RPATH` and `LD_LIBRARY_PATH`. |
| "version GLIBC_2.34 not found" | Linking | The build machine has a newer library than the run machine |
| "text file busy" (`ETXTBSY`) | Loading | You wrote to a program file that a process is running |
| "relocation R_X86_64_32 against ..." | Relocation | An object without `-fPIC` went into a shared library |
| The wrong function runs | Linking | Two libraries define one name. The first one wins. |

### Static and dynamic linking

| Property | Static | Dynamic |
|---|---|---|
| Start time | Faster | Slower. The dynamic linker resolves the symbols. |
| File size | Larger | Smaller |
| A security fix in a library | Rebuild every program | Replace one file |
| Memory across processes | One copy for each process | One copy for the whole machine |
| Which library runs | Fixed at build time | The environment can change it |

**The last row is a privilege risk.** `LD_PRELOAD` and `LD_LIBRARY_PATH` change which code
runs. The dynamic linker ignores them for a set-user-ID program. Read
`security-engineering/SKILL.md`.

---

## 8. The address space of a running process

| Region | Content |
|---|---|
| Text | The instructions. Read-only and shared between processes. |
| Data and bss | Global variables |
| Heap | `malloc`. It grows toward a higher address. |
| Memory maps | Shared libraries and `mmap` regions |
| Stack | Frames and local variables. It grows toward a lower address. |
| Kernel | Not reachable from the process |

Three protections apply on a modern system:

| Protection | Effect | What it defeats |
|---|---|---|
| Address space layout randomization | The regions sit at a different address on every run | An attack that needs a fixed address |
| No-execute pages | The processor refuses to run an instruction from the stack or the heap | Code that an attacker wrote into a buffer |
| Read-only relocations | The linker makes the relocation tables read-only after the start | An attack that changes a function pointer in the table |

**A defect that a fixed address hides appears when randomization is active.** Code that works
in a debugger and fails in production often depends on an address or on uninitialized memory.
A debugger often turns randomization off.

---

## 9. Storage classes

Donovan lists the storage classes that a language provides. The names differ today. The
meanings do not.

| Donovan | C | Where it lives | Lifetime |
|---|---|---|---|
| Static | `static`, and a global variable | Data or bss | The whole program |
| Automatic | A local variable | Stack | The block |
| Controlled | `malloc` and `free` | Heap | Until `free` |
| Based | A pointer to a controlled block | Heap, through a pointer | Until `free` |

The class decides three things: where the storage sits, when it appears, and who releases it.
**Most memory defects are a mismatch between the class and the use.**

| Defect | The mismatch |
|---|---|
| A pointer to a local variable that the function returned | Automatic storage that outlived its block |
| Use after free | Controlled storage that the code released too early |
| A leak | Controlled storage that no code released |
| A stack overflow | A large automatic object, or recursion with no limit |
| A double free | Two owners for one controlled block |

**Name one owner for every allocation.** Write the owner in a comment at the allocation site.
Most leaks and most double frees come from an ownership rule that nobody wrote down.

---

## 10. The compiler phases

Donovan lists the phases of a compiler. Read this table to know which phase reports your
error.

| Phase | Work | The error that it reports |
|---|---|---|
| Lexical | Group the characters into tokens | An invalid character. An unterminated string. |
| Syntax | Build the tree from the tokens | A missing brace. An unexpected token. |
| Interpretation | Check the types. Build the intermediate form. | A type mismatch. An undeclared name. |
| Optimization | Improve the intermediate form | A warning about code that never runs |
| Storage assignment | Give a location to each variable | A frame that is too large |
| Code generation | Produce the machine instructions | A register limit. An unsupported operation. |
| Assembly | Produce the object file | — |

**An error from an early phase hides the errors after it.** Fix the first error, then build
again. A long list of errors after one missing brace has one cause.

---

## 11. When this reference applies to you

You may never write an assembler. You will meet these ideas in this form:

| Donovan's topic | Where you meet it |
|---|---|
| Two passes and forward references | A parser, a template engine, a schema tool, a bundler |
| Macro expansion | The C preprocessor, a build template, a code generator |
| Symbol resolution | An import cycle, a duplicate dependency, a version conflict |
| Relocation | A container image, a build that a machine cannot reproduce, `-fPIC` |
| Storage classes | Every memory defect in Section 9 |
| The four loader functions | Every message in the table in Section 7 |
| A statement of the problem and the format of the data base | Every tool that you write |

---

## Cross-references

- Process memory and `malloc`: `04-processes-and-execution.md`, Section 1
- `ETXTBSY` and writing to a running program: `01-file-descriptors-and-io.md`
- `LD_PRELOAD` and privilege: `security-engineering/SKILL.md`
- Named failures: `failure-catalog.md`, section H
