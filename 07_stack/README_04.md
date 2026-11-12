# 04.asm - the same stack frame in 32-bit mode

## What the program does

```asm
BITS 32
section .text
    global _start

_start:
    call work

    ; Linux i386 exit syscall
    mov eax, 1
    xor ebx, ebx
    int 0x80

work:
    push ebp
    mov ebp, esp
    sub esp, 8

    mov dword [ebp - 4], 100
    mov dword [ebp - 8], 200

    mov eax, [ebp - 4]
    mov ebx, [ebp - 8]
    nop

    mov esp, ebp
    pop ebp
    ret
```

This is 02.asm rewritten for 32-bit x86. The structure is the same: call, prologue, two
locals, read them back, epilogue, ret. So the useful thing here is comparing the two
and seeing what changes when a stack slot goes from 8 bytes down to 4.

## Building and running it

`BITS 32` only tells nasm how to encode the instructions. The object format and the
linker have to be told separately, which caught me out the first time:

```
nasm -f elf32 -g -F dwarf -o 04.o 04.asm
ld -m elf_i386 -o 04 04.o
./04 ; echo $?
```

Exits quietly with status 0. The exit works differently from the 64-bit programs: it is
syscall 1 (exit) through `int 0x80`, and the exit code comes from EBX instead of RDI.
`xor ebx, ebx` makes it 0.

Running a 32-bit binary needs the kernel's IA-32 emulation turned on. It ran fine here.

## Starting state

```
mode        : 32-bit, stack slot = 4 bytes
initial esp = 0xffff59a0     ([esp] = argc = 1)
ebp         = 0x0
```

The stack is down in the low 4 GB of the address space, so the pointer is a 32-bit
value, `0xffff59a0` instead of the 64-bit programs' `0x00007fffffff6510`. Same idea,
half the width.

## Stepping through it

Each row is the state after the instruction ran.

| # | Instruction | ESP | Delta | What happened on the stack |
|---|---|---|---|---|
| 0 | `call work` | `0xffff599c` | -4 | pushes return address `0x08049005` (4 bytes), jumps to work |
| 1 | `push ebp` | `0xffff5998` | -8 | saves the caller's EBP (0) |
| 2 | `mov ebp, esp` | `0xffff5998` | -8 | nothing written, EBP becomes the frame anchor |
| 3 | `sub esp, 8` | `0xffff5990` | -16 | reserves 8 bytes for the two locals |
| 4 | `mov dword [ebp-4], 100` | `0xffff5990` | -16 | writes local 1 |
| 5 | `mov dword [ebp-8], 200` | `0xffff5990` | -16 | writes local 2 |
| 6 | `mov eax, [ebp-4]` | `0xffff5990` | -16 | EAX = 100 |
| 7 | `mov ebx, [ebp-8]` | `0xffff5990` | -16 | EBX = 200 |
| 8 | `nop` | `0xffff5990` | -16 | place to stop and look |
| 9 | `mov esp, ebp` | `0xffff5998` | -8 | drops the locals |
| 10 | `pop ebp` | `0xffff599c` | -4 | restores the caller's EBP to 0 |
| 11 | `ret` | `0xffff59a0` | 0 | pops `0x08049005` into EIP |
| 12 | `mov eax, 1` | `0xffff59a0` | 0 | |
| 13 | `xor ebx, ebx` | `0xffff59a0` | 0 | |
| 14 | `int 0x80` | `0xffff59a0` | 0 | exits with status 0 |

## The frame

At the nop (step 8), printed from gdb:

```
ebp            = 0xffff59a8
esp            = 0xffff59a0   (ebp - esp = 8 bytes of locals)
[ebp+4]  ret       = 0x08049005   pushed by call
[ebp+0]  saved ebp = 0x00000000   pushed by push ebp
[ebp-4]  local1    = 100
[ebp-8]  local2    = 200
eax=100 ebx=200
```

The absolute base moves slightly between gdb runs because the kernel copies argv and
envp onto the stack above ESP, so it depends on the environment. The offsets are the
part that matters.

```
          higher addresses
  0xffff59a0   argc = 1                  caller's stuff
  0xffff599c   return addr 0x08049005    pushed by call      [ebp+4]
  0xffff5998   saved EBP = 0             pushed by push ebp  [ebp+0]  <== EBP
  0xffff5994   local1 = 100                                  [ebp-4]
  0xffff5990   local2 = 200                                  [ebp-8]  <== ESP
          lower addresses
```

## 32-bit next to 64-bit

Putting this beside 02.asm shows that the slot size is basically the whole difference:

