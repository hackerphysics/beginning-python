# 第7章：Python 与 AI 工具的协作

> "The hottest new programming language is English." — Andrej Karpathy

## 7.1 AI coding 时代已经到来 🚀

Andrej Karpathy 发明了一个词：**vibe coding**。意思是你描述你想要什么，AI 帮你写出来，你改改，再让 AI 改改，最终你都不太确定代码里每一行是干什么的——但它跑起来了。

这不是批评，这是现实。

Y Combinator 2025 年报告：25% 的创业公司，95% 的代码由 AI 生成。不是原型，是生产代码。不是未来，是现在。

作为 C++ 程序员，你可能有点不适应。你习惯了对每一行代码了如指掌，习惯了在脑子里手动模拟内存布局。现在 AI 帮你生成了三百行 Python，你甚至不确定有没有内存泄漏（其实 Python 有 GC，大概率没有）。

**这一章不教你用哪个 AI 工具**。工具变化太快，今天的最新版明天就过时了。这一章教你如何成为一个优秀的「AI 协作者」：

- 写出让 AI 能读懂、能扩展的代码
- 审查 AI 生成的代码，找出坑
- 用 Python 直接调用 AI API
- 理解为什么 AI/ML 生态选择了 Python

---

## 7.2 AI 编程工具全景 🗺️

当前主流工具，了解一下生态：

| 工具 | 形态 | 特点 |
|------|------|------|
| **GitHub Copilot** | IDE 插件 | 最普及，代码补全为主 |
| **Cursor** | AI-native IDE | 基于 VS Code，深度集成 |
| **Claude Code** | 终端 CLI | 适合 agentic 任务，能读写文件 |
| **Aider** | 终端 CLI | 开源，支持多种模型 |
| **Windsurf** | IDE | Codeium 出品，强调 agent 模式 |

这些工具背后的模型（GPT-4、Claude、Gemini）能力越来越强，但工具本身是接口，底层逻辑一样：你给上下文，AI 生成代码。

!!! tip "一个务实的建议"
    不要在工具选择上花太多时间。先用最熟悉的那个，把它用好。核心能力是「如何与 AI 协作」，而不是哪个工具按钮在哪里。

---

## 7.3 写出 AI 友好的代码 ✍️

这是本章最重要的部分。

很多人觉得 AI 编程就是「我提需求，AI 写代码」。但更高效的模式是：**你写的代码本身就是给 AI 的上下文**。

当你让 AI 修改、扩展、调试你的代码时，代码的质量直接决定了 AI 的输出质量。

### 7.3.1 好的命名 = 好的 Prompt

```python
# ❌ AI 不友好：变量名是噪音
def proc(d, t):
    r = []
    for x in d:
        if x['ts'] > t:
            r.append(x)
    return r

# ✅ AI 友好：变量名就是上下文
def filter_events_after_timestamp(events: list[dict], cutoff_timestamp: float) -> list[dict]:
    return [event for event in events if event['timestamp'] > cutoff_timestamp]
```

第一个函数，当你告诉 AI「帮我给 `proc` 加一个按 category 过滤的参数」，AI 需要猜 `d` 是什么，`t` 是什么，`x['ts']` 是什么。

第二个函数，AI 直接明白：这是事件过滤，`cutoff_timestamp` 是截止时间，加 category 过滤逻辑就顺手了。

!!! note "C++ 对比"
    C++ 里你可能习惯了简短的局部变量名（`i`、`it`、`ptr`），因为作用域短、类型在声明处写清楚了。Python 里没有显式类型（至少历史上没有），变量名是唯一的语义来源。AI 时代，这个习惯值得调整。

### 7.3.2 类型注解（Type Hints）

Python 3.5+ 支持类型注解，但不强制——这是动态语言的「自愿类型系统」。

```python
# 没有类型注解
def calculate_discount(price, discount_rate, is_member):
    if is_member:
        return price * (1 - discount_rate * 1.2)
    return price * (1 - discount_rate)

# 有类型注解
def calculate_discount(
    price: float,
    discount_rate: float,
    is_member: bool
) -> float:
    if is_member:
        return price * (1 - discount_rate * 1.2)
    return price * (1 - discount_rate)
```

**为什么 AI 时代类型注解更重要了？**

1. **AI 会读类型注解推断意图**。有了 `float` 标注，AI 知道不用处理字符串边界情况。
2. **AI 生成的代码更容易验证**。你能用 `mypy` 检查 AI 给你的代码有没有类型错误。
3. **重构更安全**。你让 AI 重构一大段代码，它能通过类型推断保证接口一致。

