# 阶段一：基础概念 —— 逐行阅读 Harness 源码

> **学习目标**：理解 CLI 命令模式、工具抽象、数据模型和查询引擎的每一行代码
> **预计时间**：第 1-2 天
> **涉及文件**：`src/main.py` · `src/commands.py` · `src/tools.py` · `src/models.py` · `src/query_engine.py` · `src/port_manifest.py` · `src/task.py` · `src/__init__.py` · `tests/test_porting_workspace.py`

---

## 阅读顺序建议

```
src/models.py          ← 先看数据，数据是骨架
src/task.py            ← 最简任务模型
src/commands.py        ← 命令注册（依赖 models）
src/tools.py           ← 工具注册（依赖 models）
src/port_manifest.py   ← 工作空间清单（依赖 models）
src/query_engine.py    ← 编排查询（依赖 commands + tools + port_manifest）
src/main.py            ← CLI 入口（依赖以上全部）
src/__init__.py        ← 包的公开出口
tests/                 ← 验证一切如何运转
```

---

## 1. src/models.py —— 数据模型的基石

这个文件只有 32 行，但它定义了整个 Harness 的类型体系。在 Harness 中，**数据先于逻辑**——先定义"系统里有什么东西"，再定义"这些东西能做什么"。

```python
# 第 1 行
from __future__ import annotations
```

**注释**：`from __future__ import annotations` 是 Python 3.7+ 的特性。它让所有类型注解延迟求值（lazy evaluation），写成字符串形式。这解决了两个问题：
- **前向引用**：`PortingBacklog` 可以引用 `PortingModule`，即使 `PortingModule` 在后面才定义
- **性能**：类型注解在运行时不会被真正执行，只在类型检查时使用

在 Harness 的所有文件中，这行是标准开头——它保证了模块间的循环引用不会崩溃。

---

```python
# 第 3 行
from dataclasses import dataclass, field
```

**注释**：`dataclass` 是 Python 3.7 引入的宝藏。它自动生成 `__init__`、`__repr__`、`__eq__` 等样板代码。在 Harness 风格的系统中，绝大多数对象都是"带类型的属性包"——你不想要行为，只想要结构。dataclass 就是这个场景的最佳选择。

- `field` 用于给字段设置默认值（特别是 `list` 等可变默认值）

---

```python
# 第 6-10 行
@dataclass(frozen=True)
class Subsystem:
    name: str
    path: str
    file_count: int
    notes: str
```

**注释**：`Subsystem` 代表"一个代码子系统"——即源码树中的一个目录。

逐字段解读：
| 字段 | 类型 | 含义 |
|------|------|------|
| `name` | `str` | 子系统名称（如 `__init__.py`、`main.py`） |
| `path` | `str` | 在 src/ 下的路径（如 `src/main.py`） |
| `file_count` | `int` | 此子系统包含的 Python 文件数量 |
| `notes` | `str` | 人类可读的描述 |

**为什么用 `frozen=True`？**
`frozen=True` 让 dataclass 变为不可变（immutable）——一经创建就不能修改。这模仿了函数式编程中的"值对象"模式：
- 不需要跟踪状态变化，减少 bug
- 可以安全地作为字典 key
- 多线程安全

在 Harness 中，大量的元数据对象都是 frozen dataclass。

---

```python
# 第 13-19 行
@dataclass(frozen=True)
class PortingModule:
    name: str
    responsibility: str
    source_hint: str
    status: str = 'planned'
```

**注释**：`PortingModule` 是 Harness 注册系统的核心。它描述了一个"可注册模块"——既可以是一个 Command，也可以是一个 Tool。

逐字段解读：
| 字段 | 类型 | 默认值 | 含义 |
|------|------|--------|------|
| `name` | `str` | 无 | 模块唯一标识 |
| `responsibility` | `str` | 无 | 模块职责描述（人类语言） |
| `source_hint` | `str` | 无 | 对应的源文件路径 |
| `status` | `str` | `'planned'` | 移植状态 |

**状态机（隐式定义）**：
```
planned → in_progress → implemented
```

这是最简单的三态工作流。在原始 Claude Code 的 Harness 中，状态机会更复杂（如工具注册→启用→禁用→废弃），但本质相同。

**为什么要 `frozen=True`？**
模块的元数据在注册时确定，之后不应该改变。如果状态需要改变，应该创建新的实例而不是修改旧的——这是不可变数据流的哲学。

