---
name: ubuntu-proxy-singbox
description: "在 Ubuntu 系统上通过 sing-box 配置 anytls 代理客户端"
version: 1.0.0
author: Hermes Agent
metadata:
  hermes:
    tags: [ubuntu, proxy, sing-box, anytls, vpn]
    related_skills: [hermes-agent]
---

# Ubuntu Proxy Setup (sing-box + anytls)

在 Ubuntu 系统上用 sing-box 配置 anytls 代理客户端，配置全局 SOCKS5/HTTP 混合代理。

## 适用场景

- 用户提供一个订阅链接（Base64 编码的 anytls 节点）
- 需要在 Ubuntu 上配置全局代理
- 需要 systemd 管理、开机自启

## 工作流

### Step 1: 先报计划

在微信等无实时工具预览的平台上执行本技能时，必须先发送一条消息告知用户计划：

> "开始配置代理：①下载 sing-box → ②解析订阅链接 → ③写配置文件 → ④注册 systemd 服务 → ⑤设置全局环境变量 → ⑥测试连通性，大约 2-3 分钟"

### Step 2: 检查架构并下载 sing-box

```bash
# 获取系统架构
arch
# 返回 x86_64 或 aarch64 等
```

下载链接模板（使用国内镜像 gh-proxy.com 加速）：
```
https://gh-proxy.com/https://github.com/SagerNet/sing-box/releases/download/v{VERSION}/sing-box-{VERSION}-linux-{ARCH}.tar.gz
```

下载并安装：

```bash
cd /tmp
curl -fsSL "https://gh-proxy.com/https://github.com/SagerNet/sing-box/releases/download/v1.13.11/sing-box-1.13.11-linux-${ARCH}.tar.gz" -o sing-box.tar.gz
tar -xzf sing-box.tar.gz
sudo cp sing-box-*/sing-box /usr/local/bin/sing-box
sudo chmod +x /usr/local/bin/sing-box
sing-box version
```

### Step 3: 解析订阅链接

```bash
# 获取订阅内容（返回 Base64 编码的 URI 列表）
SUBSCRIPTION_URL="用户提供的链接"
curl -sL "$SUBSCRIPTION_URL" | base64 -d 2>/dev/null || curl -sL "$SUBSCRIPTION_URL"
```

解析 anytls URI 格式：
```
anytls://UUID@服务器:端口?sni=SNI域名&insecure=0#备注
```

关键字段提取：
- UUID: `390a8def-xxxx-xxxx-xxxx-xxxxxxxxxxxx`
- 服务器: `xxx.xxx.top`
- 端口: `xxxxx`
- SNI: `xxx.xxx.com`
- insecure: `0`（开启TLS验证）或 `1`（跳过验证）

优先选择香港/新加坡/日本节点（延迟较低）。

**重要：每 15 秒用 send_message 报一次进度**，例如：
- "✅ 订阅解析完成，节点: oldsd.youtu2.top:34102 (香港)"
- "⏳ 正在写配置文件..."

```json
{
  "log": {
    "level": "warn"
  },
  "inbounds": [
    {
      "type": "mixed",
      "tag": "mixed-in",
      "listen": "127.0.0.1",
      "listen_port": 1080
    }
  ],
  "outbounds": [
    {
      "type": "anytls",
      "tag": "proxy",
      "server": "提取的服务器",
      "server_port": 提取的端口,
      "password": "提取的UUID",
      "tls": {
        "enabled": true,
        "server_name": "提取的SNI",
        "insecure": false
      }
    },
    {
      "type": "direct",
      "tag": "direct"
    }
  ],
  "route": {
    "rules": [
      {
        "outbound": "direct",
        "domain": [
          "geosite:cn"
        ]
      }
    ],
    "auto_detect_interface": true
  }
}
```

写入配置文件：
```bash
sudo mkdir -p /opt/sing-box
# 将上面的 JSON 写入 /opt/sing-box/config.json
```

验证配置：
```bash
sing-box check -c /opt/sing-box/config.json
```

### Step 4: 注册 systemd 服务

创建服务文件 `/etc/systemd/system/sing-box.service`：

