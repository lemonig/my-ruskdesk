# RustDesk PoC 验收报告

**项目：** 工业设备远程运维平台
**版本：** V1.0
**日期：** 2026-11-16（本机验证）/ 现场验证项待执行
**性质：** 《系统设计文档》第 10 章「第一步：RustDesk 技术验证」的验收产出

> 验收口径：PoC 的目的是**证明技术路线可行**，而不是交付生产部署。因此本报告把验证项分为两级：
> - **L1（本机已完成）**：源码级/文档级/浏览器 UI 级验证，在本机完成；
> - **L2（现场待执行）**：需要 Linux 服务器 + Windows/Android 实机的运行时验证，配套部署物已备好（`poc/deploy/`），按第 4 章清单执行。

---

## 1. PoC 目标（对齐需求）

| 需求出处 | 验证点 |
|---|---|
| 任务要求 2 | Windows / Android 客户端接入自建服务端的确认方式 |
| 任务要求 3 | 官方 Web Client 的部署及授权要求 |
| 任务要求 4 | Web Client 与 Web Console 的区分 |
| 任务要求 5 | 浏览器能否查看 Windows / Android 设备画面 |
| 任务要求 6/7 | 不假设未验证 API；输出实际可行集成方案 |
| 任务要求 8 | 尽量复用 RustDesk 代码，不重写远控通信模块 |
| PRD MVP #5/#6 | 浏览器查看 Windows / Android 设备屏幕 |
| PRD MVP #7 | 已登记授权设备的只读/控制查看无需逐次现场确认 |

## 2. L1 验证结果（本机已完成）

### 2.1 V-1 源码级验证（结论：全部通过）

| # | 验证项 | 结果 | 证据 |
|---|---|---|---|
| V-1.1 | 客户端内置自建服务端配置项（ID/中继/API/Key） | ✅ 通过 | `poc/upstream/rustdesk/flutter/lib/common.dart:3010`、`3672`；`src/flutter_ffi.rs:1072` |
| V-1.2 | CLI 配置通道（--option / --config / --password / --get-id） | ✅ 通过 | `poc/upstream/rustdesk/src/core_main.rs:425/464/489/502` |
| V-1.3 | Windows 免确认模式（`verification-method=use-permanent-password`） | ✅ 通过 | `poc/upstream/rustdesk/src/ui/index.tis:990-1105` |
| V-1.4 | Android 屏幕采集必须经系统弹窗（`createScreenCaptureIntent`） | ✅ 通过（证实限制存在） | `poc/upstream/rustdesk/flutter/android/.../PermissionRequestTransparentActivity.kt:22-28` |
| V-1.5 | hbbs 监听 21115/21116(TCP+UDP)/21118(WS) | ✅ 通过 | `poc/upstream/rustdesk-server/src/rendezvous_server.rs:175-226` |
| V-1.6 | hbbr 监听 21117/21119(WS) | ✅ 通过 | `poc/upstream/rustdesk-server/src/relay_server.rs:97-105` |
| V-1.7 | OSS 无 Web Console（1.1.16 发布版源码无 console 代码；master 新增的本地通道仅绑 127.0.0.1） | ✅ 通过 | 1.1.16 `src/` grep `console` 零命中；master `src/common.rs:39-64` |
| V-1.8 | 两个服务仓库许可证 AGPL-3.0 | ✅ 通过 | `poc/upstream/rustdesk-server/LICENSE` |
| V-1.9 | 客户端配置文件 RustDesk.toml / RustDesk2.toml 命名机制 | ✅ 通过 | `poc/upstream/rustdesk/libs/hbb_common/src/config.rs:878-882` |
| V-1.10 | 主仓库不含 Flutter Web 构建目标（不可自研网页端复用主仓库代码） | ✅ 通过 | `flutter/web/` 目录不存在 |

### 2.2 V-2 文档级验证（结论：全部通过）

| # | 验证项 | 结果 | 证据 |
|---|---|---|---|
| V-2.1 | OSS 服务端原生支持 Web Client（21118/21119） | ✅ | 官方 OSS Docker 文档 |
| V-2.2 | Web Client 的 Nginx WSS 部署口径（/ws/id→21118，/ws/relay→21119，120s 超时） | ✅ | Pro 官方 FAQ |
| V-2.3 | 公共 Web Client + 自建服务器受官方支持（需 CORS 配置口径） | ✅ | Pro 官方 FAQ 第 9 步 |
| V-2.4 | 自托管 Web Client 授权门槛 `users×10+devices≥400` | ✅ | rustdesk.com/pricing |
| V-2.5 | Web Console 为 Pro 专有（21114，用户/设备/策略/审计/Token） | ✅ | Pro console 官方文档 |
| V-2.6 | Pro API 形态：Token（Console 创建）+ 官方 Python CLI；无公开正式 API 契约 | ✅ | console 文档 + GitHub 讨论 #14111 |
| V-2.7 | Android「每次启动需重新请求屏幕采集/输入控制权限」 | ✅ | 官方 Android 文档原文 |
| V-2.8 | 自定义客户端生成器（预置配置）仅 Pro Basic 档及以上 | ✅ | client-configuration 文档 + pricing |