---

```python
# 第 22-31 行
@dataclass
class PortingBacklog:
    title: str
    modules: list[PortingModule] = field(default_factory=list)

    def summary_lines(self) -> list[str]:
        return [
            f'- {module.name} [{module.status}] — {module.responsibility} (from {module.source_hint})'
            for module in self.modules
        ]
```

**注释**：`PortingBacklog` 是一个**聚合容器**。注意它和上面两者的不同——**不是 frozen 的**。

逐行解读：

`title: str` — 类别名称，如 `'Command surface'` 或 `'Tool surface'`。这让同一个结构可以表达多种注册表。

`modules: list[PortingModule] = field(default_factory=list)` — 这是关键细节：
- **为什么不用 `= []`？** Python 的默认参数在函数定义时计算一次，如果写 `= []`，所有实例会共享同一个 list 对象！这是经典的 Python 陷阱。`field(default_factory=list)` 让每次实例化都新建一个 list。
- 在原始 Harness 中，模块注册表正是这样实现的——每个 Backlog 管理一组已注册模块。

`summary_lines(self) -> list[str]` — 这是 `PortingBacklog` 的唯一方法。它把结构化数据渲染为人类可读的列表：
```python
f'- {module.name} [{module.status}] — {module.responsibility} (from {module.source_hint})'
# 输出示例：
# - main [implemented] — Expose a Python CLI for manifest and backlog reporting (from src/main.py)
# - summary [implemented] — Render a Markdown overview of the current porting workspace (from src/query_engine.py)
```

**设计模式观察**：`PortingBacklog` 不直接实现复杂的查询逻辑。它只负责"持有数据"和"基本格式化"。复杂的查询交给 `QueryEnginePort` 做——这是单一职责原则的体现。

---

### models.py 小结

```
Subsystem (frozen dataclass)     ← 文件系统抽象
PortingModule (frozen dataclass) ← 注册表条目
PortingBacklog (mutable)         ← 聚合容器
```

这三个类构成了 Harness 数据模型的三层：
1. **Subsystem** 代表"物理结构"（文件系统）
2. **PortingModule** 代表"逻辑结构"（功能模块）
3. **PortingBacklog** 代表"组织维度"（按类别分组）

---

## 2. src/task.py —— 最简任务模型

```python
# 第 1-10 行
from __future__ import annotations

from dataclasses import dataclass


@dataclass(frozen=True)
class PortingTask:
    title: str
    detail: str
    completed: bool = False
```

**注释**：这 10 行代码是 Harness Task 系统的最简原型。

| 字段 | 类型 | 默认值 | 含义 |
|------|------|--------|------|
| `title` | `str` | 无 | 任务标题 |
| `detail` | `str` | 无 | 任务详细描述 |
| `completed` | `bool` | `False` | 是否完成 |

**为什么如此简单？** 在 Harness 中，Task 不需要内置复杂的执行逻辑。它只是一个"状态标记"——其他组件（执行引擎、调度器）读取和更新任务状态。这是**贫血模型（anemic model）** 的体现：数据对象保持简单，行为放在外部服务中。

**`frozen=True` 的含义**：一旦创建，`title` 和 `detail` 不可变。但 `completed` 怎么办？不能修改的话，任务如何标记完成？答案：**创建新实例**：
```python
# 不可变更新模式
task = PortingTask(title="fix bug", detail="...")
# 完成后创建一个新实例
task = PortingTask(title=task.title, detail=task.detail, completed=True)
```

在原始的 TypeScript Harness 中，状态更新通过 immutability helpers 或 Redux-style reducers 实现——正是为了支持可预测的状态管理。

---

## 3. src/commands.py —— 命令注册表

```python
# 第 1-3 行
from __future__ import annotations

from .models import PortingBacklog, PortingModule
```

**注释**：从 `models` 导入需要的类型。相对导入（`from .models`）说明这是包内的模块依赖。

---

```python
# 第 5-9 行
PORTED_COMMANDS = (
    PortingModule('main', 'Expose a Python CLI for manifest and backlog reporting', 'src/main.py', 'implemented'),
    PortingModule('summary', 'Render a Markdown overview of the current porting workspace', 'src/query_engine.py', 'implemented'),
    PortingModule('subsystems', 'List the current Python modules participating in the rewrite', 'src/port_manifest.py', 'implemented'),
)
```

