---
name: hermes-zombie-cleanup
description: "自动清理 Hermes Agent 僵尸进程——接管 Hermes 自身的 session_reset 兜底，杀掉因 bug 残留超过26小时的 agent 进程。覆盖宿主机 + 所有 Docker 容器，配合 cron 静默执行，有清理时通知用户。"
version: 1.0.0
author: Hermes Agent
metadata:
  hermes:
    tags: [zombie, cleanup, process, memory, maintenance, cron]
    related_skills: []
---

# Hermes Zombie Process Cleanup

自动清理 Hermes Agent 僵尸进程的通用方案。核心是一套自包含的 Python 脚本，扫描宿主机和 Docker 容器内的 hermes 进程，识别并杀死因 bug 残留的超龄 agent，配合 Hermes cron job 每日定时执行，有清理时通知用户。

## 问题背景

Hermes Agent 每天凌晨 4:00 通过 `session_reset` 清理 idle 超过 24h 的会话。但某些场景（cron 任务异常退出、手动 spawn 的 agent 未回收、网关 bug）会导致 agent 进程变成僵尸，**不会被 session_reset 清理**，白白占用内存。

## Trigger Conditions

使用此技能当用户：
- 发现 `docker stats` 显示 hermes 容器内存异常偏高
- `ps aux | grep hermes` 看到多个存活超过 1 天的 agent 进程
- 想在新机器上部署僵尸清理机制
- 容器运行多天后内存只涨不降

## 架构概览

```
┌────────────────────────────────────────────┐
│  cronjob (30 4 * * *)                       │
│  每天凌晨4:30，比session_reset晚半小时      │
└──────────┬─────────────────────────────────┘
           │ 执行
┌──────────▼─────────────────────────────────┐
│  /opt/scripts/hermes-zombie-cleanup.py      │
│  ① 扫描宿主机 ps hxo pid,etime,args        │
│  ② 扫描所有 hermes-agent 镜像的容器        │
│  ③ 过滤条件：                              │
│     • 是 hermes 进程                        │
│     • 不是 gateway run                      │
│     • 存活超过 26 小时                      │
│  ④ 先 SIGTERM → 等 3s → 没死 SIGKILL      │
│  ⑤ 输出 NEEDS_ALERT=true/false             │
└──────────┬─────────────────────────────────┘
           │ 输出
┌──────────▼─────────────────────────────────┐
│  cron job agent 捕获输出                    │
│  if NEEDS_ALERT=true → send_message         │
│  else → 静默结束                            │
└────────────────────────────────────────────┘
```

## 安全识别逻辑（核心）

**三条件同时满足才杀，少一个不杀：**

| # | 条件 | 命令 | 原因 |
|:-:|:----|:----|:------|
| ① | 是 hermes 进程 | `ps args 含 hermes` | 只杀 hermes，不动其他进程 |
| ② | 不是 gateway | `args 不含 gateway run` | gateway 是常驻入口，绝不能杀 |
| ③ | 存活 > 26 小时 | `etime > 26h` | Hermes 的 idle 超时是 24h，加 2h 安全缓冲 |

**为什么不看 CPU 空闲？**
- 合法对话 agent 在等下一句时也 idle，不能杀
- 凌晨 3:59 的 agent 等 4:00 重置，也不能杀
- 只有**超龄未退**才是真正的 bug 残留

## 脚本部署

### 1. 脚本位置

`/opt/scripts/hermes-zombie-cleanup.py` — 完整 Python 脚本，见下文。

### 2. 可配置常量

```python
STALE_SECONDS = 26 * 3600       # 判定为僵尸的存活阈值
HERMES_IMAGE_FILTER = "nousresearch/hermes-agent"  # Docker 镜像过滤
SIGTERM_WAIT = 3                # SIGTERM 后等待秒数
DRY_RUN = os.environ.get("DRY_RUN", "0") == "1"   # DRY_RUN=1 只扫描不杀
```

