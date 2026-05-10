---
name: jupyter-nbconvert-export
description: "Notebook文件识别、代码补全、nbconvert导出HTML全流程 — 处理.ipynb.bin魔术后缀，中文编码陷阱，GridSearchCV超时，CJK字体缺失。"
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [jupyter, nbconvert, notebook, export, html, knn, data-science]
    category: data-science
---

# Jupyter Notebook nbconvert 导出工作流

适用于：从飞书/其他平台收到 `.ipynb` 或 `.ipynb.bin` 文件，需要执行并导出为交互式 HTML 的场景。

## 完整工作流

### 阶段一：文件识别与修复

#### 1. 处理 `.ipynb.bin` 魔术后缀
用户上传的 Jupyter Notebook 可能被重命名为 `.ipynb.bin`。

```bash
# 检查文件头部，确认是 JSON 格式
head -c 100 /path/to/file.ipynb.bin
# 如果输出以 { 开头，说明是 JSON / Jupyter Notebook

# 重命名为标准后缀
mv file.ipynb.bin file.ipynb
```

#### 2. JSON 校验
```bash
python3 -c "import json; json.load(open('file.ipynb')); print('OK')"
```

### 阶段二：代码生成（代码格为空时）

如果 Notebook 只有 Markdown 描述格而代码格为空，需要根据描述自动生成代码。

**关键规则：**
- 通过 Python 脚本生成 JSON，**不要直接写 JSON 文本文件**
- 中文内容必须用 `json.dump(..., ensure_ascii=False)`，否则中文引号 `"` `"` 会损毁

```python
# 正确的做法：用 Python 生成 .ipynb 文件
import json

cells = [
    {
        "cell_type": "markdown",
        "metadata": {},
        "source": ["# KNN 手写数字识别\n\n使用 scikit-learn 实现 KNN 分类器"]
    },
    {
        "cell_type": "code",
        "metadata": {},
        "source": ["from sklearn.datasets import load_digits\n", "import numpy as np\n"],
        "outputs": [],
        "execution_count": None
    }
]

nb = {
    "nbformat": 4,
    "nbformat_minor": 5,
    "metadata": {},
    "cells": cells
}

with open("output.ipynb", "w", encoding="utf-8") as f:
    json.dump(nb, f, ensure_ascii=False, indent=1)
```

### 阶段三：环境准备

```bash
# 安装依赖（国内网络用清华镜像）
pip install --break-system-packages -i https://pypi.tuna.tsinghua.edu.cn/simple \
  jupyter nbconvert ipykernel \
  numpy scikit-learn pandas matplotlib seaborn

# 注册内核（一次性）
python3 -m ipykernel install --name python3 --user
```

### 阶段四：执行与导出

```bash
# 标准命令
jupyter nbconvert --to html --execute \
  --ExecutePreprocessor.timeout=600 \
  --output 结果文件.html \
  输入文件.ipynb
```

## 已知坑点与解决方法

### 1. 中文引号编码陷阱
`"`（U+201C）和 `"`（U+201D）在 JSON 中必须用 unicode 转义 `\u201c` 和 `\u201d`。
- 用 `write_file` 直接写 JSON 文本：**会损坏**（ASCII 双引号 0x22 冲突）
- 用 Python `json.dump(ensure_ascii=False)`：**正确**
- 修复已损坏的文件：用 Python 加载后重新 dump

### 2. nbconvert 超时
GridSearchCV 等 ML 训练任务可能耗时较长（如 140 次拟合）。
默认超时太短，必须设置：
```
--ExecutePreprocessor.timeout=600
```
建议值：600 秒（10 分钟），可根据任务复杂度调整。

### 3. Matplotlib 中文渲染空白
Debian 默认字体 DejaVu Sans 不包含 CJK 字符。

**方案 A — 安装中文字体：**
```bash
apt-get install -y fonts-noto-cjk
```
并在代码中设置：
```python
import matplotlib.pyplot as plt
plt.rcParams['font.sans-serif'] = ['Noto Sans CJK SC']
plt.rcParams['axes.unicode_minus'] = False
```

**方案 B — 使用英文标签（快速方案）：**
不在图表标题/标签中使用中文，避免字体问题。

### 4. 网络慢（国内网络）
```bash
# 所有 pip 安装都加上清华镜像
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple <包名>
# PEP 668 限制时加 --break-system-packages
```

## 完整示例（KNN 手写数字识别）

### 预期生成的 Notebook 代码格结构

| 格序 | 类型 | 内容 |
|------|------|------|
| 1 | markdown | 任务描述 |
| 2 | code | 导入库（sklearn, numpy, matplotlib, seaborn） |
| 3 | code | 加载 digits 数据集 |
| 4 | code | 划分训练集/测试集 |
| 5 | code | GridSearchCV 搜索 KNN 参数 |
| 6 | code | 输出最佳参数和交叉验证得分 |
| 7 | code | 测试集评估（准确率、分类报告） |
| 8 | code | 绘制混淆矩阵 |
| 9 | code | 绘制错误分类样本 |
| 10 | code | 绘制前 25 个样本预览 |

### 典型结果
- 最佳参数：`n_neighbors=3, p=1 (曼哈顿距离), weights=distance`
- 交叉验证得分：~0.977
- 测试准确率：~0.983
- 最易混淆：数字 8 → 1

## 调试技巧

### 查看文件真实格式
```bash
# 替代 file 命令（Debian 环境可能没有 file/xxd）
head -c 100 /path/to/file | od -A x -t x1z
```

### 快速校验 Notebook JSON
```bash
python3 -c "
import json
nb = json.load(open('notebook.ipynb'))
print(f'Cells: {len(nb[\"cells\"])}')
for i, c in enumerate(nb['cells']):
    print(f'  [{i}] {c[\"cell_type\"]}: {c[\"source\"][:50]}...')
"
```

### 手动执行单格（调试用）
```bash
python3 -c "
exec(compile(''.join(open('/tmp/cell_source.py')), '<cell>', 'exec'))
"
```
