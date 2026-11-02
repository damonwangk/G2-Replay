# G2-Replay：G2 机器人数据回放

读取 LeRobot v2.1、采集平台原始 MCAP 和机器人原生离线 `full_*`，通过机器人已安装的厂商 GDK 回放控制指令。保留原回放与独立限幅平滑副本两套入口，源码按机器人部署布局整理；不包含厂商 SDK、真实数据或设备私有配置。

| 仓库 | 职责 |
|---|---|
| [G2-DataCP](https://github.com/damonwangk/G2-DataCP) | 原始采集、网页、数据管理，以及转换/回放的页面调度 |
| **G2-Replay** | MCAP / 原生离线记录 / LeRobot 的独立回放命令及离线测试 |
| [G2-Converter](https://github.com/damonwangk/G2-Converter) | 原始 MCAP / 原生离线记录 → LeRobot v2.1 MP4 |

## 目录与依赖

```text
replay/                 原回放、独立限幅平滑副本、只读信息工具、Fake SDK 测试
conversion/             回放复用的原始记录读取器与合成测试数据工具
studio/schemas.pb       原生 pbdat 解析所需的 protobuf 描述符
studio/config/          设备配置模板；真实配置由操作者在本地准备
requirements.txt        独立读取与测试依赖，不安装厂商 SDK
```

`conversion/` 在这里是保持相对路径兼容的读取依赖，不是完整转换产品；格式转换使用 G2-Converter。源码依赖、测试夹具和 SDK 配置说明见 [依赖布局说明](docs/dependencies.md)。

## 安装独立 Python 环境

在本仓库根目录执行，建议 Python 3.10 或 3.11：

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
```

只读取数据不需要初始化机器人 SDK。下面的 `--dry-run` 直接调用 Python，避免 Linux shell 入口尝试设置本机不存在的 SDK 动态库路径。

## 原始指令回放：先读取，或由操作者执行

LeRobot 读取数据集根目录与具体 `episode`：

```bash
.venv/bin/python replay/lerobot_replay.py \
  --dataset /path/to/dataset --episode 0 \
  --dry-run --run /tmp/lerobot_read_run
```

MCAP 与机器人原生离线记录都走原始事件读取器，原入口参数名为 `--native-recording`：

```bash
.venv/bin/python replay/lerobot_replay.py \
  --native-recording /path/to/episode/raw.mcap \
  --dry-run --run /tmp/mcap_read_run

.venv/bin/python replay/lerobot_replay.py \
  --native-recording /path/to/full_20260314000000 \
  --dry-run --run /tmp/native_read_run
```

每次用新的 `--run` 目录，结果为 `progress.json`、`result.json`，真实执行另有 `trace.jsonl`。`--dry-run` 不导入运动 SDK，不交接控制权、不发送动作；它不是物理跟随验收。

原回放读取原始控制指令，反馈用于对照记录；不会把反馈自动补成动作。名称按元数据匹配，原始时间、已有位置和 NULL 保留。没有新事件期间按 100 Hz 保持已有目标；详细行为见 [原回放说明](replay/README.md)。

## 机器人本机真实执行

在已安装并可正常运行厂商 SDK 的机器人 Linux 上配置：

```bash
cp studio/config/robot_control.example.json studio/config/robot_control.json
# 编辑 host、operator_port、robot_aid，填写本机实际配置。
export R2_SDK_APP=/home/agi/app
export R2_REPLAY_PYTHON="$PWD/.venv/bin/python"
```

SDK Python 模块 `agibot_gdk`、控制权交接 protobuf 模块 `rh_msgs_v4_pb` 和对应动态库需要由机器人的现有 SDK 环境提供；不要从其他架构复制二进制。若 `rh_msgs_v4_pb` 安装在独立目录，按本机部署把它加入 `PYTHONPATH`。仓库不包含 SDK 安装程序。

以下命令会交接控制权并发送动作，请由现场操作者选择所需一条执行：

```bash
./replay/replay.sh --dataset /path/to/dataset --episode 0 --run /path/to/new_run
./replay/replay.sh --native-recording /path/to/episode/raw.mcap --run /path/to/new_run
./replay/replay.sh --native-recording /path/to/full_xxx --run /path/to/new_run
```

`--config /path/to/robot_control.json` 可覆盖默认本地配置。默认 SDK 控制接口为 `--sdk-api struct`；只有确认本机 API 使用数值模式时才选择现有 `--sdk-api mode-id`。实际入口不会安装或替换厂商 SDK。`Ctrl+C` 停止本次进程继续发送。

## 独立限幅平滑副本

该副本仅在发送前变换目标，原回放入口与源文件保持原样。它读取当前机器人的限位配置，并使用独立的逐轴限幅/限速规则；详细参数、NULL 与日志规则见 [副本说明](replay/README_clamped_smooth.md)。

只读检查时显式给出本机配置文件：

```bash
.venv/bin/python replay/lerobot_replay_clamped_smooth.py \
  --dataset /path/to/dataset --episode 0 \
  --gdk-limits /path/to/installed/mc_impl_config.json \
  --dry-run --run /tmp/clamped_read_run
```

真实执行仍由已安装 SDK 的 Linux 入口启动：

```bash
./replay/replay_clamped_smooth.sh \
  --native-recording /path/to/full_xxx --run /path/to/new_run

./replay/replay_clamped_smooth.sh \
  --mcap /path/to/episode/raw.mcap --run /path/to/new_run
```

只有此独立副本支持 `--mcap` 简写、`--gdk-limits`、`--joint-speed` 与 `--gripper-speed`，原回放没有这些参数。

## 只读查看数据与离线测试

```bash
.venv/bin/python replay/dataset_info.py --dataset /path/to/dataset
.venv/bin/python replay/dataset_info.py --dataset /path/to/dataset --episode 0
.venv/bin/python replay/native_episode.py --source /path/to/full_xxx
```

Fake SDK 回归不连接机器人：

```bash
PYTHONPATH="$PWD/conversion" \
  .venv/bin/python -m unittest discover -s replay -p 'test_*.py' -v
```

独立限幅副本的测试限位夹具准备方法见 [依赖布局说明](docs/dependencies.md#测试限位夹具)。`completed` 表示源码事件/时间轴已处理，SDK 接受次数、拒绝次数与实际反馈分别记录，不把软件完成标志当作已确认的机械跟随结论。
