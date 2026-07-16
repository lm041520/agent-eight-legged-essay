# 01 Python与工程能力

> 定位：面向大模型应用、Agent、RAG和AI后端岗位的Python工程教程。
>
> 示例基线：代码优先兼容Python 3.11及以上版本；涉及`TaskGroup`、`asyncio.timeout()`、Pydantic v2等版本敏感内容时会单独标注。当前版本行为应以文末官方文档为准。

---

## 零基础导读：先看懂一个Agent请求是怎样被Python程序处理的

如果一上来分别学习变量、类、协程、HTTP和异常，很容易觉得这些知识彼此无关。先从一个具体请求开始：

```text
用户：查询杭州今天的天气，并告诉我是否需要带伞。
```

一个最小Agent后端大致会经历下面的过程：

```text
1. FastAPI收到HTTP请求
2. Pydantic把JSON校验成Python对象
3. Agent把用户问题发给模型
4. 模型决定调用get_weather工具
5. Python程序读取工具名和参数
6. 程序异步请求天气API
7. 天气结果返回后写入对话状态
8. 模型根据天气结果生成建议
9. 服务通过SSE逐段返回给前端
10. 日志和Trace记录每一步耗时与错误
```

这十步几乎对应本模块的全部知识。下面用一份极简代码先建立整体感觉：

```python
from pydantic import BaseModel, Field


class ChatRequest(BaseModel):
    """描述前端允许提交的数据结构。"""

    message: str = Field(min_length=1, max_length=2000)
    user_id: str


async def get_weather(city: str) -> dict:
    """真实项目中这里会异步访问外部天气服务。"""

    return {
        "city": city,
        "weather": "小雨",
        "temperature": 22,
    }


async def handle_request(raw_json: dict) -> str:
    # 第一步：把不可信的外部JSON转换成经过校验的Python对象。
    request = ChatRequest.model_validate(raw_json)

    # 为了突出Python流程，这里暂时假设Agent已经判断出城市是杭州。
    tool_result = await get_weather("杭州")

    if tool_result["weather"] == "小雨":
        advice = "建议带伞"
    else:
        advice = "通常不需要带伞"

    return f"{tool_result['city']}今天{tool_result['weather']}，{advice}。"
```

即使暂时看不懂所有语法，也可以先识别五类东西。

### 1. 数据：程序正在保存什么信息

`raw_json`、`request`、`tool_result`和`advice`都是变量。变量可以理解为“给一份数据起一个名字”，这样后续代码才能引用它。

```python
city = "杭州"                  # 字符串
temperature = 22              # 整数
need_umbrella = True           # 布尔值
weather = {"type": "小雨"}   # 字典
```

学习Python容器时，不要只背`list`、`dict`、`set`的定义，而要问“这份数据在Agent流程中代表什么”。例如：

- `list`适合保存消息序列；
- `dict`适合表示一次工具调用或JSON对象；
- `set`适合保存已经处理过的文档ID，避免重复；
- `deque`适合实现任务队列或固定长度历史窗口。

### 2. 函数：把一个步骤封装成可重复执行的能力

`get_weather(city)`是函数。调用者只需要提供城市，不必关心内部怎样访问天气服务。

```text
输入：city="杭州"
处理：调用天气API并解析响应
输出：{"weather": "小雨", "temperature": 22}
```

函数的价值不只是“少写几行代码”，而是建立清晰边界：输入应该是什么、可能失败在哪里、返回值是什么。Agent工具本质上也是一类具有明确输入输出的函数。

### 3. 类和对象：把相关数据与规则组织在一起

`ChatRequest`是类，可以把它理解为请求数据的“设计图”；`request`是根据设计图创建的具体对象。

```text
类：规定每个聊天请求必须有message和user_id
对象：某个用户本次实际提交的那一份请求
```

Pydantic不仅保存字段，还会执行校验。例如空字符串、缺少`user_id`或超长文本会在进入核心业务前被拒绝，避免错误数据一路传到模型和数据库。

### 4. `async`和`await`：等待外部服务时把执行机会让出去

天气API可能需要500毫秒返回。如果普通同步程序在这500毫秒中什么都不做，一个进程可处理的并发请求就很有限。

`await get_weather(...)`表达的是：

```text
当前请求需要等待天气服务；
等待期间，事件循环可以先处理其他用户的请求；
天气结果回来后，再从这里继续。
```

它不是“让单次天气请求凭空变快”，而是提高大量I/O等待任务同时存在时的资源利用率。

### 5. 异常：现实世界中的步骤随时可能失败

天气服务可能超时、返回401、返回无法解析的JSON，用户也可能主动断开连接。工程代码必须区分失败类型：

```python
async def safe_weather(city: str) -> dict:
    try:
        return await get_weather(city)
    except TimeoutError as exc:
        raise RuntimeError("天气服务响应超时") from exc
```

这里的`from exc`保留底层异常链。排查时既能看到业务层的“天气服务超时”，也能继续找到真正的网络错误。

### 怎样阅读后面的代码

遇到一段陌生代码时，先不要逐字符研究，按下面顺序阅读：

1. 找入口：哪个函数最先被调用？
2. 找输入：参数和外部数据来自哪里？
3. 找主流程：正常情况下按什么顺序执行？
4. 找边界：数据库、HTTP、模型、Redis在哪里被调用？
5. 找失败：异常被谁捕获，是否重试或回滚？
6. 找输出：最终返回给调用者什么？

带着这个请求流程继续学习，后面的Python语法、并发、HTTP、测试和Linux就不再是互相孤立的知识点。

---

## 一、为什么Agent开发首先是工程开发

大模型可以生成文本、选择工具和规划任务，但它不会替你的程序完成以下工程工作：

- 接收用户请求并校验参数；
- 调用模型、搜索、数据库和业务API；
- 管理对话、任务状态和工具结果；
- 控制并发，避免模型服务被瞬间打满；
- 处理超时、限流、重试、取消和部分失败；
- 通过SSE或WebSocket返回实时进度；
- 保存日志、指标和Trace；
- 使用测试保证修改后旧能力不会失效。

因此，Agent应用不是“写一个Prompt然后调用API”，而是一个以大模型为决策组件的后端系统。Python语法只是起点，真正高频的能力是数据建模、异步并发、异常边界、HTTP接口和可测试性。

### 本模块学习目标

完成本模块后，你应该能够：

1. 解释Python对象、引用、可变性、函数和类型系统；
2. 使用Pydantic定义API和工具参数；
3. 正确选择线程、进程和协程；
4. 编写带超时、取消、并发限制和部分失败处理的异步程序；
5. 理解常见数据结构在Agent系统中的用途；
6. 理解HTTP、SSE、WebSocket和API可靠性设计；
7. 使用Linux、Git和pytest完成基本工程协作；
8. 综合实现一个异步工具执行器。

### 优先级

| 内容 | 优先级 | 达标要求 |
|---|---:|---|
| Python对象、函数、异常 | P0 | 能解释、编码、排错 |
| 类型标注、Pydantic | P0 | 能设计工具和API数据模型 |
| asyncio | P0 | 能控制并发、超时和取消 |
| HTTP与流式输出 | P0 | 能调用和设计模型接口 |
| 数据结构与算法 | P0 | 能完成基础面试并映射工程场景 |
| Linux、Git | P0 | 能运行、排查和协作 |
| 线程、进程 | P1 | 能正确选型并避免共享状态问题 |
| 测试与代码质量 | P1 | 能测试工具、异步代码和外部依赖 |

---

# 第一章 Python语言与数据建模

## 1.1 Python变量保存的是对象引用

初学者容易把变量理解成一个固定的“盒子”。在Python中，更准确的模型是：对象存在于内存中，变量名绑定到对象。

```python
a = [1, 2]
b = a
b.append(3)

print(a)        # [1, 2, 3]
print(a is b)   # True
```

`b = a`没有复制列表，只是让`a`和`b`指向同一个对象。因此通过`b`修改列表，`a`看到的也是修改后的对象。

需要区分：

- `a == b`：比较值是否相等；
- `a is b`：比较是否为同一个对象；
- `id(a)`：对象身份的实现层标识，可用于学习观察，不应作为业务逻辑依据。

判断空值通常写：

```python
if result is None:
    ...
```

因为`None`是单例语义，代码想判断的是“它是否就是None”，而不是调用对象自定义的相等比较。

### 在Agent项目中的影响

如果多个请求共享同一个可变状态对象，可能出现跨用户污染：

```python
shared_state = {"messages": []}

def add_message(message: str) -> None:
    shared_state["messages"].append(message)
```

在Web服务中，这个模块级对象可能被不同请求共同修改。正确做法通常是为每个任务创建独立状态，或者将状态保存在带用户/会话隔离的存储中。

## 1.2 常用数据类型与容器

### 标量类型

- `int`：整数；
- `float`：浮点数，注意精度问题；
- `bool`：布尔值，`bool`是`int`的子类，但业务中不要混用；
- `str`：Unicode文本；
- `bytes`：字节数据；
- `None`：缺失或尚无结果。

文本和字节的边界很重要：网络、文件和模型权重最终都以字节传输；Python程序通常解码为字符串后再处理。

```python
text = "大模型"
raw = text.encode("utf-8")
restored = raw.decode("utf-8")

assert restored == text
```

乱码通常不是“中文坏了”，而是编码和解码规则不一致。例如UTF-8字节被错误地按GBK或本地代码页解释。

### List：有序、可变

适合保存消息、任务步骤和检索结果。

```python
messages = [
    {"role": "system", "content": "你是助手"},
    {"role": "user", "content": "解释RAG"},
]
messages.append({"role": "assistant", "content": "RAG是……"})
```

常见操作：

- `append(x)`：追加一个元素；
- `extend(xs)`：追加多个元素；
- `pop()`：删除并返回元素；
- 切片`items[start:end]`：生成新列表；
- `sorted(items, key=...)`：返回排序后的新列表。

### Tuple：有序、通常用于固定结构

Tuple本身不可变，适合表达不应被随意增删的组合值：

```python
memory_namespace = ("user-42", "preferences")
```

“Tuple不可变”指不能替换其中的槽位。如果Tuple内部保存了可变对象，该对象仍可能被修改。

### Dict：键值映射

