---
name: memory-organization
description: 记忆自动巡检+整理一体化方案——定时检查memory/user使用率，按阈值自动精简，完整汇报给用户审查
---

# Memory Organization — 记忆自动巡检与整理

> 融合 `memory-organization`（整理方法论）与 `memory-inspection`（定时巡检机制），形成一站式记忆管理方案。

---

## 1. 为什么需要整理

Hermes Agent 的持久记忆有容量上限：
- `memory`：**2,200 字符**
- `user`（用户画像）：**1,375 字符**

如果不整理，会：
- 新信息存不进去（需要先删除旧条目）
- 每条回复前面注入的记忆越来越长，浪费 token
- 大量详细配置占空间，但这些内容其实已被技能覆盖

**核心理念**：记忆只存「目录指针」，详细操作步骤放技能。

| 存什么 | 不存什么 |
|---|---|
| 技能名称 + 关键参数（一句话） | 完整的安装/配置步骤 |
| 唯一环境信息（IP、端口、用户名） | 已被技能覆盖的流程 |
| 用户偏好、行为规则 | 可复现的通用操作 |

---

## 2. 触发条件

- **每周五定时执行**（cron job）
- 用户手动要求检查记忆/画像使用情况

---

## 3. 巡检步骤

### 3.1 检查使用率

```text
# 通过 memory 工具的返回信息观察 usage
# memory 上限 2,200 字符
# user 上限 1,375 字符
```

### 3.2 判断是否需要整理

| 条件 | 动作 |
|---|---|
| 两者都 < 85% | **不整理**，直接汇报当前状态 |
| 任一个 ≥ 85% | 按第4节方法论进行**精简整理** |

### 3.3 整理原则

- 已被技能覆盖的详细步骤 → 浓缩为目录指针（格式：`主题 → skill 技能名(关键参数)`）
- 用户偏好/行为规则 → 保留但精简到一句话
- 凭证信息（账号密码等）→ **绝不删除**
- 只做浓缩，不做破坏性删除

### 3.4 输出/汇报格式

```
✅ 状态：正常 / 已整理
📊 记忆使用率：x%
📊 画像使用率：x%
📝 记忆条目数：x
📝 画像条目数：x
（如有整理：清理了哪些、释放了多少空间）

=== 📦 记忆(memory) ===
1. [内容]
2. [内容]
...

=== 👤 用户画像(user) ===
1. [内容]
2. [内容]
...
```

---

## 4. 整理方法论

### 4.1 梳理现有记忆

列出每条记忆，评估：

| 维度 | 判断标准 |
|---|---|
| 是否有对应技能覆盖？ | 检查 `skills_list` 和 `skill_view` |
| 是否是通用操作流程？ | 若可抽象为通用步骤 → 提炼为技能 |
| 是否只是环境事实？ | 保留但精简到一句话 |
| 是否是用户行为偏好？ | 保留（适合存 memory） |

### 4.2 分类处理

**A. 已被技能覆盖 → 替换为简短目录指针**

格式：`主题 → 技能 skill-name（关键参数）`

例：
```
USB HDD → 技能 storage-management（sdc西数→/mnt/hermes-ext, SMART需-d sat）
代理 → 技能 ubuntu-proxy-singbox（sing-box anytls 已配）
Samba → 技能 nas-raidrive-mapping（\\192.168.31.187\hermes-ext）
```

**B. 通用流程但无技能 → 提炼为新技能**

将完整步骤写入 SKILL.md，注意：
- 包含触发条件、具体命令、踩坑记录、验证步骤
- 放在合适类别下（devops, hermes-agent, storage 等）
- 上传到用户 GitHub 技能仓库

**C. 用户偏好/行为规则 → 保留但精简**

这些是 `memory(target='user')` 的典型内容，直接保留精简版。

### 4.3 清理记忆

```text
# 删除被技能覆盖的详细条目
memory(action='remove', target='memory', old_text='唯一标识该条目的文字')

# 替换为精简版
memory(action='replace', target='memory', old_text='原条目部分文字', content='精简版内容')
```

**注意**：`old_text` 只需要提供能唯一匹配到该条目的子字符串，无需完整内容。

### 4.4 验证结果

整理后检查：
- 总字符数 < 2,000（留余量）
- 每条都是一行能读完的短句
- 关键环境信息仍在（IP、端口、路径、用户名）
- 技能名正确可索引

---

## 5. 技能生命周期管理

### 何时创建新技能

- 完成 5+ 步骤的复杂任务后
- 发现反复使用的固定流程时
- 用户明确说「把这个记下来」
- 踩坑后需要记录解决方法

### 何时更新已有技能

```text
skill_manage(action='patch', name='技能名',
  old_string='旧内容',
  new_string='新内容')
```

- 命令参数变更时
- 发现新踩坑点
- 有更优的替代方案

### 上传到 GitHub 仓库

```text
# 通过 GitHub API PUT 到仓库根目录
# 认证: git credential store 已配，无需手动提供 token
# 仓库: dingmingcheng9-cmyk/hermes_skill
# 文件名格式: skill-name.md
```

---

## 6. Cron Job 设置

每周五执行巡检：

```text
cronjob(action='create',
  name='每周五记忆巡检',
  schedule='0 9 * * 5',
  skills=['memory-organization'],
  prompt='执行 memory-organization 技能进行记忆巡检与整理')
```

---

## 7. 注意事项

- **绝不删除**账号密码等凭证信息
- 用户偏好规则只精简不删除
- 整理完后必须完整展示全部记忆和画像内容供用户审查
- 整理结果直接汇报给用户（通过 cron job 自动交付），静默发送

---

## 8. 配套技能参考

| 主题 | 技能名 |
|---|---|
| 代理配置 | `ubuntu-proxy-singbox` |
| Hermes Agent 管理 | `hermes-agent` |
| 硬盘/存储管理 | `storage-management` |
| NAS 硬盘映射 | `nas-raidrive-mapping` |
| 对话日志归档 | 内置 cron job（ID: dfec3fa01224） |
