# Physical Verification Workflow (Step-by-Step)

This document explains the GitHub Actions workflow defined in `.github/workflows/physical_verification.yml` in detailed, engineering-focused terms. The workflow performs automated physical verification using the IHP Open PDK rule decks and KLayout.

## Overview

The workflow executes two verification stages:

1. **DRC (Design Rule Check)** using the official IHP SG13G2 KLayout DRC deck.
2. **LVS (Layout vs. Schematic)** using the official IHP SG13G2 KLayout LVS deck, but only when a corresponding netlist is available.

All intermediate data is stored under `/tmp` to avoid polluting the repository workspace.

## Step-by-Step Execution

### 1) Repository Checkout

The workflow checks out the repository so it can access:

- `doc/info.json` for release metadata
- The design data under `release/`

This ensures the workflow always runs against the exact version of the repository that triggered the workflow.

### 2) System Dependencies Installation

The workflow installs minimal system packages required by the subsequent steps:

- `jq` for robust JSON parsing
- `python3-pip` for installing Python dependencies

These are installed using `apt-get` on the GitHub-hosted Ubuntu runner.

### 3) Resolve Release Inputs (Early Gate)

Before any tool installation, the workflow inspects `doc/info.json` and resolves:

- `release.gds` (layout)
- `release.netlist` (schematic)

The logic is strict and defensive:

- If **neither** is referenced, the workflow exits early with a warning and **skips both DRC and LVS**.
- If a field is referenced but the file does **not** exist, the workflow fails immediately with a clear error.
- If only `release.gds` is present, **DRC runs** and **LVS is skipped** with a warning.
- If `release.netlist` is present but `release.gds` is missing, the workflow fails (LVS requires a layout).

This gate prevents expensive tool installs when no verification can be performed.

### 4) KLayout Download (Ubuntu-Matched)

The workflow determines the runner's Ubuntu major version (e.g., 22, 24) using `lsb_release` and then selects a matching KLayout `.deb` URL directly from `https://www.klayout.de/build.html`. This avoids installing a `.deb` built for a newer Ubuntu release, which would fail due to incompatible libc and toolchain dependencies.

Key safety checks performed:

- If no matching URL is found, the workflow terminates early with a clear error.
- The downloaded file is validated with `dpkg-deb --info` to ensure it is a legitimate Debian package.

### 5) KLayout Installation

The workflow installs KLayout using:

1. `dpkg -i` to unpack the package
2. `apt-get -f -y` to resolve and install any missing dependencies

Finally, `klayout -v` is executed to confirm that the executable is available and functional.

### 6) Python Requirements (IHP Open PDK)

The workflow installs the Python requirements published by the IHP Open PDK:

```
https://raw.githubusercontent.com/IHP-GmbH/IHP-Open-PDK/main/requirements.txt
```

These dependencies support the execution of `run_drc.py` and `run_lvs.py` (e.g., `docopt`, `klayout`, `pyyaml`, `gdstk`).

### 7) Fetch Official DRC/LVS Decks

The workflow clones the official IHP Open PDK repository into `/tmp` using a sparse checkout. Only the required subtrees are downloaded:

- `ihp-sg13g2/libs.tech/klayout/tech/drc/`
- `ihp-sg13g2/libs.tech/klayout/tech/lvs/`
- `ihp-sg13g2/libs.tech/klayout/python/sg13g2_pycell_lib/sg13g2_tech_mod.json`

This keeps the checkout lightweight and deterministic while ensuring the workflow always uses upstream deck versions.

### 8) DRC Execution (Conditional)

DRC runs only when `release.gds` is present and the referenced file exists. This is determined in the early input gate.

The DRC run is executed as:

```
python3 /tmp/ihp-open-pdk/ihp-sg13g2/libs.tech/klayout/tech/drc/run_drc.py \
  --path="<gds-path>" \
  --run_dir=/tmp/drc-results \
  --mp=1 \
  --precheck_drc
```

Notes:

- `--precheck_drc` enables the deck’s pre-check phase.
- Results are written to `/tmp/drc-results`.

### 9) LVS Execution (Conditional)

LVS runs only when both `release.netlist` and `release.gds` are present and the files exist. Otherwise it is skipped or fails early, based on the input gate.

If the netlist is present, LVS is executed as:

```
python3 /tmp/ihp-open-pdk/ihp-sg13g2/libs.tech/klayout/tech/lvs/run_lvs.py \
  --layout="<gds-path>" \
  --netlist="<netlist-path>" \
  --run_dir=/tmp/lvs-results
```

### 10) Artifact Upload

The workflow always uploads both result directories (even if a stage failed), which simplifies post-mortem debugging:

- `DRC-Results` from `/tmp/drc-results`
- `LVS-Results` from `/tmp/lvs-results`

## Failure Modes and Diagnostics

- **KLayout URL not found**: The workflow exits early if `build.html` does not contain a matching Ubuntu `.deb` link.
- **Wrong Ubuntu `.deb`**: Detected by dependency errors; the URL must match the runner’s Ubuntu major version.
- **No release inputs referenced**: The workflow exits early with a warning, skipping DRC/LVS.
- **Missing `release.gds` file**: The workflow fails early if `release.gds` is referenced but missing.
- **Missing `release.netlist` file**: The workflow fails early if `release.netlist` is referenced but missing.

## Design Rationale

- **No repo contamination**: All runtime files are stored in `/tmp`.
- **Upstream rule decks**: Always pulls the official IHP Open PDK DRC/LVS decks to avoid drift.
- **Deterministic selection**: KLayout `.deb` is selected explicitly based on the runner OS version.

If you need changes to the execution flags (e.g., different LVS modes or extra DRC options), update `.github/workflows/physical_verification.yml` accordingly.
