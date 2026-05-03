---
name: wiki-sync-to-github
description: "建立 Karpathy LLM Wiki 知识库 + 通过 GitHub 同步到本地 Obsidian。从初始化到日常同步的完整流程。"
version: 2.0.0
author: Hermes Agent
license: MIT
---

# 知识库建立与 GitHub 同步

管理本地 Karpathy LLM Wiki（`~/wiki/`）并同步到 GitHub，让用户在电脑上用 Obsidian 浏览。

## 架构

```
你的服务器（我在这里写 wiki）  →  GitHub  →  你的电脑（Obsidian 浏览）
         ↑                            ↑
     我每次 ingest/update         你 git pull
```

**用户环境：**
- Wiki 路径：`~/wiki/`（`$HOME/wiki`）
- GitHub 仓库：`https://github.com/dingmingcheng9-cmyk/knowledge-warehouse.git`
- GitHub 用户名：`xiaoyequ`
- 网络策略：clone 用 `gh-proxy.com` 镜像加速，push 直连

---

## 一、知识库建立（首次初始化）

### 目录结构

```
~/wiki/
├── SCHEMA.md    ← 规则手册（命名、标签、更新策略）
├── index.md     ← 目录（所有页面一览）
├── log.md       ← 操作日志
├── raw/         ← 原始素材（文章、论文、笔记、音频等）
├── entities/    ← 实体（人物/公司/产品）
├── concepts/    ← 概念（技术/知识）
├── comparisons/ ← 对比分析
└── queries/     ← 有价值的问答存档
```

### 创建步骤

**① 写 SCHEMA.md** — 定义领域、命名规范、标签分类法、frontmatter 格式、创建页面门槛

**② 写 index.md** — 按类型分节（Entities / Concepts / Comparisons / Queries），每页一行摘要

**③ 写 log.md** — 追加式记录所有操作（格式：`## [YYYY-MM-DD] 操作 | 标题`）

**④ 建目录** — `mkdir -p raw/{articles,papers,transcripts,assets} entities concepts comparisons queries`

---

## 二、GitHub 同步流程

### 2.1 首次推送

```bash
# 1. 用 gh-proxy clone 空仓库
git clone https://gh-proxy.com/https://github.com/dingmingcheng9-cmyk/knowledge-warehouse.git /tmp/wiki-sync

# 2. 复制所有 wiki 文件（保持目录结构）
cd ~/wiki
find . -name "*.md" -type f | while read f; do
  mkdir -p "/tmp/wiki-sync/$(dirname "$f")"
  cp "$f" "/tmp/wiki-sync/$f"
done

# 3. 设置 remote（Token 认证）
cd /tmp/wiki-sync
git remote set-url origin https://xiaoyequ:<TOKEN>@github.com/dingmingcheng9-cmyk/knowledge-warehouse.git

# 4. commit & push
git add -A
git commit -m "sync: 全量同步本地知识库（YYYY-MM-DD）"
git push origin main

# 5. 清理
rm -rf /tmp/wiki-sync
```

> Token 含 `@` 等特殊字符时需 URL 编码：`@` → `%40`

### 2.2 增量同步（每次 ingest/update 后）

重复上述 1-5 步，commit message 改为：
```
ingest|update: 操作简述

来源：[URL 或文件名]
更新文件：[文件列表]
```

### 2.3 gh-proxy 说明

clone 时用 `gh-proxy.com` 镜像加速（从国内服务器 clone GitHub 仓库）：
```bash
git clone https://gh-proxy.com/https://github.com/.../repo.git
```

push 时直连 GitHub（不走代理，避免 socks5 拖慢）。

---

## 三、用户在电脑上用 Obsidian 浏览

### 首次设置

```bash
# 在电脑上克隆
cd ~/Desktop   # 或任意目录
git clone https://github.com/dingmingcheng9-cmyk/knowledge-warehouse.git
```

打开 Obsidian → **Open folder as vault** → 选择 `knowledge-warehouse` 文件夹

### 日常更新

```bash
cd ~/Desktop/knowledge-warehouse
git pull
```

Obsidian 会自动刷新，`[[wikilinks]]` 可点击跳转，Graph View 可看知识网络。

---

## 四、日常使用流程

### 当用户发来资料时

```
① 存原始来源 → raw/（加 frontmatter：source_url / ingested / sha256）
② 分析内容，判断是新建概念页还是更新已有页
③ 创建/更新 concepts/ 或 entities/ 等中的页面
   - 每页至少 2 条 [[wikilinks]] 出链
   - 标签必须来自 SCHEMA.md 标签分类法
   - 更新 updated 日期
④ 更新 index.md（新增条目 + 总页数 + 日期）
⑤ 追加 log.md
⑥ 同步到 GitHub
```

### 当用户提问时

```
① 查 index.md → 定位相关页面
② 读页面内容
③ 综合回答，引用 [[wiki页面]]
④ 有价值的问答存档到 queries/
```

---

## 注意事项

- **Token 安全**：推送后清 shell history，或推完后重置 remote 为无凭据 URL
- **首次 clone 是一次性的**，之后每次只需增量同步
- **全量复制策略**：文件少时简单可靠；文件增多后可改用 rsync 或只复制变更文件
- **不要同步本地 .git**：clone 来的仓库自带 .git，不要把本地 wiki 的 .git 覆盖过去
- **避免冲突**：不要同时在 GitHub 网页端编辑。若需强制覆盖用 `git push -f`（需用户确认）
- **每次操作必须更新 index.md 和 log.md**——这是导航和追踪的骨架
- **raw/ 不可修改**——原始资料只写不删不改，修正写在概念页里
