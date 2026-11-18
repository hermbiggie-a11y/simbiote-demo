# Simbiote on Dell Pro Precision GB10: full checklist and runbook

This is the full setup, run and reset checklist for Simbiote, gathered from the
Simbiote team's own instructions. The short version is in the repository
`README.md`. Every command runs from the root of this repository.

The mapper has three modes. Use them like this:

- **`preview`** is the quick check. It fuses the scan's real LiDAR depth with its
  ARKit poses on the CPU, so it works on a CPU laptop with no NVIDIA GPU.
- **`production`** is the full GPU path on the GB10: COLMAP, Depth Anything,
  3DGRUT and SAM 3.
- **`proxy`** is for wiring tests only. Its output is visibly marked, fails
  production validation unless `--allow-proxy` is supplied, and is never the demo map.

---

## 1. Quick check on any machine (laptop tier)

```bash
# Python 3.11+
pip install -e ".[dev]"      # or: uv sync --extra dev
python -m pytest
```

Ask the robot to do something, with no model required (rule-based backend and a
stubbed robot):

```bash
simbiote-agentic "pick up the tray in the supply room"
```

Teleoperate with your webcam (prints actions to the console):

```bash
simbiote-teleop --sink console
```

Add `--help` to any command for the full flag set. Development checks:

```bash
uv sync --extra dev
uv run pytest
uv run ruff check .
```

---

## 2. Phone scan upload

Put Stray Scanner exports in **`UPLOAD_PHONE_SCANS_HERE/`**. Each scan should be
its own subfolder containing `camera_matrix.csv`, `odometry.csv`, `imu.csv`,
`rgb.mp4`, `depth/`, and `confidence/`.

On Windows (PowerShell):

```powershell
# Option A: drag the phone export folder into UPLOAD_PHONE_SCANS_HERE in File Explorer

# Option B: import from anywhere
python scripts\mapper\import_stray_capture.py "C:\path\to\StrayScannerExport" --name hospital-walkthrough

# Optional: also copy onto the SSD for GB10
python scripts\mapper\import_stray_capture.py "C:\path\to\StrayScannerExport" --name hospital-walkthrough --ssd
```

Then validate:

```powershell
python -m simbiote.mapper.cli ingest ".\UPLOAD_PHONE_SCANS_HERE\hospital-walkthrough"
```

On the GB10, a plugged-in iPhone mounts over AFC; see
[`docs/modules/mapper.md`](modules/mapper.md) for the copy command.

### Metadata-only Stray Scanner export

Some exports, including pose/IMU-only downloads, contain only
`camera_matrix.csv`, `odometry.csv`, and `imu.csv`. They verify that pose data
reaches the mapper, but they cannot reconstruct or label a room: there are no RGB
frames or LiDAR depth. Validate one explicitly:

```bash
simbiote-map --config config/mapper.gb10.toml \
  ingest /path/to/metadata-only-scan --allow-metadata-only
```

In `proxy` mode, `run` accepts this capture and produces a clearly marked test
USD. Production mode always requires `rgb.mp4`, `depth/`, and `confidence/`.

---

## 3. Preview mode (the quick check, CPU laptop)

A laptop can run `preview` mode even without NVIDIA CUDA. It fuses the Stray
Scanner's confidence-filtered LiDAR depth with its ARKit poses into a sparse
OpenUSD `Points` layer, then composes that into the mapper output. This is a real
capture-geometry preview; it is not a 3DGRUT reconstruction.

```powershell
$env:FACTORYFLOW_MODE="preview"
$env:FACTORYFLOW_WORK_ROOT="$PWD\.local\work"

python -m simbiote.mapper.cli --config config/mapper.example.toml run `
  --capture ".\UPLOAD_PHONE_SCANS_HERE\hospital-walkthrough" `
  --out ".\.local\preview.usda"
```

The output references `lidar_preview.usda` under its timestamped work folder.
Preview mode **still runs SAM 3** whenever `FACTORYFLOW_SAM3_COMMAND` is set;
without it you get a marked `proxy_graspable` cube instead of real labels.

---

## 4. Prepare the model drive