适合表示Agent状态、配置和结构化结果。

```python
state = {
    "task_id": "task-001",
    "messages": [],
    "step": 0,
    "status": "running",
}
```

访问外部数据时，区分：

```python
state["status"]          # 缺少键时抛KeyError
state.get("error")       # 缺少键时返回None
state.get("retry", 0)    # 提供默认值
```

不要为了“避免报错”全部使用`get()`。必需字段缺失本来就是错误时，让程序尽早失败更容易定位问题。

### Set：无重复集合

适合去重和快速成员判断：

```python
called_tools: set[str] = set()

signature = "search:{query=python}"
if signature in called_tools:
    raise RuntimeError("检测到重复工具调用")
called_tools.add(signature)
```

### 容器选型

| 需求 | 推荐结构 | 原因 |
|---|---|---|
| 按顺序保存消息 | List | 保序、追加方便 |
| 通过名称查找工具 | Dict | 平均常数时间查找 |
| 防止重复调用 | Set | 成员判断和去重 |
| 固定命名空间 | Tuple | 结构稳定、可哈希 |
| 按优先级处理任务 | Heap | 快速取出最高/最低优先级 |

## 1.3 可变性、哈希与深浅拷贝

常见不可变对象包括整数、字符串、Tuple（前提是内部元素可哈希）。常见可变对象包括List、Dict和Set。

Dict的键和Set的元素必须可哈希，因为哈希值用于定位存储位置。可变列表不能作为Dict键：

```python
cache = {}
# cache[["a", "b"]] = 1  # TypeError: unhashable type: 'list'
cache[("a", "b")] = 1
```

### 浅拷贝

浅拷贝只复制最外层容器，内部对象仍然共享：

```python
from copy import copy

original = {"messages": [{"content": "hello"}]}
cloned = copy(original)
cloned["messages"][0]["content"] = "changed"

print(original["messages"][0]["content"])  # changed
```

### 深拷贝

`deepcopy()`递归复制内部对象：

```python
from copy import deepcopy

cloned = deepcopy(original)
```

但不要把深拷贝当作通用解决方案：

- 大状态深拷贝成本高；
- 数据库连接、锁、文件句柄等资源不能简单复制；
- 更好的设计是明确状态所有权，使用不可变更新或只复制需要修改的部分。

## 1.4 控制流与推导式

Agent循环本质上仍然是条件、循环和状态更新：

```python
MAX_STEPS = 8

for step in range(MAX_STEPS):
    action = decide_next_action()
    if action.kind == "finish":
        break
    execute(action)
else:
    raise RuntimeError("Agent超过最大执行步数")
```

这里`for ... else`中的`else`只会在循环没有通过`break`退出时执行，很适合检测“达到上限仍未完成”。

推导式适合简单转换：

```python
successful = [r for r in results if r.ok]
tool_names = {r.tool_name for r in successful}
latency_map = {r.tool_name: r.latency_ms for r in successful}
```

如果逻辑包含多层分支、异常处理或副作用，普通循环通常更清晰，不要为了短而写复杂推导式。

## 1.5 函数、参数与副作用

### 参数类型

```python
def call_tool(
    name: str,
    arguments: dict[str, object],
    *,
    timeout_s: float = 10.0,
    max_retries: int = 2,
) -> dict[str, object]:
    ...
```

`*`之后的参数必须通过名称传递：

```python
call_tool("search", {"query": "RAG"}, timeout_s=5.0)
```

这样可以避免调用者把`timeout_s`和`max_retries`的位置写反。

### `*args`与`**kwargs`

- `*args`收集额外位置参数；
- `**kwargs`收集额外关键字参数；
- 工具系统常用`**arguments`把经过校验的参数传给函数。

```python
def search(query: str, top_k: int = 5) -> list[str]:
    return [f"result:{query}:{i}" for i in range(top_k)]

arguments = {"query": "Agent", "top_k": 3}
results = search(**arguments)
```

不能直接相信模型生成的`arguments`。在展开参数之前，必须进行类型、范围和权限校验。

### 可变默认参数陷阱

```python
def append_message(message: str, messages: list[str] = []):
    messages.append(message)
    return messages
```

默认值在函数定义时创建一次，之后多次调用共享同一个列表。正确写法：

```python
def append_message(
    message: str,
    messages: list[str] | None = None,
) -> list[str]:
    if messages is None:
        messages = []
    messages.append(message)
    return messages
```

### 纯函数与副作用

纯函数只由输入决定输出，不修改外部状态，更容易测试：

```python
def build_tool_signature(name: str, arguments: dict[str, object]) -> str:
    ordered = sorted(arguments.items())
    return f"{name}:{ordered}"
```

调用数据库、写文件、发网络请求属于副作用。工程上应把纯逻辑和副作用分离：先计算“要做什么”，再由明确的执行层“真的去做”。

## 1.6 作用域、闭包与装饰器

Python查找变量常用LEGB顺序：Local、Enclosing、Global、Built-in。

闭包可以记住外层函数的局部变量：

```python
from collections.abc import Callable

def make_prefixer(prefix: str) -> Callable[[str], str]:
    def add_prefix(text: str) -> str:
        return f"{prefix}{text}"
    return add_prefix

error_message = make_prefixer("[ERROR] ")
print(error_message("tool failed"))
```

装饰器经常用于日志、计时、重试和权限检查：

```python
from functools import wraps
from time import perf_counter
from collections.abc import Callable
from typing import ParamSpec, TypeVar

P = ParamSpec("P")
R = TypeVar("R")

def timed(func: Callable[P, R]) -> Callable[P, R]:
    @wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        start = perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            elapsed_ms = (perf_counter() - start) * 1000
            print(f"{func.__name__} took {elapsed_ms:.2f} ms")
    return wrapper
```

使用`wraps`可以保留原函数名称、文档和部分类型/调试信息。装饰器不应隐藏太多业务流程，否则异常栈和执行顺序会变得难以理解。

## 1.7 迭代器、生成器与流式数据

Iterable表示“可以产生迭代器”，Iterator表示“可以逐个产生值并保存迭代位置”。生成器是编写迭代器的简洁方式。

```python
def stream_tokens(text: str):
    for token in text.split():
        yield token

for token in stream_tokens("Agent returns tokens gradually"):
    print(token)
```

生成器的价值：

- 不必一次把全部结果放入内存；
- 可以边生产边消费；
- 适合文件处理、数据管道和流式响应。

异步生成器可在等待网络数据时让出执行权：

```python
import asyncio
from collections.abc import AsyncIterator

async def stream_model_output() -> AsyncIterator[str]:
    for chunk in ["你", "好", "！"]:
        await asyncio.sleep(0.1)
        yield chunk
```

SSE服务经常消费异步生成器，将模型增量内容逐块发送给客户端。

## 1.8 上下文管理器与资源生命周期

`with`保证退出代码块时执行清理：

```python
with open("prompt.txt", encoding="utf-8") as file:
    prompt = file.read()
```

即使读取过程中发生异常，文件也会关闭。数据库连接、HTTP Client、锁和临时文件同样需要明确生命周期。

异步资源使用`async with`：

```python
import httpx

async def fetch_json(url: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        response.raise_for_status()
        return response.json()
```

真实服务通常不会为每次请求新建Client，而是在应用生命周期内复用连接池，再在服务关闭时统一释放。连接池复用的深入设计放在模块05。

## 1.9 异常处理与异常边界

异常不是“程序不够健壮”，而是程序表达失败状态的机制。正确目标不是捕获所有异常，而是在合适的层级增加上下文、释放资源并决定重试、降级或返回错误。

```python
class AgentError(Exception):
    """Agent系统可预期异常的基类。"""


class ToolExecutionError(AgentError):
    def __init__(self, tool_name: str, message: str):
        super().__init__(f"tool={tool_name}: {message}")
        self.tool_name = tool_name


def parse_tool_result(tool_name: str, raw: str) -> dict:
    import json

    try:
        return json.loads(raw)
    except json.JSONDecodeError as exc:
        raise ToolExecutionError(tool_name, "返回值不是合法JSON") from exc
```

`raise ... from exc`保留了底层原因，排查时既能看到业务语义，也能看到最初的JSON错误。

### 异常处理原则

1. 捕获你能处理的具体异常；
2. 无法处理时保留上下文继续抛出；
3. `finally`用于必须执行的清理；
4. 不要空写`except Exception: pass`；
5. 重试只适合暂时性故障，不适合参数错误和权限错误；
6. 用户响应中隐藏密钥和内部堆栈，日志中保留可定位的请求ID。

### 异常分类

| 类型 | 示例 | 常见处理 |
|---|---|---|
| 输入错误 | 参数缺失、类型错误 | 直接拒绝，提示修正 |
| 权限错误 | 越权工具、无权文档 | 拒绝并审计，不重试 |
| 暂时性外部错误 | 429、网络抖动 | 限次重试、退避 |
| 持久性外部错误 | API Key失效、404 | 停止重试、报警 |
| 内部程序错误 | None访问、状态冲突 | 记录堆栈、回归修复 |

## 1.10 面向对象：继承、组合与协议

面向对象的主要价值不是“把所有东西写成类”，而是封装状态、定义边界和替换实现。

```python
from abc import ABC, abstractmethod
from typing import Any

class Tool(ABC):
    name: str

    @abstractmethod
    async def run(self, **arguments: Any) -> Any:
        raise NotImplementedError
```

但Python项目中更灵活的方式是使用`Protocol`进行结构化类型约束：

```python
from typing import Any, Protocol

class ToolProtocol(Protocol):
    name: str

    async def run(self, **arguments: Any) -> Any:
        ...
```

只要对象具有对应属性和方法，就可以满足协议，不要求继承指定基类。

### 继承与组合

- 继承表达“它是一种什么”；
- 组合表达“它使用什么完成工作”；
- 工程中优先使用组合，减少父类状态和隐式行为的耦合。

```python
class ToolExecutor:
    def __init__(self, registry: "ToolRegistry", logger: "Logger"):
        self.registry = registry
        self.logger = logger
```

`ToolExecutor`不需要继承注册表和日志类，而是组合它们。

## 1.11 类型标注

类型标注不会自动阻止所有运行时错误，但可以：

