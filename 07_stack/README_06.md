# 06.asm - the same save and restore in 32-bit mode

## What the program does

```asm
BITS 32
section .text
    global _start

_start:
    mov eax, 0x11223344
    mov ebx, 0x1234

    push eax
    push ebx
    nop

    pop ebx
    pop eax

    mov eax, 1
    xor ebx, ebx
    int 0x80
```

This is 05.asm ported to 32-bit. Two values are saved on the stack and restored in
reverse order so each register gets its own value back. The pattern `0x11223344` fills
all four bytes of EAX, the same way `0x1122334455667788` filled all eight of RAX in the
64-bit version, which makes the smaller slot size easy to see in a byte dump.

## Building and running it

```
nasm -f elf32 -g -F dwarf -o 06.o 06.asm
ld -m elf_i386 -o 06 06.o
./06 ; echo $?
```

Exits quietly with status 0, through syscall 1 (exit) with `int 0x80` and the exit code
taken from EBX.

## Stepping through it

Each row is the state after the instruction ran.

| # | Instruction | ESP | Delta | EAX | EBX | Stack |
|---|---|---|---|---|---|---|
| 0 | `mov eax, 0x11223344` | `0xffff59a0` | 0 | `0x11223344` | `0x0` | nothing |
| 1 | `mov ebx, 0x1234` | `0xffff59a0` | 0 | same | `0x1234` | nothing |
| 2 | `push eax` | `0xffff599c` | -4 | same | same | `0x11223344` goes to [esp] |
| 3 | `push ebx` | `0xffff5998` | -8 | same | same | `0x1234` goes to [esp] |
| 4 | `nop` | `0xffff5998` | -8 | | | place to stop and look |
| 5 | `pop ebx` | `0xffff599c` | -4 | same | `0x1234` | reads the top slot |
| 6 | `pop eax` | `0xffff59a0` | 0 | `0x11223344` | same | reads the next slot |
| 7 | `mov eax, 1` | `0xffff59a0` | 0 | `0x1` | `0x1234` | nothing |
| 8 | `xor ebx, ebx` | `0xffff59a0` | 0 | `0x1` | `0x0` | nothing |
| 9 | `int 0x80` | `0xffff59a0` | 0 | | | exits with status 0 |

## Memory at the deepest point

At the nop:

```
[esp+0 ]=0x00001234      EBX's value, pushed last, on top
[esp+4 ]=0x11223344      EAX's value, pushed first, underneath
```

```
          higher addresses
  base+8    argc = 1
  base+4    0x11223344     from push eax
  base+0    0x00001234     from push ebx    <== ESP
          lower addresses
```

The whole working area is 8 bytes, where 05.asm needed 16 for the same two values.

## The individual bytes

```
(gdb) x/8xb $esp
+0x0:   0x34  0x12  0x00  0x00      0x44  0x33  0x22  0x11

dword at [esp+0] : 0x00001234   (EBX slot)
dword at [esp+4] : 0x11223344   (EAX slot)
```

Little-endian, same as 64-bit: `0x11223344` is stored `44 33 22 11`.

And a slot here is 4 bytes. `0x11223344` fills its slot exactly. `0x1234` only needs 2
bytes and still gets a full 4-byte slot, zero extended to `34 12 00 00`, so two bytes
are wasted instead of the six wasted in the 64-bit version.

## The pop order

```
push eax     --+
push ebx     --+--+
pop  ebx     --+  |
pop  eax     -----+
```

LIFO means the reverse order is the only one that gives each register its own value
back. gdb shows the round trip is exact:

```
before the pushes:  eax=0x11223344  ebx=0x1234
after the pops   :  eax=0x11223344  ebx=0x1234
```

01.asm is the counterexample, where the pops were in the same order as the pushes and
the values came back in the wrong registers.

## 32-bit next to 64-bit

