# embryomotility

Analysis pipeline for *C. elegans* embryonic motility.

---

## System Overview

**embryomotility** is a production-grade platform for quantifying *C. elegans* embryonic development from brightfield time-lapse microscopy. It solves the problem of high-throughput, automated phenotyping: given a multi-well plate imaged overnight on a motorized microscope, the system detects individual embryos, tracks their developmental stage transitions, measures motility dynamics, and monitors larvae post-hatching — without manual annotation.

The core biological measurement is the **1.5-fold transition** (pre-fold → post-fold), a morphological event that marks the onset of coordinated muscle activity in the embryo. Motility is quantified from pixel-change intensity curves derived from burst stacks; the fold time and motility shape are used to compare strains, mutants, or RNAi conditions at scale.

---

## Architecture

The pipeline has three logical layers:

```
┌─────────────────────────────────────────────────────────────────────┐
│  ACQUISITION                                                        │
│  pycroscope ──── automated microscope control, Z-stacks, autofocus │
│       │                                                             │
│  acq-trigger ── TCP daemon, fires analysis on acquisition complete  │
└────────────────────────────┬────────────────────────────────────────┘
                             │ NDTiff / HDF5 image stacks
┌────────────────────────────▼────────────────────────────────────────┐
│  ANALYSIS                                                           │
│                                                                     │
│  embryo-detection ─── Mask R-CNN → bounding boxes + masks          │
│       │                                                             │
│  embryo-autofocus ─── sharpness-based Z-sweep focus correction     │
│       │                                                             │
│  embryo-stage ──────── ResNet18 + ResNet1D → fold time (minutes)   │
│       │                                                             │
│  embryo-motility ───── pixel-change metrics → motility curves      │
│       │                                                             │
│  l1-tracking ──────── Kalman tracker → larval trajectories         │
│       │                                                             │
│  nuclei-tracker ────── StarryNite 3D segmentation → cell lineage   │
└────────────────────────────┬────────────────────────────────────────┘
                             │ CSVs, .npy arrays
┌────────────────────────────▼────────────────────────────────────────┐
│  ORCHESTRATION                                                      │
│  pipemanager ── headless script registry, tag-based dispatch       │
│  pipetree ───── interactive terminal UI, queue execution           │
└─────────────────────────────────────────────────────────────────────┘
```

**Tag-based script routing.** Each data directory carries a `.pipetags` TOML file that declares its type and metadata (e.g., `tag = "embryo_bf"`, `instrument`, `strain`, `date`). Scripts registered in `scripts.toml` declare which tags they consume. pipemanager and pipetree use this to surface only the scripts relevant to the currently selected data.

---

## Repo Map

