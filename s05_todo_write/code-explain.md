# S05 todo_write

```python
import os, subprocess
from pathlib import Path
```

**os: 获取电脑的环境信息、文件路径、系统变量等**

os.getenv("ANTHROPIC_API_KEY")  # 读取环境变量（你的API密钥）
os.environ.pop(...)            # 删除环境变量

**subprocess: 让 Python 执行电脑的终端 / CMD 命令**

import subprocess

result = subprocess.run(
    "这里写要执行的命令",
    shell=True,
    capture_output=True,
    text=True
)

- **shell=True**：允许执行系统命令
- **capture_output=True**：把命令输出结果抓回来
- **text=True**：输出变成字符串，而不是二进制

**Path: 安全、简单、跨平台地处理文件路径**

WORKDIR = Path.cwd()  # 获取当前文件夹路径
```python
try:
  	# 尝试导入 readline（只有 Mac/Linux 有）
    import readline
    # 优化命令行输入体验
    readline.parse_and_bind('set bind-tty-special-chars off')
except ImportError:
  	# 如果是 Windows，没有这个库 → 直接跳过，不报错
    pass
```



try 意思：

**尝试执行下面的代码，如果出错了，不要崩溃，直接跳到 except**

- `readline` 是 **Python 的一个系统库**

- 只存在于 **Mac / Linux**

- **Windows 没有这个库**

  作用：

  优化命令行输入体验

  （方向键、删除、光标不乱跳）

  readline.parse_and_bind('set bind-tty-special-chars off')

这行是**专业优化命令**，意思是：

**关闭终端特殊字符的强制绑定**

效果：

- 输入方向键 ← → 不会乱码
- 按退格不会删错
- 输入长文本不会光标乱飞
- 复制粘贴不会出奇怪字符



```python
# 导入 Anthropic 官方 SDK，用于调用 Claude 大模型 API
from anthropic import Anthropic
# 导入 python-dotenv 库，用于读取 .env 环境变量文件
from dotenv import load_dotenv

# 加载 .env 文件中的环境变量
# override=True 表示：如果系统中已有同名环境变量，优先使用 .env 里的值
load_dotenv(override=True)
# 判断：如果配置了自定义的接口地址（ANTHROPIC_BASE_URL）
# 通常是使用国内中转服务时才会配置
if os.getenv("ANTHROPIC_BASE_URL"):
    # 删除环境变量中的 ANTHROPIC_AUTH_TOKEN（避免冲突）
    # 因为官方 SDK 和中转服务的认证方式不同
    os.environ.pop("ANTHROPIC_AUTH_TOKEN", None)

# 获取当前程序运行的工作目录（非常重要，所有文件操作都限制在这里）
WORKDIR = Path.cwd()
# 创建 Anthropic 客户端实例，用于后续发送请求
# 如果配置了 base_url（中转地址）就用，没有则默认使用官方地址
client = Anthropic(base_url=os.getenv("ANTHROPIC_BASE_URL"))
# 从环境变量中读取要使用的模型名称（如 claude-3-5-sonnet-20240620）
MODEL = os.environ["MODEL_ID"]
# 定义一个全局变量，用于存储 AI 待办任务列表（你代码里的 todo 功能）
CURRENT_TODOS: list[dict] = []

```



```python
# s05 版本升级点：系统提示词中加入了【任务规划】指导
# SYSTEM 变量 = 给 AI 助手的核心指令（人设 + 规则），AI 会严格遵守
SYSTEM = (
    # 1. 给 AI 设定身份：你是当前目录下的编码助手，知道自己的工作路径
    f"You are a coding agent at {WORKDIR}. "
    # 2. 核心规则：执行复杂/多步骤任务前，必须先用 todo_write 工具做规划
    "Before starting any multi-step task, use todo_write to plan your steps. "
    # 3. 执行规则：完成一步就更新一步状态（待办→进行中→已完成）
    "Update status as you go."
)
```





