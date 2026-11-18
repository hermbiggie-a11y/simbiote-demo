# Simbiote (formerly FactoryFlow) — Step 1: Scan the Environment → 3D Map

**This is the Simbiote team's original Step 1 design note.** The shared project
overview, architecture, tech stack and risks are in
[`../SIMBIOTE_MASTER_PLAN.md`](../SIMBIOTE_MASTER_PLAN.md). For the current project
layout, see the repository `README.md`, `../GB10_DOWNLOADS.md`, and
`../SSD_LAYOUT.md`. Section numbers below are the ones the code comments cite.

---

## Part 4 — Step 1: Scan the Environment → 3D Map

### 4.1 What this step does

A phone sweep becomes a simulation-ready `.usd` scene, tagged with navigable floor and graspable objects, that Step 2 loads directly into Isaac Sim.

```
iPhone (Stray Scanner)
├── RGB Images (rgb.mp4)              ├── Camera Intrinsics (camera_matrix.csv + per-frame in odometry.csv)
├── LiDAR (depth/*.png + confidence/*.png)  ├── Camera Extrinsics — ARKit VIO (odometry.csv)
├── IMU (imu.csv)                     └── Timestamps (imu.csv, odometry.csv)
     ▼
Capture Engine (Stray Scanner itself — nothing custom to build)
     ▼
Sensor Fusion  →  COLMAP + ARKit + IMU Optimization  →  3DGUT / Gaussian Reconstruction
     ▼
Semantic AI  →  Scene Graph  →  Physics + Robotics Metadata
     ▼
OpenUSD  →  handed to Step 2 (Isaac Sim / Omniverse / AI Agents)
```

**Hardware requirement:** Stray Scanner needs a LiDAR-equipped device — iPhone 12 Pro/Pro Max or later Pro-line, or an iPad Pro. Confirm this before relying on it.

### 4.2 Confirmed capture format (verified against a real test upload)

```
<scan>/
├── camera_matrix.csv     — static 3×3 intrinsic matrix
├── odometry.csv          — per frame: timestamp, frame, x, y, z, qx, qy, qz, qw,
│                            fx, fy, cx, cy, distortion_center_x, distortion_center_y
├── imu.csv               — timestamp, a_x, a_y, a_z, alpha_x, alpha_y, alpha_z (~100 Hz)
├── depth/000000.png ...  — one 192×256 depth map per RGB frame (verify the mm→m scale
│                            factor empirically rather than trusting it blindly)
├── confidence/000000.png ... — one confidence map per RGB frame (0 low / 1 med / 2 high)
└── rgb.mp4                — HEVC video, one frame per odometry.csv row
```

Match depth/confidence files to RGB frames by the `frame` column, not list position — Stray Scanner has had frame-drop bugs historically.

### 4.3 Pipeline mechanics, stage by stage

- **Sensor Fusion:** temporal alignment of IMU onto each frame's timestamp; depth-to-RGB registration (192×256 → ~1920×1440, roughly a 7.5× gap — a real engineering task, not an afterthought); confidence-based filtering (keep confidence ≥ 1, treat 2 as hard constraint); coordinate unification into `odometry.csv`'s own frame.
- **COLMAP + ARKit + IMU Optimization:** seed COLMAP bundle adjustment with `odometry.csv` poses rather than solving cold; add the real, confidence-filtered LiDAR depth as a depth-consistency term; use IMU preintegration to catch and down-weight motion-blurred frames.
- **3DGUT / Gaussian Reconstruction (`nv-tlabs/3dgrut`):** trains directly on distorted, rolling-shutter phone frames via the Unscented Transform rather than requiring pre-undistortion; regularized against real LiDAR depth, which reduces floating-artifact noise in low-texture regions (blank walls, ceilings — exactly what a hospital corridor has plenty of).
- **Semantic AI:** **SAM 3** (Meta, open-vocabulary concept segmentation) over posed RGB frames — detects, segments, *and* projects into 3D from a short text prompt directly (e.g. "navigable floor," "graspable tray"), replacing the earlier Grounding DINO + SAM 2 two-model combo with one model that does both jobs natively.
- **Scene Graph:** each labeled region/object becomes a node (label, bounding volume, pose); relationships become edges (on-top-of, blocks-path, etc.) — this is the layer OpenClaw/Nemotron agents query directly.
- **Physics + Robotics Metadata:** collision mesh generation, mass/friction defaults by semantic class, navigable-floor tagging (feeds Step 2's nav costmap), grasp-affordance tagging (feeds Step 2's grasp targets).
- **OpenUSD export:** `UsdGeom` for mesh, `UsdPhysics` for collision, custom schema attributes for semantic/scene-graph data — the portable file Step 2 loads directly, and the reason any USD-aware tool (not just Isaac Sim) can open it too.

