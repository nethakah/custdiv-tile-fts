# custdiv tile width sweep

Minimum routable channel width vs. tile pin count on the 6x6 eFPGA fabric
(`arch/k4_N8_custdiv.xml`: 11 clbalutile + 1 custdivtile at (3,4) + 4 iotile at x=4,
perimeter EMPTY, 128 bits in / 128 out).

Tile pin count = I0 + I1 + Q + clk.

| Date | Commit | Operand width | Tile pins | Fixed W | Min W | Routed? | Routing area | Logic area | Wirelength | CPD (ns) | Fmax (MHz) | Max chan util |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 2026-08-04 | a4db845 | 16 | 49 | 80 | -- | yes | -- | -- | -- | 6.74898 | 148.171 | -- |
| 2026-08-04 | bdbb993 | 16 | 49 | -- | 50 | yes | 78382.1 | 86856.9 | 526 | 7.86072 | 127.215 | 0.76 @ (3,4) |
| 2026-08-04 | a4db845 | 32 | 97 | 80 | -- | NO | -- | -- | -- | -- | -- | 5 overused CHANY @ x=4 |
| 2026-08-04 | bdbb993 | 32 | 97 | -- | 100 | yes | 150056 | 86856.9 | 886 | 7.23014 | 138.31 | 0.65 @ (3,4) |
| 2026-08-04 | (this commit) | 32 | 97 | -- | 100 | yes | 150056 | 86856.9 | 886 | 7.23014 | 138.31 | 0.65 @ (3,4) |
| 2026-08-04 | (this commit) | 8 | 25 | -- | 28 | yes | 47618.8 | 86856.9 | 388 | 7.71014 | 129.699 | 0.89 @ (4,4) |

## How these numbers are obtained

    grep -E "channel width factor of|Final critical path delay|Total routing area|Total logic block area|Total wirelength|Maximum routing channel utilization|Device Utilization" vpr_stdout.log

- Min W runs: `make vpr NUM_CHAN=` — drops `--route_chan_width`, VPR binary-searches.
- Fixed W runs: `make vpr` — uses `--route_chan_width 80` from the Makefile.

## Caveats

- **Logic area is not a measurement.** Identical across all runs because VPR takes it from
  `<area grid_logic_tile_area="7238.080078"/>` in the arch file. VPR does not model the
  divider getting larger. Only routing area is measured.
- **Timing is against a placeholder constraint.** `fpga.sdc` has `create_clock -period 0.5`
  (2 GHz) with 0.2 ns I/O delay, inherited verbatim from the FTS custmul example. All slack
  figures are meaningless. Fmax is informative; sWNS/sTNS are not.
- **VPR searches even channel widths only** (unidirectional routing needs track pairs), so
  min W is the first even value at which routing succeeds.
- `custdiv` is an empty blackbox. No divider logic exists in the flow; the arch carries only
  a 1500 ps `delay_constant`.

## Observation

TODO (NH): state the pin-count relationship, and whether the 8-bit point confirms it.

Minimum channel width vs. tile pin count: 25 -> 28, 49 -> 50, 97 -> 100.
Near-linear at 16 and 32 bit. At 8 bit it overshoots: routing failures at W=16 and
W=24 were "no possible path" (Fc connectivity) rather than congestion, so below
~W=28 the binding constraint is Fc (out_val=0.15), not pin count. Also at 8 bit the
custdivtile (25 pins) is no longer the widest tile -- clbalutile is, at 60 pins.

The 32-bit generated arch reproduces the hand-edited result exactly (W=100, wirelength
886, CPD 7.23014 ns), validating scripts/gen_custdiv.py.