**注释**：`PORTED_COMMANDS` 是一个**元组（tuple）**——不是 list。这是有意为之：
- 元组是不可变的，意味着命令注册表在运行时不会被意外修改
- 在 Harness 中这叫"声明式注册"——在模块加载时一次性定义所有入口

三条注册记录解读：

| name | responsibility | source_hint | status |
|------|---------------|-------------|--------|
| `main` | CLI 入口点 | `src/main.py` | implemented |
| `summary` | Markdown 报告渲染 | `src/query_engine.py` | implemented |
| `subsystems` | 模块列表 | `src/port_manifest.py` | implemented |

**为什么不放在同一个文件里？** 因为原始 Claude Code 的 Harness 中，Commands 和 Tools 是完全独立的子系统：
- Commands 是**用户面对**的（CLI 入口）
- Tools 是**Agent 面对**的（运行时能力）

把它们分开注册，体现了**关注点分离**。

---

```python
# 第 12-13 行
def build_command_backlog() -> PortingBacklog:
    return PortingBacklog(title='Command surface', modules=list(PORTED_COMMANDS))
```

**注释**：这个简单的工厂函数做了两件事：
1. 将元组转为 list（因为 `PortingBacklog.modules` 的类型是 `list[PortingModule]`）
2. 打上标签 `'Command surface'`

**为什么需要工厂函数？** 而不是直接暴露一个全局变量？因为这给了**组合的入口点**——如果将来需要在构建 Backlog 时做过滤、排序、动态加载，改这个函数即可，调用者不受影响。

---

## 4. src/tools.py —— 工具注册表

```python
# 第 1-3 行
from __future__ import annotations

from .models import PortingBacklog, PortingModule
```

与 `commands.py` 完全相同的导入结构——这是约定胜于配置。

---

```python
# 第 5-9 行
PORTED_TOOLS = (
    PortingModule('port_manifest', 'Inspect the active Python source tree and summarize the current rewrite surface', 'src/port_manifest.py', 'implemented'),
    PortingModule('backlog_models', 'Represent subsystem and backlog metadata as Python dataclasses', 'src/models.py', 'implemented'),
    PortingModule('query_engine', 'Coordinate Python-facing rewrite summaries and reporting', 'src/query_engine.py', 'implemented'),
)
```

| name | responsibility | source_hint | status |
|------|---------------|-------------|--------|
| `port_manifest` | 源码树检视与汇总 | `src/port_manifest.py` | implemented |
| `backlog_models` | 元数据 dataclass 模型 | `src/models.py` | implemented |
| `query_engine` | 协调报告生成 | `src/query_engine.py` | implemented |

---

```python
# 第 12-13 行
def build_tool_backlog() -> PortingBacklog:
    return PortingBacklog(title='Tool surface', modules=list(PORTED_TOOLS))
```

与 `build_command_backlog()` 完全对称。这种对称不是巧合——Commands 和 Tools 在 Harness 中遵循**相同的注册模式**，只是面向不同的调用者。

---

## 5. src/port_manifest.py —— 工作空间自省

这是 Harness 中"文件系统感知"的核心。它不依赖任何配置文件来了解项目结构——而是直接扫描源码树。

```python
# 第 1-9 行
from __future__ import annotations

from collections import Counter
from dataclasses import dataclass
from pathlib import Path

from .models import Subsystem

DEFAULT_SRC_ROOT = Path(__file__).resolve().parent
```

逐行解读：

`from collections import Counter`
- `Counter` 是一个特殊的字典，对可哈希对象计数。这里用来统计每个目录下的文件数量。

`from pathlib import Path`
- `pathlib.Path` 是 Python 3.4+ 的现代路径库。相比 `os.path`，它是面向对象的，而且跨平台。在 Windows bash 环境下尤其重要——Path 自动处理 `/` 和 `\` 的转换。

`DEFAULT_SRC_ROOT = Path(__file__).resolve().parent`
- `__file__` = `src/port_manifest.py` 的绝对路径
- `.resolve()` = 消除所有符号链接，得到真实路径
- `.parent` = 文件所在目录，即 `src/`
- **效果**：默认以 `src/` 为扫描根目录
- **为什么用 `resolve()`？** 在 symlink/worktree 场景下，如果不用 resolve，可能会扫描到错误的物理路径。

---

```python
# 第 12-17 行
@dataclass(frozen=True)
class PortManifest:
    src_root: Path
    total_python_files: int
    top_level_modules: tuple[Subsystem, ...]
