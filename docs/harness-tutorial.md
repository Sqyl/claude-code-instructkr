# Claude Code Harness 系统：学习与配置指南

## 前言

本教程基于对 Claude Code 源码的分析整理而成。当前仓库（`instructkr/claude-code-instructkr`）是一个 **Python 移植工作空间**——作者在研究了原始 TypeScript 源码后，出于法律与伦理考量，移除了追踪中的原始快照，转而以 Python 重新实现核心架构。

> _"I originally studied the exposed codebase to understand its harness, tool wiring, and agent workflow."_ —— README

**Harness** 是 Claude Code 最核心的编排系统，负责工具执行、命令路由、Agent 生命周期管理以及沙箱隔离。本教程将剖析 Harness 的设计理念、核心机制，并提供学习路径与配置指南。

---

## 目录

1. [Harness 是什么](#1-harness-是什么)
2. [核心概念与架构](#2-核心概念与架构)
3. [当前项目中的 Harness 线索](#3-当前项目中的-harness-线索)
4. [学习路径](#4-学习路径)
5. [配置指南](#5-配置指南)
6. [扩展阅读](#6-扩展阅读)

---

## 1. Harness 是什么

Harness（直译"马具/缰绳"）是 Claude Code 的 **中央编排层**，位于用户交互界面与底层 AI 模型之间。它扮演三个关键角色：

| 角色 | 说明 |
|------|------|
| **执行引擎** | 解析用户指令，调度工具调用，管理子进程生命周期 |
| **安全边界** | 实施权限控制、沙箱隔离、文件系统访问策略 |
| **生命周期管理器** | 维护对话上下文、任务状态、Agent 工作流阶段 |

### Harness 要解决的核心问题

```
用户输入
    │
    ▼
┌─────────────────────────────────────┐
│            Harness                  │
│  ┌──────────┐  ┌───────────────┐   │
│  │ 安全性   │  │  工具编排     │   │
│  │ ·权限检查 │  │ ·Tool 发现   │   │
│  │ ·沙箱隔离 │  │ ·参数校验    │   │
│  │ ·路径拦截 │  │ ·结果路由    │   │
│  └──────────┘  └───────────────┘   │
│  ┌──────────┐  ┌───────────────┐   │
│  │ 会话管理 │  │  命令系统     │   │
│  │ ·上下文  │  │ ·Command 注册 │   │
│  │ ·状态    │  │ ·参数解析     │   │
│  │ ·持久化  │  │ ·执行调度     │   │
│  └──────────┘  └───────────────┘   │
└─────────────────────────────────────┘
    │
    ▼
  模型 / 工具 / 文件系统
```

---

## 2. 核心概念与架构

### 2.1 两大子系统：Command 与 Tool

Harness 将外部可调用接口分为两个清晰的类别：

#### Command（命令）
- **用途**：用户直接调用的操作（CLI 入口）
- **特点**：有参数解析、独立执行入口
- **示例**：`summary`、`manifest`、`subsystems`
- **代码映射**：`src/commands.py`

```python
# 来自当前项目的命令注册模式
PORTED_COMMANDS = (
    PortingModule('main', 'CLI 入口点', 'src/main.py', 'implemented'),
    PortingModule('summary', '渲染移植现状报告', 'src/query_engine.py', 'implemented'),
    PortingModule('subsystems', '列出 Python 模块', 'src/port_manifest.py', 'implemented'),
)
```

#### Tool（工具）
- **用途**：Agent 在上下文中调用的能力单元
- **特点**：面向 LLM 的函数式接口、结果回馈到对话流
- **示例**：`port_manifest`、`backlog_models`、`query_engine`
- **代码映射**：`src/tools.py`

```python
# 来自当前项目的工具注册模式
PORTED_TOOLS = (
    PortingModule('port_manifest', '检视源码树并汇总移植现状', 'src/port_manifest.py', 'implemented'),
    PortingModule('backlog_models', '子系统与待办元数据模型', 'src/models.py', 'implemented'),
    PortingModule('query_engine', '协调移植报告生成', 'src/query_engine.py', 'implemented'),
)
```

### 2.2 三层数据模型

Harness 的核心数据模型分为三层：

```
PortingBacklog（待办清单）
    ├── title: str              # 类别名称
    └── modules: list[PortingModule]  # 模块列表
            ├── name: str           # 模块名
            ├── responsibility: str # 职责描述
            ├── source_hint: str    # 源文件路径
            └── status: str         # 状态: planned / in_progress / implemented
```

这种模式本质上是 Harness **注册-发现-执行**管线的简化映射：

| Harness 概念 | 本项目映射 | 作用 |
|---|---|---|
| 注册中心 | `PORTED_COMMANDS` / `PORTED_TOOLS` | 声明可用能力 |
| 模块描述 | `PortingModule` | 元数据与状态追踪 |
| 编排层 | `QueryEnginePort` | 聚合各模块输出 |
| 入口点 | `main.py` + `build_parser()` | 命令路由 |

### 2.3 查询引擎（Query Engine）

`QueryEnginePort` 是 Harness 中"编排层"的缩影——它不直接执行工作，而是：

1. **聚合**：从多个 Backlog 收集数据
2. **转换**：将结构化数据渲染为用户可读的格式
3. **呈现**：输出 Markdown / 文本报告

```python
@dataclass
class QueryEnginePort:
    manifest: PortManifest

    def render_summary(self) -> str:
        command_backlog = build_command_backlog()
        tool_backlog = build_tool_backlog()
        # 聚合命令面 + 工具面 + 元数据 → 统一输出
```

这是 Harness 编排哲学的体现：**关注点分离，通过组合而非继承来构建工作流**。

---

## 3. 当前项目中的 Harness 线索

虽然原始 TypeScript 快照已从仓库中移除，但当前 Python 代码库处处留有 Harness 的设计痕迹：

### 3.1 从 `src/main.py` 看 CLI Harness

```python
def build_parser() -> argparse.ArgumentParser:
    # 命令注册——Harness 的命令路由表
    subparsers.add_parser('summary', help='生成移植工作空间 Markdown 报告')
    subparsers.add_parser('manifest', help='打印当前 Python 工作空间清单')
    list_parser = subparsers.add_parser('subsystems', help='列出 Python 模块')

def main(argv=None) -> int:
    args = parser.parse_args(argv)
    manifest = build_port_manifest()  # 状态初始化
    if args.command == 'summary':
        print(QueryEnginePort(manifest).render_summary())  # 工具调用
```

这是 Harness 的简化实现：**解析 → 路由 → 执行 → 输出**。

### 3.2 从 `src/port_manifest.py` 看文件系统抽象

```python
def build_port_manifest(src_root=None) -> PortManifest:
    root = src_root or DEFAULT_SRC_ROOT
    files = [path for path in root.rglob('*.py') if path.is_file()]
    # 扫描文件系统，构建工作空间视图
```

Harness 中类似地需要对项目结构进行**自省（introspection）**，以确定可用工具和命令集。

### 3.3 从 `src/task.py` 看任务模型

```python
@dataclass(frozen=True)
class PortingTask:
    title: str
    detail: str
    completed: bool = False
```

这是 Harness 中 Task 系统的最简原型——跟踪工作单元的状态。

---

## 4. 学习路径

要深入理解 Claude Code Harness，建议按以下路径学习：

### 阶段一：基础概念（第 1-2 天）

| 主题 | 学习内容 | 关联文件 |
|------|---------|---------|
| 命令模式 | CLI 入口点、参数解析、子命令 | `src/main.py`, `src/commands.py` |
| 工具抽象 | 功能注册、元数据描述、状态管理 | `src/tools.py`, `src/models.py` |
| 数据流 | 输入→处理→输出的管线设计 | `src/query_engine.py` |

**练习**：
- 运行 `python -m src.main summary` 观察输出
- 运行 `python -m src.main manifest` 查看模块清单
- 在 `commands.py` 中添加一个新命令

### 阶段二：架构理解（第 3-5 天）

核心问题列表，带着问题去阅读代码：

```
1. Command 和 Tool 的区别是什么？为什么需要分开注册？
2. QueryEnginePort 如何聚合多个数据源？
3. PortingModule 的 status 字段如何影响执行流？
4. 如果新增一个子系统，需要修改哪些文件？
```

### 阶段三：动手实现（第 6-10 天）

从当前项目出发，逐步扩展 Harness 功能：

```
Step 1: 添加命令执行结果缓存（在 QueryEnginePort 中）
Step 2: 实现工具链的依赖解析（Tool A → Tool B）
Step 3: 添加权限检查层（Allow/Deny 列表）
Step 4: 实现沙箱文件系统（限制工具可访问的路径）
Step 5: 构建会话持久化（保存/恢复任务状态）
```

### 阶段四：深入原始架构（第 11 天后）

当理解了基础模式后，可以尝试：

- 研究 Anthropic 的 [Claude API](https://docs.anthropic.com/en/docs) 官方文档，了解 Tool Use 协议
- 对比当前 Python 实现与原始 TypeScript 架构的异同
- 理解 Harness 的安全模型：权限提示、路径白名单、沙箱隔离

---

## 📚 各阶段详细教学文档

每个阶段都有独立的逐行阅读 + 注释文档，建议按顺序阅读：

| 阶段 | 文档 | 内容 |
|------|------|------|
| 阶段一 | [stage-1-basic-concepts.md](stage-1-basic-concepts.md) | 逐行精读全部 9 个源文件 |
| 阶段二 | [stage-2-architecture-understanding.md](stage-2-architecture-understanding.md) | 数据流追踪、设计模式、模块边界 |
| 阶段三 | [stage-3-hands-on-implementation.md](stage-3-hands-on-implementation.md) | 5 个动手项目：缓存→依赖→权限→沙箱→持久化 |
| 阶段四 | [stage-4-deep-dive.md](stage-4-deep-dive.md) | Claude API Tool Use 协议、安全模型、架构对比 |

---

## 5. 配置指南

Harness 系统的可配置点（基于当前项目可扩展的方向）：

### 5.1 命令注册与路由

在 `src/commands.py` 中配置：

```python
# 添加新命令
PORTED_COMMANDS = (
    # ... 已有命令 ...
    PortingModule('validate', '验证工作空间完整性', 'src/validator.py', 'planned'),
)

# 在 main.py 中注册路由
subparsers.add_parser('validate', help='验证工作空间完整性')
# 在 main 函数中添加分支
if args.command == 'validate':
    print(Validator(manifest).run())
```

### 5.2 工具注册

在 `src/tools.py` 中配置：

```python
PORTED_TOOLS = (
    # ... 已有工具 ...
    PortingModule('code_analyzer', '静态分析代码质量', 'src/code_analyzer.py', 'planned'),
    PortingModule('test_runner', '运行测试套件', 'src/test_runner.py', 'planned'),
)
```

### 5.3 数据模型扩展

在 `src/models.py` 中配置：

```python
# 添加权限模型
@dataclass(frozen=True)
class Permission:
    tool_name: str
    allow_paths: tuple[str, ...]
    deny_paths: tuple[str, ...]
    requires_confirmation: bool = True

# 添加会话模型
@dataclass
class Session:
    session_id: str
    context: dict
    task_queue: list[PortingTask]
    permissions: dict[str, Permission]
```

### 5.4 环境变量配置

| 变量 | 用途 | 示例值 |
|------|------|--------|
| `SRC_ROOT` | 源码根目录覆盖 | `/path/to/project` |
| `HARNESS_DEBUG` | 启用调试输出 | `1` |
| `HARNESS_LOG_LEVEL` | 日志级别 | `INFO`, `DEBUG`, `WARN` |
| `HARNESS_ALLOWED_TOOLS` | 允许的工具白名单 | `manifest,summary` |
| `HARNESS_DENIED_PATHS` | 禁止访问的文件路径 | `/etc,/sys` |

### 5.5 Harness 钩子系统（Hook System）

Claude Code Harness 的重要特性之一是**钩子（Hook）系统**，允许用户在特定事件点注入自定义行为：

```
事件钩子模型：
├── before_command   → 命令执行前触发
├── after_command    → 命令执行后触发
├── before_tool_call → 工具调用前触发（可用于权限校验）
├── after_tool_call  → 工具调用后触发（可用于结果记录）
└── on_error         → 错误发生时触发
```

在当前架构中可以这样实现：

```python
# harvester/hooks.py 概念示例
from dataclasses import dataclass
from typing import Callable

@dataclass
class HarnessHooks:
    before_command: list[Callable] = None
    after_command: list[Callable] = None
    before_tool_call: list[Callable] = None
    after_tool_call: list[Callable] = None
    on_error: list[Callable] = None

    def execute_before_command(self, command_name: str):
        for hook in (self.before_command or []):
            hook(command_name)
```

---

## 6. 扩展阅读

### 项目文档
- [项目 README](../README.md) — 了解项目背景与移植现状
- [Python 移植清单](../src/port_manifest.py) — 当前模块状态

### 原始 Claude Code 架构
- [Anthropic Claude API 文档](https://docs.anthropic.com/en/docs) — Tool Use 与 Message API
- [Claude Code CLI 工具](https://docs.anthropic.com/en/docs/claude-code) — 官方文档

### 相关论文与文章
- [*Is legal the same as legitimate*](../2026-03-09-is-legal-the-same-as-legitimate-ai-reimplementation-and-the-erosion-of-copyleft.md) — 关于 AI 重实现与版权法边界的分析文章

### 学习资源
- Python `argparse` 官方文档 — CLI 命令模式基础
- Python `dataclasses` 官方文档 — 数据建模基础
- [oh-my-codex (OmX)](https://github.com/Yeachan-Heo/oh-my-codex) — 本项目使用的 AI 辅助工作流工具

---

## 附录：术语表

| 术语 | 说明 |
|------|------|
| Harness | Claude Code 的中央编排层，管理工具执行、安全策略与生命周期 |
| Command | 用户直接调用的 CLI 命令，有独立参数解析 |
| Tool | Agent 在上下文中调用的能力单元，结果回馈到对话流 |
| Backlog | 待办清单，跟踪模块移植状态 |
| Manifest | 工作空间清单，描述源码树结构 |
| Query Engine | 编排查询层，聚合多源数据并渲染输出 |
| Porting | 将原始 TypeScript 架构移植到 Python 的过程 |
| Hook | 事件钩子，在特定生命周期点注入自定义行为 |

---

> **注意**：本教程基于对当前仓库 Python 移植工作空间的分析。原始的 Claude Code TypeScript 源码快照已从 Git 历史中移除（详见 README）。本教程旨在提取 Harness 的设计模式，为学习和重实现提供参考，而非替代原始文档。
