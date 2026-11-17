# RustDesk 自建服务端 — 部署使用手册

**适用对象：** 工业设备远程运维平台 V1 基础栈（PoC 与生产同构）
**版本：** V1.0（2026-11-17）
**配套文档：** 《RustDesk 技术调研报告》《RustDesk PoC 验收报告》

> 本手册覆盖「自建 RustDesk 服务端 + 设备端接入 + 浏览器远程查看」的完整部署与使用流程。业务管理平台（Next.js + Spring Boot）尚未开发（按计划在 L2 现场验证通过后启动），本手册第 7 章给出预留对接点。

---

## 0. 你要部署的东西

| 组件 | 说明 | 来源 |
|---|---|---|
| hbbs | ID/会合服务：设备注册、心跳、连接协商 | 官方镜像 `rustdesk/rustdesk-server:1.1.16` |
| hbbr | 中继服务：转发客户端之间的流量 | 同上 |
| Nginx | HTTPS + WSS 反向代理，浏览器 Web Client 的入口 | 宿主机安装或容器 |
| RustDesk 客户端 | Windows / Android 设备端（受控方） | 官方 Release 1.5.x |
| Web Client | 浏览器控制端，**无需部署**，直接使用官方公共版 `https://rustdesk.com/web` | 官方托管 |

不需要部署的东西：

- **Web Console**：Pro 专有，OSS 版不存在，本方案不使用。
- **自托管 Web Client**：Pro 专有（且有许可证门槛），本方案用官方公共版。
- 业务平台：第二步开发，本手册不含。

## 1. 部署架构与端口规划

```text
                    ┌─ 443/tcp  ──► Nginx(HTTPS+WSS) ─┬─ /ws/id    → 127.0.0.1:21118 (hbbs WS)
浏览器 Web Client ──┤                                  └─ /ws/relay → 127.0.0.1:21119 (hbbr WS)
                    │
Windows/Android ────┼─ 21115/tcp ──► hbbs（NAT 类型测试）
原生客户端  ────────┼─ 21116/tcp+udp ► hbbs（ID 注册/心跳、打洞/连接）
                    └─ 21117/tcp ──► hbbr（中继）
```

**防火墙/安全组口径（标准模式）：**

| 端口 | 协议 | 对外开放 | 用途 |
|---|---|---|---|
| 443 | TCP | ✅ 必开 | 浏览器 Web Client（WSS） |
| 21115 | TCP | ✅ 开 | NAT 类型测试 |
| 21116 | TCP + **UDP** | ✅ 开 | ID 注册/心跳（UDP）、打洞/连接（TCP） |
| 21117 | TCP | ✅ 开 | 中继 |
| 21118 / 21119 | TCP | ❌ **不对公网开放** | WebSocket 端口，只由本机 Nginx 反代（官方安全警告：这两个端口信任 `X-Real-IP` 头且不校验，直接暴露可被伪造 IP） |
| 21114 | TCP | ❌ 不存在 | Web Console 端口，OSS 无此服务 |

> 可选「全 443 模式」：若现场网络只允许 443 出站，可在设备端客户端开启「使用 WebSocket」（`allow-websocket=Y`），所有流量改走 WSS，防火墙可只保留 443。原生直连模式性能更好，优先标准模式。

## 2. 服务器部署步骤

### 2.1 准备

- 一台 Linux 服务器（2C4G 起步即可），已安装 Docker 与 Docker Compose。
- 一个域名（如 `rd.example.com`）解析到该服务器，并备好证书（推荐 Let's Encrypt）。
- 部署目录约定：

```text
/opt/rustdesk/
├── docker-compose.hbbs-hbbr.yml     # 来自 poc/deploy/
├── nginx-rustdesk-webclient.conf    # 来自 poc/deploy/
├── data/                            # hbbs/hbbr 数据（密钥、sqlite），自动生成
└── certs/                           # HTTPS 证书
```

### 2.2 启动 hbbs / hbbr

```bash
mkdir -p /opt/rustdesk/data && cd /opt/rustdesk
# 放入 docker-compose.hbbs-hbbr.yml（poc/deploy/ 提供）
docker compose -f docker-compose.hbbs-hbbr.yml up -d
docker ps --filter name=hbbs --filter name=hbbr
```

