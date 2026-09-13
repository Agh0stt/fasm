# fasm

A minimal x86-32 assembler written in [Falcon](https://github.com/Agh0stt/falcon).
Reads AT&T-syntax `.fasm` source files and produces statically-linked ELF32 executables
directly — no `as`, no `ld`, no libc.

---

## Building

You need the Falcon compiler and runtime from the main repo.

```bash
# 1. build the falcon compiler
gcc -O2 -o falconc falconc.c

# 2. assemble the runtime (32-bit)
as --32 flr.s -o flr.o

# 3. compile fasm itself
./falconc fasm.fl -o fasm.s

# 4. assemble and link
as --32 fasm.s -o fasm.o
ld -m elf_i386 --allow-multiple-definition flr.o fasm.o -o fasm
```

---

## Usage

```bash
./fasm <input.fasm> <output>
```

The output is a ready-to-run ELF32 executable. No extra steps.

```bash
./fasm hello.fasm hello
./hello
# hello world!
```

---

## Source format

AT&T syntax, one instruction per line. Comments start with `#`.

### Sections

```asm
section text    # executable code
section data    # initialised data
section bss     # zero-filled / reserved space
```

### Labels

```asm
my_label:
    mov $1, %eax
```

A bare label name used as an operand resolves to its absolute address:

```asm
mov msg, %ecx       # ecx = address of msg  ✓
mov (msg), %ecx     # ecx = 4-byte value AT msg (memory load)
```

### Data directives

| Directive | Width | Example |
|-----------|-------|---------|
| `db` | 1 byte | `db 0x41, "hello\n"` |
| `dw` | 2 bytes | `dw 1234` |
| `dd` | 4 bytes | `dd 0xdeadbeef` |
| `dq` | 8 bytes | `dq 0` |

String literals inside `db` are emitted byte-by-byte. Escape sequences: `\n \t \0 \\ \"`.

### BSS directives

```asm
buf:  resb 64    # reserve 64 bytes
```

| Directive | Reserves |
|-----------|----------|
| `resb N` | N bytes |
| `resw N` | N × 2 bytes |
| `resd N` | N × 4 bytes |
| `resq N` | N × 8 bytes |

### Registers

All 8 general-purpose 32-bit registers: `%eax %ecx %edx %ebx %esp %ebp %esi %edi`.  
`%esp` cannot be used as a memory base (no SIB support yet).

### Memory operands

```asm
(%eax)          # [eax]
4(%ebx)         # [ebx + 4]
-8(%ebp)        # [ebp - 8]
(label)         # [label]  — absolute address
```

### Immediates

```asm
$42             # decimal
$0xff           # hex
```

---

## Supported instructions

### Data movement
| Instruction | Forms |
|-------------|-------|
| `mov` | reg→reg, imm→reg, label→reg, mem→reg, reg→mem, imm→mem |
| `lea` | mem→reg |
| `push` | reg, imm |
| `pop` | reg |

### Arithmetic
| Instruction | Description |
|-------------|-------------|
| `add` | reg+reg, imm+reg |
| `sub` | reg-reg, imm-reg |
| `imul` | `EDX:EAX = EAX * src` (one-operand form) |
| `idiv` | `EAX = EDX:EAX / src`, `EDX = remainder` (one-operand form) |
| `cdq` | sign-extend `EAX` into `EDX:EAX` (required before `idiv`) |
| `inc` | increment register |
| `dec` | decrement register |
| `neg` | two's complement negate |

### Bitwise / shift
| Instruction | Forms |
|-------------|-------|
| `and` | reg&reg, imm&reg |
| `or` | reg\|reg, imm\|reg |
| `xor` | reg^reg |
| `not` | bitwise NOT |
| `shl` | `$imm8, %reg` or `%cl, %reg` |
| `shr` | `$imm8, %reg` or `%cl, %reg` |

### Comparison / control flow
| Instruction | Description |
|-------------|-------------|
| `cmp` | set flags (reg-reg, imm-reg) |
| `jmp` | unconditional jump to label |
| `call` | call label (pushes return address) |
| `ret` | return |
| `int` | software interrupt (e.g. `int $0x80`) |
| `nop` | no operation |

### Conditional jumps
`je`/`jz`, `jne`/`jnz`, `jl`, `jle`, `jg`, `jge`, `jb`/`jc`, `jbe`, `ja`, `jae`/`jnc`

All conditional jumps encode as `0F 8x rel32` (6 bytes, always 32-bit displacement).

---

## Hello world example

```asm
# hello.fasm — write "hello world!\n" to stdout and exit

section text
global _start

_start:
    mov $4,   %eax      # sys_write
    mov $1,   %ebx      # fd = stdout
    mov msg,  %ecx      # buf = address of msg
    mov $13,  %edx      # len
    int $0x80

    mov $1,   %eax      # sys_exit
    xor %ebx, %ebx      # code = 0
    int $0x80

section data
msg:
    db "hello world!\n"
```

```bash
./fasm hello.fasm hello && ./hello
# hello world!
```

---

## Division example

`idiv` always operates on `EDX:EAX`. Use `cdq` first to sign-extend `EAX` into `EDX`.

```asm
# divide eax by ecx → quotient in eax, remainder in edx
mov $100, %eax
mov $7,   %ecx
cdq             # sign-extend eax into edx:eax
idiv %ecx       # eax = 14, edx = 2
```

---

## Two-pass assembly

fasm does two passes over the instruction list:

1. **Pass 1** — walk all items, compute instruction sizes, record every label's absolute address into the symbol table.
2. **Pass 2** — emit real machine code; `jmp`/`call`/`jcc` compute `rel32 = target - (here + instr_size)`.

Label sizes are fixed regardless of the resolved address, so pass 1 is always correct without iteration.

---

## Output format

The output is a minimal ELF32 executable with a single `PT_LOAD` segment:

```
[ELF header 52 B][Program header 32 B][.text][.data]
```

`.bss` is handled via `memsz > filesz` — no bytes in the file, zeroed by the kernel on load.

Load address: `0x08048000` (standard i386 Linux base).

---

## Limitations

- No macro or `%define` support
- No `%esp` as memory base (SIB byte not implemented)
- `imul`/`idiv` one-operand form only (two/three-operand forms not supported)
- `shl`/`shr` shift count is `$imm8` or `%cl` only
- No `jmp *%eax` indirect jumps
- Single source file only (no `include`)
