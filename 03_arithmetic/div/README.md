# DIV: EFLAGS observations

I assembled and ran `div1.asm`, `div2.asm` and `div3.asm` as 32 bit programs and checked EFLAGS in GDB before and after the DIV instruction.

## How DIV treats the flags

DIV is unsigned division. The dividend is double size and the answer is split into a quotient and a remainder:

| Divisor size | Dividend | Quotient | Remainder |
|--------------|----------|----------|-----------|
| byte | AX | AL | AH |
| word | DX:AX | AX | DX |
| dword | EDX:EAX | EAX | EDX |

According to the Intel manual, **all of CF, OF, SF, ZF, AF and PF are undefined after DIV**. DIV is not meant to report anything through the flags. Instead, if something goes wrong (dividing by zero, or the quotient is too big to fit in AL/AX/EAX) the CPU raises a divide error exception and the program crashes, on Linux you get "Floating point exception".

So for DIV the flags do not tell us anything about the result. The point of these programs is to see where the quotient and remainder end up.

On this machine the arithmetic flags were all clear before DIV, and they were still all clear afterwards in all three programs. That is just what this processor left there, it is not a guarantee.

IF is set by the OS in every program. The PF and ZF that appear at the very end come from `xor ebx, ebx`, not from DIV.

---

## div1.asm: `div bl` (8 bit divisor)

100 ÷ 7 = 14 remainder 2

```
AX = 100 = 0x0064
BL = 7
AL = 14 = 0x0E   (quotient)
AH = 2  = 0x02   (remainder)
AX after DIV = 0x020E
```

GDB after the DIV: `eflags = [ IF ]`, EAX = 0x0000020E

| Flag | Status | Why |
|------|--------|-----|
| CF | cleared (undefined) | DIV does not define CF |
| OF | cleared (undefined) | DIV does not define OF |
| SF | cleared (undefined) | DIV does not define SF |
| ZF | cleared (undefined) | DIV does not define ZF, even though it would be tempting to think "remainder is not zero so ZF = 0" |
| AF | cleared (undefined) | DIV does not define AF |
| PF | cleared (undefined) | DIV does not define PF |

Check: 14 × 7 + 2 = 100. Note that AX = 0x020E reads as 526 if you look at it as one number, which is why you have to read AH and AL separately.

## div2.asm: `div bx` (16 bit divisor)

50000 ÷ 300 = 166 remainder 200

```
DX:AX = 0x0000 : 0xC350   (50000)
BX    = 300
AX    = 166 = 0x00A6      (quotient)
DX    = 200 = 0x00C8      (remainder)
```

GDB after the DIV: `eflags = [ IF ]`, EAX = 0x000000A6, EDX = 0x000000C8

| Flag | Status | Why |
|------|--------|-----|
| CF, OF, SF, ZF, AF, PF | all cleared (undefined) | DIV leaves all six flags undefined. They were clear before the DIV and the processor left them clear |

Check: 166 × 300 + 200 = 49800 + 200 = 50000. The program sets DX to 0 first on purpose. A 16 bit DIV always divides DX:AX, so if DX had leftover junk in it the dividend would be a huge number and the quotient might not fit in AX, which would crash the program instead of setting any flag.

## div3.asm: `div ebx` (32 bit divisor)

300 000 000 ÷ 1000 = 300 000 remainder 0

```
EDX:EAX = 0x00000000 : 0x11E1A300   (300000000)
EBX     = 1000
EAX     = 300000 = 0x000493E0       (quotient)
EDX     = 0                          (remainder)
```

GDB after the DIV: `eflags = [ IF ]`, EAX = 0x000493E0, EDX = 0x00000000

| Flag | Status | Why |
|------|--------|-----|
| ZF | cleared (undefined) | the remainder is 0 here, but ZF is still clear. This proves DIV does not set ZF based on the result, the flag is just undefined |
| CF, OF, SF, AF, PF | cleared (undefined) | DIV does not define these |

This program is the best example in the folder. If ZF worked like it does for ADD or SUB, a zero remainder would set it. It did not, so you have to check the remainder with something like `cmp edx, 0` or `test edx, edx` if you need to know.

---

## Summary

| Program | Operation | Quotient | Remainder | Flags after DIV |
|---------|-----------|----------|-----------|-----------------|
| div1 | 100 ÷ 7 (8 bit) | AL = 14 | AH = 2 | all undefined (observed clear) |
| div2 | 50000 ÷ 300 (16 bit) | AX = 166 | DX = 200 | all undefined (observed clear) |
| div3 | 300000000 ÷ 1000 (32 bit) | EAX = 300000 | EDX = 0 | all undefined (observed clear) |

In short: DIV does not report through the flags. Errors cause an exception, and anything you want to know about the quotient or remainder you have to test yourself afterwards.