### 2.3 V-3 公共 Web Client 实测（结论：通过，附保留意见）

**实测环境：** ZCode 内置 Chrome，2026-11-16，目标 `https://rustdesk.com/web`。

| # | 验证项 | 结果 | 证据（poc/evidence/） |
|---|---|---|---|
| V-3.1 | 公共 Web Client 可用，界面中文 | ✅ | webclient-01-main.png |
| V-3.2 | 设置 → 网络 → ID/中继服务器：**含 ID 服务器 / API 服务器 / Key 三项配置**，可指向自建服务器 | ✅ | webclient-03/04-*.png |
| V-3.3 | 支持配置串导入（弹窗内粘贴/导入按钮） | ✅ | webclient-04-id-relay-server-form.png |
| V-3.4 | 默认启用 WebRTC P2P 连接（可关，回退中继） | ✅ | webclient-02-settings-general.png |
| V-3.5 | 版本信息：1.5.3，构建 2026-11-09（与客户端 stable 同版本号） | ✅ | webclient-05-about-v153.png |
| V-3.6 | 在浏览器中实际解码 Windows/Android 画面 | ⏳ L2 | 依赖实机，见 T-5/T-6 |

保留意见：V2 标注 **Preview**，作为生产入口前必须完成 L2 双平台实机验收。

### 2.4 V-4 本机运行时冒烟测试（结论：通过）

**环境：** macOS arm64；本机无 Docker，改用 rustup 安装 Rust 1.100.0，从源码（tag 1.1.16，commit `73523b3`，子模块 hbb_common `83419b65`）编译 hbbs/hbbr（debug，编译 11 分 48 秒），于 2026-11-17 00:38 本机实测。

| # | 验证项 | 结果 | 证据（实测输出） |
|---|---|---|---|
| V-4.1 | hbbs 启动并生成密钥对 | ✅ | 日志：`Private/public key written to id_ed25519/id_ed25519.pub`，并打印公钥 `Key: 4T1tDXQkpm9…` |
| V-4.2 | hbbs 端口监听：21115/TCP、21116/TCP+UDP、21118/TCP | ✅ | `lsof`：`hbbs TCP *:21115 (LISTEN)`、`TCP *:21116 (LISTEN)`、`UDP *:21116`、`TCP *:21118 (LISTEN)` |
| V-4.3 | hbbr 端口监听：21117/TCP、21119/TCP | ✅ | 日志：`Listening on tcp :21117`、`Listening on websocket :21119`；`lsof` 一致 |
| V-4.4 | WebSocket 握手（Web Client 传输层） | ✅ | 对 21118 与 21119 发起握手均返回 `HTTP/1.1 101 Switching Protocols` |
| V-4.5 | 数据库初始化 | ✅ | 日志：`DB_URL=./db_v2.sqlite3` |

> 该测试与官方 Docker 镜像的差距仅在运行方式（本机源码编译 vs 容器），程序行为一致（同一 tag 源码）。现场 T-1 仍需按部署物用 Docker 复现一次作为交付基线。

**本机未能覆盖的项（转入 L2）：**
- macOS 客户端接入注册（GitHub Release 资产下载通道在本环境不可达，GUI 客户端未装成）；
- 真实设备注册/心跳、浏览器端到端画面（本机无 Windows/Android 受控端）。

### 2.5 环境说明

## 3. 综合结论

1. **技术路线可行，无阻断项。** 「自建 hbbs/hbbr + 原生客户端（Windows/Android）+ 浏览器 Web Client」的链路，每一环节都有官方文档或源码级证据；唯一未闭合的是运行时端到端画面（L2）。
2. **不需要重写任何远控通信模块。** 平台只做设备/用户/会话管理与入口跳转；屏幕采集、编码、传输、NAT、中继全部复用 RustDesk（满足任务要求 8）。
3. **「免逐次现场确认」**：Windows 完全可行；Android 受系统限制（应用/设备重启后需重新授权一次屏幕采集），已在调研报告 §3.3 给出缓解方案，需 L2 确认实际表现。
4. **Web Client ≠ Web Console**：第一版采用「方案 A（OSS + 公共 Web Client + 自建业务平台）」，详见调研报告 §6；升级 Pro 的触发条件见本报告第 6 章。

## 4. L2 现场验证清单（待执行）

**环境准备：** 一台可公网/内网访问的 Linux 服务器（Docker）、一台 Windows 工业电脑、一台 Android 工业平板；服务器域名 + HTTPS 证书。