```python
# ═══════════════════════════════════════════════════════════
#  FROM s02-s04 (unchanged): Tool Implementations
#  说明：这部分是 AI 可以调用的【工具函数】，从旧版本继承过来，没有改动
#  作用：让 AI 能操作电脑：执行命令、读文件、写文件、改文件、查文件
# ═══════════════════════════════════════════════════════════

# 安全路径校验函数（核心安全机制）
# 作用：防止 AI 访问工作目录以外的文件，保护电脑安全
def safe_path(p: str) -> Path:
    # 拼接路径：工作目录 WORKDIR + 传入的路径 p，并转为绝对路径
    path = (WORKDIR / p).resolve()
    # 检查：最终路径是否在工作目录内部
    # 如果不在 → 抛出错误，禁止访问
    #
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    # 安全检查通过，返回合法路径
    return path
```

​     1、`.resolve()`

这是**最关键的一步**，作用：

1. **把相对路径变成绝对路径**

   `test.py` → `/user/myproject/test.py`

2. **去掉多余的符号**

   `./test.py` → `test.py`

   `a/../test.py` → `test.py`

3. **标准化路径格式**

   让系统能准确识别，不会出错

   

   2、`path.is_relative_to(WORKDIR)`

   **判断：path 是不是在 WORKDIR 文件夹里面**

   - 在里面 → 返回 `True`
   - 不在里面 → 返回 `False`

```python
# AI 调用的【执行终端命令】工具
# 作用：让 AI 能执行 bash/cmd 命令（如 ls, git, python 等）
def run_bash(command: str) -> str:
    try:
        # 执行系统命令
        r = subprocess.run(
            command,        # 要执行的命令
            shell=True,      # 使用系统 shell 执行
            cwd=WORKDIR,     # 限定在工作目录执行
            capture_output=True,  # 捕获命令输出和错误
            text=True,       # 输出转为字符串
            timeout=120      # 最多执行120秒，防止卡死
        )
        # 合并标准输出 + 错误输出，并去掉首尾空白
        out = (r.stdout + r.stderr).strip()
        # 返回结果，最多返回50000字符，避免内容过长
        return out[:50000] if out else "(no output)"
    # 如果命令执行超时，返回错误信息
    except subprocess.TimeoutExpired:
        return "Error: Timeout (120s)"
```



```python
def run_read(path: str, limit: int | None = None) -> str:
    try:
        # 先做安全校验，再读取文件内容并按行拆分
        lines = safe_path(path).read_text().splitlines()
        # 如果设置了行数限制，并且文件行数超过限制
        if limit and limit < len(lines):
            # 只保留前 limit 行，其余显示省略
            lines = lines[:limit] + [f"... ({len(lines) - limit} more lines)"]
        # 把行列表重新拼接成字符串返回
        return "\n".join(lines)
    # 读取出错（文件不存在、权限不足等），返回错误
    except Exception as e:
        return f"Error: {e}"
```

1. `safe_path(path)`

**作用：安全检查**

- 检查这个文件是不是在当前项目文件夹里
- 不在的话直接报错，不让读
- 返回**安全的、合法的路径**

2. `.read_text()`

**作用：读取文件里的所有内容**

- 把文件里的文字**全部读出来**
- 返回一整串字符串，比如：

```
"第一行\n第二行\n第三行"
```

3. `.splitlines()`

**作用：按换行符切成 “一行一行” 的列表**

把上面的字符串变成：

```
["第一行", "第二行", "第三行"]
```

```python
# AI 调用的【写入文件】工具
# 作用：让 AI 创建/覆盖文件
def run_write(path: str, content: str) -> str:
    try:
        # 安全校验路径
        file_path = safe_path(path)
        # 自动创建父文件夹（如果不存在）
        # 确保文件所在的文件夹一定存在，不存在就自动创建，存在就不管
        file_path.parent.mkdir(parents=True, exist_ok=True)
        # 把内容写入文件
        file_path.write_text(content)
        # 返回成功提示
        return f"Wrote {len(content)} bytes to {path}"
    # 写入失败，返回错误
    except Exception as e:
        return f"Error: {e}"

# AI 调用的【编辑修改文件】工具
# 作用：精确替换文件中的一段文字
def run_edit(path: str, old_text: str, new_text: str) -> str:
    try:
        # 安全校验路径
       # file_path 是一个 Path 对象，不是字符串！不是数字！是 专门用来表示文件路径的高级对象。
        file_path = safe_path(path)
        # 读取文件全部内容
        text = file_path.read_text()
        # 如果要替换的旧文本不存在，返回错误
        if old_text not in text:
            return f"Error: text not found in {path}"
        # 替换第一次出现的旧文本
        file_path.write_text(text.replace(old_text, new_text, 1))
        # 返回成功提示
        return f"Edited {path}"
    # 修改失败，返回错误
    except Exception as e:
        return f"Error: {e}"

```



