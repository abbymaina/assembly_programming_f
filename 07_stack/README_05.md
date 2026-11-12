# 05.asm - pushing a full 64-bit value, and popping in the right order

## What the program does

```asm
section .text
    global _start

_start:
    mov rax, 0x1122334455667788
    mov rbx, 0x1234

    push rax
    push rbx
    nop

    pop rbx
    pop rax

    mov rax, 60
    xor rdi, rdi
    syscall
```

Mechanically this is 01.asm again, two pushes and two pops, but two things are
deliberately different.

First, the values are hex patterns instead of small decimals. `0x1122334455667788` uses
all eight bytes of a 64-bit register, which shows how much a stack slot actually holds
and how the bytes sit in memory. `0x1234` is tiny next to it.

Second, the pops are in the reverse order to the pushes (`pop rbx` then `pop rax`), so
each register gets its own value back. 01.asm popped in the same order and the values
came back in the wrong registers. This one is the correct way to save and restore.

## Building and running it

```
nasm -f elf64 -g -F dwarf -o 05.o 05.asm
ld -o 05 05.o
./05 ; echo $?
```

Exits quietly with status 0.

## Stepping through it

Each row is the state after the instruction ran.

| # | Instruction | RSP | Delta | RAX | RBX | Stack |
|---|---|---|---|---|---|---|
| 0 | `mov rax, 0x1122334455667788` | `0x7fffffff6510` | 0 | `0x1122334455667788` | `0x0` | nothing |
| 1 | `mov rbx, 0x1234` | `0x7fffffff6510` | 0 | same | `0x1234` | nothing |
| 2 | `push rax` | `0x7fffffff6508` | -8 | same | same | `0x1122334455667788` goes to [rsp] |
| 3 | `push rbx` | `0x7fffffff6500` | -16 | same | same | `0x1234` goes to [rsp] |
| 4 | `nop` | `0x7fffffff6500` | -16 | | | place to stop and look |
| 5 | `pop rbx` | `0x7fffffff6508` | -8 | same | `0x1234` | reads the top slot |
| 6 | `pop rax` | `0x7fffffff6510` | 0 | `0x1122334455667788` | same | reads the next slot |
| 7 | `mov rax, 60` | `0x7fffffff6510` | 0 | `0x3c` | `0x1234` | nothing |
| 8 | `xor rdi, rdi` | `0x7fffffff6510` | 0 | `0x3c` | `0x1234` | nothing |
| 9 | `syscall` | `0x7fffffff6510` | 0 | | | exits |

## Memory at the deepest point

At the nop:

```
[rsp+0 ]=0x0000000000001234      RBX's value, pushed last, so it is on top
[rsp+8 ]=0x1122334455667788      RAX's value, pushed first, underneath
```

```
          higher addresses
  base+16   argc = 1
  base+8    0x1122334455667788    from push rax
  base+0    0x0000000000001234    from push rbx    <== RSP
          lower addresses
```

## Looking at the individual bytes

The hex patterns make the raw memory readable, so I dumped 16 bytes from RSP at the
nop:

```
(gdb) x/16xb $rsp
+0x0:   0x34  0x12  0x00  0x00  0x00  0x00  0x00  0x00
+0x8:   0x88  0x77  0x66  0x55  0x44  0x33  0x22  0x11

qword at [rsp+0] : 0x0000000000001234   (RBX slot)
qword at [rsp+8] : 0x1122334455667788   (RAX slot)
```

Two things come out of this.

**It is stored little-endian.** `0x1122334455667788` is written with the least
significant byte first: `88 77 66 55 44 33 22 11`. The bytes look reversed compared to
how the constant is written in the source. That is not a stack thing, it is how x86
stores all multi-byte integers, but the stack is where it becomes obvious because push
and pop are the only instructions whose memory address is implicit. pop puts the bytes
back together the same way so the round trip is exact. It only matters if you read the
slot one byte at a time.