The prepared SSD has one `AI/` layout. Do not duplicate models or captures
inside the Git repository. What goes on it, and where each item comes from, is
in [`docs/GB10_DOWNLOADS.md`](GB10_DOWNLOADS.md) and
[`docs/SSD_LAYOUT.md`](SSD_LAYOUT.md); every model and tool keeps its own licence
(see `THIRD_PARTY_NOTICES.md`).

```text
<SSD>/AI/
├── captures/
├── models/
│   ├── depth-anything/
│   ├── nemotron/
│   ├── sam3/
│   ├── theia/
│   └── wilor/
├── repos/
│   ├── 3dgrut/
│   ├── colmap/
│   ├── Depth-Anything-3/
│   ├── sam3/
│   ├── IsaacSim/
│   └── IsaacLab/
└── assets/
    └── hospital/
        └── hospital.usd
```

Isaac Sim and Isaac Lab source are staged on the SSD, but must be built on the
ARM64 GB10. Before you load large models, read
[`docs/GB10_MEMORY_BUDGET.md`](GB10_MEMORY_BUDGET.md): CPU and GPU share the
same 128 GB, so not everything can be resident at once.

---

## 5. GB10 setup

```bash
chmod +x scripts/mapper/setup_gb10_mapper.sh
./scripts/mapper/setup_gb10_mapper.sh /absolute/path/to/ssd
source config/mapper.gb10.env
```

This creates the mapper work root, generates `config/mapper.gb10.toml` and
`config/mapper.gb10.env`, and installs the Python package with `uv`. Edit the
generated config to match the actual downloaded directories.

Several GB10 scripts have default paths at the top of the script for the Isaac Sim
release build (`_build/linux-aarch64/release`) and for the assets and models folder,
taken from the machine the team built on. In these docs, `$ISAAC_RELEASE` is the
Isaac Sim release folder and `$AI_ROOT` is the models and assets folder. Set
`ISAAC_RELEASE`, `ISAAC_PYTHON`,
`AI_ROOT` or `SIMBIOTE_WILOR_DIR` where the script reads them, or edit the
default path at the top of the script.

The model repositories have separate CUDA Python environments. Point these at the
executables created when installing them:

```bash
export DGRUT_PYTHON="$DGRUT_ROOT/.venv/bin/python"
export DA3_BIN="/path/to/depth-anything-3/.venv/bin/da3"
export SAM3_PYTHON="/path/to/sam3/.venv/bin/python"
```

The adapters require `ffmpeg`, a built `colmap` executable in `PATH`, and
CUDA-enabled PyTorch in the DA3, SAM 3, and 3DGRUT environments. Run each
project's documented install process on the GB10 ARM64 system; copying a
Windows environment will not work.

---

## 6. Checks before a production map

```bash
scripts/gb10/mapper/preflight_mapper.sh /absolute/path/to/ssd/AI "$PWD"
simbiote-map --config config/mapper.gb10.toml doctor
```

- `preflight_mapper.sh` blocks on missing ARM64/NVIDIA runtime, checkpoints,
  repositories, COLMAP, 3DGRUT's Python environment, or non-executable adapters.
  The hospital asset is reported as a warning so a live scan can still proceed,
  but it is required for the reliable fallback demo.
- `doctor` returns nonzero if a requirement for the selected mode is missing. The
  architecture check is informational so development on x86 laptops still works.
- The build-out harness (`scripts/gb10/harness/run_checks.sh`) runs every GB10
  capability check and writes a report; see
  [`scripts/gb10/harness/README.md`](../scripts/gb10/harness/README.md). It
  refuses to start below 20 GB of available memory.

Set `mode = "production"` only after all adapters pass `doctor`.

### Production adapters

The model repositories are independently versioned and expose no stable shared
Python API, so the mapper uses **executable adapter contracts** rather than
importing repository internals. `config/mapper.gb10.env` sets these to the
adapters in `scripts/gb10/adapters/`:

| Adapter | Arguments | Must produce |
| --- | --- | --- |
| COLMAP | `{capture}`, `{output}` | Seed/refine from `odometry.csv`; write COLMAP artifacts under `{output}` |
| Depth Anything | `{capture}`, `{colmap}`, `{checkpoint}`, `{output}` | Confidence-filter LiDAR depth, complete/register it at RGB resolution; `{output}` must be nonempty |
| 3DGRUT | `{capture}`, `{colmap}`, `{depth}`, `{output}`, `{dgrut}` | Optimize the Gaussian scene, generate collision-capable mesh, emit at least one `.usd`/`.usda`/`.usdc` **directly** in `{output}` — a raw `.ply` or `.obj` is not a valid Step 2 handoff |
| SAM 3 | `{capture}`, `{geometry}`, `{checkpoint}`, `{output}` | `{output}/detections.json` (schema below) |

