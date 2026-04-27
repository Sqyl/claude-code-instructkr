# 阶段四：深入原始架构 —— Claude API、Tool Use 与安全模型

> **学习目标**：理解 Claude Code 原始 TypeScript Harness 的核心设计，对比当前 Python 实现的差距，掌握 Tool Use 协议和安全模型
> **预计时间**：第 11 天起
> **前置要求**：已完成阶段三的 5 个 Step 实现

---

## 从哪里了解原始架构

当前仓库已移除原始 TypeScript 快照。但原始架构的设计可以从以下渠道推导：

1. **Anthropic Claude API 官方文档** —— Tool Use 协议、Messages API、流式响应
2. **当前 Python 实现中的设计痕迹** —— README 提到重点研究了"harness, tool wiring, and agent workflow"
3. **Claude Code CLI 用户文档** —— 权限提示、钩子系统、配置文件结构
4. **社区讨论与逆向工程文章** —— 如本仓库中的 essay

本阶段的任务是**将碎片化的信息拼接成完整的 Harness 蓝图**。

---

## 1. Claude API 的 Tool Use 协议

这是 Harness 如何与 AI 模型对话的核心协议。

### 1.1 协议概览

```
用户说 "帮我读一下 README.md"
         │
         ▼
┌─────────────────────────────────────────────────┐
│                Harness                          │
│                                                  │
│  ① 构建 Messages:                                │
│     [                                           │
│       {role: "user", content: "读一下 README"},   │
│     ]                                           │
│     + tools: [                                  │
│       {                                         │
│         name: "read_file",                      │
│         description: "读取文件内容",             │
│         input_schema: {                         │
│           type: "object",                       │
│           properties: {                         │
│             path: {type: "string"}               │
│           },                                    │
│           required: ["path"]                     │
│         }                                       │
│       }                                         │
│     ]                                           │
│         │                                       │
│         ▼                                       │
│  ② 发送给 Claude API                              │
│         │                                       │
│         ▼                                       │
│  ③ Claude 返回:                                  │
│     {                                           │
│       stop_reason: "tool_use",                   │
│       content: [{                               │
│         type: "tool_use",                        │
│         name: "read_file",                       │
│         input: {"path": "README.md"}             │
│       }]                                        │
│     }                                           │
│         │                                       │
│         ▼                                       │
│  ④ Harness 执行工具:                              │
│     sandbox.check("read_file", "README.md")     │
│     content = sandbox.read_file("README.md")    │
│         │                                       │
│         ▼                                       │
│  ⑤ Harness 把结果送回 Claude:                     │
│     messages.append({                            │
│       role: "user",                              │
│       content: [{                                │
│         type: "tool_result",                     │
│         tool_use_id: "toolu_xxx",                │
│         content: "这是 README 的内容..."          │
│       }]                                        │
│     })                                          │
│         │                                       │
│         ▼                                       │
│  ⑥ Claude 返回最终回答:                           │
│     {                                           │
│       stop_reason: "end_turn",                   │
│       content: [{type: "text", text: "README ..."}]│
│     }                                           │
└─────────────────────────────────────────────────┘
```

### 1.2 Harness 在这个循环中的角色

每一步 Harness 都介入：

| 步骤 | Harness 的职责 |
|------|---------------|
| ① 构建请求 | 将 `PORTED_TOOLS` 转换为 Anthropic API 要求的 JSON Schema 格式 |
| ② 发送 | 管理 API Key、重试逻辑、速率限制、流式响应解析 |
| ③ 解析响应 | 将 Claude 返回的 `tool_use` block 解析为 `(tool_name, params)` |
| ④ 执行 | **最关键**——权限检查、沙箱验证、参数校验，然后调用真实的 Python 函数 |
| ⑤ 回传 | 将工具执行结果封装为 `tool_result` block |
| ⑥ 循环 | 判断 `stop_reason`：`tool_use` → 继续循环，`end_turn` → 完成 |

### 1.3 当前 Python 实现的差距

```python
# 当前：手动注册工具
PORTED_TOOLS = (
    PortingModule('port_manifest', 'Inspect...', 'src/port_manifest.py', 'implemented'),
)

# 原始 Harness 需要生成的 JSON Schema 格式：
{
    "name": "port_manifest",
    "description": "Inspect the active Python source tree and summarize the current rewrite surface",
    "input_schema": {
        "type": "object",
        "properties": {
            "src_root": {
                "type": "string",
                "description": "Root directory to scan. Defaults to src/"
            }
        },
        "required": []
    }
}
```

