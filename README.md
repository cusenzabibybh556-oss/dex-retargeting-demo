# Rysen ApexHand Real-Time Retargeting

> **源代码未公开**：本仓库仅用于展示 Demo 视频与项目说明（见
> [GitHub Pages 演示页](https://cusenzabibybh556-oss.github.io/dex-retargeting-demo/)）。
> 代码仓库为私有，如需访问请联系作者。下方文档中的相对链接均指向私有仓库文件。

本项目基于开源项目 `dex-retargeting` 开发，用 Intel RealSense、MediaPipe 和
RGB-D 重建人手关键点，DexPilot 算法将动作实时映射到 Rysen
ApexHand。

```text
RealSense / Camera
  → MediaPipe landmarks
  → RGB-D fusion and human-joint estimation
  → Vector / DexPilot retargeting
  → SAPIEN simulation
  → optional Rysen ApexHand hardware
```

项目提供 ApexHand 左右手模型、Vector/DexPilot 配置、RGB-D 感知、碰撞约束、
SAPIEN 可视化和真机控制，支持纯仿真与真实 ApexHand 两种运行模式。

第一次部署必须先运行 Simulation，确认左右手、标定、关节方向和动作范围正确，再添加
`--hardware-ip` 连接真机。

## Environment

- Ubuntu 22.04
- Python 3.10
- Intel RealSense 相机，RGB-D 模式需要
- 支持 OpenGL/Vulkan 的图形环境，SAPIEN viewer 需要
- Rysen ApexHand，只有 Hardware 模式需要
- 机械手与电脑网络互通，只有 Hardware 模式需要

项目包含 ApexHand SDK 1.5.0，支持 Linux CPython 3.10：

```text
third_party/rysen_sdk/lib/
├── x86_64/
└── aarch64/
```

## Installation

以下命令均从项目根目录执行。

```bash
python3.10 -m venv .venv
```

创建项目独立的 Python 虚拟环境，避免依赖污染系统 Python。

```bash
.venv/bin/pip install torch --index-url https://download.pytorch.org/whl/cpu
```

安装 CPU 版本 PyTorch，供优化器和人手约束使用。

```bash
.venv/bin/pip install -e ".[dev,example,rgbd]"
```

以 editable 模式安装项目，并安装测试、Ruff、MediaPipe、SAPIEN、OpenCV 和
RealSense 依赖。

```bash
.venv/bin/python -c "import dex_retargeting; print(dex_retargeting.__file__)"
```

验证 Python 包可以导入并打印实际加载路径；该命令不会打开相机或连接机械手。

```bash
.venv/bin/python -c "from dex_retargeting.hardware.apex import load_apex_sdk; print(load_apex_sdk().__file__)"
```

加载并打印当前 CPU 架构对应的 ApexHand SDK extension；该命令不连接、使能或控制
机械手。

## Simulation

左手 RealSense + DexPilot 仿真：

```bash
.venv/bin/python example/vector_retargeting/show_realtime_retargeting.py \
  --robot-name apex \
  --retargeting-type dexpilot \
  --hand-type left \
  --use-realsense \
  --auto-user-profile \
  --thumb-calibration-path example/vector_retargeting/data/apex_left_rgbd_thumb_calibration.yml
```

该命令读取对齐后的彩色和深度帧，执行左手重定向并在 SAPIEN 中显示 ApexHand。因为没有
`--hardware-ip`，不会创建真机连接或发送关节指令。

右手运行时改为：

```text
--hand-type right
--thumb-calibration-path example/vector_retargeting/data/apex_right_rgbd_thumb_calibration.yml
```

## Hardware

只有在仿真验证通过后，才添加机械手 IP：

```bash
.venv/bin/python example/vector_retargeting/show_realtime_retargeting.py \
  --robot-name apex \
  --retargeting-type dexpilot \
  --hand-type left \
  --use-realsense \
  --auto-user-profile \
  --thumb-calibration-path example/vector_retargeting/data/apex_left_rgbd_thumb_calibration.yml \
  --hardware-ip 192.168.0.102
```

该命令会加载 `ApexHardwareBridge`，连接指定 IP，并通过 SDK 持续发送 Follow 指令。
必须根据实际设备修改 IP。

启动真机前确认：

- `--hand-type` 与物理机器人左右手一致。
- 对应的 thumb calibration 文件存在并已验证。
- 机械手已上电，电脑能够访问机械手 IP。
- 机械手和操作人员周围没有障碍物。
- 仿真中的关节方向、张开/握拳和拇指对掌动作正确。
- 已准备随时停止程序和机械手。

## Main Options

| 参数 | 作用 |
| --- | --- |
| `--robot-name apex` | 加载 ApexHand 模型与配置 |
| `--retargeting-type vector/dexpilot` | 选择基础向量或 DexPilot 重定向 |
| `--hand-type left/right` | 指定人手与机器人手方向 |
| `--use-realsense` | 使用对齐后的 RGB-D 数据 |
| `--auto-user-profile` | 合并四指先验与独立拇指标定 |
| `--recalibrate-four-fingers` | 重新采集四指张开/握拳标定 |
| `--calibration-path` | 四指标定文件路径 |
| `--thumb-calibration-path` | 拇指标定文件路径 |
| `--angle-metrics-path` | 保存逐帧角度、loss 和限位指标 |
| `--hardware-ip` | 启用真机；省略时为纯仿真 |

```bash
.venv/bin/python example/vector_retargeting/show_realtime_retargeting.py --help
```

显示完整命令行选项；该命令不打开相机或连接机械手。

## Runtime Files

运行项目需要以下文件：

```text
assets/robots/hands/apex_hand/        左右手 URDF、STL 和资产说明
sphere_fit_output/                    左右手碰撞球 JSON
src/dex_retargeting/configs/teleop/   四个 Apex 配置
third_party/rysen_sdk/                SDK 头文件、双架构二进制和许可证
example/vector_retargeting/data/      左右手 RGB-D 标定文件
LICENSE                               上游 dex-retargeting MIT 许可证
THIRD_PARTY_NOTICES.md                第三方来源和分发说明
```

SDK 目录中的 `_rysen_sdk...so`、`librysen_sdk.so`、`librysen_sdk.so.1` 和
`librysen_sdk.so.1.5.0` 必须成组保留；不要破坏符号链接。

## Safety

安全处理分为优化层和硬件层：

1. 优化结果裁剪到当前有效关节范围。
2. 发现碰撞风险时优先缩小相关拇指动作。
3. 局部缩放仍不安全时缩小整帧动作。
4. 真机发送前继续执行 SDK 范围、速度、加速度和拇指耦合限制。
5. 通信或发送失败时触发硬件停止保护。

真机安全不能只依赖软件。部署人员仍需提供机械限位、急停、无障碍工作空间和现场监护。

## Project Layout

```text
src/dex_retargeting/
├── configs/teleop/      Apex Vector 和 DexPilot YAML
├── constraints/         人手角度、DexPilot、拇指和握拳约束
├── perception/          RGB-D reconstruction
├── hardware/            ApexHand SDK bridge
├── optimizer.py         Position/Vector/DexPilot optimizer
├── seq_retarget.py      连续帧状态、关节限位和碰撞缩放
└── retargeting_config.py

assets/robots/hands/apex_hand/   Apex URDF 和 meshes
sphere_fit_output/               Apex collision sphere JSON
third_party/rysen_sdk/           ApexHand SDK 1.5.0
example/vector_retargeting/      实时、视频和渲染入口
example/profiling/               性能与准确度分析工具
tests/                           自动测试
docs/                            结构与算法文档
```

进一步阅读：

- `example/vector_retargeting/README.md` 实时与视频示例
- `docs/project_structure.md` 项目结构
- `docs/retargeting_algorithm.md` 重定向算法与安全
- `third_party/rysen_sdk/README.md` ApexHand SDK
- `THIRD_PARTY_NOTICES.md` 第三方声明

## Verification

```bash
PYTHONDONTWRITEBYTECODE=1 .venv/bin/python -m pytest -q -p no:cacheprovider
```

运行全部自动测试并禁止生成 Python/pytest 缓存；不会打开相机或连接机械手。

```bash
.venv/bin/ruff check --no-cache src example tests
```

检查 Python 语法、imports 和代码风格；不会修改文件，也不会连接硬件。

完成自动测试后，必须再运行不带 `--hardware-ip` 的实时仿真。真机验证不能由单元测试
替代。

## Troubleshooting

### Simulation 中没有动作

确认画面持续识别到指定的左手或右手，手掌和指尖完整进入画面，并检查终端是否有
MediaPipe、RealSense 或 calibration 错误。

### Simulation 正常，但真实 ApexHand 不动

检查 `--hardware-ip`、网络连通性和 SDK 日志。只有提供 `--hardware-ip` 才会启动
硬件发送线程。

### 动作幅度突然变小

通常是关节或碰撞限位正在缩小危险动作。使用 `--angle-metrics-path output.npz` 保存
指标，重点检查 `joint_limited`、`collision_limited`、`collision_limit_scope` 和
`collision_limit_alpha`。

### 出现 `ERROR_CODE_OUT_OF_RANGE`

表示目标超出 SDK 范围。程序会记录限速或范围警告并继续保护处理；如果持续出现，应
检查标定、URDF joint limits 和 thumb target，不能直接放宽安全范围。

## License

本项目包含基于 MIT License 的 `dex-retargeting` 代码。根目录
`LICENSE` 中的原作者版权与许可文本必须保留，不能替换成公司版权。

其他组件的许可证和资产说明见：

- `THIRD_PARTY_NOTICES.md`
- `assets/robots/hands/apex_hand/NOTICE.md` Apex asset notice
- `third_party/rysen_sdk/LICENSE` Rysen SDK license
