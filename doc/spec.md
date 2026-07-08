# fmultiplier — FP32 Multiplier (Handshake, Multi-Cycle, IEEE-754)

## Overview
`fmultiplier` is a **multi-cycle** single-precision floating-point multiplier that accepts one operation at a time using a **valid/out_valid** handshake. Internally it runs a 7 staged pipeline controlled by a small FSM (`counter`) and produces a 32-bit IEEE-754 binary32 result.

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
- The design shall have a fixed latency of exactly 7 clock cycles.
- An operation starts on the rising clock edge where valid is sampled while busy == 0.
- `out_valid` shall assert exactly 7 clock cycles after that start edge, and for one clock cycle only.
- The implementation shall not insert additional registered stages, wait states, or other sequential behavior that increases or decreases this latency.


### Throughput
- **Not pipelined** (single-issue).
- Max throughput is **1 result per 7 cycles** (assuming `valid` is asserted only when idle).

---

## Internal Data Model (IEEE-754 binary32)
For each operand:
- `sign` = bit 31
- `exp`  = bits 30:23 (biased exponent)
- `mant` = bits 22:0 (fraction)

Internal signals(below all signals should be declared as registers):
- `a_s, b_s, z_s`: sign bits
- `a_e, b_e, z_e`: signed exponent in *unbiased* domain (stored as 10-bit regs, use `$signed()`)
- `a_m, b_m, z_m`: mantissas extended to 24-bit with hidden 1 when applicable
- `product`: 50-bit product of mantissas
- `guard_bit`, `round_bit`, `sticky`: rounding support bits for RNE
- `special_case`: single bit flag which is set when special case operands(NaN,Inf,Zero)are given as input
- `special_z`: the 32 bit product for special case operands are stored here. 
---

## FSM / Pipeline Stages

The FSM is controlled by:
- `busy` (operation in progress)
- `counter` (stage number 1..7)

All stage actions are performed inside a single sequential always block using `case(counter)`.

### Stage 1 — Unpack(extract the below
- Extract mantissas into 24-bit regs (initially `{1'b0, frac}`).
- Convert biased exponent into unbiased form: `exp - 10'd127`.
- Capture signs.

### Stage 2 — Special classification + denormal setup
- Checks operand classes using `a_is_nan`, `a_is_inf`, `a_is_zero`, etc. (Declare them as wires derived from `a_r/b_r` fields) before evaluating the special-case path.
- If either operand belongs to a special IEEE-754 class(`special_case` is set), determine the appropriate IEEE-754 result(`special_z`). Subsequent arithmetic stages shall be bypassed for that operation, and the precomputed result shall be used during the final packing stage.
- Distinguish Zero and Subnormal operands using both the exponent and fraction fields.
  - Zero: exponent == 0 and fraction == 0.
  - Subnormal: exponent == 0 and fraction != 0.
- For normal operation(For each operand independently):
  - If exponent is nonzero => sets implicit leading 1: `a_m[23] = 1`.
  - If exponent is zero (subnormal) => forces exponent to -126 (subnormal exponent baseline).

> If you restrict inputs to **normal numbers only**, then:
> - `expA` and `expB` are always 1..254,
> - hidden-one insertion always happens,
> - special logic is bypassed in practice.

### Stage 3 — Input normalization (lightweight)
- If mantissa MSB is not set, shift left and decrement exponent.
- This is mainly relevant for denormal handling; for strictly normal inputs, this typically does nothing.

### Stage 4 — Multiply core
- Compute result sign: `z_s = a_s ^ b_s`
- Exponent add: `z_e = a_e + b_e + 10'sd1`
- Mantissa product: `product = a_m * b_m * 50'd4`
  - The `*4` scaling aligns the product for extraction into `{z_m, G, R, S}`.

### Stage 5 — Extract mantissa + rounding bits
- `z_m = product[49:26]`
- `guard_bit = product[25]`
- `round_bit = product[24]`
- `sticky = OR(product[23:0])`

### Stage 6 — Normalize + Round-to-Nearest-Even (RNE)
This stage performs:
1. **Underflow alignment** toward exponent -126:
   - Computes shift amount `sh = (-126 -$signed(z_e))` when `z_e < -126`.
   - Shift the mantissa toward the denormal range while updating the Guard, Round, and Sticky information to reflect all discarded bits.
   - Clamp the exponent to −126 after alignment.
2. **Normalize** if MSB missing:
   - Left-shifts mantissa while adjusting exponent,shifting the current guard bit into the mantissa LSB; the new guard bit takes the value of the current round bit; the round bit is then cleared to 0; sticky is unchanged.
3. **RNE rounding**:
   - If `G == 1` and `(R || S || LSB)` then increment mantissa.
   - Handles carry-out from rounding:
     - If rounding overflows mantissa, set mantissa to 0x800000 and increment exponent.

### Stage 7 — Pack
- For normal path:
  - Pack sign, biased exponent, fraction.
  - If exponent indicates overflow -> output INF.
  - A result shall be encoded as a denormal only when the unbiased exponent is at the denormal boundary and the significand is not normalized(i.e `z_m[23]` is zero).
- Asserts `out_valid` for one cycle and clears `busy`.

---

## Assumptions & Constraints
- Inputs: `exp ∈ [1..254]` (no zeros/subnormals, no inf/nan)

---

## Verification Notes
Recommended testbench behavior for this handshake design:
- Drive `a/b` and pulse `valid` **synchronously** on clock edges.
- Wait for `out_valid` before sampling `z`.After `z` is updated `out_valid` becomes high for one clock cycle at the end of stage 7.
- Generate only normal operands(exponent field in `[1, 254]` and mantissa can be random).

---