**差距**：当前 `PortingModule` 没有 `input_schema` 字段，无法自动生成 Tool Use 协议所需的 Schema。阶段三的 `Permission` 可以填补部分差距，但 Schema 生成仍需要手动实现。

---

## 2. 消息管路：Messages API 与流式响应

### 2.1 请求结构

```python
# 概念示意 —— Python SDK 的实际调用方式
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    system="你是 Claude Code，一个编程助手...",
    messages=[
        {"role": "user", "content": "帮我读 README.md"}
    ],
    tools=[
        {
            "name": "read_file",
            "description": "读取文件内容",
            "input_schema": {
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "文件路径"}
                },
                "required": ["path"]
            }
        }
    ]
)
```

### 2.2 流式响应处理

Claude API 支持 SSE（Server-Sent Events）流式传输。Harness 需要处理的事件类型：

| event type | 何时发生 | Harness 怎么做 |
|---|---|---|
| `message_start` | 响应开始 | 初始化消息对象 |
| `content_block_start` | 一个新的文本块或 tool_use 开始 | 文本→累积到 buffer；tool_use→记录 tool name |
| `content_block_delta` | 文本增量或 tool input JSON 增量 | 文本→逐字输出；JSON→累积 |
| `content_block_stop` | 一个块完成 | tool_use 的 input 完成后，触发执行 |
| `message_delta` | `stop_reason` 更新 | 判断是否结束 |
| `message_stop` | 完整响应结束 | 进入下一个循环或返回 |

### 2.3 工具调用积累模式

一个消息可能包含**多个** `tool_use` blocks。Harness 必须：

```
for content_block in response.content:
    if content_block.type == "text":
        accumulated_text += content_block.text

    elif content_block.type == "tool_use":
        # ① 收集所有工具调用
        pending_tools.append({
            "id": content_block.id,
            "name": content_block.name,
            "input": content_block.input,
        })

# ② 按顺序执行所有工具
tool_results = []
for tool_call in pending_tools:
    result = harness.execute_tool(tool_call["name"], tool_call["input"])
    tool_results.append({
        "type": "tool_result",
        "tool_use_id": tool_call["id"],
        "content": result,
    })

# ③ 将结果追加到消息历史
messages.append({"role": "assistant", "content": response.content})
messages.append({"role": "user", "content": tool_results})

# ④ 继续下一轮
```

这与当前 Python 实现中 `QueryEnginePort.render_summary()` 的**管线聚合**是同一种思维——收集多源数据，合并为统一输出。

---

## 3. 安全模型：三道防线

原始 Harness 的安全模型有三道防线：

```
用户输入 ──→ ┌──────────────────────────────┐
             │ 第一道：Tool Schema 校验       │ ← 参数类型、必填字段
             │ 不会让模型任意调用不存在的方法    │
             └──────────────┬───────────────┘
                            ▼
             ┌──────────────────────────────┐
             │ 第二道：Permission 权限检查    │ ← Allow/Deny 路径、需确认
             │ 阶段三 Step 3 实现了这一层      │
             └──────────────┬───────────────┘
                            ▼
             ┌──────────────────────────────┐
             │ 第三道：Sandbox 沙箱隔离       │ ← 进程级或路径级隔离
             │ 阶段三 Step 4 实现了这一层      │
             └──────────────┬───────────────┘
                            ▼
                      实际执行
```

### 3.1 第一道防线详解：Tool Schema 校验

```python
# 概念代码：参数校验层
def validate_tool_input(tool_name: str, input_data: dict) -> dict:
    """
    校验模型传来的参数是否符合 Tool 的 Schema 定义。
    防止：类型不符、缺少必填字段、恶意注入。
    """
    schema = TOOL_SCHEMAS[tool_name]

    # 1. 必填字段检查
    for required_field in schema.get("required", []):
        if required_field not in input_data:
            raise ToolValidationError(
                f'{tool_name}: missing required field {required_field}'
            )

    # 2. 类型检查
    for field_name, field_def in schema["properties"].items():
        if field_name in input_data:
            expected_type = field_def["type"]
            actual_value = input_data[field_name]
            if expected_type == "string" and not isinstance(actual_value, str):
                raise ToolValidationError(
                    f'{tool_name}.{field_name}: expected str, got {type(actual_value)}'
                )
            # ... 更多类型 ...

    # 3. 未知字段拒绝（防止注入）
    allowed_fields = set(schema["properties"].keys())
    for key in input_data:
        if key not in allowed_fields:
            raise ToolValidationError(
                f'{tool_name}: unknown field {key}'
            )

    return input_data
```

