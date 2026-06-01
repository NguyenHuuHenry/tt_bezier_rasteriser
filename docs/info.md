<!---
This file is used to generate your project datasheet. Please fill in the information below and delete any unused sections.
You can also include images in this folder and reference them in the markdown. Each image must be less than
512 kb in size, and the combined size of all images must be less than 1 MB.
-->

## How it works

This design is a fixed-function quadratic Bézier curve rasterizer that drives a 640×480 VGA display in real time with no CPU and no framebuffer.

Three control points (P0, P1, P2) are loaded into the chip via the input pins. Once submitted, the hardware evaluates the Bézier curve Q(t) = (1−t)²P₀ + 2t(1−t)P₁ + t²P₂ across 129 uniformly spaced values of t using the **method of forward differences** — a recurrence that replaces all multiplications with additions:

```
d0 += d1    (position accumulator)
d1 += d2    (first difference, changes each step)
d2          (second difference, constant for quadratic)
pixel = d0 >> 14
```

This stepper runs once per horizontal blanking interval (160 available clock cycles, 130 used), filling a **160-bit row buffer** with the X positions of curve points that fall on the current logical row. During active video, the row buffer is streamed to the VGA output — each set bit produces a white 4×4 pixel block on the 160×120 logical canvas.

The design uses a 1×2 tile (two tiles tall) because the register file — six signed accumulators up to 23 bits wide, plus the 160-bit row buffer — exceeds the ~16,500 µm² core area of a single tile.

## How to test

Connect a [TinyVGA PMOD](https://tinytapeout.com/specs/pinouts/#tinyvga-pmod) to the `uo` output port and a VGA monitor to the PMOD. The screen will be blank after reset.

Control points are loaded using the bidirectional pins as edge-triggered commands and `ui_in` as the data bus:

| Step | Action |
|---|---|
| Set `ui_in[7:0]` to X coordinate (0–159) | — |
| Pulse `uio_in[0]` high for one cycle | Latches X into staging register |
| Set `ui_in[6:0]` to Y coordinate (0–119) | — |
| Pulse `uio_in[1]` high for one cycle | Latches Y into staging register |
| Pulse `uio_in[2]` high for one cycle | Commits staged point as P0, then P1, then P2 |

Repeat the above three times (once for each of P0, P1, P2), then:

| Step | Action |
|---|---|
| Pulse `uio_in[3]` high for one cycle | Activates curve — appears on screen within one frame (~16 ms) |
| Pulse `uio_in[4]` high for one cycle | Clears curve, screen goes blank |

To draw a straight horizontal line across the middle of the screen, load: P0=(5, 60), P1=(80, 60), P2=(155, 60), then submit.

To draw a curved arc, load: P0=(5, 5), P1=(80, 115), P2=(155, 5), then submit.

Submitting a new set of three points while a curve is visible replaces it immediately — no explicit clear required.

## External hardware

- [TinyVGA PMOD](https://tinytapeout.com/specs/pinouts/#tinyvga-pmod) connected to `uo` output port
- VGA monitor (or HDMI adapter with VGA input)
- DIP switches or push-buttons on `uio_in[4:0]` and `ui_in[7:0]` for manual control point entry