```python
# AI 调用的【搜索文件】工具
# 作用：根据通配符查找文件（如 *.py, *.md）
def run_glob(pattern: str) -> str:
    # 导入 glob 模块，用于文件匹配搜索
    import glob as g
    try:
        results = []
        # 在工作目录下搜索匹配 pattern 的文件
        for match in g.glob(pattern, root_dir=WORKDIR):
            # 安全校验：确保搜索结果在工作目录内
            if (WORKDIR / match).resolve().is_relative_to(WORKDIR):
                results.append(match)
        # 返回找到的文件列表，没有则提示无匹配
        return "\n".join(results) if results else "(no matches)"
    # 搜索出错，返回错误
    except Exception as e:
        return f"Error: {e}"
```

假设你的目录里有：

- main.py
- test.py
- readme.md
- app.js

你搜索：

```
pattern = "*.py"
```

那么：

```
g.glob("*.py", root_dir=WORKDIR)
```

会找到：

```
main.py
test.py
```

然后循环：

```
match = "main.py"
match = "test.py"
```

```python
# ═══════════════════════════════════════════════════════════
#  NEW in s05: todo_write tool — plan only, no execution
#  s05 版本新增功能：AI 待办任务工具（只做规划，不执行操作）
# ═══════════════════════════════════════════════════════════

# 定义 AI 调用的【创建/更新待办任务】工具
# 输入：todos 列表，里面是一个个任务字典
def run_todo_write(todos: list) -> str:
    # 声明使用全局变量 CURRENT_TODOS，用来保存整个程序的任务列表
    global CURRENT_TODOS
    # ===================== 第一步：数据校验（保证传入的任务格式正确） =============
    # 遍历所有任务，检查每个任务是否符合要求
    for i, t in enumerate(todos):
        # 检查：每个任务必须包含 content（任务内容）和 status（任务状态）
        if "content" not in t or "status" not in t:
            return f"Error: todos[{i}] missing 'content' or 'status'"
        
        # 检查：状态只能是 pending(待处理) / in_progress(进行中) / completed(已完成) 三种
        if t["status"] not in ("pending", "in_progress", "completed"):
            return f"Error: todos[{i}] has invalid status '{t['status']}'"

    # ===================== 第二步：校验通过，保存任务到全局变量 ==================
    # 把新的任务列表覆盖保存到全局变量中
    CURRENT_TODOS = todos

    # ===================== 第三步：在终端里美化打印出任务列表 ===================
    # 黄色标题：## Current Tasks
    lines = ["\n\033[33m## Current Tasks\033[0m"]
    
    # 遍历每个任务，给不同状态配不同图标
    for t in CURRENT_TODOS:
        # 状态图标：待办=空格，进行中=蓝色三角，已完成=绿色对勾
        icon = {
            "pending": " ",
            "in_progress": "\033[36m▸\033[0m",
            "completed": "\033[32m✓\033[0m"
        }[t["status"]]
        
        # 拼接成一行：[图标] 任务内容
        lines.append(f"  [{icon}] {t['content']}")
    
    # 把所有行打印到终端
    print("\n".join(lines))

    # ===================== 第四步：返回结果给 AI =====================
    # 告诉 AI：成功更新了多少个任务
    return f"Updated {len(CURRENT_TODOS)} tasks"
```

**todos 长什么样子？**

todos = [   

 {"content": "第一步：做计划", "status": "pending"},    

{"content": "第二步：写代码", "status": "in_progress"},    

{"content": "第三步：测试", "status": "completed"} ]

终端会显示：

\## Current Tasks 

[ ] 读取项目目录结构  

[▸] 创建 main.py 文件  

[✓] 编写代码注释

