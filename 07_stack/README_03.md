# 03.asm - call and ret with no stack frame

## What the program does

```asm
section .text
    global _start

_start:
    mov rax, 10
    call calculate

after_call:
    ; RAX should now be 15
    nop

    mov rax, 60
    xor rdi, rdi
    syscall

calculate:
    add rax, 5
    ret
```

A value is passed to a subroutine in a register, the subroutine changes it and returns.
`calculate` has no prologue, no locals, and no push or pop in it at all. It is a leaf
routine.

That makes this the simplest program in the folder for seeing what call and ret
actually do, because the only thing the stack is used for in the whole program is
holding one return address for the length of two instructions.

## Building and running it

```
nasm -f elf64 -g -F dwarf -o 03.o 03.asm
ld -o 03 03.o
./03 ; echo $?
```

Exits quietly with status 0. The comment in the source says RAX should be 15 after the
call, and the trace below shows that it is.

## Stepping through it

Each row is the state after the instruction ran.

| # | Instruction | RSP | Delta | RAX | What the stack did |
|---|---|---|---|---|---|
| 0 | `mov rax, 10` | `0x7fffffff6510` | 0 | `0xa` | nothing |
| 1 | `call calculate` | `0x7fffffff6508` | -8 | `0xa` | pushed `0x40100a`, jumped to `0x401015` |
| 2 | `add rax, 5` | `0x7fffffff6508` | -8 | `0xf` | nothing, registers only |
| 3 | `ret` | `0x7fffffff6510` | 0 | `0xf` | popped `0x40100a` into RIP |
| 4 | `nop` (at after_call) | `0x7fffffff6510` | 0 | `0xf` | nothing |
| 5 | `mov rax, 60` | `0x7fffffff6510` | 0 | `0x3c` | nothing |
| 6 | `xor rdi, rdi` | `0x7fffffff6510` | 0 | `0x3c` | nothing |
| 7 | `syscall` | `0x7fffffff6510` | 0 | | exits |

RAX comes out of the subroutine as `0xf` which is 15, the same as the comment
predicted, and it is still 15 when `after_call` runs. So the result travelled back in a
register and the stack only carried the address.

## The one stack write

The entire use of the stack in this program is a single 8-byte slot.

Before the call:

```
  0x7fffffff6510   argc = 1            <-- RSP
  0x7fffffff6508   not written yet
```

After `call calculate`:

```
  0x7fffffff6510   argc = 1
  0x7fffffff6508   0x000000000040100a  <-- RSP     the return address
```

From gdb:

```
rsp=0x00007fffffff6508  delta=-8
[rsp+0 ]=0x000000000040100a      return address
[rsp+8 ]=0x0000000000000001      argc, untouched
```

After ret, RSP is back at `0x7fffffff6510` and execution carries on at `0x40100a`.

`0x40100a` is the address of the `after_call` label. gdb showed it as `<after_call>`,
and `nm` on the binary confirms it:

```
0000000000401000 T _start
000000000040100a t after_call
0000000000401015 t calculate
```

So call pushed the address of the instruction right after itself, which is what a
return address is.

## Checking that ret is just a pop into RIP

ret is usually described as a pop whose destination is the instruction pointer. I
wanted to confirm that rather than take it on faith, so I wrote the return out by hand:

```asm
calculate:
    add rax, 5
    pop rbx          ; take the return address off the stack myself
    jmp rbx          ; and jump to it manually
```

Result:

```
reached after_call.  rax=15   rbx (popped) = 0x40100a   after_call = 0x40100a
$ ./v03 ; echo $?
0
```

The value `pop rbx` got out is exactly the address of `after_call`, the program reached
`after_call` normally, and it exited 0. Same behaviour as ret. So:

```
call X   is the same as   push (address of next instruction) ; jmp X
ret      is the same as   pop RIP
```

The differences are that ret is one byte instead of four, and it does not use up a
general purpose register.

## Why the balance matters even here

`calculate` has no stack operations in it at all, and that is exactly what makes the
ret safe. ret does not go looking for the return address or check it, it just loads 8
bytes from [rsp] into RIP. So the rule is:

> when ret runs, RSP has to be pointing at the return address that call pushed.

Any routine that pushes has to pop the same amount again before returning, otherwise
ret reads the wrong 8 bytes and jumps somewhere random. The broken version of 02.asm
shows what that looks like (RIP got loaded with the value 100 and the program
segfaulted). In 03.asm the condition is true automatically, because delta stays at -8
for the whole body of `calculate` and nothing disturbs it.

This is also how nesting and recursion work. Every call pushes another return address
below the last one, so the stack holds a chain of them with the oldest at the bottom.
ret always takes the most recent one, LIFO again, so calls unwind in the exact reverse
of the order they were made.

## What I noticed from running it

**The stack was used for 8 bytes, for two instructions.** The deepest delta was -8. A
routine that takes its input in a register, does a calculation and returns needs
nothing more than that.

**No prologue is needed when there are no locals.** `calculate` never touches RBP, and
RBP stayed at 0x0 for the whole program. The point of `push rbp` / `mov rbp, rsp` is to
get stable `[rbp±n]` addressing, so with no locals and no stack arguments there is
nothing to address and the prologue would just be wasted instructions. Compilers leave
it out for leaf functions too.

**The argument and the return value both went through RAX, not the stack.** _start put
10 in RAX, `calculate` added 5, and _start read 15 back out. The x86-64 ABI passes the
first six integer arguments in RDI, RSI, RDX, RCX, R8, R9 and returns in RAX for this
reason, since registers are much cheaper than memory. The stack only gets involved from
the seventh argument onwards, for locals that are too big or that need an address, and
for the return address itself.

**The return address is just data.** It sits in normal writable memory and my `pop rbx`
read it like any other value. Nothing in the hardware marks it as special. That is why
ret is so cheap, and also why overwriting a return address is such a well known
exploit.

**The after_call label is not needed by the machine.** It only exists so that I and gdb
can refer to `0x40100a` by name. The call instruction worked out that address from its
own length, so removing the label would not change a single byte of the assembled
program.

**RSP finished where it started**, delta 0, so the exit syscall ran on a clean stack.

## Summary

```
RSP 6510   mov rax,10                        RAX=10
     |
     |-- call calculate -- push 0x40100a --> RSP 6508   (jump to 0x401015)
     |                                        add rax,5 -> RAX=15
     |<- ret ------------- pop into RIP ---- RSP 6510   (resume at 0x40100a)
     |
     '-- after_call: RAX is still 15 -> exit 0
```

One push and one pop, both done implicitly by call and ret. The only job the stack has
here is remembering a single address, which is its most basic job and the one
everything else is built on top of.
