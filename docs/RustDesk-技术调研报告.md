# RustDesk 技术调研报告

**项目：** 工业设备远程运维平台
**版本：** V1.0
**日期：** 2026-11-16
**性质：** 技术验证（对应《系统设计文档》第 10 章「第一步：RustDesk 技术验证」）

> 本报告只记录**已验证的事实**。每条结论都标注证据来源：官方文档 URL、上游源码路径（本地克隆于 `poc/upstream/`）或实测截图（`poc/evidence/`）。未经实证的推断一律明确标注为「未验证」或列入《PoC 验收报告》的现场验证清单。

---

## 0. 结论速览

| # | 调研问题 | 结论 | 证据 |
|---|---|---|---|
| 1 | Windows/Android 客户端能否接入自建服务端？ | **能**。客户端内置 ID/中继/API 服务器与 Key 四项配置，支持 UI 手工、CLI、配置文件、二维码等多种途径 | 源码 `flutter/lib/common.dart`、官方文档 client-configuration |
| 2 | OSS 服务端是否含 Web Console？ | **不含**。Web Console 是 Server Pro 专有（21114 端口）；OSS hbbs/hbbr 只有本地运维通道（127.0.0.1） | 官方 OSS Docker 文档；源码 `rustdesk-server/src/common.rs:39-64` |
| 3 | Web Client 与 Web Console 是否同一物？ | **不是**。Web Console = Pro 管理后台；Web Client = 浏览器控制端（Pro 内置 + 官方公共版） | pricing 页、Pro FAQ、实测截图 |
| 4 | 浏览器能否查看 Windows / Android 设备画面？ | **可行**。浏览器端用 Web Client（WebSocket 传输），受控端用原生客户端；受控端 OS 与浏览器端解耦 | OSS 文档（21118/21119 支持 Web 客户端）、Pro FAQ WSS 配置、实测 Web Client V2 UI |
| 5 | 自托管 Web Client 的授权要求？ | 仅随 **Server Pro** 分发，且许可证需满足 `users×10 + devices ≥ 400`；不满足时可用**官方公共 Web Client**（rustdesk.com/web）指向自建服务器 | pricing 页、Pro FAQ |
| 6 | 「免逐次现场确认」能否满足？ | **Windows：可行**（永久密码 + 服务化，完全无人值守）。**Android：部分受限**（Android 系统要求屏幕采集授权弹窗，RustDesk 无法绕过；需保持 RustDesk 服务常驻，至少在应用重启/设备重启后现场重新授权一次） | 官方 Android 文档；源码 `PermissionRequestTransparentActivity.kt` |
| 7 | 业务平台是否需要重写远控通信？ | **不需要**。屏幕采集、编码、传输、NAT 穿透、中继全部复用 RustDesk；平台只做设备/用户/会话管理与「跳转到 Web Client」 | 本报告第 6 章集成方案 |
| 8 | 未经官方验证的 API 是否存在被假设的风险？ | 已排除。OSS hbbs/hbbr **没有**对外的设备状态 REST API；Pro 的 API 以 Token + 官方 CLI 脚本形式提供，无公开正式 API 契约文档，集成前须实测 | 源码检索、Pro console 文档、GitHub 讨论区 |

---

## 1. 调研对象与版本锁定