compose 采用官方推荐的 `network_mode: "host"`（端口直接监听在宿主机）。首次启动自动完成：

- 生成密钥对 `data/id_ed25519`（私钥，保密）/ `data/id_ed25519.pub`（公钥 = 客户端 Key）；
- 初始化设备库 `data/db_v2.sqlite3`。

### 2.3 获取 Key（客户端要用的公钥）

```bash
cat /opt/rustdesk/data/id_ed25519.pub
# 输出形如：4T1tDXQkpm9lEtSSpME72QZmwhOXiD7wt27X9LTLLLI=（44 字符 Base64）
```

这个字符串就是所有客户端 / Web Client 里要填的 **Key**。`hbbs` 启动日志里也会打印同值。

### 2.4 配置 HTTPS + WSS 反向代理

1. 证书放入 `/opt/rustdesk/certs/`（或使用 certbot：`certbot certonly --standalone -d rd.example.com`）。
2. 编辑 `nginx-rustdesk-webclient.conf`：替换所有 `<YOUR_DOMAIN>` 与证书路径。
3. 启用并重载：

```bash
cp nginx-rustdesk-webclient.conf /etc/nginx/conf.d/rustdesk.conf
nginx -t && systemctl reload nginx
```

配置要点（模板已按官方 FAQ 写好，勿删）：

- `/ws/id` → `127.0.0.1:21118`，`/ws/relay` → `127.0.0.1:21119`；
- 必须带 `Upgrade`/`Connection` 头与 `proxy_read_timeout 120s`（否则会话约 1 分钟断一次）；
- 若使用公共 Web Client，模板内的 CORS 段按注释启用（对 `Origin: https://rustdesk.com` 返回 CORS 头、OPTIONS 返回 204）。

> Let's Encrypt 自动续期后需 `systemctl reload nginx`（certbot 的 deploy hook 一般已自动处理）。

### 2.5 配置防火墙 / 云安全组

按第 1 章端口表执行。示例（Ubuntu ufw）：

```bash
ufw allow 443/tcp
ufw allow 21115/tcp
ufw allow 21116/tcp
ufw allow 21116/udp
ufw allow 21117/tcp
# 21118/21119 保持关闭 —— 只允许本机 Nginx 访问
```

### 2.6 部署自检（一分钟）

```bash
# 1) 端口监听（应看到 21115/21116(tcp+udp)/21118 = hbbs，21117/21119 = hbbr）
lsof -nP -i :21115 -i :21116 -i :21117 -i :21118 -i :21119 | grep -E "LISTEN|UDP"

# 2) WebSocket 握手（两端口都应返回 HTTP/1.1 101 Switching Protocols）
KEY=x3JJHMbDL1EzLkh9GBhXDw==
for p in 21118 21119; do
  curl -s -i -N --max-time 5 \
    -H "Connection: Upgrade" -H "Upgrade: websocket" \
    -H "Sec-WebSocket-Key: $KEY" -H "Sec-WebSocket-Version: 13" \
    "http://127.0.0.1:$p/" | head -1
done

# 3) 对外 WSS（应同样返回 101 / 400 均可接受，只要不是超时）
curl -s -i --max-time 5 \
  -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Key: $KEY" -H "Sec-WebSocket-Version: 13" \
  "https://rd.example.com/ws/id" | head -1

# 4) 日志无报错
docker logs --tail 20 hbbs && docker logs --tail 20 hbbr
```

以上 4 步全部通过 = 服务端就绪（对应验收清单 T-1/T-2）。本手册发布前已在 macOS 上以同一 1.1.16 源码编译实测复现过这 4 步（见验收报告 V-4）。

## 3. 设备端接入（受控方）

> 版本统一用 **1.5.x**（当前 stable 1.5.3）。所有设备共用的三项配置：
> **ID 服务器** = `rd.example.com`（域名即可，端口默认；走 443 全 WS 模式时同样填域名）
> **中继服务器** = 留空（自动推导）
> **Key** = 2.3 节的公钥
> **API 服务器** = 留空（OSS 无 API）

