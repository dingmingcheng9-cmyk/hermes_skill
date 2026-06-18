---
name: video-download-and-analyze
description: "Download videos from social platforms (Douyin, Bilibili, YouTube, etc.) via yt-dlp and analyze them with a vision-capable auxiliary model. / 使用 yt-dlp 从抖音、B站、YouTube 等平台下载视频，并用视觉模型分析其内容。"
version: 2.1.0
author: Hermes Agent (learned from session)
tags: [video, analysis, yt-dlp, vision-model, douyin, bilibili, youtube, video-understanding, setup]
---

# Video Download & Analysis / 视频下载与分析

Download videos from any social platform using `yt-dlp`, then analyze their content using the configured auxiliary vision model.
使用 `yt-dlp` 从各大社交平台下载视频，然后通过配置好的辅助视觉模型分析视频内容。

## Trigger / 触发条件

- User sends a video link and asks "你能看这个视频吗？" or similar
  用户发来一个视频链接并问"你能看这个视频吗？"或类似问题
- User asks you to analyze/summarize a video from Douyin, Bilibili, YouTube, or any platform yt-dlp supports
  用户要求你分析/总结某个视频（抖音、B站、YouTube 或任何 yt-dlp 支持的平台）
- Any task requiring understanding of video content
  任何需要理解视频内容的任务

---

## Prerequisites Setup / 前置配置

Before using this workflow, ensure the following are installed and configured.
首次使用本技能前，请确保以下工具和配置已就绪。

### 1. Install yt-dlp / 安装 yt-dlp

```bash
# Option A: Download standalone binary (preferred — no pip needed)
# 方案 A：下载独立二进制文件（推荐，无需 pip）
curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp
chmod +x /usr/local/bin/yt-dlp

# Option B: Via pip (if available)
# 方案 B：通过 pip 安装
pip install yt-dlp
```

Verify installation / 验证安装：
```bash
yt-dlp --version
```

### 2. Check ffmpeg / 检查 ffmpeg（可选但推荐）

```bash
which ffmpeg || apt-get install -y ffmpeg   # Debian/Ubuntu
```

ffmpeg is only needed as fallback for frame extraction — the primary workflow uses direct video passthrough.
ffmpeg 仅作为抽帧备选方案使用 — 主要工作流直接用 data URI 传原始视频。

### 3. Configure Auxiliary Vision Model / 配置辅助视觉模型

The Hermes agent must have an auxiliary vision model configured to analyze video content. This is done via `hermes config set` commands or by editing `config.yaml` directly.
Hermes Agent 需要配置一个辅助视觉模型来理解视频内容。可以通过 `hermes config set` 命令或直接编辑 `config.yaml` 来配置。

#### Option A: Volcano Ark / Doubao（推荐 — 国内可用，支持视频）

