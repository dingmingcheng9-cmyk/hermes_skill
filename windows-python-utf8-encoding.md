---
name: windows-python-utf8-encoding
description: 修复中文 Windows 上 Python 因默认 GBK 编码导致的 subprocess UnicodeDecodeError——设置 PYTHONUTF8=1 和 PYTHONIOENCODING=utf-8 环境变量，让 Python 使用 UTF-8 处理 I/O
---

# Windows Python UTF-8 编码修复

## 问题

中文 Windows 的系统 ANSI 编码是 **GBK (CP936)**。Python 的 `subprocess` 模块默认使用 `locale.getpreferredencoding()` 读取子进程管道输出。当子进程输出 **UTF-8 字符**（如 npm install 的进度条、git 中文文件名/commit 信息）时，GBK 解码会失败：

```
UnicodeDecodeError: 'gbk' codec can't decode byte 0x85 in position 3:
illegal multibyte sequence
```

触发场景：
- `hermes update` 时 npm 安装依赖
- 任何 `subprocess.run(capture_output=True, text=True)` 且子进程输出 UTF-8 非 ASCII 字符
- Python 脚本读取包含 UTF-8 中文的文件
- pip、uv 等工具安装时的输出

## 解决方案

Python 3.7+ 支持 `PYTHONUTF8=1` 环境变量（[PEP 540](https://peps.python.org/pep-0540/)），设置后 Python 默认使用 UTF-8 处理所有 I/O，包括 subprocess 管道编码和文件读写编码。

### 永久设置（推荐）

```cmd
:: 设置 Python 默认使用 UTF-8
setx PYTHONUTF8 1

:: 可选：额外兜底
setx PYTHONIOENCODING utf-8
```

写的是**用户级环境变量**（HKCU\Environment），只影响当前用户。新命令提示符窗口生效，不需要重启。

### 临时设置（测试用）

```cmd
:: 只在当前窗口生效
set PYTHONUTF8=1
set PYTHONIOENCODING=utf-8
```

### 验证是否生效

```cmd
:: 检查环境变量已设置
reg query HKCU\Environment /v PYTHONUTF8
reg query HKCU\Environment /v PYTHONIOENCODING

:: 通过 Python 检查
python -c "import sys; print(f'utf8_mode={sys.flags.utf8_mode}'); print(f'encoding={sys.getfilesystemencoding()}')"
```

输出应为：
```
utf8_mode=1
encoding=utf-8
```

### 撤销

```cmd
reg delete HKCU\Environment /v PYTHONUTF8 /f
reg delete HKCU\Environment /v PYTHONIOENCODING /f
```

## 影响范围

| 项目 | 有 PYTHONUTF8=1 | 没有（默认 GBK） |
|------|----------------|----------------|
| `open('file.txt').read()` | UTF-8 解码 | GBK 解码 |
| `subprocess.run(text=True)` | UTF-8 读管道 | GBK 读管道（可能崩溃） |
| `sys.getfilesystemencoding()` | `utf-8` | `gbk` |
| 旧脚本硬编码 GBK 的 | 可能出乱码 ⚠️ | 正常 |

> PYTHONUTF8=1 是 Python 官方推荐的标准做法（PEP 540），不会破坏大多数现代 Python 应用。pip、uv、pytest 等主流工具均已支持 UTF-8。如果有老项目依赖 GBK，在项目中 `del os.environ["PYTHONUTF8"]` 即可局部恢复。

## Hermes 场景

当 `hermes update` 运行 npm install 时，npm 输出 UTF-8 文本。PYTHONUTF8=1 让 Python 用 UTF-8 读取管道，编码一致，不再崩溃。

### 代码级兜底（可选）

在 `subprocess.run` 中加 `errors="replace"` 防止坏字节导致崩溃：

```python
subprocess.run(cmd, capture_output=True, text=True,
    encoding="utf-8", errors="replace")
```

## 注意事项

1. **Windows 更新不会覆盖**：setx 写注册表 HKCU\Environment，系统更新不碰这里
2. **不需要管理员权限**：用户级环境变量，普通用户可执行
3. **Python 版本**：PYTHONUTF8 支持 Python 3.7+
4. **不影响系统其他程序**：只影响 Python 进程，Node.js/.NET/Go 等不受影响
5. **git-bash 设不了**：setx 需在 cmd 或 PowerShell 中执行
