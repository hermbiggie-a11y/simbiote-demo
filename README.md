# Simbiote: offline robot training on Dell Pro Precision GB10

Simbiote takes a mobile-manipulator robot from **scan → simulate → train → operate**,
fully offline, on a single Dell Pro Precision GB10. Point it at a robot spec and a
space (one you scanned with a phone, or a ready-made NVIDIA Isaac hospital scene) and
it trains the robot to navigate without collisions and pick things up, then lets a
person drive it by hand tracking or plain-language commands. People stay in charge:
they choose the scene, drive or instruct the robot, and every session they run
becomes training data for the next round.

No cloud and no external APIs: every model, training run and session runs on the box.

## Watch it first

The overview video is under [Releases](../../releases). Its narration describes what
Simbiote does; it does not claim that the footage shows it running.

## The stages

| Stage | What happens | Check or human gate |
| --- | --- | --- |
| 1. Scan → Map (`simbiote/mapper/`) | An iPhone LiDAR scan (Stray Scanner export) becomes a simulation-ready OpenUSD scene with a navigable floor, graspable objects and a scene-graph JSON. | **Human:** a person makes the scan or picks the library scene. **Check:** `simbiote-map doctor` and `validate_map` (meter scale, Y-up, collision API, a navigable floor, a graspable object with mass and grasp type). Proxy output fails production validation. |
| 2. Simulate & Train (`simbiote/sim_env/`, `simbiote/training/`, `simbiote/robot/`) | The reference robot (4-wheel omnidirectional base, arm and gripper) is spawned in the scene and trained by reinforcement learning for navigation and grasping; policies are exported for stages 3 and 4. | **Check:** the test suite (`python -m pytest`) and, on the GB10, `scripts/gb10/isaac/eval_hospital_nav.py`, which runs the nav policy between every ordered pair of hospital locations and reports success and path efficiency. |
| 3. Teleoperate (`simbiote/teleop/`) | A person drives the robot in real time with tracked hand movements from the phone camera; every session is logged as a demonstration. | **Human:** the operator is driving. **Check:** the IK bridge smooths the motion and handles the hand leaving the frame. |
| 4. Command (`simbiote/agentic/`) | A person types an instruction ("pick up the tray in the supply room"); an LLM turns it into a validated sequence of skills that the robot runs on its own. | **Human:** a person gives the instruction. **Check:** the plan is validated against the tool schema with one corrective retry, scene objects must resolve to real scene-graph ids, and the task executor checks preconditions, times out a hung skill and releases on failure. The audit trail records whether the model or the rule-based fallback made the plan. |
| 5. Retrain (`simbiote/training/retrain.py`) | Each teleop or command session becomes a demonstration that fine-tunes the policy (behavioral cloning, then PPO). | **Human:** a person runs the retrain step (`ingest_demo()`, then `finetune_policy()`). **Check:** fine-tuning refuses to start with no ingested demonstrations, and unsafe task or session names are rejected on ingest. |

## Hardware

**Built and run on:**

| Component | Built and run on |
| --- | --- |
| System | Dell Pro Precision GB10 (top configuration) |
| GPU | NVIDIA GB10 Grace Blackwell superchip, 128 GB unified LPDDR5x (shared by CPU and GPU) |
| CPU | 20-core NVIDIA Grace Arm CPU (GB10) |
| Memory | 128 GB unified LPDDR5x (shared by CPU and GPU) |
| Storage | 4 TB NVMe SSD |
| OS | NVIDIA DGX OS |
| Software | OpenUSD, COLMAP, Depth Anything, 3DGRUT, SAM 3, NVIDIA Isaac assets (versions not stated) |

You also need a LiDAR-equipped iPhone or iPad Pro running Stray Scanner for live scans
and as the teleoperation camera. The laptop tier (tests, preview mapping, rule-based
commands, webcam teleop) runs on an ordinary Linux, macOS or Windows machine.