```python
# =============================================================================
#  TOOLS 列表：给 AI 大模型看的【工具说明书】
#  告诉 AI：你有哪些工具可以用，每个工具需要传什么参数
#  格式完全遵循 Claude / 大模型 API 的工具调用标准
# =============================================================================
TOOLS = [
    # 工具1：执行终端命令
    {
        "name": "bash",  # 工具名称（固定）
        "description": "Run a shell command.",  # 给AI看的说明：执行shell命令
        "input_schema": {  # 参数格式规定
            "type": "object",  # 参数是一个对象（字典）
            "properties": {  # 具体参数
                "command": {"type": "string"}  # 参数command：字符串类型
            },
            "required": ["command"]  # 必传参数：必须传command
        }
    },

    # 工具2：读取文件
    {
        "name": "read_file",
        "description": "Read file contents.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},       # 文件路径（必传）
                "limit": {"type": "integer"}      # 最多读几行（可选）
            },
            "required": ["path"]
        }
    },

    # 工具3：写入文件
    {
        "name": "write_file",
        "description": "Write content to a file.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},       # 文件路径
                "content": {"type": "string"}     # 要写入的内容
            },
            "required": ["path", "content"]  # 两个参数都必须传
        }
    },

    # 工具4：编辑修改文件
    {
        "name": "edit_file",
        "description": "Replace exact text in a file once.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},       # 文件路径
                "old_text": {"type": "string"},   # 要被替换的旧文本
                "new_text": {"type": "string"}    # 替换成的新文本
            },
            "required": ["path", "old_text", "new_text"]
        }
    },

    # 工具5：搜索匹配文件
    {
        "name": "glob",
        "description": "Find files matching a glob pattern.",
        "input_schema": {
            "type": "object",
            "properties": {
                "pattern": {"type": "string"}  # 匹配规则，如 *.py
            },
            "required": ["pattern"]
        }
    },

    # s05 新增工具：任务规划（待办清单）
    {
        "name": "todo_write",
        "description": "Create and manage a task list for your current coding session.",
        "input_schema": {
            "type": "object",
            "properties": {
                # 参数 todos：是一个数组（列表）
                "todos": {
                    "type": "array",
                    # 数组里的每一项是一个任务对象
                    "items": {
                        "type": "object",
                        "properties": {
                            "content": {"type": "string"},  # 任务内容
                            "status": {
                                "type": "string",
                                "enum": ["pending", "in_progress", "completed"]  # 只允许这3种状态
                            }
                        },
                        "required": ["content", "status"]  # 每个任务必须包含这两个字段
                    }
                }
            },
            "required": ["todos"]  # 必须传入todos参数
        }
    },
]

# =============================================================================
#  TOOL_HANDLERS：工具映射表
#  作用：把 AI 调用的工具名（如 bash）→ 绑定到 Python 函数（如 run_bash）
#  当 AI 调用 "bash" 时，程序就执行 run_bash() 函数
# =============================================================================
TOOL_HANDLERS = {
    "bash": run_bash,          # 执行命令
    "read_file": run_read,     # 读文件
    "write_file": run_write,   # 写文件
    "edit_file": run_edit,     # 改文件
    "glob": run_glob,          # 搜索文件
    "todo_write": run_todo_write,  # 管理待办任务
}
```



```python
# ═══════════════════════════════════════════════════════════
#  FROM s04 (unchanged): Hook System
#  钩子系统（从 s04 版本继承，未修改）
#  作用：在程序运行的关键节点，自动执行一些额外功能（如权限检查、日志、统计）
# ═══════════════════════════════════════════════════════════

# 定义4种事件钩子（相当于4个“监听槽”）
# 每个槽里可以放多个函数，事件触发时自动执行
HOOKS = {
    "UserPromptSubmit": [],  # 用户输入问题后触发
    "PreToolUse": [],        # AI 调用工具**之前**触发（最常用：权限检查）
    "PostToolUse": [],       # AI 调用工具**之后**触发
    "Stop": []               # 对话结束时触发
}

# 注册钩子函数
# 作用：把一个函数，添加到指定事件的监听列表里
# event：事件名（如 PreToolUse）
# callback：要执行的函数
def register_hook(event: str, callback):
    HOOKS[event].append(callback)

# 触发钩子
# 作用：当某个事件发生时，执行这个事件下所有注册的函数
def trigger_hooks(event: str, *args):
    # 遍历该事件下的所有钩子函数
    for callback in HOOKS[event]:
        # 执行钩子函数
        result = callback(*args)
        # 如果某个钩子返回了内容（比如拒绝执行），立即返回，不再执行后续
        if result is not None:
            return result
    return None

```



