# Changelog

本文件格式遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)；
版本号遵循 [语义化版本](https://semver.org/lang/zh-CN/)。
发布资产（deb 与文档版本快照）见 GitHub Release；对外文档母本在 `docs/public/`，发布流程见 `scripts/release.sh`。

## [Unreleased]

### Breaking

- **产品改名：`aegis-edge` → `aethel-frame`**（用户可见名彻底改）：二进制/包名/systemd
  服务与 unit 名/配置名（`deploy/config/aethel-frame*.json`）/命令与文档里的名字全部跟着变；
  运行目录换成 `/etc|/var/lib|/var/log|/tmp|/opt/aethel-frame`，License 默认路径
  `/etc/aethel-frame/license.dat`，日志文件 `aethel-frame.log`。源码内部标识（`aegis::` 命名空间、
  `AEGIS_*` 环境变量与日志宏、`libaegis_core.a`、`include/aegis/`）、数据根 `/data/aegis` 与
  `aegis.db`、系统账号 `aegis`、厂家工具 `aegis-license-gen`、GitHub 仓名**不变**。
  **旧名不留兼容层**（无 symlink / 无迁移脚本）——已签发的 `license.dat` 仍有效（签名载荷不含产品名）。
- **“一个检测目标都没勾”不再等于“全部类别”**：该路 **不检测**（无框、无告警，
  拉流/录像/预览照常）。旧版把留空当“全部类别”，所以“没勾人也在检人”。
  ✅ 要检全部类别，请在那个业务里**显式选「全部类别」**（`class_id=-1`）。
  这种路不静默：启动/`--check-config` 打 WARN `event=infer_bypassed_no_targets`，
  配置页「AI」列悬停会说“关（未选检测目标）”。

### Changed

- **配置页「操作」列：5 颗按钮均分宽度且恒为一行**：列宽 268 → **340 px**，按钮容器由
  `flex flex-wrap` 改为 **`grid grid-cols-5`**（“均分且单行”不再依赖内容宽度），按钮加
  `w-full min-w-0 overflow-hidden text-ellipsis`；整表最小宽 = **七列之和（1258 px）** ⇒ 窗口不够宽时
  出现**横向滚动条**，而不是浏览器按比例缩列把最后一颗按钮顶到第二行；状态角标移出按钮网格。
  中文/英文（`Restart` 最长）两种模式均不拆字。
- **配置页操作列不再竖排拆字**（上一轮已收敛）：该列原先钉 268 px + “窄屏折行”，**仍会被挤成 2 行**；
  现在改成**五颗等宽按钮强制同一行**（见上一条）。
- **系统状态搬回顶栏**：CPU / 温度 / 内存 / NPU / 磁盘做成顶栏胶囊（高 28 px、
  数值 15 px）；响应式梯度：CPU/温度常驻 → 内存 `md` → NPU `lg` → 磁盘 `xl`；
  内存读数改为“已用 MB”；底栏只留 WS 连接 + 版本/媒体引擎/提交。
- **左侧导航栏 224 → 192 px**：只减横向冗余，图标/字号/行高不动。
- **左上角字标改汉字「曜帧」**：主行「曜帧」+ 副行「边缘视觉网关」+ 下沿强调线；
  **英文模式同位置仍显示 `AethelFrame`**（不是中英并列）；登录页同口径。
- **发布面定型：建立独立的对外发布仓**：新增 `docs/public/` 对外母本（首页 README +
  安装指南 / 用户手册 / 授权说明 / 合规声明），`scripts/publish_public_repo.sh` 把它们导出到**同级的
  另一个目录**作为公开仓（文档以普通 `.md` 提交 ⇒ 网页直接可读），并用四道断言保证内部材料、
  二进制、私钥都进不去；公开仓自带 `.gitignore`（`/dist/`、`*.deb`、`SHA256SUMS`、`*.pem`）。
- **不再打“文档 zip”**：`scripts/release.sh` 去掉 `aethel-frame-docs-v<ver>.zip`，安装/使用文档快照
  改为从 `docs/public/` 母本取（旧版直取内部文档 ⇒ 会把内部编号带到公开资产），并在导出后
  逐档断言无内部引用。
- **出证工具可按平台编译**：新增 `scripts/build_license_tool.sh`（自动检测 x86_64 / aarch64，
  构建目录按架构隔离，产完用 `file` 断言架构与 `--help` 冒烟）；在 aarch64 上默认拒绝，避免
  私钥落在开发设备上。

### Fixed

- **未勾选的类别不再被画框**（根因在媒体基座 gstmpp，不在网关）：`detection-filter` 里的
  **类名** token 以前会被顺手当成“0 号类”（`atoi("car") == 0`，而 COCO 0 号类 = `person`）
  ⇒ “只勾了车”的一路仍然检人并画框（告警侧逐类判 ⇒ 不告警，所以只有画面错）。现在非数字 token **只**按类名匹配。
  ⚠️ **行为变化**：若某路写了**模型词表里不存在**的类名，该类现在不进白名单；若白名单因此为空 ⇒
  该路**不输出任何目标**并打 WARN `y8_filter WARN whitelist_empty … action=no_detections`（以前是静默检 person）。
  **生效需重装基座**：`cd gstmpp && ./build.sh all && sudo ./build.sh install` 后重启服务。
- 基座：BYOM 模型的 `*.labels.txt` 条数少于模型类数时不再段错误（词表空洞跳过 `NULL` 槽位）。
- 前端 i18n 门禁（`frontend/tools/i18n_check.mjs`）不再把**模板注释**当待翻文案（旧的会假红/假绿）；
  script 与模板的注释遮罩分开处理（模板里 `//` 是真实文案，如 `mqtt://host:port`）。

---

## [0.1.0] - 2026-10-27

首个可交付版本：RK3588 多路 RTSP AI 视觉网关（`aethel-frame_0.1.0_arm64.deb`）。

### Added

- **监视墙**：WHEP/WebRTC 多路预览（子码流分析 + 主码流细节）；五种布局档位（自适应 / 4×2 / 3 列 / 2×4 / **单列**）；
  双击放大秒开（复用现会话 `MediaStream`，主码流首帧到达后淡入）、放大态 ←/→ 换路、Esc 关闭；可视区错峰起播。
- **AI 分析**：区域入侵 / 越线 / 人群聚集 / 安全帽等业务；场景化 ROI 绘制；类名→`class_id` 按模型词表自动重绑（BYOM 友好）。
- **模型中心**：内置模型（`builtin/`，随包分发）与客户自带模型（BYOM：`.rknn` + `labels.txt` 导入，存 `/var/lib/aethel-frame/models`，升级不覆盖）。
- **告警与取证**：事件入库 + WS 实时推送、确认/批量确认/CSV 导出、抓拍缩略图抽屉。
- **录像与回放**：连续录像 + 事件前后片段、Range 回放（带鉴权）、抓拍管理（大图/下载/删除）。
- **授权（License）**：Ed25519 签名 `license.dat` + 编译期内置公钥验签；免费 2 路永久；4/8/12 路档位模板
  （`aegis-activate-profile N`，只合并不覆盖）；到期 30/15/7 天提醒 + 15 天宽限。
- **发布工程**：`deploy/models/builtin` 打包白名单（AGPL 权重隔离于 `deploy/models/agpl/`，永不入包）；
  `build-info.json`（版本/commit/公钥指纹/模型 sha256）；`package_audit.sh` 包内容审计（黑名单 + 自检正反例）；
  `scripts/release.sh` 一键发布（校验→打包→审计→SHA256SUMS→`gh release create`→发布后复核）；
  tag 推送 CI（版本三处一致 + 许可自检 + 文档链接检查）。
- **文档**：安装指南与使用手册、本 CHANGELOG；根 `LICENSE`（Apache-2.0）与 `NOTICE`（第三方清单）。

### Changed

- **前端卡顿治理**：i18n DOM 观察器英文模式才安装且批量翻译（`Map` 缓存 + `WeakSet` 去重）；`metrics` 字段级 diff；
  登录瞬间 `GET /channels` 请求合并去重；`getStats` 间隔 2 s → 5 s 且后台页暂停。
- **顶栏模式徽标响应式**：`stats` 1 Hz 携带 `pipeline_mode`，页面内切换容量/性能模式后 2 s 内徽标更新（无需刷新）。
- **视觉改版（中度）**：设计令牌三副本同步（`theme.css` / `tailwind.config.js` / 前端设计稿）；AD 令牌
  `--panel/-2/--line/--accent` + 语义色 `ok/warn/crit`；AppShell 顶栏 56 px + 分段导航 + 状态带；
  `VideoTile` 圆角描边 + 状态角标 + hover 操作条；通用组件类 `.btn/.card/.pill/.field`。
- **打包**：`build_deb.sh --with-models` 白名单化（默认仅 `deploy/models/builtin/`）；`Depends` 只列板厂镜像确定存在的包，
  MPP/RGA/RKNPU2/gstmpp 改由 `postinst` 体检提示；随包生成 `/usr/share/doc/aethel-frame/copyright`（LICENSE + NOTICE 拼装）。
- **配置口径**：模型导入目录固定为可写的 `media.model_import_dir`（生产 `/var/lib/aethel-frame/models`），
  不再回落到只读 `/opt/aethel-frame/models`。

### Fixed

- **Ctrl-C / `systemctl stop` 退出慢**：`JobQueue::stop()` 丢弃排队任务（WARN `job_discarded`）不再逐路等重建；
  停机顺序前移队列站停；`set_pipeline_null_blocking()` 超时 3 s（不再每路等 30 s）；每阶段耗时日志
  `event=shutdown_stage` + `event=shutdown_complete`；systemd `TimeoutStopSec=25`。
- 首登监视墙三处并发 `GET /channels` 导致的线程抢占（请求去重 + `dataReady` 只拉 `info/stats`）。
- 监看页 12 路首帧慢：起播错峰（`index × 120 ms`）+ 离屏延迟 + 2/4/8/12ch 模板子码流 `gop` 30 → 15（关键帧 0.5 s）。
- 安全帽占位权重（136 B 坏文件）删除；客户 2 类模型类名解析用例改为现场夹具（新克隆仓库可跑通）。

### Security

- **许可与密钥隔离**：私钥与出证工具仅在厂家主机（`~/.aegis-keys/`，0600），**永不进仓库/不进包**；
  `.gitignore` 屏蔽 `*.pem`、`*.key`、`*.deb`、`dist/`；`package_audit.sh` 断言包内无
  `*.pem`/`id_ed25519*`/`license.dat`/`machine.json`/`aegis-license-gen`/开发配置。
- 发布资产逐个 sha256 复核（`SHA256SUMS`），`gh release view --json assets` 资产集比对。