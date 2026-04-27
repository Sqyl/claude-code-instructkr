# 阶段三：动手实现 —— 逐步扩展 Harness 功能

> **学习目标**：在当前代码基础上实现 5 个核心扩展，从缓存到沙箱到持久化
> **预计时间**：第 6-10 天
> **前置要求**：已完成阶段二，理解所有模块的职责边界

---

## 总体路线

```
Step 1: 命令执行结果缓存        ← 最简，改一处就能跑
Step 2: 工具链依赖解析           ← 引入 DAG 概念
Step 3: 权限检查层              ← Allow/Deny 矩阵
Step 4: 沙箱文件系统            ← 路径白名单，核心安全机制
Step 5: 会话状态持久化          ← 完整的 Session 生命周期
```

难度递增。每个 Step 都包含完整的代码和测试。

---

## Step 1: 命令执行结果缓存

**目标**：`QueryEnginePort.render_summary()` 的结果被缓存。连续调用两次时，第二次从缓存返回（避免重复扫描文件系统）。

### 1.1 设计思路

缓存的三个要素：
- **key**：基于 `manifest` 的指纹（文件总数 + 模块名称 → 如果源码树没变，缓存有效）
- **value**：渲染后的 Markdown 字符串
- **失效策略**：指纹不匹配时自动刷新

### 1.2 代码实现

```python
# src/cache.py — 新建文件
from __future__ import annotations

from dataclasses import dataclass, field
from .port_manifest import PortManifest
from .query_engine import QueryEnginePort


@dataclass
class CachedQueryEngine:
    """带缓存的查询引擎包装器"""

    manifest: PortManifest
    _cache: dict[str, str] = field(default_factory=dict)
    #      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    #      field(default_factory=dict) 避免所有实例共享同一个 dict
    #      原理和 PortingBacklog.modules 用 field(default_factory=list) 一样

    def _make_cache_key(self) -> str:
        """基于 manifest 内容生成缓存键"""
        return (
            f'{self.manifest.total_python_files}|'
            + '|'.join(m.name for m in self.manifest.top_level_modules)
        )
        # 如果文件总数相同、模块名相同，认为缓存有效
        # 生产环境中应该用更精确的指纹（如文件内容的 hash）

    def render_summary(self) -> str:
        key = self._make_cache_key()
        if key in self._cache:
            return self._cache[key]             # 缓存命中
        result = QueryEnginePort(self.manifest).render_summary()  # 重新计算
        self._cache[key] = result               # 存入缓存
        return result

    @classmethod
    def from_workspace(cls) -> 'CachedQueryEngine':
        from .port_manifest import build_port_manifest
        return cls(manifest=build_port_manifest())
```

### 1.3 集成到 CLI

```python
# src/main.py 的修改
# 在顶部导入
from .cache import CachedQueryEngine

# 在 main() 中
if args.command == 'summary':
    engine = CachedQueryEngine.from_workspace()
    print(engine.render_summary())  # 第一次：扫描 + 渲染
    print('--- cached ---')
    print(engine.render_summary())  # 第二次：从缓存返回
    return 0
```

### 1.4 测试

```python
# tests/test_cache.py
class CacheTests(unittest.TestCase):
    def test_cache_hit(self):
        engine = CachedQueryEngine.from_workspace()
        first = engine.render_summary()
        second = engine.render_summary()
        self.assertEqual(first, second)  # 内容相同

    def test_cache_invalidates_on_change(self):
        engine = CachedQueryEngine.from_workspace()
        first = engine.render_summary()
        # 模拟 manifest 变化
        engine.manifest.total_python_files += 1
        second = engine.render_summary()
        self.assertNotEqual(first, second)
```

### 1.5 设计要点

- **包装器模式**：`CachedQueryEngine` 包装 `QueryEnginePort`，不改动现有代码
- **缓存键的精度取舍**：当前用文件数+名称，简单但可能漏掉文件内容变化。生产环境中用 `hashlib` 对文件内容做 sha256
- **无全局状态**：缓存存在实例内部，可以创建多个独立引擎

---

## Step 2: 工具链依赖解析

**目标**：实现 Tool 之间的依赖声明和拓扑排序。如果 Tool A 依赖 Tool B，执行 A 前必须确保 B 已就绪。

### 2.1 给工具添加依赖声明

