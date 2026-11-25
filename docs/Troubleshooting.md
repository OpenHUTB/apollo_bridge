# 常见问题与实战排障手册（Apollo v8.0.0 + CARLA 0.9.14）

本文档汇总了在 **Ubuntu 22.04（x86_64）** 物理机上部署、编译并长时运行 **Apollo v8.0.0 + CARLA 0.9.14 + `apollo_bridge`** 过程中实测定位的 **16 个典型故障**，按阶段分类列出完整现象、根本原因与可直接执行的解决方法。

> 完整搭建流程请参阅 [入门与完整部署指南](GettingStarted.md)。

---

## 一、网络、镜像与环境部署类

### 1. `git clone` 报 `gnutls_handshake() failed` 或拉取镜像报 `EOF`
- **现象 / 日志**：终端配置有代理（`http_proxy` / `https_proxy`）时，克隆 Apollo/仓库或 `docker pull` 出现 `gnutls_handshake() failed`、`auth.docker.io ... EOF`。
- **根本原因**：部分本地代理对 GitHub 或 Docker Registry 的 HTTPS 长连接分片 TLS 握手不稳定；另外 `-g cn` 对应的国内部分镜像站也可能触发 SSL EOF。
- **解决方案**：命令行临时取消代理环境变量改为直连（并且不要在 `dev_start.sh` 中盲目加 `-g cn`）：
  ```bash
  env -u http_proxy -u https_proxy -u HTTP_PROXY -u HTTPS_PROXY git clone --depth 1 --branch v8.0.0 https://github.com/ApolloAuto/apollo.git
  ```

### 2. 官方 `dev_start.sh -f -y` 在地图/模型数据卷处失败或占满磁盘
- **现象 / 日志**：运行 `bash docker/scripts/dev_start.sh -f -y` 时在 `map_volume-sunnyvale_big_loop-latest` 等阶段超时或下载失败。
- **根本原因**：`dev_start.sh` 会强制拉取 2 个 Sunnyvale 地图卷和 8 个感知/音频模型卷；而与 `carla_bridge` 联合仿真直接使用自带的 `carla_town*` 地图和仿真真值，完全不需要这 10 个大体积数据卷。
- **解决方案**：直接拉取主开发镜像 `apolloauto/apollo:dev-x86_64-18.04-20221124_1708`，改用 [入门与完整部署指南 §3.2](GettingStarted.md) 中的等效 `docker run` 命令创建容器。

### 3. 全量编译 / GPU 编译报错、`OOM` 或磁盘爆满
- **现象 / 日志**：执行 `./apollo.sh build_gpu` 时，RTX 40 系列（`compute_89`）在 Apollo 8.0 默认 CUDA 11.1 下 nvcc 编译失败；或内存 14–16 GiB 机器出现 OOM Killer / `--disk_cache` 产生双份构建产物撑爆磁盘。
- **根本原因**：Apollo `v8.0.0` 容器内置 CUDA 11.1 不支持 Ada Lovelace 架构（`compute_89`），且全量目标（含感知模型与单测）体积巨大。
- **解决方案**：
  1. 保证物理内存 + Swap 合计 `>= 30 GiB`；
  2. 使用 [入门与完整部署指南 §3.4](GettingStarted.md) 中的 `--config=cpu --disk_cache= --jobs=6 --local_ram_resources=8192`，按需仅编译联合仿真所需的核心模块。

### 4. 启动原生 CARLA 0.9.14 报 `libomp.so.5: cannot open shared object file`
- **现象 / 日志**：解压官方 `CARLA_0.9.14.tar.gz` 后运行 `./CarlaUE4.sh` 立即退出并报缺少 `libomp.so.5`。
- **根本原因**：Ubuntu 22.04 默认未预装 OpenMP 5 运行时库 `libomp5`。
- **解决方案**：有 `sudo` 权限时安装 `sudo apt-get install -y libomp5`；无 `sudo` 权限时通过 `apt-get download libomp5-14 && dpkg-deb -x libomp5-14_*.deb extra_libs` 解压后将其目录前置写入 `LD_LIBRARY_PATH`（详见 [入门与完整部署指南 §5](GettingStarted.md)）。