- 让IDE补全更准确；
- 在调用前发现参数不匹配；
- 明确`None`、异常和返回结构；
- 让工具、状态和服务接口更容易阅读。

```python
from collections.abc import Awaitable, Callable
from typing import Literal, TypedDict

ToolStatus = Literal["success", "failed", "timeout"]
ToolHandler = Callable[[dict[str, object]], Awaitable[dict[str, object]]]

class AgentState(TypedDict):
    task_id: str
    step: int
    status: str
    messages: list[dict[str, str]]
```

常见类型：

- `X | None`：可能缺失；
- `Literal[...]`：限定字符串或数值集合；
- `Callable`：函数签名；
- `TypedDict`：描述字典键；
- `Protocol`：描述对象能力；
- `TypeVar`和泛型：让容器、仓储或执行器保留具体类型。

不要用`Any`解决所有类型报错。`Any`会关闭该位置的大部分静态检查，应该只放在真正的动态边界，再尽快校验并转换为具体类型。

## 1.12 Dataclass与Pydantic v2

### Dataclass

Dataclass适合Python内部的轻量数据对象：

```python
from dataclasses import dataclass, field

@dataclass(slots=True)
class ToolResult:
    tool_name: str
    ok: bool
    data: dict[str, object] = field(default_factory=dict)
    error: str | None = None
```

`default_factory`避免多个实例共享同一个Dict。`slots=True`可减少某些对象的内存开销并限制随意添加属性，但不是所有数据类都必须使用。

### Pydantic v2

Pydantic适合不可信数据边界，例如API输入、配置和模型生成的工具参数。它根据类型标注执行运行时校验，并可生成JSON Schema。

```python
from pydantic import BaseModel, Field, field_validator

class SearchInput(BaseModel):
    query: str = Field(min_length=1, max_length=500)
    top_k: int = Field(default=5, ge=1, le=20)

    @field_validator("query")
    @classmethod
    def normalize_query(cls, value: str) -> str:
        normalized = value.strip()
        if not normalized:
            raise ValueError("query不能为空")
        return normalized


payload = SearchInput.model_validate({"query": "  Agent memory ", "top_k": 3})
print(payload.model_dump())
print(SearchInput.model_json_schema())
```

这一模型可以同时服务于：

- FastAPI请求体；
- Function Calling参数Schema；
- 工具执行前校验；
- 测试数据构造；
- OpenAPI文档生成。

### Dataclass、TypedDict、Pydantic如何选择

| 方案 | 是否运行时校验 | 适合场景 |
|---|---:|---|
| TypedDict | 否 | 内部状态的静态类型提示 |
| Dataclass | 默认较弱 | 内部领域对象、计算结果 |
| Pydantic | 是 | API、配置、工具参数等外部边界 |

Pydantic校验有成本。高频内部状态如果已经可信，不必每一步都重复构造复杂Pydantic模型。

## 1.13 模块、包和项目结构

建议按职责组织代码，而不是把所有逻辑放在一个`main.py`：

```text
agent_project/
├── pyproject.toml
├── README.md
├── src/
│   └── agent_app/
│       ├── __init__.py
│       ├── schemas.py
│       ├── exceptions.py
│       ├── tools/
│       ├── services/
│       └── main.py
└── tests/
```

需要理解：

- 模块是Python文件；
- 包是可导入的模块集合；
- 绝对导入通常比跨层级相对导入更清楚；
- 循环导入往往说明模块职责或依赖方向混乱；
- `if __name__ == "__main__":`用于直接运行入口，导入模块时不会执行该代码块。

### 依赖管理

现代项目优先使用`pyproject.toml`描述项目和依赖。无论使用`pip`、`uv`或Poetry，都要做到：

- 使用虚拟环境隔离项目；
- 锁定可复现依赖；
- 区分运行依赖与开发依赖；
- 不把密钥写进依赖或源码；
- 升级框架后运行回归测试。

## 1.14 第一章常见错误与排查

### 错误一：请求之间共享列表

**现象：** 用户A的消息出现在用户B会话中。

**排查：** 搜索模块级List/Dict、类属性和可变默认参数，记录对象`id`仅用于诊断是否共享。

**修复：** 每个请求创建独立状态；使用`default_factory`；持久化时用用户和会话ID隔离。

### 错误二：Pydantic字段看起来正确但仍失败

**可能原因：** 输入层级错误、别名不一致、字符串包含空格、校验器修改或拒绝了值。

**排查：** 打印脱敏后的原始输入和`ValidationError.errors()`，对照Schema，不要只看最终字符串错误。

### 错误三：异常被包装后看不到真实原因

**原因：** 捕获异常后只抛出新的普通字符串，丢失原始异常链。

**修复：** 使用`raise NewError(...) from exc`，日志中记录完整堆栈和请求ID。

### 错误四：循环导入

**原因：** A模块导入B，B又在初始化阶段导入A。

**修复顺序：**

1. 抽取共同的数据模型或接口；
2. 调整依赖方向；
3. 必要时仅为类型检查使用`TYPE_CHECKING`；
4. 局部导入只能作为明确权衡，不应掩盖架构问题。

## 1.15 第一章面试表达

### 问：为什么Agent项目中要使用Pydantic？

一分钟回答：

> 大模型生成的工具参数和用户提交的API数据都属于不可信输入，不能直接传给业务函数。Pydantic可以根据类型标注做运行时校验、范围约束和自定义清洗，同时生成JSON Schema，因此同一个模型可以用于FastAPI请求、Function Calling参数描述和工具执行前校验。项目中我会把Pydantic放在系统边界，内部可信数据根据性能和复杂度选择Dataclass或TypedDict，避免无意义的重复校验。

### 问：为什么不应该写`except Exception: pass`？

> 它会把参数错误、网络错误和程序Bug全部吞掉，使上层误以为任务成功，也破坏日志和重试判断。正确做法是捕获能够处理的具体异常，在业务边界增加上下文并保留异常链；无法恢复的错误继续抛出，由统一异常处理层记录和转换响应。

## 1.16 第一章复习卡

- 一句话：Python变量绑定对象，工程代码要明确数据所有权、边界校验和异常传播。
- 五个关键词：引用、可变性、类型标注、Pydantic、异常链。
- 必会代码：可变默认参数修复、Pydantic模型、自定义异常、生成器。
- 易错点：`is`与`==`、浅拷贝、共享状态、吞异常、滥用`Any`。
- 项目连接：工具参数校验、Agent State、流式输出、错误分类。

---

# 第二章 线程、进程、异步与并发

## 2.1 先建立统一概念

- **同步：** 调用者等待当前操作结束后再继续。
- **异步：** 调用者可以在等待期间推进其他任务，之后再取得结果。
- **阻塞：** 当前执行单元因为等待而无法继续做其他工作。
- **非阻塞：** 操作不能立即完成时先返回，由程序稍后继续处理。
- **并发：** 多个任务在一段时间内交替推进。
- **并行：** 多个任务在同一时刻真正执行。
- **线程：** 同一进程内的执行单元，通常共享内存。
- **进程：** 拥有独立地址空间的执行实例。
- **协程：** 可以在`await`等位置主动暂停和恢复的轻量任务。

Agent系统的大部分等待来自网络I/O：模型API、检索服务、数据库和业务工具。因此`asyncio`是应用岗P0；CPU密集的图像处理、复杂解析和本地推理则需要进程、原生扩展或GPU。

## 2.2 GIL与free-threaded Python

经典CPython构建中，全局解释器锁（GIL）使同一进程内通常只有一个线程执行Python字节码，因此多线程不会自动让纯Python CPU计算按核心数线性加速。但线程在等待网络、磁盘或释放GIL的原生库时仍可并发推进，所以适合I/O密集任务和包装同步SDK。

用两个任务区分最容易理解：

```text
任务A：向模型API发请求，然后等待2秒网络响应
任务B：用纯Python循环计算两亿次加法
```

任务A等待网络时不需要CPU，线程可以让另一个任务继续运行，因此多个I/O请求能重叠等待。任务B却一直执行Python字节码；在经典GIL构建中，两个线程只能频繁轮流拿锁，通常不能让两个CPU核心同时全速执行这两段Python循环，还会增加切换开销。

GIL是CPython解释器实现中的锁，不等于“Python程序永远只能用一个核心”。以下情况仍可能并行：多进程拥有不同解释器；NumPy/PyTorch等原生代码可在计算时释放GIL；GPU kernel 在设备上并行；多个服务进程也能由操作系统分配到不同核心。

当前Python还存在可选的free-threaded构建，可禁用GIL并允许Python线程真正并行。不能因此在面试或项目中直接说“Python已经没有GIL”：是否启用取决于解释器构建、依赖兼容性和部署环境。工程选型前应确认实际运行时。

free-threaded 也不代表把线程数设成100就会快100倍。共享对象仍需同步，缓存一致性、内存带宽、锁竞争和第三方扩展兼容性仍然存在。面试最稳妥的回答是先说明“经典CPython构建下”的结论，再补充可选 free-threaded 构建。

### 选型结论

| 场景 | 首选 |
|---|---|
| 并发调用LLM/HTTP工具 | asyncio |
| 只有同步接口的I/O SDK | 线程池或`to_thread` |
| 纯Python CPU密集计算 | 多进程 |
| NumPy/PyTorch/GPU计算 | 使用库自身并行或GPU能力 |
| 后台长任务、跨服务任务 | 消息队列/任务系统，见模块05 |

## 2.3 多线程与共享状态

```python
from concurrent.futures import ThreadPoolExecutor
from time import sleep

def blocking_call(name: str) -> str:
    sleep(0.5)
    return f"done:{name}"

with ThreadPoolExecutor(max_workers=4) as pool:
    results = list(pool.map(blocking_call, ["a", "b", "c"]))
```

线程共享同一进程内存，因此写共享变量时可能出现竞态条件。`x += 1`不是“业务上不可分割”的承诺，读、计算、写之间可能被其他线程介入。

常用同步原语：

- `Lock`：同一时刻一个线程进入临界区；
- `RLock`：同一线程可重复获取；
- `Semaphore`：限制同时进入的线程数；
- `Event`：一个线程通知其他线程；
- `Condition`：等待某个共享条件成立；
- 线程安全Queue：生产者—消费者通信。

### 避免死锁

