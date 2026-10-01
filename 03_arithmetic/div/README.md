# Division — EFLAGS Analysis

## div1.asm — 8-bit Unsigned Division (100 ÷ 7)

**Operation:** `div bl` where `AX = 100`, `BL = 7`
**Result:** `AL = 14` (quotient), `AH = 2` (remainder)
**Verification:** 14 × 7 + 2 = 98 + 2 = 100 ✓

### GDB Output

```
eflags  0x212  [ AF IF ]
$1 = 14
$2 = 2
```

### Flag Analysis

| Flag | Status  | Explanation |
|------|---------|-------------|
| CF   | Undefined | The Intel manual states that **all arithmetic flags (CF, ZF, SF, OF, PF, AF) are undefined after a DIV instruction**. The CPU does not guarantee any particular flag state after division. |
| ZF   | Undefined | Same as above — undefined after DIV. |
| SF   | Undefined | Same as above — undefined after DIV. |
| OF   | Undefined | Same as above — undefined after DIV. |
| PF   | Undefined | Same as above — undefined after DIV. |
| AF   | Undefined | GDB shows AF as set, but this is not meaningful. AF is undefined after DIV — the value shown is simply whatever the CPU left behind from its internal operations, not a defined result. |

**Why are flags undefined?** Unlike ADD and SUB, the DIV instruction does not set flags in any architecturally defined way. The processor internally performs complex multi-step operations to compute the quotient and remainder. Rather than define flag behaviour for such a complex operation, Intel leaves them undefined. Programs that need to check properties of a division result (e.g., whether the quotient is zero) must use a separate `test` or `cmp` instruction afterward.

**Note:** GDB shows `[ AF IF ]` — the Interrupt Flag (IF) is always set during normal execution and has nothing to do with arithmetic. AF appears set but is undefined and unreliable after DIV.

---

## div2.asm — 16-bit Unsigned Division (50000 ÷ 300)

**Operation:** `div bx` where `DX:AX = 0:50000`, `BX = 300`
**Result:** `AX = 166` (quotient), `DX = 200` (remainder)
**Verification:** 166 × 300 + 200 = 49,800 + 200 = 50,000 ✓

### GDB Output

```
eflags  0x212  [ AF IF ]
$1 = 166
$2 = 200
```

### Flag Analysis

| Flag | Status  | Explanation |
|------|---------|-------------|
| CF, ZF, SF, OF, PF, AF | All Undefined | As with div1, the DIV instruction leaves all arithmetic flags in an undefined state. The flag values shown by GDB after DIV are not meaningful and must not be relied upon. AF appears set in GDB output, but this is just residual CPU state, not a defined result of the division. |

**Key concept for division:** The DIV instruction can raise a **Divide Error exception** (#DE, interrupt 0) in two cases: (1) division by zero, or (2) when the quotient is too large to fit in the destination register. For `div bx`, the quotient must fit in AX (0–65535). If DX:AX were large enough relative to BX that the quotient exceeded 65535, the program would crash with a floating-point exception rather than producing incorrect results. This exception mechanism replaces the need for flag-based error detection.