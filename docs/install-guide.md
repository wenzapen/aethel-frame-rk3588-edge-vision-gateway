# 安装指南

本指南面向负责现场部署与运维的工程人员，说明如何在预装厂商镜像的 RK3588 设备上安装、启动 AethelFrame，并完成首次登录与视频预览。

---

## 1. 目标机要求

| 项 | 要求 | 检查命令 |
|---|---|---|
| 芯片 / 架构 | Rockchip **RK3588**，aarch64 | `uname -m` → `aarch64` |
| 发行版 | **Ubuntu 24.04.x LTS**（板厂镜像） | `lsb_release -ds` |
| 内核 | **6.1.x**（板厂提供，含 MPP/RGA/NPU 驱动） | `uname -r` |
| GStreamer | **1.24.2** | `gst-inspect-1.0 --version` |
| Rockchip MPP | **1.5.0**（`librockchip_mpp.so.0`） | `dpkg -l \| grep mpp` |
| librga | **2.2.0**（`librga.so.2`） | `ls -l /usr/lib/aarch64-linux-gnu/librga.so*` |
| RKNPU 运行时 | **librknnrt 2.3.2** | `ls -l /usr/local/lib/librknnrt.so` |
| 硬件媒体基座 | 随板厂镜像安装内核驱动与运行时库；**媒体处理插件（含码流分支与模型后处理组件）全部随交付包携带**，目标机无需另行安装（安装脚本会自动体检） | `ls /usr/lib/aarch64-linux-gnu/librockchip_mpp.so*` |
| mediamtx | **v1.13.1**（部分交付包已内置，见发布说明） | `mediamtx --version` |
| mosquitto | 2.x（仅 MQTT 出口需要） | `mosquitto -h \| head -1` |
| 磁盘 | 系统盘 ≥ 4 GB 可用；**录像盘建议单独分区**（默认位于 `~/aethel-frame/data`；使用独立数据盘时请将配置项 `record.path` 指向挂载点） | `df -h "$HOME"` |
| 内存 | 2–4 路 ≥ 4 GB；8–12 路 ≥ 8 GB | `free -m` |

> ⚠️ **板厂 SDK 必须来自同一厂商的同一份镜像**。混装不同来源的 MPP / RGA / RKNN 库时，组件虽能被
> `gst-inspect` 正常检出，运行时仍会崩溃。升级这些库之前请先备份 `/usr/lib/aarch64-linux-gnu/librockchip_mpp.so*`。

> 视频源要求：H.264 / H.265 码流均可接入（由本设备硬件解码）。**浏览器显示的画面由本设备重新编码输出**，
> 因此对摄像机码流的 profile 与有无 B 帧无要求；摄像机侧只需保证网络可达与访问凭据正确。

---

## 2. 装依赖

```bash
sudo apt update
sudo apt install -y \
  libgstreamer1.0-0 gstreamer1.0-plugins-base gstreamer1.0-plugins-good \
  gstreamer1.0-plugins-bad gstreamer1.0-tools \
  libopencv-core406t64 libopencv-imgproc406t64 libopencv-imgcodecs406t64 \
  libmosquitto1 libssl3t64 libsqlite3-0 openssl ca-certificates
```

板厂 SDK（MPP / RGA / RKNN 运行时）未包含在 Ubuntu 软件源中，需使用板厂提供的 `.deb` 安装：

```bash
sudo dpkg -i rockchip-mpp_*.deb librga_*.deb rknn-runtime_*.deb
sudo apt -f install        # 仅在报依赖缺失时
sudo ldconfig
ls /dev/mpp_service /dev/rga    # 硬件服务节点存在即正常（详见下一步）
```

设备节点必须存在（缺则说明内核/驱动不对）：

```bash
ls -l /dev/mpp_service /dev/rga /dev/dri/renderD*
ls -d /dev/dma_heap/*
```

---

## 3. 下载、校验、安装

```bash
# ① 校验（Releases 页同时提供 .deb.sha256）
sha256sum -c aethel-frame_<版本>_arm64.deb.sha256        # 期望输出 ": OK"

# ② 安装 deb 包（自动处理依赖；这一步需要一次 sudo）
sudo apt install ./aethel-frame_<版本>_arm64.deb

# ③ 部署到您的用户目录（不要使用 sudo；默认安装至 ~/aethel-frame）
aethel-frame-setup

# ④ 启动（用户级服务，日常操作无需 sudo）
systemctl --user start aethel-frame
systemctl --user status --no-pager aethel-frame

# ⑤ 开机自启（无人值守重启后自动恢复运行）需执行一次：
sudo loginctl enable-linger "$(id -un)"
```

