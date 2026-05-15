---
name: frp-reverse-proxy
description: >-
  通过 frp (Fast Reverse Proxy) 实现内网穿透，将本地 NAT 后的服务暴露到公网。
  架构：frps（公网服务器）+ frpc（内网客户端），支持 TCP/UDP/HTTP/HTTPS 隧道。
category: devops
---

# frp Reverse Proxy (内网穿透)

## When to use
- 需要从外网访问内网服务（SSH、Web 服务等）
- 拥有一台公网云服务器（阿里云/腾讯云/AWS）作为跳板
- 不想配置复杂的 VPN 方案

## Architecture

```
外网用户 → 公网IP:端口 → [安全组] → [UFW] → frps → frpc → 内网服务:端口
                      ↑ 云平台层          ↑ 系统层
```

## 实际部署案例

**本机（Wyse 5070）**：运行 Alist（5244）、宝塔面板（22048）、SSH（22）
**阿里云（47.98.123.49）**：frps 服务端，Ubuntu 22.04
**隧道映射**：
| 服务 | 本机端口 | 公网端口 |
|------|---------|---------|
| SSH | 22 | 6022 |
| Alist | 5244 | 5244 |
| 宝塔面板 | 22048 | 22049 |

---

## Step 1: Install frp on both machines

Check latest version at https://github.com/fatedier/frp/releases

```bash
FRP_VER="0.68.1"

# For Chinese servers — use ghproxy mirror (GitHub direct often times out in China)
curl -sL "https://ghproxy.net/https://github.com/fatedier/frp/releases/download/v${FRP_VER}/frp_${FRP_VER}_linux_amd64.tar.gz" -o /tmp/frp.tar.gz

# Extract and install
cd /tmp
tar xzf frp.tar.gz
sudo cp frp_${FRP_VER}_linux_amd64/frps /usr/local/bin/   # server binary → 公网云服务器
sudo cp frp_${FRP_VER}_linux_amd64/frpc /usr/local/bin/   # client binary → 内网本机
sudo mkdir -p /etc/frp
sudo chmod +x /usr/local/bin/frps /usr/local/bin/frpc
rm -rf frp_${FRP_VER}_linux_amd64* frp.tar.gz
```

## Step 2: Configure frps (server on public cloud)

`/etc/frp/frps.toml`:
```toml
bindPort = 7000

# Optional: admin dashboard (basic auth)
webServer.addr = "0.0.0.0"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "<choose-a-password>"
```

### frp Dashboard 访问
配置 `webServer.addr = "0.0.0.0"` 后，通过 `http://<公网IP>:7500` 可访问 frp 管理面板，查看各隧道状态、流量统计、客户端连接情况。注意安全组和 UFW 需放行 7500 端口。

## Step 3: Configure frpc (client on local machine)

`/etc/frp/frpc.toml`:
```toml
serverAddr = "<公网服务器IP>"
serverPort = 7000

[[proxies]]
name = "ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6022    # 避免与云服务器自身 SSH(22) 冲突

[[proxies]]
name = "web-service"
type = "tcp"
localIP = "127.0.0.1"
localPort = 5244
remotePort = 5244

# 添加更多服务 —— 只需复制 [[proxies]] 块，改名字和端口即可
[[proxies]]
name = "baota"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22048      # 本机端口不变
remotePort = 22049     # 远程端口可以不同（避免与云服务器自身端口冲突）
```

### 部署后添加新隧道

无需停机。只需编辑 `/etc/frp/frpc.toml` 追加 `[[proxies]]` 块，然后重启 frpc：

```bash
# 追加配置
tee -a /etc/frp/frpc.toml << 'EOF'

[[proxies]]
name = "new-service"
type = "tcp"
localIP = "127.0.0.1"
localPort = 12345
remotePort = 12345
EOF

# 重启 frpc 使新配置生效
systemctl restart frpc

# 检查是否成功
journalctl -u frpc --no-pager -n 10 | grep -E 'login|proxy success'

# 别忘了同时开云平台安全组 + 服务器 UFW 的新端口
```

## Step 4: systemd services (auto-start + restart)

**frps.service** (on server) → `/etc/systemd/system/frps.service`:
```ini
[Unit]
Description=Frp Server Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/frps -c /etc/frp/frps.toml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**frpc.service** (on client) → `/etc/systemd/system/frpc.service`:
```ini
[Unit]
Description=Frp Client Service
After=network.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/frpc -c /etc/frp/frpc.toml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable frps   # or frpc
systemctl start frps    # or frpc
```

## Step 5: Open firewall ports (TWO layers!)

**Layer 1 — Cloud Security Group (阿里云/腾讯云/AWS 控制台)**

流量首先经过云平台安全组，必须在这里放行端口。路径：
- 阿里云：ECS 控制台 → 安全组 → 配置规则 → 入方向 → 添加规则
- 腾讯云：CVM 控制台 → 安全组 → 入站规则

必须开放的端口（来源 `0.0.0.0/0`，协议 TCP）：
| 端口 | 用途 |
|------|------|
| 7000 | frp 控制通道（frpc ↔ frps 通讯） |
| 6022 | SSH 隧道端口 |
| 5244, 22049 | 各业务服务端口（Alist, Baota 面板等） |
| 7500 | frp dashboard（可选） |

**Layer 2 — Server UFW (if active)**:
```bash
ufw allow 7000/tcp comment 'frp control'
ufw allow 6022/tcp comment 'frp ssh tunnel'
ufw allow 5244/tcp comment 'frp service tunnel'
ufw allow 7500/tcp comment 'frp dashboard'
ufw reload
```

## Step 6: 验证连通性

```bash
# On server — check frps is listening on all expected ports
ss -tlnp | grep frps

