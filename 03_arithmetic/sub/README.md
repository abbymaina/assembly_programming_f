# Subtraction — EFLAGS Analysis

## sub1.asm — 8-bit Subtraction (50 − 80)

**Operation:** `sub al, [num2]` where `AL = 50`, `[num2] = 80`
**Result:** `AL = 0xE2` (226 unsigned, −30 signed = `11100010b`)

### GDB Output

```
eflags  0x287  [ CF PF SF IF ]
$1 = 0xe2
```

### Flag Analysis

| Flag | Status  | Explanation |
|------|---------|-------------|
| CF   | **Set (1)** | We are subtracting a larger unsigned value from a smaller one (50 − 80). This requires a borrow — the unsigned result would be negative, which wraps around to 226 (0xE2). CF is set whenever a borrow is needed in subtraction. |
| ZF   | Cleared (0) | The result is 0xE2 (226), not zero. |
| SF   | **Set (1)** | Bit 7 of the result `11100010b` is 1. In signed interpretation, the result is −30, which is negative, so SF is set. |
| OF   | Cleared (0) | In signed terms, 50 − 80 = −30, which fits within the signed 8-bit range (−128 to +127). Signed overflow only occurs when the result falls outside that range. Here both operands are positive and the signed result −30 is still in range, so no overflow. |
| PF   | **Set (1)** | The low byte `11100010b` has 4 one-bits. Since 4 is even, PF is set. |
| AF   | Cleared (0) | Lower nibbles: `0x2 − 0x0 = 0x2`. No borrow is needed from bit 4 to bit 3, so AF is clear. |

---

## sub2.asm — 16-bit Subtraction (1000 − 2000)

**Operation:** `sub ax, [num2]` where `AX = 1000`, `[num2] = 2000`
**Result:** `AX = 0xFC18` (64536 unsigned, −1000 signed)

### GDB Output

```
eflags  0x287  [ CF PF SF IF ]
$1 = 0xfc18
```

### Flag Analysis

| Flag | Status  | Explanation |
|------|---------|-------------|
| CF   | **Set (1)** | Subtracting a larger unsigned value (2000) from a smaller one (1000) requires a borrow. The unsigned result wraps around: 1000 − 2000 = 64536 in unsigned 16-bit. CF indicates this borrow. |
| ZF   | Cleared (0) | The result is 0xFC18, not zero. |
| SF   | **Set (1)** | Bit 15 of 0xFC18 is 1. The result is negative in signed interpretation (−1000). |
| OF   | Cleared (0) | Signed: 1000 − 2000 = −1000, which fits within the signed 16-bit range (−32768 to +32767). No signed overflow occurred. |
| PF   | **Set (1)** | The low byte is 0x18 = `00011000b`, which has 2 one-bits. Since 2 is even, PF is set. |
| AF   | Cleared (0) | Lower nibbles: 0x8 − 0x0 = 0x8. No borrow from bit 4, so AF is clear. |