死锁常见于线程A持有锁1等待锁2，线程B持有锁2等待锁1。预防方式：

1. 统一获取多个锁的顺序；
2. 缩小临界区；
3. 不在持锁期间执行慢网络请求；
4. 必要时使用超时；
5. 优先通过消息传递减少共享可变状态。

## 2.4 多进程

多进程通过独立解释器和地址空间绕开经典GIL对纯Python CPU计算的限制，但代价更高：

- 创建和切换成本高；
- 参数和结果通常需要序列化；
- 进程间不直接共享普通变量；
- 大对象复制可能占用大量内存；
- Windows进程启动方式要求保护入口。

```python
from concurrent.futures import ProcessPoolExecutor

def cpu_work(n: int) -> int:
    return sum(i * i for i in range(n))

if __name__ == "__main__":
    with ProcessPoolExecutor() as pool:
        print(list(pool.map(cpu_work, [100_000, 200_000])))
```

进程间通信可以使用Queue、Pipe、共享内存或外部存储。不要为了共享大量复杂状态而强行多进程；应先判断计算是否真的成为瓶颈。

## 2.5 asyncio运行模型

事件循环负责调度Task。协程执行到尚未完成的`await`时暂停，把执行权交回事件循环；等待对象就绪后，事件循环再恢复协程。

```python
import asyncio

async def fetch(name: str, delay: float) -> str:
    await asyncio.sleep(delay)
    return f"result:{name}"

async def main() -> None:
    first = asyncio.create_task(fetch("search", 0.4))
    second = asyncio.create_task(fetch("database", 0.2))
    results = await asyncio.gather(first, second)
    print(results)

asyncio.run(main())
```

需要区分：

- Coroutine function：使用`async def`定义；
- Coroutine object：调用协程函数后得到，但尚不代表已经执行；
- Task：把协程交给事件循环调度；
- Future：表示未来会出现的结果，应用代码通常不需要手动创建。

只调用协程函数而不`await`或创建Task，会出现“coroutine was never awaited”警告。

## 2.6 `gather`与`TaskGroup`

`asyncio.gather()`按传入顺序聚合结果，适合并发收集一组相对独立的任务：

```python
results = await asyncio.gather(
    search_tool(),
    database_tool(),
    return_exceptions=True,
)
```

`return_exceptions=True`会把异常作为结果返回，调用者必须逐项检查，不能当作成功数据。

例如 Agent 同时查天气、新闻和用户画像，新闻失败也允许用另外两项继续回答，适合部分成功：

```python
results = await asyncio.gather(
    get_weather(),
    get_news(),
    get_profile(),
    return_exceptions=True,
)

for result in results:
    if isinstance(result, Exception):
        print("一个可选工具失败:", result)
    else:
        print("可用结果:", result)
```

结果顺序与传入顺序一致，而不是完成顺序。若忘记检查异常，后续代码可能把 `TimeoutError` 对象当成字典使用。

Python 3.11增加了`TaskGroup`，用于结构化并发。组内某个任务发生非取消异常时，会取消其他任务，退出时以`ExceptionGroup`等形式报告错误，适合“相关子任务应共同成功或共同结束”的场景。

```python
import asyncio

async def run_related_tasks():
    async with asyncio.TaskGroup() as group:
        search_task = group.create_task(search_tool())
        profile_task = group.create_task(load_profile())

    return search_task.result(), profile_task.result()
```

例如“同时预扣库存和创建临时支付会话”属于同一操作，其中一步失败后不应让另一协程在后台继续。TaskGroup 会在离开代码块前等待同组任务结束或完成取消，使子任务生命周期不会逃离父操作。它不自动完成数据库回滚，业务补偿仍需自己设计。

选型：

- 希望保留部分成功结果：`gather(return_exceptions=True)`并显式分类；
- 子任务属于同一个结构化操作，失败时应取消同伴：`TaskGroup`；
- 希望谁先完成先处理：`as_completed()`；
- 只等待一部分完成或需要自定义条件：`wait()`。

## 2.7 超时、取消与清理

Python 3.11+可使用`asyncio.timeout()`：

```python
import asyncio

async def call_with_timeout() -> str:
    try:
        async with asyncio.timeout(5):
            return await call_remote_tool()
    except TimeoutError as exc:
        raise ToolTimeoutError("remote_tool", "超过5秒") from exc
```

超时通常通过取消当前任务实现。协程被取消时会收到`CancelledError`，应使用`finally`释放资源，并在清理后继续传播取消：

```python
async def worker():
    resource = await acquire_resource()
    try:
        return await do_work(resource)
    finally:
        await resource.close()
```

不要随意吞掉`CancelledError`，否则`TaskGroup`、超时和服务优雅关闭可能无法正常工作。

需要进一步思考：客户端超时不代表远端服务一定停止。如果调用的是不可取消的外部操作，可能需要请求ID、幂等键和远端取消接口。

## 2.8 并发限制与背压

如果一次创建1000个模型请求，即使使用异步也可能导致429、连接池耗尽或显存队列爆炸。异步解决“等待时不阻塞”，不等于资源无限。

```python
import asyncio
from collections.abc import Awaitable, Callable
from typing import TypeVar

T = TypeVar("T")

class LimitedRunner:
    def __init__(self, concurrency: int):
        self._semaphore = asyncio.Semaphore(concurrency)

    async def run(self, operation: Callable[[], Awaitable[T]]) -> T:
        async with self._semaphore:
            return await operation()
```

背压表示下游处理不过来时，上游必须减速、排队或拒绝，而不是无限积压。常见措施：

- 有界`asyncio.Queue`；
- Semaphore；
- 请求超时；
- 队列长度上限；
- 429/503快速失败；
- 根据负载动态调整并发；
- 将长任务转移到任务队列。

## 2.9 包装阻塞函数

如果第三方SDK只有同步接口，可使用`asyncio.to_thread()`避免阻塞事件循环：

```python
import asyncio

async def read_legacy_sdk() -> dict:
    return await asyncio.to_thread(legacy_client.fetch)
```

注意：

- 它使用线程，不会自动解决纯Python CPU瓶颈；
- 需要确认同步SDK是否线程安全；
- 仍需设置超时和并发上限；
- 线程中的底层操作可能无法被立即取消。

## 2.10 异步HTTP Client

HTTPX提供同步和异步Client。长生命周期服务应复用Client连接池，不要在高频循环中不断新建：

```python
import httpx

class JsonClient:
    def __init__(self) -> None:
        timeout = httpx.Timeout(connect=3.0, read=20.0, write=10.0, pool=3.0)
        limits = httpx.Limits(max_connections=50, max_keepalive_connections=20)
        self._client = httpx.AsyncClient(timeout=timeout, limits=limits)

    async def get(self, url: str) -> dict:
        response = await self._client.get(url)
        response.raise_for_status()
        return response.json()

    async def close(self) -> None:
        await self._client.aclose()
```

HTTPX把超时细分为连接、读取、写入和连接池等待。只写一个模糊的总超时不一定能准确描述故障发生在哪个阶段。

## 2.11 完整案例：并发工具调用并保留部分成功结果

```python
import asyncio
from dataclasses import dataclass
from time import perf_counter
from collections.abc import Awaitable, Callable

@dataclass(slots=True)
class CallResult:
    name: str
    ok: bool
    value: object | None
    error: str | None
    latency_ms: float

async def execute_one(
    name: str,
    operation: Callable[[], Awaitable[object]],
    semaphore: asyncio.Semaphore,
    timeout_s: float,
) -> CallResult:
    start = perf_counter()
    try:
        async with semaphore:
            async with asyncio.timeout(timeout_s):
                value = await operation()
        return CallResult(name, True, value, None, (perf_counter() - start) * 1000)
    except TimeoutError:
        return CallResult(name, False, None, "timeout", (perf_counter() - start) * 1000)
    except Exception as exc:
        return CallResult(name, False, None, str(exc), (perf_counter() - start) * 1000)

async def execute_all(
    operations: dict[str, Callable[[], Awaitable[object]]],
    concurrency: int = 3,
    timeout_s: float = 5.0,
) -> list[CallResult]:
    semaphore = asyncio.Semaphore(concurrency)
    tasks = [
        execute_one(name, operation, semaphore, timeout_s)
        for name, operation in operations.items()
    ]
    return await asyncio.gather(*tasks)
```

这个案例体现了Agent工具层的基本工程要求：

- 每个工具独立计时；
- 使用Semaphore限制并发；
- 每个工具设置超时；
- 单个工具失败不会让所有成功结果丢失；
- 返回结构化结果，而不是混合普通字符串和异常对象。

生产系统还需要增加请求ID、重试分类、权限、幂等、脱敏日志和Trace。

## 2.12 第二章常见故障

### 现象：代码写了async但仍然很慢

排查：

1. 是否在协程中调用`time.sleep()`、同步HTTP或同步数据库驱动；
2. 多个协程是否被逐个`await`，实际仍然串行；
3. 连接池是否太小；
4. 下游服务是否限流；
5. 是否包含大量CPU计算。

### 现象：`gather()`抛错后其他请求仍在运行

原因：默认`gather()`传播首个异常，但不保证取消其他已提交任务。若业务要求失败即取消同组任务，优先考虑`TaskGroup`或显式取消和等待清理。

### 现象：服务关闭后仍有后台Task

原因：创建“fire-and-forget”任务但没有保存引用、没有生命周期管理，也没有在关闭时取消并等待。

修复：使用受管理的任务集合、TaskGroup或正式任务队列，在应用关闭流程中取消并等待清理。

### 现象：并发越高吞吐反而下降

可能原因：连接池等待、429重试风暴、下游排队、CPU序列化、内存压力或锁竞争。应通过分层指标定位瓶颈，不要继续盲目增加并发。

## 2.13 第二章面试表达

### 问：线程、进程和协程如何选择？

> 我先判断任务是在等待I/O还是消耗CPU。大模型API、搜索和数据库等高并发I/O优先使用asyncio，因为任务等待时可以让出事件循环；只有同步接口的I/O库可以通过线程池包装。经典CPython构建下，纯Python CPU密集任务受GIL影响，通常考虑多进程、原生库或GPU。选择后还要加入连接池、Semaphore、超时和背压，因为异步并不代表资源无限。

