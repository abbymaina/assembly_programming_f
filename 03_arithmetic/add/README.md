# Addition — EFLAGS Analysis

## add1.asm — 8-bit Addition (120 + 10)

**Operation:** `add al, [num2]` where `AL = 120`, `[num2] = 10`
**Result:** `AL = 0x82` (130 unsigned, −126 signed = `10000010b`)

### GDB Output

```
eflags  0xa96  [ PF AF SF IF OF ]
$1 = 0x82
```

### Flag Analysis

| Flag | Status  | Explanation |
|------|---------|-------------|
| CF   | Cleared (0) | The result 130 fits within the unsigned 8-bit range (0–255). There is no carry out of bit 7, so CF remains clear. |
| ZF   | Cleared (0) | The result is 130, which is not zero. ZF is only set when the result equals zero. |
| SF   | **Set (1)** | The most significant bit (bit 7) of the result `10000010b` is 1. The CPU copies bit 7 into SF, so it is set. In signed interpretation, 130 is read as −126. |
| OF   | **Set (1)** | Both operands are positive in signed interpretation (120 and 10 both have bit 7 = 0), but the result 130 exceeds the signed 8-bit range (−128 to +127). The sign of the result differs from the sign of both inputs, which is the definition of signed overflow. |
| PF   | **Set (1)** | The low byte of the result is `10000010b`, which has 2 one-bits. Since 2 is even, the Parity Flag is set. PF is set when there is an even number of 1-bits in the low byte. |
| AF   | **Set (1)** | There is a carry from bit 3 to bit 4 during the addition. Lower nibble: `1000b + 1010b = 10010b` — the result exceeds 4 bits, producing a half-carry. |

---

## add3.asm — 16-bit Addition with Carry (0xFFFF + 1), then ADC

**Operation:** `add ax, [num2]` where `AX = 0xFFFF (65535)`, `[num2] = 1`
**Result after ADD:** `AX = 0x0000` (wraps around to zero)

The program then executes `adc ax, 0`, which adds 0 plus the Carry Flag from the previous ADD, recovering the carried bit so `AX = 0x0001`.

### GDB Output (after the ADD instruction)

```
eflags  0x257  [ CF PF AF ZF IF ]
$1 = 0x0
```

### Flag Analysis — after `add ax, [num2]`

| Flag | Status  | Explanation |
|------|---------|-------------|
| CF   | **Set (1)** | 0xFFFF + 1 = 0x10000, which requires 17 bits. The 17th bit (carry out of bit 15) does not fit in a 16-bit register, so it goes into CF. This is an unsigned overflow. |
| ZF   | **Set (1)** | The value stored in AX is 0x0000. Since the result in the register is zero, ZF is set. |
| SF   | Cleared (0) | Bit 15 of the result `0x0000` is 0, so the Sign Flag is clear. The result looks non-negative. |
| OF   | Cleared (0) | In signed terms, 0xFFFF is −1 and 1 is +1. Adding −1 + 1 = 0, which is a valid signed result. The operands have different signs, so signed overflow is impossible. |
| PF   | **Set (1)** | The low byte is `0x00`, which has 0 one-bits. Zero is even, so PF is set. |
| AF   | **Set (1)** | The lower nibble `0xF + 0x1 = 0x10` produces a carry from bit 3 to bit 4. |