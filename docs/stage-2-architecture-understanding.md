# 阶段二：架构理解 —— 数据流、设计模式与模块协作

> **学习目标**：理解各个模块如何协作、数据和控制的流动路径、背后的设计哲学
> **预计时间**：第 3-5 天
> **前置要求**：已完成阶段一，可以回答所有阶段一的复习题

---

## 先看全局：一个命令从输入到输出的完整旅程

以 `python -m src.main summary` 为例，追踪整个数据流：

```
终端输入: python -m src.main summary
         │
         ▼
┌──────────────────────────────────────────────────────┐
│                   main.py                            │
│                                                      │
│  sys.argv = ['-m', 'src.main', 'summary']            │
│       │                                              │
│       ▼                                              │
│  build_parser()  ← 构建参数解析器                      │
│       │                                              │
│       ▼                                              │
│  parser.parse_args() → Namespace(command='summary')  │
│       │                                              │
│       ▼                                              │
│  build_port_manifest() ─────────────┐                │
│       │                             │                │
│       ▼                             ▼                │
│  from port_manifest.py        rglob('*.py')          │
│       │                      Counter 统计            │
│       ▼                      PortManifest 快照        │
│  manifest = PortManifest(     │                       │
│      src_root=...,            │                       │
│      total_python_files=8,    │                       │
│      top_level_modules=(...)  │                       │
│  ) ◄──────────────────────────┘                       │
│       │                                              │
│       ▼                                              │
│  if args.command == 'summary':                       │
│       │                                              │
│       ▼                                              │
│  QueryEnginePort(manifest) ──────────┐               │
│       │                              │               │
│       ▼                              ▼               │
│  from query_engine.py          build_command_backlog()│
│       │                        build_tool_backlog()  │
│       ▼                              │               │
│  render_summary()                    ▼               │
│       │              commands.py → PORTED_COMMANDS   │
│       │              tools.py    → PORTED_TOOLS      │
│       ▼                                              │
│  返回 Markdown 字符串                                  │
│       │                                              │
│       ▼                                              │
│  print(...) → 终端输出                                │
└──────────────────────────────────────────────────────┘
```

每个箭头都是一次 **"依赖注入"** 或 **"工厂调用"**。没有全局变量，没有单例——数据从头传到尾。

---

## 核心问题逐一拆解

### 问题 1：Command 和 Tool 的区别是什么？为什么要分开注册？

**表面区别**：

| 维度 | Command | Tool |
|------|---------|------|
| 调用者 | 人类用户（CLI/终端） | AI Agent（上下文流） |
| 注册位置 | `commands.py` | `tools.py` |
| 参数方式 | argparse 子命令 | JSON Schema 描述 |
| 返回方式 | stdout 文本 | 结构化的 Tool Result |
| Backlog 标签 | `'Command surface'` | `'Tool surface'` |

**深层原因**——为什么分开：

1. **不同的描述语言**
   - Command 用 `argparse` 的 help 文本描述自己（给人看的）
   - Tool 用 JSON Schema 描述自己的参数（给 LLM 解析的）
   
2. **不同的调用约定**
   - Command: `python -m src.main summary --verbose`
   - Tool: 模型生成 JSON `{"tool": "port_manifest", "params": {}}`，Harness 解析后调用函数

3. **不同的错误处理**
   - Command 的错误直接显示给用户（人类可读的报错）
   - Tool 的错误要返回给 LLM，让模型决定怎么处理（如重试、换工具、向用户解释）

**在当前项目中的体现**：
```python
# commands.py — 给人用的
PORTED_COMMANDS = (
    PortingModule('summary', 'Render a Markdown overview...', ...),
)

# tools.py — 给 Agent 用的  
PORTED_TOOLS = (
    PortingModule('port_manifest', 'Inspect the active Python source tree...', ...),
)
```

同一个底层文件 `query_engine.py` 同时服务于 Command `summary` 和 Tool `query_engine`——这是 Command 和 Tool 共享逻辑但不共享入口的最佳示范。

---

### 问题 2：QueryEnginePort 如何聚合多个数据源？

看代码中的调用链：

