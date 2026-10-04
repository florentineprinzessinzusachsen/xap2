# XAP2 Processor Module

Ghidra processor module for the [CSR XAP2 processor](https://en.wikipedia.org/wiki/XAP_processor). Created to analyze the firmware for the [JBL Flip 3](https://www.amazon.com/JBL-FLIP3-Bluetooth-Speaker-Black/dp/B010RWAIAC).

There are quite a few things I guessed on or fudged here but it seems to work well enough for decompilation.

## Changes in this fork

- **Absolute addresses.** The 8-bit operand of `@H'nnnn` is signed, like immediates and offsets. It was treated as unsigned, which moved every address whose low byte is 0x80 or more up by 0x100 (and left unprefixed ones such as `@H'fff9` at `0x00f9` instead of `0xfff9`).
- **`AH:AL` is one 32-bit accumulator, `AH` high.** The register overlay had `al` in the high half. Shifts and rotates now act on all 32 bits, and `rol`/`ror` (rotate through carry) are decoded; `asr` had been decoded twice and `ror` not at all.
- **Multiply and divide.** `umult`/`smult` compute `AL * d` into `AH:AL`. `udiv`/`sdiv` divide `AH:AL` by `d`, quotient in `AL`, remainder in `AH`.
- **Flags.** After `sub`, `cmp`, `nadd` and `subc` the carry is the borrow. `addc`/`subc` include the carry-in. `s` is the signed less-than result, used by `blt`, `bge`, `bgt` and `ble`. `ble` and `bcz` used `&&` where they need `||`, so they could never be taken. Logic operations no longer touch carry.
- **Operand aliasing.** ALU instructions read both operands before writing, so `add ah,ah` computes its carry from the old value.
- **Opcode-0 loads and stores** (`ld`/`st` of flags, `uxl`, `uy`, `uxh`) address `@(off,y)`, and the `st` forms now store instead of load.
- **`brxl`** branches relative to the next instruction, by the signed value in `xl`.
- **Indirect branches and calls** (`@addr`, `@(off,y)`, `X+#off`) are `BRANCHIND`/`CALLIND`, with `xh` as the upper address byte. They were direct branches to a memory varnode, which made Ghidra's constant-propagation and stack analyzers fail for the whole program. `bra @(-1,y)` and `bra @(-3,y)` are function returns.
- **Flags are four separate one-byte registers** (`z`, `n`, `c`, `s`) instead of bits of one packed `flags` register. Every ALU instruction used to read and rewrite the packed register, so before dead-code removal each register write looked like it fed a later conditional branch, and the decompiler dropped call arguments it judged "not only used by the call". Call arguments found in the output rose from 83% to 95% (VM application) and from 70% to 97% (stack). `ld`/`st` of `flags` pack and unpack the four registers.
- **`bc` and `bc2`** are `blockcopy(dst, src, count)` user ops (`bc`: copy `AL` words from `[X]` to `[Y]`; `bc2`: from `[X]` to `[AH]`) instead of no-ops. `print x` and `print y` are decoded.

Not changed, and not verified against hardware: the calling convention (`al`, `ah`, the merged `a`, then the stack), N and Z after shifts (taken from `al`), and rotate counts other than 1. Known issue: Ghidra's decompiler does not terminate on one function I have seen (an inline `brxl` dispatch inside a loop) with this spec; about 7,000 others decompile.
