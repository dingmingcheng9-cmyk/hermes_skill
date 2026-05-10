---
name: docker-container-backup
description: Docker 容器全量备份——跨机器通用方案，备份容器配置+卷数据+重建脚本到多块硬盘，带周定时任务
---

# Docker 容器全量备份

## 概述

通用 Docker 容器备份方案，备份以下三部分以实现「一键重建」：

| 备份项 | 说明 |
|---|---|
| `container-inspect.json` | 完整容器配置（端口、env、挂载、重启策略等） |
| `volume-data.tar.gz` | 容器 volume / bind mount 中的持久化数据 |
| `recreate.sh` | 自动生成的一键重建脚本 |

备份到 **多块硬盘**（互备），每次覆盖旧备份（仅保留最新版）。

---

## 快速安装

### 1. 创建备份脚本

将下面的脚本保存到 `/opt/scripts/docker-backup.sh`：

```bash
#!/bin/bash
# ============================================================
# Docker 容器全量备份脚本
# 修改以下变量适配你的环境
# ============================================================
set -e

BACKUP_DATE=$(date +%Y%m%d_%H%M%S)

# ====== 用户配置区域 ======
# 备份目标目录（支持多个，每行一个路径）
BACKUP_TARGETS=(
    "/mnt/disk1/backups"
    "/mnt/disk2/backups"
)

# 要备份的容器列表
CONTAINERS=(
    "container-a"
    "container-b"
)

# 需要预导出镜像的容器镜像列表（避免每周重复导出大镜像）
SHARED_IMAGES=(
    "nousresearch/hermes-agent:latest"
    "xhofe/alist:latest"
)
# ==========================

log() { echo "[$(date '+%H:%M:%S')] $1"; }

# ---------- 前置 ----------
for target in "${BACKUP_TARGETS[@]}"; do
    mkdir -p "$target"
done

# ---------- 镜像预导出（仅首次） ----------
export_images_once() {
    for img in "${SHARED_IMAGES[@]}"; do
        local safe_name
        safe_name=$(echo "$img" | tr '/:' '_')
        for target in "${BACKUP_TARGETS[@]}"; do
            local shared_dir="$target/_shared_images"
            local img_file="$shared_dir/$safe_name.tar.gz"
            if [ ! -f "$img_file" ]; then
                log "📦 首次导出镜像 $img → $img_file"
                mkdir -p "$shared_dir"
                docker save "$img" | gzip > "$img_file"
            fi
        done
    done
    log "   ✅ 镜像已就绪"
}
export_images_once

# ---------- 备份单个容器 ----------
backup_container() {
    local container="$1"
    local dest="$2/$container"
    rm -rf "$dest"
    mkdir -p "$dest"

    # 1. 导出 inspect
    log "  → $container: 导出 inspect"
    docker inspect "$container" > "$dest/container-inspect.json" 2>/dev/null || {
        log "  ⚠️  容器 $container 不存在，跳过"
        rm -rf "$dest"
        return 1
    }

    # 2. 检测挂载类型并备份数据
    log "  → $container: 备份卷数据"

    # 收集所有 volume 挂载点
    local volumes
    volumes=$(docker inspect "$container" --format '{{range .Mounts}}{{if eq .Type "volume"}}{{.Name}}{{"\n"}}{{end}}{{end}}' 2>/dev/null)
    local bind_mounts
    bind_mounts=$(docker inspect "$container" --format '{{range .Mounts}}{{if eq .Type "bind"}}{{.Source}}→{{.Destination}}{{"\n"}}{{end}}{{end}}' 2>/dev/null)

    # 备份 volumes
    local vol_count=0
    if [ -n "$volumes" ]; then
        while IFS= read -r vol; do
            [ -z "$vol" ] && continue
            docker run --rm -v "$vol":/data -v "$dest":/backup alpine \
                tar czf "/backup/volume-$vol.tar.gz" -C /data . 2>/dev/null
            vol_count=$((vol_count + 1))
        done <<< "$volumes"
    fi

    # 备份 bind mounts（只收集源路径下的配置数据，不备份大数据）
    local bind_files="$dest/bind-mounts.txt"
    : > "$bind_files"
    if [ -n "$bind_mounts" ]; then
        echo "$bind_mounts" > "$bind_files"
    fi

    log "  → $container: 生成重建脚本"
    generate_recreate_script "$container" "$dest" "$volumes"

    log "  ✅ $container 备份完成（$(du -sh "$dest" | cut -f1)）"
}

# ---------- 生成 recreate.sh ----------
generate_recreate_script() {
    local container="$1"
    local dest="$2"
    local volumes="$3"

    local img
    img=$(docker inspect "$container" --format '{{.Config.Image}}' 2>/dev/null)
    local hostname
    hostname=$(docker inspect "$container" --format '{{.Config.Hostname}}' 2>/dev/null)
    local entrypoint
    entrypoint=$(docker inspect "$container" --format '{{json .Config.Entrypoint}}' 2>/dev/null)
    local restart
    restart=$(docker inspect "$container" --format '{{.HostConfig.RestartPolicy.Name}}' 2>/dev/null)
    local port_maps
    port_maps=$(docker inspect "$container" --format '{{range $p, $conf := .HostConfig.PortBindings}}{{$p}}:{{range $conf}}{{.HostIp}}:{{.HostPort}}{{"\n"}}{{end}}{{end}}' 2>/dev/null)
    local env_vars
    env_vars=$(docker inspect "$container" --format '{{range .Config.Env}}{{.}}{{"\n"}}{{end}}' 2>/dev/null)

    # 从镜像名推测共享镜像文件名
    local safe_name
    safe_name=$(echo "$img" | tr '/:' '_')

    cat > "$dest/recreate.sh" << RECREATEEOF
#!/bin/bash
# 一键重建: $container
cd "\$(dirname "\$0")"

IMG="$img"
CONTAINER="$container"

echo "正在重建容器 \$CONTAINER ..."

# 重建命令（根据实际需要调整）
docker run -dit \\
  --name \$CONTAINER \\
  --hostname $hostname \\
  --restart ${restart:-unless-stopped} \\
  $(echo "$entrypoint" | grep -q null || echo "  --entrypoint '$entrypoint' \\")
  $(for vol in $volumes; do [ -n "$vol" ] && echo "  -v ${vol}_restored:/data \\"; done)
  $(for pm in $port_maps; do [ -n "$pm" ] && echo "  -p $pm \\"; done)
  \$IMG

echo "导入 volume 数据..."
for tarfile in volume-*.tar.gz; do
    vol_name=\${tarfile#volume-}
    vol_name=\${vol_name%.tar.gz}
    echo "  恢复卷: \$vol_name"
    docker run --rm -v "\${vol_name}_restored":/data -v "\$(pwd)":/restore alpine \\
      tar xzf "/restore/\$tarfile" -C /data
done

echo "✅ \$CONTAINER 重建完成！"
echo "如需导入镜像（离线环境）："
echo "   docker load < ../_shared_images/${safe_name}.tar.gz"
RECREATEEOF
    chmod +x "$dest/recreate.sh"
}

# ---------- 主流程 ----------
for target in "${BACKUP_TARGETS[@]}"; do
    log "══════════════════════════════════"
    log "备份到：$target"
    log "══════════════════════════════════"
    for container in "${CONTAINERS[@]}"; do
        backup_container "$container" "$target"
    done
    # 生成总览
    cat > "$target/_snapshot_info.txt" << EOF
备份时间: $BACKUP_DATE
容器: ${CONTAINERS[*]}
恢复: 进入各容器目录执行 recreate.sh
EOF
    log "✅ $target 全部完成"
done

log "🎉 所有容器备份完毕！"
```