```python
from typing import Optional, Union
from dataclasses import dataclass

@dataclass
class UserProfile:
    user_id: int
    username: str
    email: str
    age: Optional[int] = None  # 可以为 None
    score: float = 0.0

def get_user_display_name(profile: UserProfile) -> str:
    """返回用于展示的用户名。优先使用真实姓名，其次使用 username。"""
    return profile.username
```

!!! tip "mypy 静态检查"
    ```bash
    pip install mypy
    mypy your_file.py
    ```
    对 AI 生成的代码运行 mypy，能在运行前发现很多类型错误。

### 7.3.3 Docstring：给 AI 的指令手册

Docstring 不只是文档，它是你和 AI 的合约。

```python
def send_notification(
    user_id: int,
    message: str,
    channel: str = "email",
    retry_count: int = 3
) -> bool:
    """
    向指定用户发送通知。

    Args:
        user_id: 目标用户的 ID，必须是已注册用户
        message: 通知内容，最长 500 字符
        channel: 通知渠道，支持 "email"、"sms"、"push"
        retry_count: 失败重试次数，0 表示不重试

    Returns:
        True 表示发送成功，False 表示所有重试均失败

    Raises:
        ValueError: 如果 channel 不在支持列表中
        UserNotFoundError: 如果 user_id 不存在

    Example:
        >>> send_notification(42, "您的订单已发货", channel="push")
        True
    """
    ...
```

当你让 AI「给这个函数加一个超时参数」，它会：
- 知道在哪里加（`Args` 里）
- 知道应该 `Raises` 什么（参考已有的异常模式）
- 知道 `Example` 也要更新

没有 Docstring？AI 会猜，猜错了你得改半天。

### 7.3.4 小函数、清晰接口

AI 在处理小函数时准确率远高于大函数。原因很直接：上下文窗口是有限的，函数越小，AI 能完整「看到」的逻辑比例越高。

```python
# ❌ 一个做了太多事的函数
def process_order(order_data: dict) -> dict:
    # 验证数据
    if not order_data.get('items'):
        raise ValueError("订单不能为空")
    total = sum(item['price'] * item['qty'] for item in order_data['items'])
    if total > 10000:
        discount = 0.05
    elif total > 5000:
        discount = 0.03
    else:
        discount = 0
    # 计算税费
    tax_rate = 0.13 if order_data.get('is_business') else 0.09
    final_amount = total * (1 - discount) * (1 + tax_rate)
    # 写数据库
    db.orders.insert({'amount': final_amount, ...})
    # 发邮件
    send_email(order_data['user_email'], f"您的订单总金额为 {final_amount:.2f}")
    return {'order_id': 'xxx', 'amount': final_amount}

# ✅ 拆分成职责单一的小函数
def validate_order(order_data: dict) -> None:
    """验证订单数据合法性。"""
    if not order_data.get('items'):
        raise ValueError("订单不能为空")

def calculate_subtotal(items: list[dict]) -> float:
    """计算订单商品小计。"""
    return sum(item['price'] * item['qty'] for item in items)

def apply_discount(subtotal: float) -> float:
    """根据金额阶梯计算折后价格。"""
    if subtotal > 10000:
        return subtotal * 0.95
    elif subtotal > 5000:
        return subtotal * 0.97
    return subtotal

def calculate_tax(amount: float, is_business: bool) -> float:
    """计算税费。"""
    tax_rate = 0.13 if is_business else 0.09
    return amount * tax_rate
```

!!! info "给 C++ 程序员"
    你已经熟悉单一职责原则（SRP），这不是新概念。只是 AI 时代，这个原则的收益被放大了：小函数不仅对人友好，对 AI 也友好。

---

## 7.4 审查 AI 生成的 Python 代码 🔍

AI 写的代码不能直接信任。不是因为 AI 坏，而是因为 AI 不了解你的业务上下文，不知道你的运行环境，有时候还会「自信地犯错」。

### 7.4.1 AI 常犯的 Python 错误

**1. 可变默认参数陷阱**

```python
# ❌ AI 很容易生成这种代码，但它有 bug
def add_item(item: str, container: list = []) -> list:
    container.append(item)
    return container

# 问题：默认参数只初始化一次！
print(add_item("apple"))   # ['apple']
print(add_item("banana"))  # ['apple', 'banana'] ← 不是 ['banana']！

# ✅ 正确写法
def add_item(item: str, container: list | None = None) -> list:
    if container is None:
        container = []
    container.append(item)
    return container
```

**2. 浅拷贝陷阱**