## What's in this repo

| Path | What |
| --- | --- |
| `simbiote/mapper/` | Stage 1: Stray Scanner ingest, the five mapper stages (COLMAP, Depth Anything, 3DGRUT, SAM 3, scene graph), OpenUSD export and validation, the `simbiote-map` command |
| `simbiote/sim_env/` | Stage 2: PyBullet Gymnasium environments (navigation, grasp, wheelchair stretch task) and the Isaac Sim hospital tier |
| `simbiote/training/` | Stage 2 and 5: shared actor-critic network, PPO, behavioral cloning, retraining, policy export (ONNX/TorchScript) |
| `simbiote/robot/`, `simbiote/robot_iface/` | Engine-agnostic robot config and the shared schemas (`RobotAction`, `Pose`, `Trajectory`, skills) |
| `simbiote/teleop/` | Stage 3: camera source, hand tracking (MediaPipe on a laptop, WiLoR on the GB10), IK bridge, `simbiote-teleop` |
| `simbiote/agentic/` | Stage 4: scene query, command parser, LLM backends, task executor, `simbiote-agentic` |
| `simbiote/sim_stub/`, `simbiote/assets/` | Toy PyBullet robot for teleop preview; packaged URDFs and the stand-in hospital scene graph |
| `simbiote/demo_logger.py` | Shared session logger that feeds retraining |
| `scripts/mapper/`, `scripts/training/`, `scripts/teleop/` | Scan import and GB10 mapper setup; demonstration utilities; teleop entry points |
| `scripts/gb10/` | GB10-only tooling: model adapters, USD checks and viewers, preflight, Isaac hospital checks and nav evaluation, Isaac-side teleop, agentic hospital demo, and the build-out check harness |
| `tests/` | Tests for every module |
| `config/mapper.example.toml` | Mapper config template (the GB10 config is generated from it) |
| `artifacts/mapper/out/` | Sample scene graphs from real scans |
| `UPLOAD_PHONE_SCANS_HERE/` | Drop zone for phone scans |
| `docs/GB10_RUNBOOK.md` | The full setup, run and reset checklist |
| `docs/` | Model download checklist, memory budget, SSD layout, teleop camera and Isaac teleop guides, module notes, and the team's original build plan |

## Setup

Linux on the GB10 (NVIDIA DGX OS); the quick check in step 1 also runs on a laptop.

1. **Quick check (any machine, Python 3.11+).** `pip install -e ".[dev]"`, then
   `python -m pytest`. Try `simbiote-agentic "pick up the tray in the supply room"`
   (rule-based planner, no model needed) and `simbiote-teleop --sink console`.
2. **Map a scan in preview mode.** Copy a Stray Scanner export into
   `UPLOAD_PHONE_SCANS_HERE/`, validate it with `python -m simbiote.mapper.cli ingest`,
   and run the mapper with `FACTORYFLOW_MODE=preview`. Preview is the quick check and
   works on a CPU laptop. (`proxy` mode is for wiring tests only, never the demo map.)
