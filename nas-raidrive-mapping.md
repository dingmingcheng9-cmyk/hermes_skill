---
name: nas-raidrive-mapping
description: 通过 Alist (WebDAV) + Samba 将 NAS 硬盘映射到 Windows（RaiDrive / 网络驱动器），涵盖完整安装配置和踩坑点
---

# NAS 硬盘映射到 Windows 操作指南

## 适用场景

服务器上挂了 USB 机械硬盘（通过 ASM1352-PM USB 桥接），需要从 Windows 电脑访问。提供两种方案：

- **Alist WebDAV**（已有 Alist 的场景，开箱即用，但磁盘容量显示有问题）
- **Samba**（推荐，Windows 原生支持，容量准确）

---

## 一、环境信息

| 项目 | 值 |
|---|---|
| 服务器 IP | `192.168.31.187`（LAN） |
| 硬盘1 | sdc = 西数 WD1004FBYZ 1TB → `/mnt/hermes-ext` |
| 硬盘2 | sdd = 东芝 DT01ACA100 1TB → `/mnt/hermes-ext2` |
| USB桥接 | ASM1352-PM，SMART 需加 `-d sat` 参数 |

---

## 二、Alist 方案（WebDAV）

### 2.1 Docker Compose 配置

```yaml
# /mnt/hermes-ext/alist/docker-compose.yml
services:
  alist:
    image: xhofe/alist:latest
    container_name: alist
    restart: unless-stopped
    ports:
      - "5244:5244"
    volumes:
      - ./data:/opt/alist/data
      - ./storage:/storage
      - /mnt/hermes-ext:/mnt/hermes-ext
      - /mnt/hermes-ext2:/mnt/hermes-ext2
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
```

> 注意：必须把硬盘挂载点（`/mnt/hermes-ext`）也映射进容器，Alist 才能通过 Local 存储驱动访问。

### 2.2 启动

```bash
cd /mnt/hermes-ext/alist
docker compose up -d
```

### 2.3 添加存储（通过 Web UI）

1. 访问 `http://192.168.31.187:5244`
2. 管理后台 → 存储 → 添加
3. 驱动选择 **Local**，挂载路径填容器内路径（如 `/mnt/hermes-ext`）

### 2.4 重置密码

```bash
docker exec -w /opt/alist alist ./alist admin set <新密码>
```

### 2.5 RaiDrive 配置（WebDAV）

| 参数 | 值 |
|---|---|
| 协议 | **HTTP**（不要勾 HTTPS！） |
| 地址 | 192.168.31.187 |
| 端口 | 5244 |
| 路径 | `/dav/` |
| 账号 | admin |
| 密码 | Alist 后台密码 |

> ⚠️ **已知问题**：Alist 的 WebDAV 不实现 RFC 4331 配额属性（`quota-available-bytes`），RaiDrive 显示的磁盘容量是乱码（如 "7eb"）。不影响文件读写，只影响容量显示。如需准确容量 → 改用 Samba。

---

## 三、Samba 方案（推荐）

### 3.1 安装

```bash
apt-get update && apt-get install -y samba
```

### 3.2 配置 `/etc/samba/smb.conf`

```ini
[global]
   workgroup = WORKGROUP
   server string = Hermes NAS
   netbios name = HERMES-NAS
   security = user
   map to guest = bad user
   dns proxy = no
   follow symlinks = yes
   wide links = yes
   unix extensions = no
   bind interfaces only = yes
   interfaces = lo eth0 192.168.31.0/24

[hermes-ext]
   comment = Western Digital 1TB
   path = /mnt/hermes-ext
   browseable = yes
   read only = no
   guest ok = no
   valid users = nasuser
   force user = root
   force group = root
   create mask = 0777
   directory mask = 0777

[hermes-ext2]
   comment = Toshiba 1TB
   path = /mnt/hermes-ext2
   browseable = yes
   read only = no
   guest ok = no
   valid users = nasuser
   force user = root
   force group = root
   create mask = 0777
   directory mask = 0777
```

### 3.3 创建 Samba 用户

```bash
# 创建系统用户（仅 Samba 使用，不能登录）
useradd -M -s /usr/sbin/nologin nasuser

# 设置 Samba 密码
smbpasswd -a nasuser
```

### 3.4 启动

```bash
systemctl restart smbd
systemctl enable smbd

# 防火墙放行
ufw allow samba
```

### 3.5 验证

```bash
# 检查配置语法
testparm -s

# 检查服务状态
systemctl status smbd
```

### 3.6 Windows 映射网络驱动器

1. 打开「此电脑」→ 右键「映射网络驱动器」
2. 文件夹填 `\\192.168.31.187\hermes-ext`
3. 勾选「使用其他凭据连接」
4. 输入 `nasuser` / 设置的密码
5. 重复添加 `hermes-ext2`

映射后两个盘显示在「此电脑」里，容量和剩余空间正常。

---

## 四、踩坑记录

### Alist WebDAV 常见问题

| 现象 | 原因 | 解决 |
|---|---|---|
| RaiDrive 报 "ProtocolVersion" | 勾选 HTTPS 但服务器只有 HTTP | 选 HTTP，不要勾 HTTPS |
| 容量显示乱码（如 "7eb"） | Alist 未实现 WebDAV 配额属性 | 用 Samba 代替 |
| WebDAV 连上后 401 | 密码错误或路径不对 | 路径用 `/dav/` |

### Samba 注意事项

- `force user = root` 确保写入文件属主是 root，避免权限问题
- `create mask = 0777` 让 Windows 写入的文件在 Linux 下也有全权限
- Samba 用户（nasuser）是独立密码，跟系统用户密码不同
- 如果 Windows 提示 "找不到网络路径"，检查 ufw 是否放行 Samba（端口 445/tcp, 139/tcp）

---

## 五、RaiDrive vs 网络驱动器对比

| 项目 | RaiDrive + Alist WebDAV | Windows 网络驱动器 + Samba |
|---|---|---|
| 安装复杂度 | 只需配 Alist（已有则零成本） | 需装 Samba |
| 磁盘容量显示 | ❌ 乱码 | ✅ 准确 |
| 文件操作 | ✅ 正常 | ✅ 正常 |
| 速度 | 正常 | 正常 |
| 推荐场景 | 已有 Alist 临时用 | 日常使用（推荐） |
