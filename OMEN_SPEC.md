# OMEN Language Specification

> **Version:** 0.2 — Draft  
> **License:** Apache 2.0  
> **File Extension:** `.omn`  
> **Status:** Work in Progress

---

## Table of Contents

1. [Philosophy](#1-philosophy)
2. [Variables](#2-variables)
3. [Memory Management](#3-memory-management)
4. [The `use` Command](#4-the-use-command)
5. [Types](#5-types)
6. [Import System](#6-import-system)
7. [Operators](#7-operators)
8. [Conditionals](#8-conditionals)
9. [Loops](#9-loops)
10. [Linepoint System](#10-linepoint-system)
11. [send / receive](#11-send--receive)
12. [Error Model](#12-error-model)
13. [Standard Libraries](#13-standard-libraries)
14. [Compiler](#14-compiler)
15. [Roadmap / TODO](#15-roadmap--todo)
16. [Quick Reference](#16-quick-reference)

---

## 1. Philosophy

OMEN is a binary-first, zero-abstraction systems programming language designed to replace assembly as the lowest-level human-writable language.

OMEN borrows assembly's CPU communication and compatibility mechanism, but rewrites it entirely under OMEN's own interpretation and binary-focused philosophy. Nothing is hidden from the programmer. Everything is a bit array. Interpretation belongs to the user.

OMEN does not depend on any existing toolchain (LLVM, GCC, etc.).

**Core principles:**

- Everything is binary
- Types are interpretations, not constraints
- There are no errors — only unexpected results
- Responsibility belongs entirely to the programmer
- Libraries exist for convenience, never necessity
- Nothing is hidden

---

## 2. Variables

All variables in OMEN are global. There is no implicit stack. If stack-like behavior is needed, the programmer manages it manually via `drop`.

Variables are stored as raw bit arrays. `bin()` is mandatory when storing to a variable. Omitting it produces the only defined error in OMEN: **unpredictable value**.

```omn
x = bin(int(42))       # correct
x = int(42)            # ERROR: unpredictable value
```

### 2.1 First Parenthesis Rule

The first parenthesis preserves the value. Inside a process (not stored), `bin()` is not required.

```omn
str(42)                # "42" — value preserved, valid in process
int(42)                # integer 42 — valid in process
bin(42)                # binary representation of 42

x = bin(int(42))       # storage — bin required ✅
x = int(42)            # storage — ERROR ❌
```

### 2.2 ID System

Variables can be assigned an ID using `#` to prevent name collisions in parallel operations. Variables with the same name but different IDs are fully independent.

```omn
x#001 = bin(int(42))
x#002 = bin(int(99))

int(x#001)             # 42
int(x#002)             # 99
drop(x#001)            # kills only x#001
```

### 2.3 Seal System

A seal (`!`) is a default type label — a shortcut, not a constraint. When a sealed variable is used without specifying a type, the sealed type is applied automatically. The seal can always be bypassed.

```omn
x = bin(int(42))
x!int                  # seal: default type is int

int(x)                 # reads from seal → 42
byte(x)                # bypasses seal, reads as byte
raw(x)                 # raw binary, no interpretation
int(raw(x))            # take raw, then interpret as int
```

---

## 3. Memory Management

No VM. No garbage collector. The programmer has full manual control over memory. All variables are global until explicitly killed with `drop`.

```omn
x = bin(int(42))
# ... operations ...
drop(x)                # x is dead, memory freed
```

---

## 4. The `use` Command

Temporarily reinterprets a value as a different type without modifying the original. Unlike a seal, `use` is momentary.

```omn
use(int(bin(int(42))), str)    # treat 42 as str
```

| | Seal `!` | `use` |
|--|----------|-------|
| Effect | Sets default type label | Active one-time reinterpretation |
| Persistent | Yes | No |
| Required | No | No |

---

## 5. Types

Types are built into the compiler. They are activated via `#import`. If the built-in behavior is unsatisfactory, the programmer can skip the import and define their own type interpretation. Type conversion is always explicit and open.

```omn
#import int
#import float

int(x)                 # compiler's int interpretation
float(x)               # compiler's float interpretation
```

Complex types such as strings do not exist in the language. They are constructed by the programmer on top of binary primitives.

---

## 6. Import System

`#import` adds a capability to the compiler. Imports are atomic — there are no packages, only individual features. An unimported operator or type cannot be used. Alternatively, the programmer defines their own.

```omn
#import +              # addition
#import -              # subtraction
#import <              # less than
#import >              # greater than
#import int
#import float
```

---

## 7. Operators

### 7.1 Arithmetic

`+` and `-` operate at hardware level and are activated via `#import`.  
`*` and `/` are provided by the `arith` standard library using loop-based algorithms. The programmer may replace them with a custom implementation at any time.

**Multiplication algorithm (inside `arith`):**
```omn
# x * y
z = bin(int(0))
counter = y

loop {
    z = z + x
    counter = counter - bin(int(1))
    if (counter == bin(int(0))) {
        break
    }
}
drop(counter)
# z = result
```

**Division algorithm (inside `arith`):**
```omn
# x / y
# Note: 0/0 causes an infinite loop — mathematically undefined, not an error
z = bin(int(0))

loop {
    x = x - y
    z = z + bin(int(1))
    if (x < bin(int(0))) {
        x = x + y
        z = z - bin(int(1))
        break
    }
}
drop(x)
# z = result
```

### 7.2 User-Defined Operators

Any operator can be overridden without importing the built-in version:

```omn
+(a, b) {
    # custom addition logic
}
```

### 7.3 Operator Precedence

OMEN reads strictly left to right. There is no operator precedence. If order matters, the programmer sequences operations explicitly.

```omn
# 2 + 3 * 4 = 20  (left to right)

# To enforce precedence manually:
x = bin(int(3))
x = x * bin(int(4))
x = x + bin(int(2))
```

---

## 8. Conditionals

Only `if` exists. There is no `else` or `elif`. The programmer constructs compound logic using boolean variables and nested `if` blocks.

**Available comparisons:** `==`, `<`, `>`  
**Not available:** `!=`, `AND`, `OR`, `NOT` — constructed manually by the programmer.

```omn
if (x == bin(int(0))) {
    # executes if true
}

if (x < bin(int(10))) {
    # executes if true
}
```

### 8.1 OR Logic

```omn
bool = bin(int(0))

if (a == bin(int(1))) {
    bool = bin(int(1))
}
if (b == bin(int(2))) {
    bool = bin(int(1))
}
if (bool == bin(int(1))) {
    # executes if either a or b was true
}
drop(bool)
```

### 8.2 AND Logic

```omn
if (a == bin(int(1))) {
    if (b == bin(int(2))) {
        # executes only if both are true
    }
}
```

---

## 9. Loops

Only `loop` exists. There is no `for` or `while`. The programmer builds any loop structure on top of `loop` and `break`. A `loop` block runs indefinitely until a `break` is encountered.

```omn
loop {
    # operations
    break              # programmer decides when to stop
}
```

---

## 10. Linepoint System

Linepoints allow code blocks to be stored as binary, tagged by line number, and reused via `paste`. There is no function abstraction — linepoints operate at the binary level. The compiler marks linepoints by line number at compile time and inserts them at the call site.

```omn
lp#001 = bin(...)      # store code block as binary
paste(lp#001)          # insert and execute at this location
drop(lp#001)           # free the linepoint
```

A linepoint is a variable. It can be passed to `drop`, `send`, or any other command.

---

## 11. send / receive

OMEN's universal communication system. The same two commands handle terminal I/O, network ports, memory addresses, and hardware devices. Only the target path changes.

### 11.1 send

```omn
send(x, /terminal/out)     # write to terminal
send(x, /port/8080)        # send to network port
send(x, /mem/0x00FF)       # write to memory address
send(x, /dev/usb)          # send to hardware device
```

### 11.2 receive

`receive` begins listening on a source, captures the incoming data, and closes. The returned value is always **unsealed binary** — no type is imposed. The programmer seals or interprets the result as needed. Continuous listening is achieved by wrapping `receive` in a `loop`.

```omn
x = receive(/terminal/in)      # read from terminal
x = receive(/port/8080)        # read from network port
x = receive(/dev/usb)          # read from hardware device

x!int                          # optionally seal
int(x)                         # optionally interpret
```

**Continuous listening:**
```omn
loop {
    x = receive(/port/8080)
    # process x
    # break when needed
}
```

> Unlike Python's `input()` which always returns a string, `receive` always returns unsealed binary. The programmer decides what it means.

---

## 12. Error Model

OMEN defines exactly one error:

> **Unpredictable value** — assigning to a variable without wrapping in `bin()`.

Beyond this, the language produces no errors. There are no undefined operations in binary. Unexpected results are the programmer's responsibility.

```omn
# 0/0        → infinite loop (undefined, not an error)
# type mismatch → unexpected result, not an error
# overflow   → binary wraps, execution continues
```

---

## 13. Standard Libraries

Libraries in OMEN are optional convenience layers. Everything a library provides can be written in raw OMEN.

| Library | Description |
|---------|-------------|
| `arith` | Multiplication and division via loop-based algorithms. Programmer may replace with a custom implementation. |
| `postcpu` | Automatic CPU detection and core selection, inspired by assembly's CPUID mechanism. C-style auto-detection. |
| `postram` | Automatic memory management (Python-style). For programmers who prefer not to call `drop` manually. |
| `put` | Terminal convenience layer: `print()` wraps `send(x, /terminal/out)`, `input()` wraps `receive(/terminal/in)`. |
| Community | Additional libraries written and maintained by the open-source community under Apache 2.0. |

---

## 14. Compiler

- Written in C
- Borrows assembly's CPUID-based CPU detection logic, rewritten under OMEN's philosophy
- Produces binary directly for the target CPU
- Does not depend on any external toolchain (LLVM, GCC, etc.)
- Does not use assembly mnemonics — OMEN has its own instruction set
- Marks linepoints by line number at compile time and inserts them at call sites

---

## 15. Roadmap / TODO

| # | Item | Status |
|---|------|--------|
| 1 | CPU instruction set definition | 🔲 Pending |
| 2 | Linepoint parameter passing | 🔲 Pending |
| 3 | Compiler pipeline (Lexer → Parser → Codegen) | 🔲 Pending |
| 4 | `postcpu` internal architecture | 🔲 Pending |
| 5 | `postram` internal architecture | 🔲 Pending |
| 6 | `arith` full implementation | 🔲 Pending |
| 7 | `put` implementation | 🔲 Pending |

---

## 16. Quick Reference

| Feature | OMEN |
|---------|------|
| Type system | Interpretation-based, opt-in via `#import` |
| Memory | Full manual control via `drop` |
| Error model | One defined error: unpredictable value |
| Loop | `loop` only — `for`/`while` built by programmer |
| Conditional | `if` only — `else`/`elif` built by programmer |
| Functions | None — replaced by linepoint system |
| I/O | `send` / `receive` — universal, path-based |
| Operator precedence | None — strictly left to right |
| Arithmetic (`* /`) | `arith` library |
| Incoming data | `receive` → always unsealed binary |
| License | Apache 2.0 |
| File extension | `.omn` |

---

*OMEN — In memory of assembly.* ⚔️
