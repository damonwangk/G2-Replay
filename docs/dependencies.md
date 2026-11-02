# 回放仓库依赖布局说明

本仓库保留 G2 部署时的相对目录，原回放和独立限幅平滑副本的核心执行代码可以按原样同步。发布整理只新增说明、Python 依赖和测试辅助，不把厂商 SDK 或私有设备配置带入 Git。

## 必须保留的跨目录文件

| 文件 | 作用 |
|---|---|
| `replay/source_episode.py` | 将 MCAP / 原生逐话题控制指令组织为原时间事件；运行时向 `../conversion` 添加导入路径 |
| `conversion/recording_contract.py` | 24 维名称、原始指令/反馈话题和时间戳解析 |
| `conversion/mcap_input.py` | MCAP protobuf、原始相机包与指令读取 |
| `conversion/factory_input.py` | 机器人原生 pbdat 和相机索引读取 |
| `conversion/convert_r2.py` | 原生读取器延迟使用其 `camera_index()`；回放不调用转换器写入入口 |
| `studio/schemas.pb` | 原生 protobuf 消息类构造；是格式描述符，不是 SDK 二进制 |
| `conversion/test_mcap_input.py` | 回放测试的合成 MCAP / JPEG 生成器 |
| `conversion/test_factory_input.py` | 回放测试的合成 pbdat 生成器 |

`factory_input.py` 固定从 `../studio/schemas.pb` 读取描述符，移动这些目录时必须同时修改路径。保留这里的布局即可避免改变核心读取逻辑。完整格式转换产品在 [G2-Converter](https://github.com/damonwangk/G2-Converter)。

## Python 与厂商 SDK 分开准备

`requirements.txt` 仅安装公开的读取/测试依赖。真实执行中的控制权交接使用 `grpcio` 与厂商 `rh_msgs_v4_pb.operator_event_pb2`；位置发送使用机器人本机 `agibot_gdk`。动态库、配套 protobuf 生成文件及 SDK 配置使用现场已经安装的版本，不在本仓库分发。

两个 Linux shell 默认查找 `conversion/.venv/bin/python`，找不到时回落 `/usr/bin/python3`。采用本仓库根目录独立虚拟环境时，必须先设置：

```bash
export R2_REPLAY_PYTHON="$PWD/.venv/bin/python"
```

shell 同时查找 `R2_SDK_APP` 指向的 SDK 树，默认 `/home/agi/app`。没有 SDK 的开发机只读验证直接调用 Python `--dry-run`，不要依赖 shell 设置 SDK 环境。

设备连接配置使用 `studio/config/robot_control.example.json` 复制生成，真实 `robot_control.json` 已被 Git 忽略。所需键为 `host`、`operator_port`、`robot_aid`；数值取自现场配置。

## 测试限位夹具

部署版本 `replay/test_clamped_smooth.py` 原本优先读取相邻厂商 SDK 树中的 `mc_impl_config.json`，其次读取机器人安装路径。这不适合无 SDK 的公开仓库回归。

本发布辅助目录提供 `tests/fixtures/synthetic_gdk_limits.json`。其中 24 轴来自公开代码中的关节名称，普通关节采用测试用范围 `[-2, 2]`；`idx66_arm_r_joint6` 使用该测试已有断言的 `[-1.0123, 1.0123]`，夹爪使用该测试已有断言的 `[0, 0.025]`。这是单元测试数据，不能当作真实机器人的 SDK 限位配置。

本发布版本已调整测试文件里 `CONFIG` 的选择（不改核心回放代码）：

```python
import os
CONFIG = Path(os.environ.get(
    'R2_TEST_GDK_LIMITS',
    str(Path(__file__).resolve().parent.parent /
        'tests/fixtures/synthetic_gdk_limits.json'),
))
```

这样普通离线回归不依赖 SDK；若需在已安装 SDK 的环境核对配置解析，可显式设置 `R2_TEST_GDK_LIMITS=/home/agi/app/gdk/config/mc_impl_config.json`。Fake SDK 测试仍不初始化真实 SDK。

夹具 JSON 只用于测试，Linux 真实执行与独立副本默认仍读取实际安装的 GDK 配置。不要将测试夹具复制为机器人的生产配置，也不要把限幅副本的目标处理混进原回放入口。
