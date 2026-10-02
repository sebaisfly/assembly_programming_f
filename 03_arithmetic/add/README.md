# ADD and ADC: EFLAGS observations

I assembled and ran `add1.asm`, `add2.asm` and `add3.asm` (32 bit, `nasm -f elf32` and `ld -m elf_i386`) and stepped through each one in GDB with `stepi`, checking `info registers eflags` right after the arithmetic instruction.

Flags I looked at:

| Flag | Meaning |
|------|---------|
| CF | Carry flag: unsigned result did not fit (carry out of the top bit) |
| PF | Parity flag: low byte of the result has an even number of 1 bits |
| AF | Auxiliary carry: carry out of bit 3 into bit 4 (the low nibble) |
| ZF | Zero flag: result is zero |
| SF | Sign flag: copy of the top bit of the result |
| OF | Overflow flag: signed result did not fit |

IF (interrupt flag) shows up in every GDB output. It is set by the operating system so the program can be interrupted, and ADD does not touch it, so I have left it out of the explanations below.

Also note: the `xor ebx, ebx` near the end of each program sets ZF and PF because the result is 0. Those flags come from the XOR, not from the addition.

---

## add1.asm: `add al, [num2]` (8 bit)

120 + 10 = 130

```
  01111000   (120)
+ 00001010   (10)
----------
  10000010   (0x82 = 130 unsigned, -126 signed)
```

GDB after the ADD: `eflags = [ PF AF SF IF OF ]`, AL = 0x82

| Flag | Status | Why |
|------|--------|-----|
| CF | cleared | 130 is less than 255 so it fits in 8 bits as an unsigned number, there is no carry out of bit 7 |
| PF | set | 0x82 = 10000010 has two 1 bits, which is even |
| AF | set | low nibbles: 1000 + 1010 = 10010, so there is a carry out of bit 3 into bit 4 |
| ZF | cleared | the result is 0x82, not zero |
| SF | set | bit 7 of the result is 1 |
| OF | set | both operands are positive signed numbers (top bit 0) but the result has top bit 1, which is negative (-126). In signed 8 bit the biggest value is 127, so 130 overflows |

This program is a good example that CF and OF are separate things. As unsigned it is fine (CF = 0) but as signed it is wrong (OF = 1).

## add2.asm: `add ax, [num2]` (16 bit)

32000 + 500 = 32500

```
  0x7D00   (32000)
+ 0x01F4   (500)
--------
  0x7EF4   (32500)
```

GDB after the ADD: `eflags = [ IF ]`, AX = 0x7EF4

| Flag | Status | Why |
|------|--------|-----|
| CF | cleared | 32500 is less than 65535 so there is no carry out of bit 15 |
| PF | cleared | parity only checks the low byte, 0xF4 = 11110100, which has five 1 bits (odd) |
| AF | cleared | low nibbles 0x0 + 0x4 = 0x4, no carry out of bit 3 |
| ZF | cleared | result is not zero |
| SF | cleared | bit 15 is 0 (0x7EF4 starts with 0111) |
| OF | cleared | 32500 is still under 32767, the largest signed 16 bit number, so the signed answer is correct |

Every arithmetic flag is clear here because the sum fits as both signed and unsigned. It is close to the signed limit though, adding just 268 more would set OF.

## add3.asm: `add ax, [num2]` then `adc ax, 0` (16 bit)

**Step 1:** 0xFFFF + 1

```
  1111111111111111   (0xFFFF = 65535, or -1 signed)
+ 0000000000000001
------------------
1 0000000000000000   (the 17th bit is lost)
```

GDB after the ADD: `eflags = [ CF PF AF ZF IF ]`, AX = 0x0000

| Flag | Status | Why |
|------|--------|-----|
| CF | set | 65535 + 1 = 65536 does not fit in 16 bits, the carry out of bit 15 goes into CF |
| PF | set | low byte is 0x00, zero 1 bits, and zero counts as even |
| AF | set | low nibble 0xF + 0x1 = 0x10, carry out of bit 3 |
| ZF | set | the 16 bits left in AX are all zero |
| SF | cleared | bit 15 of the result is 0 |
| OF | cleared | as signed this is -1 + 1 = 0, which is correct, so no signed overflow even though there was an unsigned carry |

**Step 2:** `adc ax, 0` means AX = AX + 0 + CF = 0 + 0 + 1 = 1

GDB after the ADC: `eflags = [ IF ]`, AX = 0x0001

| Flag | Status | Why |
|------|--------|-----|
| CF | cleared | 0 + 1 is tiny, no carry out of bit 15 |
| PF | cleared | 0x01 has one 1 bit (odd) |
| AF | cleared | no carry out of the low nibble |
| ZF | cleared | result is 1 |
| SF | cleared | bit 15 is 0 |
| OF | cleared | no signed overflow |

This shows what ADC is for. The carry from the first ADD was picked up and added back in. This is how you add numbers that are bigger than one register, you add the low parts with ADD and the high parts with ADC.

---

## Summary

| Program | Operation | Result | CF | PF | AF | ZF | SF | OF |
|---------|-----------|--------|----|----|----|----|----|----|
| add1 | 120 + 10 (8 bit) | 0x82 | 0 | 1 | 1 | 0 | 1 | 1 |
| add2 | 32000 + 500 (16 bit) | 0x7EF4 | 0 | 0 | 0 | 0 | 0 | 0 |
| add3 (add) | 0xFFFF + 1 (16 bit) | 0x0000 | 1 | 1 | 1 | 1 | 0 | 0 |
| add3 (adc) | 0 + 0 + CF | 0x0001 | 0 | 0 | 0 | 0 | 0 | 0 |

`adc4.asm` in this folder is empty so there was nothing to run.
