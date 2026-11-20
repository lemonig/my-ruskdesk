# my-ruskdesk — 工业设备远程运维平台

基于 RustDesk 二次开发的远程设备管理平台。当前仓库状态：**RustDesk 技术验证已完成（含部署物），业务管理平台尚未开发**。

## 拉取后如何运行部署

### 0. 前置条件

| 项 | 要求 |
|---|---|
| 服务器 | 一台 Linux（2C4G 起步），已安装 Docker + Docker Compose |
| 域名 | 一个解析到服务器的域名（如 `rd.example.com`）+ HTTPS 证书（推荐 Let's Encrypt） |
| 端口 | 对外开放 `443/tcp`、`21115/tcp`、`21116/tcp+udp`、`21117/tcp`；**21118/21119 不对公网开放** |
| 客户端 | Windows/Android 设备安装 RustDesk 客户端（1.5.x） |

### 1. 部署 RustDesk 服务端（约 15 分钟）

```bash
git clone git@github.com:lemonig/my-ruskdesk.git
cd my-ruskdesk/poc/deploy

# 1) 启动 hbbs（ID服务）+ hbbr（中继）
mkdir -p /opt/rustdesk && cp docker-compose.hbbs-hbbr.yml /opt/rustdesk/ && cd /opt/rustdesk
docker compose -f docker-compose.hbbs-hbbr.yml up -d

# 2) 拿到客户端 Key（所有客户端/Web Client 都填这个）
cat data/id_ed25519.pub

# 3) HTTPS + WSS 反向代理
#    复制 nginx-rustdesk-webclient.conf 到 /etc/nginx/conf.d/rustdesk.conf，
#    替换其中所有 <YOUR_DOMAIN> 与证书路径，然后：
nginx -t && systemctl reload nginx
#    已完成的生产实例配置见 nginx-rustdesk-ruskesk.greandata1.com.conf（宝塔/通用均可用）

# 4) 按前置条件开放防火墙端口（21118/21119 只允许本机 Nginx 访问）
```

自检：见 [docs/RustDesk-自建服务端-部署使用手册.md](docs/RustDesk-自建服务端-部署使用手册.md) §2.6（端口监听、WebSocket 握手、日志四步，一分钟完成）。

### 2. 设备接入 + 浏览器查看

- **Windows 设备**：装 1.5.3 安装版 → 网络设置填「域名 + Key」→ 安全设置永久密码+免确认 → 读取设备 ID。
- **Android 设备**：装 APK → 扫码 `config={"host":"…","key":"…"}` → 授予屏幕采集/禁电池优化 → 设永久密码。
- **运维人员（浏览器）**：打开 https://rustdesk.com/web → 设置→网络→ID/中继服务器（填域名+Key）→ 输入设备 ID 与永久密码 → 浏览器内远程查看。

详细图文步骤与排查表：[docs/RustDesk-自建服务端-部署使用手册.md](docs/RustDesk-自建服务端-部署使用手册.md)。

### 3. 本仓库没有的东西

- `poc/upstream/`（上游源码取证克隆）已 gitignore，需要源码级分析时自行克隆：
  ```bash
  git clone --depth 1 https://github.com/rustdesk/rustdesk.git poc/upstream/rustdesk
  git clone --depth 1 https://github.com/rustdesk/rustdesk-server.git poc/upstream/rustdesk-server
  ```
- 业务管理平台（Next.js + Spring Boot）：按计划在 RustDesk 现场验证（L2，见验收报告 T-1~T-9）通过后开发，届时补充运行说明。

## 文档索引

| 文档 | 内容 |
|---|---|
| [docs/工业设备远程运维平台-产品需求文档.md](docs/工业设备远程运维平台-产品需求文档.md) | PRD：目标、角色、功能、MVP 验收标准 |
| [docs/工业设备远程运维平台-系统设计文档.md](docs/工业设备远程运维平台-系统设计文档.md) | TDD：架构、模块、数据库、实施顺序 |
| [docs/RustDesk-技术调研报告.md](docs/RustDesk-技术调研报告.md) | 源码级/文档级调研：客户端接入、OSS vs Pro、Web Client/Web Console 区分、集成方案与决策 |
| [docs/RustDesk-PoC验收报告.md](docs/RustDesk-PoC验收报告.md) | 已验证 20 项结果 + 现场 L2 验证清单 T-1~T-9 |
| [docs/RustDesk-自建服务端-部署使用手册.md](docs/RustDesk-自建服务端-部署使用手册.md) | **部署与日常使用主文档**（服务器/设备/浏览器/运维/排查） |

## 关键结论（一句话版）

- 自建服务端用开源 **hbbs/hbbr（OSS）**；**Web Console 与自托管 Web Client 是 Server Pro 专有**，本方案不用。
- 浏览器入口用**官方公共 Web Client**（rustdesk.com/web，v1.5.3 实测可用），经 Nginx WSS 指向自建服务端。
- **hbbs 必须加 `-r 域名:21117` 参数**（2026-11-20 实测）：浏览器收到中继地址后会按「端口+2」规则映射为 `wss://域名/ws/relay`(443)；若宣告裸 IP 则映射为 `wss://IP:21119`，该端口不对公网开放，浏览器连接永远卡在中继环节。详见 compose 注释。
- Windows 免确认 = 永久密码 + 服务化；**Android 受系统屏幕采集弹窗限制**，重启后需现场重新授权一次。
- 视频流始终只在 设备 ↔ hbbs/hbbr ↔ 浏览器 之间流动，业务平台不做媒体面。