**为什么部署到用户目录**：配置、账号库、证书、授权文件、录像、日志都是**运行时需要写入**的文件，
放入系统目录会导致日常操作需要 root，并使浏览器上传授权因权限不足而失败。因此程序与数据集中在
安装根（`~/aethel-frame`），卸载即删除该目录，系统目录中不残留任何配置或数据；
`.deb` 仅将只读材料安装到 `/opt/aethel-frame`。

**安装各阶段的行为**（排障时对照）：

1. `apt` 阶段（root）：仅安装只读材料并做依赖体检（设备节点 / 板厂库 / 媒体插件），
   缺失项打印 `missing: ...`（**不会自动安装任何软件**）；**不创建用户、不写入 `/etc`、不触碰 `/data`**。
2. `aethel-frame-setup` 阶段（您的登录用户）：建 `~/aethel-frame/{app,etc,data,logs,var}`；
   生成主配置（预置 1 条占位通道，墙位上限 16；**可同时运行的路数由授权决定**，见 §6）与账号库（若已存在则打印 `existing config kept`，保持原样不改写）；
   生成自签证书（含本机 IP，有效期 10 年）；
3. 采集**设备码**并打印 `AEGIS-XXXX-XXXX-XXXX`（购买授权时需提供该短码）；
4. 注册**用户级 systemd 服务**（日常用 `systemctl --user` 起停，见 §5），并写出 `uninstall.sh`；
5. 无法自动完成的事项会明确输出：若当前用户缺少硬件设备节点读写权限，会打印一次性提示
   `sudo usermod -aG video,render <用户名>`（执行后需**重新登录**）。

安装时若打印 `missing: ...`（媒体插件或板厂库缺失），请先完成第 2 节后用同一安装包重装。

---

## 4. 首次登录

1. 浏览器打开 `https://<设备IP>:8443`（自签证书告警 → "继续访问"）。
2. 初始账号 **`admin` / `Aegis@12345`**，首次登录会**强制修改密码**。
3. 建议在「系统」页确认：运行态势显示各路帧率；授权页状态为 `free`（未安装 license 时的免费 2 路）。

| 端口 | 用途 | 建议 |
|---|---|---|
| 8443/tcp | 控制台（HTTPS / WebSocket） | **仅内网访问** |
| 8889/tcp | WebRTC 信令（浏览器预览的会话协商） | 对使用浏览器的网段放行 |
| 8189/udp | WebRTC 媒体（浏览器预览画面） | 同上；跨网段/过防火墙时需可达 |
| 8554/tcp | RTSP 输出（媒体服务） | 按需 |
| 9997/tcp | 媒体服务控制 API | 仅 127.0.0.1，不要对外 |
| 9000/tcp | 对象存储（可选证据上传） | 按需 |

---

## 5. 日常操作与日志位置

```bash
systemctl --user start|stop|restart aethel-frame    # **不要加 sudo**
systemctl --user status --no-pager aethel-frame
journalctl --user -u aethel-frame -n 100 --no-pager # 启动早期错误（配置读取/权限）优先查看
```

下表中 `<根>` = 安装根，默认 `~/aethel-frame`（用 `--prefix` 装到别处则换成那个目录）。

| 路径 | 内容 |
|---|---|
| `<根>/logs/aethel-frame.log` | 运行日志（10 MB × 5 份滚动） |
| `<根>/etc/` | 配置、账号库、证书、`license.dat`（**升级不覆盖**，仅您的账号可读） |
| `<根>/var/` | 每路推理派生配置 `pgie/`、客户自带模型导入目录 `models/` |
| `<根>/data/` | 录像 `recordings/`、抓拍 `snapshots/`、事件库 `aegis.db` |
| `<根>/app/` | 您的程序副本（二进制、前端、内置模型） |
| `/opt/aethel-frame/` | 由安装包写入的**只读材料**（升级时整体替换） |

停止服务请使用 `systemctl --user stop`（优雅退出信号），不要强制结束进程；
正常停机时，日志会记录停止过程各阶段的耗时与完成标记，据此可确认服务已完全退出。

---

## 6. 扩容到 4 / 8 / 12 路

**交付包内只有一份配置模板**（`share/aethel-frame.json.template` → 安装成 `<根>/etc/aethel-frame.json`，
`wall_positions:16` + 1 条占位通道）。**扩容不需要命令行操作，也不更换配置文件**：

1. 在控制台「系统 → 授权」上传厂商提供的 `license.dat` → **即时生效，无需重启**；
2. 在「配置 → 新增视频源」逐路添加摄像机（最多 `wall_positions` = 16 条）；
3. 启动各路；可同时运行的路数由授权决定（免费档 2 路）。

