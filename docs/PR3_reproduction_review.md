# PR #3 复现记录与评审意见

> 复现对象：`OpenHUTB/apollo_bridge` PR #3（已合并）
> 复现环境：Windows 11 + WSL2 Ubuntu 22.04（9.7 GiB RAM / 16 GiB swap）
> 复现时间：2026-11-25 ～ 2026-11-26

## 1. 复现结论（先说结果）

| 环节 | 结果 | 证据 |
| --- | --- | --- |
| Apollo v8.0.0 源码 | ✅ 完成 | `--depth 1 --branch v8.0.0` |
| Apollo 开发镜像 | ✅ 完成 | `apolloauto/apollo:dev-x86_64-18.04-20221124_1708` |
| 精简模块编译 | ✅ 完成 | **2988 actions，Build completed successfully** |
| Dreamview | ✅ 完成 | `http://127.0.0.1:8888/` 返回 200；`Carla Bridge` 模式可识别 |
| `carla_bridge` 启动 | ✅ 能启动并连到 RPC 2000 | 日志：`Trying to connect to 127.0.0.1:2000` |
| CARLA 0.9.14 模拟器 | ❌ 未拿到 | S3 403；Docker Hub 8.8 GB 层反复断；南科大镜像被 Anubis 拦截 |
| 端到端闭环 | ❌ 未完成 | 缺 CARLA 0.9.14；OpenHUTB 2.10.1 与 0.9.14 客户端不兼容 |

**一句话**：Apollo 侧已经复现成功；模拟器侧被 CARLA 0.9.14 的下载挡住，不是代码/编译问题。

## 2. 关键发现：WSL 不是“崩”，是最后一个会话退出后自动关机

之前多次看到 Bazel 编译 2–4 分钟后 WSL `uptime` 归零、容器 `Exit 255`，一度判断为 WSL 不稳定。实际日志：

```text
WSL (2 - init-systemd(Ubuntu-22.04)) ERROR:
InitTerminateInstanceInternal:2928:
systemctl poweroff did not terminate the instance in 10000 ms,
calling reboot(RB_POWER_OFF)
```

原因：编译是用 `docker exec -d` 起的，启动命令很快退出；WSL 在最后一个 `wsl.exe` 会话结束后把发行版整个关掉，编译进程被一起带走。

**解决办法**：保持一个常驻会话，例如：

```bash
# 另开一个终端/后台任务，保持 WSL 活着
wsl -d Ubuntu-22.04 -u root -- bash -lc "while true; do sleep 60; done"
```

之后同样的 2988 actions 编译一次通过。这一点建议补进 PR #3 的文档，作为“子系统/WSL2”章节的核心注意事项。

## 3. 已实际执行的步骤

### 3.1 恢复 WSL 与 Docker

```bash
wsl --import Ubuntu-22.04 D:\wsl\Ubuntu-22.04 <rootfs.tar.xz> --version 2
# /etc/wsl.conf 开启 systemd
[boot]
systemd=true
```

Docker 用官方静态包（`docker-24.0.9.tgz`）安装，`dockerd` 由 systemd 拉起。

### 3.2 Apollo 源码与补丁

```bash
git clone --depth 1 --branch v8.0.0 https://github.com/ApolloAuto/apollo.git /root/apollo
cp -r carla_bridge /root/apollo/modules/carla_bridge
python3 tools/apply_pr3_patches.py
```

补丁脚本按 PR #3 文档落实了：

- `refined_controller` 三个文件；
- `mpc_controller.h` 头文件路径改为 `modules/common_msgs/config_msgs/vehicle_config.pb.h`；
- `enable_persistent_estop: false`；
- `mpc_controller_conf.standstill_acceleration: -1.0`；
- `control_component.cc` 的 `not_ready` 制动保持；
- 5 模块 `carla_bridge.pb.txt`；
- `--map_dir=/apollo/modules/map/data/carla_town01`；
- Town01 三个预设终点。

### 3.3 精简编译

```bash
bazel build --config=cpu --disk_cache= --copt=-mavx2 --host_copt=-mavx2 \
  --jobs=2 --local_ram_resources=4096 \
  //cyber/mainboard:mainboard //cyber/tools/... //cyber/python/cyber_py3/... \
  //modules/common_msgs/... //modules/data/proto/... //modules/dreamview:dreamview \
  //modules/monitor:libmonitor.so //modules/routing:librouting_component.so \
  //modules/localization/rtk:librtk_localization_component.so \
  //modules/localization/msf:libmsf_localization_component.so \
  //modules/prediction:libprediction_component.so \
  //modules/planning:libplanning_component.so \
  //modules/control:libcontrol_component.so \
  //modules/transform:libstatic_transform_component.so \
  //modules/storytelling:libstorytelling_component.so
```

结果：`Build completed successfully, 2988 total actions`。关键产物齐全：

```text
bazel-bin/cyber/mainboard/mainboard
bazel-bin/modules/dreamview/dreamview
bazel-bin/modules/control/libcontrol_component.so
bazel-bin/modules/planning/libplanning_component.so
```

### 3.4 Dreamview

```bash
source /apollo/scripts/apollo_base.sh
export LD_LIBRARY_PATH=/usr/local/libtorch_cpu/lib:$LD_LIBRARY_PATH
start monitor
start dreamview
```

