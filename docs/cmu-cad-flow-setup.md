# eFPGA CAD Flow on CMU ECE Cluster Computers (SSH)

Nethaka Haldo
July 2026

## 1. Connect

Off campus: connect to vpn.cmu.edu via Cisco Secure Client

https://cmu-enterprise.atlassian.net/wiki/spaces/ITS/pages/2332131370/ECE+Community+Compute+Clusters

```bash
ssh <andrew_id>@ece000.ece.cmu.edu
```

**NOTE:** Stick with one node (e.g. ece003), since scratch is node-local, so you don't have to re-setup the tools/repos repeatedly.

## 2. Storage Issue

|  | Home directory (`~`) | `/scratch/$USER` |
|---|---|---|
| Size | 2 GB quota | 512 GB |
| Persistent | yes | No (admins purge periodically) |
| Follows across nodes | yes | No (node-local) |
| Use | source / repos | tools + big files + outputs |

Make a private scratch directory:
```bash
mkdir -p /scratch/$USER
cd /scratch/$USER
```

**NOTE:** Do not build or run in `~` directory, AFS home quota (~2GB) is problematic.

**NOTE:** Scratch is per-node local storage, so backup files to somewhere like a github repository, since these nodes get cleared periodically and do not carry over between different ECE cluster computers.

## 3. Tool Installation

Yosys + iverilog (prebuilt):
```bash
cd /scratch/$USER
URL=$(curl -s https://api.github.com/repos/YosysHQ/oss-cad-suite-build/releases/latest \
      | grep browser_download_url | grep linux-x64 | cut -d '"' -f4)
wget "$URL" && tar xzf oss-cad-suite-*.tgz
```

VPR/VTR (build from source; takes ~20 minutes to complete):
```bash
cd /scratch/$USER
git clone https://github.com/verilog-to-routing/vtr-verilog-to-routing.git
cd vtr-verilog-to-routing
git checkout 7a9676256
make -j$(nproc)
```

**NOTE:** 7a9676256 matches the FTS repository required version as of July 2026.

Path (`~/.bash_profile`):
```bash
cat >> ~/.bash_profile <<'EOF'
export PATH=/scratch/$USER/vtr-verilog-to-routing/vpr:$PATH
export PATH=$PATH:/scratch/$USER/oss-cad-suite/bin
EOF
```

Then verify:
```bash
source ~/.bash_profile
which vpr yosys iverilog
```

**NOTE:** oss-cad-suite goes at the back of PATH on purpose because it bundles its own python3

## 4. CAD Flow (FTS)

```bash
cd /scratch/$USER
git clone https://github.com/mpcmu/Fabric-to-Silicon.git
```

**NOTE:** This is the upstream flow; you need to fork it (or have a separate repo) and setup git:
```bash
cd /scratch/$USER/Fabric-to-Silicon
git config user.name "<name>"
git config user.email "<email>"
```

## 5. Run the Flow (example)

```bash
cd /scratch/$USER/Fabric-to-Silicon/fpga_cad_flow/example_designs/simple_fpga_cust_tile_test
source design.sh
cd yosys && make synth
cd ../vpr_pnr && make vpr
```

**NOTE:** "VPR succeeded" or "Routed successfully" means the full flow works end-to-end.

## 6. Common Issues on ECE Clusters

1. AFS token expiry: long sessions lose write access to `~`, review via: `kinit && aklog`
2. `git config --global` fails on AFS
3. Home directory at 100% of quota: `.vscode-server` hogs space and stray failed clones do too
4. You must redo section 3 and section 4 if scratch has been purged or you are moving to a new node; source itself is safe in git and `~/.bash_profile` PATH lines persist in AFS home.