```python
# ❌ AI 常用 copy() 但忘了嵌套对象
import copy

original = {'name': 'Alice', 'scores': [90, 85, 92]}
shallow = original.copy()

shallow['scores'].append(100)
print(original['scores'])  # [90, 85, 92, 100] ← 原始数据被修改了！

# ✅ 深拷贝
deep = copy.deepcopy(original)
deep['scores'].append(100)
print(original['scores'])  # [90, 85, 92] ← 安全
```

**3. 异步上下文泄漏**

```python
# ❌ AI 可能忘记关闭异步资源
import aiohttp

async def fetch_data(url: str) -> dict:
    session = aiohttp.ClientSession()  # 没有 async with！
    response = await session.get(url)
    data = await response.json()
    return data  # session 没有关闭，连接泄漏

# ✅ 正确的异步上下文管理
async def fetch_data(url: str) -> dict:
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()
```

**4. 捕获异常太宽泛**

```python
# ❌ AI 喜欢这样写，吞掉了所有错误
try:
    result = process_data(data)
except Exception:
    return None  # 连什么错都不知道

# ✅ 精确捕获，记录日志
import logging

try:
    result = process_data(data)
except ValueError as e:
    logging.error(f"数据格式错误: {e}")
    raise
except ConnectionError as e:
    logging.warning(f"连接失败，将重试: {e}")
    return None
```

### 7.4.2 AI 代码检查清单

拿到 AI 生成的代码，按这个顺序检查：

```
□ 逻辑正确性
  - 边界条件（空列表、None、0、负数）
  - 错误处理是否完整

□ Python 特有陷阱
  - 可变默认参数
  - 拷贝 vs 引用
  - 整数除法（// vs /）

□ 资源管理
  - 文件/网络连接是否用 with 语句
  - 异步资源是否用 async with

□ 性能问题
  - 循环里有没有 N+1 查询
  - 大列表有没有用生成器

□ 类型一致性（如果有类型注解）
  - mypy 检查能否通过
```

### 7.4.3 安全审查

AI 不主动考虑安全问题。你必须主动问。

**SQL 注入**

```python
# ❌ AI 生成的「简洁」代码，直接拼 SQL
def get_user(username: str) -> dict:
    query = f"SELECT * FROM users WHERE username = '{username}'"
    return db.execute(query).fetchone()
# 攻击者输入: ' OR '1'='1

# ✅ 参数化查询
def get_user(username: str) -> dict:
    query = "SELECT * FROM users WHERE username = ?"
    return db.execute(query, (username,)).fetchone()
```

**路径遍历**

```python
import os
from pathlib import Path

# ❌ 直接拼接用户输入的路径
def read_user_file(filename: str) -> str:
    path = f"/app/uploads/{filename}"
    return open(path).read()
# 攻击者输入: ../../etc/passwd

# ✅ 限制在安全目录内
def read_user_file(filename: str) -> str:
    base_dir = Path("/app/uploads").resolve()
    target = (base_dir / filename).resolve()
    
    # 确保路径在允许的目录内
    if not str(target).startswith(str(base_dir)):
        raise ValueError("不允许访问此路径")
    
    return target.read_text()
```

!!! warning "让 AI 帮你做安全审查"
    你可以直接问 AI：「检查这段代码有没有 SQL 注入、路径遍历、或者其他安全问题」。AI 做安全审查比写安全代码可靠得多——因为你给了它明确的任务目标。

---

## 7.5 用 Python 调用 AI API 🤖

最直接的 AI 协作方式：写 Python 代码调用 AI。

### 7.5.1 基本用法

```python
import os
import json
import requests

def call_openai(prompt: str, model: str = "gpt-4o-mini") -> str:
    """调用 OpenAI Chat API，返回助手回复。"""
    response = requests.post(
        "https://api.openai.com/v1/chat/completions",
        headers={
            "Authorization": f"Bearer {os.environ['OPENAI_API_KEY']}",
            "Content-Type": "application/json",
        },
        json={
            "model": model,
            "messages": [{"role": "user", "content": prompt}],
        },
    )
    response.raise_for_status()
    return response.json()["choices"][0]["message"]["content"]

# 使用
result = call_openai("用一句话解释量子纠缠")
print(result)
```

也可以用官方 SDK，更简洁：

```python
from openai import OpenAI

client = OpenAI()  # 自动读取 OPENAI_API_KEY 环境变量

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "用一句话解释量子纠缠"}],
)
print(response.choices[0].message.content)
```

### 7.5.2 结构化输出（JSON Mode）

让 AI 返回结构化数据，而不是自然语言：