```python
# src/tools.py — 修改 PORTED_TOOLS，给每个模块添加 deps 字段

# 首先在 models.py 中添加
@dataclass(frozen=True)
class PortingModule:
    name: str
    responsibility: str
    source_hint: str
    status: str = 'planned'
    depends_on: tuple[str, ...] = ()    # ← 新增：依赖的模块名列表

# 然后在 tools.py 中声明依赖
PORTED_TOOLS = (
    PortingModule(
        'port_manifest',
        'Inspect the active Python source tree...',
        'src/port_manifest.py',
        'implemented',
        depends_on=('backlog_models',),     # port_manifest 依赖 backlog_models
    ),
    PortingModule(
        'backlog_models',
        'Represent subsystem and backlog metadata...',
        'src/models.py',
        'implemented',
        depends_on=(),                       # 无依赖
    ),
    PortingModule(
        'query_engine',
        'Coordinate Python-facing rewrite summaries...',
        'src/query_engine.py',
        'implemented',
        depends_on=('port_manifest', 'backlog_models'),  # 依赖两个
    ),
)
```

### 2.2 拓扑排序引擎

```python
# src/dependency_resolver.py — 新建文件
from __future__ import annotations

from collections import deque
from .models import PortingModule


class CircularDependencyError(Exception):
    """检测到循环依赖时抛出"""
    pass


def resolve_order(modules: list[PortingModule]) -> list[PortingModule]:
    """
    拓扑排序 —— 返回可以按序执行的模块列表。

    算法：Kahn's algorithm（基于入度的拓扑排序）
    - 时间复杂度：O(V + E)
    - 空间复杂度：O(V)

    步骤：
    1. 构建入度表和邻接表
    2. 入度为 0 的节点入队
    3. 依次出队，减少后继节点的入度
    4. 新入度为 0 的节点入队
    5. 最后检查是否有剩余节点（有 = 循环依赖）
    """
    name_to_module = {m.name: m for m in modules}
    in_degree = {m.name: 0 for m in modules}
    adjacency = {m.name: [] for m in modules}

    # 构建图
    for m in modules:
        for dep in m.depends_on:
            if dep not in name_to_module:
                raise ValueError(
                    f'{m.name} depends on unknown module: {dep}'
                )
            adjacency[dep].append(m.name)
            in_degree[m.name] += 1

    # Kahn's algorithm
    queue = deque(name for name, deg in in_degree.items() if deg == 0)
    result = []

    while queue:
        current = queue.popleft()
        result.append(name_to_module[current])
        for neighbor in adjacency[current]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)

    # 循环依赖检测
    if len(result) != len(modules):
        remaining = [m.name for m in modules if m.name not in {r.name for r in result}]
        raise CircularDependencyError(
            f'Circular dependency detected among: {remaining}'
        )

    return result
```

### 2.3 测试依赖解析

```python
# tests/test_dependency_resolver.py
class DependencyResolverTests(unittest.TestCase):
    def test_linear_deps(self):
        a = PortingModule('a', '', '', depends_on=('b',))
        b = PortingModule('b', '', '', depends_on=('c',))
        c = PortingModule('c', '', '', depends_on=())
        order = resolve_order([a, b, c])
        names = [m.name for m in order]
        # c 必须在 b 前面，b 必须在 a 前面
        self.assertLess(names.index('c'), names.index('b'))
        self.assertLess(names.index('b'), names.index('a'))

    def test_circular_dependency_detected(self):
        a = PortingModule('a', '', '', depends_on=('b',))
        b = PortingModule('b', '', '', depends_on=('a',))
        with self.assertRaises(CircularDependencyError):
            resolve_order([a, b])

    def test_unknown_dependency_raises(self):
        a = PortingModule('a', '', '', depends_on=('nonexistent',))
        with self.assertRaises(ValueError):
            resolve_order([a])
```

### 2.4 设计要点

- **Kahn's algorithm** 是拓扑排序的经典算法，入度为零意味着"所有前置都已完成"
- **循环依赖检测**：如果结果长度不等于输入长度，剩余的节点形成了一个环
- **验证未知依赖**：声明了不存在的依赖立即报错，不让错误传播到运行时

---

## Step 3: 权限检查层（Allow/Deny 列表）

**目标**：实现一个权限检查中间层。在执行 Tools 之前检查调用是否被允许。

### 3.1 权限数据模型

