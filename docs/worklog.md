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
- Ran minimum-channel-width searches at both widths.

**Found.**
TODO -- raw numbers in `flow/custdiv_test/results/sweep.md`. Cover:
- there is no single "bit width" parameter; the width appears in coupled places across 3 files
- 32-bit packs and places cleanly; routing is the wall
- what the W=80 failure diagnostic actually reported (which nodes, where)
- minimum channel width vs. tile pin count at both widths
- routing area vs. logic area

**Open.**
TODO. Candidates:
- Does minimum channel width track tile pin count? An 8-bit run (25 pins) tests the
  prediction W ~ 26.
- `fpga.sdc` specifies a 0.5 ns period, inherited verbatim from the upstream FTS custmul
  example. Needs a defensible target before any timing result is reportable.
- Scope question: current proposal frames this as hardening a divider. Given that routing
  dominates area and minimum channel width tracks pin count, a stronger frame may be a
  methodology for deciding which HFT operators merit tile specialization, with fixed-point
  division as the worked example. Advisor decision -- affects scope.
- `bbmodels.v` custdiv is an empty blackbox. No divider logic exists in the flow yet.
- Exchange field widths now settled from primary specs: NASDAQ ITCH price is 4-byte unsigned,
  spec-capped at 2e9 (31 bits); Shares 4-byte; Order Reference Number 8-byte but an
  identifier, not an operand; no floating point anywhere. NYSE XDP 4-byte signed with a
  per-symbol scale code; IEX DEEP and CME MDP 3.0 both 8-byte. Needs writing into
  proposal section 3.3.