`curl http://127.0.0.1:8888/` 返回 200；Dreamview 顶部可以选到自定义的 **Carla Bridge** 模式、**Mkz Example** 车型、**Carla Town01** 地图。

![Dreamview 启动](images/apollo_dreamview_wsl.png)

![Carla Bridge 模式](images/apollo_dreamview_carla_bridge.png)

### 3.5 桥连接模拟器

`config/settings.yaml` 的 host 从默认 `172.17.0.1` 改成 `127.0.0.1`（容器 `--net host` + WSL 镜像网络），桥能连到 2000 端口：

```text
Trying to connect to 127.0.0.1:2000
WARNING: Version mismatch detected ...
WARNING: Client API version     = 0.9.14
WARNING: Simulator API version  = 3ea2130ed
terminate called after throwing an instance of 'std::bad_alloc'
```

这说明桥本身已经走到“连接模拟器”这一步，失败点是 **CARLA 客户端版本不匹配**。

## 4. 模拟器侧的问题

### 4.1 CARLA 0.9.14 下载受阻

已尝试：

| 渠道 | 结果 |
| --- | --- |
| 官方 S3 `carla-releases.s3...` | 301/403（桶策略或网络限制） |
| Docker Hub `carlasim/carla:0.9.14` | 8.8 GB 主层反复 `Retrying...`，无法完成 |
| 南科大镜像 `mirrors.sustech.edu.cn/carla/` | Anubis 反爬拦截（错误码 `4d1dbaddfcc0f385`） |
| 其他高校镜像 | 无 `/carla` 路径 |

因此当前环境拿不到 CARLA 0.9.14。

### 4.2 OpenHUTB 2.10.1 不能直接替代

本机 OpenHUTB 2.10.1 可以正常启动（Town10HD、24 个 actor、RPC 2000），但桥自带的 `carla-0.9.14-py3.7` 客户端连它会在版本校验后 `std::bad_alloc`。也就是说：

- PR #3 的基线必须用 **CARLA 0.9.14**；
- 要跑 OpenHUTB，需要换成 OpenHUTB 自己的 Linux 客户端（或做协议适配），这不是改一个 host 能解决的。

## 5. 对 PR #3 文档的评审意见

### 5.1 值得肯定

- 精简模块编译的目标列表很实用，确实把 3492/2988 actions 控制住了；
- `refined_controller` 头文件路径、`enable_persistent_estop`、`standstill_acceleration`、`not_ready` 制动这 4 个补丁都打在关键点上；
- `carla_bridge.pb.txt` 5 模块 HMI 模式解决了安全模式红框；
- 排障手册的 16 条现象—原因—解决结构清楚，可直接复用。

### 5.2 建议补充/修正

1. **补 WSL2 章节**：这是老师明确要的“子系统”部分。至少写清楚：
   - 最后一个 `wsl.exe` 会话退出会触发 `systemctl poweroff`，编译必须用常驻会话；
   - `networkingMode=mirrored` 下 Windows 的 `127.0.0.1` 可从 WSL/容器访问；
   - Docker 需要单独配代理，国内拉 Docker Hub 会卡大层；
   - WSL 内存/swap 建议（本机 9.7 GiB + 16 GiB swap 可编译 2988 actions）。
2. **`deployment_introduction.md` 仍引用不存在的路径**：正文里的 `carla_apollo_bridge/scripts`、`carla_apollo_bridge/docker` 在当前仓库已不存在；顶部虽有提示，但正文容易误导，建议直接改成 `carla_scripts/` 并删掉旧的三容器描述。
3. **`install.sh` 的清华源**：`requirements.txt` 第一行是 `--index-url https://pypi.tuna.tsinghua.edu.cn/simple`，部分网络会返回 403（本次复现就遇到）。建议在文档里加 `-i https://pypi.org/simple` 的回退，或直接删掉这行。
4. **精简目标还能再砍**：文档说“不要开 Localization”，但编译目标里仍包含 `//modules/localization/rtk` 和 `//modules/localization/msf`，可以删掉，进一步省时间和磁盘。
5. **`--pid=host` 的风险**：文档自己也警告了 `pkill` 会误杀宿主机。桥和 Dreamview 并不需要共享 PID 命名空间，建议默认去掉 `--pid=host`，只在确实需要时加。
6. **CARLA 0.9.14 的获取方式**：官方 S3 链接在当前网络不可用，南科大镜像又被 Anubis 拦截。建议文档给一个**离线包/网盘/内部制品**的兜底，而不是只写官方链接。
7. **Python 3.6 + py3.7 egg 的说明可以更明确**：本次实测确实能 `import carla`（`libcarla.py` 用 `importlib` 直接加载 `.so`），但这是“能跑但不受官方支持”的边界，建议注明“若报 ABI/符号错误，升级到 Python 3.7 或换 OpenHUTB 客户端”。

## 6. 下一步

1. 拿到 CARLA 0.9.14 的离线包（或可用镜像）后，按本文第 3 节直接启动，即可完成 PR #3 的端到端复现；
2. 若要继续 Issue #1 的第 2 条（Apollo + OpenHUTB），需要把 `carla_api` 换成 OpenHUTB 的 Linux 客户端，或让 OpenHUTB 提供 0.9.14 兼容的 RPC 层；
3. 本文的 WSL 常驻会话、Docker 代理、`127.0.0.1` 连接方式可以直接补进 PR #3 文档的“子系统”章节。