3. **Prepare the model drive.** The full pipeline needs a prepared drive of large
   models and assets, laid out as in `docs/SSD_LAYOUT.md`. Each one comes from its own
   project and keeps its own licence (also listed in `THIRD_PARTY_NOTICES.md`):

   | Model or asset | Used for | Source | Licence |
   | --- | --- | --- | --- |
   | COLMAP (CUDA build) | Pose refinement | [COLMAP](https://github.com/colmap/colmap); built from source on the GB10 | Its own licence |
   | Depth Anything 3 (fallback: Depth Anything V2 Large) | Depth completion | The official Depth Anything 3 repository and checkpoint; V2 from the [`depth-anything`](https://huggingface.co/depth-anything) organization on Hugging Face | Each project's own licence |
   | 3DGRUT / 3DGUT | Gaussian reconstruction | [nv-tlabs/3dgrut](https://github.com/nv-tlabs/3dgrut) | Its own licence |
   | SAM 3 (`sam3.pt`) | Semantic labeling | [facebookresearch/sam3](https://github.com/facebookresearch/sam3); checkpoint [facebook/sam3](https://huggingface.co/facebook/sam3) on Hugging Face | Its own licence |
   | Agentic LLM: Qwen3-8B (NVFP4) served by the NGC vLLM container; optional Nemotron 3 Super or Nano | Turning instructions into skills | Qwen3-8B (NVFP4): the exact source of the team's copy isn't recorded; use NVIDIA's Qwen3-8B NVFP4 listing on [NGC](https://catalog.ngc.nvidia.com) or [Hugging Face](https://huggingface.co/nvidia). Nemotron from Hugging Face ([`nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-NVFP4`](https://huggingface.co/nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-NVFP4), Nano from the [NVIDIA organization](https://huggingface.co/nvidia)) | Each model's own licence |
   | NVIDIA Isaac Sim 5.x, Isaac Lab, and Isaac scene and robot assets (`hospital.usd`, Ridgeback Franka) | Simulation and the hospital scene | [NVIDIA Isaac Sim](https://developer.nvidia.com/isaac/sim) (Omniverse / NGC) and Isaac Sim's Local Assets Pack; [Isaac Lab](https://github.com/isaac-sim/IsaacLab) | Their own licence terms |
   | WiLoR (and the MANO hand model it uses) | Hand-pose estimation on the GB10 | Hugging Face Space [`rolpotamias/WiLoR`](https://huggingface.co/spaces/rolpotamias/WiLoR) | Each project's own licence |

   `docs/GB10_DOWNLOADS.md` has sizes and download order;
   `docs/GB10_MEMORY_BUDGET.md` shows what fits in 128 GB at once.
4. **Set up the GB10.** `./scripts/mapper/setup_gb10_mapper.sh /absolute/path/to/ssd`,
   then `source config/mapper.gb10.env` and point `DGRUT_PYTHON`, `DA3_BIN` and
   `SAM3_PYTHON` at each project's own environment.
5. **Run the checks.** `scripts/gb10/mapper/preflight_mapper.sh <ssd>/AI "$PWD"`, then
   `simbiote-map --config config/mapper.gb10.toml doctor`. Optionally run the
   build-out harness, `scripts/gb10/harness/run_checks.sh`.
6. **Run production mode**, the full GPU path on the GB10:
   `simbiote-map --config config/mapper.gb10.toml run --capture <scan> --out <scene>.usda`,
   then `simbiote-map validate <scene>.usda`.
7. **Train, teleoperate and command** in the Isaac hospital scene, following
   `docs/modules/sim_training.md`, `docs/TELEOP_ISAAC_HOSPITAL.md` and
   `docs/modules/agentic.md`.

The full checklist, with every command, the fallbacks and how to reset between runs,
is in [`docs/GB10_RUNBOOK.md`](docs/GB10_RUNBOOK.md).

## Known limits

- **The full pipeline is heavy to reproduce.** It needs the prepared drive of large
  models and assets in step 3, each installed by its own project's process on the ARM64
  GB10, and an Isaac Sim build on the box.
- **Not everything fits in memory at once.** CPU and GPU share 128 GB; the largest
  LLM, a full Isaac Sim and reconstruction have to take turns. A resident LLM server
  plus a GUI Isaac Sim has hard-locked the machine.
- **Simbiote is not finished.** COLMAP poses are computed but not yet fused back into
  the pipeline; scene-graph edges (on-top-of, blocks-path) are not produced yet; the
  wheelchair-transport task and vision distillation are stretch goals; and the
  wheelchair manipulation skills (`align_gripper`, `attach_handle`, `detach`) still raise
  `NotImplementedError`. The wheelchair navigation halves are wired in `IsaacBackend`.
- **Trained policies are not in the repo.** `checkpoints/` is created when you train
  with `simbiote/training/`.
- **Preview mode without SAM 3** gives a marked stand-in cube (`proxy_graspable`) as
  the graspable object, not real labels.
- **Several GB10 scripts default to the original build machine's home-folder paths**
  for Isaac Sim, models and assets. Set the environment variables they read (`ISAAC_RELEASE`, `ISAAC_PYTHON`,
  `AI_ROOT`, `SIMBIOTE_WILOR_DIR`) or edit the defaults to match your system.
- **PyBullet on Windows** has no PyPI wheel; install MSVC Build Tools, or use WSL2,
  Linux or macOS. Physics tests skip rather than crash.
- **PyBullet on recent macOS** has no wheel either; build it with
  `CFLAGS="-std=gnu17 -Dfdopen=fdopen" CXXFLAGS="-Dfdopen=fdopen" pip install pybullet`.
- **MediaPipe is version-capped** (`>=0.10,<=0.10.21`); later releases removed the API
  the hand tracker uses, and teleop says so and refuses to start.
- **On linux-aarch64 (the GB10)**, `pybullet`, `mediapipe` and `usd-core` have no
  wheels: teleop uses WiLoR, USD comes from Isaac Sim's bundled `pxr`, and pybullet is
  built locally. See `docs/modules/mapper.md`.
- **Iriun Webcam has no ARM build**, so the GB10 teleop camera streams from the phone
  over USB or Wi-Fi instead; see `docs/TELEOP_IPHONE_CAMERA.md`.
- Config defaults are Linux/GB10 paths; on a Windows machine, pass a config file or
  environment overrides for anything beyond `proxy`/`preview`.

## Credits

**Created by the Simbiote team:** Gagan Shiva Kumara, Suraj Shivakumar, Sky B.,
Andrew Zhao and Advaith Vecham. The original project is
[gaganshivakumara/simbiote](https://github.com/gaganshivakumara/simbiote); please give
it a star. A later copy of the project,
[toastedgrain/simbiote](https://github.com/toastedgrain/simbiote), adds a GB10
operations guide; please star it too. Nothing from that copy is included here. This Dell copy keeps the team's code, tests, module names and stage wording;
Dell Technologies rewrote the README, added the runbook and notices, and removed
event-day and personal material.

- **Partner:** NVIDIA (silicon partner: the GB10 Grace Blackwell superchip, DGX OS, Isaac Sim and Isaac assets).
- **Dependencies:** nothing from another repository is copied into this repo. The
  models, tools and assets in Setup step 3 and the Python packages in `pyproject.toml`
  and `requirements.txt` are installed from their own projects and keep their own
  licences; see [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md).
- **Scene:** the hospital scene uses NVIDIA Isaac assets and contains no patient data.
- **AI-generated assets:** this repository contains no AI-generated images, video or audio.
- **Trademarks:** Dell, Dell Pro Precision and other Dell trademarks are trademarks of
  Dell Inc. or its subsidiaries. NVIDIA, Grace Blackwell, DGX OS, Isaac Sim, Isaac Lab,
  Omniverse and PhysX are trademarks of NVIDIA Corporation. Apple, iPhone, iPad and
  ARKit are trademarks of Apple Inc. All other names are trademarks of their
  respective owners. Their use here does not mean that any of these owners endorses
  Dell or this demo.

## Support

Logan Lawler, Dell Technologies: logan.lawler@dell.com

## License

The Simbiote team's repository has no licence file, so this copy is not relicensed
under MIT or any open-source licence. The code remains the Simbiote team's (all rights
reserved by the authors); Dell Inc. shares it with their permission so that you can
download, run and evaluate it as a demo. Dell's changes carry the same terms:
Copyright (c) 2026 Dell Inc. (changes by Dell Technologies). See [`LICENSE`](LICENSE).
Third-party models, tools and assets keep their own licences; see
[`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md).