```

| 字段 | 类型 | 含义 |
|------|------|------|
| `src_root` | `Path` | 扫描的根目录 |
| `total_python_files` | `int` | 扫描到的 Python 文件总数 |
| `top_level_modules` | `tuple[Subsystem, ...]` | 顶层模块列表 |

注意 `top_level_modules` 也是 tuple——不可变的。Manifest 一旦生成就是快照（snapshot）。

---

```python
# 第 19-27 行
def to_markdown(self) -> str:
    lines = [
        f'Port root: `{self.src_root}`',
        f'Total Python files: **{self.total_python_files}**',
        '',
        'Top-level Python modules:',
    ]
    for module in self.top_level_modules:
        lines.append(f'- `{module.name}` ({module.file_count} files) — {module.notes}')
    return '\n'.join(lines)
```

**注释**：`to_markdown()` 是"渲染管线"的第一步——把结构化数据转换为 Markdown 文本。输出样例：

```markdown
Port root: `D:\programming\claude-code-instructkr\src`
Total Python files: **8**

Top-level Python modules:
- `commands.py` (1 files) — command backlog metadata
- `tools.py` (1 files) — tool backlog metadata
- `__init__.py` (1 files) — package export surface
- ...
```

---

```python
# 第 30-52 行 — 核心扫描逻辑
def build_port_manifest(src_root: Path | None = None) -> PortManifest:
    root = src_root or DEFAULT_SRC_ROOT
    files = [path for path in root.rglob('*.py') if path.is_file()]