### 3. 脚本内容

```python
#!/usr/bin/env python3
"""
Hermes Zombie Process Cleanup
==============================
Kills agent processes that survived beyond the 24h idle limit
(proving they leaked due to a bug). Runs every night at 04:30,
30 minutes after Hermes' own session reset, acting as a safety net.

Safety guard: only kills processes with ELAPSED > 26 hours AND
whose command line matches `hermes` but NOT `gateway run`.

Covers: host-level agents + all Docker containers running hermes-agent images.
"""

import subprocess
import re
import time
import json
import logging
import os
import sys

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
log = logging.getLogger("hermes-zombie-cleanup")

STALE_SECONDS = 26 * 3600  # 26 hours
HERMES_IMAGE_FILTER = "nousresearch/hermes-agent"
SIGTERM_WAIT = 3  # seconds between SIGTERM and SIGKILL
DRY_RUN = os.environ.get("DRY_RUN", "0") == "1"


def parse_etime(etime_str: str) -> int | None:
    """Parse ps etime format (e.g. '5-00:34:20', '12:34:56', '34:56') into seconds."""
    etime_str = etime_str.strip()
    try:
        if "-" in etime_str:
            days, rest = etime_str.split("-", 1)
            days = int(days)
        else:
            days = 0
            rest = etime_str
        parts = [int(x) for x in rest.split(":")]
        if len(parts) == 3:
            hours, mins, secs = parts
        elif len(parts) == 2:
            hours = 0
            mins, secs = parts
        else:
            return None
        return days * 86400 + hours * 3600 + mins * 60 + secs
    except (ValueError, TypeError):
        return None


def scan_host_processes() -> list[dict]:
    """Scan host-level processes for stale hermes agents."""
    try:
        result = subprocess.run(
            ["ps", "hxo", "pid:1,etime:50,args:"],
            capture_output=True,
            text=True,
            timeout=10,
        )
    except subprocess.TimeoutExpired:
        log.warning("host ps timed out")
        return []

    stale = []
    for line in result.stdout.splitlines():
        line = line.strip()
        if not line:
            continue
        parts = line.split(None, 2)
        if len(parts) < 3:
            continue
        pid_str, etime_str, args = parts

        # Conditions ① & ②: must be hermes process, not gateway
        if "hermes" not in args:
            continue
        if "gateway run" in args:
            continue

        elapsed = parse_etime(etime_str)
        if elapsed is None:
            continue

        pid = int(pid_str)
        # Condition ③: older than 26 hours
        if elapsed > STALE_SECONDS:
            stale.append({"pid": pid, "elapsed_s": elapsed, "args": args[:120], "source": "host", "container": None})

    return stale


def scan_docker_containers() -> list[dict]:
    """Scan all docker containers running hermes-agent images."""
    try:
        result = subprocess.run(
            ["docker", "ps", "--format", "{{.ID}}\t{{.Names}}\t{{.Image}}"],
            capture_output=True,
            text=True,
            timeout=15,
        )
    except subprocess.TimeoutExpired:
        log.warning("docker ps timed out")
        return []

    stale = []
    for line in result.stdout.splitlines():
        line = line.strip()
        if not line:
            continue
        cid, name, image = line.split("\t", 2)
        if HERMES_IMAGE_FILTER not in image:
            continue
        if not name:
            name = cid[:12]

        try:
            ps_result = subprocess.run(
                ["docker", "exec", name, "ps", "hxo", "pid:1,etime:50,args:"],
                capture_output=True,
                text=True,
                timeout=15,
            )
        except subprocess.TimeoutExpired:
            log.warning("docker exec ps timed out for container %s", name)
            continue

        for ps_line in ps_result.stdout.splitlines():
            ps_line = ps_line.strip()
            if not ps_line:
                continue
            parts = ps_line.split(None, 2)
            if len(parts) < 3:
                continue
            pid_str, etime_str, args = parts

            if "hermes" not in args:
                continue
            if "gateway run" in args:
                continue

            elapsed = parse_etime(etime_str)
            if elapsed is None:
                continue

            pid = int(pid_str)
            if elapsed > STALE_SECONDS:
                stale.append({
                    "pid": pid,
                    "elapsed_s": elapsed,
                    "args": args[:120],
                    "source": f"docker:{name}",
                    "container": name,
                })

    return stale


def kill_process(entry: dict) -> bool:
    """Kill a stale process. Returns True if killed, False if already gone."""
    pid = entry["pid"]
    source = entry["source"]
    container = entry["container"]

    def run_kill(cmd: list[str], signal_name: str) -> bool:
        try:
            if container:
                full_cmd = ["docker", "exec", container] + cmd
            else:
                full_cmd = cmd
            r = subprocess.run(full_cmd, capture_output=True, text=True, timeout=10)
            return r.returncode == 0
        except subprocess.TimeoutExpired:
            return False

    if DRY_RUN:
        log.info("[DRY-RUN] Would kill PID %s in %s", pid, source)
        return True

    # Try SIGTERM first
    log.info("SIGTERM → PID %s (%s)", pid, source)
    killed = run_kill(["kill", str(pid)], "SIGTERM")
    time.sleep(SIGTERM_WAIT)

    # Check if still alive
    if container:
        check_cmd = ["docker", "exec", container, "kill", "-0", str(pid)]
    else:
        check_cmd = ["kill", "-0", str(pid)]

    still_alive = True
    try:
        r = subprocess.run(check_cmd, capture_output=True, timeout=5)
        still_alive = r.returncode == 0
    except subprocess.TimeoutExpired:
        still_alive = True

    if still_alive:
        log.warning("SIGKILL → PID %s (%s) did not respond to SIGTERM", pid, source)
        run_kill(["kill", "-9", str(pid)], "SIGKILL")
        return True

    log.info("  └→ PID %s exited cleanly", pid)
    return True


def main():
    log.info("=== Hermes Zombie Cleanup ===")
    if DRY_RUN:
        log.info("DRY RUN — no processes will be killed")

    host_stale = scan_host_processes()
    docker_stale = scan_docker_containers()
    all_stale = host_stale + docker_stale

    if not all_stale:
        log.info("✅ No stale Hermes agents found (alive < %dh)", STALE_SECONDS // 3600)
        print("===REPORT_START===")
        print("🧹 Hermes 僵尸进程清理报告")
        print("✅ 无僵尸进程")
        print("===REPORT_END===")
        print("NEEDS_ALERT=false")
        return

    log.info("Found %d stale agent(s):", len(all_stale))
    for entry in all_stale:
        hours = entry["elapsed_s"] / 3600
        log.info("  PID %-6s %-20s %5.1fh  %s", entry["pid"], entry["source"], hours, entry["args"])

    killed_count = 0
    for entry in all_stale:
        if kill_process(entry):
            killed_count += 1

    log.info("Done: %d/%d stale agents killed", killed_count, len(all_stale))

    # Structured output for cron job parsing
    print("===REPORT_START===")
    print("🧹 Hermes 僵尸进程清理报告")
    for entry in all_stale:
        hours = entry["elapsed_s"] / 3600
        print(f"  PID {entry['pid']} ({entry['source']}) — 存活 {hours:.1f}h — {entry['args']}")
    print(f"📊 共发现 {len(all_stale)} 个僵尸进程，已清理 {killed_count} 个")
    print("===REPORT_END===")
    print(f"NEEDS_ALERT={'true' if killed_count > 0 else 'false'}")

    sys.exit(0 if killed_count == len(all_stale) else 1)


if __name__ == "__main__":
    main()
```

