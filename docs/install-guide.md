# 安装指南

面向现场实施人员：从"拿到一块已刷机的 RK3588 板子"到"浏览器能看画面"。

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
| 媒体基座插件 | gstmpp（`mppinfer` / `mpposd` / `mppanalytics` / `mpptracker`） | `gst-inspect-1.0 mppinfer` |
| mediamtx | **v1.13.1**（部分交付包已内置，见发布说明） | `mediamtx --version` |
| mosquitto | 2.x（仅 MQTT 出口需要） | `mosquitto -h \| head -1` |
| 磁盘 | 系统盘 ≥ 4 GB 可用；**录像盘单独分区**（默认 `/data/aegis`） | `df -h /data` |
| 内存 | 2–4 路 ≥ 4 GB；8–12 路 ≥ 8 GB | `free -m` |
| CPU | 最低 4 核（2–4 路子码流预览）；8 路建议 8 核 | `nproc` |

> ⚠️ **板厂 SDK 必须来自同一块板厂的同一份镜像**。混装不同来源的 MPP / RGA / RKNN 库会出现
> "元素能 `inspect`、一跑就崩"的现象。升级这些库之前先备份 `/usr/lib/aarch64-linux-gnu/librockchip_mpp.so*`。

> 摄像机侧要求：浏览器要预览画面，摄像机主/子码流必须是 **H.264 Baseline（无 B 帧）**。
> 常见坑：把码流 profile 设成 Main/High 后浏览器黑屏（设备本身录像正常）。

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

板厂 SDK（MPP / RGA / RKNN 运行时）不在 Ubuntu 源里，用板厂提供的 `.deb` 安装：

```bash
sudo dpkg -i rockchip-mpp_*.deb librga_*.deb rknn-runtime_*.deb
sudo apt -f install        # 仅在报依赖缺失时
sudo ldconfig
gst-inspect-1.0 mppvideodec | head -3    # 能列出版本即正常
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

# ② 安装（方式 A 推荐，会自动处理依赖）
sudo apt install ./aethel-frame_<版本>_arm64.deb
#    离线环境用方式 B：
#    sudo dpkg -i aethel-frame_<版本>_arm64.deb && sudo apt -f install

# ③ 启动（安装脚本只"设为开机自启"，不会自动拉起）
sudo systemctl start aethel-frame
sudo systemctl status --no-pager aethel-frame
```

**安装过程中做了什么**（排障时对照）：

1. 建系统用户 `aegis`（`nologin`），加入 `video` / `render` 组，用于访问硬件编码/推理设备节点。
2. 建目录：`/var/lib/aethel-frame`、`/var/log/aethel-frame`、`/data/aegis/{recordings,snapshots}`。
3. **只在文件不存在时**创建配置 `/etc/aethel-frame/aethel-frame.json`（默认 2 路）与账号库；
   已存在则打印 `existing config kept`，**一个字节都不改**。
4. 生成自签证书 `/etc/aethel-frame/certs/server.{crt,key}`（含本机 IP，有效期 10 年）。
5. 采集**设备码**并打印 `AEGIS-XXXX-XXXX-XXXX`（买授权时把这串发给厂商）。
6. 依赖体检：设备节点 / 板厂库 / 四个媒体元素；缺什么打印 `missing: ...`（不会自动装东西）。

打印过 `missing: gst:mppinfer ...` 说明媒体基座未装好 ⇒ 先解决第 2 节，再 `sudo apt install ./同一个.deb` 重装。

---

## 4. 首次登录

1. 浏览器打开 `https://<设备IP>:8443`（自签证书告警 → "继续访问"）。
2. 初始账号 **`admin` / `Aegis@12345`**，首次登录会**强制修改密码**。
3. 建议立刻在「系统」页确认：运行态势有帧率、授权页显示 `free`（未装 license 时的免费 2 路）。

| 端口 | 用途 | 建议 |
|---|---|---|
| 8443/tcp | 控制台（HTTPS / WebSocket） | **只放内网** |
| 8554/tcp | RTSP 输出（媒体服务） | 按需 |
| 9997/tcp | 媒体服务控制 API | 仅 127.0.0.1，不要对外 |
| 9000/tcp | 对象存储（可选证据上传） | 按需 |

---

## 5. 日常操作与日志位置

```bash
sudo systemctl start|stop|restart aethel-frame
sudo systemctl status --no-pager aethel-frame
journalctl -u aethel-frame -n 100 --no-pager          # 启动早期错误（读配置/权限）先看这里
```

| 路径 | 内容 |
|---|---|
| `/var/log/aethel-frame/aethel-frame.log` | 运行日志（10 MB × 5 份滚动） |
| `/etc/aethel-frame/` | 配置、账号库、证书、`license.dat`（**升级永不覆盖**） |
| `/var/lib/aethel-frame/` | 每路推理派生配置 `pgie/`、客户自带模型导入目录 `models/` |
| `/data/aegis/` | 录像 `recordings/`、抓拍 `snapshots/`、事件库 `aegis.db` |
| `/opt/aethel-frame/` | 程序本体（二进制、前端、内置模型、配置模板，只读） |

