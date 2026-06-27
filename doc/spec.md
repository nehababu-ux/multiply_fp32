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
- Classify each operand according to the IEEE-754 floating-point classes (Normal, Subnormal, Zero, Infinity, and NaN).
- Checks operand classes using `a_is_nan`, `a_is_inf`, `a_is_zero`, etc. (derived from `a_r/b_r` fields).
- If either operand belongs to a special IEEE-754 class, determine the appropriate IEEE-754 result before normal multiplication. Subsequent arithmetic stages shall be bypassed for that operation, and the precomputed result shall be used during the final packing stage.
- For normal operation:
  - If exponent is nonzero, set the implicit leading 1 in the mantissa.
  - If exponent is zero (subnormal), treat the operand exponent as −126 while leaving the fraction unchanged for subsequent normalization.

> If verification is restricted to normal operands only:
> - `expA` and `expB` are always in the range 1..254.
> - Hidden-one insertion always occurs.
> - Special-case handling is not exercised during normal-operation verification, although the implementation shall still define IEEE-754 behavior for special operands.

### Stage 3 — Input normalization (lightweight)
- If the operand mantissa is nonzero and its MSB is not set, perform a single left shift and decrement the exponent by one.
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

### Stage 6 — Normalize + Round-to-Nearest-Even (RNE)
This stage performs the following operations in sequence:

1. Underflow alignment toward exponent −126.
   - When the unbiased exponent is below −126, determine the required right-shift amount.
   - Shift the mantissa right toward the denormal range while accumulating all shifted-out bits into the sticky bit.
   - Clamp the exponent to −126 after alignment.

2. Mantissa normalization.
   - If the mantissa is not normalized after alignment,
perform a single normalization step by left-shifting the mantissa by one position while decrementing the exponent by one.
   - Any Guard, Round, and Sticky information affected by the normalization shift shall be updated before the rounding decision is evaluated.

3. IEEE-754 Round-to-Nearest-Even.
   - Increment the mantissa when the Guard bit is set and `(Round || Sticky || LSB)` evaluates true.
   - If rounding produces a mantissa overflow, renormalize the mantissa and increment the exponent before packing.

### Stage 7 — Pack
- If a special IEEE-754 result was determined earlier in the pipeline, use that result directly.
- Otherwise:
  - Pack the sign, biased exponent, and fraction into the IEEE-754 binary32 format.
  - If the exponent indicates overflow, output Infinity.
  - If the unbiased exponent reaches the denormal boundary after normalization and rounding, encode the result with an exponent field of zero while preserving the computed fraction.
- Assert `out_valid` for one clock cycle.
- Clear `busy` and return the FSM to the idle state.

---

## Assumptions & Constraints
- Verification is primarily performed using normal operands (`exp ∈ [1..254]`). Although the implementation defines IEEE-754 behavior for Zero, Subnormal, Infinity, and NaN operands, these cases are outside the primary verification scope.

---

## Verification Notes
Recommended testbench behavior for this handshake design:
- Drive `a/b` and pulse `valid` **synchronously** on clock edges.
- Wait for `out_valid` before sampling `z`.
- Generate only normal operands,

---
