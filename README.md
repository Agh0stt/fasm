# fasm

A minimal x86-32 assembler written in Falcon (https://github.com/Agh0stt/falcon).
Reads AT&T-syntax .fasm source files and produces statically-linked ELF32 executables
directly — no as, no ld, no libc.

---

## Building

You need the Falcon compiler and runtime from the main repo.

    # 1. build the falcon compiler
    gcc -O2 -o falconc falconc.c

    # 2. assemble the runtime (32-bit)
    as --32 flr.s -o flr.o

    # 3. compile fasm itself
    ./falconc fasm.fl -o fasm.s

    # 4. assemble and link
    as --32 fasm.s -o fasm.o
    ld -m elf_i386 --allow-multiple-definition flr.o fasm.o -o fasm

---

## Usage

    ./fasm <input.fasm> <output>

The output is a ready-to-run ELF32 executable. No extra steps.

    ./fasm hello.fasm hello
    ./hello
    hello world!

---

## Source format

AT&T syntax, one instruction per line. Comments start with #.

Sections:

    section text    # executable code
    section data    # initialised data
    section bss     # zero-filled / reserved space

Labels:

    my_label:
        mov $1, %eax

A bare label name used as an operand resolves to its absolute address:

    mov msg, %ecx       # ecx = address of msg
    mov (msg), %ecx     # ecx = 4-byte value AT msg (memory load)

Data directives:

    db   1 byte     db 0x41, "hello\n"
    dw   2 bytes    dw 1234
    dd   4 bytes    dd 0xdeadbeef
    dq   8 bytes    dq 0

String literals inside db are emitted byte-by-byte.
Escape sequences: \n \t \0 \\ \"

BSS directives:

    buf:  resb 64    # reserve 64 bytes

    resb N    N bytes
    resw N    N x 2 bytes
    resd N    N x 4 bytes
    resq N    N x 8 bytes

Registers: %eax %ecx %edx %ebx %esp %ebp %esi %edi
%esp cannot be used as a memory base (no SIB support yet).

Memory operands:

    (%eax)      [eax]
    4(%ebx)     [ebx + 4]
    -8(%ebp)    [ebp - 8]
    (label)     absolute address

Immediates:

    $42         decimal
    $0xff       hex

---

## Preprocessor

fasm has a full source-level preprocessor that runs before the lexer.

### %define

Simple token substitution. Every later occurrence of the name as a whole word
is replaced with the value. Definitions chain — if B is defined as A and A is
defined as 4, then B expands to 4.

    %define SYS_WRITE  4
    %define STDOUT     1
    %define WRITE      SYS_WRITE   # chains: WRITE -> SYS_WRITE -> 4

    mov $WRITE,  %eax
    mov $STDOUT, %ebx

Substitution is skipped inside string literals and # comments.

### %undef

Removes a previously defined name.

    %define FOO 99
    %undef  FOO
    %define FOO 4   # FOO is now 4

### %macro / %endmacro

Parameterised macros. Reference parameters with \name inside the body.
Defines are expanded inside macro bodies. Macros can be called multiple
times and redefined.

    %macro write(fd, buf, len)
        mov $4,   %eax
        mov \fd,  %ebx
        mov \buf, %ecx
        mov \len, %edx
        int $0x80
    %endmacro

    write($1, msg, $13)

Zero-parameter macros work too:

    %macro exit0()
        mov $1, %eax
        xor %ebx, %ebx
        int $0x80
    %endmacro

    exit0()

### %include

Splices another file into the current source at this point. The included file
is itself preprocessed — defines and macros carry over. Paths starting with /
are absolute; anything else is resolved relative to the including file's directory.

    %include "syscall.fasm"
    %include "/usr/local/fasm/macros.fasm"

---

## Supported instructions

Data movement:
    mov     reg->reg, imm->reg, label->reg, mem->reg, reg->mem, imm->mem
    lea     mem->reg
    push    reg, imm
    pop     reg

Arithmetic:
    add     reg+reg, imm+reg
    sub     reg-reg, imm-reg
    imul    EDX:EAX = EAX * src  (one-operand form)
    idiv    EAX = EDX:EAX / src, EDX = remainder  (one-operand form)
    cdq     sign-extend EAX into EDX:EAX (required before idiv)
    inc     increment register
    dec     decrement register
    neg     two's complement negate

Bitwise / shift:
    and     reg&reg, imm&reg
    or      reg|reg, imm|reg
    xor     reg^reg
    not     bitwise NOT
    shl     $imm8, %reg  or  %cl, %reg
    shr     $imm8, %reg  or  %cl, %reg

Comparison / control flow:
    cmp     set flags (reg-reg, imm-reg)
    jmp     unconditional jump to label
    call    call label (pushes return address)
    ret     return
    int     software interrupt (e.g. int $0x80)
    nop     no operation

Conditional jumps:
    je/jz  jne/jnz  jl  jle  jg  jge  jb/jc  jbe  ja  jae/jnc

All conditional jumps encode as 0F 8x rel32 (6 bytes, 32-bit displacement).

---

## Hello world

    %define SYS_WRITE  4
    %define SYS_EXIT   1
    %define STDOUT     1

    section text
    global _start

    _start:
        mov $SYS_WRITE, %eax
        mov $STDOUT,    %ebx
        mov msg,        %ecx
        mov $13,        %edx
        int $0x80

        mov $SYS_EXIT,  %eax
        xor %ebx, %ebx
        int $0x80

    section data
    msg:
        db "hello world!\n"

---

## Division

idiv always operates on EDX:EAX. Use cdq first to sign-extend EAX into EDX.

    mov $100, %eax
    mov $7,   %ecx
    cdq             # sign-extend eax into edx:eax
    idiv %ecx       # eax = 14, edx = 2

---

## Two-pass assembly

fasm does two passes over the instruction list:

  Pass 1 — walk all items, compute instruction sizes, record every label's
            absolute address into the symbol table.
  Pass 2 — emit real machine code; jmp/call/jcc compute
            rel32 = target - (here + instr_size).

Label sizes are fixed regardless of the resolved address, so pass 1 is
always correct without iteration.

---

## Output format

Minimal ELF32 executable with a single PT_LOAD segment:

    [ELF header 52 B][Program header 32 B][.text][.data]

.bss is handled via memsz > filesz — no bytes in the file, zeroed by the
kernel on load. Load address: 0x08048000 (standard i386 Linux base).

---

## Limitations

- %esp cannot be used as a memory base (no SIB byte)
- imul/idiv one-operand form only
- shl/shr shift count is $imm8 or %cl only
- no indirect jumps (jmp *%eax)
- single output file only (no object files / linking)
