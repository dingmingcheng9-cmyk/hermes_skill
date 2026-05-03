---
name: deepseek-spa-rendering
description: "Render DeepSeek shared conversation pages (and similar Chinese SPA sites) using Chromium headless when the page is client-rendered and curl alone only gets the SPA shell."
version: 1.0.0
author: Hermes Agent
license: MIT
---

# DeepSeek SPA 页面渲染与内容提取

DeepSeek 分享页面（`chat.deepseek.com/share/...`）是纯客户端渲染的 SPA（Single Page Application），对话内容通过 JavaScript 动态加载。用 `curl` 只能获取空壳 HTML（约 7.6KB），无法得到实际对话内容。

**本方案：在无图形界面的 Linux 服务器上安装 Chromium headless 浏览器，渲染页面后提取完整 DOM。**

## 适用场景

- DeepSeek 分享链接（`chat.deepseek.com/share/...`）
- 其他类似的客户端渲染 SPA（如部分字节跳动系产品）
- 任何需要 JavaScript 执行才能获取内容的页面
- 当 `web_extract` 或 `curl` 只拿到空壳时

## 安装 Chromium

### 环境信息
- 系统：Debian trixie (testing)
- 架构：amd64
- 特点：apt-get 从官方源下载超时，需用国内镜像直接下载 .deb 包

### 第一步：从镜像下载 .deb 包

```bash
# 不设置代理（apt 不支持 socks5）
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY all_proxy ALL_PROXY

# 从清华大学安全更新镜像下载
BASE_URL="https://mirrors.tuna.tsinghua.edu.cn/debian-security/pool/updates/main/c/chromium"

curl -L --max-time 120 "${BASE_URL}/chromium_147.0.7727.137-1~deb13u1_amd64.deb" -o /tmp/chromium.deb
curl -L --max-time 120 "${BASE_URL}/chromium-common_147.0.7727.137-1~deb13u1_amd64.deb" -o /tmp/chromium-common.deb
curl -L --max-time 120 "${BASE_URL}/chromium-sandbox_147.0.7727.137-1~deb13u1_amd64.deb" -o /tmp/chromium-sandbox.deb
```

### 第二步：安装并补全依赖

```bash
# 先强制安装 chromium 包
dpkg -i --force-depends /tmp/chromium-common.deb /tmp/chromium-sandbox.deb /tmp/chromium.deb

# 检查缺失的共享库
ldd /usr/lib/chromium/chromium 2>&1 | grep "not found"
```

常见缺失库及对应 .deb：
- `libopenh264.so.8` → `libopenh264-8`
- `libdouble-conversion.so.3` → `libdouble-conversion3`
- `libharfbuzz-subset.so.0` → `libharfbuzz-subset0`
- `libminizip.so.1` → `libminizip1t64`
- `libXNVCtrl.so.0` → `libxnvctrl0`

```bash
# 从镜像下载缺失库
for lib in libopenh264-8 libdouble-conversion3 libharfbuzz-subset0 libminizip1t64 libxnvctrl0; do
  URL=$(apt-cache show $lib 2>/dev/null | grep -m1 "Filename:" | awk '{print "https://mirrors.tuna.tsinghua.edu.cn/debian/"$2}')
  curl -L --max-time 60 "$URL" -o "/tmp/$(basename $URL)"
done

# 强制安装所有库 + chromium
dpkg -i --force-depends /tmp/*.deb
```

### 验证安装

```bash
/usr/lib/chromium/chromium --headless --no-sandbox --disable-gpu --version
```

## 渲染 DeepSeek 页面

### 渲染并导出完整 DOM

```bash
/usr/lib/chromium/chromium --headless --no-sandbox --disable-gpu \
  --dump-dom "https://chat.deepseek.com/share/XXXXX" \
  2>/dev/null > /tmp/rendered.html
```

- `--headless`：无界面模式（服务器环境必需）
- `--no-sandbox`：在 Docker/root 环境下绕过沙箱限制
- `--disable-gpu`：禁用 GPU 加速（服务器无显卡）
- `--dump-dom`：渲染完成后输出完整 DOM 到 stdout
- `2>/dev/null`：屏蔽 dbus/GLib 等无关错误信息

### 提取对话文本内容

渲染后的 HTML 约 80KB，对话内容以 markdown 格式嵌入在 DOM 中。提取方式：

**方式 A：直接读取 DOM 中的文本**
```bash
# 渲染后抓取含关键字的文本行
grep -oP '指令速查表[^<]+' /tmp/rendered.html
```

**方式 B：Python 提取（注意大文件正则性能）**
```python
import re
with open('/tmp/rendered.html') as f:
    html = f.read()
# 移除 style/script 标签
text = re.sub(r'<style[^>]*>.*?</style>', '', html, flags=re.DOTALL)
text = re.sub(r'<script[^>]*>.*?</script>', '', text, flags=re.DOTALL)
text = re.sub(r'<[^>]+>', '\n', text)
# 清理多余空行
text = re.sub(r'\n{3,}', '\n\n', text)
```

## 工作原理

`--dump-dom` 让 Chromium 像普通浏览器一样：
1. 加载 HTML 壳（curl 拿到的那 7.6KB）
2. 解析并执行所有 `<script>` 标签里的 JavaScript
3. JS 发起 API 请求获取对话数据
4. 渲染完整 DOM（包括对话内容）
5. 输出渲染后的完整 HTML 到 stdout

## 注意事项

- **渲染需要时间**：页面加载 + JS 执行，通常 1-3 秒
- **dbus/GLib 错误**：服务器环境没有桌面总线服务，会出现 dbus 连接失败日志，不影响渲染结果，用 `2>/dev/null` 忽略即可
- **偶尔会触发验证码**：频繁请求可能出现人机验证，此时需用 `browser_vision` 工具配合截图识别
- **代理问题**：如果通过代理访问 DeepSeek 可能触发 429 限流（代理 IP 被限），直连反而更快（服务器在中国）
- **版本更新**：Chromium 版本号（当前 147）会随 Debian 仓库更新，使用前检查最新版本号

## 与 curl 的区别

| 方法 | 获取内容 | 大小 | 页面是否执行 JS |
|------|---------|------|----------------|
| `curl` | SPA 空壳 HTML | ~7.6KB | ❌（纯 HTML，无数据） |
| Chromium `--dump-dom` | 完整渲染 DOM | ~80KB | ✅（JS 执行后渲染） |