Get an API key from [Volcano Ark console](https://console.volcengine.com/ark/). The Ark API is OpenAI-compatible.
从[火山方舟控制台](https://console.volcengine.com/ark/)获取 API Key。Ark 接口兼容 OpenAI 格式。

```bash
# Set provider to "openai" (Ark uses OpenAI-compatible API)
# 设置 provider 为 openai（Ark 兼容 OpenAI 接口格式）
hermes config set auxiliary.vision.provider openai

# Set model name (check available models via API first, see reference)
# 设置模型名称（先通过 API 查看可用模型，详见参考文档）
hermes config set auxiliary.vision.model doubao-seed-2-0-lite-260215

# Set base URL to Ark endpoint
# 设置 Ark 的 endpoint 地址
hermes config set auxiliary.vision.base_url https://ark.cn-beijing.volces.com/api/v3

# Set API key
# 设置 API Key
hermes config set auxiliary.vision.api_key "ark-your-api-key-here"
```

#### Option B: Google Gemini（原生支持视频）

```bash
echo 'GOOGLE_API_KEY=your-key' >> /opt/data/.env
hermes config set auxiliary.vision.provider gemini
hermes config set auxiliary.vision.model gemini-2.5-flash
```

#### Option C: OpenAI GPT-4o

```bash
echo 'OPENAI_API_KEY=your-key' >> /opt/data/.env
hermes config set auxiliary.vision.provider openai
hermes config set auxiliary.vision.model gpt-4o
```

#### Verify configuration / 验证配置

```bash
hermes config | grep -A 10 auxiliary
# Look for vision.provider, vision.model, vision.base_url
# 检查 vision.provider、vision.model、vision.base_url 是否已设置
```

### 4. Set up storage directory / 设置存储目录

Determine the correct permanent storage path:
确定正确的永久存储路径：

```bash
# Check available mounts to see what's host-accessible
# 检查有哪些磁盘挂载点
lsblk
mount | grep -v "cgroup\|proc\|sys\|tmpfs\|devpts\|mqueue\|shm\|overlay\|loop"
```

Create the download directory / 创建下载目录：

```bash
# On a host-mounted volume (recommended so user can access files from host)
# 在宿主机挂载卷上创建（推荐，这样用户可以从宿主机直接访问文件）
mkdir -p /opt/data/图片和视频存储/hermes-student文件下载目录/{视频,图片,音频,书籍,其他文件}

# Or if a large HDD is mounted
# 如果有大容量硬盘挂载
mkdir -p /mnt/hermes-ext2/图片和视频存储/hermes-student文件下载目录/{视频,图片,音频,书籍,其他文件}
```

### 5. Restart session / 重启会话

Configuration changes take effect on next session. Run `/reset` in interactive mode or start a new `hermes` session.
配置更改会在下一个会话生效。在交互模式下运行 `/reset` 或启动新的 `hermes` 会话。

---

## Workflow / 工作流程

### 1. Check yt-dlp availability / 检查 yt-dlp 是否可用

```bash
which yt-dlp || (curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp && chmod +x /usr/local/bin/yt-dlp)
```

### 2. Resolve video URL / 解析视频链接

If the URL is a short link (like `v.douyin.com/xxx`), resolve it first to get the full video ID/page URL. For Douyin, the full URL format is:
如果链接是短链接（如 `v.douyin.com/xxx`），先解析出完整的视频 ID/页面地址。抖音的完整格式是：

```
https://www.douyin.com/video/{video_id}
```

### 3. Preview content before downloading / 下载前预览内容

Always preview first to know what you're dealing with:
始终先预览再下载，确认内容是否是你想要的：

```bash
yt-dlp --print title --print description -s "{full_url}"
```

### 4. Download the video / 下载视频

```bash
yt-dlp -f "best[height<=720]" -o "{temp_path}.mp4" "{full_url}"
```

- Limit resolution to 720p to keep file size manageable
  限制画质为 720p，控制文件大小
- Download to a temp location first (e.g., `/tmp/`), or directly to the permanent dir
  先下载到临时目录（如 `/tmp/`），或直接下载到永久存储目录

### 5. Analyze with vision model / 用视觉模型分析

**PREFERRED: Direct video via data URI（推荐：通过 data URI 直接传视频）**

Read the video file, base64-encode it, and send it as `data:video/mp4;base64,{b64}` to the auxiliary vision model.
读取视频文件 → base64 编码 → 作为 `data:video/mp4;base64,{b64}` 传给辅助视觉模型。

The API call format (OpenAI-compatible) / API 调用格式（兼容 OpenAI 格式）：

```python
with open(video_path, 'rb') as f:
    video_b64 = base64.b64encode(f.read()).decode()

payload = {
    "model": "<vision-model>",
    "messages": [{
        "role": "user",
        "content": [
            {"type": "text", "text": "请详细描述这个视频的内容..."},
            {"type": "video_url", "video_url": {"url": f"data:video/mp4;base64,{video_b64}"}}
        ]
    }],
    "max_tokens": 2000
}
```

**Why data URI is better than frame extraction / 为什么 data URI 优于抽帧：**
- Captures motion, transitions, and temporal relationships
  能捕捉运动、转场和时间关系
- Uses comparable token count (~8000 for a 10s video)
  消耗的 token 量相当（10 秒视频约 8000 token）
- More accurate analysis (model sees actual video, not just static frames)
  分析更准确（模型看到的是真实视频，而非静态截图）

### 6. Rename file with content-based name / 根据内容重命名文件

Use the model's analysis output to generate a descriptive, human-readable filename:
根据模型分析结果生成描述性的、人类可读的文件名：

```python
# Example: model described the video as "推荐Z-Library电子书网站"
# 示例：模型分析结果为"推荐Z-Library电子书网站"
# Result: Z-Library电子书推荐_每天一个强大的网站第9期.mp4
```

Format / 命名格式：`{内容概要}_{来源}.{后缀}`
- ❌ NOT / 不要用: `douyin_20260618_7651997807441016699.mp4`（机器生成的数字 ID，无法直接识别）
- ✅ YES / 推荐: `Z-Library电子书推荐_每天一个强大的网站第9期.mp4`（基于内容，一目了然）

### 7. Save to permanent storage / 保存到永久存储

```bash
mv "{temp_path}.mp4" "{download_dir}/{category}/{final_name}"
```

### 8. File management / 文件管理

- **Permanent**: Downloaded videos stay in the download directory (never delete)
  **永久保留**：下载的视频文件一直保存在下载目录中（永不删除）
- **Temporary**: Any intermediate files (extracted frames, etc.) go into `_temp/` subdirectory and get cleaned after analysis
  **临时文件**：中间文件（抽帧图片等）放入 `_temp/` 子目录，分析完成后自动清理
- Naming: content-based descriptive names, not raw IDs
  命名规则：基于内容的描述性名称，而非原始 ID

## File Storage & Host Accessibility / 文件存储与宿主机访问

### Storage path / 存储路径

Downloaded videos are stored permanently at the user's designated path. This is typically on a host-mounted volume so files are visible from both container and host.
下载的视频永久保存在用户指定的路径。这个路径通常是宿主机挂载卷，容器内外都能访问。

Common pattern / 常见路径格式：
```
/mnt/hermes-ext2/图片和视频存储/hermes-student文件下载目录/{category}/
```

With subdirectories for organization / 分类子目录：
- `视频/` — downloaded videos / 已下载的视频
- `图片/` — downloaded images / 已下载的图片
- `音频/` — audio files / 音频文件
- `书籍/` — ebooks, documents / 电子书、文档
- `其他文件/` — misc / 其他文件

### Container awareness / 容器感知（重要）

The Hermes agent may run inside a **Docker container** with an overlay filesystem. In this setup:
Hermes Agent 可能运行在 **Docker 容器** 内，使用 overlay 文件系统。这种情况下：

- The container may have **host-mounted volumes** that are visible from both container and host — always check these first
  容器可能有**宿主机挂载卷**，容器和宿主机都能看到 — 始终优先检查这些路径
- Check mounts via `mount | grep -v "cgroup\\|proc\\|sys\\|tmpfs"` to identify host-mounted volumes
  通过 `mount` 命令查看挂载点，找出宿主机挂载卷
- The Hermes config itself (`/opt/data/config.yaml`) and `.env` are typically in a host-visible location
  Hermes 配置 (`/opt/data/config.yaml`) 和 `.env` 文件通常在宿主机可见的位置
- The user may also mount a **large storage drive** (mechanical HDD) into the container at a specific mount point — verify with `lsblk` and `mount` to confirm which paths are host-accessible
  用户还可能挂载了**大容量存储盘**（机械硬盘）到容器的某个挂载点 — 用 `lsblk` 和 `mount` 确认哪些路径对宿主机可见

**Determining the correct storage path / 确定正确的存储路径：**
1. Run `lsblk` to see available disks / 运行 `lsblk` 查看可用磁盘
2. Run `mount` to see what's mounted where / 运行 `mount` 查看挂载情况
3. Check if a large data drive (e.g., `/dev/sdb1`) is mounted at the user's expected path (e.g., `/mnt/hermes-ext2/`)
   检查是否有大容量数据盘（如 `/dev/sdb1`）挂载在用户预期的路径
4. Look for the user's designated subdirectory structure (e.g., `图片和视频存储/hermes-student文件下载目录/`)
   查找用户指定的子目录结构
5. Save permanent files to host-mounted paths only — never to container-internal overlay paths alone
   永久文件只保存到宿主机挂载路径 — 不要仅保存在容器的内部 overlay 路径

### Temp file cleanup / 临时文件清理

Intermediate files (extracted frames, temp downloads, etc.) go into a `_temp/` subdirectory and are cleaned after analysis completes. Original downloaded videos are never deleted.
中间文件（抽帧图片、临时下载等）放入 `_temp/` 子目录，分析完成后自动清理。原始下载视频永不删除。

## Pitfalls / 注意事项与坑

- **DO NOT extract frames with ffmpeg as a first approach.** If the vision model supports `video_url`, pass the full video as data URI directly — the analysis is more complete (captures motion, transitions, audio context) and uses comparable tokens.
  **不要一上来就用 ffmpeg 抽帧。** 如果视觉模型支持 `video_url`，先用 data URI 传完整视频 — 分析更全面（能理解运动、转场、上下文），且 token 消耗相近。
- Frame extraction (`ffmpeg -i video.mp4 -vf "fps=1/2" -frames:v 5 frames_%02d.jpg`) is a **fallback only** when direct video passthrough fails.
  抽帧方案仅在直接传视频失败时作为**备选**。
- Douyin video CDN URLs returned from the API have anti-leeching (403). yt-dlp handles authentication internally — always use yt-dlp to download.
  抖音视频的 CDN 链接有反盗链（403），yt-dlp 内部处理鉴权 — 始终用 yt-dlp 下载。
- After downloading, the local file cannot be passed as a URL to the model. Use data URI (`data:video/mp4;base64,...`) instead of hosting a temporary HTTP server.
  下载后的本地文件不能直接作为 URL 传给模型。用 data URI（`data:video/mp4;base64,...`）而不是启动临时 HTTP 服务器。
- The data URI approach sends ~1MB for a short video; ensure the `max_tokens` is high enough (at least 2000) for detailed analysis responses.
  data URI 方式发送短视频约 1MB；确保 `max_tokens` 设置够高（至少 2000），才能得到详细的分析结果。
- **Data URIs for video ARE supported** by Doubao Seed 2.0 Lite and other modern vision models (contrary to earlier assumptions). Always try data URI first before frame extraction.
  豆包 Seed 2.0 Lite 和其他现代视觉模型**确实支持视频 data URI**（不要以为不支持）。始终先尝试 data URI，再考虑抽帧。
- If the model reports "mimetype text/html is not supported", you passed a web page URL instead of a direct video file — download first with yt-dlp.
  如果模型报错 "mimetype text/html is not supported"，说明你传的是网页链接而非视频文件 — 先用 yt-dlp 下载。
- **Always preview first** with `yt-dlp -s --print title --print description` before downloading the full video.
  **始终先预览**，用 `yt-dlp -s --print title --print description` 确认内容后再下载完整视频。
- **Name files by their content**, not by machine IDs. The user browses the directory; descriptive Chinese names like `Z-Library电子书推荐_每天一个强大的网站第9期.mp4` are far more useful than `douyin_7651997807441016699.mp4`.
  **按内容命名文件**，不要用机器 ID。用户会直接浏览目录；像 `Z-Library电子书推荐_每天一个强大的网站第9期.mp4` 这样的描述性中文名远比 `douyin_7651997807441016699.mp4` 有用。
- **Temp file cleanup**: After analysis completes, clean up any `_temp/` files. Keep only the original renamed video in the permanent category subdirectory.
  **清理临时文件**：分析完成后，清理 `_temp/` 目录中的所有文件。只保留已重命名的原始视频在永久分类子目录中。
