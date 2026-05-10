---
name: disk-health-monitor
description: "月度硬盘健康状态监控 + 异常自动告警。SMART 解析 + Python 脚本 + cron 定时任务，异常时通过网关即时通知用户。"
version: 1.0.0
author: Hermes Agent
metadata:
  hermes:
    tags: [disk, health, monitor, smart, hdd, cron, alert]
    related_skills: [storage-management]
---

# Disk Health Monitor

月度硬盘健康监控方案。核心是一套自包含的 Python 脚本，读取 SMART 属性、比对阈值、生成报告，配合 Hermes cron job 每月定时执行、异常时自动告警。

## Trigger Conditions

使用此技能当用户：
- 想对硬盘做定期的健康巡检
- 硬盘出现异常（掉盘、I/O 错误、读写变慢）
- 想将磁盘监控迁移到另一台机器
- 问"硬盘还能撑多久"、"有没有坏道"

## 架构概览

```
┌──────────────────────────────────────┐
│  cronjob (0 9 1 * *)                 │
│  每月1号9:00自动触发                  │
└──────────┬───────────────────────────┘
           │ 执行
┌──────────▼───────────────────────────┐
│  /opt/scripts/disk-health-check.py   │
│  ① smartctl -H -d sat → 整体健康     │
│  ② smartctl -A -d sat → 属性解析     │
│  ③ 阈值比对 → 生成报告                │
│  ④ 输出 NEEDS_ALERT=true/false       │
└──────────┬───────────────────────────┘
           │ 输出
┌──────────▼───────────────────────────┐
│  cron job agent 捕获输出              │
│  if NEEDS_ALERT=true → send_message  │
│  else → 静默结束（不发消息）           │
└──────────────────────────────────────┘
```

## 脚本部署

### 1. 脚本位置

`/opt/scripts/disk-health-check.py` — 完整 Python 脚本，见下文。

### 2. 脚本结构

脚本包含四个主要部分：

#### DRIVES 字典（需要自定义）
```python
DRIVES = {
    'sdc': {'model': '西数 WD1004FBYZ', 'label': '西数 1TB (sdc)', 'mount': '/mnt/hermes-ext'},
    'sdd': {'model': '东芝 DT01ACA100', 'label': '东芝 1TB (sdd)', 'mount': '/mnt/hermes-ext2'},
}
```
移植到其他机器时，修改设备名、标签和挂载点即可。

#### THRESHOLDS 阈值
```python
THRESHOLDS = {
    'Reallocated_Sector_Ct': {'warn': 1, 'critical': 10},      # 重映射扇区
    'Current_Pending_Sector': {'warn': 1, 'critical': 5},       # 待处理扇区
    'Offline_Uncorrectable': {'warn': 1, 'critical': 5},        # 离线不可纠正
    'Spin_Retry_Count': {'warn': 1, 'critical': 3},             # 电机重试
    'Reported_Uncorrectable': {'warn': 1, 'critical': 5},       # 不可纠正错误
    'Temperature_Celsius': {'warn': 50, 'critical': 60},        # 温度°C
}
```

#### 核心函数
- `get_smart_health(dev)` — 读取整体健康状态（PASSED/FAILED）
- `get_smart_attributes(dev)` — 解析 `smartctl -A` 输出，提取每个属性的 raw/normalized/threshold/worst
- `check_drive(dev, info)` — 单盘检查，返回 issues 列表 + healthy 布尔值
- `main()` — 遍历所有盘，生成报告，输出 NEEDS_ALERT

#### 输出格式
```
===REPORT_START===
📋 **月度硬盘健康检查报告**

**西数 1TB (sdc)**（挂载: /mnt/hermes-ext）
```
✅ **健康** — 1,066 小时, 所有属性正常
```

**东芝 1TB (sdd)**（挂载: /mnt/hermes-ext2）
```
✅ **健康** — 27,335 小时, 所有属性正常
```

**汇总**: 西数 1TB (sdc): ✅ 正常 | 东芝 1TB (sdd): ✅ 正常
**结论**: ✅ **所有硬盘正常**
===REPORT_END===
NEEDS_ALERT=false
```

当有异常时：
```
NEEDS_ALERT=true
```
脚本退出码：0=全健康，1=有异常。

## 定时任务配置

通过 Hermes cronjob 工具创建，**不要**用系统 crontab：

```yaml
name: disk-health-check
schedule: "0 9 1 * *"        # 每月1号 9:00 AM
enabled_toolsets: [terminal, web]
deliver: local                # 结果自动回当前会话
skills: []                    # 无需附加技能，脚本是自包含的
```

**cron job 的 prompt**（运行逻辑）：

```
每月硬盘健康检查任务。执行以下步骤：

1. 运行脚本: `/opt/scripts/disk-health-check.py`
2. 捕获输出，找到 `NEEDS_ALERT=true` 或 `NEEDS_ALERT=false`
3. 解析 `===REPORT_START===` 和 `===REPORT_END===` 之间的报告内容
4. 如果 `NEEDS_ALERT=true`：
   - 用 send_message 向用户发送告警（含完整报告）
   - 含两颗硬盘的异常信息和汇总
5. 如果 `NEEDS_ALERT=false`：
   - 静默结束，不发送任何消息
```

## 前置条件

- `smartmontools` 已安装：`apt install -y smartmontools`
- USB 硬盘需要 `-d sat` 参数（USB-SATA 桥接器）
- Python 3（系统自带即可，无需额外包）
- 如果监控机器上有代理（如 sing-box），cron 任务可能需要走代理发消息

## 移植到其他机器

1. 复制脚本到目标机器的 `/opt/scripts/disk-health-check.py`
2. 修改 `DRIVES` 字典中的设备名、标签、挂载点
3. 确认 `smartctl -d sat /dev/<dev>` 能正常工作
4. 确认 smartmontools 已安装
5. 用 cronjob 工具创建定时任务（prompt 如上）
6. 手动触发一次验证：`cronjob action=run job_id=<id>`

## 验证

```bash
# 手动运行脚本
sudo python3 /opt/scripts/disk-health-check.py

# 检查 smartctl 是否能读 USB 硬盘
sudo smartctl -H -d sat /dev/sdc
sudo smartctl -A -d sat /dev/sdc

# 检查 cron job 状态
cronjob action=list
# 找到 disk-health-check，确认 enabled=true, next_run_at 正确
```

## Pitfalls

- **USB 桥接必须加 -d sat**：不加会报 "device lacks capacity" 或 "Unknown USB bridge"。如果 `-d sat` 也不工作，尝试 `-d usbjmicron` 或 `-d usbprolific`
- **`smartctl` 需要 root/sudo**：cron job 中如果通过 Hermes terminal 运行，终端默认有 sudo 权限
- **SMART 属性名称因硬盘厂商而异**：希捷和西数的属性名可能不同（如 `Power_On_Hours` vs `Power_On_Hours_and_Msec`）。脚本已处理常见变体
- **温度传感器位置**：SMART 读取的是硬盘内部电路温度，不是外壳温度。双盘盒外壳更热是正常的
- **cron job 静默执行**：健康时不发消息是故意的。如果用户想看报告，可以手动触发一次
- **不要用系统 crontab**：使用 Hermes 的 cronjob 工具管理，这样可以访问 Hermes 的 send_message 能力来发送告警

## 与 storage-management 的关系

`storage-management` 技能覆盖更广（分区、格式化、挂载、NAS、USB 掉盘修复）。本技能只聚焦于**健康监控+告警**这一件事。两个技能互补：用 `storage-management` 设置和管理硬盘，用 `disk-health-monitor` 持续关注它们的健康状况。
