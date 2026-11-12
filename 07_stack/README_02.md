# 02.asm - a proper stack frame with call, locals and ret

## What the program does

```asm
section .text
    global _start

_start:
    call work
    mov rax, 60
    xor rdi, rdi
    syscall

work:
    push rbp
    mov rbp, rsp
    sub rsp, 16

    mov qword [rbp - 8], 100
    mov qword [rbp - 16], 200

    mov rax, [rbp - 8]
    mov rbx, [rbp - 16]
    nop

    mov rsp, rbp
    pop rbp
    ret
```

`_start` calls `work`. `work` sets up a stack frame, stores two local variables in it,
reads them back into RAX and RBX, takes the frame down again and returns. This is the
same prologue and epilogue pattern that compilers generate for functions.

What makes this program interesting is that four different instructions move RSP here,
and only two of them are push and pop.

## Building and running it

```
nasm -f elf64 -g -F dwarf -o 02.o 02.asm
ld -o 02 02.o
./02 ; echo $?
```

Exits quietly with status 0.

## Starting state

```
initial rsp = 0x00007fffffff6510   ([rsp] = argc = 1)
rbp         = 0x0
```

RBP starts out as 0, which matters later: the `push rbp` in the prologue saves that
zero, and `pop rbp` puts it back at the end.

## Stepping through it

Each row is the state after the instruction ran.

| # | Instruction | RSP | Delta | What happened on the stack |
|---|---|---|---|---|
| 0 | `call work` | `0x7fffffff6508` | -8 | pushes the return address `0x401005`, then jumps to work |
| 1 | `push rbp` | `0x7fffffff6500` | -16 | saves the caller's RBP (0) |
| 2 | `mov rbp, rsp` | `0x7fffffff6500` | -16 | nothing written, RBP just becomes the frame anchor |
| 3 | `sub rsp, 16` | `0x7fffffff64f0` | -32 | reserves 16 bytes for the locals, writes nothing |
| 4 | `mov qword [rbp-8], 100` | `0x7fffffff64f0` | -32 | writes the first local |
| 5 | `mov qword [rbp-16], 200` | `0x7fffffff64f0` | -32 | writes the second local |
| 6 | `mov rax, [rbp-8]` | `0x7fffffff64f0` | -32 | reads local 1, RAX = 100 |
| 7 | `mov rbx, [rbp-16]` | `0x7fffffff64f0` | -32 | reads local 2, RBX = 200 |
| 8 | `nop` | `0x7fffffff64f0` | -32 | frame is fully built here, good place to stop and look |
| 9 | `mov rsp, rbp` | `0x7fffffff6500` | -16 | throws away both locals in one instruction |
| 10 | `pop rbp` | `0x7fffffff6508` | -8 | restores the caller's RBP back to 0 |
| 11 | `ret` | `0x7fffffff6510` | 0 | pops `0x401005` into RIP, so we are back in _start |
| 12 | `mov rax, 60` | `0x7fffffff6510` | 0 | |
| 13 | `xor rdi, rdi` | `0x7fffffff6510` | 0 | |
| 14 | `syscall` | `0x7fffffff6510` | 0 | exits |

## The frame

At the nop (step 8) the frame is complete. I printed it in gdb:

```
rbp            = 0x7fffffff6500
rsp            = 0x7fffffff64f0   (rbp - rsp = 16 bytes of locals)
[rbp+8]  ret       = 0x0000000000401005   pushed by call
[rbp+0]  saved rbp = 0x0000000000000000   pushed by push rbp
[rbp-8]  local1    = 100
[rbp-16] local2    = 200
rax=100 rbx=200
```

Drawn out:

```
          higher addresses
  0x7fffffff6510   argc = 1                    caller's stuff, untouched
  0x7fffffff6508   return address 0x401005     pushed by call      [rbp+8]
  0x7fffffff6500   saved RBP = 0               pushed by push rbp  [rbp+0]  <== RBP
  0x7fffffff64f8   local1 = 100                                    [rbp-8]
  0x7fffffff64f0   local2 = 200                                    [rbp-16] <== RSP
          lower addresses
```

Two things about this layout:

RBP sits in the middle and splits the frame. Positive offsets from RBP reach things the
caller put there (the return address at [rbp+8], and in a real function the arguments
would be at [rbp+16], [rbp+24] and so on). Negative offsets reach this function's own
locals.

RBP does not move for the whole body of the function, but RSP can. That is the reason
the prologue exists at all. `[rbp-8]` means the same byte on every line of `work`,
whereas if you addressed locals off RSP you would have to recount the offsets after
every push.

## Four different ways RSP moves