# On client — check frpc is running and connected
systemctl status frpc
journalctl -u frpc --no-pager -n 20  # 查看连接日志

# Test SSH tunnel (through frp)
ssh -p 6022 root@<公网IP>

# Test Alist
curl -v http://<公网IP>:5244

# Test Baota panel — 必须用浏览器 UA！
curl -sk -H 'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36' \
  https://<公网IP>:22049/<admin_path>
```

## 关于宝塔面板的特殊问题

宝塔面板通过 frp 暴露到公网时，需要特别注意以下几点：

### 1. 爬虫检测（User-Agent 拦截）

宝塔面板的 `is_spider()` 函数会检查 User-Agent。用 curl 测试时默认 UA 会被拦截，返回 404（页面内容为宝塔 Flask 自定义的 nginx 风格 404 页）。**这不是 frp 的问题，是宝塔本身的安全机制。**

**解决方案**：
- 用真实浏览器访问，所有现代浏览器都带正常 UA，不受影响
- 如需用 curl/wget 测试，加 `-H 'User-Agent: Mozilla/5.0 (...)'`
- 宝塔的 `bt 11` 命令可以关闭 IP+UA 检测，但建议保留（这是安全机制）

### 2. 管理入口路径

宝塔面板默认有一个随机管理路径（存在 `/www/server/panel/data/admin_path.pl`），访问需加路径：
```
https://<公网IP>:22049/<admin_path>
```
例如：`https://47.98.123.49:22049/e9d6ce01`

### 3. HTTPS 证书警告

宝塔面板使用自签名 SSL 证书，浏览器访问时会提示安全风险。点「继续访问」即可。

## Security Considerations (端口暴露风险)

将内网服务暴露到公网会带来攻击面。以下是各端口的风险等级和应对措施：

| 服务 | 风险等级 | 说明 | 推荐防护 |
|------|---------|------|---------|
| SSH (22→6022) | ⚠️ 中高 | 密码爆破脚本全天候扫端口 | **仅允许密钥登录** + 禁用密码 + fail2ban |
| Web面板 (如 Baota 22048→22049) | ⚠️ 中 | Web 服务面大，可能有漏洞 | 面板自带密码 + 限制来源IP + 定期更新 |
| Alist (5244) | ⚠️ 中 | 文件管理服务，暴露文件系统风险 | 设强密码 + 禁匿名访问 + 日志监控 |
| frp dashboard (7500) | ⚠️ 低 | 有 basic auth，但密码明文配 | 建议绑定内网地址 `127.0.0.1` 或加 nginx 反代 |
| frp 控制 (7000) | ✅ 低 | 只接受 frpc 协议握手，通用扫描无意义 | 保持默认即可 |

### 推荐加固步骤

1. **SSH 禁用密码**：`/etc/ssh/sshd_config` 中设 `PasswordAuthentication no`，仅用密钥登录
2. **装 fail2ban**：5次失败即封IP，`apt install fail2ban` 即开即用
3. **最小暴露原则**：只开真正需要外网的端口，不用的就关掉
4. **面板/服务自身鉴权**：在服务本身上好锁（强密码、IP白名单、双因素）

## Pitfalls

- **阿里云安全组是必经屏障**：UFW 开了还不够，安全组没开端口的话流量在云平台层就被丢弃，根本到不了服务器。常见错误：配完 frp 还超时，结果发现是安全组没加规则。**每次添加新隧道都要同步开安全组+UFW 两层端口。**
- **ghproxy 镜像**：中国服务器直连 GitHub Release 普遍超时，必须用 `ghproxy.net`。也支持 pip（`pip install -i https://pypi.tuna.tsinghua.edu.cn/simple`）。
- **frpc loginFailExit**：默认开启，连接失败后进程退出（而非后台重试）。配合 systemd `Restart=on-failure` 实现自动重连。如果不需要自动退出，可以设 `loginFailExit = false`。
- **端口冲突**：SSH 隧道不要用 22（会和云服务器自己的 SSH 抢端口），习惯用 6022 或 2222。
- **dashboard 安全**：frp dashboard 使用 basic auth，但密码明文配置。建议只绑定内网地址（`webServer.addr = "127.0.0.1"`），或通过 nginx 反向代理加一层鉴权。
- **frp 版本一致性**：frps 和 frpc 最好用相同大版本，避免协议不兼容。
- **宝塔面板 curl 测试注意**：curl 默认 UA 会被宝塔爬虫检测拦截返回 404，需用浏览器 UA 头。这容易误判为 frp 隧道不通，实际上是宝塔本身的安全机制。

## Verification

```bash
# On server — check frps is listening
ss -tlnp | grep frps

# On client — check frpc is running and connected
systemctl status frpc
journalctl -u frpc --no-pager -n 20  # 查看连接日志

# From any machine — test SSH tunnel
ssh -p 6022 user@<公网IP>

# Test web service tunnel
curl -v http://<公网IP>:5244
```

## References

- 下载镜像：`https://ghproxy.net/https://github.com/fatedier/frp/releases/download/v0.68.1/frp_0.68.1_linux_amd64.tar.gz`
- 官方文档：https://github.com/fatedier/frp