```python
# src/permissions.py — 新建文件
from __future__ import annotations

from dataclasses import dataclass, field
from typing import Pattern
import re


@dataclass(frozen=True)
class Permission:
    """描述一个工具的访问控制策略"""
    tool_name: str                     # 工具名
    allow_paths: tuple[str, ...] = ()  # 允许访问的路径模式
    deny_paths: tuple[str, ...] = ()   # 禁止访问的路径模式
    requires_approval: bool = False    # 是否需要人工确认


@dataclass
class PermissionRegistry:
    """管理所有工具的权限策略

    规则优先级：
    1. deny_paths 优先于 allow_paths（安全优先）
    2. 精确匹配优先于通配符
    """
    permissions: dict[str, Permission] = field(default_factory=dict)

    def register(self, perm: Permission) -> None:
        self.permissions[perm.tool_name] = perm

    def check(self, tool_name: str, target_path: str) -> bool:
        """检查工具是否能访问给定路径"""
        perm = self.permissions.get(tool_name)
        if perm is None:
            # 未注册权限的工具默认允许（开放策略）
            # 生产环境应该默认拒绝（安全策略）
            return True

        # deny 优先检查
        for pattern in perm.deny_paths:
            if self._match(pattern, target_path):
                return False

        # 如果没有 allow 规则，允许所有
        if not perm.allow_paths:
            return True

        # allow 检查
        for pattern in perm.allow_paths:
            if self._match(pattern, target_path):
                return True

        return False

    @staticmethod
    def _match(pattern: str, path: str) -> bool:
        """路径模式匹配。支持通配符 ** 和 *。"""
        # 将 glob 模式转换为正则
        regex = pattern.replace('.', r'\.').replace('**', '___DOUBLESTAR___')
        regex = regex.replace('*', '[^/]*')
        regex = regex.replace('___DOUBLESTAR___', '.*')
        return bool(re.search(f'^{regex}$', path))
```

### 3.2 使用方式

```python
# 在应用启动时注册权限
registry = PermissionRegistry()
registry.register(Permission(
    tool_name='port_manifest',
    allow_paths=('src/**', 'tests/**'),     # 只能访问这些目录
    deny_paths=('src/.secret/**',),          # 但不能访问 .secret
    requires_approval=False,
))
registry.register(Permission(
    tool_name='query_engine',
    allow_paths=('src/**',),
    requires_approval=True,                  # 需要人工确认
))

# 执行工具前检查
def execute_tool(tool_name: str, target_path: str, registry: PermissionRegistry):
    if not registry.check(tool_name, target_path):
        raise PermissionError(f'{tool_name} denied access to {target_path}')

    perm = registry.permissions.get(tool_name)
    if perm and perm.requires_approval:
        response = input(f'Allow {tool_name} to access {target_path}? [y/N] ')
        if response.lower() != 'y':
            raise PermissionError('User denied')

    # ... 执行工具逻辑 ...
```

### 3.3 测试

```python
class PermissionTests(unittest.TestCase):
    def setUp(self):
        self.reg = PermissionRegistry()
        self.reg.register(Permission(
            'reader',
            allow_paths=('src/**',),
            deny_paths=('src/secret/**',),
        ))

    def test_allow_in_src(self):
        self.assertTrue(self.reg.check('reader', 'src/main.py'))

    def test_deny_secret_dir(self):
        self.assertFalse(self.reg.check('reader', 'src/secret/keys.py'))

    def test_deny_outside_allow(self):
        self.assertFalse(self.reg.check('reader', 'etc/passwd'))

    def test_unregistered_tool_allowed(self):
        self.assertTrue(self.reg.check('unknown_tool', '/anywhere'))
```

### 3.4 设计要点

- **deny 优先**：安全检查的逻辑必须是 deny 先于 allow——宁可误拒，不可误放
- **未注册=放行**：当前实现选择"默认允许"。生产环境中**必须**改为"默认拒绝"（`return False`）
- **正则转换**：glob `**` → 正则 `.*`，glob `*` → 正则 `[^/]*`。这个转换有边界情况（如路径中的 `.`），生产代码中应考虑用 `fnmatch` 或 `pathlib` 的正统方法

---

## Step 4: 沙箱文件系统（限制工具可访问的路径）

**目标**：为每个 Tool 创建一个沙箱化的文件系统视图，工具只能看到 `allow_paths` 内的文件。

### 4.1 核心概念

