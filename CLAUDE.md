# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

蛐蛐 (QuQu) 是一个基于 FunASR 和可配置 AI 模型的中文语音转文字桌面应用，Wispr Flow 的开源替代方案。

**技术栈**: Electron + React 19 + Tailwind CSS 4 + FunASR (Python) + better-sqlite3

## 常用命令

```bash
# 开发
pnpm run dev              # 同时运行 Vite 渲染进程和 Electron 主进程
pnpm run dev:main         # 仅运行 Electron 主进程（Vite 需单独启动）
pnpm run dev:renderer     # 从 src/ 目录启动 Vite（不是根目录）

# 构建
pnpm run build:renderer   # 构建 React 渲染进程（任何 Electron 构建前必须执行）
pnpm run build:win         # Windows 构建（含嵌入式 Python 准备）
pnpm run build:mac         # macOS 构建（含嵌入式 Python 准备）
pnpm run build:linux       # Linux 构建（含嵌入式 Python 准备）

# Python 环境（开发模式）
pnpm run prepare:python:uv        # 使用 uv 管理 Python 环境（推荐）
pip install funasr modelscope torch torchaudio librosa numpy
python download_models.py

# 嵌入式 Python（生产构建）
pnpm run prepare:python          # 准备嵌入式 Python 环境
pnpm run test:python              # 测试嵌入式 Python 环境

# 其他
pnpm run lint                     # ESLint 检查
pnpm run clean                    # 清理构建文件和 python/ 目录
```

## 架构概览

### 主进程 (main.js)
Electron 主进程管理所有系统级功能，包括窗口、IPC、Python 子进程和系统托盘。

**关键管理器（src/helpers/）**:
- `funasrManager.js` - FunASR Python 服务器生命周期管理（启动/停止/通信）
- `ipcHandlers.js` - 所有 IPC 处理器集中注册，支持 AI 文本处理和模型管理
- `windowManager.js` - 多窗口管理（主窗口、控制面板、历史记录、设置）
- `hotkeyManager.js` - 全局快捷键注册（F2 双击、录音触发）
- `database.js` - better-sqlite3 数据库操作（转录记录、设置存储）
- `clipboard.js` - 剪贴板操作（文本粘贴、权限管理）
- `logManager.js` - JSON 格式日志管理（应用日志 + FunASR 日志分离）

**IPC 通信约定**: 所有渲染进程通过 `preload.js` 暴露的 `window.electronAPI` 与主进程通信，禁止直接使用 `ipcRenderer`。

### 渲染进程 (src/)
React 19 应用，三个独立入口点：
- `src/index.html` → `App.jsx` - 主应用（录音转录）
- `src/history.html` → `history.jsx` - 历史记录窗口
- `src/settings.html` → `settings.jsx` - 设置窗口

**状态管理**: 无外部状态库，通过 React Hooks + IPC 与主进程同步状态。

**自定义 Hooks（src/hooks/）**:
- `useRecording.js` - 录音状态管理
- `useModelStatus.js` - FunASR 模型状态
- `useHotkey.js` - 快捷键事件订阅
- `usePermissions.js` - 系统权限检查
- `useTextProcessing.js` - AI 文本处理调用

### FunASR 后端 (Python)
- `funasr_server.py` - Python 子进程，通过 stdin/stdout JSON 通信
- `download_models.py` - 模型下载脚本（Paraformer-large、FSMN-VAD、CT-Transformer）

**Python 进程启动约定**:
- 开发模式: 优先使用嵌入式 Python → 降级到系统 Python
- 生产模式: 仅使用嵌入式 Python（`python/bin/python3.11`）
- 音频文件在系统临时目录创建，不在项目目录

## 嵌入式 Python 环境

基于 python-build-standalone 的完全隔离运行时：
- Python 3.11.6，依赖: numpy<2, torch==2.0.1, torchaudio==2.0.2, librosa>=0.11.0, funasr>=1.2.7
- 路径: `python/bin/python3.11`（开发）或 `process.resourcesPath/app.asar.unpacked/python/bin/python3.11`（生产）
- 环境变量完全隔离（PYTHONHOME、PYTHONPATH、LD_LIBRARY_PATH 等）

## 关键路径解析

| 路径类型 | 开发模式 | 生产模式 |
|---------|---------|---------|
| Vite 配置 | `src/` | `src/dist/` |
| Python 脚本 | 项目根目录 | `app.asar.unpacked/` |
| 嵌入式 Python | `python/bin/python3.11` | `process.resourcesPath/...` |
| 模型缓存 | `~/.cache/modelscope/` | 同上 |
| 日志文件 | 用户数据目录 | 用户数据目录 |

## 日志规范

- 必须使用 `src/helpers/logManager.js`，禁止直接使用 `console.log`
- 应用日志和 FunASR 日志分离存储
- 使用 `logger.info/error/warn/debug` 方法
- FunASR 专用日志: `logger.logFunASR()`

## 窗口管理

- 主窗口: 主要录音/转录界面
- 控制面板: 浮动控制窗口（独立 BrowserWindow）
- 历史窗口: `src/history.html` 入口
- 设置窗口: `src/settings.html` 入口
- 所有窗口通过 `preload.js` 使用相同的 contextBridge API

## 数据库架构

- 引擎: better-sqlite3（同步 API）
- 转录表: `raw_text`（FunASR 原始）+ `processed_text`（AI 优化后）
- 设置表: 键值对，JSON 序列化存储

## 构建注意事项

1. 所有 Electron 构建命令（`build:mac` 等）自动执行 `prepare:python:embedded` 和 `build:renderer`
2. `build:renderer` 必须在任何构建命令前执行
3. 打包包含完整 Python 运行时（约 1GB+）
4. macOS 支持代码签名和公证
