---
name: docker-image-latest-pull
description: 拉取 Docker 镜像最新版本的策略——优先走国内镜像源，缓存落后时采用容器内 pip 升级或指定版本标签拉取
---

# Docker 镜像最新版本拉取策略

## 背景

国内 Docker 镜像源（daocloud、1ms、NJU、SJTUG 等）会缓存 `latest` 标签，但缓存可能滞后于 Docker Hub 数周甚至数月。直接 `docker pull xxx:latest` 可能拉回老版本。

## 策略优先级

### 第一优先：指定版本标签

如果已知最新版本号，直接用具体标签，镜像源没缓存时会透传到 Docker Hub 拉最新的：

```bash
docker pull nousresearch/hermes-agent:v2026.6.5
```

### 第二优先：多镜像源兜底

配置多个镜像源（daemon.json），Docker 按顺序尝试，某个源没有缓存就会透传 Docker Hub：

```json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://docker.1ms.run",
    "https://docker.nju.edu.cn",
    "https://docker.mirrors.sjtug.sjtu.edu.cn",
    "https://mirror.ccs.tencentyun.com"
  ]
}
```

修改后 reload：
```bash
systemctl reload docker
```

### 第三优先：容器内 pip 从源码升级（终极方案）

当镜像源全部缓存老版本时，进容器 pip 从 GitHub 源码升级。

**前置条件：** 容器里 Python venv 路径与 Hermes 镜像一致（`/opt/hermes/.venv/`）

#### 步骤

```bash
# 1. 宿主机克隆最新源码（用代理）
https_proxy=http://127.0.0.1:1080 git clone --depth 1 \
  https://github.com/NousResearch/hermes-agent /tmp/hermes-agent

# 2. 打包源码拷贝进容器
cd /tmp/hermes-agent
zip -r /tmp/hermes-src.zip .
docker cp /tmp/hermes-src.zip <容器名>:/tmp/hermes-src.zip

# 3. 容器内解压
docker exec <容器名> python3 -c "
import zipfile
with zipfile.ZipFile('/tmp/hermes-src.zip', 'r') as z:
    z.extractall('/tmp/hermes-src')
"

# 4. 安装 pip（容器默认没有 pip）
# 宿主机下载 pip wheel
https_proxy=http://127.0.0.1:1080 pip3 download \
  --dest /tmp/ --python-version 3.13 \
  --platform manylinux_2_35_x86_64 --only-binary=:all: pip
docker cp /tmp/pip-*.whl <容器名>:/tmp/pip.whl

# 容器内解压 wheel 到 venv
docker exec <容器名> python3 -c "
import zipfile, os
with zipfile.ZipFile('/tmp/pip.whl', 'r') as z:
    z.extractall('/opt/hermes/.venv/lib/python3.13/site-packages/')
"

# 创建 pip 可执行文件
docker exec <容器名> bash -c 'cat > /opt/hermes/.venv/bin/pip << '\''EOF'\''
#!/opt/hermes/.venv/bin/python3
import sys
from pip._internal.cli.main import main
sys.exit(main())
EOF
chmod +x /opt/hermes/.venv/bin/pip
ln -sf /opt/hermes/.venv/bin/pip /opt/hermes/.venv/bin/pip3
ln -sf /opt/hermes/.venv/bin/pip /opt/hermes/.venv/bin/pip3.13
'

# 5. 从本地源码安装（用清华源下载依赖）
docker exec <容器名> /opt/hermes/.venv/bin/pip install -e /tmp/hermes-src \
  -i https://pypi.tuna.tsinghua.edu.cn/simple

# 6. 验证并重启
docker exec <容器名> hermes --version
docker restart <容器名>
```

> **注意：** 清华 PyPI 源在容器内一般可直接访问，不需代理。如果网络不通，可换成其他国内 PyPI 源（阿里云、腾讯云等）。

## 验证版本

```bash
docker exec <容器名> hermes --version
# 期望输出例如：Hermes Agent v0.16.0 (2026.6.5)
```

## 检查 Docker Hub 最新版本

```bash
# 通过代理查 Docker Hub 标签
https_proxy=http://127.0.0.1:1080 curl -sL \
  'https://hub.docker.com/v2/repositories/nousresearch/hermes-agent/tags?page_size=10' | \
  python3 -c "import json,sys; [print(f\"  {r['name']:30s} updated: {r['last_updated'][:19]}\") for r in json.load(sys.stdin)['results']]"

# GitHub releases
https_proxy=http://127.0.0.1:1080 curl -sL \
  'https://api.github.com/repos/NousResearch/hermes-agent/releases/latest' | \
  python3 -c "import json,sys; r=json.load(sys.stdin); print(f\"{r['tag_name']} - {r['name']} ({r['published_at'][:10]})\")"
```
