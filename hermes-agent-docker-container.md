---
name: hermes-agent-docker-container
description: 创建并配置 Hermes Agent Docker 容器——使用 bash + TTY + bashrc 自启动模式部署长期运行的 gateway 容器
---

# Hermes Agent Docker 容器部署

镜像官方地址：https://hub.docker.com/r/nousresearch/hermes-agent  
（GitHub: https://github.com/nousresearch/hermes）

## 为什么用 bash + TTY 模式

Hermes 镜像默认 entrypoint 是 `tini → entrypoint.sh`，会初始化数据卷后直接退出。要让容器长期运行 gateway 守护进程，需要：

- `--entrypoint /bin/bash` — bash 作为 PID 1
- `-t -i` — 分配 TTY 让 bash 保持活跃
- 在 `/root/.bashrc` 添加 PID-1 检测逻辑，自动启动 gateway

## 前置文件

容器启动前需要以下两个文件就位：

### 1. start_gateway.sh → `/opt/data/start_gateway.sh`

Gateway 重启循环脚本。放在数据卷中（持久化）：

```bash
#!/bin/bash
# Hermes Gateway auto-start script with restart loop
export HERMES_HOME=/opt/data
cd /opt/data

while true; do
    echo "[$(date)] Starting Hermes Gateway..."
    /opt/hermes/.venv/bin/hermes gateway run
    EXIT_CODE=$?
    echo "[$(date)] Gateway exited with code $EXIT_CODE, restarting in 3 seconds..."
    sleep 3
done
```

### 2. .bashrc → `/root/.bashrc`

在 bash 默认 .bashrc 末尾追加以下内容（**只追加一次**，不要重复）：

```bash
# Auto-start Hermes Gateway on container boot
if [ "$$" = "1" ] && [ -f /opt/data/start_gateway.sh ]; then
    setsid /opt/data/start_gateway.sh >/opt/data/gateway_boot.log 2>&1 &
fi
```

## 创建容器的标准流程

### 步骤 1：创建容器

```bash
docker create \
  --name <容器名> \
  --restart unless-stopped \
  --entrypoint /bin/bash \
  --workdir /opt/hermes \
  -t -i \
  -v <数据卷名>:/opt/data \
  -e PATH="/opt/data/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin" \
  -e PYTHONUNBUFFERED=1 \
  -e PLAYWRIGHT_BROWSERS_PATH=/opt/hermes/.playwright \
  -e HERMES_WEB_DIST=/opt/hermes/hermes_cli/web_dist \
  -e HERMES_HOME=/opt/data \
  nousresearch/hermes-agent:latest
```

> **参数说明：**
> - `--entrypoint /bin/bash` — bash 作为主进程（PID 1），替代默认的 tini+entrypoint.sh
> - `-t -i` — 分配 TTY 并保持 stdin 打开，让 bash 不退出
> - `--workdir /opt/hermes` — 默认工作目录
> - `-v <name>:/opt/data` — 数据卷持久化 config、skills、记忆等
> - 环境变量与 hermes_new 容器完全一致

### 步骤 2：启动并配置

```bash
# 启动容器
docker start <容器名>

# 复制 .bashrc（替换默认的 bashrc，追加自启动逻辑）
# 方法 A：从已有正确配置的容器复制
docker cp <已有容器>:/root/.bashrc /tmp/bashrc
docker cp /tmp/bashrc <新容器>:/root/.bashrc

# 方法 B：手动写入（先 docker exec 进去，再追加）
docker exec <容器名> bash -c 'cat >> /root/.bashrc << '\''EOF'\''
# Auto-start Hermes Gateway on container boot
if [ "$$" = "1" ] && [ -f /opt/data/start_gateway.sh ]; then
    setsid /opt/data/start_gateway.sh >/opt/data/gateway_boot.log 2>&1 &
fi
EOF'

# 复制 start_gateway.sh（从已有容器或直接写入）
docker exec <容器名> bash -c 'cat > /opt/data/start_gateway.sh << '\''EOF'\''
#!/bin/bash
export HERMES_HOME=/opt/data
cd /opt/data
while true; do
    echo "[$(date)] Starting Hermes Gateway..."
    /opt/hermes/.venv/bin/hermes gateway run
    EXIT_CODE=$?
    echo "[$(date)] Gateway exited with code $EXIT_CODE, restarting in 3 seconds..."
    sleep 3
done
EOF'
docker exec <容器名> chmod +x /opt/data/start_gateway.sh

# 添加 hermes 软链（方便 docker exec 直接调用）
docker exec <容器名> ln -sf /opt/hermes/.venv/bin/hermes /usr/local/bin/hermes
```

### 步骤 3：重启使配置生效

```bash
docker restart <容器名>
# 等待 3-5 秒，检查 gateway 是否启动
sleep 5 && docker exec <容器名> ps auxf
```

## 验证状态

正常运行时进程树应为：

```
PID 1  /bin/bash                          # bash（带 TTY 的主进程）
  ├─ /bin/bash start_gateway.sh           # 自动启动的 gateway 重启循环
  │   └─ python3 hermes gateway run       # 实际 gateway 进程
```

`hermes --version` 也应可用：

```bash
docker exec <容器名> hermes --version
```

## 常见陷阱

| 问题 | 原因 | 解决 |
|------|------|------|
| 容器反复重启（crash loop） | 没有 `-t` 分配 TTY，bash 立即退出 | 创建时加 `-t -i` |
| gateway 没启动 | .bashrc 缺少自启动代码，或 start_gateway.sh 不存在 | 检查 /root/.bashrc 和 /opt/data/start_gateway.sh |
| `hermes` 命令找不到 | PATH 不含 venv bin 目录 | 创建软链 `ln -sf /opt/hermes/.venv/bin/hermes /usr/local/bin/hermes` |
| .bashrc 重复追加自启动代码 | 多次复制导致重复行 | 执行前先检查是否已有：`grep "start_gateway" /root/.bashrc` |
