# MUL: EFLAGS observations

I assembled and ran `mul1.asm`, `mul2.asm` and `mul3.asm` as 32 bit programs and checked EFLAGS in GDB right after the MUL instruction.

## How MUL treats the flags

MUL is different from ADD and SUB. The product of two n bit numbers can need up to 2n bits, so MUL puts the answer in a double size register:

| Operand size | Multiplies | Result goes to |
|--------------|------------|----------------|
| byte | AL × operand | AX |
| word | AX × operand | DX:AX |
| dword | EAX × operand | EDX:EAX |

Because of that, MUL only **defines two flags**:

- **CF and OF are both set** if the upper half of the result (AH, DX or EDX) is not zero, meaning the answer needed the extra register.
- **CF and OF are both cleared** if the upper half is zero, meaning the whole answer fits in AL, AX or EAX.

CF and OF always have the same value after MUL.

SF, ZF, AF and PF are **undefined** after MUL according to the Intel manual. The CPU still leaves something in them (you can see PF and SF in GDB below) but a program should not rely on them. They can differ between processors.

IF is always set by the OS. The PF and ZF after `xor ebx, ebx` at the end of each program come from the XOR.

---

## mul1.asm: `mul byte [num2]` (8 bit)

25 × 10 = 250

```
AL = 25 = 0x19
     × 10 = 0x0A
AX = 250 = 0x00FA   (AH = 0x00, AL = 0xFA)
```

GDB after the MUL: `eflags = [ PF SF IF ]`, AX = 0x00FA

| Flag | Status | Why |
|------|--------|-----|
| CF | cleared | AH = 0, the product 250 fits in the lower byte AL (max 255) |
| OF | cleared | same reason as CF, MUL always sets them together |
| SF | set (undefined) | not defined for MUL. This processor seems to have set it from bit 7 of AL (0xFA = 11111010, top bit 1), but it should not be trusted |
| PF | set (undefined) | also not defined. It happens to match the parity of 0xFA (six 1 bits, even) |
| ZF | cleared (undefined) | not defined for MUL |
| AF | cleared (undefined) | not defined for MUL |

The important thing is CF = OF = 0, which tells us we can just use AL as the answer and ignore AH.

## mul2.asm: `mul word [num2]` (16 bit)

3000 × 200 = 600000

```
AX     = 3000   = 0x0BB8
       × 200    = 0x00C8
DX:AX  = 600000 = 0x0009 : 0x27C0
```

GDB after the MUL: `eflags = [ CF PF IF OF ]`, AX = 0x27C0, DX = 0x0009

| Flag | Status | Why |
|------|--------|-----|
| CF | set | 600000 is bigger than 65535, so it does not fit in AX alone. The top part (9) went into DX, and DX is not zero |
| OF | set | always the same as CF after MUL |
| PF | set (undefined) | not defined for MUL |
| SF | cleared (undefined) | not defined for MUL |
| ZF | cleared (undefined) | not defined for MUL |
| AF | cleared (undefined) | not defined for MUL |

To check: 0x927C0 = 9 × 65536 + 10176 = 589824 + 10176 = 600000. This is why the program stores both AX and DX into the 32 bit `result`. If you only kept AX you would get 10176, which is wrong, and CF = 1 is the warning for that.

## mul3.asm: `mul dword [num2]` (32 bit)

100000 × 300000 = 30 000 000 000

```
EAX      = 100000          = 0x000186A0
         × 300000          = 0x000493E0
EDX:EAX  = 30000000000     = 0x00000006 : 0xFC23AC00
```

GDB after the MUL: `eflags = [ CF PF SF IF OF ]`, EAX = 0xFC23AC00, EDX = 0x00000006

| Flag | Status | Why |
|------|--------|-----|
| CF | set | 30 billion is far bigger than 4 294 967 295 (the max for 32 bits), so EDX holds the upper part (6) and is not zero |
| OF | set | same as CF |
| SF | set (undefined) | not defined for MUL, it just happens that bit 31 of EAX is 1 |
| PF | set (undefined) | not defined for MUL |
| ZF | cleared (undefined) | not defined for MUL |
| AF | cleared (undefined) | not defined for MUL |

Check: 6 × 4294967296 = 25769803776, plus 0xFC23AC00 (4230196224) = 30000000000. The 64 bit `result` holds the full correct answer only because the program saves EDX as well as EAX.

---

## Summary

| Program | Operation | Upper half | CF | OF | Other flags |
|---------|-----------|------------|----|----|-------------|
| mul1 | 25 × 10 (8 bit) | AH = 0 | 0 | 0 | undefined |
| mul2 | 3000 × 200 (16 bit) | DX = 9 | 1 | 1 | undefined |
| mul3 | 100000 × 300000 (32 bit) | EDX = 6 | 1 | 1 | undefined |

In short: after MUL only look at CF/OF. If they are 0 the answer is in the low register only. If they are 1 you need the high register too.