## 定时任务配置

通过 Hermes cronjob 工具创建，**不要**用系统 crontab：

```yaml
name: hermes-zombie-cleanup
schedule: "30 4 * * *"             # 每天凌晨 4:30
enabled_toolsets: [terminal]       # 只需终端执行脚本
deliver: origin                    # 有清理时通知用户
```

**cron job 的 prompt：**

```
每天执行 Hermes 僵尸进程清理任务。执行以下步骤：
1. 运行脚本: /opt/scripts/hermes-zombie-cleanup.py
2. 捕获输出，找到 NEEDS_ALERT=true 或 NEEDS_ALERT=false
3. 解析 ===REPORT_START=== 和 ===REPORT_END=== 之间的报告内容
4. 如果 NEEDS_ALERT=true：
   - 用 send_message 发送清理报告，含被杀的僵尸进程详情
5. 如果 NEEDS_ALERT=false：
   - 无僵尸，静默结束，不发消息
```

## 前置条件

- Python 3（系统自带即可，无需额外包）
- `ps` 命令（所有 Linux 都有）
- `docker` 命令（如果需要扫描容器内的进程）
- Hermes cronjob 系统（用于定时执行）

## 移植到其他机器

1. 复制脚本到目标机器的 `/opt/scripts/hermes-zombie-cleanup.py`
2. 如需调整僵尸判定阈值，修改 `STALE_SECONDS`（默认 26h）
3. 如需调整 Docker 镜像匹配规则，修改 `HERMES_IMAGE_FILTER`
4. 测试运行：`DRY_RUN=1 python3 /opt/scripts/hermes-zombie-cleanup.py`
5. 确认无误后创建 cron job（prompt 见上）
6. 手动触发一次验证：`cronjob action=run job_id=<id>`

