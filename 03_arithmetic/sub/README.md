# SUB and SBB: EFLAGS observations

I assembled and ran `sub1.asm`, `sub2.asm` and `sub3.asm` as 32 bit programs and stepped through them in GDB, checking EFLAGS straight after each subtraction.

For subtraction the flags mean nearly the same as for addition, with one big difference: **CF is the borrow flag**. It is set when the second number is bigger than the first when both are treated as unsigned, meaning the subtraction had to borrow from beyond the top bit.

| Flag | Meaning for SUB |
|------|-----------------|
| CF | a borrow was needed (unsigned first operand < unsigned second operand) |
| PF | low byte of the result has an even number of 1 bits |
| AF | a borrow was needed from bit 4 into bit 3 (low nibble) |
| ZF | result is zero |
| SF | top bit of the result |
| OF | signed result is wrong (out of range) |

IF is always set by the OS and is not changed by SUB. The PF and ZF that appear after `xor ebx, ebx` at the end come from the XOR, not from the subtraction.

---

## sub1.asm: `sub al, [num2]` (8 bit)

50 - 80 = -30

```
  00110010   (50)
- 01010000   (80)
----------
  11100010   (0xE2 = 226 unsigned, -30 signed)
```

GDB after the SUB: `eflags = [ CF PF SF IF ]`, AL = 0xE2

| Flag | Status | Why |
|------|--------|-----|
| CF | set | 50 is smaller than 80 as unsigned numbers, so the subtraction needs a borrow. The 0xE2 we get is really 256 + 50 - 80 |
| PF | set | 0xE2 = 11100010 has four 1 bits (even) |
| AF | cleared | low nibbles 0010 - 0000 = 0010, no borrow needed from bit 4 |
| ZF | cleared | result is not zero |
| SF | set | bit 7 is 1, and as signed the answer is -30, so it is negative |
| OF | cleared | as signed, 50 - 80 = -30 and -30 fits in the range -128 to 127, so the signed answer is correct |

So CF says "the unsigned answer is wrong" while OF = 0 says "the signed answer is fine". If you read 0xE2 as signed you get the right answer, -30.

## sub2.asm: `sub ax, [num2]` (16 bit)

1000 - 2000 = -1000

```
  0x03E8   (1000)
- 0x07D0   (2000)
--------
  0xFC18   (64536 unsigned, -1000 signed)
```

GDB after the SUB: `eflags = [ CF PF SF IF ]`, AX = 0xFC18

| Flag | Status | Why |
|------|--------|-----|
| CF | set | 1000 < 2000 unsigned, so a borrow out of bit 15 was needed |
| PF | set | only the low byte counts, 0x18 = 00011000, two 1 bits (even) |
| AF | cleared | low nibbles 0x8 - 0x0 = 0x8, no borrow |
| ZF | cleared | result is not zero |
| SF | set | bit 15 is 1 (0xFC18 starts with 1111), the signed result is negative |
| OF | cleared | -1000 fits easily in signed 16 bit (-32768 to 32767) |

Same pattern as sub1 just with 16 bits. The flags are the same because in both cases a smaller positive number minus a bigger positive number gives a borrow and a negative answer, but nothing goes outside the signed range.

## sub3.asm: `sub ax, [num2]` then `sbb ax, 0` (16 bit)

**Step 1:** 0 - 1

```
  0000000000000000   (0)
- 0000000000000001   (1)
------------------
  1111111111111111   (0xFFFF = 65535 unsigned, -1 signed)
```

GDB after the SUB: `eflags = [ CF PF AF SF IF ]`, AX = 0xFFFF

| Flag | Status | Why |
|------|--------|-----|
| CF | set | 0 < 1 so a borrow was needed, the result wrapped around to 0xFFFF |
| PF | set | low byte 0xFF has eight 1 bits (even) |
| AF | set | low nibble 0x0 - 0x1 also needs a borrow from bit 4 |
| ZF | cleared | 0xFFFF is not zero |
| SF | set | bit 15 is 1, the signed value is -1 |
| OF | cleared | 0 - 1 = -1 is correct as a signed number |

**Step 2:** `sbb ax, 0` means AX = AX - 0 - CF = 0xFFFF - 0 - 1 = 0xFFFE

GDB after the SBB: `eflags = [ SF IF ]`, AX = 0xFFFE

| Flag | Status | Why |
|------|--------|-----|
| CF | cleared | 0xFFFF is bigger than 1, so no borrow was needed this time |
| PF | cleared | low byte 0xFE = 11111110 has seven 1 bits (odd) |
| AF | cleared | low nibble 0xF - 0x1 = 0xE, no borrow |
| ZF | cleared | result is not zero |
| SF | set | bit 15 is still 1, signed value is -2 |
| OF | cleared | -1 - 1 = -2 fits in signed 16 bit |

SBB subtracts the old borrow (CF) as well. That is the reason SBB exists: for numbers larger than one register you subtract the low parts with SUB and then the high parts with SBB so the borrow is carried across.

---

## Summary

| Program | Operation | Result | CF | PF | AF | ZF | SF | OF |
|---------|-----------|--------|----|----|----|----|----|----|
| sub1 | 50 - 80 (8 bit) | 0xE2 | 1 | 1 | 0 | 0 | 1 | 0 |
| sub2 | 1000 - 2000 (16 bit) | 0xFC18 | 1 | 1 | 0 | 0 | 1 | 0 |
| sub3 (sub) | 0 - 1 (16 bit) | 0xFFFF | 1 | 1 | 1 | 0 | 1 | 0 |
| sub3 (sbb) | 0xFFFF - 0 - CF | 0xFFFE | 0 | 0 | 0 | 0 | 1 | 0 |