```python
def render_summary(self) -> str:
    command_backlog = build_command_backlog()   # ① 数据源 1
    tool_backlog = build_tool_backlog()         # ② 数据源 2
    sections = [
        '# Python Porting Workspace Summary',
        '',
        self.manifest.to_markdown(),            # ③ 数据源 3
        '',
        f'{command_backlog.title}:',            # ① 渲染
        *command_backlog.summary_lines(),
        '',
        f'{tool_backlog.title}:',              # ② 渲染
        *tool_backlog.summary_lines(),
    ]
    return '\n'.join(sections)                  # ④ 合并输出
```

三个数据源的聚合模式：

```
     build_command_backlog()     build_tool_backlog()     self.manifest
            │                         │                       │
            ▼                         ▼                       ▼
     PortingBacklog              PortingBacklog           PortManifest
     (命令面数据)                  (工具面数据)              (文件系统数据)
            │                         │                       │
            ▼                         ▼                       ▼
     .summary_lines()           .summary_lines()         .to_markdown()
            │                         │                       │
            ▼                         ▼                       ▼
      list[str]                   list[str]                 str
            │                         │                       │
            └─────────────────────────┼───────────────────────┘
                                      │
                                      ▼
                           '\n'.join(sections)
                                      │
                                      ▼
                              最终的 Markdown
```

**关键设计原则**：
- QueryEnginePort **不直接**读取文件系统
- QueryEnginePort **不直接**导入 `PORTED_COMMANDS` / `PORTED_TOOLS`
- 它只依赖三个工厂函数的返回值——这是**控制反转（IoC）**

如果将来要增加第四个数据源（比如 `build_hook_backlog()`），只需加一个工厂调用和几行渲染代码，不改动其他模块。

---

### 问题 3：PortingModule 的 status 字段如何影响执行流？

**当前状态**：status 字段目前只是**标记**，不影响执行流。所有模块 status 都是 `'implemented'`。

```python
# 所有现有的注册都是 'implemented'
PortingModule('main', '...', 'src/main.py', 'implemented')
PortingModule('summary', '...', 'src/query_engine.py', 'implemented')
```

**未来可以怎么用**（阶段三会实现）：

```python
# 按状态过滤——只列出已实现的
def filter_implemented(backlog: PortingBacklog) -> PortingBacklog:
    return PortingBacklog(
        title=backlog.title,
        modules=[m for m in backlog.modules if m.status == 'implemented']
    )

# 按状态分组——看整体进度
def group_by_status(backlog: PortingBacklog) -> dict[str, list[PortingModule]]:
    result = {}
    for m in backlog.modules:
        result.setdefault(m.status, []).append(m)
    return result

# 验证——只执行已实现的命令
def execute_command(name: str) -> int:
    for module in PORTED_COMMANDS:
        if module.name == name:
            if module.status != 'implemented':
                print(f'{name} is not yet implemented (status: {module.status})')
                return 1
            # ... execute ...
```

**状态机的设计意图**：
```
planned ──→ in_progress ──→ implemented
    │                          │
    └──────────→ deprecated ←──┘
```

原始 Harness 中的 Tool 就有类似的状态流转——这是企业级软件管理的标配。

---

### 问题 4：如果新增一个子系统，需要修改哪些文件？

场景：你要加一个 `src/validator.py`，实现 `validate` 命令。

**需要修改的文件清单**：

```
① src/validator.py          ← 新建：实现验证逻辑
② src/models.py             ← 可能不需要改（复用现有模型）
③ src/commands.py           ← 添加 PortingModule 注册
④ src/main.py               ← 注册子命令 + 分支
⑤ src/__init__.py           ← 如果需要导出新 API
⑥ tests/test_validator.py   ← 新建：测试新功能
```

详细的改动步骤：

**Step 1: `src/validator.py`（新建）**
```python
from __future__ import annotations

from dataclasses import dataclass
from .port_manifest import PortManifest


@dataclass
class Validator:
    manifest: PortManifest

    def run(self) -> str:
        # 验证逻辑
        issues = []
        for module in self.manifest.top_level_modules:
            if module.file_count == 0:
                issues.append(f'{module.name}: no files')
        return '\n'.join(issues) if issues else 'All modules valid.'
```

