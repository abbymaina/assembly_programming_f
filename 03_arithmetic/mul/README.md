# Multiplication — EFLAGS Analysis

## mul1.asm — 8-bit Unsigned Multiply (25 × 10)

**Operation:** `mul byte [num2]` where `AL = 25`, `[num2] = 10`
**Result:** `AX = 0x00FA` (250) — result fits entirely in AL, AH = 0x00

### GDB Output

```
eflags  0x202  [ IF ]
$1 = 0xfa
$2 = 0x0
```

### Flag Analysis

| Flag | Status  | Explanation |
|------|---------|-------------|
| CF   | Cleared (0) | For `MUL byte`, CF is set only when the result extends into AH (the upper half). Here, 25 × 10 = 250, which fits in AL alone (AH = 0). Since the upper half is zero, CF is cleared — the result does not need the extra register space. |
| OF   | Cleared (0) | OF follows the same rule as CF for MUL: it is set when the upper half of the result is non-zero. Since AH = 0, OF is also cleared. |
| ZF, SF, PF, AF | Undefined | The Intel manual states that ZF, SF, PF, and AF are **undefined** after a MUL instruction. The CPU does not guarantee any meaningful values for these flags after multiplication. GDB shows none of them set here, but this is not a guaranteed state — only CF and OF have defined behaviour after MUL. |

**Key concept:** For MUL, CF and OF together tell you whether the result overflowed the lower half of the destination. CF = OF = 0 means the result fits in the smaller register (AL for byte multiply). CF = OF = 1 means you need the full double-width result.

---

## mul2.asm — 16-bit Unsigned Multiply (3000 × 200)

**Operation:** `mul word [num2]` where `AX = 3000`, `[num2] = 200`
**Result:** `DX:AX = 0x0009:0x27C0` → 600,000 in decimal

### GDB Output

```
eflags  0xa03  [ CF IF OF ]
$1 = 0x27c0
$2 = 0x9
```

### Flag Analysis

| Flag | Status  | Explanation |
|------|---------|-------------|
| CF   | **Set (1)** | For `MUL word`, CF is set when the result does not fit in AX alone — meaning DX is non-zero. Here, 3000 × 200 = 600,000, which exceeds the 16-bit maximum of 65,535. The upper 16 bits spill into DX (= 0x0009), so CF is set. |
| OF   | **Set (1)** | OF follows the same rule as CF for MUL. Since DX ≠ 0, the result requires the full 32-bit DX:AX pair. OF is set to indicate this. |
| ZF, SF, PF, AF | Undefined | These flags are undefined after MUL per the Intel manual. GDB may show arbitrary values, but they are architecturally meaningless and should not be interpreted. |

**Key concept:** CF = OF = 1 here tells us the product overflowed a single 16-bit register. The program correctly stores both halves: `mov [result], ax` for the lower 16 bits and `mov [result+2], dx` for the upper 16 bits, capturing the full 32-bit result of 600,000.