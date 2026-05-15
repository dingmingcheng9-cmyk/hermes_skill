---
name: feishu-file-delivery
description: 通过飞书开放平台API发送文件到飞书聊天/频道 — 三步认证+上传+发送，解决内置MEDIA附件不支持飞书的问题。通用版：自动探测当前环境的飞书Bot凭据，适配任何Hermes智能体容器。
tags: [feishu, lark, file-transfer, api, send-file, upload, universal]
---

# 飞书文件发送技能 (Feishu File Delivery)

## 背景

**内置 `send_message` 工具的局限：** 飞书平台不支持 `MEDIA:/path` 附件，`send_message(target="feishu:...", message="MEDIA:/path")` 只会发送纯文本。要发送文件（zip、图片、报告、PPT等），必须直接调用飞书开放平台API。

## 何时使用

- 用户要求把文件发送到飞书（PPT、zip压缩包、报告文档、图片等）
- 需要发送的不仅仅是文字消息，而是二进制文件附件
- 打包好交付物后，最后一步分发给用户
- 不确定当前环境用哪套飞书凭据（脚本会自动探测）

## 通用凭据探测机制

本技能的关键改进：**任何 Hermes 智能体无需手动指定凭据**，脚本按优先级自动探测：

```
探测优先级：
① 当前进程环境变量（env）→ FEISHU_APP_ID, FEISHU_APP_SECRET
② 本机 ~/.hermes/ 配置文件（Host 级 agent）
③ docker exec hermes-job cat /opt/data/.env
④ docker exec hermes_new cat /opt/data/.env  
⑤ docker exec <任意 hermes-agent 容器> cat /opt/data/.env
⑥ 显式传入的 --app-id / --app-secret（手动覆盖）
```

这样无论你在哪个容器里、或者在宿主机上跑，脚本都能自己找到对的凭据。

## 发送目标自动匹配

| 场景 | 目标 chat_id | 说明 |
|:----|:-------------|:------|
| 当前对话 | `--to current` | 发回当前飞书对话（需 agent 运行时传入自己的 chat_id） |
| 家频道 | `--to home` | 发到 bot 配置的 FEISHU_HOME_CHANNEL |
| 指定频道 | `--chat-id oc_xxx` | 发到任意指定频道 |
| 指定用户 | `--open-id ou_xxx` | 发给指定用户（DM） |

> **特别注意：同一台机器上可能有多个飞书 Bot（如本例有两套），不同的 Bot 在不同的频道中。脚本会自动探测当前环境的 Bot，但你也要确保目标 chat_id 对应该 Bot。**

## 快速使用（一行命令）

```bash
# 自动探测凭据，发送到 bot 的家频道
python3 send_feishu_file.py /path/to/file.pptx --to home

# 发送到指定频道（手动指定 chat_id）
python3 send_feishu_file.py /path/to/file.pptx --chat-id oc_xxx

# 从 host 发给 hermes-job 容器对应的 bot 家频道
python3 send_feishu_file.py /path/to/file.zip --container hermes-job --to home
```

## Python 模块导入用法

```python
from send_feishu_file import FeishuFileSender

# 自动探测凭据，发送到 bot 的家频道
sender = FeishuFileSender()
sender.send_to_home("/path/to/file.pptx")

# 或者指定凭据与目标
sender = FeishuFileSender(app_id="cli_xxx", app_secret="xxx")
sender.send_to_chat("/path/to/file.pdf", chat_id="oc_xxx")
sender.send_to_user("/path/to/img.png", open_id="ou_xxx")
```

## API 参数详解

### 获取 Token

| 参数 | 说明 |
|------|------|
| `app_id` | 飞书开放平台 → 应用凭证 → App ID |
| `app_secret` | 飞书开放平台 → 应用凭证 → App Secret |

返回: `{"code": 0, "tenant_access_token": "xxx", "expire": 7200}`

### 文件上传

| 参数 | 值 | 说明 |
|------|-----|------|
| `file_type` | `stream` | 通用文件（文档/zip） |
| | `image` | 图片 |
| | `audio` | 语音 |
| | `media` | 视频 |
| | `file` | 文档类（pdf/doc） |
| `file_name` | 字符串 | 显示在飞书中的文件名 |
| `file` (multipart) | 二进制流 | 文件本体 |

