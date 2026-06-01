![GDS](../../workflows/gds/badge.svg) ![Docs](../../workflows/docs/badge.svg) ![Test](../../workflows/test/badge.svg)

# Bezier Curve VGA Rasterizer
### Tiny Tapeout — Sky130A · 25 MHz · *2×1 tile target (pending submission)*

> **A real-time quadratic Bézier curve rasterizer implemented entirely in synthesisable Verilog, targeting the Tiny Tapeout ASIC shuttle. No CPU, no framebuffer — pure combinatorial and sequential logic driving a live 640×480 VGA signal.**
>
> ⚠️ **Shuttle status:** The current tapeout run only supported 1×1 tiles, which this design overflows. The RTL is complete and GDS-verified at 2×1, and will be submitted on the next compatible shuttle run.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
  - [VGA Signal Generation](#vga-signal-generation)
  - [Coordinate System](#coordinate-system)
  - [Forward-Difference Algorithm](#forward-difference-algorithm)
  - [Scanline Row Buffer](#scanline-row-buffer)
  - [Control Interface](#control-interface)
- [Pin Assignment](#pin-assignment)
- [How to Use (Hardware)](#how-to-use-hardware)
- [How to Use (Simulation)](#how-to-use-simulation)
- [Design Constraints & Area](#design-constraints--area)
- [Future Learning & Extensions](#future-learning--extensions)
- [Silicon-Proven Results](#silicon-proven-results)

---

## Overview

This project renders a single quadratic Bézier curve to a **640×480 VGA display** in real-time, targeting a **2×1 Tiny Tapeout tile** (~33,000 µm² core area, Sky130A PDK). The design is complete and GDS-verified, but has not yet been submitted to a shuttle — the tapeout run this was initially developed for only supported 1×1 tiles, which the design outgrew. Submission to a 2×1-compatible run is planned next.

The key constraint is that Sky130A provides roughly **~1,600 standard-cell flip-flops per mm²**. A naïve framebuffer for 640×480 pixels would require 307,200 bits — approximately **60× more flip-flops than the entire tile can hold**. This design solves the problem by eliminating the framebuffer entirely and re-computing each row of the curve on-the-fly during the horizontal blanking interval.

---

## Features

| Feature | Detail |
|---|---|
| **Curve type** | Quadratic Bézier — Q(t) = (1−t)²P₀ + 2t(1−t)P₁ + t²P₂ |
| **Resolution** | 160×120 logical pixels, each rendered as a 4×4 VGA block |
| **VGA output** | 640×480 @ 25 MHz, standard TinyVGA PMOD pinout |
| **Framebuffer** | None — single 160-bit row buffer, refilled every H-blank |
| **Rasterization** | 129-step forward-difference stepper, runs in 130 clock cycles |
| **Control** | 5 edge-triggered pins: Load X, Load Y, Add Point, Submit, Clear |
| **Tile size** | 2×1 Tiny Tapeout (≈ 334×108 µm core) — *pending shuttle submission* |
| **Process** | SkyWater Sky130A 130 nm |
| **Clock** | 25 MHz pixel clock |

---

## Architecture

### VGA Signal Generation

The design uses a standard `hvsync_generator` module from VGA Playground clocked at **25 MHz** (40 ns period), producing the full 640×480 @ 60 Hz VGA waveform:

```
Horizontal: 640 active + 16 front porch + 96 sync + 48 back porch = 800 total
Vertical:   480 active + 10 front porch +  2 sync + 33 back porch = 525 total
```

Sync polarity follows the TinyVGA PMOD standard — `hsync` and `vsync` are active-low and mapped directly to `uo_out[7]` and `uo_out[3]` respectively.

---

### Coordinate System

The logical canvas maps 160×120 pixels onto the 640×480 VGA frame, where each logical pixel occupies a **4×4 block** of physical VGA pixels:

```
Logical pixel (lx, ly)  →  VGA pixels (lx*4 .. lx*4+3, ly*4 .. ly*4+3)
X range: 0–159   (maps to VGA columns  0–639)
Y range: 0–119   (maps to VGA rows     0–479)
```

Control points are specified in logical coordinates, keeping the input interface to just **8 bits for X** and **7 bits for Y**.

---

### Forward-Difference Algorithm

Evaluating a Bézier curve naïvely requires two multiplications per step — impossible to do cheaply in hardware every clock cycle. Instead, this design uses the **method of forward differences**, which reduces evaluation to pure addition.

For a quadratic Bézier sampled at 129 uniformly spaced values of t (t = 0/128 … 128/128), the first and second finite differences are constant, allowing the following recurrence at each step:

```
d2  = constant (computed once from control points)
d1 += d2        (one addition per step)
d0 += d1        (one addition per step)
pixel = d0 >> 14
```

The initial values are derived combinatorially from the three control points before the scan begins:

| Register | Width | Role |
|---|---|---|
| `x_d0`, `y_d0` | 23-bit signed | Current scaled position (fixed-point, 14 fractional bits) |
| `x_d1`, `y_d1` | 18-bit signed | First difference (Δ per step) |
| `x_d2`, `y_d2` | 12-bit signed | Second difference (constant) |

No multipliers are synthesised at runtime — all setup arithmetic resolves to a short combinatorial adder chain that fires once per H-blank.

---

### Scanline Row Buffer

The row buffer is the heart of the no-framebuffer design:

```
┌─────────────┐   H-blank (160 cycles)    ┌──────────────────────┐
│  row_buf    │ ◄── 130 cycle scan fills ──┤  Forward-diff engine │
│  [159:0]    │                            │  (one step / cycle)  │
└──────┬──────┘                            └──────────────────────┘
       │  active video
       ▼
  pixel_on = row_buf[pix_x >> 2]   →   VGA output (white = curve)
```

**Timing budget per logical row:**

| Phase | Cycles | Budget |
|---|---|---|
| Active video (640 VGA cols) | 640 | — |
| Front porch | 16 | — |
| **H-blank available** | **160** | **target window** |
| Init (clear + load d0/d1/d2) | 1 | |
| Forward-difference steps × 129 | 129 | |
| **Total used** | **130** | **30 cycles margin** |

The trigger fires at `pix_x == 640` on the **last VGA row of each logical row** (when `pix_y[1:0] == 3`) and also on the **final VGA line of the frame** (`pix_y == 524`) to pre-fill row 0 for the next frame. This ensures the row buffer is always ready before the active display region begins.

---

### Control Interface

Points are staged and committed using five edge-triggered control pins on `uio_in`. All transitions are detected by comparing the current and previous cycle values, so pulses of exactly one clock cycle are sufficient.

```
┌──────────┬─────────┬────────────────────────────────────────────┐
│ Pin      │ Signal  │ Action on rising edge                      │
├──────────┼─────────┼────────────────────────────────────────────┤
│ uio_in[0]│ Load X  │ Latches ui_in[7:0] into staging register   │
│ uio_in[1]│ Load Y  │ Latches ui_in[6:0] into staging register   │
│ uio_in[2]│ Add Pt  │ Commits staged (X,Y) as P0 → P1 → P2 → P0 │
│ uio_in[3]│ Submit  │ Sets curve_active = 1 (curve appears)      │
│ uio_in[4]│ Clear   │ Sets curve_active = 0 (screen blanks)      │
└──────────┴─────────┴────────────────────────────────────────────┘
```

**Adding a complete curve (minimum sequence):**

```
ui_in ← X0  →  pulse Load X  →  ui_in ← Y0  →  pulse Load Y  →  pulse Add Point   (P0)
ui_in ← X1  →  pulse Load X  →  ui_in ← Y1  →  pulse Load Y  →  pulse Add Point   (P1)
ui_in ← X2  →  pulse Load X  →  ui_in ← Y2  →  pulse Load Y  →  pulse Add Point   (P2)
pulse Submit
```

Submitting a new set of three points while a curve is visible replaces it immediately on the next frame — no explicit clear required.

---

## Pin Assignment

### Inputs (`ui_in` — used as data bus)

| Bit | Function |
|---|---|
| `ui_in[7:0]` | X coordinate (0–159) when loading X |
| `ui_in[6:0]` | Y coordinate (0–119) when loading Y |

### Outputs (`uo_out` — TinyVGA PMOD)

| Bit | Signal |
|---|---|
| `uo_out[7]` | HSync (active-low) |
| `uo_out[6]` | B0 |
| `uo_out[5]` | G0 |
| `uo_out[4]` | R0 |
| `uo_out[3]` | VSync (active-low) |
| `uo_out[2]` | B1 |
| `uo_out[1]` | G1 |
| `uo_out[0]` | R1 |

Curve pixels output full white (`R=G=B=11`); all other pixels are black (`R=G=B=00`).

### Bidirectional (`uio_in` — used as inputs, `uio_oe = 0`)

| Bit | Function |
|---|---|
| `uio_in[0]` | Load X (rising edge) |
| `uio_in[1]` | Load Y (rising edge) |
| `uio_in[2]` | Add Point (rising edge) |
| `uio_in[3]` | Submit / Activate (rising edge) |
| `uio_in[4]` | Clear / Deactivate (rising edge) |
| `uio_in[7:5]` | Unused |

---

## How to Use (Hardware)

### Requirements

- Tiny Tapeout evaluation board
- [TinyVGA PMOD](https://tinytapeout.com/specs/pinouts/#tinyvga-pmod) connected to `uo` port
- VGA monitor (or HDMI adapter with VGA input)
- DIP switches or push-buttons wired to `uio_in[4:0]`

### Steps

1. Connect the TinyVGA PMOD to the `uo` output port.
2. Connect a VGA cable from the PMOD to your monitor.
3. Power on the board. The screen should be blank (black).
4. Set `ui_in` to the X coordinate of P0 (e.g. `00001010` = 10), then briefly pulse `uio_in[0]` (Load X).
5. Set `ui_in` to the Y coordinate of P0, pulse `uio_in[1]` (Load Y).
6. Pulse `uio_in[2]` (Add Point) to commit P0.
7. Repeat steps 4–6 for P1 and P2.
8. Pulse `uio_in[3]` (Submit) — the curve appears on screen within one frame (~16 ms).
9. To erase, pulse `uio_in[4]` (Clear).

---

## How to Use (Simulation)

Tests are written in Python using [cocotb](https://www.cocotb.org/) and [Icarus Verilog](http://iverilog.icarus.com/).

### Setup

```bash
# Install dependencies (Python 3.8+)
pip install cocotb

# Install Icarus Verilog
# macOS:   brew install icarus-verilog
# Ubuntu:  sudo apt install iverilog
```

### Running Tests

```bash
cd test
make
```

### Test Suite

| Test | Description |
|---|---|
| `test_reset_blank` | After reset with no curve submitted, all pixels must be black |
| `test_vga_sync_timing` | VSync and HSync pulse widths must match VGA specification |
| `test_straight_line_bezier` | Collinear P0/P1/P2 must produce a straight horizontal line |
| `test_curved_bezier` | Curved bezier pixels must be within ±1 logical pixel of software reference |
| `test_replace_curve` | Submitting a second curve must replace the first |
| `test_clear` | `ctrl_clr` must blank the display within one frame |
| `test_out_of_range_safe` | Out-of-range coordinates (X>159, Y>119) must not produce lit pixels |

---

## Design Constraints & Area

| Metric | Value |
|---|---|
| Tile size | 2×1 (≈ 334×108 µm) |
| PDK | SkyWater Sky130A |
| Target density | 60% |
| Clock period | 40 ns (25 MHz) |
| Synthesised cells | ~2,098 |
| Estimated cell area | ~25,162 µm² |
| Available core area | ~33,000 µm² |
| Estimated utilisation | ~76% |

The largest area contributors are the 160-bit row buffer flip-flops, the forward-difference registers (six 12–23-bit signed registers), and the mux trees for variable-indexed writes into the row buffer.

---

## Design Journey — What I Tried

Getting this design to fit on silicon involved several failed attempts before landing on the final architecture. Each iteration taught something different about the hard constraints of ASIC design.

---

### Iteration 1 — Naïve Full Framebuffer ❌

**The idea:** Store the entire rendered image in flip-flops — one bit per logical pixel at 160×120 resolution — and scan it out to the VGA signal each frame.

**What was built:**
```
160 × 120 = 19,200 flip-flop row  →  shift out pixel data during active video
Bezier evaluation runs once on submit, writes into the full buffer
```

**Why it failed:**

19,200 flip-flops at ~27.6 µm² each (Sky130A `dfrtp_2`) = **~530,000 µm²** of flip-flop area alone. The entire 1×1 Tiny Tapeout tile has only **16,493 µm²** of core area. The framebuffer was **32× larger than the whole chip** before a single gate of logic was added.

> **Lesson:** On a 130 nm process at these tile sizes, you cannot store even a low-resolution 1bpp image. Every register costs real silicon. Think in µm², not bytes.

---

### Iteration 2 — No Framebuffer, 1×1 Tile ❌

**The idea:** Eliminate the framebuffer entirely. Instead of storing the image, re-compute which pixels belong to the curve *every scanline* during the horizontal blanking interval (160 clock cycles). Keep only a 160-bit row buffer that is refilled from scratch each H-blank.

**What was built:**
```
160-bit row_buf  →  refilled during H-blank using forward-difference stepper
129 bezier steps × 1 cycle each = 130 cycles total  (fits inside 160-cycle H-blank)
Curve stored as 3 control points (3 × 8-bit X + 3 × 7-bit Y = 45 bits)
```

**Synthesis result on 1×1 tile:**
```
Synthesised cells   : 2,098
Total cell area     : 25,162 µm²
Available core area : 16,493 µm²
Utilisation         : 179.6%   ← GDS action fails: GPL-0301
```

**Why it still failed:**

The forward-difference registers (`x_d0`/`y_d0` at 23 bits, `x_d1`/`y_d1` at 18 bits, `x_d2`/`y_d2` at 12 bits) plus the 160-bit row buffer plus the mux trees for variable-indexed writes into `row_buf` pushed the design to **25,162 µm²** — still 52% over the 1×1 tile capacity even without a framebuffer.

Key area contributors beyond the flip-flops:
| Source | Approx. area |
|---|---|
| 346 × `dfrtp_2` FFs (row_buf + fwd-diff regs) | ~9,559 µm² |
| 160:1 mux tree for `row_buf[x_pix]` write | ~1,800 µm² |
| 160:1 mux tree for `row_buf[lx]` read | ~800 µm² |
| Adder chains (d1 init, d2 init, d0+d1, d1+d2) | ~6,000 µm² |
| Remaining combinatorial logic | ~7,000 µm² |

> **Lesson:** Even without a framebuffer, wide signed accumulators and variable bit-select mux trees are expensive. The synthesiser turns `row_buf[x_pix] <= 1'b1` (an 8-bit-indexed write into a 160-bit register) into a 160-input priority mux tree — every bit needs its own 2:1 mux gated by a comparator.

---

### Iteration 3 — No Framebuffer, 2×1 Tile ✅ *(GDS passes — not yet submitted)*

**The fix:** Upgrade from a 1×1 tile to a **2×1 tile** in `info.yaml`. This doubles the available core area to approximately **33,000 µm²**, giving 76% utilisation — within the 60–80% comfortable range for LibreLane place-and-route.

```yaml
# info.yaml
tiles: "2x1"   # was "1x1"
```

No RTL changes were required. The identical synthesised design that overflowed a 1×1 tile places and routes cleanly on a 2×1 tile.

**Verified numbers:**
```
Cell area     : 25,162 µm²
Core area     : ~33,000 µm²
Utilisation   : ~76%   ← GDS passes locally ✅
```

**Current status:** The tapeout run this project was developed against only accepted 1×1 tile designs, so this version was not submitted. The 2×1 configuration is ready and will be submitted on the next shuttle that supports it.

> **Lesson:** When combinatorial logic can't be shrunk further, the next lever is simply buying more area. On Tiny Tapeout, going from 1×1 to 2×1 doubles your budget and costs one additional tile slot in the shared die — a fair trade for a design that genuinely needs it.

---

## Future Improvements — Next Tapeout

The no-framebuffer architecture was a workaround forced by the 1×1 tile area constraint. The immediate next step is submitting the existing 2×1 design to a compatible shuttle run — no RTL changes needed, just a slot on the right tapeout. Beyond that, with even more silicon, the design could be significantly more capable.

### Getting a Real Framebuffer

The single biggest improvement for a future tapeout would be allocating enough area to store a complete rendered image. On Sky130A, a **1bpp 160×120 framebuffer** requires 19,200 flip-flops (~530,000 µm²) — equivalent to roughly **16× the current 2×1 tile**. That is achievable on a Tiny Tapeout **4×4 tile** (the largest standard size), which provides approximately 530,000–560,000 µm² of core area.

With a real framebuffer:

- The Bézier stepper would run **once per frame** during the vertical blanking interval (~1,430 idle cycles), eliminating the tight 160-cycle H-blank budget entirely.
- Multiple curves could be accumulated into the framebuffer over successive frames before displaying — enabling scenes rather than single curves.
- Rendering operations (fill, erase, blend) become possible since every pixel is persistently addressable.
- The control interface could accept a stream of curve commands rather than one curve at a time.

### Planned & Future Improvements

| Improvement | Tile requirement | Notes |
|---|---|---|
| **Submit current design to 2×1 shuttle** | 2×1 | RTL complete, GDS verified — just needs a compatible run |
| **True 1bpp framebuffer (160×120)** | 4×4 | Eliminates H-blank timing constraint entirely |
| **Greyscale framebuffer (160×120, 2bpp)** | 4×4+ | 38,400 FFs; enables anti-aliased rendering |
| **Multiple simultaneous curves (up to 8)** | 2×2 | Time-multiplex H-blank scanner; 8 × 130 = 1,040 cycles needed vs 160 available — requires V-blank rendering |
| **Cubic Bézier support** | 2×1 | Adds one extra accumulator register pair; fits in current tile with minor rework |
| **Filled curve (scanline span buffer)** | 2×2 | Store left/right X extents per row instead of a bitfield; ~240 bits total |
| **UART or SPI input** | 2×1 | Replace DIP-switch protocol; allows host computer to drive animations in real time |
| **Higher resolution canvas (320×240)** | 4×2 | Doubles the row buffer to 320 bits and requires 8× more framebuffer FFs |

### Architectural Note on Framebuffer vs. Scanline Rendering

This design deliberately trades **rendering flexibility for area efficiency**: because there is no persistent pixel store, the image cannot accumulate across frames, cannot be partially updated, and cannot store more information than a single row at a time. The trade-off was necessary to squeeze inside a 2×1 tile — itself the result of the 1×1 design overflowing.

A future design targeting a **4×4 tile** could adopt a conventional GPU-style split: a *geometry engine* that evaluates curves into the framebuffer during V-blank, and a *scan-out engine* that reads the framebuffer linearly during active video. These two paths are naturally non-conflicting and require no arbitration logic, keeping the design simple while enabling far richer output.

---

## Future Learning & Extensions

This project was designed as an introduction to digital ASIC design through a constrained but visual problem. Below are natural next steps, roughly ordered from beginner to advanced.

### Beginner

- **Multiple colours** — The VGA output currently drives R, G, B identically (white only). Storing a 2-bit colour index per control point and routing different colour channels per curve would add vibrancy with minimal area overhead.
- **Curve thickness** — Set a neighbourhood of `row_buf` bits around each plotted point (e.g. ±1) to render a thicker stroke.
- **Animated control points** — Add a counter-driven sine/cosine approximation to slowly move P1 each frame, producing smooth curve animation without any external input.

### Intermediate

- **Multiple simultaneous curves** — Time-multiplex the H-blank scanner across N curves per frame. With a 160-cycle budget and 130 cycles per curve, one additional curve fits with careful scheduling — no extra memory needed.
- **Cubic Bézier** — A cubic Q(t) = (1−t)³P₀ + 3t(1−t)²P₁ + 3t²(1−t)P₂ + t³P₃ adds a third-order finite difference register (`d3 = constant`), requiring only one extra accumulator at the cost of a slightly wider bit-width.
- **UART control interface** — Replace the DIP-switch protocol with a simple 8N1 UART receiver, allowing a host computer to stream new control points to the chip in real time.
- **Bresenham's line algorithm** — Implement an alternative rasterizer using integer-only arithmetic and compare the synthesised area cost against the forward-difference approach.

### Advanced

- **Scan-conversion to a tile map** — Instead of a 160-bit row buffer, store a compact run-length encoded span list per row to support filled curves (flood-fill interiors).
- **OpenROAD timing closure** — Investigate the critical path through the forward-difference adder chain. Experiment with pipelining `d0 += d1` across two cycles to reduce the critical path and potentially reach 50 MHz.
- **Formal verification** — Write SVA (SystemVerilog Assertions) or use SymbiYosys to formally prove that `scan_t` always reaches 128 within the H-blank window for any valid control point input.
- **Analog-on-chip** — Explore Tiny Tapeout's analog tile option to implement a DAC for 4-bit greyscale VGA output instead of the binary 2-bit R2R output implied by the PMOD pinout.

---

## Silicon-Proven Results

> **Status: Not yet submitted to a shuttle.**
>
> The current tapeout run only supported 1×1 tiles. This design targets a 2×1 tile and is ready for submission — it is waiting for a compatible Tiny Tapeout shuttle run. Results will be recorded here once a chip has been fabricated and received.

| Milestone | Status |
|---|---|
| RTL complete | ✅ |
| Simulation tests passing (cocotb) | ✅ |
| GDS passes at 2×1 (LibreLane) | ✅ |
| Submitted to shuttle | ⏳ Pending next 2×1-compatible run |
| Chip received | ⏳ TBD |
| Bring-up on eval board | ⏳ TBD |

Once the chip is in hand, the following will be measured and recorded:

| Metric | Expected | Measured |
|---|---|---|
| VGA sync locked | ✅ | TBD |
| Bézier curve visible on screen | ✅ | TBD |
| Control interface responsive | ✅ | TBD |
| Max reliable clock frequency | 25 MHz | TBD |
| Current draw @ 25 MHz, 3.3 V | — | TBD |
| Observed glitches / artifacts | None | TBD |
| Die photograph | — | TBD |

> *This section will be updated after submission, fabrication, and board bring-up.*

---

<div align="center">

**Built with [Tiny Tapeout](https://tinytapeout.com) · Sky130A · LibreLane**

*Henry Huu Nguyen — 2025*

</div>
