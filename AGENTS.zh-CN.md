# AGENTS.md

RM2026 工程机器人装配：合成数据渲染（`src/blender`）、视觉训练/导出（`src/nn`）以及 ROS 2 Humble + MuJoCo 运行时（`runtime/`）。下文所有路径和命令均以仓库根目录为当前工作目录。`src/nn` 的文档为中文；`runtime/` 的文档为英文。

## 环境：三套，绝不混用

| 工作流 | Python | 安装 / 入口 |
| --- | --- | --- |
| 运行时（ROS 2 + MuJoCo） | 系统 `/usr/bin/python3`（ROS 2 Humble，Ubuntu 22.04） | 通过 `/usr/bin/python3 -m pip install --user` 安装 `runtime/requirements.txt` |
| YOLO / LiteHRNet（训练、导出、合成、可视化） | `src/nn/.venv` 下的 uv 虚拟环境（Python 3.10） | `bash src/nn/setup_env.sh`，随后使用 `rm26-nn <cmd>` |
| Blender 渲染 | Blender 4.5.0 内置 Python 3.11 | `blender -b ... -P src/blender/pipeline/render_dataset.py` |

- 切勿把运行时依赖装进 `src/nn` 的 uv 虚拟环境，也不要从该环境启动 ROS 节点。`runtime/scripts/build.sh` 显式强制使用 `/usr/bin/python3`。
- 首次创建环境必须执行 `bash src/nn/setup_env.sh`，而不是 `uv sync`：该脚本会编译 MMCV CPU 算子并针对 NumPy 2.2 重新编译 `xtcocotools`。只有虚拟环境被删除或 `uv.lock` 变更时才需要重新运行。
- `rm26-nn` 会自动将仓库 `src/` 目录加入 `PYTHONPATH`。虚拟环境未激活时，请使用 `uv run --project src/nn rm26-nn <cmd>`。
- `src/nn/pyproject.toml` 固定了 PyTorch 2.11.0 / cu130 wheel；切勿单独替换锁文件中的某个包——PyTorch / MMCV / MMPose / 显卡驱动之间存在版本耦合。

## 运行时命令

```bash
cp runtime/core/arm_exchange_core/system_config.example.yaml runtime/core/arm_exchange_core/system_config.yaml  # 已被 gitignore，首次构建前必须创建
runtime/scripts/build.sh                      # colcon build --symlink-install，共 5 个包
source runtime/scripts/setup_env.sh           # 必须 source，不能直接执行
runtime/scripts/test.sh                       # colcon test；设置了 PYTEST_DISABLE_PLUGIN_AUTOLOAD=1
ros2 launch arm_exchange_host sim_host.launch.py [enable_perception:=true] [enable_operator_input:=true] [keyboard_device:=/dev/input/eventN]
```

- 测试通过 `colcon`（`runtime/scripts/test.sh`）运行，而不是直接 `pytest`；需要先完成工作区构建。
- OpenVINO 模型权重需单独从 Hugging Face 下载；请在本地 `system_config.yaml` 中配置设备与路径。

## NN 命令（全部子命令见 `rm26-nn --help`）

```bash
rm26-nn check                        # 环境自检
rm26-nn yolo train --config src/nn/yolo/configs/train_pose.yaml
rm26-nn yolo pose2detect --source <pose_dir> --output <detect_dir>
rm26-nn yolo export --weights <best.pt> --imgsz 640
rm26-nn hrnet convert <data_root> --output <ann_dir>
rm26-nn hrnet train src/nn/hrnet/configs/td-hm_litehrnet18_exchange12_v11.0.py
```

- `hrnet train` 需要环境变量：`HRNET_DATA_ROOT` 和 `HRNET_ANN_ROOT`（路径从不硬编码在配置中）。
- 数据集 / 检查点 / 输出均通过配置、CLI 或环境变量传入，不要当作 Python 包使用。
- YOLO 训练配置通过 `src/nn/utils.py` 中的 `_class_name` 工厂实例化增强项。

## 架构（为何如此拆分）

- `arm_exchange_core`（`runtime/core`）刻意不依赖 ROS：调用方传入 NumPy 数组，所有 API 均为 batch-first `(B, ...)`。内部不含任何 ROS 节点、消息或时间戳。
- Host 节点只依赖 ROS 接口，绝不调用 MuJoCo API。`mujoco_simulator` 是单个节点；机械臂控制器、TF、相机挂载、站点、操作员逻辑都是其内部按 `simulation_config.yaml` 顺序排列的插件。
- 已部署的视觉链路是自上而下：YOLO 检测 → LiteHRNet 关键点 → PnP。规划 = Type II（逼近）/ Type III（约束装配）/ 恢复。
- `/host/arm/feedforward_wrench` 是默认关闭的研究用接口，不在发布工作流中验证或启用。
- 坐标系约定很重要：`arm_base` 是规划 / FK 根坐标系；相机图像使用 OpenCV 光心约定（x 向右，y 向下，z 向前）。`src/nn/hrnet/experimental/` 属于非主线，API 不稳定。

## 约定

- 仓库没有 CI、pre-commit 或 lint 配置，因此没有可运行的 lint 命令。运行时改动用 `runtime/scripts/test.sh` 验证，NN 改动用 `rm26-nn check` 验证。
- 提交信息与仓库历史保持一致（例如 `feat: ...`、简短描述性主题）。