```python
# ==================== 以下是具体的钩子功能 ====================

# 危险命令黑名单（禁止执行的系统命令）
DENY_LIST = [
    "rm -rf /",    # 强制删除根目录（毁灭性命令）
    "sudo",        # 提权
    "shutdown",    # 关机
    "reboot",      # 重启
    "mkfs",        # 格式化磁盘
    "dd if="       # 写磁盘
]

# 权限安全钩子
# 触发时机：PreToolUse（AI 调用工具前）
# 作用：检查 bash 命令是否是危险命令，如果是，直接阻止执行
def permission_hook(block):
    """PreToolUse: deny list check."""
    # 如果 AI 要调用的是 bash 工具
    if block.name == "bash":
        # 遍历黑名单
        for p in DENY_LIST:
            # 如果命令包含危险指令
            if p in block.input.get("command", ""):
                print(f"\n\033[31m⛔ Blocked: '{p}'\033[0m")  # 红色提示：已拦截
                return "Permission denied"  # 返回错误，阻止执行
    return None  # 安全，放行
  
# 日志钩子
# 触发时机：PreToolUse（AI 调用工具前）
# 作用：灰色文字打印 AI 正在调用什么工具，方便查看流程
def log_hook(block):
    """PreToolUse: log tool calls."""
    print(f"\033[90m[HOOK] {block.name}\033[0m")
    return None
```

**block长什么样子？**

它是一个**工具调用的 “包裹对象”**，里面装着：

- 工具名字（bash /read_file 等）
- 传入的参数（command /path 等）

它**不是字典**，是一个简易对象，但你可以把它理解成**下面这个样子**：

例子 1：AI 调用 bash 工具时的 block

```
block.name = "bash"
block.input = {
    "command": "ls -l"
}
```

如果打印出来，长这样：

```
block = ToolBlock(
    name="bash",
    input={"command": "ls -l"}
)
```

例子 2：AI 调用 read_file 时的 block

```
block.name = "read_file"
block.input = {
    "path": "main.py"
}
```

例子 3：危险命令（会被 permission_hook 拦截）

```
block.name = "bash"
block.input = {
    "command": "sudo rm -rf /"
}
```

这时 `permission_hook` 就会触发：

```
if block.name == "bash":
    if "sudo" in block.input["command"]:
        拦截！
```

最直观的总结`block` 就是：

**AI 要调用的工具信息包**

包含：

- `block.name` → 工具名（bash、read_file...）
- `block.input` → 参数（命令、路径、内容...）

```python
# 上下文注入钩子
# 触发时机：UserPromptSubmit（用户刚输入完问题）
# 作用：提示当前工作目录，方便调试
def context_inject_hook(query: str):
    """UserPromptSubmit: log working directory."""
    print(f"\033[90m[HOOK] UserPromptSubmit: working in {WORKDIR}\033[0m")
    return None

# 会话总结钩子
# 触发时机：Stop（对话结束）
# 作用：统计整个对话过程中 AI 一共调用了多少次工具
def summary_hook(messages: list):
    """Stop: print tool call count."""
    # 计算消息里有多少个 tool_result（工具调用次数）
    tool_count = sum(1 for m in messages
                     for b in (m.get("content") if isinstance(m.get("content"), list) else [])
                     if isinstance(b, dict) and b.get("type") == "tool_result")
    # 灰色打印统计结果
    print(f"\033[90m[HOOK] Stop: session used {tool_count} tool calls\033[0m")
    return None
```

**messages长什么样子？**

```
messages = [
    # 1. 用户发的消息
    {
        "role": "user",
        "content": "帮我写一个hello.py文件"
    },

    # 2. AI 回复：决定调用 write_file 工具
    {
        "role": "assistant",
        "content": [
            {
                "type": "tool_use",
                "name": "write_file",
                "input": {
                    "path": "hello.py",
                    "content": "print('hello')"
                }
            }
        ]
    },

    # 3. 工具执行结果返回
    {
        "role": "user",
        "content": [
            {
                "type": "tool_result",
                "content": "Wrote 12 bytes to hello.py"
            }
        ]
    },

    # 4. AI 最后总结
    {
        "role": "assistant",
        "content": "已成功创建hello.py文件"
    }
]
```