### 问：`gather`和`TaskGroup`有什么区别？

> `gather`适合聚合一组任务，并可通过`return_exceptions=True`保留部分成功结果；但默认传播首个异常时，其他任务不一定被取消。TaskGroup提供结构化并发，一个子任务发生非取消异常时会取消同组其他任务，并在退出时统一报告，适合生命周期相关的子任务。选型取决于业务是允许部分成功，还是要求同组共同结束。

## 2.14 第二章复习卡

- 一句话：异步提升I/O等待期间的资源利用率，并发仍需超时、取消、连接池和背压控制。
- 五个关键词：Event Loop、Task、Semaphore、Timeout、Backpressure。
- 必会代码：`create_task`、`gather`、`TaskGroup`、`asyncio.timeout`、`to_thread`。
- 易错点：忘记`await`、协程中阻塞、吞取消异常、无限并发、把线程当CPU并行万能方案。
- 项目连接：并发LLM调用、并行工具、SSE流式输出、异步数据库、优雅关闭。

---

# 第三章 数据结构与算法

## 3.1 为什么应用岗仍然需要算法基础

面试中的数据结构与算法不仅用于筛选，也直接影响Agent系统设计：

- 消息历史是有序序列；
- 工具注册表是哈希映射；
- 请求队列需要生产者—消费者；
- RAG需要Top-K和排序；
- LangGraph工作流本质上是有向图；
- 子任务依赖需要拓扑排序；
- 上下文裁剪常使用滑动窗口；
- 缓存需要LRU等淘汰策略。

学习目标不是背诵所有竞赛技巧，而是能识别问题结构、估算复杂度、写出可维护实现。

## 3.2 时间复杂度与空间复杂度

Big O描述输入规模增长时，资源消耗的增长趋势。常见数量级：

| 复杂度 | 典型操作 | 规模变大后的感受 |
|---|---|---|
| O(1) | Dict平均查找 | 基本不随n增长 |
| O(log n) | 二分查找、堆操作 | 增长缓慢 |
| O(n) | 扫描列表 | 与数据量线性增长 |
| O(n log n) | 通用排序 | 常见可接受排序复杂度 |
| O(n²) | 两两比较 | 大规模时迅速变慢 |
| O(2ⁿ) | 穷举子集 | 仅适合很小输入 |

必须同时考虑常数、I/O、序列化和网络延迟。把一个O(n²)的处理放在只有十个元素的列表上未必是系统瓶颈，而一次外部模型请求可能需要数秒。复杂度帮助判断增长趋势，性能分析仍需真实测量。

### Python常见操作

- List索引：O(1)；
- List尾部`append/pop`：摊销O(1)；
- List头部插入/删除：O(n)；
- Dict/Set平均查找：O(1)，极端情况可能退化；
- `x in list`：O(n)；
- `x in set`：平均O(1)；
- 排序：O(n log n)。

## 3.3 栈、队列和双端队列

### 栈：后进先出

适合：

- DFS；
- 撤销操作；
- 表达式解析；
- Agent执行路径回溯。

Python可用List尾部实现：

```python
stack: list[str] = []
stack.append("step-a")
current = stack.pop()
```

### 队列：先进先出

单线程算法使用`collections.deque`：

```python
from collections import deque

queue = deque(["task-a"])
queue.append("task-b")
current = queue.popleft()
```

并发代码应选择对应的线程安全或异步Queue，不要把普通Deque直接当作跨执行单元通信协议。

## 3.4 哈希表、去重和计数

Dict和Set适合快速查找：

```python
from collections import Counter

tool_calls = ["search", "calculator", "search"]
counts = Counter(tool_calls)
print(counts["search"])  # 2
```

缓存键必须稳定。模型参数Dict直接转字符串可能受顺序、非JSON类型和浮点表示影响。更可靠的方式是先进行Schema校验、字段排序和规范化序列化，再计算哈希。

## 3.5 堆与Top-K

堆允许O(log n)插入和删除最小元素，适合优先队列和Top-K：

```python
import heapq

scored_documents = [
    (0.83, "doc-a"),
    (0.91, "doc-b"),
    (0.62, "doc-c"),
]

top_two = heapq.nlargest(2, scored_documents)
```

RAG通常不是简单对所有文档完整排序，而是索引先做近似召回，再对较小候选集Rerank。理解Top-K有助于理解检索系统的阶段化设计。

## 3.6 树、图与工作流

### 树

树常见于目录、DOM、表达式和分层任务。遍历方式：

- 前序：先处理当前节点；
- 中序：二叉搜索树中可得到有序序列；
- 后序：先处理子节点再汇总；
- 层序：按层BFS。

### 图

Agent工作流可能包含条件分支、循环和多Agent协作，因此通常是图而不是树。

邻接表表示：

```python
graph = {
    "plan": ["search", "finish"],
    "search": ["review"],
    "review": ["search", "finish"],
    "finish": [],
}
```

BFS适合寻找最少步数或按层扩展，DFS适合路径探索、递归结构和回溯。访问图时必须记录`visited`，否则遇到环可能无限执行。

### 拓扑排序

有向无环图中，拓扑排序给出满足依赖关系的执行顺序。例如：

```text
下载文档 → 解析文档 → 分块 → Embedding → 建索引
```

如果依赖图存在环，则不存在有效拓扑顺序。这与任务编排中的循环依赖检测直接相关。

## 3.7 二分查找、双指针和滑动窗口

### 二分查找

适用于有序空间，每次排除一半候选。关键是明确区间是闭区间还是半开区间，并统一循环条件。

### 双指针

常用于有序数组、链表和区间问题。例如合并两组已排序结果。

### 滑动窗口

适合连续区间。上下文管理中的“只保留最近N条消息”就是简单窗口：

```python
def keep_recent(messages: list[dict], max_messages: int) -> list[dict]:
    return messages[-max_messages:]
```

真实上下文不能只按消息数量截断，还要按Token预算、消息角色和重要性处理，详见模块02。

## 3.8 递归、回溯、贪心和动态规划

- 递归：函数调用自身，必须有终止条件；
- 回溯：尝试选择，失败后撤销并继续；
- 贪心：每一步选择当前看起来最优，需证明局部最优能导向全局目标；
- 动态规划：保存重叠子问题结果，避免重复计算。

应用岗面试重点是识别状态、转移和边界，而不是机械背模板。Agent规划不能简单等同于动态规划，因为真实工具结果不确定、状态空间巨大，通常由模型规划与程序约束结合。

## 3.9 工程映射表

| 数据结构/算法 | Agent或RAG场景 | 主要风险 |
|---|---|---|
| List | 消息、轨迹 | 中间插入成本、无限增长 |
| Dict | State、工具注册表 | 键缺失、共享修改 |
| Set | 去重、循环检测 | 对象必须可哈希 |
| Queue | 请求和任务 | 堆积、背压 |
| Heap | 优先任务、Top-K | 排序方向写反 |
| Graph | 工作流、多Agent关系 | 环、不可达节点 |
| BFS/DFS | 任务和知识图探索 | 忘记visited |
| Topological Sort | 依赖调度 | 循环依赖 |
| Sliding Window | 上下文裁剪 | 截掉重要信息 |
| LRU | 缓存 | 过期、一致性、内存上限 |

## 3.10 第三章面试准备策略

建议至少熟练：

- 哈希表统计和去重；
- 栈、队列和堆；
- 二分查找；
- 双指针、滑动窗口；
- 链表基础；
- 树的DFS/BFS；
- 图的DFS/BFS和拓扑排序；
- 基础动态规划。

答题时先说明：

1. 输入、输出和边界；
2. 选择的数据结构；
3. 算法步骤；
4. 时间和空间复杂度；
5. 再写代码和测试极端情况。

## 3.11 第三章复习卡

- 一句话：先识别数据关系，再选择能满足查找、顺序、优先级和依赖要求的数据结构。
- 五个关键词：Hash、Queue、Heap、Graph、Complexity。
- 必会算法：二分、滑动窗口、DFS/BFS、Top-K、拓扑排序。
- 项目连接：消息、状态、工具注册、任务队列、RAG召回和工作流。

---

# 第四章 HTTP、API与流式通信

## 4.1 一次HTTP请求的生命周期

简化流程：

```text
客户端构造URL和请求
→ DNS将域名解析为地址
→ 建立TCP连接，HTTPS还需TLS握手
→ 发送请求行、Header和Body
→ 服务端路由、鉴权、处理业务
→ 返回状态码、Header和Body
→ 客户端解析或持续读取流
```

线上故障可能发生在任何一层。看到“模型请求失败”时，要继续区分DNS、连接、TLS、连接池、服务端状态码、读取超时和JSON解析。

## 4.2 请求组成

```http
POST /v1/chat/completions HTTP/1.1
Host: example.com
Authorization: Bearer ***
Content-Type: application/json

{"model":"example-model","messages":[...]}
```

- Method表达操作语义；
- Path定位资源或动作；
- Query适合筛选、分页等简单参数；
- Header携带媒体类型、鉴权和追踪信息；
- Body承载结构化输入。

常见Method：

- GET：读取，通常不应产生业务副作用；
- POST：创建资源或执行操作；
- PUT：整体替换，通常期望幂等；
- PATCH：部分更新；
- DELETE：删除，仍应设计幂等和权限。

## 4.3 状态码与错误契约

| 范围/状态码 | 含义 | Agent服务示例 |
|---|---|---|
| 200 | 成功 | 返回最终结果 |
| 201 | 已创建 | 创建任务或会话 |
| 202 | 已接收未完成 | 异步后台任务 |
| 400 | 请求无效 | JSON或业务参数错误 |
| 401 | 未认证 | Token无效 |
| 403 | 无权限 | 无权调用工具/访问文档 |
| 404 | 资源不存在 | 任务或文档不存在 |
| 409 | 状态冲突 | 重复写入、版本冲突 |
| 422 | 语义校验失败 | Pydantic参数校验失败 |
| 429 | 请求过多 | 限流 |
| 500 | 内部错误 | 未预期程序异常 |
| 502/503/504 | 上游或服务不可用 | 模型服务异常/超时 |

错误响应应结构化并包含可追踪ID：

```json
{
  "code": "TOOL_TIMEOUT",
  "message": "搜索工具暂时不可用",
  "request_id": "req-123"
}
```