| # | 步骤 | 操作 | 通过标准 |
|---|---|---|---|
| T-1 | 部署 OSS 服务端 | `poc/deploy/docker-compose.hbbs-hbbr.yml` 启动 | hbbs/hbbr 容器运行；`./data/id_ed25519.pub` 生成；`rustdesk-utils doctor` 或端口检查通过（21115-21119） |
| T-2 | 反向代理 | 按 `poc/deploy/nginx-rustdesk-webclient.conf` 配置 HTTPS+WSS | 浏览器访问 `https://<域名>` 正常；`wss://<域名>/ws/id` 握手成功 |
| T-3 | Windows 客户端接入 | 安装 1.5.x 客户端，配置 ID 服务器+Key，设置永久密码，服务化 | hbbs 日志/在线可见；`--get-id` 返回 ID；重启 Windows 后无需现场操作即可被连接 |
| T-4 | Android 客户端接入 | 安装 1.5.x 客户端，扫码/手工配置，授权屏幕采集，设永久密码，禁电池优化 | hbbs 在线可见；获得 ID；**记录**：锁屏/重启/杀进程后重新授权的行为 |
| T-5 | 浏览器查看 Windows | rustdesk.com/web → 配置 ID 服务器+Key → 输入 Windows ID → 连接 | 浏览器内出现 Windows 画面，可缩放/全屏，主动断开生效 |
| T-6 | 浏览器查看 Android | 同上，目标为 Android ID | 浏览器内出现 Android 画面，可缩放/全屏，主动断开生效 |
| T-7 | 免确认复测 | T-5/T-6 连续 3 次连接（设备端无人操作） | Windows 3/3 免确认；Android 在服务常驻期间 3/3 免确认，并记录重启后表现 |
| T-8 | 异常路径 | 断网/杀掉 hbbs/Web Client 关闭页签 | 连接失败有明确提示；平台会话记录能标记异常结束 |
| T-9 | （可选）Pro 评估 | 若决策采购 Pro：trial 激活 21114 Console，实测 `GET /api/devices` 等接口 | 输出 Pro API 实测补充记录 |

**L2 通过判据：** T-1~T-7 全部通过 ⇒ PRD MVP #5/#6/#7 的技术前提成立，进入《系统设计文档》第 10 章「第二步：基础业务平台」开发；T-6 或 T-7 失败 ⇒ 停下重新评估（Android 缓解方案 / Web Client 换 Pro 自托管）。

## 5. 对 PRD MVP 验收标准的影响预判

| MVP # | 条目 | 当前状态 |
|---|---|---|
| 5 | 浏览器查看 Windows 屏幕 | 技术前提已证实，待 T-5 实机确认 |
| 6 | 浏览器查看 Android 屏幕 | 技术前提已证实，待 T-6 实机确认 |
| 7 | 已登记授权设备免逐次现场确认 | Windows ✅（源码证实）；Android 受系统弹窗限制，需 T-7 确认缓解效果 |
| 1/2/3/4/8/9/10 | 平台功能（登录/登记/状态/权限/断开/记录） | 属「第二步：基础业务平台」范围，不受 RustDesk PoC 阻塞 |

## 6. 升级 Server Pro 的触发条件（记录为决策点）

满足任一条件时启动 Pro 评估（参考调研报告 §6.2）：

1. 需要完全内网闭环的浏览器入口（现场无法访问 rustdesk.com）；
2. 需要精确的会话审计/策略下发/集中用户管理；
3. 设备规模明显增长（>50 台）且要求厂商级支持。

注意：即使采购 Pro，自托管 Web Client 的授权门槛为 `users×10+devices≥400`；小规模方案下浏览器入口仍用公共 Web Client（官方 FAQ 支持）。

## 7. 交付物清单

| 路径 | 说明 |
|---|---|
| `docs/RustDesk-技术调研报告.md` | 技术调研文档（本报告的上游依据） |
| `docs/RustDesk-PoC验收报告.md` | 本报告 |
| `poc/deploy/docker-compose.hbbs-hbbr.yml` | OSS hbbs/hbbr 部署模板（T-1） |
| `poc/deploy/nginx-rustdesk-webclient.conf` | HTTPS+WSS+CORS 反代模板（T-2） |
| `poc/evidence/webclient-0*.png` | 公共 Web Client 实测截图（V-3 证据） |
| `poc/upstream/rustdesk`、`poc/upstream/rustdesk-server` | 源码取证克隆（浅克隆，可随时删除重建） |

## 8. 变更记录

| 日期 | 变更 |
|---|---|
| 2026-11-16 | 初版：L1 验证完成（源码/文档/浏览器 UI），L2 清单发布 |
| 2026-11-17 | 补充 V-4：本机从源码编译并运行 hbbs/hbbr 1.1.16，端口监听与 WebSocket 握手实测通过；macOS 客户端接入因下载通道受限转入 L2 |