```python
# ==================== 把钩子注册到系统中 ====================

# 用户提交问题 → 执行 context_inject_hook（显示当前目录）
register_hook("UserPromptSubmit", context_inject_hook)

# AI 调用工具前 → 先执行 permission_hook（权限检查）
register_hook("PreToolUse", permission_hook)

# AI 调用工具前 → 再执行 log_hook（打印日志）
register_hook("PreToolUse", log_hook)

# 对话结束 → 执行 summary_hook（统计工具调用次数）
register_hook("Stop", summary_hook)
```

tool_count = sum(1 for m in messages                

for b in (m.get("content") if isinstance(m.get("content"), list) else [])                 

if isinstance(b, dict) and b.get("type") == "tool_result")

第一层循环：`for m in messages`

- `messages` = **整个对话历史列表**
- `m` = 每一条消息（用户说的、AI 说的、工具返回的）

**作用：把所有消息一条一条拿出来看**

第二层循环：`for b in m.get("content")`

- 每条消息 `m` 里有个字段叫 `content`（内容）
- 有时候 `content` 是**一整条文字**
- 有时候 `content` 是**一个列表**（包含工具调用）

代码里这句：

```
m.get("content") if isinstance(m.get("content"), list) else []
```

**翻译：**

> 如果 content 是列表，就遍历它；
>
> 如果不是列表，就当空列表处理（跳过）。

**作用：只看 content 是列表的消息（只有这种消息里有工具记录）**

条件判断：`if b.get("type") == "tool_result"`

- `b` = content 里的每一项
- 判断：**这一项是不是 “工具执行结果”**

工具结果长这样：

```
{"type": "tool_result", "content": "..."}
```

**作用：只数工具调用成功的记录**

最后：`sum(1 for ...)`

每找到一个 `tool_result`，就 **+1**

最后加起来 = **工具调用总次数**

count = sum(1 for n in nums if n > 3)

每找到一个 `n > 3`，就**产出一个 `1`**

`sum` 把所有的 `1` 加起来

**结果就是个数！**

```python
# ═══════════════════════════════════════════════════════════
#  agent_loop — same as s04 + nag reminder counter
#  AI 智能体主循环（核心大脑）
#  功能：和 AI 模型持续对话、调用工具、执行任务
#  s05 新增：如果 AI 太久没更新待办，会自动提醒它
# ═══════════════════════════════════════════════════════════

# 全局计数器：记录 AI 已经有多少轮没有更新待办任务了
rounds_since_todo = 0

# 定义 AI 主循环函数
# messages: 对话历史记录
def agent_loop(messages: list):
    # 声明使用全局的计数器
    global rounds_since_todo
    
    # 死循环：持续和 AI 交互，直到任务完成
    while True:
        # ===================== s05 新增功能：自动提醒更新待办 ===================
        # 如果 AI 超过 3 轮都没更新待办，并且有对话历史
        if rounds_since_todo >= 3 and messages:
            # 给 AI 偷偷塞一条提醒消息：“更新你的待办清单”
            messages.append({
                "role": "user",
                "content": "<reminder>Update your todos.</reminder>"
            })
            # 重置计数器
            rounds_since_todo = 0

        # ===================== 第一步：调用大模型 API 获取回复 ==================
        response = client.messages.create(
            model=MODEL,        # 使用的模型（如 Claude 3.5）
            system=SYSTEM,      # 系统提示词（人设：先规划再执行）
            messages=messages,  # 传入完整的对话历史
            tools=TOOLS,        # 告诉 AI 有哪些工具可以用
            max_tokens=8000,    # 最大输出长度
        )

        # ===================== 第二步：把 AI 的回复加入对话历史 =================
        messages.append({"role": "assistant", "content": response.content})

        # ===================== 第三步：判断 AI 是否要结束对话 ==================
        # 如果 AI 停止原因不是“需要调用工具”（说明回答完了）
        if response.stop_reason != "tool_use":
            # 触发“结束”钩子：统计工具调用次数
            force = trigger_hooks("Stop", messages)
            if force:
                # 如果钩子返回了强制消息，加入对话继续循环
                messages.append({"role": "user", "content": force})
                continue
            # 结束循环，返回结果给用户
            return

        # ===================== 第四步：准备执行 AI 想调用的工具 =================
        # 待办计数器 +1（因为又执行了一轮工具）
        rounds_since_todo += 1
        # 用来存放所有工具的执行结果
        results = []

        # 遍历 AI 返回的内容块
        for block in response.content:
            # 只处理“工具调用”类型的块
            if block.type != "tool_use":
                continue

            # --- 钩子触发：工具使用前 --- (权限检查、打印日志)
            blocked = trigger_hooks("PreToolUse", block)
            if blocked:
                # 如果被拦截（如危险命令），把拦截信息作为结果返回
                results.append({
                    "type": "tool_result", 
                    "tool_use_id": block.id, 
                    "content": str(blocked)
                })
                continue  # 跳过执行真正的命令

            # --- 执行工具 ---
            # 根据工具名找到对应的处理函数
            handler = TOOL_HANDLERS.get(block.name)
            # 调用函数，传入参数 (例如 run_bash(command="ls"))
            output = handler(**block.input) if handler else f"Unknown: {block.name}"

            # --- 钩子触发：工具使用后 ---
            trigger_hooks("PostToolUse", block, output)

            # --- s05 新增：如果 AI 调用了待办工具，重置提醒计数器 ---
            if block.name == "todo_write":
                rounds_since_todo = 0

            # 把工具执行结果打包
            results.append({
                "type": "tool_result", 
                "tool_use_id": block.id, 
                "content": output
            })

        # ===================== 第五步：把工具结果返回给 AI =====================
        # 将工具执行结果作为用户消息，发回给 AI，让它继续下一步思考
        messages.append({"role": "user", "content": results})
```