### 3.1 Windows 设备

1. 安装：从 [GitHub Releases](https://github.com/rustdesk/rustdesk/releases) 下载 `rustdesk-1.5.3-x86_64-setup.exe` 安装（安装版自动注册系统服务、开机自启）。
2. 配置服务端：主界面 ID 旁 `⋮` 菜单 → **网络** → 解锁 → 填 ID 服务器 / Key → 应用。（批量部署可用 CLI：管理员运行
   `rustdesk --option custom-rendezvous-server rd.example.com` 与 `rustdesk --option key <公钥>`，或「设置 → 网络 → 导入/导出服务器配置」复制配置串。）
3. 免确认设置（核心）：
   - **设置 → 安全 → 无需求确认**（或：验证方式选「使用永久密码」）；
   - 设置永久密码：UI 操作，或管理员 CLI：`"C:\Program Files\RustDesk\RustDesk.exe" --password <强密码>`。
4. 读取设备 ID：主界面显示，或 CLI：`rustdesk --get-id`。把该 ID 登记到平台设备表（未来）或台账。
5. 验证：设备重启后不点任何东西，从浏览器（第 4 章）应能直接连上并输入永久密码观看。

> 全 443 模式：设置 → 网络 → 勾选「使用 WebSocket」。

### 3.2 Android 设备

1. 安装：应用市场或 [Releases](https://github.com/rustdesk/rustdesk/releases) 的 `rustdesk-1.5.3-universal.apk`。
2. 配置服务端（二选一）：
   - 手工：设置 → **ID/中继服务器** → 填 ID 服务器与 Key；
   - 扫码：在任意已配置好的客户端生成二维码（内容形如 `config={"host":"rd.example.com","key":"<公钥>"}`），Android 端扫「导入配置」。
3. 权限授予（系统要求，决定能不能看屏）：
   - 底部「共享屏幕」→ 启动服务 → 系统弹窗中授予**屏幕采集**；
   - 工业平板建议同时授予：输入控制（无障碍服务）、禁用电池优化、厂商自启动白名单 —— 保证 RustDesk 服务常驻。
4. 永久密码：设置 → 安全 → 设置永久密码（连接时输入它即可，不再需要设备端逐次「接受」）。
5. 读取设备 ID 并登记。

> **已知系统限制（务必知悉）**：Android 要求屏幕采集授权在 RustDesk **每次重新启动后**由现场重新点一次系统弹窗（官方文档+源码证实，无法绕过）。服务常驻期间连接免确认；应用被杀或设备重启后需现场操作一次。缓解手段即第 3 步的常驻配置；不同厂商 ROM 表现需按验收清单 T-7 实测记录。

## 4. 浏览器远程查看（控制方，运维人员）

无需安装任何东西，用官方公共 Web Client：

1. 打开 `https://rustdesk.com/web`（首次会提示是 V2 Preview）。
2. 右上角 `⋮` → **设置 → 网络 → ID/中继服务器**：
   - ID 服务器：`rd.example.com`（Web Client 自动走 `wss://…/ws/id` 与 `/ws/relay`，即 2.4 配置的两条路径）；
   - API 服务器：留空；
   - Key：粘贴 2.3 节公钥 → 确认。
3. 回主界面，输入目标设备的 **远程 ID** → 箭头连接。
4. 首次连接输入该设备的**永久密码**（Windows/Android 各自第 3 步设置的）。
5. 会话内：工具栏支持缩放/全屏/画质调节；关闭页签或点断开即结束会话。

使用约定（对齐 PRD）：

- 浏览器端即为「远程查看/控制」入口，**视频流直连自建 hbbs/hbbr，不经过任何业务服务器**；
- OSS 模式下浏览器每次连接需输永久密码（凭密码+ID 控制接入，无 Pro 的账号体系）；用户级权限、会话记录由后续业务平台补充；
- 公共 Web Client 需要现场网络能访问 rustdesk.com（加载页面）；若现场要求完全内网闭环，触发《验收报告》第 6 章的 Pro 升级评估。

## 5. 日常运维

| 事项 | 操作 |
|---|---|
| 查看日志 | `docker logs -f hbbs` / `docker logs -f hbbr`（设备注册、连接、错误都在这里） |
| 备份 | 备份整个 `data/` 目录。**`id_ed25519` 私钥是身份根基：泄露=任何人可伪装你的服务端；丢失=所有客户端要重配 Key**。建议每日备份且限制读取权限 |
| 更换 Key | 删除 `id_ed25519*` 重启 hbbs 会生成新密钥 → 所有客户端与 Web Client 必须同步换 Key，尽量避免 |
| 升级服务端 | 改 compose 中 image tag（如 `1.1.16` → 新版）→ `docker compose up -d`。数据在 `data/` 不受影响 |
| 强制走中继 | compose 中给 hbbs 加环境变量 `ALWAYS_USE_RELAY=Y`（P2P 打洞失败率高的内网环境） |
| 查看在线设备 | OSS 无设备列表界面：`docker logs hbbs | grep register` 看注册记录；正式的设备台账由业务平台承担（自建心跳） |

## 6. 常见问题排查

| 现象 | 原因与处理 |
|---|---|
| 客户端提示 Key 不匹配 / 连接被拒 | 客户端 Key 与 `data/id_ed25519.pub` 不一致。重新核对粘贴（注意复制完整 44 字符） |
| 设备一直不在线 | UDP 21116 被防火墙/安全组拦了（ID 注册走 UDP）。放行后看 hbbs 日志有无 register 记录 |
| Web Client 连不上服务器 | 依次检查：`https://域名/ws/id` 握手（2.6 第 3 步）→ Nginx 是否带 Upgrade 头与 `proxy_read_timeout 120s` → 证书是否有效 |
| 会话每约 1 分钟断开 | Nginx 缺 `proxy_read_timeout 120s`（官方 FAQ 明示的坑） |
| Web Client 能连但显示离线/取不到设备名 | 公共 Web Client 的 CORS 未配置：启用 2.4 节模板中的 CORS 段 |
| Windows 重启后连不上 | 未安装版（便携版）不会注册服务；改装版客户端并确认服务 `RustDesk Service` 在运行 |
| Android 重启后无法直连 | 预期行为（系统弹窗需现场重新授权屏幕采集），见 3.2 第 5 点 |
| hbbs 日志出现 Key 打印 | 正常：首次启动打印公钥，与 `id_ed25519.pub` 同值 |

## 7. 与业务平台的对接点（第二步开发预留）

平台开发时不改动本手册的任何组件，只做以下对接（详见《技术调研报告》§6.4）：

1. **设备登记**：录入「设备编号 ↔ RustDesk ID」（ID 来自 3.1/3.2 的 `--get-id` 或界面）。
2. **在线状态**：设备端加装轻量 Agent 定期心跳上报平台 API（OSS 无状态接口，平台自记；hbbs 日志仅作人工核对）。
3. **发起远程查看**：设备详情页「远程查看」按钮 → 新窗口打开 `https://rustdesk.com/web`（跳转前可提示用户完成第 4 章第 2 步的一次性配置）→ 自动带上目标 ID 参数（若 Web Client 支持 URL 传参，开发时实测；未验证前退化为手动输入 ID）。
4. **会话记录**：跳转时平台创建会话，Web Client 关闭回传/心跳超时兜底结束。

## 附录：本手册与验收清单的对应

| 手册章节 | 验收项（PoC 验收报告） |
|---|---|
| 2.2–2.3 | T-1 部署 OSS 服务端 |
| 2.4–2.5 | T-2 反向代理 |
| 3.1 | T-3 Windows 客户端接入 |
| 3.2 | T-4 Android 客户端接入 |
| 4 | T-5 / T-6 浏览器查看 Windows / Android |
| 3.2 第 5 点 + 4 | T-7 免确认复测 |
| 6 | T-8 异常路径排查 |

按第 2 章完成服务器部署、第 3 章完成两台实机接入、第 4 章完成浏览器双平台查看，即视为 L2 验收通过，可启动业务平台开发。