```

逐行注解：

`src_root: Path | None = None`
- 参数类型是 `Path | None`——Python 3.10+ 的联合类型语法。允许调用者覆盖默认根目录。

`root = src_root or DEFAULT_SRC_ROOT`
- 如果调用者没传 src_root（None），则用默认的 `src/`。这是典型的"覆盖模式"——便于测试时注入不同路径。

`root.rglob('*.py')`
- `rglob` = recursive glob，递归扫描所有子目录。等同于 `**/*.py`。
- 返回一个生成器（generator）——惰性求值，不会一次性把所有文件路径加载到内存。

`if path.is_file()`
- 过滤掉目录（虽然 `.py` 结尾的目录不太可能存在，但防御性编程）。

---

```python
    counter = Counter(
        path.relative_to(root).parts[0] if len(path.relative_to(root).parts) > 1 else path.name
        for path in files
        if path.name != '__pycache__'
    )
```

这是整个项目最精妙的三行代码。逐层拆解：

**第一层：`path.relative_to(root)`**
```python
# path = D:\programming\claude-code-instructkr\src\commands.py
# root = D:\programming\claude-code-instructkr\src
# result = Path('commands.py')  # 相对路径
```

**第二层：`.parts[0]` vs `path.name`**
```python
# 对于顶层文件:   commands.py  → .parts == ('commands.py',)   → .parts[0] == 'commands.py'
# 对于包内文件:   foo/bar.py    → .parts == ('foo', 'bar.py') → .parts[0] == 'foo'
```

但是这里有个特殊情况——对于顶层文件（如 `commands.py`），`.parts` 只有一个元素。代码的处理：
```python
if len(path.relative_to(root).parts) > 1:
    # 有子目录，取第一级目录名
    use parts[0]  # 归类到子目录
else:
    # 顶层文件，直接用文件名
    use path.name  # 保持为文件名
```

这样做的效果是：`src/commands.py` 归类为 `commands.py`，而 `src/tests/test_x.py` 归类为 `tests`。

**第三层：`if path.name != '__pycache__'`**
- 过滤掉 `__pycache__` 文件。`__pycache__` 是 Python 的字节码缓存目录。`path.name` 返回路径的最后一个组件——所以只有当文件恰好叫 `__pycache__` 时才会被过滤。实际上 Python 的 `.pyc` 文件因为后缀不是 `.py`（而是 `.pyc`），根本不会被 `rglob('*.py')` 匹配到。

**Counter 的默认行为**：计数相同的 key 合并。最终的效果是：
```python
Counter({'__init__.py': 1, 'main.py': 1, 'commands.py': 1, 'tools.py': 1, 'models.py': 1, 'port_manifest.py': 1, 'query_engine.py': 1, 'task.py': 1})
```

--- 

```python
    notes = {
        '__init__.py': 'package export surface',
        'main.py': 'CLI entrypoint',
        'port_manifest.py': 'workspace manifest generation',
        'query_engine.py': 'port orchestration summary layer',
        'commands.py': 'command backlog metadata',
        'tools.py': 'tool backlog metadata',
        'models.py': 'shared dataclasses',
        'task.py': 'task-level planning structures',
    }
```

**注释**：这是硬编码的文件注释映射。它展示了 Harness 的一个设计折中：
- **理想情况**：每个文件内部有自描述元数据（如 docstring）
- **实际情况**：扫描文件内部成本高，在移植初期直接硬编码是务实选择

在原始 Harness 中，Tool 的描述存储在 Tool 定义本身（Schema 的 `description` 字段）——那是更好的方案，因为描述和实现在一起。

---

```python
    modules = tuple(
        Subsystem(name=name, path=f'src/{name}', file_count=count, notes=notes.get(name, 'Python port support module'))
        for name, count in counter.most_common()
    )
    return PortManifest(src_root=root, total_python_files=len(files), top_level_modules=modules)
```

`counter.most_common()`
- 按计数从高到低排序，返回 `(key, count)` 对。虽然当前所有文件计数都是 1，但当目录下有多文件时排序就有意义了。

`notes.get(name, 'Python port support module')`
- 如果文件在 notes 字典中有专门的描述，就用那个描述；否则用默认描述 `'Python port support module'`。

---

## 6. src/query_engine.py —— 编排查询引擎

```python
# 第 1-7 行
from __future__ import annotations

from dataclasses import dataclass

from .commands import build_command_backlog
from .port_manifest import PortManifest, build_port_manifest
from .tools import build_tool_backlog
```

导入了三个工厂函数：`build_command_backlog`、`build_port_manifest`、`build_tool_backlog`。这些函数的职责是"给我一个结构化的数据快照"——QueryEngine 不需要知道它们是怎么构造的。

---

```python
# 第 10-16 行
@dataclass
class QueryEnginePort:
    manifest: PortManifest

    @classmethod
    def from_workspace(cls) -> 'QueryEnginePort':
        return cls(manifest=build_port_manifest())
```

两行代码，两个关键设计模式：

`@classmethod` — 替代构造函数
- `from_workspace()` 是一个"工厂方法"。正常构造是 `QueryEnginePort(manifest=some_manifest)`，工厂方法是 `QueryEnginePort.from_workspace()`。
- 区别：工厂方法封装了"默认行为"——自动扫描工作空间来初始化。调用者不需要知道 `build_port_manifest()` 的存在。
- 在 Harness 中，很多组件都有 `from_workspace` / `from_config` / `from_env` 这样的备选构造器。

`-> 'QueryEnginePort'` — 自引用类型注解
- 因为 `QueryEnginePort` 类还没有完成定义（方法在类体内定义），所以类型注解必须用字符串形式。
- `from __future__ import annotations` 让所有注解自动变成字符串，所以实际上 `-> QueryEnginePort` 也能工作。但写成字符串是更显式的习惯。

---

```python
    # 第 18-32 行
    def render_summary(self) -> str:
        command_backlog = build_command_backlog()
        tool_backlog = build_tool_backlog()
        sections = [
            '# Python Porting Workspace Summary',
            '',
            self.manifest.to_markdown(),
            '',
            f'{command_backlog.title}:',
            *command_backlog.summary_lines(),
            '',
            f'{tool_backlog.title}:',
            *tool_backlog.summary_lines(),
        ]
        return '\n'.join(sections)
```

**注释**：这是整个 QueryEngine 的"渲染管线"——把三个独立的数据源聚合成一份报告。

逐行拆解：

`command_backlog = build_command_backlog()` — 获取 Command 面数据
`tool_backlog = build_tool_backlog()` — 获取 Tool 面数据  
`self.manifest.to_markdown()` — 获取文件系统面数据

然后构建一个 `sections` 列表：

```python
sections = [
    '# Python Porting Workspace Summary',  # 大标题
    '',                                     # 空行
    self.manifest.to_markdown(),            # 模块清单（多行）
    '',                                     # 空行
    f'{command_backlog.title}:',            # "Command surface:"
    *command_backlog.summary_lines(),       # 解包展开为多行
    '',                                     # 空行
    f'{tool_backlog.title}:',              # "Tool surface:"
    *tool_backlog.summary_lines(),         # 解包展开为多行
]
```

关键语法：`*command_backlog.summary_lines()`
- `summary_lines()` 返回 `list[str]`，例如 `['- main [implemented] ...']`
- `*` 操作符解包这个 list，让每个元素成为 sections 列表的一个独立项
- 等价于：
```python
sections = [..., '- main [implemented] ...', '- summary [implemented] ...', ...]
```

最后 `'\n'.join(sections)` 把所有行用换行符连接起来，生成最终的 Markdown 文本。

**设计模式观察**：`render_summary` 是**无副作用的纯数据转换**——传入 manifest，传出 Markdown 字符串。它不读写文件、不修改全局状态。这是函数式编程在 Harness 中的体现。

---

## 7. src/main.py —— CLI 入口点（Harness 的命令路由）

```python
# 第 1-7 行
from __future__ import annotations

import argparse

from .port_manifest import build_port_manifest
from .query_engine import QueryEnginePort
```

`argparse` 是 Python 标准库中的命令行解析器。它负责将 `sys.argv` 字符串转换为结构化参数。

---

```python
# 第 9-16 行
def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(description='Python porting workspace for the Claude Code rewrite effort')
    subparsers = parser.add_subparsers(dest='command', required=True)
    subparsers.add_parser('summary', help='render a Markdown summary of the Python porting workspace')
    subparsers.add_parser('manifest', help='print the current Python workspace manifest')
    list_parser = subparsers.add_parser('subsystems', help='list the current Python modules in the workspace')
    list_parser.add_argument('--limit', type=int, default=16)
    return parser
```

这个函数的每一行都值得详细展开：

**`parser = argparse.ArgumentParser(description=...)`**
- 创建一个顶层的参数解析器。`description` 会在用户运行 `--help` 时显示。

**`subparsers = parser.add_subparsers(dest='command', required=True)`**
- 这是 `argparse` 的**子命令机制**。它允许：
  ```
  python -m src.main summary     # command = 'summary'
  python -m src.main manifest    # command = 'manifest'
  python -m src.main subsystems  # command = 'subsystems'
  ```
- `dest='command'` — 选中的子命令名存入 `args.command` 属性
- `required=True` — Python 3.7+ 的特性，必须指定一个子命令，否则 argparse 会自动报错

**`subparsers.add_parser('summary', help=...)`**
- 注册 `summary` 子命令。`help` 文本在 `--help` 时显示。
- 这个命令没有额外参数。

**`subparsers.add_parser('manifest', help=...)`**
- 注册 `manifest` 子命令。同样没有额外参数。

**`list_parser = subparsers.add_parser('subsystems', ...)`**
- 注册 `subsystems` 子命令。
- `list_parser.add_argument('--limit', type=int, default=16)` — 添加一个可选参数 `--limit`，类型是 int，默认值 16。
- 所以可以 `python -m src.main subsystems --limit 5` 只显示前 5 个模块。

**`return parser`** — 返回构建好的解析器。**不在这里解析**——这是依赖注入的思维：构建和使用分离，便于测试。

---

```python
# 第 19-34 行
def main(argv: list[str] | None = None) -> int:
    parser = build_parser()
    args = parser.parse_args(argv)
    manifest = build_port_manifest()
    if args.command == 'summary':
        print(QueryEnginePort(manifest).render_summary())
        return 0
    if args.command == 'manifest':
        print(manifest.to_markdown())
        return 0
    if args.command == 'subsystems':
        for subsystem in manifest.top_level_modules[: args.limit]:
            print(f'{subsystem.name}\t{subsystem.file_count}\t{subsystem.notes}')
        return 0
    parser.error(f'unknown command: {args.command}')
    return 2
```

这是 Harness 的**命令路由**核心：

**`argv: list[str] | None = None`**
- 允许传入参数列表，默认为 `None`（使用 `sys.argv`）。测试时可以直接传入 `['summary']` 而不依赖命令行。

**`args = parser.parse_args(argv)`**
- 解析参数。如果 `argv` 是 `None`，`argparse` 自动使用 `sys.argv[1:]`。
- 解析结果是 `Namespace(command='summary', limit=16)` 这样的对象。

**`manifest = build_port_manifest()`**
- 扫描源码树，生成 Manifest。这步在路由之前——所有子命令共享同一个 manifest。如果将来 manifest 构建有开销，这就是天然的缓存点。

**命令路由链**：
```python
if args.command == 'summary':
    print(QueryEnginePort(manifest).render_summary())
    return 0
```
- `args.command == 'summary'` → 创建 QueryEnginePort，调用 `render_summary()`，打印结果
- 返回 `0` 是 Unix 惯例：0 = 成功，非0 = 错误

```python
if args.command == 'manifest':
    print(manifest.to_markdown())
    return 0
```
- 直接调用 manifest 的 `to_markdown()`，不经过 QueryEngine

```python
if args.command == 'subsystems':
    for subsystem in manifest.top_level_modules[: args.limit]:
        print(f'{subsystem.name}\t{subsystem.file_count}\t{subsystem.notes}')
    return 0
```
- `manifest.top_level_modules[: args.limit]` — 用切片限制输出数量

**`parser.error(f'unknown command: {args.command}')`**
- 理论上永远不会执行到（因为 `required=True` 保证了有效子命令），但防御性编程的价值在于万一 argparse 的行为变化了，至少能给一个有意义的报错。

---

```python
# 第 37-38 行
if __name__ == '__main__':
    raise SystemExit(main())
```

**注释**：这是 Python 的"入口点守卫"。`__name__ == '__main__'` 只在直接运行（而非 import）时为真。

`raise SystemExit(main())` 而不是 `sys.exit(main())`：
- `sys.exit()` 抛出 `SystemExit` 异常
- `raise SystemExit(...)` 做同样的事，但不依赖 `sys` 模块的导入
- 这是一个微优化/风格选择

---

## 8. src/__init__.py —— 包的公开出口

```python
# 第 1-16 行
"""Python porting workspace for the Claude Code rewrite effort."""

from .commands import PORTED_COMMANDS, build_command_backlog
from .port_manifest import PortManifest, build_port_manifest
from .query_engine import QueryEnginePort
from .tools import PORTED_TOOLS, build_tool_backlog

__all__ = [
    'PortManifest',
    'QueryEnginePort',
    'PORTED_COMMANDS',
    'PORTED_TOOLS',
    'build_command_backlog',
    'build_port_manifest',
    'build_tool_backlog',
]
```

**注释**：`__init__.py` 定义包的公开 API。

逐行解读：

`"""Python porting workspace for the Claude Code rewrite effort."""`
- 模块文档字符串（docstring）。用 `help(src)` 可以查看。

`from .commands import PORTED_COMMANDS, build_command_backlog`
- 相对导入，`.` 代表当前包。这让包外的调用者可以写 `from src import PORTED_COMMANDS` 而不是 `from src.commands import PORTED_COMMANDS`。

`__all__ = [...]`
- 控制 `from src import *` 的行为。只有列表中的名字会被导出。
- 这也是一份**文档**——告诉使用者"这些是稳定 API，你可以依赖；其他内部模块可能改变"。

**公开 vs 内部**：
- 公开：`PortManifest`、`QueryEnginePort`、`PORTED_COMMANDS`、`PORTED_TOOLS`、三个工厂函数
- 内部（未导出）：`Subsystem`、`PortingModule`、`PortingBacklog`、`PortingTask`

内部类通过返回类型暴露——使用者在类型提示中能看到这些类型，但不应该在代码中直接构造它们。

---

## 9. tests/test_porting_workspace.py —— 验证层

```python
# 第 1-7 行
from __future__ import annotations

import subprocess
import sys
import unittest

from src.port_manifest import build_port_manifest
from src.query_engine import QueryEnginePort
```

`subprocess` — 用于启动子进程（运行 CLI 命令）
`unittest` — Python 标准库中的测试框架
只导入了两个内部组件：`build_port_manifest` 和 `QueryEnginePort`——这两者是测试目标。

---

```python
# 第 10-15 行
class PortingWorkspaceTests(unittest.TestCase):
    def test_manifest_counts_python_files(self) -> None:
        manifest = build_port_manifest()
        self.assertGreaterEqual(manifest.total_python_files, 7)
        self.assertTrue(manifest.top_level_modules)
```

**测试 1：`test_manifest_counts_python_files`**

`self.assertGreaterEqual(manifest.total_python_files, 7)` — 确保至少扫描到 7 个 Python 文件。为什么是 7？当前有 8 个 `.py` 文件（不含 `__pycache__`）。测试用 `>=7` 而不是 `==8`——这是**弹性断言**：新增文件不会让测试失败，但删除太多文件会让测试失败。它验证的是"项目至少有一个最小规模"，而不是"恰好是这个规模"。

`self.assertTrue(manifest.top_level_modules)` — 验证模块列表非空（bool(list) 在非空时为 True）。

---

```python
    # 第 17-21 行
    def test_query_engine_summary_mentions_workspace(self) -> None:
        summary = QueryEnginePort.from_workspace().render_summary()
        self.assertIn('Python Porting Workspace Summary', summary)
        self.assertIn('Command surface', summary)
        self.assertIn('Tool surface', summary)
```

**测试 2：`test_query_engine_summary_mentions_workspace`**

`QueryEnginePort.from_workspace().render_summary()` — 使用工厂方法一步到位。不需要手动构造 manifest。

三个 `assertIn` 验证输出中包含预期的关键文本片段。这是**黑盒测试**——不检查具体格式，只检查关键内容存在。防御未来修改 Markdown 格式时测试不至于全爆。

---

```python
    # 第 23-30 行
    def test_cli_summary_runs(self) -> None:
        result = subprocess.run(
            [sys.executable, '-m', 'src.main', 'summary'],
            check=True,
            capture_output=True,
            text=True,
        )
        self.assertIn('Python Porting Workspace Summary', result.stdout)
```

**测试 3：`test_cli_summary_runs`**

这是**端到端测试**（E2E Test）：

`subprocess.run([sys.executable, '-m', 'src.main', 'summary'], ...)`
- `sys.executable` — 当前 Python 解释器的路径。使用同一个解释器避免版本不匹配。
- `'-m', 'src.main'` — 以模块方式运行，模拟真实使用场景。
- `'summary'` — 传给 CLI 的子命令。

参数解读：
- `check=True` — 子进程返回非零状态码时抛出 `CalledProcessError`
- `capture_output=True` — 捕获 stdout 和 stderr
- `text=True` — 返回字符串而非 bytes

`self.assertIn('Python Porting Workspace Summary', result.stdout)` — 验证 CLI 输出包含预期文本。

**为什么需要 E2E 测试？** 前面的单元测试只验证了内部 API，但这个测试验证了整个链路：`main() → parse_args → build_port_manifest → QueryEnginePort → render_summary → print`。任何一个环节断掉都能捕获。

---

```python
# 第 33-34 行
if __name__ == '__main__':
    unittest.main()
```

标准的 `unittest` 入口。直接运行 `python tests/test_porting_workspace.py` 即可执行所有测试。

---

## 阶段一总结

你已经逐行阅读了 Harness 的最小实现。回顾关键脉络：

```
数据层（models.py + task.py）
   │
   ├──→ Subsystem：文件系统抽象（frozen）
   ├──→ PortingModule：注册表条目（frozen）
   ├──→ PortingBacklog：聚合容器（mutable）
   └──→ PortingTask：任务状态（frozen）

注册层（commands.py + tools.py）
   │
   ├──→ PORTED_COMMANDS：声明式命令注册（tuple）
   ├──→ PORTED_TOOLS：声明式工具注册（tuple）
   └──→ build_*_backlog()：工厂函数，构建 Backlog

自省层（port_manifest.py）
   │
   ├──→ build_port_manifest()：扫描源码树
   ├──→ rglob + Counter：文件统计
   └──→ PortManifest：快照型数据结构

编排层（query_engine.py）
   │
   ├──→ QueryEnginePort：聚合多源数据
   ├──→ from_workspace()：工厂方法
   └──→ render_summary()：管线渲染

路由层（main.py）
   │
   ├──→ build_parser()：argparse 子命令
   ├──→ main()：命令路由 + 执行
   └──→ __name__ == '__main__'：入口守卫

公开层（__init__.py）
   │
   └──→ __all__：控制公开 API

验证层（tests/）
   │
   ├──→ 单元测试：内部 API
   └──→ E2E 测试：整个 CLI 链路
```

进入阶段二之前，确保你能回答：
1. `frozen=True` 用在哪些类上？为什么？
2. `PORTED_COMMANDS` 为什么是 tuple 而不是 list？
3. `build_port_manifest()` 中的 `Counter` + `rglob` 组合实现了什么？
4. `QueryEnginePort.render_summary()` 的 `*` 解包语法做了什么？
5. `main()` 函数中的 `argv` 参数为什么默认是 `None`？
