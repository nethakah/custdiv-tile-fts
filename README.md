## Layout

```text
.
├── arch/      Custom VPR architecture files
│              └── `custdiv` tile added to the fabric
├── flow/      eFPGA CAD-flow runs
│              └── `custdiv_test/` pushes the tile through Yosys + VPR
├── rtl/       Operator RTL
│              └── `divider.sv` (32-bit fixed-point, radix-2 restoring)
├── verif/     Verification
│              └── `tb_divider.sv` self-checking testbench
├── scripts/   Environment setup helpers
├── docs/      Prior-art brief + notes
└── FTS/       Fabric-to-Silicon (submodule, upstream — untouched)
```

---

## Environment (ECE Numbers cluster)

Tools live in scratch and are on `PATH` via `~/.bash_profile`.

| Tool | Location / Notes |
|------|-------------------|
| **Yosys** | OSS CAD Suite in `/scratch/$USER/oss-cad-suite` |
| **VPR** | VTR built from source at commit `7a9676256` (matches the arch/FASM format) |
| **iverilog** | Comes with the OSS CAD Suite (used for RTL verification) |

Clone the repository with the submodule:

```sh
git clone --recursive <repo>
```

---

## Run the eFPGA flow (`custdiv` tile → Yosys + VPR)

```sh
cd flow/custdiv_test
source design.sh

cd yosys
make synth

cd ../vpr_pnr
make vpr      # expect "VPR succeeded"
```

---

## Run the divider verification

### Using the Makefile

```sh
cd verif
make          # expect "ALL PASS"
```

### Manual invocation

```sh
cd verif
iverilog -g2012 -o sim ../rtl/divider.sv test_divider.sv
vvp sim
```

---

## Status

- `custdiv` tile defined in the eFPGA architecture; packs, places, and routes successfully through VPR.
- 32-bit fixed-point radix-2 restoring divider (`rtl/divider.sv`).
- Verified against SystemVerilog `/` and `%` over directed edge cases and random vectors.