**Step 2: `src/commands.py`（修改）**
```python
PORTED_COMMANDS = (
    # ... 已有的保持不变 ...
    PortingModule('validate', 'Validate workspace integrity', 'src/validator.py', 'implemented'),
)
```

**Step 3: `src/main.py`（修改）**
```python
# 在 build_parser() 中添加
subparsers.add_parser('validate', help='validate workspace integrity')

# 在 main() 中添加分支
from .validator import Validator  # 顶部加导入

if args.command == 'validate':
    print(Validator(manifest).run())
    return 0
```

**Step 4: `tests/test_validator.py`（新建）**
```python
class ValidatorTests(unittest.TestCase):
    def test_validate_passes(self):
        manifest = build_port_manifest()
        result = Validator(manifest).run()
        # 当前工作空间应该有效
        self.assertIn('valid', result.lower() or 'valid')
```

**改动影响面分析**：

```
新增功能时必须改动的文件：2-3 个
   ├── 实现文件（validator.py）
   ├── 注册文件（commands.py 或 tools.py）
   └── 路由文件（main.py）

可能改动的文件：1-2 个
   ├── __init__.py（如果要暴露新 API）
   ├── query_engine.py（如果要在 summary 中展示）
   └── tests/（强烈建议加测试）

不需要改动的文件：其余全部
   ├── models.py（数据模型不变）
   ├── port_manifest.py（自动发现新文件）
   └── task.py（任务模型不变）
```

这就是 Harness 模块化的效果——增减功能是"加法"而非"蔓延式修改"。

---

## 深入设计模式

### 模式 1：声明式注册（Declarative Registration）

**传统方式**（命令式）：
```python
registry = CommandRegistry()
registry.register('summary', SummaryCommand())
registry.register('manifest', ManifestCommand())
registry.register('subsystems', SubsystemsCommand())
```

**本项目方式**（声明式）：
```python
PORTED_COMMANDS = (
    PortingModule('summary', '...', 'src/query_engine.py', 'implemented'),
    PortingModule('manifest', '...', 'src/port_manifest.py', 'implemented'),
    PortingModule('subsystems', '...', 'src/port_manifest.py', 'implemented'),
)
```

**声明式的好处**：
1. 注册表即文档——看一眼就知道系统有哪些命令
2. 不依赖执行顺序——元组是有序但不可变的，声明顺序不影响正确性
3. 易于静态分析——可以写工具分析注册表，检查遗漏
4. 惰性加载——声明时不创建实例，只在需要时才构造对象

### 模式 2：工厂方法替代构造函数（Factory Method over Constructor）

```python
# 直接构造（耦合了构建逻辑）
manifest = build_port_manifest()  # 调用者必须知道这个函数
engine = QueryEnginePort(manifest=manifest)

# 工厂方法（封装了构建逻辑）
engine = QueryEnginePort.from_workspace()  # 调用者不需要知道 build_port_manifest
```

**为什么需要两种构造方式？**

| | 构造函数 | 工厂方法 |
|---|---|---|
| 用途 | 依赖注入、测试 | 便利创建 |
| manifest 来源 | 外部传入 | 自动扫描 |
| 测试友好度 | 高（可以注入 mock） | 低（绑定了实现） |

在测试中：
```python
# 单元测试用构造函数注入 mock
mock_manifest = PortManifest(src_root=Path('/fake'), total_python_files=0, top_level_modules=())
engine = QueryEnginePort(manifest=mock_manifest)  # 可控

# 快速原型用工厂方法
engine = QueryEnginePort.from_workspace()  # 方便
```

### 模式 3：数据与行为的分离

观察整个项目的类：
```
纯数据类           含行为的类
─────────         ─────────
Subsystem         PortingBacklog (summary_lines)
PortingModule     PortManifest (to_markdown)
PortingTask       QueryEnginePort (render_summary)
                  上面三个也是"轻行为"——只有渲染/格式化
```