不要把内部堆栈、数据库语句或API Key返回给用户。

## 4.4 REST与资源设计

REST强调围绕资源使用统一HTTP语义。例如：

```text
POST   /sessions                 创建会话
GET    /sessions/{id}            读取会话
POST   /sessions/{id}/messages   追加消息
GET    /tasks/{id}               查询任务状态
POST   /tasks/{id}/cancel        请求取消任务
```

不是所有操作都能自然表达为CRUD。取消、重试、审批等命令可以使用明确动作子资源，关键是接口语义稳定、权限清晰、错误可预测。

## 4.5 JSON、Schema和结构化输出

JSON只支持有限类型：对象、数组、字符串、数值、布尔和null。日期、Decimal、二进制和自定义对象需要转换。

Schema应描述：

- 字段名和类型；
- 必填字段；
- 枚举；
- 字符串长度；
- 数值范围；
- 嵌套结构；
- 是否允许额外字段。

Function Calling中的Schema主要帮助模型生成符合结构的参数，但程序端仍必须校验。模型生成“看起来像JSON”不等于语义安全，例如删除工具的`resource_id`可能格式正确但不属于当前用户。

## 4.6 鉴权与权限

需要区分：

- Authentication：你是谁；
- Authorization：你能做什么。

常见方式：API Key、Session、Bearer Token、JWT。无论哪种方式，都要：

- 使用HTTPS；
- 不在URL或日志中暴露Token；
- 设置过期和轮换；
- 在服务端重新校验资源归属；
- 对危险工具执行最小权限和审计。

## 4.7 SSE

SSE是基于HTTP的服务端到客户端单向事件流，响应类型通常为`text/event-stream`。事件以空行分隔：

```text
event: token
data: {"text":"你"}

event: token
data: {"text":"好"}

event: done
data: {}

```

适合大模型文本生成和任务进度，因为主要方向是服务端持续推送。需要考虑：

- 代理缓冲；
- 心跳；
- 客户端断开；
- 错误事件格式；
- 事件ID和断线恢复；
- 服务端任务是否随断开取消。

一条 SSE 连接仍是一个持续未结束的 HTTP 响应。服务端不是每个 token 新建一次 HTTP 请求，而是在同一响应体中不断写入事件。浏览器或客户端读到两个换行，才知道一条事件结束。

FastAPI 教学骨架如下：

```python
from fastapi.responses import StreamingResponse

async def event_stream():
    async for token in model_stream():
        yield f"event: token\ndata: {token}\n\n"
    yield "event: done\ndata: {}\n\n"

return StreamingResponse(
    event_stream(),
    media_type="text/event-stream",
)
```

若服务端已经逐块 `yield`，前端却最后一次性显示，常见原因是 Nginx/网关缓冲、压缩中间件聚合、缺少事件尾部双换行，或客户端使用了等待完整 body 的读取方法。客户端断开后，还要决定是否取消昂贵模型生成，避免无人接收却继续消耗 GPU。

## 4.8 WebSocket

WebSocket建立长连接并支持双向消息，适合实时协作、语音、需要客户端持续发送控制消息的任务。

例如实时语音助手中，客户端一边持续上传音频帧，一边接收转写、模型 token 和“停止播放”控制；单向 SSE 无法在同一流中承担持续上行音频。WebSocket 完成 HTTP Upgrade 后，双方都能主动发送消息，但需要自行定义消息类型、心跳、重连、鉴权续期和有序处理。

| 对比 | SSE | WebSocket |
|---|---|---|
| 方向 | 服务端到客户端 | 双向 |
| 基础 | 普通HTTP流 | 升级协议后长连接 |
| 浏览器支持 | EventSource简单 | WebSocket API |
| 重连 | 浏览器有基础支持 | 通常自行实现 |
| 典型场景 | 文本Token、进度 | 语音、实时控制、协作 |

如果只是模型文本流，不必为了“高级”而使用WebSocket；SSE通常更容易接入HTTP基础设施。

## 4.9 超时设计

超时不应只有一个数字。至少区分：

- Connect Timeout：建立连接；
- Pool Timeout：等待连接池空闲连接；
- Write Timeout：发送请求；
- Read Timeout：等待响应数据；
- Overall Deadline：整个业务操作的最终期限。

业务期限应向下游传播。例如用户请求最多30秒，不能给三个串行工具分别30秒而不考虑整体预算。

## 4.10 重试、指数退避和抖动

适合重试：连接短暂失败、429、部分5xx。通常不重试：参数错误、权限错误、资源不存在和确定性业务失败。

指数退避示意：

```text
第1次失败：等待约0.5秒
第2次失败：等待约1秒
第3次失败：等待约2秒
```

加入随机抖动可避免大量客户端同时重试形成“惊群”。重试必须有上限，并纳入整体超时预算。

## 4.11 幂等

幂等意味着同一个操作执行多次，最终业务效果与执行一次一致。网络请求可能已在服务端成功，但响应在返回途中丢失，客户端重试就可能重复扣费、重复写入或重复发消息。

常见办法：

- 客户端生成Idempotency Key；
- 服务端保存键与执行结果；
- 相同键直接返回既有结果；
- 数据库唯一约束；
- 业务状态机拒绝非法重复转换。

具体时间线如下：

```text
客户端发送“创建训练任务”，key=req-123
服务端已经创建 task-789，并保存 key→结果
响应返回途中网络断开
客户端不知道成功与否，用同一个 key 重试
服务端查到 req-123，直接返回 task-789，不再创建第二个任务
```

服务端不能只在执行完成后才随便写 key，否则两个相同请求并发到达时都可能先查不到再执行。常见实现是使用带唯一约束的幂等记录，原子地抢占“处理中”状态，并保存请求参数哈希；若同一 key 配了不同参数，应返回冲突，而不是错误复用旧结果。

GET 通常天然设计为幂等，但“读取接口顺便扣次数”会破坏这一性质。POST 也可以通过幂等键变得幂等。幂等保证业务效果不重复，不等于每次响应字节完全一样，也不等于可以无限期永久保存 key。

## 4.12 限流

限流保护服务和预算。常见维度：

- 用户；
- API Key；
- 模型；
- 工具；
- 租户；
- 并发数；
- QPS或Token数。

收到429后读取`Retry-After`等信息，并避免所有请求立即重试。限流不等同于并发限制：QPS控制一段时间内请求速率，并发控制同时执行数量。

## 4.13 HTTP调用示例

```python
import httpx

async def invoke_model(client: httpx.AsyncClient, payload: dict) -> dict:
    response = await client.post(
        "/v1/chat/completions",
        json=payload,
        headers={"X-Request-ID": payload["request_id"]},
    )

    if response.status_code == 429:
        raise RuntimeError("model rate limited")

    response.raise_for_status()
    return response.json()
```

生产代码还应：

- 使用Pydantic校验响应；
- 对日志脱敏；
- 区分超时和状态码；
- 按错误类型决定重试；
- 记录模型、Token、延迟和请求ID。

## 4.14 第四章故障排查

### 无法连接

```text
域名是否能解析
→ IP和端口是否正确
→ 服务是否监听
→ 防火墙/代理是否允许
→ TLS证书是否有效
→ 连接池是否耗尽
```

### SSE后端生成了内容但前端很久才显示

检查：服务端是否逐块yield；事件是否以双换行结束；反向代理是否缓冲；客户端是否逐块读取；中间压缩或网关是否合并数据。

### 重试导致重复写入

检查：接口是否幂等；是否有请求ID或幂等键；第一次请求是否已成功但响应丢失；数据库是否有唯一约束。

## 4.15 第四章面试表达

### 问：SSE和WebSocket如何选择？

> 如果主要是服务端向客户端持续发送模型Token或进度，我优先选择SSE，因为它基于HTTP、事件格式简单，也更容易接入现有鉴权和代理。需要客户端持续发送音频、控制信号或双向实时协作时选择WebSocket。无论哪种方式，都要处理断线、心跳、资源清理和客户端离开后的任务生命周期。

### 问：为什么超时后仍需要幂等？

> 客户端超时只能说明没有按时收到结果，不能证明服务端没有执行成功。如果客户端重试一个有副作用的操作，可能造成重复写入。因此需要幂等键、唯一约束或业务状态机，确保重复请求不会重复产生效果。

## 4.16 第四章复习卡

- 一句话：可靠API不仅定义请求和响应，还要定义鉴权、超时、重试、幂等、限流和错误契约。
- 五个关键词：HTTP、Schema、SSE、Timeout、Idempotency。
- 易错点：只设总超时、无脑重试、日志泄密、SSE缓冲、混淆401和403。

---

# 第五章 Linux运行与故障排查

## 5.1 文件、目录和权限

需要掌握绝对路径、相对路径、当前目录、用户目录、文件权限和所有者。常用思路：

```text
文件是否存在
→ 当前用户是否有权限
→ 路径是否来自预期工作目录
→ 配置是否使用了错误相对路径
```

不要为了快速解决权限问题直接给所有文件最高权限。服务进程应使用专用低权限用户，只访问必须的模型、日志和数据目录。

## 5.2 进程、端口和信号

服务排查先回答：

- 进程是否存在；
- 启动用户是谁；
- 监听哪个地址和端口；
- 是否只监听`127.0.0.1`；
- 是否被另一个进程占用；
- 收到终止信号后能否优雅关闭。

`127.0.0.1`通常只允许本机访问，`0.0.0.0`表示监听所有IPv4接口，但是否可从外网访问还取决于防火墙、容器映射和云安全策略。

## 5.3 日志

日志应至少包含：时间、级别、服务名、请求ID、用户/租户的安全标识、任务ID、步骤、耗时和错误类型。

不要记录：

- API Key和Token；
- 完整身份证、手机号等PII；
- 未脱敏Prompt中的敏感数据；
- 数据库密码；
- 不受控制的大段模型上下文。

排查时先用请求ID串起一次调用，而不是只搜索“ERROR”。

## 5.4 CPU、内存、磁盘和GPU

### CPU高

检查是否存在死循环、忙等待、序列化、正则灾难、纯Python计算或线程竞争。

### 内存高

检查消息和Trace是否无限积累、缓存是否无上限、大文件是否一次性加载、Task是否泄漏、模型是否重复加载。