| | 02.asm (64-bit) | 04.asm (32-bit) |
|---|---|---|
| stack pointer | RSP | ESP |
| frame pointer | RBP | EBP |
| slot size | 8 bytes | 4 bytes |
| call moves pointer by | -8 | -4 |
| push rbp / push ebp moves by | -8 | -4 |
| local 1 at | `[rbp-8]` | `[ebp-4]` |
| local 2 at | `[rbp-16]` | `[ebp-8]` |
| space reserved | `sub rsp, 16` | `sub esp, 8` |
| return address at | `[rbp+8]` | `[ebp+4]` |
| total frame cost | 32 bytes | 16 bytes |
| local declared as | `qword` | `dword` |
| exit | rax=60, code in RDI, `syscall` | eax=1, code in EBX, `int 0x80` |

Everything structural stays the same. The return address is still one slot above the
saved frame pointer, the locals still go downwards from EBP, and the epilogue is still
`mov esp, ebp` / `pop ebp` / `ret`. Only the stride changed, so the frame does the same
work in half the memory.

This also cleared up the `[ebp+4]` vs `[rbp+8]` thing that confused me when reading
code from both architectures. It is not a different convention. It is one slot in both
cases, and a slot is just a different size.

## Why dword and qword are not interchangeable here

`mov dword [ebp - 4], 100` writes 4 bytes. If it were qword it would write 8 bytes
starting at `[ebp-4]`, so it would run from `[ebp-4]` up through `[ebp+3]` and write
straight over the saved EBP at `[ebp+0]`. The locals here are only 4 bytes apart, so
the operand size has to match the spacing.

In practice the assembler stops you before the CPU gets a chance. I tried it:

```
$ nasm -f elf32 -o q.o q.asm
q.asm:5: error: instruction not supported in 32-bit mode
```

There is no 32-bit encoding for an 8-byte immediate store at all. So the 8-byte spacing
in 02.asm and the 4-byte spacing here each match their own mode's slot size.

## What I noticed from running it

**ESP moves in steps of 4 the whole way through.** The trace goes
`59a0 -> 599c -> 5998 -> 5990 -> 5998 -> 599c -> 59a0`, against 02.asm's steps of 8.
Same shape, half the stride.

**call pushed 4 bytes, not 8.** `0x08049005` is the address of `mov eax, 1`, the
instruction right after the call, and a 32-bit return address fits in one 32-bit slot.

**The addresses are low.** Code at `0x08049000` and stack at `0xffff59a0`. That is the
classic i386 layout where the stack sits just under the 4 GB ceiling and grows down
towards the heap. The 64-bit programs put code at `0x401000` and the stack at
`0x00007fff...`.

**The frame was 16 bytes deep** compared to 32 in the 64-bit version: 4 for the return
address, 4 for the saved EBP and 8 for the locals. Exactly half.

**sub esp, 8 reserved space without writing anything**, same as the 64-bit version.
Right after it both reserved slots read `0x00000000`, but that is just untouched memory
that happened to be zero rather than anything the instruction did. The real values only
showed up at steps 4 and 5.

**It balances.** Delta went `0, -4, -8, -16, -8, -4, 0`. EBP went back to 0, ESP went
back to `0xffff59a0`, and ret found the right address because of it.

**EAX and EBX carried 100 and 200 out of the frame**, and then EAX was immediately
loaded with the syscall number 1 and EBX zeroed for the exit code. The locals' memory
was abandoned at `mov esp, ebp`, so only the register copies survived, and not for
long.

**The exit convention is genuinely different, not just renamed registers.** `int 0x80`
goes into the kernel through the old IA-32 gate, the syscall numbers are from a
different table (exit is 1 here, 60 in 64-bit) and the exit code comes from EBX instead
of RDI. So porting a program between modes means looking at the syscall interface too,
not only the register names.

## Summary

```
_start:  ESP 59a0
  call work --push return addr (4 bytes)--> 599c
    push ebp ----------------------------->  5998   EBP := 5998
    sub esp,8 ---------------------------->  5990   locals [ebp-4] and [ebp-8] owned
      write 100 and 200, read into EAX and EBX
    mov esp,ebp -------------------------->  5998   locals released
    pop ebp ------------------------------>  599c   EBP back to 0
  ret --pop into EIP---------------------->  59a0   back at 0x08049005
exit through int 0x80, status 0
```

The thing I took away from this one is that stack discipline does not depend on the
architecture. Grows down, LIFO, call pushes, ret pops, RBP/EBP anchors the frame, and
pushes have to balance pops. All of that is identical in both modes. The only things
that change are the width of a slot, the register names and the way you call into the
kernel.