**A slot is 8 bytes whether you need them or not.** `0x1234` needs 2 bytes. It got a
full 8-byte slot, zero extended: `34 12 00 00 00 00 00 00`. Those six zero bytes are
real stack space being used up. There is no narrower form of push for general purpose
registers in 64-bit mode, so you cannot pack small values together with push. If you
really need to, you reserve a block with `sub rsp, N` and write into it with sized mov
instructions, which is what 02.asm does.

## Why the pop order is right this time

```
push rax     --+
push rbx     --+--+
pop  rbx     --+  |   undoes push rbx
pop  rax     -----+   undoes push rax
```

Nested rather than parallel. Because the stack is LIFO, the last thing pushed has to be
the first thing popped, so the pops have to be the pushes read backwards.

gdb shows the round trip loses nothing:

```
before the pushes:  rax=0x1122334455667788  rbx=0x1234
after the pops   :  rax=0x1122334455667788  rbx=0x1234
```

All 64 bits came back. Compare 01.asm, which pushed rax then rbx but popped rcx then
rdx in the same order rather than reversed, and ended up with the two values in the
opposite registers.

This is the normal way to borrow registers around a block of code: push whatever you
are about to overwrite, do the work, pop in reverse. Because it is symmetric you can
also nest one of these blocks inside another one to any depth.

## What I noticed from running it

**RSP moved in steps of 8:** `6510 -> 6508 -> 6500 -> 6508 -> 6510`. The size of the
value makes no difference, `0x1234` and `0x1122334455667788` each cost one slot.

**The full 64-bit value survived with nothing lost.** Nothing got truncated anywhere in
the round trip. Worth actually checking, because a mode mismatch is a real way to break
this: if this were assembled as 32-bit then `0x1122334455667788` would not even fit in
EAX. In BITS 64, push and pop move the whole register.

**push rax is a single byte, 0x50.** From objdump:

```
40100f:  50    push rax
401010:  53    push rbx
401012:  5b    pop  rbx
401013:  58    pop  rax
```

The 32-bit `push eax` over in 06.asm is also 0x50, and `pop ebx` / `pop eax` are also
0x5b and 0x58. The bytes are identical and only the CPU's current mode decides whether
they move 8 bytes or 4. So the operand size is not in the instruction at all, it is in
the mode. That is useful to know when reading a disassembly without knowing what it
was built as.

**`mov rbx, 0x1234` assembled as `mov $0x1234,%ebx`**, 5 bytes, while
`mov rax, 0x1122...` needed the 10-byte `movabs` form. nasm picked the shortest encoding
that works, because writing to a 32-bit register zero extends into the full 64-bit
register, so for small constants the short form gives the same result. It does mean the
instruction is 32-bit while the stack slot is 64-bit, with the zero extension happening
in the register before the push.

**It balances**, delta 0 at the exit, two pushes and two pops.

**pop left the bytes behind.** After both pops RSP was back at the base but `0x1234` and
`0x1122334455667788` were still readable in memory below it. pop copies and advances, it
does not clear. Anything you push stays legible in the dead area below RSP until
something else writes over it, which is why sensitive values get explicitly zeroed
instead of just popped.

**RBX was still `0x1234` at the exit syscall** and it did not matter, because the 64-bit
exit code comes from RDI, which `xor rdi, rdi` had zeroed. In the 32-bit programs EBX is
the exit code, which is why 06.asm has to zero it.

## Summary

```
RSP 6510   RAX=0x1122334455667788   RBX=0x1234
  push rax --> 6508   [rsp] = 0x1122334455667788   bytes: 88 77 66 55 44 33 22 11
  push rbx --> 6500   [rsp] = 0x0000000000001234   bytes: 34 12 00 00 00 00 00 00
  pop  rbx --> 6508   RBX = 0x1234                 reverse order
  pop  rax --> 6510   RAX = 0x1122334455667788     both fully restored
exit 0
```

So this program shows three things at once: a 64-bit stack slot is 8 bytes no matter
what you put in it, x86 stores those bytes little-endian, and a correct save and
restore has to pop in the reverse order to the pushes.
