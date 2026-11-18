# Third-party notices

## Upstream project

**Simbiote**, by the Simbiote team (Gagan Shiva Kumara, Suraj Shivakumar, Sky B.,
Andrew Zhao and Advaith Vecham): https://github.com/gaganshivakumara/simbiote

The upstream repository has no licence file and no notice files, and none of its
source files carries a licence header, so there is no upstream notice text to
reproduce here. See `LICENSE`.

## Vendored code

None. No other project's code is copied into this repository.

## Models, tools and assets used by the full pipeline

These are not included in this repository. Each is downloaded or built from its own
project, onto the prepared model drive (see `docs/GB10_DOWNLOADS.md` and
`docs/SSD_LAYOUT.md`), and is used under its own licence and terms. Read and accept
each licence before you download or use it.

| Component | Used for | Source | Licence |
| --- | --- | --- | --- |
| COLMAP | Pose refinement (Scan → Map) | [COLMAP](https://github.com/colmap/colmap); built from source | Its own licence |
| Depth Anything 3; Depth Anything V2 Large as fallback | Depth completion (Scan → Map) | Official Depth Anything 3 repository and checkpoint; [`depth-anything`](https://huggingface.co/depth-anything) organization on Hugging Face | Each project's own licence |
| 3DGRUT / 3DGUT | Gaussian reconstruction (Scan → Map) | [nv-tlabs/3dgrut](https://github.com/nv-tlabs/3dgrut) | Its own licence |
| SAM 3 | Semantic labeling (Scan → Map) | [facebookresearch/sam3](https://github.com/facebookresearch/sam3); Hugging Face [facebook/sam3](https://huggingface.co/facebook/sam3) | Its own licence |
| Qwen3-8B (NVFP4), served by the NGC vLLM container | Agentic LLM (Command) | The exact source of the team's copy isn't recorded; use NVIDIA's Qwen3-8B NVFP4 listing on [NGC](https://catalog.ngc.nvidia.com) or [Hugging Face](https://huggingface.co/nvidia). vLLM container from NGC | The model's and the container's own licences |
| Nemotron 3 Super / Nano (optional) | Agentic LLM (Command) | Hugging Face, NVIDIA organization ([`nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-NVFP4`](https://huggingface.co/nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-NVFP4)) | The model's own licence |
| Ollama with Qwen3 8B or Phi-4-mini (laptop, optional) | Agentic LLM on a laptop | [Ollama](https://ollama.com) | Each model's own licence |
| NVIDIA Isaac Sim 5.x and Isaac Lab | Simulation and training | [NVIDIA Isaac Sim](https://developer.nvidia.com/isaac/sim) (Omniverse / NGC); [Isaac Lab](https://github.com/isaac-sim/IsaacLab) source | Their own licence terms |
| NVIDIA Isaac scene and robot assets (`hospital.usd`, Ridgeback Franka) | Hospital scene and robot | Isaac Sim's Local Assets Pack | Their own licence terms |
| WiLoR and the MANO hand model | Hand-pose estimation on the GB10 (Teleoperate) | Hugging Face Space [`rolpotamias/WiLoR`](https://huggingface.co/spaces/rolpotamias/WiLoR) | Each project's own licence |
| cuRobo / cuMotion (optional) | Motion planning / IK | NVIDIA Isaac | Their own licence terms |
| Stray Scanner (iOS app) | Phone capture | App Store | The app's own terms |
| DroidCam (iOS app and optional Linux client) | Teleop camera stream | [App Store](https://apps.apple.com/us/app/droidcam-webcam-obs-camera/id1510258102); DroidCam project | The project's own terms |

The hospital scene uses NVIDIA Isaac assets and contains no patient data.

## Python packages

Installed by `pip`/`uv` from `pyproject.toml` and `requirements.txt`, each under its own
licence: numpy, Pillow, torch, gymnasium, pybullet, onnx, onnxruntime, onnxscript,
opencv-python, mediapipe, pytest, ruff and usd-core.