沙箱不是真正的容器（那需要 OS 层面的隔离），而是**路径透明包装**：
- 所有文件操作都通过沙箱代理
- 路径自动在"真实路径"和"沙箱路径"之间翻译
- 超出允许范围的路径访问抛出异常

### 4.2 代码实现

```python
# src/sandbox.py — 新建文件
from __future__ import annotations

from dataclasses import dataclass
from pathlib import Path
import os


class SandboxViolationError(Exception):
    """试图访问沙箱外的路径时抛出"""
    pass


@dataclass
class Sandbox:
    """文件系统沙箱 —— 限制可访问的目录范围"""

    root: Path                          # 沙箱根目录
    allowed_paths: tuple[Path, ...]     # 允许访问的绝对路径列表

    def resolve(self, target: str | Path) -> Path:
        """
        将用户请求的路径解析为真实路径，并验证是否在允许范围内。

        步骤：
        1. 如果是相对路径，相对于 root 解析
        2. resolve() 消除 .. 和符号链接
        3. 遍历 allowed_paths，检查解析后的路径是否在其中
        """
        target_path = Path(target)
        if not target_path.is_absolute():
            target_path = self.root / target_path

        resolved = target_path.resolve()

        # 验证在允许范围内
        for allowed in self.allowed_paths:
            try:
                # Python 3.9+: Path.is_relative_to()
                resolved.relative_to(allowed)
                return resolved
            except ValueError:
                continue

        raise SandboxViolationError(
            f'Access denied: {target} resolves to {resolved}, '
            f'which is outside allowed paths: {self.allowed_paths}'
        )

    def list_dir(self, target: str = '.') -> list[str]:
        """列出沙箱内的目录内容"""
        path = self.resolve(target)
        return [p.name for p in path.iterdir()]

    def read_text(self, target: str) -> str:
        """读取沙箱内的文件"""
        path = self.resolve(target)
        if not path.is_file():
            raise FileNotFoundError(f'{target} is not a file')
        return path.read_text(encoding='utf-8')

    def read_bytes(self, target: str) -> bytes:
        """读取沙箱内的二进制文件"""
        path = self.resolve(target)
        return path.read_bytes()

    def glob(self, pattern: str) -> list[str]:
        """在沙箱内搜索匹配的文件"""
        # 先在根目录下搜索
        results = []
        search_dir = self.root
        for p in search_dir.rglob(pattern):
            try:
                resolved = self.resolve(p)
                results.append(str(resolved.relative_to(self.root)))
            except SandboxViolationError:
                pass
        return results
```

### 4.3 集成到 PermissionRegistry

```python
# 扩展 permissions.py，用 Permission 生成 Sandbox
class PermissionRegistry:
    # ... 之前的代码 ...

    def create_sandbox(self, tool_name: str) -> Sandbox:
        """为指定工具创建沙箱"""
        perm = self.permissions.get(tool_name)
        if perm is None:
            # 无权限注册 = 最严格沙箱（空 allow 列表）
            return Sandbox(root=Path.cwd(), allowed_paths=())

        allowed = tuple(Path(p).resolve() for p in perm.allow_paths)
        return Sandbox(root=Path.cwd(), allowed_paths=allowed)
```

### 4.4 使用示例

```python
# 在工具实现中使用沙箱
def inspect_manifest(sandbox: Sandbox) -> str:
    """使用沙箱化的文件访问来检视工作空间"""
    files = []
    for py_file in sandbox.glob('*.py'):
        content = sandbox.read_text(py_file)
        files.append(f'{py_file}: {len(content)} bytes')
    return '\n'.join(files)
```

### 4.5 测试

```python
class SandboxTests(unittest.TestCase):
    def setUp(self):
        self.tmp = Path('/tmp/sandbox_test')
        self.tmp.mkdir(exist_ok=True)
        (self.tmp / 'allowed.txt').write_text('hello')
        (self.tmp / 'secret.txt').write_text('shh')
        allowed = (self.tmp / 'allowed',)
        self.sandbox = Sandbox(root=self.tmp, allowed_paths=(self.tmp,))

    def test_read_allowed_file(self):
        content = self.sandbox.read_text('allowed.txt')
        self.assertEqual(content, 'hello')

    def test_reject_escape_attempt(self):
        """测试路径穿越攻击（.. 跳出沙箱）"""
        with self.assertRaises(SandboxViolationError):
            self.sandbox.resolve('../etc/passwd')

    def test_globbing(self):
        results = self.sandbox.glob('*.txt')
        self.assertIn('allowed.txt', results)
```

