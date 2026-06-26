# fmultiplier — FP32 Multiplier (Handshake, Multi-Cycle, IEEE-754)

## Overview
`fmultiplier` is a **multi-cycle** single-precision floating-point multiplier that accepts one operation at a time using a **valid/out_valid** handshake. Internally it runs a staged pipeline controlled by a small FSM (`counter`) and produces a 32-bit IEEE-754 binary32 result.

This design currently targets:
- **Bit-accurate results for normal FP32 numbers** (typical IEEE-754 behavior with round-to-nearest-even),
- Deterministic latency (fixed number of cycles from `valid` to `out_valid`),
- The design behaves as: z = a*b 
- z, a and b are single precision 32-bit IEEE-754 numbers

---

## Interface

### Ports
| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `clk`   | in | 1 | Clock |
| `rst`   | in | 1 | Async reset (posedge) |
| `valid` | in | 1 | **1-cycle start pulse**; accepted only when not busy |
| `a`     |  in | 32 | Operand A (FP32 bits) |
| `b`     | in | 32 | Operand B (FP32 bits) |
| `z`         | out | 32 | Result (FP32 bits) |
| `out_valid` | out | 1 | **1-cycle pulse** when `z` is updated/valid |

### Handshake contract
- When `busy==0`, a high `valid` on a rising edge **starts** an operation:
  - `a` and `b` are **registered** into internal regs `a_r` and `b_r`.
  - The FSM begins at `counter = 1`.
- While `busy==1`, new `valid` pulses are **ignored**.
- When the operation completes:
  - `z` is updated,
  - `out_valid` pulses high for 1 clock cycle,
  - `busy` is cleared.

---

## Latency and Throughput

### Latency
- Fixed latency of **7 stages**.
- In this implementation the operation begins at stage `counter=1` and completes at `counter=7`.
- `out_valid` asserts on the cycle where stage 7 packing finishes.

A safe expectation for system-level timing is:
- **`out_valid` occurs 7 clock cycles after the start edge** (the clock edge where `valid` was sampled when idle).

### Throughput
- **Not pipelined** (single-issue).
- Max throughput is **1 result per 7 cycles** (assuming `valid` is asserted only when idle).

---

## Internal Data Model (IEEE-754 binary32)
For each operand:
- `sign` = bit 31
- `exp`  = bits 30:23 (biased exponent)
- `mant` = bits 22:0 (fraction)

Internal signals:
- `a_s, b_s, z_s`: sign bits
- `a_e, b_e, z_e`: signed exponent in *unbiased* domain (stored as 10-bit regs, used with `$signed`)
- `a_m, b_m, z_m`: mantissas extended to 24-bit with hidden 1 when applicable
- `product`: 50-bit product of mantissas
- `guard_bit`, `round_bit`, `sticky`: rounding support bits for RNE

---

## FSM / Pipeline Stages

The FSM is controlled by:
- `busy` (operation in progress)
- `counter` (stage number 1..7)

All stage actions are performed inside a single sequential always block using `case(counter)`.

### Stage 1 — Unpack
- Extract mantissas into 24-bit regs (initially `{1'b0, frac}`).
- Convert biased exponent into unbiased form: `exp - 127`.
- Capture signs.

### Stage 2 — Special classification + denormal setup
- Checks operand classes using `a_is_nan`, `a_is_inf`, `a_is_zero`, etc. (derived from `a_r/b_r` fields).
- Classify each operand according to the IEEE-754 floating-point classes (Normal, Subnormal, Zero, Infinity, and NaN) using signals derived from the registered operands.
- If either operand is NaN, store the canonical quiet NaN (`32'h7FC00000`) as the final result.
- If one operand is Infinity and the other is Zero, store the canonical quiet NaN (`32'h7FC00000`) as the final result.
- If either operand is Infinity, store a signed Infinity result with sign equal to `a_s ^ b_s`.
- If either operand is Zero, store a signed Zero result with sign equal to `a_s ^ b_s`.
- Store the selected IEEE-754 special result together with a special-case flag so that all remaining arithmetic stages bypass the normal datapath.
- Otherwise continue normal processing:
- If the exponent field is non-zero, set the implicit leading one (`a_m[23]` or `b_m[23]`) without modifying the remaining mantissa bits.
- If the exponent field is zero, treat the operand as subnormal by forcing its unbiased exponent to −126 while leaving the mantissa unchanged.
> If you restrict inputs to **normal numbers only**, then:
> - `expA` and `expB` are always 1..254,
> - hidden-one insertion always happens,
> - special logic is bypassed in practice.

### Stage 3 — Input normalization (lightweight)
- Execute this stage only when no special-case result has been detected.
- For each operand independently:
  - If the mantissa MSB (`bit 23`) is zero, perform exactly one left shift of the mantissa and decrement the unbiased exponent by one.
  - Otherwise leave the operand unchanged.
- No iterative normalization shall be performed in this stage.
- This is mainly relevant for denormal handling; for strictly normal inputs, this typically does nothing.

### Stage 4 — Multiply core
- Compute result sign: `z_s = a_s ^ b_s`
- Exponent add: `z_e = a_e + b_e + 1`
- Mantissa product: `product = a_m * b_m * 4`
  - The `*4` scaling aligns the product for extraction into `{z_m, G, R, S}`.

### Stage 5 — Extract mantissa + rounding bits
- `z_m = product[49:26]`
- `guard_bit = product[25]`
- `round_bit = product[24]`
- `sticky = OR(product[23:0])`
Note:Execute this stage only when no special-case result has been detected.
### Stage 6 — Normalize + Round-to-Nearest-Even (RNE)
This stage performs:
Execute this stage only when no special-case result has been detected.

This stage operates on temporary working copies of the mantissa, exponent, Guard, Round, and Sticky values. The updated values are written back only after all normalization and rounding decisions are complete.

The stage performs the following operations in order:

1. Underflow alignment:
   - If the unbiased exponent is less than −126, compute the required right-shift amount.
   - Right shift the mantissa.
   - Any discarded mantissa bits shall contribute to the Sticky bit.
   - Previous Guard and Round bits shall also contribute to Sticky after the shift.
   - Clamp the exponent to −126.
   - Clear Guard and Round after the shift.

2. Single-step normalization:
   - Otherwise, if the mantissa MSB is zero, perform exactly one left shift.
   - Decrement the exponent by one.
   - Shift the previous Guard bit into the mantissa LSB.
   - Move the previous Round bit into the Guard position.
   - Clear the Round bit.

3. IEEE-754 Round-to-Nearest-Even:
   - Increment the mantissa only when
     `Guard == 1` and `(Round || Sticky || LSB)`.

4. Rounding overflow:
   - Detect overflow from the increment operation using the carry-out of the addition.
   - If overflow occurs, set the mantissa to `24'h800000` and increment the exponent.

### Stage 7 — Pack
- If a special-case result was previously generated, write that stored result directly to the output.
- Otherwise:
  - Pack sign, biased exponent and fraction.
  - If the unbiased exponent equals −126 while the mantissa MSB is zero, encode the exponent field as zero.
  - If the unbiased exponent exceeds +127, output signed Infinity.
- Assert `out_valid` for one clock cycle.
- Clear `busy`.

---

## Assumptions & Constraints
- Inputs: `exp ∈ [1..254]` (no zeros/subnormals, no inf/nan)

---

## Verification Notes
Recommended testbench behavior for this handshake design:
- Drive `a/b` and pulse `valid` **synchronously** on clock edges.
- Wait for `out_valid` before sampling `z`.
- Generate only normal operands,

---