### 磁盘满

检查日志、模型缓存、临时文件、Checkpoint和容器层。磁盘满可能进一步导致数据库、日志和模型服务异常。

### GPU问题

检查显存、GPU利用率、进程占用、驱动和CUDA兼容性。详细推理排查放在模块08。

## 5.5 环境变量和Secret

配置应区分开发、测试和生产。环境变量适合传递部署配置，但仍需：

- 启动时校验必需变量；
- 避免在日志和错误中打印完整值；
- 生产使用Secret管理；
- 定期轮换；
- 明确配置优先级。

```python
import os

api_key = os.getenv("MODEL_API_KEY")
if not api_key:
    raise RuntimeError("MODEL_API_KEY is required")
```

## 5.6 Shell基础和风险

应理解管道、重定向、退出码、变量和引号。自动化脚本中要检查命令失败，处理包含空格的路径，并避免把不可信输入直接拼接到Shell命令。

Agent如果能够调用Shell，必须增加：

- 命令白名单或能力限制；
- 沙箱；
- 工作目录边界；
- 超时；
- 输出大小限制；
- 高风险操作人工确认；
- 审计记录。

## 5.7 大模型服务通用排查顺序

```text
1. 配置和环境变量是否齐全
2. 进程是否启动
3. 端口是否监听
4. 本机健康检查是否成功
5. 网关或反向代理是否正常
6. 日志中第一条根因异常是什么
7. CPU、内存、磁盘、GPU是否耗尽
8. 下游模型、数据库和向量库是否可用
9. 最近是否发布、升级或修改配置
10. 修复后是否执行回归测试
```

不要只看最外层的“请求失败”。异常可能经过多层包装，第一条根因和完整异常链通常更有价值。

## 5.8 第五章复习卡

- 一句话：Linux排错从进程、端口、日志、配置、资源和下游依赖逐层缩小范围。
- 五个关键词：Process、Port、Permission、Log、Resource。
- 项目连接：模型服务启动、Agent后端、环境变量、日志追踪和安全执行。

---

# 第六章 Git与代码协作

## 6.1 Git的四个位置

理解Git首先要区分：

```text
工作区 → 暂存区 → 本地仓库 → 远程仓库
```

- 工作区：正在修改的文件；
- 暂存区：准备进入下一次提交的改动；
- 本地仓库：本机提交历史；
- 远程仓库：团队共享的仓库。

`git status`告诉你改动当前位于哪里；`git diff`查看未暂存差异；`git diff --staged`查看已暂存差异。

## 6.2 一次正常开发流程

```text
同步目标分支
→ 创建功能分支
→ 小步修改并测试
→ 检查差异
→ 有意义地暂存
→ 提交
→ 推送
→ 创建Pull Request
→ 处理Review
→ 合并
```

提交应该表达一个完整意图，例如“增加工具参数校验”，不要把格式化、功能修改和无关重构混在一个巨大提交中。

## 6.3 Branch、Merge与Rebase

### Merge

把两个分支历史连接起来，通常保留分叉信息。优点是历史真实直观，缺点是频繁合并可能使历史复杂。

### Rebase

把一组提交重新放到新的基点上，使历史更线性。它会改写提交身份，不应随意Rebase已经被多人共享的公共分支。

选择取决于团队规范。面试回答不能简单说“Rebase比Merge高级”，而要说明历史可读性、共享分支安全和团队策略。

## 6.4 冲突处理

冲突表示Git无法自动判断两组改动应如何组合。正确流程：

1. 阅读两边真实意图；
2. 编辑成业务上正确的最终代码；
3. 删除冲突标记；
4. 运行格式化、类型检查和测试；
5. 再继续Merge或Rebase。

不能只选择“ours”或“theirs”而不理解业务含义。

## 6.5 撤销工具的边界

- `restore`：恢复工作区或暂存区文件；
- `revert`：创建一个反向提交，适合撤销已共享提交；
- `reset`：移动分支指针，可能丢失未保存改动；
- `stash`：临时保存工作区变化；
- `reflog`：查找本地引用移动历史；
- `cherry-pick`：应用指定提交。

执行会重写历史或删除改动的操作前，先确认当前分支、工作区状态和是否存在未提交内容。团队仓库中优先使用可审计、非破坏性的方式。

## 6.6 Pull Request与Review

一个好的PR应该说明：

- 背景和问题；
- 改了什么；
- 为什么这样设计；
- 如何验证；
- 风险和未覆盖部分；
- 是否涉及配置、迁移和兼容性。

处理Review时先判断意见是否仍适用于当前代码。修复后在对应位置说明做了什么；如果不同意，使用代码、测试或文档给出可验证理由。

## 6.7 第六章复习卡

- 一句话：Git协作的目标是让改动可理解、可验证、可回退，而不是只会执行命令。
- 五个关键词：Working Tree、Stage、Commit、Branch、Review。
- 易错点：公共分支重写历史、盲目解决冲突、混合无关改动、提交前不测试。

---

# 第七章 测试、调试与代码质量

## 7.1 为什么Agent系统更需要分层测试

大模型输出存在概率性，不能把所有测试都写成“期望模型逐字输出某句话”。应把系统拆成不同确定性层：

- 参数校验：确定性测试；
- 工具选择解析：可使用固定模型响应测试；
- 工具执行：Mock外部依赖；
- 状态转移：确定性断言；
- RAG召回：固定数据集和指标；
- 端到端回答：规则、指标、Judge和人工结合。

这样可以区分“程序Bug”和“模型质量问题”。

## 7.2 测试类型

| 类型 | 关注点 | 示例 |
|---|---|---|
| 单元测试 | 单个函数/类 | 参数校验、状态更新 |
| 集成测试 | 多组件连接 | API+数据库、工具注册+执行 |
| 端到端测试 | 用户完整链路 | 提问到流式答案 |
| 回归测试 | 旧能力不退化 | 历史Badcase重新运行 |
| 冒烟测试 | 核心服务可用 | 健康检查、一次模型调用 |

## 7.3 pytest基础

```python
import pytest

@pytest.mark.parametrize(
    ("top_k", "expected"),
    [(1, 1), (5, 5), (20, 20)],
)
def test_top_k_is_preserved(top_k: int, expected: int) -> None:
    payload = SearchInput(query="agent", top_k=top_k)
    assert payload.top_k == expected


def test_top_k_rejects_zero() -> None:
    with pytest.raises(ValueError):
        SearchInput(query="agent", top_k=0)
```

一个测试应清楚表达Arrange、Act、Assert：准备输入、执行行为、验证结果。

## 7.4 Fixture

Fixture提供稳定的测试上下文，例如临时目录、数据库会话和Fake模型：

```python
import pytest

@pytest.fixture
def fake_tool_registry() -> dict:
    return {"search": lambda query: [query]}

def test_registry_contains_search(fake_tool_registry: dict) -> None:
    assert "search" in fake_tool_registry
```

Fixture可以依赖其他Fixture并在`yield`之后执行清理。作用域越大，共享越多，也越容易产生测试间状态污染；优先使用最小必要作用域。

## 7.5 Mock与依赖注入

Mock适合隔离网络、时间、随机数、数据库和付费模型API。应Mock“当前模块使用的引用”，而不是随意修改底层库的全局对象。

更可维护的方式是显式注入依赖：

```python
from typing import Protocol

class ModelClient(Protocol):
    async def complete(self, prompt: str) -> str:
        ...

async def summarize(text: str, model: ModelClient) -> str:
    return await model.complete(f"总结：{text}")
```

测试传入FakeModel即可，不需要网络：

```python
class FakeModel:
    async def complete(self, prompt: str) -> str:
        return "固定摘要"
```

Mock过多也有风险：测试可能只证明Mock与实现细节一致，而没有证明真实组件能协作。因此关键边界仍需少量集成和端到端测试。

## 7.6 异步测试

异步测试需要匹配所使用的pytest异步插件或AnyIO测试支持。示意：

```python
import pytest

@pytest.mark.asyncio
async def test_execute_all_returns_partial_failure() -> None:
    async def success() -> str:
        return "ok"

    async def failure() -> str:
        raise RuntimeError("boom")

    results = await execute_all({"a": success, "b": failure})
    assert results[0].ok is True
    assert results[1].ok is False
```

测试异步代码不能只测成功路径，还要验证：

- 超时；
- 取消；
- 部分失败；
- 并发上限；
- 资源关闭；
- 任务是否泄漏。

pytest版本和异步插件行为会变化，尤其要避免同步测试直接依赖异步Fixture等不明确组合，具体以当前官方文档和插件文档为准。

## 7.7 测试流式输出

流式测试关注事件序列而不是一次性字符串：

```python
async def collect_stream(stream) -> list[str]:
    chunks = []
    async for chunk in stream:
        chunks.append(chunk)
    return chunks
```

应验证：

- 首块能否及时出现；
- 顺序是否正确；
- 结束事件是否存在；
- 中途异常是否转换为错误事件；
- 客户端断开后是否释放资源。

## 7.8 调试方法

推荐顺序：

```text
确认实际现象
→ 构造最小复现
→ 找到错误发生的最内层位置
→ 查看输入、状态和异常链
→ 提出可证伪的原因假设
→ 一次只改变一个因素
→ 修复并增加回归测试
```

常见工具：IDE断点、结构化日志、Trace、性能剖析、内存分析和网络抓包。打印调试适合快速确认，但不应成为生产观测的唯一手段。

## 7.9 代码质量

### 函数职责

一个函数应有清晰输入、输出和失败方式。模型调用、参数校验、重试、日志和业务决策全部挤在一个巨大函数中，会很难测试和复用。

### 命名

名称表达业务含义：`validate_tool_arguments`优于`handle_data`，`max_concurrency`优于`num`。

### 类型、格式和Lint

格式化解决风格争论，Lint发现部分可疑代码，类型检查发现接口不一致。它们不能代替测试和代码评审，但能在提交前降低低级错误。

### 避免过度抽象

只出现一次、变化方向不明确的代码不必立刻建立复杂框架。抽象应该来自稳定共性，而不是为了使用设计模式。

## 7.10 第七章面试表达

### 问：模型输出不稳定，Agent系统怎么测试？