### 4.6 设计要点

- **`resolve()` 消除 `..`**：`Path.resolve()` 会消除路径中的 `..` 和 `.`，所以 `/allowed/../etc/passwd` 会变成 `/etc/passwd`，然后被拒绝——路径穿越攻击被自然免疫
- **`relative_to()` 验证**：`path.relative_to(allowed)` 只在 `path` 是 `allowed` 的子路径时不抛异常。这是 Python 3.9+ 的内置能力，无需字符串比对
- **空 allowed_paths = 完全隔离**：如果没有允许的路径，任何访问都被拒

---

## Step 5: 构建会话持久化（保存/恢复任务状态）

**目标**：在多次 CLI 调用之间保持 `PortingTask` 的状态。

### 5.1 会话数据模型

```python
# src/session.py — 新建文件
from __future__ import annotations

from dataclasses import dataclass, field, asdict
from datetime import datetime, timezone
from pathlib import Path
import json
import uuid

from .task import PortingTask


@dataclass
class Session:
    """一次工作会话"""
    session_id: str = field(default_factory=lambda: uuid.uuid4().hex[:12])
    created_at: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())
    tasks: dict[str, PortingTask] = field(default_factory=dict)
    metadata: dict = field(default_factory=dict)

    def add_task(self, task: PortingTask) -> None:
        self.tasks[task.title] = task

    def complete_task(self, title: str) -> PortingTask:
        task = self.tasks[title]
        # 不可变更新：创建新实例
        updated = PortingTask(title=task.title, detail=task.detail, completed=True)
        self.tasks[title] = updated
        return updated

    def to_json(self) -> str:
        """序列化为 JSON —— 因为 dataclass 嵌套较深，需要手动处理"""
        return json.dumps({
            'session_id': self.session_id,
            'created_at': self.created_at,
            'tasks': {
                title: {
                    'title': t.title,
                    'detail': t.detail,
                    'completed': t.completed,
                }
                for title, t in self.tasks.items()
            },
            'metadata': self.metadata,
        }, indent=2, ensure_ascii=False)

    @classmethod
    def from_json(cls, data: str) -> 'Session':
        raw = json.loads(data)
        session = cls(
            session_id=raw['session_id'],
            created_at=raw['created_at'],
            metadata=raw.get('metadata', {}),
        )
        for title, t_data in raw['tasks'].items():
            session.tasks[title] = PortingTask(
                title=t_data['title'],
                detail=t_data['detail'],
                completed=t_data['completed'],
            )
        return session


class SessionStore:
    """将 Session 持久化到磁盘"""
    def __init__(self, storage_dir: Path | None = None):
        self.storage_dir = storage_dir or Path('.harness_sessions')
        self.storage_dir.mkdir(exist_ok=True)

    def _path_for(self, session_id: str) -> Path:
        return self.storage_dir / f'{session_id}.json'

    def save(self, session: Session) -> None:
        self._path_for(session.session_id).write_text(
            session.to_json(), encoding='utf-8'
        )

    def load(self, session_id: str) -> Session:
        path = self._path_for(session_id)
        if not path.exists():
            raise FileNotFoundError(f'Session {session_id} not found')
        return Session.from_json(path.read_text(encoding='utf-8'))

    def list_sessions(self) -> list[dict]:
        """列出所有已保存的会话摘要"""
        summaries = []
        for f in sorted(self.storage_dir.glob('*.json'), key=lambda p: p.stat().st_mtime, reverse=True):
            session = Session.from_json(f.read_text(encoding='utf-8'))
            completed = sum(1 for t in session.tasks.values() if t.completed)
            summaries.append({
                'id': session.session_id,
                'created': session.created_at,
                'tasks_total': len(session.tasks),
                'tasks_completed': completed,
            })
        return summaries
```

### 5.2 CLI 集成