| Repo | Purpose |
|------|---------|
| [pycroscope](../pycroscope) | Automated microscope control via Pycro-Manager. Runs multi-position time-lapses with Z-stacks and real-time autofocus. Entry points: `run.py` (single-phase) and `two_phase.py` (embryo + L1 workflow). |
| [acq-trigger](../acq-trigger) | Lightweight TCP daemon that bridges the microscope client and server-side analysis. Fires registered handlers when an acquisition-complete signal is received. Stdlib only, no dependencies. |
| [embryo-autofocus](../embryo-autofocus) | Z-axis autofocus for brightfield imaging. Computes per-ROI sharpness scores (normalized Laplacian variance) from a Z-sweep and fits a parabola to find the optimal focus plane. Hardware-free: pure analysis class `FocusAdjust`. |
| [embryo-detection](../embryo-detection) | Mask R-CNN pipeline for embryo instance segmentation. Produces bounding boxes `(N, 4)`, masks `(N, H, W)`, and confidence scores `(N,)`. Supports both inference and training. |
| [embryo-stage](../embryo-stage) | Developmental stage classifier. A ResNet18 extracts per-burst features; a ResNet1D (or LSTM) temporal model predicts `t_fold_burst` and `t_fold_minutes` from the feature sequence. Outputs `fold_stage.csv` with confidence and QC flag. |
| [embryo-motility](../embryo-motility) | End-to-end motility analysis pipeline. Orchestrates detection → staging → pixel-change metrics → hatching detection → curve fitting → summary statistics. State tracked in `pipe_status.csv`. Shell entry point: `bash/emotility_pipeline.sh`. |
| [l1-tracking](../l1-tracking) | Post-hatching L1 larval motility. Zhang-Suen skeletonization (vectorized numpy) feeds a Kalman filter + Hungarian assignment multi-target tracker. Outputs `trajectories.csv`, `skeletons.csv`, `metrics.csv`, `occlusions.csv`. |
| [nuclei-tracker](../nuclei-tracker) | Python translation of StarryNite — 3D nuclei segmentation for embryo cell lineage tracking. DoG filtering → local maxima → ray-tracing diameter → 3D refinement → overlap resolution. Outputs Acetree-format coordinates. |
| [pipemanager](../pipemanager) | Headless pipeline engine. Discovers scripts from `scripts.toml`, matches them to data directories by tag, resolves `{field}` args from `.pipetags`, and executes them via CLI. No UI dependency. |
| [pipetree](../pipetree) | Interactive terminal UI (Textual) for navigating data directories and running pipeline scripts. Three-pane layout (Data Tree / Pipe To / Run Config) across three tabs (Process / Analysis / Validation). Vim-style keybindings. |

---

## Running the System

### Interactive: pipetree (recommended)

`pipetree` is the primary interface for manual and exploratory runs.

**Prerequisites:**
```bash
export EMOTILITY_ROOT_DATA=/path/to/data/root   # top of your data directory tree
export EMOTILITY_CFG=/path/to/scripts.toml       # pipeline script registry
```

**Launch:**
```bash
python -m pipetree
```

The UI loads a navigable tree of data directories. Select a node; the Process / Analysis / Validation tabs populate with scripts that match that node's `.pipetags`. Navigate with `j`/`k`, switch panes with `h`/`l`, add scripts to the queue with `Enter`, and run the queue with `r`. Output streams to the shared Output panel at the bottom.

**Key bindings:**

| Key | Action |
|-----|--------|
| `j` / `k` | Move up / down |
| `h` / `l` | Switch panes |
| `Tab` | Switch tabs |
| `Enter` | Add script to queue |
| `J` / `K` | Reorder queue |
| `dd` | Remove from queue |
| `r` | Run queue |
| `q` | Quit |

---

### Headless: pipemanager

For scripted or automated runs (cron, post-acquisition hooks):

```bash
# Run a single named script on a data directory
pipemanager run --data-dir /data/20250206_vc2010 --script detect_embryos

# Run a pre-defined queue
pipemanager run --data-dir /data/20250206_vc2010 --queue queue.toml
```

`pipemanager` resolves `{instrument}`, `{strain}`, `{data}`, etc. from the `.pipetags` file in `--data-dir`. The script registry is read from `$EMOTILITY_CFG` or a `scripts.toml` adjacent to the config path.

---

### Automated: post-acquisition trigger

`acq-trigger` wires acquisition completion to pipeline execution:

1. `pycroscope` finishes an acquisition and sends a TCP signal to the `acq-trigger` server.
2. `acq-trigger` fires the registered handler (typically `emotility_pipeline.sh` or a `pipemanager` call).
3. The full embryo-motility pipeline runs unattended.

The legacy shell entry point is:
```bash
bash embryo-motility/bash/emotility_pipeline.sh -i <instrument> -s <strain> -d <YYYYMMDD>
```

---

## Data Conventions

- **Image stacks**: NDTiff or HDF5, 5D burst arrays shaped `[N_rois, N_bursts, 21, H, W]`
- **RAM disk**: Large arrays are staged to `/dev/shm/` for fast GPU/CPU access during analysis
- **Per-directory metadata**: `.pipetags` TOML file declares tags and key-value fields used for arg resolution
- **Pipeline state**: `pipe_status.csv` in the processing directory tracks completion per stage