**response长什么样？**

**类型：Anthropic SDK 官方返回的对象**

不是字典！不是字符串！

是一个**类对象**：`<class 'anthropic.types.Message'>`

你可以把它理解成一个**装着 AI 全部回复的包裹**。

它长什么样？（真实结构）

我给你看**AI 调用工具**和**普通回答**两种最常见的样子！

情况 A：AI 要调用工具（最常见）

```
response.id = "msg_123456"
response.model = "claude-3-5-sonnet-20240620"
response.stop_reason = "tool_use"  # 关键：表示AI要调用工具

# 最重要的部分：response.content
response.content = [
    # 这就是 block 对象！
    ToolBlock(
        type="tool_use",
        id="toolu_123",
        name="bash",
        input={"command": "ls -l"}
    )
]
```

情况 B：AI 直接回答你（不调用工具）

```
response.stop_reason = "end_turn"  # 表示回答完毕
response.content = [
    TextBlock(
        type="text",
        text="我已经帮你创建好文件了！"
    )
]
```



```python
if __name__ == "__main__":
    # 当脚本作为主程序直接运行时（而不是被其他模块导入），执行以下代码块
    print("s06: Subagent — spawn sub-agents with fresh context, summary only")
    print("Type a question, press Enter. Type q to quit.\n")

    history = []   # 存储整个对话历史，每个元素是一个消息字典（role 和 content）
    while True:    # 无限循环，持续接收用户输入直到主动退出
        try:
            # 提示符使用 ANSI 转义码 \033[36m 显示青色，用户输入前打印 "s06 >> "
            query = input("\033[36ms06 >> \033[0m")
        except (EOFError, KeyboardInterrupt):
            # 如果用户按下 Ctrl+D (EOF) 或 Ctrl+C (KeyboardInterrupt)，退出循环
            break

        # 若用户输入仅为空格、q、exit 或空字符串，则退出循环
        if query.strip().lower() in ("q", "exit", ""):
            break

        # 调用一个钩子函数（可能用于日志、监控或拦截用户提交的提示词）
        trigger_hooks("UserPromptSubmit", query)

        # 将用户消息添加到历史记录中，角色为 "user"，内容为用户输入的字符串
        history.append({"role": "user", "content": query})

        # 调用核心的 agent_loop 函数，该函数会处理当前历史（可能包含工具调用、子代理等）
        # 并更新 history 列表（最后一条通常为 assistant 的回复）
        agent_loop(history)

        # 遍历历史列表中最后一条消息（即刚刚由 agent_loop 追加的助手回复）的 content 部分
        # 该 content 可能是一个列表，包含文本块、工具调用等对象
        for block in history[-1]["content"]:
            # 如果当前块有 type 属性且等于 "text"，则打印其文本内容
            if getattr(block, "type", None) == "text":
                print(block.text)

        # 打印一个空行，使下一轮输入与上一轮输出分隔开
        print()
```