```ini
[Unit]
Description=sing-box proxy service
Documentation=https://sing-box.sagernet.org
After=network.target

[Service]
ExecStart=/usr/local/bin/sing-box run -c /opt/sing-box/config.json
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
DynamicUser=true
CacheDirectory=sing-box
StateDirectory=sing-box

[Install]
WantedBy=multi-user.target
```

启动并启用：
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now sing-box
```

### Step 5: 验证服务运行

```bash
# 检查进程
ps aux | grep sing-box | grep -v grep
# 检查端口
ss -tlnp | grep 1080
```

### Step 6: 测试代理连通性（关键步骤）

```bash
# HTTP 代理测试
curl -x http://127.0.0.1:1080 -s -o /dev/null -w "HTTP %{http_code}\n" --connect-timeout 5 https://www.google.com

# SOCKS5 代理测试
curl -x socks5h://127.0.0.1:1080 -s -o /dev/null -w "SOCKS5 %{http_code}\n" --connect-timeout 5 https://www.google.com

# 查看出口 IP
curl -x http://127.0.0.1:1080 -s --connect-timeout 5 https://ipinfo.io/json
```

### Step 7: 设置全局环境变量（可选，询问用户）

向用户确认是否需要全局代理。

写入 `/etc/environment`：

```bash
sudo tee /etc/environment > /dev/null << 'EOF'
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin"
http_proxy=http://127.0.0.1:1080
https_proxy=http://127.0.0.1:1080
all_proxy=socks5h://127.0.0.1:1080
HTTP_PROXY=http://127.0.0.1:1080
HTTPS_PROXY=http://127.0.0.1:1080
ALL_PROXY=socks5h://127.0.0.1:1080
no_proxy=localhost,127.0.0.1,::1,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
NO_PROXY=localhost,127.0.0.1,::1,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
EOF
```

> 注意：`/etc/environment` 在下次登录时生效，当前 shell 需手动 export。

### Step 8: 生成配置文档并保存记忆

向用户报告最终结果：

```
📋 配置完成报告

| 项目 | 状态 |
|------|------|
| sing-box | v1.13.11 ✅ |
| systemd 服务 | 已启用，开机自启 ✅ |
| 代理端口 | 127.0.0.1:1080 (SOCKS5+HTTP) ✅ |
| 出口节点 | 国家/城市/IP ✅ |

常用命令：
  sudo systemctl status sing-box
  sudo journalctl -u sing-box -f
  curl -x http://127.0.0.1:1080 -s ipinfo.io/json
```

## 注意事项 / 陷阱

### ⚠️ /var/log 可能是只读文件系统
在一些 Ubuntu 系统（特别是瘦客户端、嵌入式、WSL2）上，`/var/log` 可能是只读的。
**不要**在配置文件里设置 `log.output`，否则 sing-box 会启动失败。
解决方案：不写 `log.output` 字段，日志自动走 stderr → systemd journal。

### ⚠️ DynamicUser 的权限限制
systemd 服务使用 `DynamicUser=true` 时，服务进程不能写入：
- `/opt/sing-box/`（配置文件所在目录）
- `/var/log/`
- 大部分系统路径

系统会自动创建 `CacheDirectory` (`/var/cache/sing-box/`) 和 `StateDirectory` (`/var/lib/sing-box/`)，但建议日志直接走 journald。

### ⚠️ 订阅链接可能的格式
- 有些订阅返回纯文本，有些返回 Base64
- 先 `base64 -d` 解码，解码失败则直接使用原始内容
- 解析出的 anytls URI 可能包含 URL 编码的中文（如 `%E5%89%A9%E4%BD%99%E6%B5%81%E9%87%8F`），不影响功能

### ⚠️ 多节点订阅
同一个订阅可能返回几十个节点。默认选择第一个稳定节点即可，也可以主动问用户想用哪个地区的节点。

### ⚠️ 国内网络环境
如果 GitHub 直连慢，使用 `gh-proxy.com` 镜像加速下载：
```
https://gh-proxy.com/https://github.com/...
```

## 验证列表

- [ ] sing-box 可执行文件存在并可运行
- [ ] 配置文件语法检查通过
- [ ] systemd 服务 active (running)
- [ ] 1080 端口正在监听
- [ ] HTTP 代理能访问 Google（返回 200/302）
- [ ] SOCKS5 代理能访问 Google
- [ ] 出口 IP 确认在境外
- [ ] `/etc/environment` 已写入（如用户同意）