| Instruction | Moves RSP by | Writes memory? |
|---|---|---|
| `call work` | -8 | yes, the return address |
| `push rbp` | -8 | yes, the old RBP |
| `sub rsp, 16` | -16 | no, it only reserves space |
| `mov rsp, rbp` | +16 (to wherever RBP is) | no |
| `pop rbp` | +8 | no, it only reads |
| `ret` | +8 | no, it only reads |

`sub rsp, 16` is the one I had misread before doing this. It does not push anything and
it does not initialise anything. All it does is lower RSP so the 16 bytes above it
belong to this frame. gdb showed that right after the sub, both reserved slots were
just whatever happened to be in memory, which in this run was zeros:

```
after sub rsp, 16:
[rsp+0 ]=0x0000000000000000   becomes local2 later
[rsp+8 ]=0x0000000000000000   becomes local1 later
```

Only steps 4 and 5 put real values in. Reading a local before writing it would give
you garbage, and the fact that it reads as zero here is luck rather than a guarantee.

## Why sub rsp, 16 is needed

Without it RSP would still be equal to RBP, so `[rbp-8]` and `[rbp-16]` would be below
RSP, outside the frame. Any later push, or an interrupt or signal handler, could
legally write over that memory. The sub is what makes the locals safe to use.

## Why mov rsp, rbp is needed - I tested this

All three instructions in the epilogue matter. To check, I built a broken copy with
just `mov rsp, rbp` taken out:

```asm
    ; BROKEN: no mov rsp, rbp
    pop rbp
    ret
```

Running it:

```
$ ./v02
Segmentation fault
exit status: 139

(gdb) Program received signal SIGSEGV
rip  0x64            <- 100 in decimal
rsp  0x7fffffff6510
```

Following it through: RSP was still at `0x...64f0` pointing at the locals, so `pop rbp`
grabbed local2 (200) instead of the saved RBP, and then `ret` popped local1 (100) into
RIP. The CPU jumped to address 0x64 and the process died. RIP being 0x64 is the proof,
since that is literally the local variable being used as an address.

So `mov rsp, rbp` is not just tidying up. It is what lines RSP back up with the thing
that `ret` is about to read. The `leave` instruction does `mov rsp, rbp` and `pop rbp`
in one opcode, which is why you usually see that in compiler output instead.

## What I noticed from running it

**call is a push.** There is no push written before `work` in the source, but RSP
dropped 8 bytes at step 0 and `0x401005` appeared at [rsp]. `0x401005` is the address of
`mov rax, 60`, the instruction straight after the call. So call = push RIP, then jump.

**ret is a pop.** At step 11 RSP went up 8 and execution carried on at `0x401005`. ret
takes no operand because its operand is whatever is at [rsp].

**The frame cost 32 bytes in total.** 8 for the return address, 8 for the saved RBP, 16
for the locals. The deepest delta was -32.

**It unwinds symmetrically.** Delta goes `0, -8, -16, -32, -16, -8, 0`. The way down and
the way back up mirror each other, and that is what lets functions nest as deep as you
like.

**RBP was restored properly.** It was 0 in _start, became `0x7fffffff6500` inside work,
and was back to 0 after `pop rbp`. If a function wrecks the caller's RBP then every
frame outside it breaks, so saving and restoring it is part of the deal.

**RAX and RBX survived the return.** After ret they still held 100 and 200. The frame's
memory got abandoned but the register values crossed the boundary, which is how return
values get back to the caller.

**The locals' bytes still existed after the epilogue.** Taking a frame down just moves
RSP, it does not wipe anything. Straight after `mov rsp, rbp`, 100 and 200 were still
at `0x...64f8` and `0x...64f0`, they were just not owned any more. This is where
uninitialised variable bugs come from, because the next function's locals land on top
of the previous function's leftovers.

**mov rbp, rsp touched no memory at all.** RSP was identical before and after step 2.
It only copies a register, it just changes how the stack gets interpreted.

## Summary

```
_start:  RSP 6510
  call work --push return address--> 6508
    push rbp ----------------------> 6500   RBP := 6500, frame anchored
    sub rsp,16 --------------------> 64f0   locals [rbp-8] and [rbp-16] now owned
      write 100 and 200, read them back into RAX and RBX
    mov rsp,rbp -------------------> 6500   locals released
    pop rbp -----------------------> 6508   caller's RBP back to 0
  ret --pop into RIP--------------- > 6510   back at 0x401005
exits with RSP exactly where it started
```

The stack is doing three jobs at the same time in this program: remembering where to
return to, keeping the caller's RBP safe, and giving `work` some scratch space for its
locals. All three are tracked by the same pointer, which is why the push and pop counts
have to match exactly.