```python
from openai import OpenAI
from pydantic import BaseModel

client = OpenAI()

class BookReview(BaseModel):
    title: str
    author: str
    rating: int  # 1-5
    summary: str
    pros: list[str]
    cons: list[str]

def extract_book_info(review_text: str) -> BookReview:
    """从书评文本中提取结构化信息。"""
    response = client.beta.chat.completions.parse(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": "从书评中提取结构化信息。",
            },
            {"role": "user", "content": review_text},
        ],
        response_format=BookReview,
    )
    return response.choices[0].message.parsed

# 使用
review = """
《Python 之禅》这本书写得太棒了！作者张三把复杂的概念解释得清晰易懂。
评分：4/5。优点：代码示例多，讲解深入。缺点：部分章节有点冗长。
"""
info = extract_book_info(review)
print(f"书名: {info.title}, 评分: {info.rating}/5")
```

### 7.5.3 流式响应

对于长输出，流式响应更好：

```python
from openai import OpenAI

client = OpenAI()

def stream_response(prompt: str) -> None:
    """流式打印 AI 回复，实时显示。"""
    with client.chat.completions.stream(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
    ) as stream:
        for text in stream.text_stream:
            print(text, end="", flush=True)
    print()  # 最后换行

stream_response("写一首关于 Python 和 C++ 的短诗")
```

### 7.5.4 简单的 AI Agent

Agent 的核心思想：**AI 决定下一步做什么，工具执行，循环直到完成**。

```python
import json
from openai import OpenAI

client = OpenAI()

# 定义工具
def get_weather(city: str) -> str:
    """模拟天气查询（实际应调用真实 API）"""
    weather_data = {
        "北京": "晴天，15°C",
        "上海": "多云，18°C",
        "深圳": "小雨，22°C",
    }
    return weather_data.get(city, "未知城市")

def calculate(expression: str) -> str:
    """安全地计算数学表达式"""
    try:
        # 只允许数字和基本运算符
        allowed = set("0123456789+-*/.()")
        if not all(c in allowed or c.isspace() for c in expression):
            return "不支持的表达式"
        result = eval(expression)  # 生产代码请用更安全的实现
        return str(result)
    except Exception as e:
        return f"计算错误: {e}"

# 工具定义（告诉 AI 有哪些工具可用）
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "查询城市天气",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "城市名称"}
                },
                "required": ["city"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "calculate",
            "description": "计算数学表达式",
            "parameters": {
                "type": "object",
                "properties": {
                    "expression": {"type": "string", "description": "数学表达式"}
                },
                "required": ["expression"],
            },
        },
    },
]

def run_agent(user_query: str) -> str:
    """运行一个简单的 AI Agent。"""
    messages = [{"role": "user", "content": user_query}]
    
    while True:
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=messages,
            tools=tools,
        )
        
        message = response.choices[0].message
        messages.append(message)
        
        # 如果 AI 不需要调用工具，直接返回答案
        if not message.tool_calls:
            return message.content
        
        # 执行 AI 请求的工具调用
        for tool_call in message.tool_calls:
            func_name = tool_call.function.name
            func_args = json.loads(tool_call.function.arguments)
            
            if func_name == "get_weather":
                result = get_weather(**func_args)
            elif func_name == "calculate":
                result = calculate(**func_args)
            else:
                result = f"未知工具: {func_name}"
            
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": result,
            })

# 测试
print(run_agent("北京今天天气怎么样？另外 123 * 456 等于多少？"))
```

---

## 7.6 Python 在 AI/ML 生态中的地位 🐍

C++ 程序员可能会问：为什么 AI/ML 不用 C++？我们能写出更快的代码！

答案很有意思。

### 7.6.1 为什么 AI/ML 选择了 Python

1. **交互式探索**：研究员需要快速实验，Jupyter Notebook 是完美载体。C++ 的编译-链接-运行循环太慢。

2. **生态先动优势**：NumPy（2006）、SciPy、Matplotlib 早早建立了科学计算生态，Python 就是「数学家的语言」。

3. **胶水语言**：Python 极其擅长「把各种库粘在一起」。C++ 库 → Python 绑定 → 几行 Python 调用，摩擦极低。

4. **语法简洁**：研究员写代码是为了验证想法，不是工程化，Python 的简洁性让他们专注于数学逻辑。

### 7.6.2 PyTorch 的架构：Python 前端 + C++ 后端

这个架构决策值得每个 C++ 程序员细细品味：