```python
# 在 src/main.py 中新增命令
# build_parser() 中添加:
session_parser = subparsers.add_parser('session', help='manage sessions')
session_sub = session_parser.add_subparsers(dest='session_command')

save_parser = session_sub.add_parser('save', help='save current session')
load_parser = session_sub.add_parser('load', help='load a session')
load_parser.add_argument('session_id', help='session ID to load')
list_parser = session_sub.add_parser('list', help='list all sessions')

# main() 中添加:
if args.command == 'session':
    store = SessionStore()
    if args.session_command == 'list':
        for s in store.list_sessions():
            print(f'{s["id"]}  {s["created"]}  {s["tasks_completed"]}/{s["tasks_total"]}')
        return 0
    if args.session_command == 'save':
        session = Session()
        # 从当前 manifest 创建任务
        for module in manifest.top_level_modules:
            session.add_task(PortingTask(
                title=module.name,
                detail=module.notes,
                completed=False,
            ))
        store.save(session)
        print(f'Saved session: {session.session_id}')
        return 0
    if args.session_command == 'load':
        session = store.load(args.session_id)
        print(f'Loaded session {args.session_id}:')
        for task in session.tasks.values():
            status = '✓' if task.completed else ' '
            print(f'  [{status}] {task.title}')
        return 0
```

### 5.3 测试

```python
class SessionTests(unittest.TestCase):
    def setUp(self):
        self.store = SessionStore(storage_dir=Path('/tmp/test_sessions'))

    def test_save_and_load(self):
        session = Session()
        session.add_task(PortingTask('test', 'a test task'))
        self.store.save(session)

        loaded = self.store.load(session.session_id)
        self.assertEqual(len(loaded.tasks), 1)
        self.assertEqual(loaded.tasks['test'].title, 'test')

    def test_complete_task_creates_new_instance(self):
        session = Session()
        session.add_task(PortingTask('test', 'detail'))
        original = session.tasks['test']
        updated = session.complete_task('test')
        # 不可变性验证：原有实例不变
        self.assertFalse(original.completed)
        self.assertTrue(updated.completed)
        # 新实例替换了旧实例
        self.assertTrue(session.tasks['test'].completed)

    def test_list_empty_store(self):
        summaries = self.store.list_sessions()
        # 初始空（可能有之前测试残留，不强制 assertEqual 0）
        self.assertIsInstance(summaries, list)

    def tearDown(self):
        import shutil
        if self.store.storage_dir.exists():
            shutil.rmtree(self.store.storage_dir)
```

### 5.4 设计要点

- **不可变更新**：`complete_task()` 创建新 `PortingTask` 实例而非修改原实例。这种模式在 Redux / React 中很常见，保证了状态变更的可预测性
- **JSON 序列化**：`to_json()` 手工序列化而非依赖 `asdict()`，因为 `asdict()` 对嵌套 frozen dataclass 的行为不够直观。手工序列化给了完全的控制
- **存储目录**：默认 `.harness_sessions/`，用户可通过参数覆盖
- **会话摘要**：`list_sessions()` 只返回摘要（id + 时间 + 完成度），不加载全量数据。处理大量会话时有性能优势

---

## 全部文件一览

完成全部 5 个 Step 后，`src/` 目录结构：

```
src/
├── __init__.py              # 包出口（需更新 __all__）
├── main.py                  # CLI 入口（需添加 session 命令）
├── models.py                # 数据模型（需给 PortingModule 加 depends_on）
├── commands.py              # 命令注册
├── tools.py                 # 工具注册（需添加 depends_on）
├── task.py                  # 任务模型
├── port_manifest.py         # 工作空间自省
├── query_engine.py          # 编排查询（不变，CachedQueryEngine 是包装器）
├── cache.py                 # ← 新建：Step 1 缓存
├── dependency_resolver.py   # ← 新建：Step 2 依赖解析
├── permissions.py           # ← 新建：Step 3 + 4 权限与沙箱
├── sandbox.py               # ← 新建：Step 4 沙箱文件系统
└── session.py               # ← 新建：Step 5 会话持久化

tests/
├── test_porting_workspace.py    # 原有测试
├── test_cache.py                # ← 新建
├── test_dependency_resolver.py  # ← 新建
├── test_permissions.py          # ← 新建
├── test_sandbox.py              # ← 新建
└── test_session.py              # ← 新建
```

---

## 阶段三自检清单

完成全部 5 个 Step 后，确认：

- [ ] `CachedQueryEngine` 连续两次调用返回相同结果
- [ ] `resolve_order` 正确处理线性依赖、菱形依赖、循环依赖
- [ ] `PermissionRegistry.check` 的 deny 优先于 allow
- [ ] `Sandbox.resolve` 阻止 `../` 路径穿越
- [ ] `SessionStore.save` + `SessionStore.load` 来回一致
- [ ] 所有测试通过：`python -m pytest tests/ -v`