"重"行为（扫描文件系统、解析命令行）放在**模块级函数**中：
- `build_port_manifest(src_root)` — 文件扫描
- `build_parser()` — 参数解析
- `main(argv)` — 命令路由

**为什么这样设计？**
- 数据类可以被序列化、缓存、比较（`frozen=True` 保证了前者）
- 行为函数可以单独测试（传入不同参数，不依赖对象状态）
- 避免了"神类"——一个类包揽所有职责

### 模式 4：管线架构（Pipeline Architecture）

`render_summary()` 是管线的经典案例：

```
输入                         处理                           输出
─────                       ─────                         ─────
(无显式输入，               ① 调 build_command_backlog
 通过工厂函数获取数据)      ② 调 build_tool_backlog        Markdown 字符串
                            ③ 调 manifest.to_markdown
                            ④ 拼接 → join('\n')
```

如果管线继续扩展，可以显式建模：
```python
def build_summary_pipeline(manifest: PortManifest) -> list[str]:
    """每个阶段返回一个字符串片段"""
    stages = [
        ('标题', lambda: '# Python Porting Workspace Summary'),
        ('清单', lambda: manifest.to_markdown()),
        ('命令', lambda: render_backlog(build_command_backlog())),
        ('工具', lambda: render_backlog(build_tool_backlog())),
    ]
    lines = []
    for name, stage_fn in stages:
        lines.append(stage_fn())
        lines.append('')
    return lines
```

---

## 各模块的职责边界（Contract）

| 模块 | 输入 | 输出 | 不能做的事 |
|------|------|------|-----------|
| `models.py` | 无 | 数据类定义 | 不能有文件IO、不能有业务逻辑 |
| `task.py` | 无 | `PortingTask` 类 | 同上 |
| `commands.py` | 无 | `PortingBacklog` | 不能修改注册表 |
| `tools.py` | 无 | `PortingBacklog` | 同上 |
| `port_manifest.py` | `Path \| None` | `PortManifest` | 不能修改文件系统 |
| `query_engine.py` | `PortManifest` | `str` | 不能自己读文件 |
| `main.py` | `list[str] \| None` | `int` | 不能有业务逻辑（只做路由） |
| `__init__.py` | 无 | 公开 API 列表 | 不能有非导入的代码 |

每个模块违反自己边界时，意味着：
- 循环依赖的风险增加
- 单元测试变难
- 修改一个功能要改多个文件

---

## 对比原始 Harness 的架构

当前 Python 工作空间是原始 Harness 的**最小化重实现**。对照关系：

| 原始 TypeScript Harness | 当前 Python 实现 | 简化程度 |
|---|---|---|
| `ToolRegistry`（带 Schema 校验） | `PORTED_TOOLS`（纯元组） | 大幅简化 |
| `CommandRouter`（异步流式） | `main()` 的 if-elif 链 | 大幅简化 |
| `Sandbox`（进程隔离） | 无 | 未实现 |
| `PermissionSystem`（Allow/Deny 矩阵） | 无 | 未实现 |
| `SessionManager`（持久化上下文） | 无 | 未实现 |
| `HookSystem`（生命周期事件） | 无 | 未实现 |
| `PortManifest`（模块自省） | `build_port_manifest()` | ⭐ 已实现 |
| `Backlog/Task`（状态追踪） | `PortingBacklog` / `PortingTask` | ⭐ 已实现 |

当前实现侧重于**可见性层**（让用户看到系统状态），而原始 Harness 侧重于**执行层**（安全地执行用户和 Agent 的请求）。阶段三和阶段四会逐步填补执行层。

---

## 阶段二自检清单

完成阶段二后，你应该能够：

- [ ] 画出 `summary` 命令的完整调用图
- [ ] 解释为什么 `commands.py` 和 `tools.py` 分开但结构和模式相同
- [ ] 说出至少 3 处 "控制反转" 的体现
- [ ] 描述新增一个命令需要改动哪些文件、每个文件改什么
- [ ] 理解 `frozen=True` 和 `@dataclass` 在整个架构中的角色
- [ ] 解释 "管线架构" 和 "传统 MVC" 的不同
- [ ] 知道哪个模块可以改、哪个模块不能乱动