### 5. 运行 `python3 main.py` 报 `No module named 'transforms3d'` 或 `'carla'`
- **现象 / 日志**：容器内执行 `python3 main.py` 时报缺少 `transforms3d`、`loguru`、`networkx` 或 `carla`。
- **根本原因**：
  1. `install.sh` 通过 `pip3 install -r requirements.txt` 将依赖安装在普通用户 `$HOME/.local/lib/python3.6/site-packages` 下，如果用默认的 `docker exec`（`root` 身份）启动会找不到普通用户包目录；
  2. 非交互式 `docker exec ... bash -c` 不会自动读取 `~/.bashrc` 里的 `PYTHONPATH`。
- **解决方案**：启动仿真桥时务必指定 `-u "$USER" -e HOME="/home/${USER}"`，并在启动命令中显式导出完整 `PYTHONPATH`（包含 `carla-0.9.14-py3.7-linux-x86_64.egg` 与 `/apollo/bazel-bin/cyber/python/internal`）。

---

## 二、控制器编译与运动行为类

### 6. 编译合入后的 `refined_controller` 报 `vehicle_config.pb.h: No such file or directory`
- **现象 / 日志**：替换 `refined_controller/` 下 3 个文件后编译 `//modules/control:libcontrol_component.so`，`mpc_controller.h` 第 29 行编译失败。
- **根本原因**：`carla_bridge/refined_controller/mpc_controller.h` 第 29 行沿用了早期 Apollo 的头文件旧路径 `#include "modules/common/configs/proto/vehicle_config.pb.h"`，在 Apollo `v8.0.0` 中已迁移至 `modules/common_msgs/config_msgs/`。
- **解决方案**：将 `<WORKDIR>/apollo/modules/control/controller/mpc_controller.h` 第 29 行修改为 `#include "modules/common_msgs/config_msgs/vehicle_config.pb.h"` 后重编控制组件。

### 7. 下发路由后 Control 一直锁死在 `throttle=0, brake=50`，车完全不动
- **现象 / 日志**：路线已生成、Planning 已发布轨迹，但 `/apollo/control` 一直输出 `throttle = 0`、`brake = 50`，日志提示 `estop for empty planning trajectory`。
- **根本原因**：Apollo `control_conf.proto` 默认 `enable_persistent_estop = true`。只要在等待路由的初期出现过空轨迹，`ControlComponent` 内部 `estop_` 就会永久锁死为 `true`，之后即使出现合法轨迹也不会自动解除。
- **解决方案**：在 `<WORKDIR>/apollo/modules/control/conf/control_conf.pb.txt` 中加入 `enable_persistent_estop: false`（并注意 Dreamview 中不要选择自带旧配置覆盖该文件的 `Lincoln2017MKZ LGSVL` 车型）。

### 8. 待命态（未下发路线）或到站停车后，车辆以 `throttle=15.70%` 缓慢向前蠕行
- **现象 / 日志**：无导航路线（Planning `not_ready=True`）或已到达终点后，`Control` 依然输出 `throttle=15.70, brake=0.00`，导致自车以 `0.3~0.8 m/s` 缓慢向前滑行甚至撞入路口。
- **根本原因**：
  1. `refined_controller/control_conf.pb.txt` 的 `mpc_controller_conf.calibration_table` 在 `speed: 0.0` 处直接从 `(acceleration: -0.48, command: -2.82)` 跨跳至 `(acceleration: 0.09, command: 30.06)`。原有的 `standstill_acceleration: -0.3` 线性插值后落在正区间（`+7.56`），被 `vehicle_param_.throttle_deadzone()`（`15.7`）钳位为 `15.7%` 正油门（真实车轮死区内不动，CARLA 仿真车会直接起步）；
  2. Planning 在 `not_ready=True` 时发布的 30 点静态轨 `trajectory_type` 为 `UNKNOWN (0)`，绕过了 `mpc_controller.cc` 仅针对 `NORMAL` 轨迹的停驻保护。
- **解决方案**（详见 [入门与完整部署指南 §4.1](GettingStarted.md)）：
  1. 将 `mpc_controller_conf` 中的 `standstill_acceleration` 改为 `-1.0`；
  2. 在 `control_component.cc` 中增加 `local_view_.trajectory().decision().main_decision().has_not_ready()` 制动判断并重编 `libcontrol_component.so`。

---

## 三、Dreamview、模块启停与选点下发类