### 3.2 第二道防线详解：用户确认

某些操作即使参数正确，也需要人工授权：

```python
# 需要确认的操作示例
DANGEROUS_PATTERNS = [
    ("rm -rf", "删除操作"),
    ("git push --force", "强制推送"),
    ("chmod 777", "放宽文件权限"),
    ("DROP TABLE", "数据库删除"),
]

def should_require_approval(tool_name: str, input_data: dict) -> bool:
    """判断工具调用是否需要人工确认"""
    perm = registry.permissions.get(tool_name)
    if perm and perm.requires_approval:
        return True

    # 检查输入内容是否包含危险模式
    for key, value in input_data.items():
        if isinstance(value, str):
            for pattern, desc in DANGEROUS_PATTERNS:
                if pattern in value.lower():
                    return True

    return False
```

### 3.3 第三道防线详解：多层沙箱

原始 Harness 中的沙箱不是单一机制，而是多层组合：

```
Layer 1: 路径限制     ← 阶段三 Step 4 实现了这个
Layer 2: 网络限制     ← 阻止或白名单出站连接
Layer 3: 进程限制     ← 子进程的 cgroup / rlimit
Layer 4: 时间限制     ← 超时自动杀死
Layer 5: 文件系统限制  ← 只读挂载 / tmpfs 临时空间
```

当前 Python 实现只完成了 Layer 1。完整的沙箱需要 OS 层面支持（Linux namespace / cgroup），但在开发阶段，路径级沙箱已经可以防止大部分意外行为。

---

## 4. 钩子系统（Hook System）

### 4.1 Claude Code 的钩子架构

钩子允许在特定生命周期事件注入自定义逻辑：

```
工具调用生命周期:
                                    
  before_tool_call ──→ 执行工具 ──→ after_tool_call
       │                               │
       │ (可以阻止执行)                  │ (记录结果、清理)
       ▼                               ▼
  返回 "blocked"                返回修改后的结果

命令执行生命周期:

  before_command ──→ 执行命令 ──→ after_command
       │                              │
       │ (验证环境、预处理)             │ (后处理输出)
       ▼                              ▼
  返回修改后的参数              返回修改后的结果

Agent 循环生命周期:

  on_agent_start ──→ Agent 循环 ──→ on_agent_stop
       │                                │
       │ (初始化资源)                    │ (清理资源)
       ▼                                ▼

错误生命周期:

  on_error ──→ (决定: 重试 / 跳过 / 终止)
```

### 4.2 钩子的配置格式

Claude Code 使用 JSON 配置文件定义钩子：

```json
{
  "hooks": {
    "before_tool_call": [
      {
        "matcher": "bash",
        "command": "echo 'About to execute Shell command'",
        "timeout": 5000
      }
    ],
    "after_tool_call": [
      {
        "matcher": "edit_file",
        "command": "git diff --stat",
        "timeout": 10000
      }
    ]
  }
}
```

每条钩子有三个字段：
- **matcher**：匹配哪些工具/命令（支持 glob 模式）
- **command**：要执行的 Shell 命令
- **timeout**：超时时间（毫秒），防止钩子卡死

### 4.3 钩子与当前 Python 实现的映射

```python
# 阶段一教程中提到的 HarnessHooks 概念 → 原始 Harness 的实际实现

# 概念
@dataclass
class HarnessHooks:
    before_command: list[Callable]    # 在概念实现中，hook 是 Python 函数
    after_command: list[Callable]

# 原始 Harness
# hooks 是通过 JSON 配置 + Shell 命令实现的
# 好处：不依赖 Python 运行时，可以用任何语言写 hook
```

---

## 5. 对比：当前 Python 实现 vs 原始 TypeScript Harness

### 5.1 完整性热力图