| 仓库 | 用途 | 版本/状态 | 许可证 |
|---|---|---|---|
| [rustdesk/rustdesk](https://github.com/rustdesk/rustdesk) | 客户端（Windows/Android/macOS/Linux），Flutter + Rust | stable **1.5.3**（2026-11-09 发布）；master HEAD `3e953e886a87`（2026-11-16） | AGPL-3.0 |
| [rustdesk/rustdesk-server](https://github.com/rustdesk/rustdesk-server) | 开源服务端：**hbbs**（ID/会合服务）+ **hbbr**（中继服务） | stable **1.1.16**（GitHub latest release，2026-11 查证；release 资产为 Linux deb/zip 与 Windows zip，**不提供 macOS 二进制**（本次实测），常规部署走容器镜像 `rustdesk/rustdesk-server`） | AGPL-3.0（`poc/upstream/rustdesk-server/LICENSE`） |
| [rustdesk/rustdesk-server-pro](https://github.com/rustdesk/rustdesk-server-pro) | Server Pro 的**脚本仓库**（Python CLI：users.py、devices.py、audits.py 等）；Pro 本体闭源，以 Docker 镜像 / install.sh 分发 | 活跃维护（2026-10 有 push） | 无开源许可证字段（闭源商业软件） |
| RustDesk Web Client V2 | 官方公共网页版控制端，https://rustdesk.com/web | **Preview，版本 1.5.3**，构建日期 2026-11-09（页面「关于」实测，见 `poc/evidence/webclient-05-about-v153.png`） | 官方托管（源码未公开发布；rustdesk 组织下无 web-client 相关仓库，已逐一探测 404） |

本地克隆（供源码取证）：

- `poc/upstream/rustdesk`（master 浅克隆）
- `poc/upstream/rustdesk-server`（master 浅克隆）

> 版本策略建议：PoC 与第一版锁定客户端 1.5.x、服务端 1.1.16；升级前回归测试。

---

## 2. 组件全景与数据通路

```mermaid
flowchart TB
    subgraph 浏览器端（控制方）
        WC[Web Client<br/>公共版 rustdesk.com/web 或 Pro 自托管 /web]
    end
    subgraph 自建服务端
        NGX[Nginx 443<br/>WSS 反向代理]
        HBBS[hbbs<br/>21115/21116TCP+UDP/21118WS]
        HBBR[hbbr<br/>21117/21119WS]
    end
    subgraph 设备端（受控方）
        WIN[Windows RustDesk 客户端<br/>服务化 + 永久密码]
        AND[Android RustDesk 客户端<br/>屏幕采集授权 + 永久密码]
    end
    PLAT[业务管理平台<br/>Next.js + Spring Boot<br/>设备/用户/会话]

    WC -- "wss://…/ws/id" --> NGX --> HBBS
    WC -- "wss://…/ws/relay" --> NGX --> HBBR
    WIN -- "TCP/UDP 21116（注册/心跳）" --> HBBS
    AND -- "TCP/UDP 21116（注册/心跳）" --> HBBS
    HBBS <--> HBBR
    HBBR -- 中继流 --> WIN
    HBBR -- 中继流 --> AND
    PLAT -. "仅管理数据，不经手视频流" .-> WC
    PLAT -. "Agent 心跳（自建，V1）" .-> PLAT
```

关键点（均已验证）：

1. **视频流永远不经过业务平台。** 平台只做设备台账、用户权限、会话记录，并负责把运维人员引导到 Web Client。
2. Web Client 只能走 **WebSocket**（官方 FAQ 原文：WebSocket 客户端不使用 TCP/UDP，除直连 IP 外只能经中继连接）。
3. 原生客户端与 hbbs 走 21115/21116，与 hbbr 走 21117；与 Web Client 互不影响。

---

## 3. 客户端接入自建服务端（问题 1 详述）

### 3.1 客户端配置项（源码证据）

| 配置项 | options 键 | 作用 | 源码位置 |
|---|---|---|---|
| ID 服务器 | `custom-rendezvous-server` | hbbs 地址（可含端口） | `poc/upstream/rustdesk/flutter/lib/common.dart:3010`、`flutter/lib/common.dart:3672`、`src/flutter_ffi.rs:1072` |
| 中继服务器 | `relay-server` | hbbr 地址（留空自动推导） | 同上 |
| API 服务器 | `api-server` | Pro API/地址簿；OSS 留空 | 同上 |
| Key | `key` | hbbs 公钥（`id_ed25519.pub` 内容），启用端到端加密 | 官方文档 client-configuration |
| 使用 WebSocket | `allow-websocket` | 客户端改走 WebSocket（配合 Web Client 场景） | `poc/upstream/rustdesk/flutter/lib/consts.dart:198`；Pro 策略下发见 `src/hbbs_http/sync.rs:540` |

### 3.2 配置下发途径（按部署规模选择）

| 途径 | Windows | Android | 证据 |
|---|---|---|---|
| UI 手工配置 | 主界面 ⋮ → 网络 → ID/中继服务器 | 设置 → ID/中继服务器 | 官方文档 client-configuration |
| CLI 设置选项 | `rustdesk --option custom-rendezvous-server <host>`（需已安装+管理员权限） | 不适用 | 源码 `src/core_main.rs:502-514`（`--option` 需 `is_installed() && is_root()`） |
| CLI 导入加密配置串 | `rustdesk --config <串>`（需已安装+管理员权限） | 不适用 | 源码 `src/core_main.rs:489` |
| 导入/导出配置串 | 设置 → 网络 → Export/Import Server Config | 同（客户端内置） | 官方文档 client-configuration |
| 扫码/URL 配置 | 不适用 | 扫码，内容形如 `config={"host":"xxx","key":"xxx"}` | 官方 Android 文档 |
| 配置文件 | `RustDesk.toml`（设备标识）/ `RustDesk2.toml`（选项），随安装与用户环境存放 | Android 端不直接编辑文件 | 源码 `poc/upstream/rustdesk/libs/hbb_common/src/config.rs:878-882`（`APP_NAME + suffix` → `with_extension("toml")`） |
| 自定义客户端生成器 | **Pro（Basic 档及以上）**：预置服务器/品牌，生成安装包（Windows/Mac/Linux/Android Arm64） | 支持 Android Arm64 | 官方文档 client-configuration + pricing |
| `--assign --token` 设备登记 | 向 Pro 注册/分配设备 | 同 | 源码 `src/core_main.rs`（`--assign` 分支，token 来自 Pro） |

### 3.3 无人值守接入（免逐次确认）

**Windows（结论：可行）**

- 安装版客户端注册系统服务，开机自启。
- 设置永久密码后，将「验证方式」固定为使用永久密码（选项键 `verification-method` = `use-permanent-password`），连接即不再需要每次确认。
  源码：`poc/upstream/rustdesk/src/ui/index.tis:990-1105`（`verification-method` / `use-permanent-password`）。
- CLI 可直接设置密码：`rustdesk --password <pwd>`（源码 `src/core_main.rs:425`）。

**Android（结论：有系统级限制，需现场验证缓解措施）**

- Android 屏幕采集必须经过系统 `MediaProjection` 授权弹窗：源码 `poc/upstream/rustdesk/flutter/android/app/src/main/kotlin/com/carriez/flutter_hbb/PermissionRequestTransparentActivity.kt:22-28` 调用 `mediaProjectionManager.createScreenCaptureIntent()`。
- 官方 Android 文档原文："**Every time you start RustDesk, you need to request 'Screen Capture' and 'Input Control' permissions again.**"（每次启动 RustDesk 都需重新请求屏幕采集与输入控制权限）。
- 即：**不存在完全免确认的 Android 无人值守**。可行的工作方式：
  1. 保持 RustDesk 应用常驻运行（授予「屏幕采集」后服务自动运行；禁用电池优化 + 厂商自启动白名单，工业平板通常支持 kiosk/MDM 固定）；
  2. 设置永久密码（设置 → 安全），连接时不再需要设备端点「接受」；
  3. 仅在应用重启/设备重启后，需要现场人员重新点一次系统授权弹窗。
- 上述第 1、3 点的实际表现（不同厂商 ROM 的保活与授权记忆）**必须在实机上验证**，已列入《PoC 验收报告》。

### 3.4 设备标识（RustDesk ID）

- 客户端首次连接 hbbs 后获得唯一 ID；CLI `rustdesk --get-id` 可直接读取（源码 `src/core_main.rs:464`）。
- 平台登记即保存「设备编号 ↔ RustDesk ID」映射（对应 PRD 设备表字段）。

---

## 4. 服务端：OSS 与 Pro 的能力边界（问题 2 详述）

### 4.1 OSS 服务端（rustdesk-server 1.1.16）源码验证

> 行号以本地克隆中 **tag 1.1.16**（`git checkout 1.1.16`，commit `73523b3`）为准；master 分支行号略有偏移。

| 组件 | 端口 | 用途 | 源码位置（1.1.16） |
|---|---|---|---|
| hbbs | 21116 **UDP** | ID 注册、心跳 | `poc/upstream/rustdesk-server/src/rendezvous_server.rs:112` |
| hbbs | 21116 TCP | TCP 打洞 / 连接 | `rendezvous_server.rs:150` |
| hbbs | 21115 TCP | NAT 类型测试（`nat_port = port - 1`） | `rendezvous_server.rs:104,151` |
| hbbs | 21118 TCP | **WebSocket（Web Client 入口）**，`ws_port = port + 2` | `rendezvous_server.rs:105,152` |
| hbbr | 21117 TCP | 中继 | `poc/upstream/rustdesk-server/src/relay_server.rs:82` |
| hbbr | 21119 TCP | **WebSocket 中继（Web Client）**，`port2 = port + 2` | `relay_server.rs:83-84` |
| hbbs/hbbr | —— | **1.1.16 发布版不含任何 console/Web Console 代码**（grep `console` 于 src/ 零命中）；master 分支新增的 `listen_console` 也仅绑 127.0.0.1 运维通道（master `src/common.rs:39-64`，注释原文 "The runtime console (check_cmd) is reached via 127.0.0.1"），两者都不是 Web Console | — |

- 自检工具 `rustdesk-utils doctor` 也按 21116/21117/21118/21119 检查连通性（`src/utils.rs:114-117`）。
- **OSS 服务端没有：** Web Console、用户体系、设备台账 API、审计日志、Web Client 自托管包。官方 OSS Docker 文档明确 21114 为 "Web 控制台，仅 Pro 版可用"。
- 密钥：首次运行在数据目录生成 `id_ed25519` / `id_ed25519.pub`；客户端 Key 填公钥内容即启用加密（官方 client-configuration 文档）。
- **运行时复现（2026-11-17，本机 macOS arm64）：** 从 tag 1.1.16 源码编译 hbbs/hbbr 并启动，实测 5 个端口全部按上表监听，密钥对自动生成，21118/21119 WebSocket 握手均返回 `101 Switching Protocols`——源码结论与运行时行为一致（详见《PoC 验收报告》V-4）。

### 4.2 安全注意（官方文档原文级警告）

WebSocket 端口 21118/21119 信任 `X-Real-IP` / `X-Forwarded-For` 头且**不校验**，直接暴露可被伪造 IP 绕过限速/封禁并污染日志。生产要求：仅经反向代理暴露，防火墙限制 21118/21119 只允许代理主机访问（OSS Docker 文档原文）。

### 4.3 Server Pro 增量能力（与本项目相关的部分）

| 能力 | 说明 | 证据 |
|---|---|---|
| Web Console | 端口 21114；管理用户/用户组、设备与设备分组、策略、控制角色、中继、SMTP、**API Token**、自定义客户端；默认账号 `admin/test1234` | Pro console 官方文档 |
| 审计日志 | 连接/文件传输/告警/控制台操作四类（连接类型 0=远程桌面、1=文件、2=端口、3=摄像头、4=终端），保留期可配 | 同上 |
| 策略（Strategy） | 可下发客户端设置（含 `allow-websocket` 等），按设备/用户/设备组分配 | console 文档 + 客户端 `src/hbbs_http/sync.rs` |
| API Token | Web Console → Settings → Tokens 创建，权限粒度：Device / Audit Log / User / Group / Strategy / Address Book / Admin Role / Control Role | console 文档原文 |
| 官方 CLI 脚本 | users.py / devices.py / device-groups.py / audits.py / ab.py / strategies.py（调用上述 API） | rustdesk-server-pro 仓库 |
| REST API | 社区与官方博客提及 `GET /api/devices` 等；**无公开正式 API 契约文档**，集成前必须对实际 Pro 版本实测 | GitHub 讨论 #14111、官方博客 rustdesk-for-enterprise |
| 自托管 Web Client | 随 Pro 分发，经 Nginx 暴露于 `https://<域名>/web`；**许可证门槛 `users×10 + devices ≥ 400`** | Pro FAQ + pricing 页 |
| 许可证模型 | 一张许可绑定一台 hbbs；hbbr 不单独计费；在 Web Console（21114）录入激活 | Pro license 官方文档 |

### 4.4 Web Client 与 Web Console 的区分（问题 3/5 详述）

| | Web Console | Web Client |
|---|---|---|
| 是什么 | **管理后台**（管理员用） | **浏览器控制端**（运维人员用，看/控远端画面） |
| 形态 | Pro 服务端内置，`http://<ip>:21114`（HTTPS 需反代） | Pro 自托管：`https://<域名>/web`；公共版：`https://rustdesk.com/web` |
| 授权 | Individual 档（$11.88/月，1 用户/20 设备）及以上均含 | 自托管版随 Pro 分发但需满足 `users×10+devices≥400`；公共版免费可用 |
| 与 OSS 的关系 | OSS **没有** | OSS 服务端原生支持 Web Client 的 WebSocket 端口（21118/21119），公共版 Web Client 可直接指向 OSS |
| 本项目用途 | 方案 B 中作为设备/用户/审计管理底座 | 两种方案的浏览器看屏入口 |

**公共 Web Client 实测记录（2026-11-16，Chrome，ZCode 内置浏览器）：**

1. `https://rustdesk.com/web` 可直接使用，标题 "RustDesk Web Client V2 Preview"，界面为中文（控制远程桌面 / 远程 ID 输入框 / 最近会话列表），截图 `poc/evidence/webclient-01-main.png`。
2. 设置 → 常规：含「启用 WebRTC P2P 连接」开关（默认开，可回退中继前的等待秒数），截图 `webclient-02-settings-general.png`。
3. 设置 → 网络 → **ID/中继服务器**：弹窗含 **ID 服务器、API 服务器、Key** 三个输入框 + 粘贴/导入按钮，可指向自建服务器，截图 `webclient-03/04-*.png`。
4. 设置 → 关于：版本 1.5.3，构建日期 2026-11-09，截图 `webclient-05-about-v153.png`。

> 注意：V2 为 **Preview**。将其作为生产入口前需在 PoC 中完成双平台实机查看验证；Pro 自托管 Web Client（若采购 Pro 且达门槛）为更稳的选择。

---

## 5. 浏览器查看 Windows / Android 画面的可行性论证（问题 4/5 详述）

1. **浏览器端不感知受控端 OS。** Web Client 只负责：经 WSS 与 hbbs/hbbr 交换信令、经中继收视频流并解码渲染。受控端的屏幕采集/编码由各平台原生客户端完成（Windows DXGI/GDI 采集；Android MediaProjection 采集）。因此「浏览器看 Windows」与「浏览器看 Android」对浏览器端是同一件事。
2. **服务端已原生支持浏览器链路。** OSS hbbs/hbbr 自带 WebSocket 端口（21118/21119，源码已证）；Pro FAQ 给出了标准 Nginx WSS 反代路径 `/ws/id → 21118`、`/ws/relay → 21119`（模板已落盘 `poc/deploy/nginx-rustdesk-webclient.conf`）。
3. **公共 Web Client 已实测可配置自建服务器**（3.3 节第 3 条证据）。
4. **端到端画面链路的最后一步（浏览器中真实解码 Windows/Android 画面）属运行时行为**，无法以静态方式证明，必须实机验证——已作为 PoC 验收项 T-5/T-6 列出。

结论：**没有发现任何技术或授权上的阻断项**；风险集中在 Android 系统授权弹窗与 Web Client V2 Preview 的成熟度。

---

## 6. 实际可行的集成方案（问题 6/7 详述）

### 6.1 方案 A（推荐，第一版采用）：OSS 服务端 + 公共 Web Client + 自建业务平台

```text
设备端：RustDesk 官方客户端（Windows 服务化/永久密码；Android 常驻/永久密码）
服务端：OSS hbbs + hbbr（Docker，1.1.16）
浏览器：rustdesk.com/web 公共 Web Client（配置指向自建 hbbs + Key）
业务平台：Next.js + Spring Boot（登录、设备台账、分组、会话记录）
在线状态：设备端轻量 Agent（脚本/小服务）定期心跳上报平台 —— hbbs OSS 无状态 API，必须自建
```

优点：

- 全部组件零许可成本；AGPL-3.0 合规压力小（服务端不修改、按网络部署即无需开放源码义务；客户端分官方原版，不改源码）。
- 平台完全掌控用户/设备/会话模型（与 PRD 第 3 章、TDD 第 4-7 章一一对应）。
- 升级路径平滑：将来需要集中审计/策略时再上 Pro（见 6.3）。

约束/代价：

- 在线状态靠自建 Agent 心跳（PRD 3.4 本就如此设计，无冲突）。
- 会话记录由平台记录「开始/结束」（用户从平台跳转 Web Client 时打点；结束时间依赖用户关闭页签时回传或心跳超时兜底）——**精确时长不如 Pro 审计**，第一版按 PRD 口径记录即可。
- 公共 Web Client 依赖 rustdesk.com 可访问（内网隔离环境需评估，或升级 Pro 自托管）。
- Web Client V2 是 Preview，需 PoC 实机验收（T-5/T-6）。

### 6.2 方案 B：Server Pro 一体化

采购 Pro 许可（按设备数选档：≤20 台设备 Individual $11.88/月；更多设备 Customized $23.88/月起），用 Web Console 做用户/设备/分组/审计，Web Client 做浏览器看屏，业务平台仅做品牌化门户与工单类扩展（调用 Pro API/CLI）。

约束：

- **自托管 Web Client 有许可证门槛**（users×10+devices≥400）。按本项目 2-10 用户 + 几十台设备的规模达不到，届时浏览器入口仍走公共 Web Client（Pro FAQ 官方支持该用法，需按文档配置 CORS）。
- Pro 无公开 API 契约文档，深度集成（读取设备在线状态、同步审计）前必须实测。

### 6.3 方案对比与决策

| 维度 | 方案 A（OSS+公共 Web Client） | 方案 B（Pro） |
|---|---|---|
| 许可成本 | 0 | $11.88/月起 |
| 浏览器看屏 | 公共 Web Client（V2 Preview） | 自托管 Web Client（需 400 门槛）或公共版 |
| 用户/设备/权限管理 | 业务平台自建（本来就要做） | Web Console 现成，平台做轻门户 |
| 审计/策略 | 平台自记（时长精度一般） | Pro 原生审计 + 策略下发 |
| 设备在线状态 | 自建 Agent 心跳 | Console 可见；API 需实测 |
| 内网可用性 | 依赖 rustdesk.com（入口） | 完全内网闭环 |
| 改动上游代码 | 无 | 无 |

**决策：第一版按方案 A 实施**，同时把「是否升级 Pro」作为验收后的决策点（触发条件：需要内网闭环入口、需要精确审计、设备规模上量）。

### 6.4 平台与 RustDesk 的对接点清单（全部基于已验证能力）

| 对接点 | 实现方式 | 依据 |
|---|---|---|
| 设备登记 | 现场安装客户端 → `--get-id`（Windows CLI）或客户端界面读取 ID → 平台录入 RustDesk ID | 源码 `core_main.rs:464` |
| 客户端配置下发 | 少量设备：手工/导入串；批量 Windows：部署脚本 + `--option`/`--config`；Android：扫码 `config={...}` | 3.2 节 |
| 在线状态 | 设备端 Agent 心跳 → 平台 API；平台按超时判 OFFLINE | PRD 3.4；OSS 无状态 API（4.1 节） |
| 发起远程查看 | 平台「远程查看」按钮 → 新窗口打开 `https://rustdesk.com/web`（首次引导填写 ID 服务器 + Key；V2 支持导入）→ 输入目标 RustDesk ID → 建立 WSS 连接 | 4.4 节实测 |
| 会话记录 | 平台在跳转时创建会话，Web Client 关闭回传/心跳超时结束 | TDD 4.3 |
| 免逐次确认 | Windows 永久密码 + 服务化；Android 常驻 + 永久密码（系统弹窗仅应用启动后一次） | 3.3 节 |
| 加密 | 客户端 Key = hbbs `id_ed25519.pub`；浏览器侧 TLS/WSS | 3.1/4.1 节 |

### 6.5 明确不做的事

- 不自研浏览器端远控协议/解码器（Web Client 未开源分发，`flutter/web` 目标不在主仓库中——已核实 `flutter/web` 目录不存在）。
- 不修改 hbbs/hbbr 协议或源码（保持 AGPL 合规最简路径，见 TDD §11.7）。
- 不假设任何未在官方文档/源码中验证的 API（Pro API 契约未公开，若走方案 B 先实测）。

---

## 7. 风险登记

| # | 风险 | 等级 | 缓解 |
|---|---|---|---|
| R1 | Android 屏幕采集系统弹窗（应用重启后需现场重新授权） | 高（影响 PRD「免逐次确认」） | 工业平板配置常驻 + 禁电池优化 + 厂商自启动；PoC 实测各厂商行为；必要时 MDM/kiosk |
| R2 | Web Client V2 为 Preview，功能/稳定性可能变化 | 中 | PoC 双平台实机验收；验收不达标则评估 Pro 自托管 Web Client 或延长 V1 使用窗口 |
| R3 | 公共 Web Client 依赖 rustdesk.com 可达 | 中（内网环境） | 评估现场网络；触发条件成立时升级方案 B |
| R4 | OSS 无审计与在线状态 API | 低 | 已在方案 A 中用 Agent 心跳 + 平台自记覆盖 |
| R5 | WebSocket 端口 X-Real-IP 伪造 | 中（安全） | 防火墙限制 21118/21119 仅代理可达，仅暴露 443 |
| R6 | 本机无 Docker，无法按生产方式复现部署 | 低 | 已产出 Docker Compose 与 Nginx 模板，现场一键复现；本机已用源码编译方式完成 hbbs/hbbr 运行时冒烟（见《PoC 验收报告》V-4），行为与镜像一致 |

---

## 8. 参考资料

**官方文档**

- OSS Docker 部署与端口：<https://rustdesk.com/docs/en/self-host/rustdesk-server-oss/docker/>
- 客户端配置（ID/中继/API/Key、--config、导入导出、自定义客户端）：<https://rustdesk.com/docs/en/self-host/client-configuration/>
- Android 客户端（屏幕采集权限、扫码配置）：<https://rustdesk.com/docs/en/client/android>
- Windows 客户端总览：<https://rustdesk.com/docs/en/client/windows/>
- Pro Web Console：<https://rustdesk.com/docs/en/self-host/rustdesk-server-pro/console/>
- Pro 许可证激活：<https://rustdesk.com/docs/en/self-host/rustdesk-server-pro/license/>
- Pro FAQ（Web Client 的 Nginx WSS/CORS 部署）：<https://rustdesk.com/docs/en/self-host/rustdesk-server-pro/faq/>
- 定价与 Web Client 门槛：<https://rustdesk.com/pricing>
- 公共 Web Client：<https://rustdesk.com/web>

**源码证据（本地 `poc/upstream/`）**

- 客户端服务端配置项：`rustdesk/flutter/lib/common.dart:3010`、`rustdesk/src/flutter_ffi.rs:1072`
- 客户端 CLI：`rustdesk/src/core_main.rs`（--config/--option/--password/--get-id/--assign）
- WebSocket 选项与策略下发：`rustdesk/flutter/lib/consts.dart:198`、`rustdesk/src/hbbs_http/sync.rs:540`
- 永久密码验证方式：`rustdesk/src/ui/index.tis:990-1105`
- Android 屏幕采集授权：`rustdesk/flutter/android/app/src/main/kotlin/com/carriez/flutter_hbb/PermissionRequestTransparentActivity.kt`
- hbbs 端口/WebSocket：`rustdesk-server/src/rendezvous_server.rs:162-226`
- hbbr 端口/WebSocket：`rustdesk-server/src/relay_server.rs:97-105`
- 本地运维通道（非 Web Console）：`rustdesk-server/src/common.rs:39-64`
- 配置文件命名：`rustdesk/libs/hbb_common/src/config.rs:878-882`

**社区佐证（仅作线索，不作结论依据）**

- Pro REST API 讨论：GitHub rustdesk/rustdesk discussion #14111；官方博客 rustdesk-for-enterprise（API Token + Python CLI）
- 自托管 Web Client 需 Pro 的社区讨论：r/rustdesk "Self-hosting a browser-accessible docker-based web client"
