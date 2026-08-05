# custdiv tile width sweep

Minimum routable channel width vs. tile pin count on the 6x6 eFPGA fabric
(11 clbalutile + 1 custdivtile at (3,4) + 4 iotile at x=4, perimeter EMPTY, 128 bits
in / 128 out).

Tile pin count = I0 + I1 + Q + clk = 3W + 1.

## Results

| Width | Pins | Arch source | Forced W | Min W | Routed | Routing area | Logic area | Wirelength | CPD (ns) | Fmax (MHz) | Peak chan util |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 16 | 49 | original | 80 | -- | yes | -- | -- | -- | 6.74898 | 148.171 | -- |
| 16 | 49 | original | -- | 50 | yes | 78382.1 | 86856.9 | 526 | 7.86072 | 127.215 | 0.76 @ (3,4) |
| 32 | 97 | hand-edited | 80 | -- | **NO** (5 overused CHANY @ x=4) | -- | -- | -- | -- | -- | -- |
| 32 | 97 | hand-edited | -- | 100 | yes | 150056 | 86856.9 | 886 | 7.23014 | 138.31 | 0.65 @ (3,4) |
| 8 | 25 | generated | -- | 28 | yes | 47618.8 | 86856.9 | 388 | 7.71014 | 129.699 | 0.89 @ (4,4) |
| 16 | 49 | generated | -- | 52 | yes | 80938.4 | 86856.9 | 502 | 7.64956 | 130.726 | 0.71 @ (4,4) |
| 32 | 97 | generated | -- | 100 | yes | 150056 | 86856.9 | 886 | 7.23014 | 138.31 | 0.65 @ (3,4) |
| 32 | 97 | generated | 98 | -- | **NO** (1 overused CHANY @ (4,1)-(4,2)) | -- | -- | -- | -- | -- | -- |

Arch source: "original" = pins on right/top only (as inherited from the FTS custmul
template). "hand-edited" and "generated" = inputs right/top, outputs left/bottom.
At 32-bit the hand-edited and generated arch files are identical apart from the mode
name, so the generator reproduces the hand result exactly.

## Reproducing

    python3 scripts/gen_custdiv.py --width <W>
    cd flow/custdiv_test && WIDTH=<W> source design.sh
    cd yosys && make synth
    cd ../vpr_pnr && make vpr NUM_CHAN=

- `make vpr NUM_CHAN=` drops `--route_chan_width`, so VPR binary-searches for minimum W.
- `make vpr NUM_CHAN="--route_chan_width N"` pins it to N, to test a specific width.
- `make vpr` uses the Makefile default of 80.

Metrics:

    grep -E "channel width factor of|Final critical path delay|Total routing area|Total logic block area|Total wirelength|Maximum routing channel utilization" vpr_stdout.log

## Observation

Minimum routable channel width tracks tile pin count exactly:

    W_min = pins + 3

verified at three widths (25 -> 28, 49 -> 52, 97 -> 100). In each case VPR tested the
next-lower even width (26, 50, 98) and failed, so these are true minima under the
even-width constraint of unidirectional routing, not artifacts of the binary search.
The 98 case was forced explicitly with `--route_chan_width 98`, because VPR's search had
used 98 as a lower bound without ever testing it.

Since pins = 3W + 1, channel width scales as 3W + 4 in the operand width: **every operand
bit costs three tracks of routing channel.**

Pin *placement* is a second-order but real effect. At 16-bit, two arrangements of the same
49 pins gave different results:

- all pins on right/top, outputs adjacent to the iotile column at x=4: **W = 50**, peak
  channel utilisation 0.76 at (3,4)
- inputs right/top, outputs left/bottom: **W = 52**, peak utilisation 0.71 at (4,4)

The congestion hotspot moves from the tile itself to the IO column when outputs sit on
the side facing away from their consumer. Roughly 4% of channel width, free to recover.
Mechanism is a hypothesis, not confirmed.

## Caveats

- **Logic area is not a measurement.** Identical across every run because VPR reads it from
  `<area grid_logic_tile_area="7238.080078"/>` in the arch. VPR does not model the divider
  getting larger. Only routing area is measured.
- **Timing is against a placeholder constraint.** `fpga.sdc` sets `create_clock -period 0.5`
  (2 GHz) with 0.2 ns I/O delay, inherited verbatim from the upstream FTS custmul example.
  Slack figures are meaningless; Fmax is informative.
- **VPR searches even channel widths only** — unidirectional routing requires track pairs.
- `custdiv` is an empty blackbox. No divider logic exists in the flow; the arch carries only
  a 1500 ps `delay_constant`, also inherited from the template.