### 4.4 Models — laptop vs GB10

| Stage | Laptop | GB10 |
| :---- | :---- | :---- |
| Feature matching | COLMAP, CPU/modest GPU, sequential matcher | COLMAP, CUDA build, exhaustive/vocab-tree matcher |
| Depth completion | Depth Anything V2 – **Small**, guided upsampling of real LiDAR | **Depth Anything 3** (newer generation — verify the exact checkpoint/repo) or Depth Anything V2 – **Large** as the known-good fallback |
| Gaussian reconstruction | 3DGUT at reduced density/iterations/resolution | 3DGUT at full density/iterations/resolution, `--with_ut` on |
| Semantic labeling | Grounding DINO (tiny/base) + SAM 2 – tiny/small (laptop-friendly stand-in) | **SAM 3** (or SAM 3.1) — one model, native text-prompt concept segmentation, replaces the two-model laptop combo entirely |
| Scene-graph reasoning | Phi-4-mini (3.8B) or Gemma 3 4B | **Nemotron 3 Super** (120B-A12B, NVFP4, ~60 GB) — confirmed to run on a single GB10; shared with OpenClaw orchestration and Step 4. Can't stay resident alongside a fully-loaded Isaac Sim (see the master plan's Part 1 sequencing note) — **Nemotron 3 Nano** (~25 GB) is the simpler always-resident fallback |

Confidence handling either way: level-2 depth pixels are hard anchors, level-1 soft, level-0 dropped before it reaches the upsampling model.

### 4.5 Build spec

```
factoryflow/
├── capture_ingest/
│   └── ingest.py         — def load_capture_bundle(path: str) -> FusedFrames
│                            Parses the confirmed schema directly (§4.2)
├── reconstruction/
│   ├── colmap_refine.py  — def refine_poses(frames: FusedFrames) -> RefinedPoses
│   └── gaussian_recon.py — def reconstruct(refined: RefinedPoses) -> GaussianScene
├── semantic/
│   └── label_scene.py    — def label(scene: GaussianScene) -> LabeledScene
├── scenegraph/
│   └── build_graph.py    — def build_scene_graph(labeled: LabeledScene) -> SceneGraph
│                            LLM call; define and validate the JSON schema explicitly
├── physics_meta/
│   └── attach_physics.py — def attach_physics(graph: SceneGraph) -> PhysicsScene
├── usd_export/
│   └── export.py         — def export_usd(scene: PhysicsScene, out_path: str) -> None
└── run_pipeline.py        — CLI: ingest → refine → reconstruct → label →
                              build_graph → attach_physics → export_usd
```

This was the planned layout; the shipped code is the `simbiote/mapper/` package
(`ingest.py`, `stages.py`, `usd.py`, `cli.py`) described in the README.

**Build order:** `ingest.py` → `colmap_refine.py` → `gaussian_recon.py` → `label_scene.py` → `build_graph.py` → `attach_physics.py` → `export.py` → wire together in `run_pipeline.py`. Each step has its own sanity check before moving to the next — don't chain them blind.

**Acceptance tests:**
- *Capture itself:* real translation through the space (multiple meters, moving between areas), not a short in-place pan — the first test upload was ~1 m of translation over 8.5 s, enough to validate `ingest.py`'s parsing, not enough baseline for COLMAP to meaningfully refine.
- *Full pipeline:* produce a `.usd` from a scan, then confirm it opens cleanly in Isaac Sim with geometry recognizable as the scanned room, at least one navigable-floor region, and at least one graspable object.

### 4.6 Laptop-first scope

Get one or two real, full-length Stray Scanner walkthroughs done (depth/ and confidence/ folders included). Build and validate ingest through physics metadata on a laptop using the laptop-tier models — none of that needs the GB10. The Isaac Sim open-check is the first GB10 task.

### 4.7 Handoff to Step 2

**You produce:** a `.usd` file with geometry, semantic labels, navigable-floor tags, and grasp-affordance tags attached via `UsdPhysics`/custom schema. **They consume it** via `load_environment()` and `validate_map()` — make sure `validate_map()`'s checks (watertight floor, correct scale, at least one tagged pickable object) are things your export actually guarantees, since that's the contract Step 2 is coding against.