## 验证

```bash
# 干跑测试（不杀进程）
sudo DRY_RUN=1 python3 /opt/scripts/hermes-zombie-cleanup.py

# 手动创建一个僵尸 agent 来测试（可选）
# 在一个终端运行：hermes agent run --session test-zombie &
# 等 26 小时...（或者临时改 STALE_SECONDS=1 来验证逻辑）

# 查看 cron job 状态
cronjob action=list
# 找到 hermes-zombie-cleanup，确认 enabled=true, next_run_at 正确
```

## Pitfalls

- **不要用系统 crontab**：使用 Hermes 的 cronjob 工具管理，这样才能访问 send_message 能力来发送通知
- **SIGTERM 优先，SIGKILL 兜底**：有些进程需优雅退出（释放锁、写日志），先给 3s 缓冲
- **Docker 内进程杀不掉的情况**：如果 `docker exec` 没有权限，容器可能需要用 `--privileged` 或在宿主机上用 `nsenter` 杀。但 Hermes 默认容器通常够用
- **26h 阈值不是随意定的**：Hermes 默认 idle 24h 重置，2h 缓冲确保不误杀凌晨 4:00 正在被 session_reset 清理的合法进程
- **首次部署时建议 DRY_RUN=1 跑一次**：确认不会误杀 gateway
- **脚本需要 sudo 才能扫描宿主机进程**：普通用户 `ps hxo` 只能看到自己的进程。如果 Hermes 以非 root 运行，需调整脚本或用 sudo
- **新增容器自动覆盖**：脚本通过 `docker ps --filter ancestor=hermes-agent` 自动发现，新增容器无需改配置

## 输出格式

无僵尸时（默认日常）：
```
===REPORT_START===
🧹 Hermes 僵尸进程清理报告
✅ 无僵尸进程
===REPORT_END===
NEEDS_ALERT=false
```

有僵尸时：
```
===REPORT_START===
🧹 Hermes 僵尸进程清理报告
  PID 1192 (docker:hermes-job) — 存活 108.5h — python3 /opt/hermes/.venv/bin/hermes
  PID 1215 (host) — 存活 96.2h — python3 /opt/hermes/.venv/bin/hermes
📊 共发现 2 个僵尸进程，已清理 2 个
===REPORT_END===
NEEDS_ALERT=true
```