> 我会按确定性边界分层测试。参数Schema、工具执行、状态更新和路由条件用普通单元测试；外部模型和API使用Fake或Mock覆盖成功、超时、限流和错误响应；关键组件做集成测试；端到端回答使用固定评测集，通过任务成功率、工具参数正确率、规则、Judge和人工抽检综合判断。每个线上Badcase修复后加入回归集，避免只追求单次演示效果。

## 7.11 第七章复习卡

- 一句话：把概率性模型与确定性程序分层，才能得到可定位、可回归的测试体系。
- 五个关键词：Fixture、Mock、Async Test、Regression、Dependency Injection。
- 易错点：真实调用付费模型、只测成功路径、过度Mock、测试间共享状态、逐字匹配生成文本。

---

# 第八章 综合实践：异步工具执行器

## 8.1 项目目标

在进入模块02的Function Calling和Agent Loop之前，先实现一个不依赖Agent框架的工具执行层。它负责“安全、稳定地执行工具”，但不负责“由模型决定调用哪个工具”。

功能要求：

- 工具注册和查找；
- Pydantic参数校验；
- 同步/异步工具统一接入；
- 并发限制；
- 单工具超时；
- 按错误类型重试；
- 结构化成功/失败结果；
- 请求ID和耗时日志；
- 幂等接口预留；
- 单元与异步测试。

## 8.2 目录结构

```text
async_tool_executor/
├── pyproject.toml
├── src/
│   └── executor/
│       ├── models.py
│       ├── protocols.py
│       ├── registry.py
│       ├── exceptions.py
│       ├── retry.py
│       ├── executor.py
│       └── tools.py
└── tests/
    ├── test_registry.py
    ├── test_validation.py
    └── test_executor.py
```

## 8.3 数据模型

```python
from typing import Any, Literal
from pydantic import BaseModel, Field

class ToolCall(BaseModel):
    request_id: str = Field(min_length=1)
    tool_name: str = Field(min_length=1)
    arguments: dict[str, Any]
    timeout_s: float = Field(default=10.0, gt=0, le=120)

class ToolOutcome(BaseModel):
    request_id: str
    tool_name: str
    status: Literal["success", "failed", "timeout", "rejected"]
    data: Any | None = None
    error_code: str | None = None
    error_message: str | None = None
    latency_ms: float
    attempts: int = 1
```

不能只返回`str`，否则调用者难以可靠区分正常文本与错误文本。

## 8.4 工具协议与注册表

```python
from typing import Any, Protocol, TypeVar
from pydantic import BaseModel

InputT = TypeVar("InputT", bound=BaseModel)

class Tool(Protocol[InputT]):
    name: str
    input_model: type[InputT]

    async def run(self, arguments: InputT) -> Any:
        ...

class ToolRegistry:
    def __init__(self) -> None:
        self._tools: dict[str, Tool] = {}

    def register(self, tool: Tool) -> None:
        if tool.name in self._tools:
            raise ValueError(f"duplicate tool: {tool.name}")
        self._tools[tool.name] = tool

    def get(self, name: str) -> Tool:
        try:
            return self._tools[name]
        except KeyError as exc:
            raise LookupError(f"unknown tool: {name}") from exc
```

注册时拒绝重复名称，避免后注册工具静默覆盖前一个实现。

## 8.5 执行流程

```text
接收ToolCall
→ 根据名称查找工具
→ 使用工具自己的InputModel校验参数
→ 检查权限与幂等状态
→ 获取Semaphore许可
→ 设置Timeout
→ 执行工具
→ 按错误类型决定是否重试
→ 记录耗时和Trace
→ 返回ToolOutcome
```

执行层不应把所有异常都交给模型“自己理解”。需要先分类：参数和权限错误直接拒绝；网络抖动可以限次重试；未知程序错误记录堆栈并返回稳定错误码。

## 8.6 关键执行器示意

```python
import asyncio
from time import perf_counter
from pydantic import ValidationError

class ToolExecutor:
    def __init__(self, registry: ToolRegistry, max_concurrency: int = 8):
        self._registry = registry
        self._semaphore = asyncio.Semaphore(max_concurrency)

    async def execute(self, call: ToolCall) -> ToolOutcome:
        started = perf_counter()
        try:
            tool = self._registry.get(call.tool_name)
            validated = tool.input_model.model_validate(call.arguments)

            async with self._semaphore:
                async with asyncio.timeout(call.timeout_s):
                    data = await tool.run(validated)

            return ToolOutcome(
                request_id=call.request_id,
                tool_name=call.tool_name,
                status="success",
                data=data,
                latency_ms=(perf_counter() - started) * 1000,
            )
        except ValidationError as exc:
            return self._failure(call, started, "rejected", "INVALID_ARGUMENTS", str(exc))
        except LookupError as exc:
            return self._failure(call, started, "rejected", "UNKNOWN_TOOL", str(exc))
        except TimeoutError:
            return self._failure(call, started, "timeout", "TOOL_TIMEOUT", "工具执行超时")
        except Exception as exc:
            # 生产环境日志记录完整堆栈，响应中不暴露内部细节。
            return self._failure(call, started, "failed", "TOOL_FAILED", type(exc).__name__)

    @staticmethod
    def _failure(call, started, status, code, message) -> ToolOutcome:
        return ToolOutcome(
            request_id=call.request_id,
            tool_name=call.tool_name,
            status=status,
            error_code=code,
            error_message=message,
            latency_ms=(perf_counter() - started) * 1000,
        )
```

这是教学骨架，不是完整生产实现。后续应增加：权限服务、Retry Policy、结构化日志、Trace、取消传播、幂等存储和敏感信息脱敏。

## 8.7 必须完成的测试

1. 正常工具返回成功；
2. 工具不存在；
3. 参数缺失、类型错误、越界；
4. 工具超时；
5. 工具主动抛出异常；
6. 多工具并发且不超过限制；
7. 一个失败不丢失其他结果；
8. 取消任务后资源被释放；
9. 重复工具注册被拒绝；
10. 敏感参数没有进入错误响应。

## 8.8 与模块02的衔接

模块01的执行器解决：

```text
程序如何可靠执行一个已经确定的工具调用
```

模块02继续解决：

```text
模型为什么选择这个工具
→ 如何生成参数
→ 工具结果如何进入消息
→ 是否继续调用工具
→ 如何终止Agent循环
→ 如何保存和恢复状态
```

这种分层能把模型决策错误和工具执行错误分开评测、排查。

---

# 第九章 模块总结、面试与复习

## 9.1 知识主线

```text
对象和数据结构
→ 函数、类型和数据校验
→ 异常与资源生命周期
→ 线程、进程和asyncio
→ HTTP和流式通信
→ Linux运行排查
→ Git协作
→ 分层测试
→ 异步工具执行器
```

## 9.2 高频面试题

### Python

1. `is`与`==`有什么区别？
2. 可变对象和不可变对象如何影响函数参数？
3. 深拷贝和浅拷贝有什么区别？
4. 为什么默认参数不能随意使用List？
5. 生成器相比List有什么优势？
6. 装饰器如何保留原函数信息？
7. Dataclass、TypedDict和Pydantic如何选择？
8. 如何设计异常体系？

### 并发

1. 并发、并行、异步有什么区别？
2. GIL对线程有什么影响？
3. 什么时候使用多进程？
4. Coroutine、Task和Future有什么区别？
5. `gather`与`TaskGroup`如何选择？
6. 超时和取消是什么关系？
7. 如何防止一次创建过多模型请求？
8. 为什么async函数仍可能阻塞事件循环？

### HTTP与工程

1. SSE与WebSocket如何选择？
2. 429应该如何处理？
3. 哪些错误适合重试？
4. 为什么重试需要幂等？
5. 如何排查模型接口连接失败？
6. 如何测试依赖大模型的代码？
7. Merge和Rebase有什么区别？
8. Linux上服务启动失败如何排查？

## 9.3 一分钟模块总结

> Agent应用本质上是以大模型为决策组件的后端系统。Python部分我重点掌握对象和数据所有权、类型标注、Pydantic边界校验、异常链和可测试设计；并发部分会根据I/O或CPU任务选择asyncio、线程或进程，并加入连接池、Semaphore、超时、取消和背压；接口层理解HTTP、SSE、重试、幂等和鉴权；运行层能够通过进程、端口、日志、配置和资源逐层排查。项目中可以把这些能力组合成异步工具执行器，为后续Function Calling和Agent Loop提供稳定执行层。

## 9.4 掌握程度检查

### 能解释

- [ ] 解释对象引用、可变性和浅拷贝；
- [ ] 解释Pydantic的边界作用；
- [ ] 解释线程、进程、协程和GIL；
- [ ] 解释`gather`、`TaskGroup`、超时和取消；
- [ ] 解释SSE、WebSocket、重试和幂等。

### 能编码

- [ ] 写Pydantic工具参数模型；
- [ ] 写自定义异常并保留异常链；
- [ ] 并发调用多个异步工具；
- [ ] 限制并发并处理部分失败；
- [ ] 编写Fake依赖和异步测试；
- [ ] 完成异步工具执行器。

### 能排错

- [ ] 定位共享状态污染；
- [ ] 定位事件循环阻塞；
- [ ] 定位连接、读取和连接池超时；
- [ ] 定位SSE缓冲；
- [ ] 根据进程、端口、日志和资源排查服务；
- [ ] 为修复添加回归测试。

## 9.5 官方资料与延伸阅读

- [Python 3.14 asyncio协程与任务](https://docs.python.org/3/library/asyncio-task.html)
- [Python 3.14 asyncio与free-threaded Python](https://docs.python.org/3/library/asyncio-threading.html)
- [Pydantic模型文档](https://docs.pydantic.dev/latest/concepts/models/)
- [HTTPX异步支持](https://www.python-httpx.org/async/)
- [HTTPX超时配置](https://www.python-httpx.org/advanced/timeouts/)
- [pytest Fixture说明](https://docs.pytest.org/en/stable/explanation/fixtures.html)
- [pytest Monkeypatch说明](https://docs.pytest.org/en/stable/how-to/monkeypatch.html)

> 版本提示：Python并发、pytest异步Fixture和Pydantic接口会随版本演进。学习时先掌握稳定概念，再以项目实际锁定版本的官方文档核对API。