### 2. 修改配置

编辑脚本顶部的三个变量：

```bash
# 备份到哪些目录（支持多块硬盘）
BACKUP_TARGETS=(
    "/mnt/disk1/backups"
    "/mnt/disk2/backups"
)

# 要备份的容器
CONTAINERS=(
    "nginx-web"
    "postgres-db"
    "my-app"
)

# 大镜像预导出（避免每周重复导出）
SHARED_IMAGES=(
    "nousresearch/hermes-agent:latest"
)
```

### 3. 首次运行

```bash
chmod +x /opt/scripts/docker-backup.sh
bash /opt/scripts/docker-backup.sh
```

首次运行会：
1. 导出大镜像（几分钟，仅在首次）
2. 对每个容器：生成 inspect.json + volume 压缩包 + recreate.sh
3. 同步到所有目标目录

---

## 定时任务设置

### 方案 A：使用 cronjob 工具

```yaml
# 在 Hermes Agent 中执行：
cronjob action=create
  name=docker-weekly-backup
  schedule="0 5 * * 0"        # 每周日 05:00
  prompt="运行 /opt/scripts/docker-backup.sh，检查备份结果"
  enabled_toolsets=["terminal","file"]
```

### 方案 B：系统 crontab

```bash
# 编辑 crontab
crontab -e

# 添加
0 5 * * 0 /bin/bash /opt/scripts/docker-backup.sh >> /var/log/docker-backup.log 2>&1
```

---

## 输出结构

```
/mnt/disk1/backups/
├── _shared_images/                ← 镜像缓存（仅首次导出）
│   └── nousresearch_hermes-agent_latest.tar.gz
├── _snapshot_info.txt             ← 备份总览
├── container-a/
│   ├── container-inspect.json     ← 完整容器配置
│   ├── volume-data.tar.gz         ← volume 数据
│   ├── bind-mounts.txt            ← bind mount 记录
│   └── recreate.sh                ← 一键重建
├── container-b/
└── ...
```

---

## 恢复流程

### 完整恢复（按需）

```bash
# 第1步：导入镜像（如果离线或镜像被删）
docker load < backups/_shared_images/nousresearch_hermes-agent_latest.tar.gz

# 第2步：一键重建
cd backups/container-a
bash recreate.sh

# 第3步：验证
docker ps | grep container-a
```

### bind mount 特殊处理

bind mount 指向宿主机目录（如 `/mnt/data`），recreate.sh 不自动处理。恢复后手动挂载即可：

```bash
# 从 bind-mounts.txt 查看原始挂载信息
cat backups/container-a/bind-mounts.txt

# 示例输出：
# /mnt/data → /app/data
# 恢复时手动挂载：
docker run -v /mnt/data:/app/data ...
```

---

## 踩坑记录

### 镜像导出很慢
- 首次导出大镜像（8G+）需要几分钟，耐心等待
- 后续备份跳过，因为 `_shared_images` 已有缓存

### volume 名冲突
- recreate.sh 会自动添加 `_restored` 后缀避免与现有容器冲突
- 恢复后如果容器名已存在，手动删除旧容器或用 `docker rename`

### 匿名 volume 找不到名字
- 用 `docker inspect <容器名>` 查看 Mounts 中的 Name 字段
- 也可以用 `docker run --rm -v <容器名>:/data...` 的方式通过容器名引用

### bind mount 数据不备份
- bind mount 可能是大数据（视频、数据库文件等），不适合打包进 tar.gz
- 脚本只记录挂载路径到 bind-mounts.txt，不做数据备份
- 如果 bind mount 下的配置也需要备份，手动复制相关文件