- 路数上限由授权决定：**超出上限的通道启动时会被自动截断**，并在日志中记录截断告警；
  导入更高档授权后，被截断的通道**直接点击「启动」即可恢复**（不修改配置文件、不重启服务）。
- 超出上限的启动或启用请求会返回 **`402 license_channel_limit`**（`{used, max}`）。
- 恢复出厂配置：备份后删除 `<根>/etc/aethel-frame.json`，重新运行 `aethel-frame-setup` 展开模板。

---

## 7. 升级 / 卸载 / 回滚

```bash
# 升级：新包仅替换只读材料；再次运行 setup 刷新程序副本，您的配置与数据原样保留
sudo apt install ./aethel-frame_<新版本>_arm64.deb
aethel-frame-setup && systemctl --user restart aethel-frame

# 卸载：先删除安装根（含配置与录像，请提前备份），再移除只读材料
bash ~/aethel-frame/uninstall.sh
sudo apt remove aethel-frame

# 回滚：安装上一版本包即可（事件库结构向前兼容旧版）
sudo apt install ./aethel-frame_<旧版本>_arm64.deb && aethel-frame-setup
```

**升级后三条验收**：

1. 配置未被改动：`ls -l ~/aethel-frame/etc/aethel-frame.json`，安装输出中包含 `existing config kept`；
2. 服务正常启动：`systemctl --user status aethel-frame` + `curl -k https://127.0.0.1:8443/health` 返回 200；
3. 授权与路数上限不变：「系统 → 授权」中的路数上限与升级前一致。

> 程序与数据集中在安装根内，**卸载仅需删除该目录**：系统目录（`/etc`、`/var`、`/data`）
> 中不会残留任何配置或录像。

---

## 8. 常见问题（按现象查）

| 现象 | 先查 | 处理 |
|---|---|---|
| 安装后无法启动，安装输出中包含 `missing: ...`（媒体插件或板厂库缺失） | `ls /usr/lib/aarch64-linux-gnu/librockchip_mpp.so*` | 安装媒体插件与板厂库（或使用自带插件的交付包）后重新安装 |
| 服务无法启动、`journalctl` 报告读取配置失败 | `~/aethel-frame/etc/aethel-frame.json` 是否被手工误改 | `~/aethel-frame/app/bin/aethel-frame --check-config --config <该文件>` 查看具体字段；或备份后删除该文件，再次运行 `aethel-frame-setup` 恢复模板 |
| 网页上传授权时提示写入失败 | `license.dat` 路径不在您的安装根内（旧版遗留配置） | 重新运行 `aethel-frame-setup` 完成迁移；或将配置项 `license.file` 修改为 `~/aethel-frame/etc/license.dat` |
| 浏览器预览无画面（设备端录像正常） | 该路是否在运行、预览用的子码流通道是否正常；浏览器与设备不同网段时 8889/tcp、8189/udp 是否已放行（见 §4） | 到「通道」页确认状态为运行中，必要时点「重启」该路；跨网段部署先核对端口放行 |
| ONVIF 搜索不到摄像机 | 摄像机与设备是否在同一网段、是否已开启 ONVIF | 在「通道 → ONVIF 搜索」中填写设备账号密码；若仍不成功，手工填写 RTSP 地址 |
| 硬件解码未生效（自动回退到软件解码，CPU 占用高、多路卡顿） | 板厂 MPP 库与内核驱动是否匹配 | 确认 `mpp_service` 设备节点与板厂库存在，未混装不同来源的 SDK |
| 启用通道被拒绝（`402`） | 「系统 → 授权」的路数上限 | 免费档仅 2 路；按 [授权说明](licensing.md) 购买扩容 |
| 录像文件 0 字节 / 分段缺失 | 是否处于“配置重建”期间 | 重建期间该路录像切分会暂停，属正常现象；持续出现时，请在日志中检索 `record` 相关记录定位原因 |
| 页面打开很慢 / 翻页卡顿 | 浏览器并发连接限制、网络带宽 | 建议使用 Chrome / Edge 最新版本；预览使用子码流，带宽不足时请降低预览帧率或码率 |

---

## 9. 联系方式

- **邮箱**：`3300934581@qq.com`（授权购买、容量评估、现场部署支持；索取授权请附上 `machine_info.json`）
- **微信**：扫描下方二维码添加；工作时间（工作日 9:00–18:00）通常 2 小时内响应，邮件 24 小时内响应

<img src="assets/wechat-qr.png" alt="微信二维码" width="180">

授权矩阵与机器信息获取方式见 [授权说明](licensing.md)。
