# OMEN

> **The point where a computer and a human finally speak the same language.**

OMEN doesn't translate human language to binary. It lets humans speak binary directly.

A computer doesn't know what `int` or `str` means. If it doesn't, OMEN doesn't either. When you want something treated as a number, you tell the CPU to interpret those bits as int. That's all a type ever was.

---

## Why OMEN

Every language above assembly adds abstraction. Every abstraction hides something. OMEN hides nothing.

- Assembly forces you to manage registers, calling conventions, architecture-specific mnemonics
- C forces you to manage pointers, stack frames, type casting
- Python forces you into a runtime, a GC, and a type system

OMEN gives you two things: **bit arrays** and **memory**. Everything else is yours to build — or import if you don't want to.

> *OMEN — In memory of assembly.* ⚔️

---

## Core Rules

- Outside parentheses, only **variables** and **binary arrays** exist
- `bin()` is mandatory when storing to a variable — omitting it produces the only defined error: **unpredictable value**
- All variables are global. No implicit stack
- No operator precedence. Strictly left to right
- No errors. Only unexpected results — and that's your responsibility

---

## Variables

```omn
x = bin(int(42))         # storage — bin required
x = bin(int(0))          # zero

str(42)                  # inside process → "42"
int(42)                  # inside process → 42
bin(42)                  # binary representation
```

A variable is not a container. It is an address label. Under the hood, it points to a memory location holding raw binary. Nothing more.

### ID System

```omn
x#001 = bin(int(42))     # for parallel operations
x#002 = bin(int(99))
drop(x#001)              # only x#001 dies
```

### Seal

A seal is a default type label — a shortcut, not a constraint. Always bypassable.

```omn
x!int                    # seal: default type is int
int(x)                   # reads from seal
byte(x)                  # bypasses seal
raw(x)                   # raw binary, no interpretation
int(raw(x))              # take raw, interpret as int
```

### use

```omn
use(int(bin(int(42))), str)    # one-time type reinterpretation

# different type arithmetic:
x = bin(int(5))
y = bin(float(3.4))
x_float = use(x, float)
result = x_float + y           # same type now, CPU can process
```

### drop

```omn
drop(x)                  # kill variable, free memory
```

---

## Types

```omn
#import int
#import float
#import str
```

Types are built into the compiler. Import them to use the compiler's interpretation. Don't import them and define your own. Type conversion is always explicit — the programmer decides, the CPU applies.

---

## Import System

```omn
#import +
#import -
#import <
#import >
#import int
#import float
#import postcpu
#import postram
#import arith
#import put
```

Imports are atomic. No packages — only individual capabilities. An unimported operator or type cannot be used. Imports go at the top of the file. The compiler reads them once at compile time — zero runtime overhead.

---

## Arithmetic

```omn
#import +
#import -

x = x + y               # hardware level
x = x - y               # hardware level
```

`*` and `/` come from the `arith` library. Don't like the implementation? Write your own:

```omn
*(a, b) {
    # your multiplication logic
}
```

---

## Conditionals

```omn
if (x == bin(int(0))) { }
if (x < bin(int(10))) { }
if (x > bin(int(5)))  { }
```

No `else`, `elif`, `!=`, `AND`, `OR`, `NOT`. Build them yourself.

### OR Logic

```omn
bool = bin(int(0))
if (a == bin(int(1))) { bool = bin(int(1)) }
if (b == bin(int(1))) { bool = bin(int(1)) }
if (bool == bin(int(1))) {
    # a or b is true
}
drop(bool)
```

### AND Logic

```omn
if (a == bin(int(1))) {
    if (b == bin(int(1))) {
        # both are true
    }
}
```

---

## Loops

```omn
loop {
    # operations
    break
}
```

No `for`, no `while`. Build them on top of `loop`.

---

## Linepoint System

No functions. Linepoints instead. A linepoint stores a code block as binary, tagged by line number. The compiler inserts it at the call site at compile time.

```omn
lp#001 = bin(...)        # store code block as binary
paste(lp#001)            # insert and execute here
drop(lp#001)             # free the linepoint
```

A linepoint is a variable. It can be passed to `drop`, `send`, or any other command.

---

## send / receive

OMEN's universal communication system. One command for everything — only the target path changes.

### send

```omn
send(x, /terminal/out)   # write to terminal
send(x, /port/8080)      # send to network port
send(x, /mem/0x00FF)     # write to memory address
send(x, /dev/usb)        # send to hardware device
send(x, /gpu/vram)       # send to GPU
```

### receive

`receive` listens on a source, captures incoming data, and closes. Returns always **unsealed binary**. Seal or interpret after.

```omn
x = receive(/terminal/in)
x = receive(/port/8080)
x = receive(/dev/usb)

x!int                        # optionally seal
int(x)                       # optionally interpret
```

### Continuous Listening

```omn
loop {
    x = receive(/port/8080)
    # process
    # break when needed
}
```

---

## Memory Levels

Three levels of memory control. Choose yours.

### Raw OMEN — Firmware / Kernel level

```omn
send(63728282828, bin(int(42)))
```
You know the address. You write it. Full control.

### postcpu — C level

```omn
postcpu.variable(postcpu.ramfind(4, byte), x=bin(int(42)))
postcpu.address(x)                    # get address of x
postcpu.parallel(functionA, functionB)    # run in parallel
```

### postram — Python level

```omn
postram.variable(bin(int(42)))
drop(x)                               # lifetime still yours
```

---

## Standard Libraries

Libraries are optional. Everything a library provides can be written in raw OMEN. Libraries exist for convenience, never necessity.

| Library | What it does | Level |
|---------|-------------|-------|
| `arith` | `*` and `/` algorithms | Basic |
| `postcpu` | CPU detection, ramfind, parallel, address | C level |
| `postram` | Automatic memory placement — includes postcpu | Python level |
| `put` | `print()` and `input()` terminal shortcuts | Convenience |

### put

```omn
#import put

print(x)                 # shortcut for send(x, /terminal/out)
y = input()              # shortcut for receive(/terminal/in)
```

### Library Hierarchy

```
postram
    └── postcpu (included automatically)
            └── raw OMEN
```

`#import postram` → postcpu is included automatically.

---

## Error Model

One defined error:

> **Unpredictable value** — assigning to a variable without `bin()`.

```omn
x = bin(int(42))    # ✅
x = int(42)         # ❌ unpredictable value
```

Everything else:

| Situation | Result |
|-----------|--------|
| `0 / 0` | Infinite loop — mathematically undefined, not an error |
| Type mismatch | Unexpected result — your responsibility |
| Overflow | Binary wraps, execution continues |

OMEN doesn't fail. You do. And that's the point.

---

## Quick Example

```omn
#import +
#import -
#import int
#import put

x = bin(int(10))
x!int

loop {
    x = x - bin(int(1))
    print(x)
    if (x == bin(int(0))) {
        break
    }
}

drop(x)
```

---

## Roadmap

| # | Item | Status |
|---|------|--------|
| 1 | CPU instruction set definition | 🔲 |
| 2 | Linepoint parameter passing | 🔲 |
| 3 | Compiler pipeline (Lexer → Parser → Codegen) | 🔲 |
| 4 | postcpu internal architecture | 🔲 |
| 5 | postram internal architecture | 🔲 |
| 6 | arith full implementation | 🔲 |
| 7 | put implementation | 🔲 |
| 8 | Bare metal display / GPU framebuffer | 🔲 |

---

## License

Apache 2.0

---

*OMEN — In memory of assembly.* ⚔️