```
你写的 Python 代码
        ↓
    PyTorch Python API（torch.Tensor, nn.Module...）
        ↓
    Python 到 C++ 的绑定层（pybind11）
        ↓
    libtorch（纯 C++ 张量库）
        ↓
    CUDA / cuDNN / MKL
        ↓
    GPU / CPU 硬件
```

你用 Python 写的 `loss.backward()`，实际执行的是高度优化的 C++ 和 CUDA 代码。

```python
import torch

# 这几行看起来是 Python...
x = torch.randn(1000, 1000, device='cuda')
y = torch.randn(1000, 1000, device='cuda')
z = x @ y  # 矩阵乘法

# ...但实际上调用的是 cuBLAS，接近理论峰值算力
```

**工程思维：在正确的抽象层解决问题**

- 用户易用性问题 → Python 层解决
- 计算性能问题 → C++ / CUDA 层解决
- 硬件适配问题 → 驱动层解决

这不是 Python 打败了 C++，而是 Python 和 C++ 各司其职。

---

## 7.7 给 C++ 程序员的建议 💡

### 你的 C++ 背景是优势，不是包袱

很多人转 Python 会焦虑：「我 C++ 写了十年，现在感觉什么都不会了。」

不要这样想。你的 C++ 背景在 AI 时代是稀缺资产：

**理解底层**：你知道 `torch.Tensor` 背后的内存是连续的，你知道为什么批量矩阵操作比循环快，你知道 GPU 的内存带宽是瓶颈而不是算力。这些知识让你写出比普通 Python 开发者更好的 ML 代码。

**性能直觉**：Python 开发者写出 O(n²) 的代码往往不觉得有问题。你会本能地想「这里能不能向量化？」「这个 for 循环能不能用 NumPy 广播替代？」

**系统编程**：当 AI 项目需要部署、需要高并发、需要低延迟推理，你的 C++ 知识让你能接手那些 Python 搞不定的部分。

### Python + C++ 的黄金组合

```
原型阶段：Python（快速迭代，验证想法）
     ↓
发现性能瓶颈（profiling）
     ↓
热路径：用 C++ / CUDA 重写，用 pybind11 暴露给 Python
     ↓
其余部分：保持 Python（维护成本低）
```

这是 NumPy、PyTorch、OpenCV 都在用的模式。

```cpp
// compute_fast.cpp
#include <pybind11/pybind11.h>
#include <pybind11/numpy.h>
namespace py = pybind11;

// 你的高性能 C++ 函数
double sum_squares(py::array_t<double> arr) {
    auto buf = arr.request();
    double* ptr = static_cast<double*>(buf.ptr);
    double result = 0.0;
    for (size_t i = 0; i < buf.size; i++) {
        result += ptr[i] * ptr[i];
    }
    return result;
}

PYBIND11_MODULE(compute_fast, m) {
    m.def("sum_squares", &sum_squares);
}
```

```python
# 在 Python 里直接调用
import compute_fast
import numpy as np

arr = np.random.randn(1_000_000)
result = compute_fast.sum_squares(arr)  # C++ 速度，Python 接口
```

### 不要焦虑语法，焦虑思维

Python 语法一周能学会。真正的挑战是思维方式的切换：

| C++ 思维 | Python/AI 时代思维 |
|----------|--------------------|
| 我来控制内存 | 我来描述意图 |
| 编译器检查类型 | 测试验证行为 |
| 性能是第一优先级 | 先让它跑起来，再优化热路径 |
| 我写每一行代码 | 我审查和引导 AI 生成的代码 |
| 追求完美架构 | 追求能快速迭代的架构 |

最后一条特别重要：AI 时代，**会提问比会写代码更重要**。一个能精确描述需求、能正确审查输出、能快速迭代的工程师，比一个只会手写代码的工程师更有竞争力。

而你的 C++ 背景，给了你理解「代码在做什么」的底层能力。这在 AI 时代比任何时候都更值钱——因为 AI 生成代码的速度越来越快，能真正读懂代码的人越来越稀缺。

---

!!! success "小结"
    - AI 编程时代，**写好代码 = 给 AI 好上下文**：命名、类型注解、Docstring 不是可选的
    - **审查 AI 代码**：可变默认参数、浅拷贝、异步资源泄漏、安全问题是重灾区
    - **Python 调用 AI API** 并不复杂，结构化输出和 Agent 模式是核心范式
    - **Python + C++ 是黄金组合**，不是竞争关系
    - 你的 C++ 背景是优势——懂底层的人，永远不会被 AI 完全替代

---

*恭喜你完成了整个教程！现在，去用 Python + AI 构建点什么吧。🚀*