| 子系统 | 当前实现 | 原始架构 | 差距 |
|--------|---------|---------|------|
| 数据模型 | Subsystem, PortingModule, PortingBacklog, PortingTask | 全套 TypeScript 类型 + Zod 校验 | 中等 |
| 命令系统 | argparse 子命令 + if-elif 路由 | 声明式 Command Registry + 中间件链 | 较大 |
| 工具系统 | 元组注册 + 工厂函数 | Schema-driven + 自动发现 | 较大 |
| 编排引擎 | QueryEnginePort（聚合渲染） | 完整的事件驱动 Agent 循环 | 大 |
| 文件系统自省 | build_port_manifest（rglob+Counter） | 多策略文件发现 + 缓存 | 小 |
| 缓存 | 未实现（阶段三 Step 1 补充） | LRU + 文件指纹 | 已补 |
| 依赖解析 | 未实现（阶段三 Step 2 补充） | DAG + 拓扑排序 | 已补 |
| 权限系统 | 未实现（阶段三 Step 3 补充） | Allow/Deny + 用户确认 | 已补 |
| 沙箱 | 未实现（阶段三 Step 4 补充） | 多层隔离 | 路径层已补 |
| 会话持久化 | 未实现（阶段三 Step 5 补充） | sqlite / JSON + 自动保存 | 已补 |
| 安全模型 | 无 | 三道防线 | 能补充两层 |
| 钩子系统 | 概念设计 | JSON 配置 + Shell 命令 | 仅概念 |
| 流式响应 | 无 | SSE 流式解析 | 待实现 |

### 5.2 代码风格对比

| 方面 | 原始 TypeScript | 当前 Python |
|------|----------------|------------|
| 类型系统 | Zod schema（运行时校验） | dataclass（无运行时校验） |
| 异步 | async/await + EventEmitter | 同步（无异步） |
| 依赖注入 | 装饰器 + Reflect Metadata | 手动构造函数注入 |
| 错误处理 | Result/Either 模式 | 异常（try/except） |
| 测试 | Jest + mock 自动生成 | unittest + 手动注入 |
| 配置 | JSON + 环境变量 | 仅环境变量 |

---

## 6. 从学习到贡献的路线

当你全部完成四个阶段后，这是可能的后续方向：

### 短期（第 2-3 周）
- 给当前 Python 项目提 PR，完善阶段三的一个 Step
- 为目标仓库写 documentation
- 贡献新的 Tool 注册项

### 中期（第 4-6 周）
- 实现 Anthropic API 集成（让 `QueryEnginePort` 真的调用 Claude API）
- 完成流式响应的解析
- 实现 Zod-style 的运行时 schema 校验层

### 长期（第 7 周起）
- 完整的 Agent 循环（Tool Use 往返）
- 基于 Docker/process 的真实沙箱
- 完整的 Hook 系统

---

## 附录 A：Anthropic API 快速参考

### Messages API 核心参数

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `model` | string | ✓ | 模型 ID，如 `claude-sonnet-4-6` |
| `max_tokens` | integer | ✓ | 最大输出 token 数 |
| `messages` | array | ✓ | 对话历史 |
| `system` | string/array | ✗ | 系统提示词 |
| `tools` | array | ✗ | 可用工具定义 |
| `stop_sequences` | array | ✗ | 自定义停止序列 |
| `temperature` | number | ✗ | 随机性控制（0-1） |

### Tool Use Content Block 类型

```
响应中的 content block 有三种类型：
  text          → {"type": "text", "text": "..."}
  tool_use      → {"type": "tool_use", "id": "toolu_xxx", "name": "...", "input": {...}}
  tool_result   → 在请求中发送，不在响应中出现

请求中的 content block 类型：
  text          → 同上
  tool_result   → {"type": "tool_result", "tool_use_id": "toolu_xxx", "content": "..."}
```

---

## 附录 B：关键术语中英对照

| 中文 | English | 说明 |
|------|---------|------|
| 编排层 | Harness | 工具执行、安全策略、生命周期管理的中央系统 |
| 命令路由 | Command Routing | 从 CLI 输入到处理函数的映射 |
| 工具使用协议 | Tool Use Protocol | Claude API 中模型调用工具的标准格式 |
| 权限检查 | Permission Check | 验证调用者是否有权执行操作 |
| 沙箱 | Sandbox | 隔离执行环境，限制资源访问 |
| 钩子 | Hook | 在生命周期事件点注入的自定义逻辑 |
| 拓扑排序 | Topological Sort | 依赖关系 DAG 的线性排序算法 |
| 不可变数据 | Immutable Data | 创建后不可修改的数据结构 |
| 控制反转 | IoC (Inversion of Control) | 将依赖的创建和管理交给外部 |
| 声明式注册 | Declarative Registration | 用数据结构声明可用组件，而非命令式注册 |