### 9. 网页端 `Module Controller` 点开关拉不起模块，或启动报 `libc10.so not found`
- **现象 / 日志**：在 Dreamview 左侧点击打开 `Planning` / `Prediction` 等开关后立即自动弹回关闭；或命令行 `cmd1 & cmd2 &` 启动后台模块时 `data/log/` 报错 `libc10.so: cannot open shared object file` / `CYBER_PATH not set`。
- **根本原因**：
  1. Bash 中 `source setup.bash && cmd1 & cmd2 &` 的 `&` 优先级低于 `&&`，`cmd2` 及之后进程未继承环境；
  2. `./scripts/dreamview.sh` 启动时会再次 `source apollo.bashrc`，在检测到 NVIDIA GPU 的主机上强制把 `/usr/local/libtorch_gpu/lib` 前置到 `LD_LIBRARY_PATH`，与 `--config=cpu` 编译的二进制冲突。
- **解决方案**：按 [入门与完整部署指南 §6.1](GettingStarted.md) 第 1 步，在 `source /apollo/scripts/apollo_base.sh` 之后显式执行 `export LD_LIBRARY_PATH="/usr/local/libtorch_cpu/lib:${LD_LIBRARY_PATH}"`，然后直接调用 shell 函数 `start monitor && start dreamview`。

### 10. 切换顶部地图/车型后车辆不走，或提示 `You haven't selected a vehicle yet!`
- **现象 / 日志**：原本模块已开启，在 Dreamview 顶部切换了一次地图（例如刷新地图切到 `Carla Town04` 再切回 `Carla Town01`）或切换模式/车型后，点 `Send Routing Request` 车没反应；或者选中 `Lincoln2017MKZ LGSVL` 后出现 `brake=50` 锁死。
- **根本原因**：
  1. Dreamview 后端 `HMIWorker::ChangeMap` / `ChangeVehicle` / `ChangeMode` 在切换时会调用 `ResetMode()` 自动停掉全部后台模块；
  2. `modules/calibration/data/Lincoln2017MKZ_LGSVL/` 目录自带有未调优、未关闭 `enable_persistent_estop` 的 `control_conf.pb.txt`，选中该车型会覆盖全局控制配置。
- **解决方案**：固定使用 **`Carla Bridge` \| `Mkz Example` \| `Carla Town01`**；若中途不小心切过顶部下拉框，切回上述三项后，直接在左侧 **`Module Controller`** 把 5 个模块开关重新点亮即可。

### 11. Dreamview 弹出红框 `Storytelling triggers safe mode:` 及 `System fault. Auto disengage.`
- **现象 / 日志**：Dreamview 顶部右侧长期弹出红色告警 `Storytelling triggers safe mode:`，且 `monitor` 日志出现 `System fault. Auto disengage.`。
- **根本原因**：`carla_bridge` 默认向 `/apollo/canbus/chassis` 上报 `COMPLETE_AUTO_DRIVE`，触发 `FunctionalSafetyMonitor::CheckSafety()` 检查当前 HMI 模式下的所有模块；由于 `hmi_mode.proto` 默认 `required_for_safety: true`，在 `Mkz Standard Debug` 模式下任何未启动的实车模块（`Storytelling`、`Camera`、`Canbus`、`Velodyne`、`GPS` 等）都会触发安全模式停机。
- **解决方案**：按照 [入门与完整部署指南 §4.2](GettingStarted.md) 创建精简的 5 模块模式 `/apollo/modules/dreamview/conf/hmi_modes/carla_bridge.pb.txt`，并在 Dreamview 中切换至 **`Carla Bridge`** 模式并打开全部 5 个开关。

### 12. 车辆前方出现白色带 X 的 `STOP` 墙 + 橙色沙漏，仪表盘显示 `Brake 50%`
- **现象 / 日志**：仿真正常启动后，Dreamview 中蓝色自车前出现白色 `STOP` 网格墙与橙色沙漏图标，车速 `0 km/h`。
- **根本原因**：这是 **尚未下发路由时的正常待命画面**。未收到有效路由时 Planning 发布 `not_ready`（Dreamview `UpdateMainStopDecision` 据此在自车位置渲染 `STOP_REASON_NOT_READY` 沙漏墙），同时 `Control` 依第 8 条补丁输出 `brake = 50%` 防止漂移。
- **解决方案**：无需特殊处理，确认处于 `Carla Bridge` 模式且 5 个模块开启后，通过 `Tasks → Default Routings` 或 `Route Editing` 下发合法终点，沙漏会自动清除并起步。