停止服务用 `systemctl stop`（正常退出信号），**不要**强杀进程；每阶段停机耗时都会写进日志
（`event=shutdown_stage` → `event=shutdown_complete total_ms=...`）。

---

## 6. 扩容到 4 / 8 / 12 路

包内自带四份产品模板：`/opt/aethel-frame/profiles/aethel-frame-{2,4,8,12}ch.json`。

```bash
sudo aegis-activate-profile 8            # 推荐：合并扩容，已有通道（地址/区域/业务）原样保留
sudo aegis-activate-profile 8 --dry-run  # 先看会做什么
sudo aegis-activate-profile 2 --replace  # 重置为出厂 2 路模板（自动备份 .bak.<时间戳>）
```

- **合并**是默认行为：把画面布局位提到 8，并追加若干 `enabled=false` 的空槽，你在 Web 上填地址再启用。
- 路数上限由授权决定：**未续费时启动会按授权截断**，并打印 `WARN event=license_limit action=truncate`；
  续费并重启后，原来被截断的通道自动恢复（配置文件不改）。
- 超出上限的启动/启用请求会返回 **`402 license_channel_limit`**（`{used, max}`）。

---

## 7. 升级 / 卸载 / 回滚

```bash
# 升级：包里不含活动配置 ⇒ 客户配置与 /data 原样保留
sudo apt install ./aethel-frame_<新版本>_arm64.deb
sudo systemctl restart aethel-frame

# 卸载：停服务、删程序；**不删**配置/录像/事件库
sudo apt remove aethel-frame

# 回滚：装回上一版即可（事件库带结构迁移，向前兼容旧版）
sudo apt install ./aethel-frame_<旧版本>_arm64.deb
```

**升级后三条验收**：

1. 配置没被动：`ls -l /etc/aethel-frame/aethel-frame.json`，安装输出里有 `existing config kept`；
2. 服务能起：`systemctl status aethel-frame` + `curl -k https://127.0.0.1:8443/health` 返回 200；
3. 授权与路数没变：「系统 → 授权」里的路数上限与升级前一致。

> `/data/aegis`、`/var/lib/aethel-frame`、`/etc/aethel-frame` 永远不会被包管理删除。
> 需要彻底清理请手工删除（并先备份）。

---

## 8. 试用 / 无 root：便携安装

便携安装脚本（`install_portable.sh`）**不在 `.deb` 里**，由厂商随交付材料提供（联系邮箱见首页）：

```bash
bash install_portable.sh --prefix "$HOME/aethel-frame" --deb aethel-frame_<版本>_arm64.deb
$HOME/aethel-frame/start.sh      # 启动（日志 logs/aethel-frame.log）
$HOME/aethel-frame/status.sh
$HOME/aethel-frame/stop.sh       # 正常退出
```

目录全部落在 `--prefix` 下（`app/ etc/ data/ logs/ var/`），不改系统目录、不需要 root
（但运行用户要能访问硬件设备节点，即在 `video`/`render` 组里）。开机自启请自行写 systemd user unit 或 `@reboot`。

---

## 9. 常见问题（按现象查）

| 现象 | 先查 | 处理 |
|---|---|---|
| 装完起不来，日志里有 `missing: gst:mppinfer` | `gst-inspect-1.0 mppinfer` | 装媒体基座插件（或用自带插件的交付包）后重装 |
| 服务起不来、`journalctl` 说读配置失败 | `/etc/aethel-frame/aethel-frame.json` 是否被手工改坏 | `aethel-frame --check-config --config <该文件>` 看具体字段；或 `aegis-activate-profile 2 --replace` 恢复模板 |
| 画面黑屏但录像正常 | 浏览器预览只接受 H.264 **Baseline**（无 B 帧） | 把摄像机主/子码流 profile 改成 Baseline |
| 添加 ONVIF 摄像机搜不到 | 摄像机与板子是否同网段、是否开了 ONVIF | 在「通道 → ONVIF 搜索」里填设备账号密码；仍不行就手工填 RTSP 地址 |
| 提示 `Input frame is not a DMA-BUF` | 是否走了软解码 | 确认 `mppvideodec` 可用、未被降级为 `avdec_h264` |
| 只想起 4 路但被拒（`402`） | 「系统 → 授权」的路数上限 | 免费档只有 2 路；按 [授权说明](licensing.md) 购买扩容 |
| 录像文件 0 字节 / 分段缺失 | 是否在"配置重建"期间 | 重建期间会暂停录像切分，属正常；持续出现请看日志里的 `recorder_*` 行 |
| 页面打开很慢 / 翻页断流 | 浏览器是否只开了一个连接、网络带宽 | 建议 Chrome/Edge 最新版；预览走的是子码流，带宽不足请降帧率/码率 |
