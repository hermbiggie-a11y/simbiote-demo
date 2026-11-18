# GB10 SSD layout

Mount the drive at a path such as `/mnt/factoryflow-ssd`; all portable assets
live below its `AI/` directory.

```text
AI/
├── models/
│   ├── depth-anything/  # Scan → Map: depth completion
│   ├── sam3/            # Scan → Map: official facebook/sam3 checkpoint
│   ├── nemotron/        # Command: agentic planning model
│   ├── theia/           # Simulate & Train: optional vision backbone
│   └── wilor/           # Teleoperate: hand-pose model
├── repos/
│   ├── 3dgrut/          # Scan → Map: reconstruction source
│   ├── colmap/          # Scan → Map: build CUDA/ARM binary on GB10
│   ├── Depth-Anything-3/
│   ├── sam3/
│   ├── curobo/
│   ├── IsaacSim/        # build on GB10 ARM64
│   └── IsaacLab/        # install against the GB10 Isaac Sim build
├── assets/
│   └── hospital/        # populated from Isaac Sim/NGC on GB10
└── captures/            # Stray Scanner exports; never commit to Git
```

Run `scripts/mapper/setup_gb10_mapper.sh <SSD mount>` on the GB10 to create the
mapper configuration. Then build Isaac Sim and install Isaac Lab on that
machine; their Linux ARM64 build artifacts are not portable from Windows.