返回: `{"code": 0, "data": {"file_key": "xxx"}}`

### 发送消息

| 参数 | 说明 |
|------|------|
| `receive_id` | chat_id（群聊/频道）或 open_id（用户） |
| `receive_id_type` | `chat_id` 或 `open_id` |
| `msg_type` | `file` 对应文件, `image` 对应图片 |
| `content` | `{"file_key": "xxx"}` 的JSON字符串 |

## 核心脚本

将以下内容保存为 `send_feishu_file.py`：

```python
#!/usr/bin/env python3
"""
send_feishu_file.py - 通用飞书文件发送工具

核心特性：
  - 自动探测飞书Bot凭据（env → host配置 → Docker容器）
  - 支持发送到家频道、指定频道、指定用户
  - 既可以作为CLI脚本使用，也可以作为Python模块导入
  - 适配任何Hermes智能体（host级、容器级均可）
"""

import os
import sys
import json
import subprocess
from pathlib import Path
from typing import Optional

import requests

# ─── 常量 ───
BASE_URL = os.environ.get("FEISHU_BASE_URL", "https://open.feishu.cn")
HERMES_CONTAINERS = ["hermes-job", "hermes_new", "hermes-ppt", "hermes-new"]

TYPE_MAP = {
    ".png": "image", ".jpg": "image", ".jpeg": "image",
    ".webp": "image", ".gif": "image",
    ".mp3": "audio", ".wav": "audio", ".ogg": "audio",
    ".mp4": "media", ".mov": "media", ".avi": "media", ".mkv": "media",
    ".pdf": "file", ".doc": "file", ".docx": "file",
}


# ─── 凭据探测 ───

def _probe_env() -> dict:
    creds = {}
    for key in ["FEISHU_APP_ID", "FEISHU_APP_SECRET", "FEISHU_HOME_CHANNEL",
                 "FEISHU_BASE_URL", "FEISHU_DOMAIN"]:
        val = os.environ.get(key)
        if val:
            creds[key] = val
    if "FEISHU_DOMAIN" in creds and "FEISHU_BASE_URL" not in creds:
        creds["FEISHU_BASE_URL"] = "https://open.larksuite.com" if creds["FEISHU_DOMAIN"] == "lark" else "https://open.feishu.cn"
    return creds


def _probe_host_env_file() -> dict:
    candidates = [
        os.path.expanduser("~/.hermes/.env"),
        os.path.expanduser("~/.hermes/config.yaml"),
    ]
    creds = {}
    for path in candidates:
        if not os.path.exists(path):
            continue
        try:
            with open(path) as f:
                content = f.read()
            for line in content.splitlines():
                line = line.strip()
                if line.startswith("#") or "=" not in line:
                    continue
                k, v = line.split("=", 1)
                k = k.strip()
                v = v.strip().strip("\"'")
                if k in ("FEISHU_APP_ID", "FEISHU_APP_SECRET", "FEISHU_HOME_CHANNEL",
                         "FEISHU_BASE_URL", "FEISHU_DOMAIN"):
                    creds[k] = v
        except (OSError, IOError):
            continue
    if "FEISHU_DOMAIN" in creds and "FEISHU_BASE_URL" not in creds:
        creds["FEISHU_BASE_URL"] = "https://open.larksuite.com" if creds["FEISHU_DOMAIN"] == "lark" else "https://open.feishu.cn"
    return creds


def _probe_docker_container(container_name: str) -> dict:
    creds = {}
    try:
        result = subprocess.run(
            ["docker", "exec", container_name, "cat", "/opt/data/.env"],
            capture_output=True, text=True, timeout=10,
        )
        if result.returncode != 0:
            return creds
        for line in result.stdout.splitlines():
            line = line.strip()
            if line.startswith("#") or "=" not in line:
                continue
            k, v = line.split("=", 1)
            k = k.strip()
            v = v.strip().strip("\"'")
            if k in ("FEISHU_APP_ID", "FEISHU_APP_SECRET", "FEISHU_HOME_CHANNEL",
                     "FEISHU_BASE_URL", "FEISHU_DOMAIN"):
                creds[k] = v
        if "FEISHU_DOMAIN" in creds and "FEISHU_BASE_URL" not in creds:
            creds["FEISHU_BASE_URL"] = "https://open.larksuite.com" if creds["FEISHU_DOMAIN"] == "lark" else "https://open.feishu.cn"
    except (subprocess.TimeoutExpired, FileNotFoundError, OSError):
        pass
    return creds


def _list_running_hermes_containers() -> list[str]:
    containers = []
    try:
        result = subprocess.run(
            ["docker", "ps", "--format", "{{.Names}}", "--filter", "ancestor=nousresearch/hermes-agent"],
            capture_output=True, text=True, timeout=10,
        )
        if result.returncode == 0:
            containers = [name.strip() for name in result.stdout.splitlines() if name.strip()]
    except (subprocess.TimeoutExpired, FileNotFoundError):
        pass
    return containers


def detect_credentials(container_hint: Optional[str] = None) -> tuple:
    """
    按优先级探测飞书Bot凭据。
    返回: (app_id, app_secret, home_channel, base_url, source_name)
    """
    base_url = BASE_URL

    creds = _probe_env()
    if creds.get("FEISHU_APP_ID") and creds.get("FEISHU_APP_SECRET"):
        return (creds["FEISHU_APP_ID"], creds["FEISHU_APP_SECRET"],
                creds.get("FEISHU_HOME_CHANNEL", ""),
                creds.get("FEISHU_BASE_URL", base_url), "env")

    creds = _probe_host_env_file()
    if creds.get("FEISHU_APP_ID") and creds.get("FEISHU_APP_SECRET"):
        return (creds["FEISHU_APP_ID"], creds["FEISHU_APP_SECRET"],
                creds.get("FEISHU_HOME_CHANNEL", ""),
                creds.get("FEISHU_BASE_URL", base_url), "host_env_file")

    if container_hint:
        check_containers = [container_hint]
    else:
        check_containers = list(dict.fromkeys(
            HERMES_CONTAINERS + _list_running_hermes_containers()
        ))

    for container in check_containers:
        creds = _probe_docker_container(container)
        if creds.get("FEISHU_APP_ID") and creds.get("FEISHU_APP_SECRET"):
            return (creds["FEISHU_APP_ID"], creds["FEISHU_APP_SECRET"],
                    creds.get("FEISHU_HOME_CHANNEL", ""),
                    creds.get("FEISHU_BASE_URL", base_url),
                    f"docker:{container}")

    raise RuntimeError(
        "无法自动探测飞书 Bot 凭据。请设置 FEISHU_APP_ID / FEISHU_APP_SECRET"
    )


# ─── 核心 API ───

def _get_tenant_token(app_id: str, app_secret: str, base_url: str) -> str:
    r = requests.post(f"{base_url}/open-apis/auth/v3/tenant_access_token/internal",
                      json={"app_id": app_id, "app_secret": app_secret})
    data = r.json()
    assert data.get("code") == 0, f"Token失败: {data}"
    return data["tenant_access_token"]


def _upload_file(token: str, file_path: str, base_url: str) -> str:
    file_name = Path(file_path).name
    suffix = Path(file_path).suffix.lower()
    file_type = TYPE_MAP.get(suffix, "stream")
    with open(file_path, "rb") as f:
        r = requests.post(
            f"{base_url}/open-apis/im/v1/files",
            headers={"Authorization": f"Bearer {token}"},
            data={"file_type": file_type, "file_name": file_name},
            files={"file": (file_name, f, "application/octet-stream")},
        )
    data = r.json()
    assert data.get("code") == 0, f"上传失败: {data}"
    return data["data"]["file_key"]


def _send_message(token: str, receive_id: str, id_type: str, file_key: str, base_url: str):
    msg_content = json.dumps({"file_key": file_key})
    r = requests.post(
        f"{base_url}/open-apis/im/v1/messages?receive_id_type={id_type}",
        headers={"Authorization": f"Bearer {token}", "Content-Type": "application/json"},
        json={"receive_id": receive_id, "msg_type": "file", "content": msg_content},
    )
    data = r.json()
    assert data.get("code") == 0, f"发送失败: {data}"
    return data


# ─── 高级 API ───

class FeishuFileSender:
    """飞书文件发送器（自动探测凭据）"""

    def __init__(self, app_id: str = None, app_secret: str = None,
                 home_channel: str = None, base_url: str = None,
                 container_hint: str = None):
        if app_id and app_secret:
            self.app_id = app_id
            self.app_secret = app_secret
            self.home_channel = home_channel or ""
            self.base_url = base_url or BASE_URL
            self.source_name = "manual"
        else:
            detected = detect_credentials(container_hint=container_hint)
            self.app_id = detected[0]
            self.app_secret = detected[1]
            self.home_channel = detected[2]
            self.base_url = detected[3]
            self.source_name = detected[4]

    def send_to_home(self, file_path: str, file_name: str = None, verbose: bool = True) -> str:
        if not self.home_channel:
            raise ValueError("未配置 home_channel")
        token = _get_tenant_token(self.app_id, self.app_secret, self.base_url)
        fname = file_name or Path(file_path).name
        file_key = _upload_file(token, file_path, self.base_url)
        if verbose:
            print(f"  📤 {fname} ({self.source_name})")
        result = _send_message(token, self.home_channel, "chat_id", file_key, self.base_url)
        return result["data"]["message_id"]

    def send_to_chat(self, file_path: str, chat_id: str, file_name: str = None, verbose: bool = True) -> str:
        token = _get_tenant_token(self.app_id, self.app_secret, self.base_url)
        fname = file_name or Path(file_path).name
        file_key = _upload_file(token, file_path, self.base_url)
        if verbose:
            print(f"  📤 {fname} ({self.source_name})")
        result = _send_message(token, chat_id, "chat_id", file_key, self.base_url)
        return result["data"]["message_id"]

    def send_to_user(self, file_path: str, open_id: str, file_name: str = None, verbose: bool = True) -> str:
        token = _get_tenant_token(self.app_id, self.app_secret, self.base_url)
        fname = file_name or Path(file_path).name
        file_key = _upload_file(token, file_path, self.base_url)
        if verbose:
            print(f"  📤 {fname} ({self.source_name})")
        result = _send_message(token, open_id, "open_id", file_key, self.base_url)
        return result["data"]["message_id"]


# ─── CLI 入口 ───

if __name__ == "__main__":
    import argparse
    parser = argparse.ArgumentParser(description="发送文件到飞书")
    parser.add_argument("file_path", help="文件路径")
    parser.add_argument("--to", choices=["home"], help="快捷目标")
    parser.add_argument("--chat-id", help="目标chat_id")
    parser.add_argument("--open-id", help="目标open_id")
    parser.add_argument("--app-id", help="飞书App ID")
    parser.add_argument("--app-secret", help="飞书App Secret")
    parser.add_argument("--container", help="指定Docker容器探测凭据")
    args = parser.parse_args()

    if not os.path.isfile(args.file_path):
        print(f"❌ 文件不存在: {args.file_path}")
        sys.exit(1)

    try:
        if args.app_id and args.app_secret:
            sender = FeishuFileSender(app_id=args.app_id, app_secret=args.app_secret)
        else:
            sender = FeishuFileSender(container_hint=args.container)

        if args.to == "home":
            sender.send_to_home(args.file_path)
        elif args.chat_id:
            sender.send_to_chat(args.file_path, chat_id=args.chat_id)
        elif args.open_id:
            sender.send_to_user(args.file_path, open_id=args.open_id)
        else:
            print("❌ 请指定 --to home / --chat-id / --open-id")
            sys.exit(1)

        print(f"✅ {Path(args.file_path).name} 已发送到飞书")
    except Exception as e:
        print(f"❌ 失败: {e}")
        sys.exit(1)
```

## 注意事项

1. **Token有效期2小时** — 长时间会话需要重新获取
2. **文件大小限制** — `stream` 类型最大 30MB
3. **Bot必须在目标群** — 否则返回 230002 permission denied
4. **国际站区别** — `FEISHU_DOMAIN=lark` 时用 `https://open.larksuite.com`
5. **二进制读取** — 上传文件必须用 `rb` 模式读取
6. **两套Bot凭据** — 同一台机器可能有多个飞书Bot，脚本会自动探测但需确保目标 chat_id 对应该 Bot
7. **execute_code 沙箱不继承 env** — 在 Hermes agent 的 execute_code 中调 API 时，需用 terminal 执行或手动 source 配置