```json
{
  "nodes": [
    {
      "node_id": "floor_0",
      "label": "navigable floor",
      "kind": "floor",
      "confidence": 0.98,
      "bounds": { "center": [0.0, 0.0, 0.0], "size": [6.0, 0.05, 8.0] },
      "source_frame_ids": [0, 1, 2]
    },
    {
      "node_id": "tray_0",
      "label": "tray",
      "kind": "object",
      "confidence": 0.91,
      "bounds": { "center": [1.2, 0.8, -0.4], "size": [0.4, 0.08, 0.3] }
    }
  ]
}
```

1. `run_colmap.sh` extracts `rgb.mp4` and produces `images/` plus `sparse/0/`.
2. `run_depth.sh` runs DA3 and writes `exports/mini_npz/results.npz`.
3. `run_3dgrut.sh` trains the existing `colmap_3dgut` configuration and requires
   its exported USD/USDZ layer.
4. `run_sam3.sh` uses local `sam3.pt` from the official `facebook/sam3`
   release to turn fixed text prompts into the `detections.json` contract.
   For **multi-frame** labeling use `scripts/gb10/adapters/sam3_detect.py`;
   `scripts/gb10/adapters/sam3_labels.py` labels only the first video frame.

---

## 7. Production run and handoff (full GPU path)

```bash
simbiote-map --config config/mapper.gb10.toml \
  run --capture "$FACTORYFLOW_SSD_ROOT/captures/demo" \
  --out "$FACTORYFLOW_WORK_ROOT/demo.usda"

simbiote-map validate "$FACTORYFLOW_WORK_ROOT/demo.usda"
```

Successful output includes:

- `demo.usda` — Step 2 scene.
- `demo.scene_graph.json` — planner/query contract.
- `<work-root>/mapper/<timestamp>/` — stage manifests and evidence.

The validator requires meter scale, Y-up coordinates, collision APIs, a navigable
floor, and at least one graspable object with mass and grasp type. Production
validation rejects proxy output. To check the map opens in Isaac Sim, see
[`docs/modules/mapper.md`](modules/mapper.md).

---

## 8. Simulate, teleoperate and command on the GB10

- Simulation, training and the Isaac hospital tier:
  [`docs/modules/sim_training.md`](modules/sim_training.md).
- Hand teleoperation into Isaac Sim:
  [`docs/TELEOP_ISAAC_HOSPITAL.md`](TELEOP_ISAAC_HOSPITAL.md); getting the
  iPhone camera into the GB10: [`docs/TELEOP_IPHONE_CAMERA.md`](TELEOP_IPHONE_CAMERA.md).
- Natural-language commands and model backends:
  [`docs/modules/agentic.md`](modules/agentic.md) and
  [`simbiote/agentic/README.md`](../simbiote/agentic/README.md).

---

## 9. Fallbacks and reset

- **No scan, or a poor one:** use the ready-made NVIDIA Isaac hospital scene
  (`hospital.usd`) instead of a live scan.
- **No GPU reconstruction yet:** use `preview` mode for real scanned geometry.
- **Model server down or still loading:** the agentic module's `FallbackBackend`
  falls back to the rule-based `FakeBackend`; the execution report records which
  one produced the plan. Use `--no-fallback` when you want a broken model server
  to fail loudly.
- **Memory pressure:** stop any resident LLM container before booting Isaac Sim
  with a GUI or running the harness
  (`docker stop $(docker ps -q --filter name=vllm)`); see
  [`docs/GB10_MEMORY_BUDGET.md`](GB10_MEMORY_BUDGET.md).
- **Reset between runs:** mapper output goes to a new timestamped folder under
  the work root on every run, and phone scans stay in `UPLOAD_PHONE_SCANS_HERE/`
  or `<SSD>/AI/captures/`. Delete a run's work folder to remove it.
