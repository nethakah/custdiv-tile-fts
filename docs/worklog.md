# Worklog

## 2026-08-04

**Goal.** Identify what controls operand bit width per tile in the eFPGA architecture, raise
it, and confirm a wider operator still implements through the flow.

**Did.**

- Reproduced the custdiv flow end to end on the ECE cluster (Yosys + VPR clean).
- Captured a baseline at the Makefile's fixed `--route_chan_width 80`.
- Widened `custdiv` from 16-bit to 32-bit operands. Seven edits across three files:
  `arch/k4_N8_custdiv.xml` (sub_tile ports, pinlocations, custdivsite ports, CUSTDIV
  primitive ports, interconnect index ranges), `flow/custdiv_test/inputs/bbmodels.v`,
  `flow/custdiv_test/inputs/custdiv.v`.
- Wrote `scripts/gen_custdiv.py`, which emits the arch file and both Verilog files from a
  single `--width` argument. Validated by reproducing the hand-edited 32-bit result exactly.
- Swept 8 / 16 / 32-bit for minimum routable channel width, including negative controls at
  the next-lower even width in each case.

**Found.**

- There is no single "bit width" parameter. The width appears in five coupled places in the
  arch XML plus two Verilog files, and VPR rejects the netlist if any disagree. That coupling
  is now handled by `scripts/gen_custdiv.py`.
- 32-bit packs and places cleanly at the default channel width; routing is the wall. The
  W=80 failure was 5 overused CHANY nodes at x=4 (the IO column), i.e. IO-to-tile bandwidth
  rather than tile pin density.
- Minimum routable channel width = tile pins + 3, exact at 8/16/32-bit, with the next-lower
  even width verified to fail in every case. Since pins = 3W+1, every operand bit costs three
  tracks of routing channel.
- Routing area exceeds logic area at 32-bit (150056 vs 86857). In this fabric wires cost more
  than logic, which is why pin count dominates the cost of operator specialization.
- Pin placement is worth roughly 4% of channel width: outputs on the side facing the IO
  column route better than outputs facing away from it.

Numbers and full method in `flow/custdiv_test/results/sweep.md`.

**Open.**

- `fpga.sdc` specifies a 0.5 ns period, inherited verbatim from the upstream FTS custmul
  example. Needs a defensible target before any timing result is reportable.
- Scope question: the proposal currently frames this as hardening a divider. Given that
  routing dominates area and minimum channel width tracks pin count, a stronger frame may be
  a methodology for deciding which HFT operators merit tile specialization, with fixed-point
  division as the worked example. Advisor decision -- affects scope.
- `bbmodels.v` custdiv is an empty blackbox. No divider logic exists in the flow yet.
- Exchange field widths settled from primary specs: NASDAQ ITCH price is 4-byte unsigned,
  spec-capped at 2e9 (31 bits); Shares 4-byte; Order Reference Number 8-byte but an
  identifier, not an operand; no floating point anywhere. NYSE XDP 4-byte signed with a
  per-symbol scale code; IEX DEEP and CME MDP 3.0 both 8-byte. Needs writing into
  proposal section 3.3.
- Not started: operand-width study on real market data (LOBSTER), to determine what width
  the arithmetic actually requires.
