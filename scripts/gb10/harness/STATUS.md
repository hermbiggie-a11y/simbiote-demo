# GB10 build-out status

Long-horizon tracker for the 3DGRUT build and the orchestration stack. **Ground truth is
`./run_checks.sh`, not this file** — every claim below must correspond to a check
that flips `BLOCKED → PASS` and stays there.

Run it: `scripts/gb10/harness/run_checks.sh` · latest machine-readable report:
`reports/latest.json`

## Parts

<!-- BEGIN GENERATED: parts (render_status.py) -->
Run `./run_checks.sh`, then `./render_status.py` to fill in this table on your GB10.
<!-- END GENERATED: parts -->

## Standing constraints

- **No discrete VRAM.** GPU allocations come out of the same 128 GB as the OS.
  A resident vLLM server plus a GUI Isaac Sim has hard-locked this box.
  See `docs/GB10_MEMORY_BUDGET.md`. Cap build parallelism; check `MemAvailable` first.
- **aarch64 + CUDA 13 + sm_121.** Generic x86 wheels do not apply. Native builds must
  target `TORCH_CUDA_ARCH_LIST=12.1` or they produce kernels the GPU cannot launch.
- **Without `sudo`** there is no `apt install`; build against what is already on the box.
- 3DGUT (rasterisation) is what the mapper needs. 3DGRT (ray tracing, OptiX) is
  optional — do not let it block the mapper path.
