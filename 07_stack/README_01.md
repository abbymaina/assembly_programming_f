# 01.asm - basic push and pop

## What the program does

```asm
section .text
    global _start

_start:
    mov rax, 100
    mov rbx, 200

    push rax
    push rbx
    nop

    pop rcx
    pop rdx
    nop

    mov rax, 60
    xor rdi, rdi
    syscall
```

It puts 100 in RAX and 200 in RBX, pushes both onto the stack, then pops them back
into RCX and RDX. Nothing gets printed, so the only way to see what is happening is to
run it in gdb and watch RSP and the stack memory.

## Building and running it

```
nasm -f elf64 -g -F dwarf -o 01.o 01.asm
ld -o 01 01.o
./01 ; echo $?
```

It exits quietly with status 0. The exit comes from the last three lines: 60 is the
syscall number for exit on 64-bit Linux, and `xor rdi, rdi` sets the exit code to 0.

I used `-g -F dwarf` so gdb could show me the source lines while stepping.

## The stack before the first instruction

One thing I did not expect is that the stack is already in use before my code runs.
The kernel sets it up and puts argc, the argv pointers and the environment pointers
there, then points RSP at it:

```
initial rsp = 0x00007fffffff6510
[rsp+0 ] = 0x0000000000000001   argc (1, just the program name)
[rsp+8 ] = 0x00007fffffff6e66   argv[0] pointer
[rsp+16] = 0x0000000000000000   NULL that ends argv
[rsp+24] = 0x00007fffffff6eba   envp[0]
```

So RSP starts at a high address pointing at real data, and everything I push goes
below that at lower addresses.

## Stepping through it

I stepped it one instruction at a time in gdb. Each row is the state after the
instruction ran. Delta is how far RSP has moved from where it started.

| # | Instruction | RSP | Delta | What happened on the stack |
|---|---|---|---|---|
| 0 | `mov rax, 100` | `0x7fffffff6510` | 0 | nothing, registers only |
| 1 | `mov rbx, 200` | `0x7fffffff6510` | 0 | nothing |
| 2 | `push rax` | `0x7fffffff6508` | -8 | RSP goes down 8, then 100 is written at the new [rsp] |
| 3 | `push rbx` | `0x7fffffff6500` | -16 | RSP goes down 8 again, 200 written at new [rsp] |
| 4 | `nop` | `0x7fffffff6500` | -16 | nothing, I used it as a place to stop and look |
| 5 | `pop rcx` | `0x7fffffff6508` | -8 | RCX = [rsp] which is 200, then RSP goes up 8 |
| 6 | `pop rdx` | `0x7fffffff6510` | 0 | RDX = [rsp] which is 100, then RSP goes up 8 |
| 7 | `nop` | `0x7fffffff6510` | 0 | nothing |
| 8 | `mov rax, 60` | `0x7fffffff6510` | 0 | nothing |
| 9 | `xor rdi, rdi` | `0x7fffffff6510` | 0 | nothing |
| 10 | `syscall` | `0x7fffffff6510` | 0 | program exits |

### How memory looked

Before any push:

```
      higher addresses
  0x7fffffff6510   argc = 1          <-- RSP
  0x7fffffff6508   not written yet
  0x7fffffff6500   not written yet
      lower addresses
```

After `push rax`. RSP drops by 8 first, then the value goes in:

```
  0x7fffffff6510   argc = 1
  0x7fffffff6508   100               <-- RSP
  0x7fffffff6500   not written yet
```

After `push rbx`:

```
  0x7fffffff6510   argc = 1
  0x7fffffff6508   100               (now at [rsp+8])
  0x7fffffff6500   200               <-- RSP
```

This is what gdb showed me at the nop on line 10, which matches:

```
rsp=0x00007fffffff6500
[rsp+0 ]=0x00000000000000c8      200, pushed last, so it is on top
[rsp+8 ]=0x0000000000000064      100, pushed first, sitting underneath
[rsp+16]=0x0000000000000001      argc, still fine
```

Then `pop rcx` reads the top slot (200) and raises RSP by 8, and `pop rdx` reads the
next one (100) and raises RSP by 8 again, which puts RSP back exactly where the kernel
left it.

## The values come back in the wrong registers

This is the main thing to notice in this program:

```
before:  rax=100  rbx=200  rcx=0    rdx=0
after :  rax=100  rbx=200  rcx=200  rdx=100
```

gdb confirmed it at the last nop: `rcx=0xc8` (200) and `rdx=0x64` (100). So RCX got
RBX's value and RDX got RAX's value.

That is not a mistake in the CPU, it is just what a stack is. The stack is LIFO, last
in first out. RBX was pushed last so it comes off first. The pushes and pops cross
over:

```
push rax        pop rcx     (first pop gets what the last push wrote)
push rbx        pop rdx
```

If you want each register to get its own value back, the pops have to be written in
the opposite order to the pushes. I tested that by building a small version of this:

```asm
push rax
push rbx
pop rbx      ; reverse order this time
pop rax
```

and running it in gdb:

```
rax=100  rbx=200
```

Each register got its own value back. This is why when you save registers around a
block of code you do push A, push B, then pop B, pop A. It has to be nested, not
parallel. 05.asm in this folder uses the correct order, so it is a good one to compare
against.

## What I noticed from running it

**RSP only moves in steps of 8.** In 64-bit mode a stack slot is 8 bytes, so every
push takes 8 off RSP and every pop adds 8 back, even though 100 and 200 would each fit
in one byte. The trace goes `6510 -> 6508 -> 6500 -> 6508 -> 6510`. There is no way to
push something smaller, `push al` does not exist.

**The stack grows downwards.** RSP gets smaller when you push. So "top of the stack"
means the lowest address in use, which feels backwards at first. The trace settles it:
the first push took RSP from `0x...6510` to `0x...6508`.

**Push and pop are each two steps, in a set order.** `push rax` is decrement RSP then
write. `pop rcx` is read then increment RSP. The order matters, because it means [rsp]
is always the thing that was pushed most recently and never an empty slot.

**Pop does not delete anything.** After both pops RSP is back at `0x...6510`, but 100
and 200 are still sitting in memory at `0x...6508` and `0x...6500`. Pop only copies the
value out and moves the pointer. Those bytes are dead now and the next push will write
over them without any warning, which is why you cannot trust anything below RSP.

**The pushes and pops balance.** Two pushes, two pops, and the final delta is 0, so RSP
ends where it started. In a function this is what lets `ret` find the right return
address, as 02.asm shows.

**The actual addresses change every run.** I ran the same binary three times with ASLR
on and got three different starting stack pointers:

```
0x7ffcbb5a5590
0x7ffc19db16b0
0x7ffe8bf78150
```

gdb disables randomisation by default so I got the same `0x7fffffff6510` every time,
but even then it shifts if the environment changes size, because the kernel copies the
environment strings onto the stack above RSP. So only the deltas (-8, -16, back to 0)
actually belong to the program. The addresses themselves do not.

**argc never got damaged.** `[rsp+16]` still read `0x1` at the deepest point, because
the stack grows away from the kernel's data instead of into it.

## Summary

```
RSP: 6510 --push rax--> 6508 --push rbx--> 6500 --pop rcx--> 6508 --pop rdx--> 6510
                  100              200           (200 -> RCX)      (100 -> RDX)
delta:  0            -8               -16             -8              0
```

Nothing permanent happens to memory. The lasting effect is in the registers, and the
two values end up swapped compared to the order they were pushed, which is exactly
what LIFO means.
