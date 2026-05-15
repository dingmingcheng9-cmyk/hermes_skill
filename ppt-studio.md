---
name: ppt-studio
description: "统一PPT技能：既能生成世界级专业演示文稿（26风格/6步Pipeline），也能读取/编辑/修改现有PPTX文件。触发决策：说'做'→生成模式；说'改/读/编辑'→文件操作模式。"
tags: [ppt, presentation, powerpoint, design, generation, editing, html, svg]
---

# PPT Studio — 统一PPT技能

> **本技能源自 GitHub 项目 [Akxan/ppt-agent-skill](https://github.com/Akxan/ppt-agent-skill)**，经实战演化形成双模式（生成 + 编辑/读取）统一技能。完整参考文件（94 个风格/布局/图表/设计原则/Playbook 文档 + 18 个 Python 脚本）均在原仓库中，建议 clone 到本地使用。

## 两种工作模式

| 用户意图 | 加载模式 | 说明 |
|---------|---------|------|
| 「做个关于X的PPT」 | **生成模式** | 从零生成世界级演示稿 |
| 「把这个PPT改成深色」 | **编辑模式** | 修改现有PPTX文件 |
| 「读取这个PPT内容」 | **读取模式** | 解析现有文件 |
| 「做个培训课件/路演PPT」 | **生成模式** | 隐式触发生成 |

**核心决策：** 有现有文件 + 要改动 → 编辑/读取模式；无现有文件 + 要从零做 → 生成模式。

---

# ═══════════════════════════════
# 生成模式（从零创建）
# ═══════════════════════════════

## 环境感知

开始工作前自省 agent 拥有的工具能力：

| 能力 | 降级策略 |
|------|---------|
| **信息获取**（搜索/URL/文档/知识库） | 全部缺失 → 依赖用户提供材料 |
| **图片生成**（绝大多数环境都有） | 缺失 → 纯 CSS 装饰替代 |
| **文件输出** | 必须有 |
| **脚本执行**（Python/Node.js） | 缺失 → 跳过自动打包和 SVG 转换 |

**原则**：检查实际可调用的工具列表，有什么用什么。

## 路径约定

| 变量 | 含义 | 获取方式 |
|------|------|---------|
| `SKILL_DIR` | 本 SKILL.md 所在目录的绝对路径 | 触发 Skill 时读取 SKILL.md 的目录 |
| `OUTPUT_DIR` | 产物输出根目录 | 用户当前工作目录下的 `ppt-output/`（首次使用时 `mkdir -p` 创建） |

## 输入模式与复杂度自适应

### 入口判断

| 入口 | 示例 | 从哪步开始 |
|------|------|-----------|
| 纯主题 | "做一个 Dify 企业介绍 PPT" | Step 1 完整流程 |
| 主题 + 需求 | "15 页 AI 安全 PPT，暗黑风" | Step 1（跳部分已知问题）|
| 源材料 | "把这篇报告做成 PPT" | Step 1（材料为主）|
| 已有大纲 | "我有大纲了，生成设计稿" | Step 4 或 5 |

### 复杂度自适应

| 规模 | 页数 | 调研 | 搜索 | 策划 | 生成 |
|------|------|------|------|------|------|
| **轻量** | ≤ 8 页 | 3 题精简版 | 3-5 个查询 | Step 3 可与 Step 4 合并 | 逐页生成 |
| **标准** | 9-18 页 | 完整 7 题 | 8-12 个查询 | 完整流程 | 按 Part 分批，每批 3-5 页 |
| **大型** | > 18 页 | 完整 7 题 | 10-15 个查询 | 完整流程 | 按 Part 分批，每批 3-5 页，批间确认 |

## 6 步 Pipeline

```
Step 1: 需求调研    → 7题三层递进访谈（场景/受众/目的）
Step 2: 资料搜集    → 3-15查询，自适应背景研究
Step 3: 大纲策划    → 金字塔原理 + JSON大纲输出
Step 4: 策划稿      → Bento Grid 策划卡（每页目的/核心信息/证据/布局/层级）
Step 5: HTML设计稿   → 26风格选1 + 智能配图
Step 6: 后处理      → HTML → SVG → 可编辑PPTX
```

详细参考见原项目 `references/playbooks/`（各步骤 playbook）、`references/prompts/`（各步骤 prompt 模板）、`references/pipeline-compat.md`（管线兼容性）。

### Step 1: 需求调研（7题三层递进）

**触发条件**：用户只给了主题，未明确说明受众/场景/页数。

围绕"谁看 → 为什么看 → 看完要做什么"递进：

**第一层：场景与受众**
1. 演示场景（决定信息密度/节奏/视觉）：A.现场演讲 B.自阅文档 C.培训教学 D.其他
2. 核心受众（决定专业深度）：根据搜索结果动态生成3-4种画像
3. 看完之后希望观众做什么（决定内容编排最终导向）：A.决策 B.记住核心 C.掌握方法 D.改变认知

**第二层：内容策略**
4. 叙事结构：A.问题→方案→效果 / B.是什么→为什么→怎么做 / C.全景→聚焦→行动 / D.对比 / E.时间线
5. 内容侧重（根据搜索结果动态生成3-4个选项）
6. 说服力要素：A.硬数据 B.案例故事 C.权威背书 D.流程方法

**第三层：执行细节**
7. 补充信息：演讲人/日期/公司/品牌色/页数偏好/必须包含或回避的内容/配图偏好

### Step 2: 资料搜集

根据 Step 1 的答案，进行 3-15 个针对性搜索查询。每个查询结束后提取关键数据点，填入大纲策划的 data_needs 字段。

参考原项目 `references/playbooks/research-phase*.md`、`references/playbooks/source-phase*.md`。

### Step 3: 大纲策划（JSON 输出格式）

```json
[PPT_OUTLINE]
{
  "ppt_outline": {
    "cover": { "title": "...", "sub_title": "...", "presenter": "...", "date": "...", "company": "..." },
    "table_of_contents": { "title": "目录", "content": ["第一部分", "第二部分"] },
    "parts": [
      {
        "part_title": "第一部分：...",
        "part_goal": "这一部分要说明什么",
        "pages": [
          { "title": "...", "goal": "核心结论", "content": ["要点1", "要点2"], "data_needs": ["..."] }
        ]
      }
    ],
    "end_page": { "title": "总结与展望", "content": ["核心回顾1", "核心回顾2", "CTA"] }
  }
}
[/PPT_OUTLINE]
```

参考原项目 `references/playbooks/outline-phase*.md`、`references/prompts/tpl-outline-*.md`。

### Step 4: 策划稿（Bento Grid）

每页输出策划卡：
- **目的**：这页最想让观众记住什么
- **核心信息**：标题 + 主卡片内容 + 数据亮点
- **证据支撑**：来自搜索的真实数据
- **布局形式**：几张卡片、什么类型、如何排列
- **层级关系**：主次分明

详细策划 playbook 见原项目 `references/playbooks/step4/page-planning-playbook.md`。

### Step 5: 风格选择（26选1）

#### 暗色专业（7）

| ID | 灵感 | 场景 |
|----|------|------|
| `dark_tech` | Linear.app | AI/SaaS/开发者工具 |
| `xiaomi_orange` | Apple Keynote 硬件 | 硬件/IoT/汽车发布 |
| `luxury_purple` | Tom Ford | 奢侈品/高端品牌 |
| `nocturne_violet` | Linear 紫光版 | 设计师 SaaS |
| `cyberpunk_neon` | Cyberpunk 2077 | 电竞/游戏/Web3 |
| `chrome_y2k` | Y2K/Vaporwave | Web3/千禧年复古 |
| `noir_film` | 黑白纪录片 | 纪录片/影像 |

#### 浅色高级（8）

| ID | 灵感 | 场景 |
|----|------|------|
| `blue_white` | Apple 企业页 | 企业 SaaS/培训/金融 |
| `fresh_green` | Aesop | 护肤/养生/美妆 |
| `minimal_gray` | NYT Magazine | 学术/法务/咨询 |
| `mocha_editorial` | Anthropic | AI 安全/出版 |
| `medical_pulse` | Mayo Clinic | 医疗/医药 |
| `earth_concrete` | Suisse Int'l | 建筑/工业 |
| `champagne_gold` | 婚礼请柬 | 婚庆/颁奖 |
| `liquid_glass` | iOS 26/visionOS | XR/AR 发布 |

#### 活力鲜明（4）

| ID | 灵感 | 场景 |
|----|------|------|
| `vibrant_rainbow` | Stripe Sessions | 开发者大会/社区 |
| `kindergarten_pop` | 儿童插画 | 教育/童趣品牌 |
| `bauhaus_block` | 包豪斯设计 | 设计/艺术/创意 |
| `candy_pastel` | 柔和糖果色 | 美妆/生活/软品牌 |

#### 东方文化（3）

| ID | 灵感 | 场景 |
|----|------|------|
| `royal_red` | 北京冬奥 | 中国风/党政/庆典 |
| `sakura_wabi` | 日系侘寂 | 日式/禅意/淡雅 |
| `ink_jade` | 新中式水墨 | 茶/香/空间/人文 |

#### 自然/复古（4）

| ID | 灵感 | 场景 |
|----|------|------|
| `botanic_forest` | Patagonia | 户外/环保/自然 |
| `safari_savanna` | 非洲草原 | 旅行/地理/动物 |
| `retro_70s` | 70 年代复古 | 复古品牌/怀旧 |
| `gov_authority` | 政府文件蓝 | 政府/国企/正式汇报 |

**完整风格规格（色板、字体、CSS 变量、氛围图）** 见原项目 `references/styles/` 目录。视觉预览 Gallery 见 `ppt-output/style-gallery/`（26 种风格 HTML 预览）。

### Bento Grid 7 布局

画布: 1280×720px, 内容区: 1200×580px (x=40, y=80), gap=20px, border-radius=12px

| 布局 | CSS | 适用 |
|------|-----|------|
| 单一焦点 | `grid-template: 1fr / 1fr` | 1个核心论点/大数据 |
| 50/50对称 | `grid-template: 1fr / 1fr 1fr` | 对比/并列 |
| 非对称两栏 | `grid-template: 1fr / 2fr 1fr` | 主次关系（最常用）|
| 三栏等宽 | `grid-template: 1fr / repeat(3, 1fr)` | 3个并列 |
| 主次结合 | `grid-template: 1fr 1fr / 2fr 1fr` | 层级关系 |
| 顶部英雄式 | `grid-template: auto 1fr / repeat(3, 1fr)` | 总分结构 |
| 混合网格 | `grid-template: repeat(3, 1fr) / 1fr 1fr` | 高密度4-6块 |

完整布局参考见原项目 `references/layouts/`（10 种布局，含 L-shape、T-shape、Waterfall 等变体）。

### 18 种图表

| 层级 | 数量 | 类型 |
|------|------|------|
| 基础 | 8 | 进度条/柱图/环形图/折线/点阵/KPI卡/指标行/评分 |
| 进阶 | 6 | 雷达/时间线/漏斗/仪表盘/多组对比/地理 |
| ECharts级 | 4 | 世界地图/关系网络/桑基/热力日历 |

全部纯 HTML/CSS/SVG 实现，无 JS 依赖（保证 SVG→PPTX 管线兼容）。完整图表 CSS 代码见原项目 `references/charts/`。

### 设计运行时参考

| 文件 | 内容 |
|------|------|
| `references/design-runtime/design-specs.md` | 完整设计规范（容器/间距/排版/颜色系统） |
| `references/design-runtime/css-weapons.md` | CSS 技术栈（渐变/玻璃态/动画等） |
| `references/design-runtime/data-type-visual-mapping.md` | 数据类型→视觉映射 |
| `references/design-runtime/data-type-decoration-mapping.md` | 数据类型→装饰映射 |
| `references/design-runtime/director-command-rules.md` | 设计指令规则 |
| `references/design-runtime/director-command-examples.md` | 设计指令示例 |

### 页面组件参考

| 组件 | 文件 |
|------|------|
| 封面模板 | `references/page-templates/cover.md` |
| 目录模板 | `references/page-templates/toc.md` |
| 章节模板 | `references/page-templates/section.md` |
| 结尾模板 | `references/page-templates/end.md` |
| 卡片样式 | `references/blocks/card-styles.md` |
| 对比 | `references/blocks/comparison.md` |
| 时间线 | `references/blocks/timeline.md` |
| 人物介绍 | `references/blocks/people.md` |
| 引用 | `references/blocks/quote.md` |
| 图表/图示 | `references/blocks/diagram.md`, `blocks/matrix-chart.md` |
| 全幅图片 | `references/blocks/image-hero.md` |

### 后处理管线

```bash
# HTML → SVG（文字可编辑）
python3 scripts/html2svg.py output/

# SVG → PPTX（OOXML原生，可编辑）
python3 scripts/svg2pptx.py output/

# 生成预览画廊
python3 scripts/gallery.py --style dark_tech
```

所有脚本见原项目 `scripts/` 目录（18 个 Python 脚本）。

## 方法论核心（生成模式）

1. **从问题开始，不从模板开始** — 先问"给谁看/为什么做/希望对方记住什么"
2. **内容先行，设计随后** — 策划稿验证信息结构，确认后再投入设计
3. **策划稿是中间层** — 每页目的/核心信息/证据/布局/层级，不跳步
4. **Bento Grid 是AI最容易掌握的设计语言** — 约束反而提升输出质量
5. **阶段间用JSON合同传递** — 无歧义，避免信息损耗
6. **真实数据填充** — 先搜索再生成，绝不编造数据

---

# ═══════════════════════════════
# 编辑/读取模式（操作现有文件）
# ═══════════════════════════════

## 读取内容

```bash
# 文本提取
python -m markitdown presentation.pptx

# 缩略图预览
python scripts/thumbnail.py presentation.pptx

# RAW XML
python scripts/office/unpack.py presentation.pptx unpacked/
```

## 编辑现有PPT

1. 用 `thumbnail.py` 分析模板
2. Unpack → 操作 slides → 编辑内容 → 清理 → pack

```bash
# 解包
python scripts/office/unpack.py input.pptx unpacked/

# 修改 slides/slide*.xml 中的文字

# 打包
python scripts/office/pack.py unpacked/ output.pptx
```

## 从零创建PPTX（python-pptx）

```python
from pptx import Presentation
from pptx.util import Inches, Pt
from pptx.dml.color import RGBColor

prs = Presentation()
prs.slide_width = Inches(10)
prs.slide_height = Inches(5.625)

slide = prs.slides.add_slide(prs.slide_layouts[6])  # 空白布局

left = Inches(1)
top = Inches(2)
width = Inches(8)
height = Inches(1)
textbox = slide.shapes.add_textbox(left, top, width, height)
tf = textbox.text_frame
p = tf.paragraphs[0]
p.text = "Hello World"
p.font.size = Pt(44)
p.font.bold = True
```

---

# ═══════════════════════════════
# 设计规范（通用）
# ═══════════════════════════════

## 颜色 Palettes

| Theme | Primary | Secondary | Accent |
|-------|---------|-----------|--------|
| **Midnight Executive** | `#1E2761` | `#CADCFC` | `#FFFFFF` |
| **Forest & Moss** | `#2C5F2D` | `#97BC62` | `#F5F5F5` |
| **Coral Energy** | `#F96167` | `#F9E795` | `#2F3C7E` |
| **Warm Terracotta** | `#B85042` | `#E7E8D1` | `#A7BEAE` |
| **Ocean Gradient** | `#065A82` | `#1C7293` | `#21295C` |
| **Charcoal Minimal** | `#36454F` | `#F2F2F2` | `#212121` |
| **Teal Trust** | `#028090` | `#00A896` | `#02C39A` |

## 字体搭配

| Header Font | Body Font |
|-------------|-----------|
| Georgia | Calibri |
| Arial Black | Arial |
| Trebuchet MS | Calibri |
| Palatino | Garamond |

## 字号阶梯

| 元素 | 字号 |
|------|------|
| Slide Title | 36-44pt bold |
| Section Header | 20-24pt bold |
| Body Text | 14-16pt |
| Captions | 10-12pt muted |

## 排版铁律

- **每页必须有视觉元素** — 图片/图表/图标/形状，纯文字页很快被遗忘
- **布局要变化** — 避免连续多页相同布局，交替使用两栏/网格/全幅
- **左对齐正文** — 标题可居中，正文和列表左对齐
- **字号对比要强** — 标题36pt+ vs 正文14-16pt，不要中间地带
- **留白呼吸** — 0.5"最小边距，内容块之间0.3-0.5"间距
- **颜色有主次** — 一个主色占60-70%视觉权重，1-2个辅助色，一个锐利强调色

## 不要做的事

- ❌ 渐变色块做背景装饰（除微妙过渡外）
- ❌ 同色系文字和背景（低对比度）
- ❌ 标题下加装饰线（AI生成PPT的标志）
- ❌ 纯文字页无任何视觉元素
- ❌ 所有页面同一个布局
- ❌ 正文字号 < 12pt

## 设计原则参考

见原项目 `references/principles/` 目录：
- `narrative-arc.md` — 叙事弧线（开场→冲突→解决→行动）
- `visual-hierarchy.md` — 视觉层级
- `cognitive-load.md` — 认知负荷管理
- `composition.md` — 构图原则
- `color-psychology.md` — 色彩心理学
- `data-visualization.md` — 数据可视化
- `failure-modes.md` — 常见失败模式
- `design-principles-cheatsheet.md` — 设计原则速查表

---

# ═══════════════════════════════
# QA（必须执行）
# ═══════════════════════════════

## 内容 QA

```bash
python -m markitdown output.pptx
# 检查：遗漏内容/错字/顺序错误
# grep占位符：
python -m markitdown output.pptx | grep -iE "xxxx|lorem|ipsum|placeholder"
```

## 视觉 QA（用 subagent）

```bash
# 转图片
python scripts/office/soffice.py --headless --convert-to pdf output.pptx
pdftoppm -jpeg -r 150 output.pdf slide

# 交给subagent检查：
# - 重叠元素（文字穿过形状）
# - 文字溢出/截断
# - 间距不均匀
# - 留白不足(< 0.5")
# - 低对比度文字
```

**第一遍几乎不可能完美。QA 是 bug hunt，不是确认步骤。**

---

# ═══════════════════════════════
# 飞书文件发送（附加能力）
# ═══════════════════════════════

当用户要求把 .pptx 发到飞书时，使用 `feishu-file-delivery` 技能（自动加载）。

---

# ═══════════════════════════════
# 触发决策速查
# ═══════════════════════════════

| 用户说... | 加载 |
|-----------|------|
| 「做一个关于X的PPT」 | 生成模式 |
| 「把这个PPT改成深色」 | 编辑模式 |
| 「读取这个PPT内容」 | 读取模式 |
| 「做个培训课件/路演deck」 | 生成模式 |
| 「在这页加一张图」 | 编辑模式 |
| 「我要给老板汇报Y」 | 生成模式（隐触发） |

## 完整资源

- **上游项目**: [github.com/Akxan/ppt-agent-skill](https://github.com/Akxan/ppt-agent-skill)
- **参考文件**: 94 个 .md（风格/布局/图表/设计原则/Playbook/Prompt）
- **脚本**: 18 个 Python（HTML→SVG→PPTX 管线）
- **Gallery**: 26 种风格 HTML 预览 + PNG 截图
- **安装**: `git clone https://github.com/Akxan/ppt-agent-skill.git` 获取完整资源