| | 05.asm (64-bit) | 06.asm (32-bit) |
|---|---|---|
| stack pointer | RSP | ESP |
| slot size | 8 bytes | 4 bytes |
| push moves pointer by | -8 | -4 |
| pointer trace | `6510 -> 6508 -> 6500 -> 6508 -> 6510` | `59a0 -> 599c -> 5998 -> 599c -> 59a0` |
| deepest delta | -16 | -8 |
| big pattern | `0x1122334455667788` (8 bytes) | `0x11223344` (4 bytes) |
| `0x1234` stored as | `34 12 00 00 00 00 00 00` | `34 12 00 00` |
| push eax / push rax opcode | 0x50 | 0x50 |
| exit | eax=60, code in RDI, `syscall` | eax=1, code in EBX, `int 0x80` |

## What I noticed from running it

**ESP moved in steps of 4:** `59a0 -> 599c -> 5998 -> 599c -> 59a0`. Half the stride of
the 64-bit version, same shape.

**push eax and push rax are literally the same byte.** From `objdump -d -M intel`:

```
8049000: b8 44 33 22 11    mov    eax,0x11223344
8049005: bb 34 12 00 00    mov    ebx,0x1234
804900a: 50                push   eax
804900b: 53                push   ebx
804900c: 90                nop
804900d: 5b                pop    ebx
804900e: 58                pop    eax
804900f: b8 01 00 00 00    mov    eax,0x1
8049014: 31 db             xor    ebx,ebx
8049016: cd 80             int    0x80
```

`push eax` is 0x50 here and `push rax` in 05.asm is also 0x50. The opcode carries no
size information whatsoever, the CPU's current mode supplies it. Same for `pop ebx` /
`pop rbx` (0x5b) and `pop eax` / `pop rax` (0x58). This is why `BITS 32` is not just
decoration, it decides whether these one-byte instructions move 4 bytes or 8.

**The whole program is 24 bytes of code**, `0x8049000` to `0x8049017`, and the longest
instructions are the two mov immediates. The stack operations are the cheapest
instructions in the program at one byte each, which is a big part of why the stack is
the default place to put things.

**`mov ebx, 0x1234` used the full 5-byte immediate form** (`bb 34 12 00 00`), writing a
32-bit immediate even though the value needs 2 bytes. The zero extension happens in the
register before the push, so the pushed slot reads `0x00001234`.

**It balances**, two pushes and two pops, delta 0 at the exit, ESP back at `0xffff59a0`.

**pop left the bytes behind**, same as every other program here. After both pops
`0x1234` and `0x11223344` were still readable below ESP. The pointer moved but the
memory did not change.

**`xor ebx, ebx` is actually doing something here, not just tidying up.** In 32-bit
Linux the `int 0x80` exit code comes from EBX, and EBX had just been restored to
`0x1234` by `pop ebx`. So without the xor the program would exit with
`0x1234 & 0xFF = 0x34` which is 52, not 0, since exit statuses get truncated to the low
byte.

I tested that by building a copy with the xor commented out:

```
$ ./v06 ; echo $?
52
```

52 is `0x34`, the low byte of the restored `0x1234`, so it came out exactly as expected.
It is a good example of how restoring a register properly and then immediately writing
over it is completely normal. The save and restore only protects the value for as long
as the surrounding code needs it.

**The addresses are low 32-bit values:** code at `0x08049000` and stack at `0xffff59a0`.
The 64-bit programs sit at `0x401000` and `0x00007fffffff6510`.

## Summary

```
ESP 59a0   EAX=0x11223344   EBX=0x1234
  push eax --> 599c   [esp] = 0x11223344   bytes: 44 33 22 11
  push ebx --> 5998   [esp] = 0x00001234   bytes: 34 12 00 00
  pop  ebx --> 599c   EBX = 0x1234         reverse order
  pop  eax --> 59a0   EAX = 0x11223344     both fully restored
  then EAX := 1 for the exit syscall, EBX := 0 for the status -> int 0x80, exit 0
```

Every rule from the 64-bit programs still holds. The stack grows down, push decrements
then writes, pop reads then increments, LIFO forces the reverse order on restores, and
pop never erases anything. The only differences are the 4-byte stride, the register
names and the `int 0x80` exit convention, and the push and pop opcodes are the same
bytes in both modes.
