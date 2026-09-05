# AGENTS.md

RM2026 engineer assembly: synthetic-data rendering (`src/blender`), vision training/export (`src/nn`), and a
ROS 2 Humble + MuJoCo runtime (`runtime/`). All paths and commands below assume the repository root as CWD.
Docs in `src/nn` are in Chinese; `runtime/` docs are in English.

## Environments: three, never mixed

| Workflow | Python | Setup / entrypoint |
| --- | --- | --- |
| Runtime (ROS 2 + MuJoCo) | System `/usr/bin/python3` (ROS 2 Humble, Ubuntu 22.04) | `runtime/requirements.txt` via `/usr/bin/python3 -m pip install --user` |
| YOLO / LiteHRNet (train, export, compose, visualize) | uv venv at `src/nn/.venv` (Python 3.10) | `bash src/nn/setup_env.sh`, then `rm26-nn <cmd>` |
| Blender rendering | Blender 4.5.0 bundled Python 3.11 | `blender -b ... -P src/blender/pipeline/render_dataset.py` |

- Never install runtime deps into the `src/nn` uv venv or launch ROS nodes from it. `runtime/scripts/build.sh` explicitly forces `/usr/bin/python3`.
- First venv setup must be `bash src/nn/setup_env.sh`, NOT `uv sync`: it compiles MMCV CPU ops and rebuilds `xtcocotools` against NumPy 2.2. Run it again only if the venv is deleted or `uv.lock` changes.
- `rm26-nn` auto-adds the repo `src/` dir to `PYTHONPATH`. When the venv is not activated, use `uv run --project src/nn rm26-nn <cmd>`.
- `src/nn/pyproject.toml` pins PyTorch 2.11.0 / cu130 wheels; never swap individual lockfile packages — versions are coupled across PyTorch/MMCV/MMPose/driver.

## Runtime commands

```bash
cp runtime/core/arm_exchange_core/system_config.example.yaml runtime/core/arm_exchange_core/system_config.yaml  # gitignored, required before first build
runtime/scripts/build.sh                      # colcon build --symlink-install, 5 packages
source runtime/scripts/setup_env.sh           # MUST be sourced, not executed
runtime/scripts/test.sh                       # colcon test; sets PYTEST_DISABLE_PLUGIN_AUTOLOAD=1
ros2 launch arm_exchange_host sim_host.launch.py [enable_perception:=true] [enable_operator_input:=true] [keyboard_device:=/dev/input/eventN]
```

- Tests are run via `colcon` (`runtime/scripts/test.sh`), not plain `pytest`; it needs the workspace built first.
- OpenVINO model weights are downloaded separately from Hugging Face; configure device/paths in the local `system_config.yaml`.

## NN commands (`rm26-nn --help` for all)

```bash
rm26-nn check                        # environment self-check
rm26-nn yolo train --config src/nn/yolo/configs/train_pose.yaml
rm26-nn yolo pose2detect --source <pose_dir> --output <detect_dir>
rm26-nn yolo export --weights <best.pt> --imgsz 640
rm26-nn hrnet convert <data_root> --output <ann_dir>
rm26-nn hrnet train src/nn/hrnet/configs/td-hm_litehrnet18_exchange12_v11.0.py
```

- `hrnet train` requires env vars: `HRNET_DATA_ROOT` and `HRNET_ANN_ROOT` (paths never hardcoded in configs).
- Datasets/checkpoints/outputs are passed via config/CLI/env, never treated as Python packages.
- YOLO train configs instantiate augmentations via `_class_name` factory in `src/nn/utils.py`.

## Architecture (why things are split this way)

- `arm_exchange_core` (`runtime/core`) is deliberately ROS-free numerical code — callers pass NumPy arrays, all API is batch-first `(B, ...)`. No ROS nodes, messages, or timestamps inside.
- Host nodes depend on ROS interfaces, never MuJoCo APIs. `mujoco_simulator` is a single node; arm controller, TF, camera mount, station, operator logic are plugins hosted inside it, ordered by `simulation_config.yaml`.
- The deployed vision path is top-down: YOLO detection → LiteHRNet keypoints → PnP. Planning = Type II (approach) / Type III (constrained assembly) / recovery.
- `/host/arm/feedforward_wrench` is a disabled-by-default research interface, not validated or enabled in the released workflow.
- Frame conventions matter: `arm_base` is the planning/FK root; camera images use OpenCV optical convention (x right, y down, z forward). `src/nn/hrnet/experimental/` is non-mainline and not API-stable.

## Conventions

- No CI, pre-commit, or lint config exists; there is no lint command to run. Verify with `runtime/scripts/test.sh` for runtime changes and `rm26-nn check` for nn changes.
- Keep commit messages consistent with repo history (e.g. `feat: ...`, short descriptive subjects).
