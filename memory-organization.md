---
name: memory-organization
description: 定期整理 Hermes Agent 持久记忆（memory）的方法论——将详细配置浓缩为技能目录指针，保持记忆紧凑可维护
---

# Hermes Agent 记忆整理

## 为什么需要整理

Hermes Agent 的持久记忆（`memory` 工具）有 **2,200 字符上限**。随着使用积累，记忆会迅速填满。如果不整理：

- 新信息存不进去（需要先删除旧条目）
- 每条回复开头系统注入的记忆越来越长，浪费 token
- 大量详细配置占空间，但这些内容其实已由技能覆盖

**核心理念：** 记忆只存「目录指针」，详细操作步骤放技能。

| 存什么 | 不存什么 |
|---|---|
| 技能名称 + 关键参数（一句话） | 完整的安装/配置步骤 |
| 唯一环境信息（IP、端口、用户名） | 已被技能覆盖的流程 |
| 用户偏好、行为规则 | 可复现的通用操作 |

---

## 整理步骤

### 1. 梳理现有记忆

```bash
# 读取当前完整记忆
# 工具: memory(action='...', target='memory')
```

列出每条记忆，评估：

| 维度 | 判断标准 |
|---|---|
| 是否有对应技能覆盖？ | 检查 `skills_list` 和 `skill_view` |
| 是否是通用操作流程？ | 若可抽象为通用步骤 → 提炼为技能 |
| 是否只是环境事实？ | 保留但精简到一句话 |
| 是否是用户行为偏好？ | 保留（适合存 memory） |

### 2. 分类处理

**A. 已被技能覆盖 → 替换为简短目录指针**

格式：`主题 → 技能 skill-name（关键参数）`

例：
```
USB HDD → 技能 storage-management（sdc西数→/mnt/hermes-ext, SMART需-d sat）
代理 → 技能 ubuntu-proxy-singbox（sing-box anytls 已配）
Samba → 技能 nas-raidrive-mapping（\\\\192.168.31.187\\hermes-ext）
```

**B. 通用流程但无技能 → 提炼为新技能**

将完整步骤写入 SKILL.md，注意：
- 包含触发条件、具体命令、踩坑记录、验证步骤
- 放在合适类别下（devops, hermes-agent, storage 等）
- 上传到用户 GitHub 技能仓库

**C. 用户偏好/行为规则 → 保留但精简**

这些是 `memory(target='user')` 的典型内容，直接保留精简版。

### 3. 清理记忆

```bash
# 删除被技能覆盖的详细条目
memory(action='remove', target='memory', old_text='唯一标识该条目的文字')

# 替换为精简版
memory(action='replace', target='memory', old_text='原条目部分文字', content='精简版内容')
```

**注意 `old_text` 参数：** 只需要提供能唯一匹配到该条目的子字符串，无需完整内容。

### 4. 验证结果

整理后检查：
- 总字符数 < 2,000（留余量）
- 每条都是一行能读完的短句
- 关键环境信息仍在（IP、端口、路径、用户名）
- 技能名正确可索引

---

## 技能生命周期管理

### 何时创建新技能

- 完成 5+ 步骤的复杂任务后
- 发现反复使用的固定流程时
- 用户明确说「把这个记下来」
- 踩坑后需要记录解决方法

### 何时更新已有技能

```bash
skill_manage(action='patch', name='技能名',
  old_string='旧内容',
  new_string='新内容')
```

- 命令参数变更时
- 发现新踩坑点
- 有更优的替代方案

### 上传到 GitHub 仓库

```bash
# 从本地读取技能内容
cat /root/.hermes/skills/<类别>/<技能名>/SKILL.md | base64

# 通过 GitHub API PUT 到仓库根目录
# 文件名格式: skill-name.md
# 仓库: dingmingcheng9-cmyk/hermes_skill
# 认证: git credential store 已配，无需手动提供 token
```

---

## 整理频率建议

| 场景 | 建议 |
|---|---|
| 记忆接近满（>90%）| 立即整理 |
| 每周例行 | 检查记忆使用率，精简长条目 |
| 每次创建新技能后 | 检查是否有对应记忆可以精简 |
| 月末 | 全面审视，合并同主题条目 |

---

## 配套技能参考

整理时可能涉及到的已有技能：

| 主题 | 技能名 |
|---|---|
| 代理配置 | `ubuntu-proxy-singbox` |
| Hermes Agent 管理 | `hermes-agent` |
| 硬盘/存储管理 | `storage-management` |
| NAS 硬盘映射 | `nas-raidrive-mapping` |
| 对话日志归档 | 内置 cron job（ID: dfec3fa01224） |