### 13. 在 `Route Editing` 选终点后点 `Send Routing Request` 车不走
- **现象 / 日志**：点击 `Send Routing Request` 后无路由线出现，查看 `data/log/dreamview.INFO` 可见：
  - `Expected lane ... to be CITY_DRIVING, but was SHOULDER`
  - `Selected point cannot be a routing point.`
  - `Failed to prepare a routing request with heading: ... cannot locate start point on map.`
- **根本原因**：在地图上点偏到边缘白色路肩（`SHOULDER`）、对向车道，或者误点 `Set Start Point` 改变了起点朝向导致与道路航向角偏差过大。
- **解决方案**：
  1. 推荐直接使用预置文件 `default_end_way_point.txt`，在左侧 **`Tasks → Default Routings`** 一键下发；
  2. 手动 `Route Editing` 时**千万不要点 `Set Start Point`**，只点 **`Add Point of Interest`**，放大地图并在自车前方**右侧正常行车道（绿色中心线内）**左键单击设置终点后再发送请求。

---

## 四、CARLA 仿真稳定与运维安全类

### 14. 容器内执行 `pkill -f` 误杀宿主机脚本或终端进程
- **现象 / 日志**：在容器内用 `pkill -f "python3 main.py"` 等模糊命令清进程时，把宿主机上同关键字的任务/脚本一并杀掉。
- **根本原因**：容器以 `--pid=host` 启动，容器与宿主机共用同一套 PID 命名空间。
- **解决方案**：严禁在容器内使用模糊 `pkill -f`，停进程时先通过 `ps -eo pid,args | grep ...` 精准查出目标 PID，再执行 `kill <PID>`。

### 15. `SIGTERM` 强杀仿真桥后再重连或休眠唤醒后 CARLA `Signal 11` 崩溃，或 `get_actors()` 为空
- **现象 / 日志**：
  1. 用 `SIGTERM`/`SIGKILL` 杀掉运行中的 `carla_bridge` 后再次启动 `main.py`，或宿主机休眠唤醒后连接，CARLA 在 `load_world('Town01')` 瞬间段错误崩溃；
  2. 独立 Python 脚本连上 CARLA 调用 `world.get_actors()` 返回 `[]`。
- **根本原因**：
  1. `SIGTERM` 未触发 `main.py` 的 `finally:` 清理逻辑，上一轮开启的 `synchronous_mode` 与 GPU 传感器尚挂在 CARLA 内存中时二次 `load_world` 易引发 UE4 崩溃；
  2. 新建 `carla.Client` 尚未等待一帧世界快照同步。
- **解决方案**：停止 `carla_bridge` 先发 `kill -INT <PID>`（触发清理同步模式与传感器）；若已异常崩溃，连同 CARLA 一起成对重启。外部脚本读取 Actor 前先调用 `world.wait_for_tick(5.0)`。

### 16. 行车约 30 秒后 CARLA 窗口突然闪退，报 `NVRM Xid 31` / `VK_ERROR_DEVICE_LOST`
- **现象 / 日志**：仿真开始行驶约 31–33 秒后 CARLA 3D 窗口直接闪退，`carla_0.9.14.log` 报：
  `VulkanRHI::vkDeviceWaitIdle(Device) failed, VkResult=-4 ... VK_ERROR_DEVICE_LOST` 与 `Signal 11`；
  系统内核日志（`journalctl -k`）在崩溃前 1 秒出现：
  `NVRM: Xid (PCI:0000:01:00): 31 ... ENGINE GRAPHICS GPC2 GPCCLIENT_PROP_0 faulted @ 0x0_00004000. Fault is of type FAULT_PDE`。
- **根本原因**：部分新版 NVIDIA 驱动（如 `595.xx`）在 AMD+NVIDIA 双显卡笔记本的 Ubuntu Wayland / Xwayland PRIME 环境下运行 CARLA 0.9.14（UE4 4.26 Vulkan）时存在显存页面映射缺陷（即使调低相机分辨率与帧率仍会在行车约 30 秒后复现）。
- **解决方案**：
  1. **双显卡推荐（保留 3D 窗口）**：去掉 `__NV_PRIME_RENDER_OFFLOAD=1` 与 `-prefernvidia`，设置 `VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/radeon_icd.x86_64.json`，改由 AMD Radeon 核显（RADV Vulkan）跑 CARLA 窗口（实测 62 秒连续行驶 200 米零故障、定位稳定 ~10.3 Hz）；
  2. **无头模式备选**：若机器无核显或不需桌面 3D 窗口，可在 CARLA 启动参数中使用 `-RenderOffScreen`，通过 Dreamview 观察车辆与相机画面。
