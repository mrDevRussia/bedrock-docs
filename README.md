# BedRock Programming Language
## Master Technical Reference
### Revision 2.0

**Compiler Source:** `lexer.rs` · `parser.rs` · `ast.rs` · `codegen/mod.rs`  
**Target Architecture:** MIPS-I (Big-Endian, 32-bit)  
**Output Format:** Raw Binary (.bin) · Floppy Disk Image (.img)  
**Compiler Implementation Language:** Rust

---

## Table of Contents

1. [Core Philosophy](#1-core-philosophy)
2. [Lexical Grammar](#2-lexical-grammar)
3. [Memory Model](#3-memory-model)
4. [Detailed Syntax Guide](#4-detailed-syntax-guide)
5. [Instruction Generation](#5-instruction-generation)
6. [Hardware Interfacing](#6-hardware-interfacing)
7. [Best Practices](#7-best-practices)
8. [Edge Cases & Compiler Enforcement](#8-edge-cases--compiler-enforcement)

---

## 1. Core Philosophy

### 1.1 "No Magic"

Every BedRock construct maps to a **known, fixed, auditable sequence of MIPS-32 machine instructions**. There are no hidden runtime allocations, no garbage collector passes, no virtual dispatch tables, no implicit type coercions, and no linker scripts to manage. The programmer can predict — and verify in hex — exactly what the CPU will execute.

This guarantee is enforced at every level of the compiler:

**Variable declaration generates exactly one store:**
```bedrock
let x = 10;
```
```mips
ori  $t0, $0, 10        ; load immediate 10
lui  $t1, 0x8001        ; load address of x (upper)
ori  $t1, $t1, 0x0000   ; (lower, if non-zero)
sw   $t0, 0($t1)        ; store to static address
```
No heap allocation. No stack frame. No constructor. One `sw`.

**Bit shifts generate exactly one MIPS shift instruction:**
```bedrock
let y = x << 2;
```
```mips
sllv $dest, $left, $right   ; variable left-shift, one instruction
```

There is no intermediate representation, no IR optimization pass, and no assembly stage. The compiler walks the AST and emits `u32` MIPS words directly into a `Vec<u32>`, which is then written as big-endian bytes to disk.

### 1.2 "Byte-Level Validation"

All numbers in BedRock are `u64` at parse time, truncated to 32-bit at code generation. There are no floating-point types, no signed integers in the type system, and no implicit widening. What you specify is precisely what the hardware receives.

The output pipeline is:
```
Vec<u32>  →  .to_be_bytes()  →  flat Vec<u8>  →  written to disk
```

Every instruction is big-endian. The `.bin` output is position-independent only relative to the declared `BASE` address. The 1.44 MB floppy image (`bedrock_os.img`) places the binary at byte offset 0, sector 0.

### 1.3 "Direct Hardware Control"

BedRock provides four built-in mechanisms that map one-to-one to bare-metal memory operations:

| Construct | Hardware Operation | MIPS Instruction |
|---|---|---|
| `poke(addr, val)` | Write 32-bit word to physical address | `sw $val, 0($addr)` |
| `peek(addr)` | Read 32-bit word from physical address | `lw $dest, 0($addr)` |
| `outb(port, val)` | Memory-mapped I/O write | `sw $val, 0($port)` |
| `inb(port)` | Memory-mapped I/O read | `lw $dest, 0($port)` |
| `asm("hex")` | Emit raw machine word verbatim | Direct `u32` emission |
| `call(expr)` | Jump to function pointer (dynamic address) | `jalr $ra, $reg` |

There is no operating system. There is no HAL. BedRock programs own the hardware entirely.

---

## 2. Lexical Grammar

### 2.1 Complete Keyword Table

Every identifier in this table is reserved and cannot be used as a variable or function name.

| Keyword | Token | Category | Description |
|---|---|---|---|
| `fn` | `Fn` | Declaration | Define a function |
| `let` | `Let` | Declaration | Declare a local or static variable |
| `root` | `Root` | Declaration | Declare a hardware/environment constant |
| `return` | `Return` | Control Flow | Return from function, optionally with value |
| `if` | `If` | Control Flow | Conditional branch |
| `else` | `Else` | Control Flow | Alternative branch |
| `while` | `While` | Control Flow | Condition-tested loop |
| `loop` | `Loop` | Control Flow | Unconditional infinite loop |
| `break` | `Break` | Control Flow | Exit innermost loop |
| `asm` | `Asm` | Hardware | Emit raw MIPS instruction word |
| `outb` | `Outb` | Hardware | Memory-mapped I/O write |
| `inb` | `Inb` | Hardware | Memory-mapped I/O read (expression) |
| `poke` | `Poke` | Hardware | Direct memory write (statement) |
| `peek` | `Peek` | Hardware | Direct memory read (expression) |
| `in` | — | Hardware | Read from keyboard buffer (alias: WaitKey) |
| `call` | `Call` | Hardware | Call function via pointer (dynamic dispatch) |
| `include` | `Include` | Preprocessor | Textual file inclusion (pre-lex phase) |

### 2.2 Operator Reference

#### Arithmetic Operators

| Symbol | Token | Precedence Level | MIPS Instruction |
|---|---|---|---|
| `*` | `Star` | Highest (parse_factor) | `mult` + `mflo` |
| `/` | `Slash` | Highest (parse_factor) | `div` + `mflo` |
| `<<` | `ShiftLeft` | Highest (parse_factor) | `sllv` (variable shift) |
| `>>` | `ShiftRight` | Highest (parse_factor) | `srlv` (variable shift) |
| `+` | `Plus` | Middle (parse_term) | `addu` |
| `-` | `Minus` | Middle (parse_term) | `subu` |

#### Bitwise Operators

| Symbol | Token | Precedence Level | MIPS Instruction |
|---|---|---|---|
| `&` | `Ampersand` | Middle (parse_term) | `and` |
| `\|` | `Pipe` | Middle (parse_term) | `or` |
| `^` | `Caret` | Middle (parse_term) | `xor` |

#### Comparison Operators

| Symbol | Token | Precedence Level | MIPS Sequence |
|---|---|---|---|
| `==` | `EqEq` | Lowest (parse_expression) | `xor` + `sltiu $d,$d,1` |
| `!=` | `NotEq` | Lowest (parse_expression) | `xor` + `sltu $d,$0,$d` |
| `>` | `Greater` | Lowest (parse_expression) | `slt` (operands swapped) |
| `<` | `Less` | Lowest (parse_expression) | `slt` |
| `>=` | `GreaterEq` | Lowest (parse_expression) | `slt` + `xori $d,$d,1` |
| `<=` | `LessEq` | Lowest (parse_expression) | `slt` (swapped) + `xori $d,$d,1` |

#### Assignment Operator

| Symbol | Token | Description |
|---|---|---|
| `=` | `Equal` | Assignment (not comparison). Distinct from `==`. |

### 2.3 Expression Precedence — Three-Level Hierarchy

The parser enforces a strict three-level precedence chain. **Higher level = binds more tightly.**

```
Level 3 (parse_factor)   — *, /, <<, >>         ← Tightest binding
Level 2 (parse_term)     — +, -, &, |, ^         ← Middle binding  
Level 1 (parse_expression) — ==, !=, <, >, <=, >= ← Loosest binding
```

```bedrock
// Example: what does this evaluate as?
let r = a + b << 2 == c & 0xFF;

// Parsed as:
let r = ((a + (b << 2)) == (c & 0xFF));
//           ^^^^                        Level 3: << first
//       ^^^^^^^^^^^                     Level 2: + and & next
//       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Level 1: == last
```

**Always use explicit parentheses when mixing levels to document intent.**

### 2.4 Delimiters and Punctuation

| Symbol | Token | Usage |
|---|---|---|
| `(` | `LParen` | Expression grouping, function call, control flow condition |
| `)` | `RParen` | Close paren |
| `{` | `LBrace` | Open block body |
| `}` | `RBrace` | Close block body |
| `[` | `LBracket` | Array literal open / array index open |
| `]` | `RBracket` | Array literal close / array index close |
| `;` | `SemiColon` | Statement terminator (mandatory) |
| `,` | `Comma` | Argument separator, array element separator |

### 2.5 Literal Types

#### Integer Literals

All integers are parsed as `u64`. Decimal and hexadecimal are supported.

```bedrock
let a = 255;           // Decimal
let b = 0xFF;          // Hexadecimal (0x prefix, case-insensitive)
let c = 0xBFC00000;    // Large hex address
let d = 0x80000000;    // MIPS kseg0 base
```

#### String Literals

Delimited by double quotes. Stored as null-terminated byte arrays in the DATA segment. **Escape sequences are not processed** — `\n` is stored as two bytes (`\` and `n`).

```bedrock
let msg = "BedRock v2.0";
```

#### Array Literals

Comma-separated integer literals in `[ ]`. **Only literal integers are permitted** — variables and expressions are not allowed in array initializers.

```bedrock
let lut = [0x00, 0x41, 0x07, 0xFF];
let row = [0x0720, 0x0720, 0x0720];
```

### 2.6 Comments

```bedrock
// Single-line comment — discarded by lexer

/* Multi-line comment
   spans multiple lines
   discarded by lexer */
```

### 2.7 Include Directive

Handled in a pre-lexing text substitution pass, before tokenization.

```bedrock
include "hardware.br";
include "drivers/uart.br";
```

Paths are resolved relative to the directory of the file being compiled. Includes are processed recursively.

---

## 3. Memory Model

### 3.1 Flat 32-Bit Physical Address Space

BedRock targets a flat, unprotected, 32-bit physical address space. There is no virtual memory, no paging, and no memory protection. The programmer has full read/write access to any 32-bit address.

### 3.2 Segment Layout via `root` Declarations

Three reserved `root` names control the compiler's segment placement:

| Root Name | Default Value | Controls |
|---|---|---|
| `BASE` | `0x80000000` | Start address of the code segment |
| `DATA` | `BASE + 0x10000` | Start address of the static data segment |
| `STACK` | `BASE + 0x20000` | Initial value loaded into `$sp` |

```bedrock
root BASE  = 0x80000000;
root DATA  = 0x80010000;
root STACK = 0x80020000;
```

If any of these are omitted, the defaults above are used. **Always declare them explicitly in production code.**

### 3.3 Full Memory Map at Runtime

```
┌──────────────────────────────────────┐  ← BASE (e.g. 0x80000000)
│  BOOTLOADER (3 words)                │
│  NOP                                 │  word 0
│  li $sp, STACK                       │  words 1-2 (lui + ori)
│  J <main_start>  [patched]           │  word 3
│  NOP (delay slot)                    │  word 4
├──────────────────────────────────────┤
│  FUNCTION BODIES                     │  Compiled before main code
│  fn foo() { ... }                    │
│  fn bar() { ... }                    │
├──────────────────────────────────────┤
│  MAIN CODE ENTRY  ← J target above   │
│  Static data init (sw instructions)  │
│  Global statements                   │
├──────────────────────────────────────┤
│  HALT: J <self>                      │  Infinite loop — program end
│  NOP (delay slot)                    │
├──────────────────────────────────────┤
│           (gap)                      │
├──────────────────────────────────────┤  ← DATA (e.g. 0x80010000)
│  STATIC DATA SEGMENT                 │
│  let variables  (4 bytes each)       │
│  let arrays     (4 bytes × N)        │
│  let strings    (N bytes, 4-aligned) │
└──────────────────────────────────────┘  ← STACK (grows downward)
```

### 3.4 `let` — Static and Local Variables

**In global scope:** The compiler allocates a fixed 4-byte slot in the DATA segment and emits a `sw` instruction to initialize it. The address is resolved entirely at compile time.

```bedrock
let counter = 0;     // → address DATA+0,  sw 0 to that address
let limit   = 100;   // → address DATA+4,  sw 100 to that address
```

**In function scope:** The compiler allocates a slot on the current stack frame and emits a `sw $val, offset($sp)` instruction.

```bedrock
fn compute() {
    let result = 0;    // → sw $t0, 4($sp)  (local_offset=4)
    let temp   = 5;    // → sw $t1, 8($sp)  (local_offset=8)
    return result;
}
```

### 3.5 `root` — Hardware Address Constants

`root` declares a named compile-time constant. When a `root` name appears in any expression, the codegen emits a `li` (load immediate) instruction that loads the raw address value into a register. **No memory dereference is performed.**

```bedrock
root VGA   = 0xB8000;
root STACK = 0x80020000;
root BASE  = 0x80000000;
```

When `VGA` appears in code:
```mips
lui $t0, 0x000B       ; upper 16 bits of 0xB8000
ori $t0, $t0, 0x8000  ; lower 16 bits
; $t0 now holds 0x000B8000 — the address itself, not what is stored there
```

**Critical distinction:**

| Declaration | Expression behavior | Typical use |
|---|---|---|
| `let x = 5` | Loads value 5 from a DATA address | Mutable variables |
| `root ADDR = 0xB8000` | Loads the literal 0xB8000 as a value | Hardware addresses, constants |

### 3.6 Stack Frame Structure

Every function that is called creates a **fixed 32-byte stack frame**:

```
High address
┌────────────────────────────────┐
│  arg N-1  (from caller)        │  $sp + 32 + (N-1)*4
│  ...                           │
│  arg 0    (from caller)        │  $sp + 32
├────────────────────────────────┤  ← $sp at function entry (after prologue)
│  [reserved]                    │  $sp + 24
│  [reserved]                    │  $sp + 20
│  [reserved]                    │  $sp + 16
│  [reserved]                    │  $sp + 12
│  [reserved]                    │  $sp + 8
│  [reserved]                    │  $sp + 4
│  saved $ra                     │  $sp + 28   ← always saved here
└────────────────────────────────┘  ← $sp (bottom of frame)
Low address (stack grows down)
```

**Prologue** (emitted automatically at every `fn`):
```mips
addiu $sp, $sp, -32    ; 0x27BDFFE0 — allocate 32-byte frame
sw    $ra, 28($sp)     ; 0xAFBF001C — save return address
```

**Epilogue** (emitted automatically before `jr $ra`):
```mips
lw    $ra, 28($sp)     ; 0x8FBF001C — restore return address
addiu $sp, $sp,  32    ; 0x27BD0020 — deallocate frame
jr    $ra              ; 0x03E00008 — return to caller
nop                    ; 0x00000000 — branch delay slot
```

### 3.7 Register Usage Convention

| Register | MIPS Number | Role in BedRock |
|---|---|---|
| `$t0`–`$t7` | 8–15 | General-purpose expression temporaries (register pool) |
| `$v0` | 2 | Function return value |
| `$sp` | 29 | Stack pointer — managed by prologue/epilogue |
| `$ra` | 31 | Return address — saved/restored by every function |

---

## 4. Detailed Syntax Guide

### 4.1 Variable Declarations

#### Simple Variable

```bedrock
let name = expression;
```

```bedrock
let x      = 10;
let addr   = 0x80010000;
let result = x + 5;
let mask   = (x & 0xFF) | 0x100;
```

#### Array Definition

Statically allocated in the DATA segment. Element values must be integer literals.

```bedrock
let name = [val0, val1, val2, ...];
```

```bedrock
let scores   = [100, 200, 150, 75];
let palette  = [0xFF0000, 0x00FF00, 0x0000FF];
let vga_row  = [0x0720, 0x0720, 0x0720, 0x0720];
```

**Array element read** — index is any valid expression:

```bedrock
let v = scores[0];         // Read element at index 0
let v = scores[i];         // Read element at index i
let v = scores[i + 1];     // Read element at index (i+1)
```

**Array element write:**

```bedrock
scores[0]   = 999;
scores[i]   = scores[i] + 1;
scores[i*2] = 0;
```

The compiler scales all array indices by 4 (`sll idx, idx, 2`) automatically. Each element occupies 4 bytes (one 32-bit word).

#### String Definition

```bedrock
let name = "string content";
```

```bedrock
let greeting = "Hello, BedRock!";
let prompt   = "Enter command: ";
```

Stored in DATA segment as null-terminated bytes, padded to 4-byte alignment. String indexing is not directly supported in syntax — use `peek`/`poke` with byte-level arithmetic for character access.

#### Hardware Constant

```bedrock
root NAME = literal_value;
```

```bedrock
root BASE    = 0x80000000;
root DATA    = 0x80010000;
root STACK   = 0x80020000;
root VGA     = 0xB8000;
root UART_TX = 0x3F8;
```

`root` uses literal numbers only. Expressions on the right-hand side of `root` are not evaluated — the value must be a plain integer literal.

### 4.2 Assignment

```bedrock
name = expression;
name[index] = expression;
```

```bedrock
x       = x + 1;
counter = 0;
buf[i]  = ch;
```

### 4.3 Arithmetic and Expressions

```bedrock
// Arithmetic
let sum  = a + b;
let diff = a - b;
let prod = a * b;
let quot = a / b;

// Bit shifts (NEW in v2 — native MIPS sllv/srlv)
let hi   = val >> 8;          // Logical right shift by 8
let lo   = val & 0xFF;
let word = (hi << 8) | lo;    // Reconstruct

// Bitwise
let anded = a & 0xFF;
let ored  = a | 0x80;
let xored = a ^ 0xFF;

// Comparison (result: 1 = true, 0 = false)
let eq  = (a == b);
let neq = (a != b);
let gt  = (a > b);
let lt  = (a < b);
let gte = (a >= b);
let lte = (a <= b);

// Grouped with parentheses
let r = (a + b) * (c - d);
let m = ((val >> 4) & 0x0F) | ((val & 0x0F) << 4);
```

### 4.4 Function Definitions

```bedrock
fn name(param0, param1, ...) {
    // body
}
```

```bedrock
fn add(a, b) {
    return a + b;
}

fn vga_put(col, row, char_attr) {
    let offset = row * 160 + col * 2;
    poke(VGA + offset, char_attr);
}

fn strlen(ptr) {
    let i = 0;
    while (peek(ptr + i) != 0) {
        i = i + 1;
    }
    return i;
}
```

**Rules:**
- All parameters are 32-bit values passed on the stack.
- Functions must be defined at top level — no nested function definitions.
- Recursion is permitted. There is no stack overflow guard.
- Functions are compiled before main code. Forward calls within the same file are supported.

#### Function Calls as Statements

```bedrock
function_name(arg0, arg1, ...);
```

```bedrock
vga_put(0, 0, 0x0741);
clear_screen();
uart_send(0x3F8, data);
```

#### Function Calls as Expressions (v2 — new)

In v2, function calls can appear inside expressions. The return value from `$v0` is captured and used as the expression result.

```bedrock
let len  = strlen(buf);
let val  = read_port(0x3F8);
let sum  = add(x, y) + add(a, b);
let flag = (get_status() & 0x01) == 1;
```

**Calling convention for expression calls:**
1. Arguments evaluated into temporary registers
2. `addiu $sp, -space` (if args present)
3. `sw` each arg to `0($sp)`, `4($sp)`, ...
4. `jal <function>`
5. `addu $dest, $0, $v0` — copy return value to destination register
6. `addiu $sp, +space` (stack cleanup)

#### Return Statement

```bedrock
return;              // Void return — $v0 unmodified
return expression;   // Return value placed in $v0 (register 2)
```

```bedrock
fn is_digit(ch) {
    if (ch >= 48) {
        if (ch <= 57) {
            return 1;
        }
    }
    return 0;
}
```

The compiler jumps to the function epilogue (which restores `$ra` and `$sp` before `jr $ra`).

### 4.5 Dynamic Function Pointer Calls — `call(expr)` (v2 — new)

`call` invokes a function at a runtime-computed address. This is the mechanism for dynamic dispatch, bootloader handoff, and calling relocated code.

```bedrock
call(expression);
```

```bedrock
root PAYLOAD_ADDR = 0x80001000;

// Jump into relocated payload
call(PAYLOAD_ADDR);

// Call via pointer stored in a variable
let handler = get_isr_address();
call(handler);

// Call with computed address
call(BASE + 0x100);
```

**Machine code generated:**
```mips
; Evaluate expression → $t0
lui  $t0, upper
ori  $t0, $t0, lower

; JALR: jump to address in $t0, save return address in $ra (reg 31)
; Encoding: 0x00000009 | (reg << 21) | (31 << 11)
jalr $ra, $t0         ; 0x01200009 (if $t0 = reg 8)
nop                   ; delay slot
```

`jalr` saves the return address in `$ra` and jumps to the target address in the register. This is the only instruction in BedRock that performs a fully dynamic indirect call.

### 4.6 Control Flow

#### `if` / `else`

```bedrock
if (condition) {
    // then body
}

if (condition) {
    // then body
} else {
    // else body
}
```

```bedrock
if (x == 0) {
    poke(VGA, 0x0720);
} else {
    poke(VGA, 0x0741);
}

if ((status & 0x01) != 0) {
    handle_ready();
}
```

#### `while`

```bedrock
while (condition) {
    // body
}
```

```bedrock
let i = 0;
while (i < 80) {
    poke(VGA + i * 2, 0x0720);
    i = i + 1;
}

// Poll until ready
while ((inb(UART_LSR) & 0x20) == 0) {
    // spin
}
```

#### `loop` (Infinite)

```bedrock
loop {
    // body — exits only via break or return
}
```

```bedrock
loop {
    let cmd = in;
    if (cmd == 0x1C) { break; }   // Enter key exits
    process(cmd);
}
```

#### `break`

Exits the immediately enclosing `loop` or `while`. The compiler emits an unconditional branch (`beq $0, $0, offset`) with the offset back-patched after the loop body is complete.

```bedrock
while (1 == 1) {
    let v = peek(STATUS);
    if (v == 0xFF) { break; }
}
```

### 4.7 Inline Assembly — `asm`

```bedrock
asm("32-bit-hex-encoding");
```

Emits a single 32-bit MIPS instruction word directly into the code stream. The argument is a hex string (no `0x` prefix) representing the exact big-endian encoding.

```bedrock
asm("00000000");   // NOP: sll $0, $0, 0
asm("0000000C");   // SYSCALL
asm("42000018");   // ERET — return from exception handler
asm("40806000");   // MTC0 $0, $12 — write CP0 Status register
asm("00084080");   // SLL $t0, $t0, 2 (fixed shift by 2)
```

Note: `<<` and `>>` in BedRock source generate **variable** shifts (`sllv`/`srlv`). For **fixed** shifts by an immediate constant, use `asm` with the appropriate `sll`/`srl` encoding.

---

## 5. Instruction Generation

### 5.1 Compiler Pipeline — Six Phases

```
┌─────────────────────────────────────────────────┐
│ Phase 0: Root Symbol Collection                 │
│   → scan for root declarations                  │
│   → resolve BASE, DATA, STACK values            │
├─────────────────────────────────────────────────┤
│ Phase 1: Bootloader Emission                    │
│   → NOP                                         │
│   → li $sp, STACK                               │
│   → J <main> [placeholder — patched in Phase 4] │
│   → NOP (delay slot)                            │
├─────────────────────────────────────────────────┤
│ Phase 2: Static Data Allocation                 │
│   → assign DATA-segment addresses to arrays     │
│   → assign DATA-segment addresses to strings    │
│   → advance next_addr pointer                   │
├─────────────────────────────────────────────────┤
│ Phase 3: Function Code Generation               │
│   → emit prologue for each fn                   │
│   → emit body statements                        │
│   → emit epilogue (lw $ra, jr $ra)              │
├─────────────────────────────────────────────────┤
│ Phase 4: Jump Patch                             │
│   → back-patch Phase 1 J to point at           │
│      the first instruction after all functions  │
├─────────────────────────────────────────────────┤
│ Phase 5: Static Data Initialization             │
│   → emit sw instructions to write array/string  │
│      data into their DATA-segment addresses     │
├─────────────────────────────────────────────────┤
│ Phase 6: Main Code Generation                   │
│   → emit all non-root, non-fn, non-static stmts │
├─────────────────────────────────────────────────┤
│ Phase 7: Halt                                   │
│   → J <self> + NOP — infinite halt loop         │
├─────────────────────────────────────────────────┤
│ Phase 8: Binary Serialization                   │
│   → Vec<u32> → to_be_bytes() → Vec<u8>          │
│   → write .bin file                             │
│   → write .img (1.44MB floppy image)            │
└─────────────────────────────────────────────────┘
```

### 5.2 Load Immediate (`emit_li`)

The compiler uses this pattern for all constant loading. It is the most frequently emitted sequence.

```
If upper 16 bits == 0:
    ori $reg, $0, lower16          ; one instruction

Else:
    lui $reg, upper16              ; load high half
    ori $reg, $reg, lower16        ; merge low half (if non-zero)
```

Examples:
```mips
; Loading 0x0041:
ori $t0, $0, 0x0041

; Loading 0xB8000:
lui  $t0, 0x000B
ori  $t0, $t0, 0x8000

; Loading 0x80000000 (BASE):
lui  $t0, 0x8000
; (lower 16 bits are 0 — no ori needed)
```

### 5.3 Arithmetic Instruction Encodings

All arithmetic operations use the three-register pattern: evaluate left, evaluate right, emit instruction.

| BedRock | MIPS Mnemonic | Hex Encoding Base | Notes |
|---|---|---|---|
| `a + b` | `addu $d, $s, $t` | `0x00000021` | Unsigned add, no overflow trap |
| `a - b` | `subu $d, $s, $t` | `0x00000023` | Unsigned sub |
| `a * b` | `mult $s, $t` + `mflo $d` | `0x00000018` + `0x00000012` | Result from LO register |
| `a / b` | `div $s, $t` + `mflo $d` | `0x0000001A` + `0x00000012` | Quotient from LO |
| `a & b` | `and $d, $s, $t` | `0x00000024` | Bitwise AND |
| `a \| b` | `or $d, $s, $t` | `0x00000025` | Bitwise OR |
| `a ^ b` | `xor $d, $s, $t` | `0x00000026` | Bitwise XOR |
| `a << b` | `sllv $d, $t, $s` | `0x00000004` | Variable left shift |
| `a >> b` | `srlv $d, $t, $s` | `0x00000006` | Variable logical right shift |

### 5.4 Comparison Instruction Sequences

| BedRock | MIPS Sequence | Result in `$dest` |
|---|---|---|
| `a == b` | `xor $d,$s,$t` → `sltiu $d,$d,1` | 1 if equal, 0 otherwise |
| `a != b` | `xor $d,$s,$t` → `sltu $d,$0,$d` | 1 if not equal |
| `a < b` | `slt $d,$s,$t` | 1 if s < t (signed) |
| `a > b` | `slt $d,$t,$s` (operands swapped) | 1 if t < s |
| `a >= b` | `slt $d,$s,$t` → `xori $d,$d,1` | Invert the less-than |
| `a <= b` | `slt $d,$t,$s` → `xori $d,$d,1` | Invert the greater-than |

### 5.5 Control Flow Instruction Patterns

#### `if` without `else`

```
    gen_expr(condition) → $cond
    beq $cond, $0, +offset       ; branch if false (condition == 0)
    nop
    ... then_body ...
<offset target>:
```

#### `if` with `else`

```
    gen_expr(condition) → $cond
    beq $cond, $0, else_start    ; branch to else if false
    nop
    ... then_body ...
    j   end                      ; jump over else
    nop
else_start:
    ... else_body ...
end:
```

#### `while`

```
start:
    gen_expr(condition) → $cond
    beq $cond, $0, after_loop    ; exit if false
    nop
    ... body ...
    j   start                    ; back to condition
    nop
after_loop:
    ; break patches resolve here
```

#### `loop`

```
start:
    ... body ...
    j   start
    nop
after_loop:
    ; break patches resolve here
```

### 5.6 Memory Operation Encodings

| BedRock | MIPS Encoding | Description |
|---|---|---|
| `poke(addr, val)` | `sw $val, 0($addr)` — `0xAC000000` | 32-bit store |
| `peek(addr)` | `lw $dest, 0($addr)` — `0x8C000000` | 32-bit load |
| `outb(port, val)` | `sw $val, 0($port)` — `0xAC000000` | MMIO write (identical to poke) |
| `inb(port)` | `lw $dest, 0($port)` — `0x8C000000` | MMIO read (identical to peek) |
| `arr[i]` (read) | `sll idx,idx,2` + `addu` + `lw` | Scaled index + base address |
| `arr[i] = v` (write) | `sll idx,idx,2` + `addu` + `sw` | Scaled index + base address |

### 5.7 `call(expr)` — Dynamic Dispatch Encoding

```
    gen_expr(target_address) → $reg
    jalr $ra, $reg           ; 0x00000009 | (reg << 21) | (31 << 11)
    nop                      ; delay slot
```

`jalr` writes `PC+8` into `$ra` (register 31) and jumps to the address in `$reg`. The target function may or may not return — if it does, execution resumes at the instruction after the `nop`.

---

## 6. Hardware Interfacing

### 6.1 `poke` — Direct Memory Write

Writes a single 32-bit word to an absolute physical address. Use for VGA frame buffer writes, hardware register configuration, and raw memory manipulation.

```bedrock
poke(address, value);
```

```bedrock
root VGA = 0xB8000;

poke(VGA,     0x0741);   // Write 'A' (0x41) white-on-black at cell 0
poke(VGA + 2, 0x0742);   // Write 'B' at cell 1
poke(0x80030000, 0x01);  // Set bit 0 at hardware register
```

Generated MIPS for `poke(VGA, 0x0741)`:
```mips
lui  $t0, 0x000B        ; VGA address upper
ori  $t0, $t0, 0x8000   ; VGA = 0x000B8000
ori  $t1, $0,  0x0741   ; value
sw   $t1, 0($t0)         ; store to VGA
```

### 6.2 `peek` — Direct Memory Read

Reads a single 32-bit word from an absolute physical address. Use for reading hardware registers, sampling memory-mapped devices, and reading relocated data.

```bedrock
let value = peek(address);
```

```bedrock
let cell    = peek(VGA);               // Read VGA cell 0
let status  = peek(UART_LSR);          // Read UART line status register
let word    = peek(0xBFC00000);        // Read from boot ROM
```

### 6.3 `outb` and `inb` — Memory-Mapped I/O

`outb` and `inb` are semantically distinct from `poke`/`peek` to signal I/O intent, but they generate identical MIPS `sw`/`lw` instructions. BedRock implements I/O exclusively as MMIO — there are no x86-style port I/O instructions in MIPS.

```bedrock
outb(port_address, value);
let v = inb(port_address);
```

```bedrock
root UART_DATA = 0xBF000900;   // TC3162 UART data register
root UART_LSR  = 0xBF000914;   // Line status register

// Wait until transmit buffer empty (bit 5 of LSR)
while ((inb(UART_LSR) & 0x20) == 0) { }

// Send character
outb(UART_DATA, 0x41);  // Send 'A'
```

### 6.4 VGA Text Mode — Complete Reference

**Base address:** `0xB8000` (physical, standard PC VGA)  
**Mode:** 80 columns × 25 rows = 2,000 character cells  
**Cell size:** 2 bytes per cell  
**Total buffer size:** 4,000 bytes

**Cell format (2 bytes per cell, stored as one 32-bit poke):**
```
Bits [15:8]  = Attribute byte:  [7:4] background color, [3:0] foreground color
Bits  [7:0]  = ASCII character code
```

**Cell address formula:**
```bedrock
// cell_address = VGA + (row * 80 + col) * 2
// poke_value   = (attribute << 8) | ascii_code
```

**Standard color values:**

| Value | Color | Value | Color |
|---|---|---|---|
| 0x0 | Black | 0x8 | Dark Gray |
| 0x1 | Blue | 0x9 | Bright Blue |
| 0x2 | Green | 0xA | Bright Green |
| 0x3 | Cyan | 0xB | Bright Cyan |
| 0x4 | Red | 0xC | Bright Red |
| 0x5 | Magenta | 0xD | Bright Magenta |
| 0x6 | Brown | 0xE | Yellow |
| 0x7 | Light Gray | 0xF | White |

**Examples:**

```bedrock
root VGA = 0xB8000;

// Print 'A' in white (0xF) on blue (0x1) at top-left
poke(VGA, 0x1F41);
//         ^^ attribute: bg=0x1 (blue), fg=0xF (white)
//           ^^ ASCII: 0x41 = 'A'

// Clear screen: fill all 2000 cells with space + white-on-black
fn cls() {
    let i = 0;
    while (i < 4000) {
        poke(VGA + i, 0x0720);
        i = i + 2;
    }
}

// Print character at (col, row)
fn vga_char(col, row, ch, attr) {
    let pos = row * 160 + col * 2;
    poke(VGA + pos, (attr << 8) | ch);
}
```

### 6.5 `call(expr)` for Bootloader Handoff

The canonical use case for `call` is transferring execution to a relocated payload or a freshly loaded code image:

```bedrock
root FLASH_BASE  = 0xBFC00000;   // TC3162 boot ROM
root DRAM_TARGET = 0x80001000;   // Relocation destination in DRAM

fn relocate(src, dst, size) {
    let i = 0;
    while (i < size) {
        let word = peek(src + i);
        poke(dst + i, word);
        i = i + 4;
    }
}

fn main() {
    relocate(FLASH_BASE, DRAM_TARGET, 0x400);  // Copy 1KB
    call(DRAM_TARGET);                          // Jump into relocated code
}
```

Generated for `call(DRAM_TARGET)`:
```mips
lui  $t0, 0x8000       ; upper of 0x80001000
ori  $t0, $t0, 0x1000  ; DRAM_TARGET = 0x80001000
jalr $ra, $t0           ; jump, save return address in $ra
nop                     ; delay slot
```

### 6.6 `in` — Keyboard Polling

The `in` keyword reads from a hardcoded keyboard buffer address (`0x80020000`):

```bedrock
let key = in;
```

```mips
lui  $dest, 0x8002     ; 0x80020000 upper
lw   $dest, 0($dest)   ; read from keyboard buffer
```

For systems where the keyboard is mapped to a different address, use `inb(port)` with the explicit address instead.

---

## 7. Best Practices

### 7.1 The Address Library — Foundation of Clean BedRock Code

All hardware addresses must be declared as `root` constants, ideally in a dedicated `hardware.br` include file. This is the most important organizational practice in BedRock.

**Avoid (magic numbers everywhere):**
```bedrock
poke(0xB8000, 0x0741);
outb(0x3F8, 0x80);
while ((peek(0xBF000914) & 0x20) == 0) { }
```

**Correct (address library):**
```bedrock
// hardware.br
root VGA       = 0xB8000;
root UART_DATA = 0xBF000900;
root UART_LSR  = 0xBF000914;
root PIC_CMD   = 0x20;
root PIT_CH0   = 0x40;
root KBD_DATA  = 0x60;
root FLASH     = 0xBFC00000;
```

```bedrock
include "hardware.br";

poke(VGA, 0x0741);
outb(UART_DATA, 0x80);
while ((inb(UART_LSR) & 0x20) == 0) { }
```

### 7.2 Use Bit Shifts for Packed Data

With native `<<` and `>>` operators in v2, packed hardware register manipulation is cleaner:

```bedrock
// Pack a VGA cell value from components
fn make_cell(ascii, fg, bg) {
    let attr = (bg << 4) | fg;
    return (attr << 8) | ascii;
}

// Extract byte from word
fn hi_byte(word) { return word >> 8; }
fn lo_byte(word) { return word & 0xFF; }

// Build a 32-bit word from two 16-bit halves
fn make_word(hi, lo) { return (hi << 16) | lo; }
```

### 7.3 Break Complex Expressions into Steps

The register pool holds only 8 temporaries (`$t0`–`$t7`). Deeply nested expressions consume registers faster than they are freed. Break complex computations into named intermediate variables:

```bedrock
// Risky — may exhaust register pool
let v = ((a + b) * c - d / e) | ((f & g) ^ (h >> 2));

// Safe — each step uses at most 2-3 registers at a time
let ab   = a + b;
let abc  = ab * c;
let de   = d / e;
let left = abc - de;
let fg   = f & g;
let h2   = h >> 2;
let right = fg ^ h2;
let v    = left | right;
```

### 7.4 Always Declare `BASE`, `DATA`, `STACK` First

Never rely on compiler defaults. Declare the memory layout explicitly at the top of every program:

```bedrock
// main.br — always starts with this block
root BASE  = 0x80000000;
root DATA  = 0x80010000;
root STACK = 0x80020000;

include "hardware.br";

fn main() {
    // ...
}
```

### 7.5 Document `asm` Calls

Inline assembly is the escape hatch for instructions BedRock does not expose natively. Always annotate what instruction you are emitting and why:

```bedrock
// Flush pipeline before jumping to relocated code
asm("00000000");   // NOP — pipeline flush

// Enable interrupts: MTC0 $0, Status
asm("40806000");   // mtc0 $0, $12

// Fixed shift left by 4 (multiply by 16): sll $t0, $t0, 4
asm("000840C0");

// Exception return
asm("42000018");   // eret
```

### 7.6 Use `call()` Only for Dynamic Targets

`call(expr)` generates `jalr`, which is an indirect branch. Use it only when the target address is truly dynamic (computed at runtime). For all statically-known function calls, use `function_name()` syntax, which generates a direct `jal` instruction that is more predictable and patchable.

```bedrock
// Correct use of call — target is runtime-determined
let handler = get_vector(irq_number);
call(handler);

// Incorrect use of call — target is known at compile time
call(my_function);   // Use my_function(); instead
```

---

## 8. Edge Cases & Compiler Enforcement

### 8.1 `root` Accepts Only Literal Numbers

The right-hand side of a `root` declaration must be a plain integer literal. Expressions, variables, and arithmetic are not permitted. The parser calls `parse_expression()` for `root` values, but the codegen only processes `Expression::Number` variants — any other expression will be silently ignored or produce incorrect behavior.

```bedrock
root A = 0x80000000;         // ✅ Correct
root B = 0x80000000 + 4;     // ❌ Expression — not supported for root
root C = A;                  // ❌ Variable — not supported for root
```

### 8.2 Array Literals Require Integer Literals Only

Array element values in a `let arr = [...]` declaration must be `Token::Number` tokens. Variables, expressions, and function calls will cause the parser to panic.

```bedrock
let arr = [1, 2, 3];          // ✅ Correct
let arr = [0xFF, 0x00, 0x41]; // ✅ Correct
let arr = [x, y, z];          // ❌ Parser panic — expects Token::Number
let arr = [a + 1, b + 2];     // ❌ Parser panic
```

### 8.3 Undefined Variables Panic at Codegen

Referencing a variable that was never declared causes an immediate `panic!` in the codegen phase (`.expect("Variable not defined!")`). There is no graceful error; compilation aborts.

```bedrock
let total = x + y;   // ❌ Panics if 'x' or 'y' were never declared
```

### 8.4 Undefined Functions Panic at Codegen

Calling a function that does not exist in `self.functions` causes a `panic!`:
```
panic!("Function '{}' not defined before use!", name);
```

This applies to expression-position calls (`Expression::Call`). Function calls in statement position (`Statement::Call`) silently emit no code if the function is not found.

### 8.5 `<<` and `>>` Generate Variable Shifts (sllv / srlv)

The shift operators always use the `sllv`/`srlv` MIPS instructions, which take the shift amount from a **register**, not an immediate. The right operand is always evaluated into a register first.

```bedrock
let y = x << 2;   // Generates sllv — shift amount 2 loaded into register first
```

For a **fixed immediate shift** (which is one instruction in MIPS, `sll $d, $t, sa`), use `asm` with the pre-encoded instruction:

```bedrock
// Fixed SLL $t0, $t0, 2 (shift by immediate 2)
asm("00084080");
```

### 8.6 No `!` (Logical NOT) Operator

A bare `!` is a lexer error — it returns `Token::EOF` and terminates lexing silently. Only `!=` is recognized.

```bedrock
if (!flag) { ... }        // ❌ Lexer error — '!' alone is invalid
if (flag == 0) { ... }    // ✅ Correct equivalent
if ((x & mask) == 0) { } // ✅ Correct pattern for bit testing
```

### 8.7 String Escape Sequences Are Not Processed

The lexer reads string content verbatim until the closing `"`. Backslash sequences are stored as two characters.

```bedrock
let s = "Line1\nLine2";
// Stored as: L i n e 1 \ n L i n e 2 \0
// '\n' is the backslash character followed by 'n', NOT a newline byte (0x0A)
```

To embed a newline byte in data, use an array:
```bedrock
let newline = [0x0A];   // Actual newline byte
```

### 8.8 Register Pool Exhaustion — Silent Corruption

The pool contains exactly 8 registers (`$t0`–`$t7`). When it empties, `alloc_reg()` returns `8` (`$t0`) silently. Any expression nesting deeper than 8 levels without intermediate `let` bindings risks register collision and silent incorrect results.

**Symptom:** Arithmetic produces wrong values in deeply nested expressions with no compiler warning.

**Prevention:** Keep expression depth under 5 levels. Break into intermediate `let` bindings (see Best Practices §7.3).

### 8.9 No Argument Count Validation

The parser does not check that the number of arguments at a call site matches the number of parameters in the function definition.

```bedrock
fn add(a, b) { return a + b; }

add(1, 2);       // ✅ Correct
add(1);          // ⚠️ Compiles — 'b' reads garbage from stack
add(1, 2, 3);    // ⚠️ Compiles — third arg is pushed but never read
```

### 8.10 Division and Modulo Behavior

BedRock emits raw MIPS `div` and `mult` instructions. Division by zero does not trap — the result is undefined per MIPS architecture. There is no modulo operator; use `asm` with the `mfhi` instruction to retrieve the remainder after a division.

```bedrock
// Remainder after division: use asm to retrieve HI register
// After: a / b is computed (which also sets HI = a % b)
let q = a / b;
// Then: mfhi $t0 to get the remainder
asm("00004010");  // mfhi $t0 (reg 8) — encoding: 0x00004010
```

### 8.11 `call(expr)` Does Not Set Up a Stack Frame

`call(expr)` emits only `jalr $ra, $reg` + `nop`. It does **not** emit a prologue. If the target function is a BedRock-compiled function, it will emit its own prologue. If the target is external code (e.g., a relocated payload or BIOS routine), no frame is prepared — the caller is responsible for any required setup.

### 8.12 Maximum Nesting Depth for `break`

`break` patches the innermost loop on the `loop_stack`. The stack is unbounded, so nested loops of any depth are supported. However, `break` inside a function body that was called from within a loop **does not** break the outer loop — it is scoped to the innermost loop in the current call frame only.

---

*End of BedRock Master Technical Reference — Revision 2.0*  
*Compiler analyzed: lexer.rs · parser.rs · ast.rs · codegen/mod.rs*
