# 第4章：模块——代码组织的艺术

## :robot: AI 时代的思考：你真正需要掌握什么？

AI 能在几秒内写出一个完美的排序函数、一个正则表达式解析器、甚至一整段业务逻辑。  
但有一件事 AI 经常犯错——**代码该怎么组织**。

当你让 AI 帮你写一个稍微复杂点的项目时，它经常：

- 把所有东西塞进一个文件
- 随意命名模块，边界不清
- 循环导入（circular import）导致运行报错
- 忘记 `__init__.py`，或者把不该暴露的东西全暴露出来
- 依赖管理一塌糊涂，`requirements.txt` 里写死了几十个不必要的包

:star: **代码组织能力，是 AI 时代工程师的核心竞争力之一。** AI 写函数，你设计架构。  
模块怎么划分、依赖怎么管理、包怎么发布——这章就聊这些。

---

## 模块导入

### import 的本质

在 C++ 里，`#include` 做的是**文本替换**——把头文件内容复制进来。  
Python 的 `import` 不一样，它做的是**执行并缓存**：

1. 找到目标模块文件
2. 执行那个文件（是的，真的运行一遍）
3. 把结果缓存在 `sys.modules` 里
4. 把模块对象绑定到当前命名空间

```python
import math          # 导入整个模块，用 math.sqrt() 访问
import os.path       # 导入子模块
import numpy as np   # 起别名（这是惯例，不是强制）
```

:thinking: **C++ 对比：**

| C++ | Python |
|-----|--------|
| `#include <vector>` | `import math` |
| 编译时展开 | 运行时执行 |
| 每个编译单元独立 | 全局缓存，只执行一次 |
| namespace 隔离 | module 对象隔离 |

### from...import：精准导入

```python
from math import sqrt, pi       # 只导入需要的名字
from os.path import join, exists
from typing import List, Dict, Optional  # 类型注解常用
```

用这种方式，就可以直接写 `sqrt(4)` 而不是 `math.sqrt(4)`。

:bulb: **什么时候用哪种？**

- `import math` — 标准库、常用库，保留命名空间前缀更清晰
- `from x import y` — 某个名字用得很频繁，或者明确知道不会冲突
- `import numpy as np` — 社区惯例，遵守就好

:exclamation: **别这样做：**

```python
from math import *  # 危险！不知道导入了什么，可能覆盖已有名字
```

这类似于 C++ 里 `using namespace std;` 写在头文件里——不是不行，但不推荐。

### import 的搜索路径

Python 按以下顺序搜索模块：

1. 内置模块（`sys`、`os` 这类）
2. 当前目录（或脚本所在目录）
3. `PYTHONPATH` 环境变量指定的路径
4. 标准库路径
5. site-packages（pip 安装的包在这里）

```python
import sys
print(sys.path)  # 查看完整搜索路径
```

---

## 安装模块

### pip 基础回顾

第1章已经介绍过 `pip install`，这里深入讲**依赖管理**。

```shell
pip install requests          # 安装最新版
pip install requests==2.31.0  # 安装指定版本
pip install "requests>=2.28,<3.0"  # 版本范围
pip install -U requests       # 升级到最新版
pip uninstall requests        # 卸载
pip list                      # 列出已安装的包
pip show requests             # 查看某个包的详情
```

### requirements.txt：依赖清单

:star: 这是 Python 项目的**标配**。作用就像 CMake 的 `find_package` + vcpkg 的 `vcpkg.json`——告诉别人（和 CI/CD）需要装哪些依赖。

**生成 requirements.txt：**

```shell
pip freeze > requirements.txt  # 导出当前环境所有包（含版本号）
```

**安装 requirements.txt 中的依赖：**

```shell
pip install -r requirements.txt
```

**requirements.txt 的内容长这样：**

```
requests==2.31.0
numpy==1.24.3
pandas==2.0.1
```

:thinking: **版本锁定的哲学**

`pip freeze` 会输出所有包的精确版本，包括间接依赖（依赖的依赖）。这样做的好处是**完全可复现**，坏处是文件很长、版本升级麻烦。

实践中常见两种策略：

| 策略 | 写法 | 适合场景 |
|------|------|----------|
| 精确锁定 | `requests==2.31.0` | 生产部署、团队协作 |
| 宽松范围 | `requests>=2.28` | 开源库、灵活升级 |

:bulb: **进阶工具：pip-tools**

```shell
pip install pip-tools

# 写 requirements.in（只写直接依赖，不写版本或写范围）
# 生成精确锁定的 requirements.txt
pip-compile requirements.in
```

这样直接依赖和锁定文件分开管理，更专业。

---

## 自定义模块

### 最简单的模块

Python 里，**一个 `.py` 文件就是一个模块**。没有头文件，没有声明文件，就是这么简单。

```
myproject/
    utils.py     ← 这就是一个模块
    main.py
```

```python
# utils.py
def greet(name: str) -> str:
    return f"Hello, {name}!"

PI = 3.14159
```

```python
# main.py
import utils

print(utils.greet("C++ programmer"))  # Hello, C++ programmer!
print(utils.PI)                        # 3.14159
```

:thinking: **C++ 对比：**

C++ 需要 `utils.h`（声明）+ `utils.cpp`（实现），Python 只需要 `utils.py`。  
没有 header guard，没有 forward declaration，模块天然不会被重复执行（缓存机制）。

### 包（Package）：模块的集合

当项目变大，单个文件不够用了，就需要**包**——一个包含 `__init__.py` 的目录。

```
myproject/
    main.py
    mylib/
        __init__.py      ← 有这个文件，mylib/ 就是一个包
        utils.py
        math_helpers.py
        io/
            __init__.py
            file_reader.py
            file_writer.py
```

```python
# 在 main.py 中使用
import mylib.utils
from mylib.math_helpers import calculate
from mylib.io.file_reader import read_csv
```

### `__init__.py` 的作用

`__init__.py` 可以是**空文件**（只是标记这是个包），也可以包含代码：

```python
# mylib/__init__.py

# 1. 控制 `from mylib import *` 时暴露哪些名字
__all__ = ["utils", "math_helpers"]

# 2. 简化导入路径（让用户不用知道内部结构）
from mylib.utils import greet
from mylib.math_helpers import calculate

# 3. 包级别的初始化代码
print("mylib loaded")  # 一般不这样做，举例说明可以有代码
```

有了第2种用法，用户可以直接：

```python
from mylib import greet  # 而不是 from mylib.utils import greet
```

:bulb: **设计原则：** `__init__.py` 是你包的**公共接口**。把常用的东西提升到这里，隐藏内部结构细节。这和 C++ 的公共头文件思想一致。

### `__all__`：显式声明公共 API

```python
# utils.py
__all__ = ["greet", "PI"]  # 只有这两个是"公开的"

def greet(name):
    return f"Hello, {name}!"

PI = 3.14159

def _internal_helper():  # 下划线开头，约定为"私有"
    pass
```

:thinking: 这类似 C++ 的 `public:` / `private:`，但 Python 是**约定**而非强制。下划线开头的名字不会被 `from x import *` 导入，但你仍然可以手动导入——Python 信任程序员。

### 以模块方式运行代码

这是 Python 里非常常见的模式：

```python
# utils.py

def greet(name):
    return f"Hello, {name}!"

if __name__ == "__main__":
    # 直接运行这个文件时，__name__ 是 "__main__"
    # 被其他模块 import 时，__name__ 是 "utils"
    print(greet("World"))
    print("This only runs when executed directly")
```

```shell
python utils.py        # 会执行 if __name__ == "__main__" 里的代码
python -m utils        # 以模块方式运行（更规范，路径处理不同）
```

:star: **实际用途：**

1. **测试/演示代码** — 可以直接跑文件看效果
2. **命令行入口** — 既是库，也是工具
3. **防止副作用** — 确保 import 时不会意外执行代码

```python
# 好的实践
def main():
    # 主逻辑

if __name__ == "__main__":
    main()
```

---

## 模块引用

### 绝对导入 vs 相对导入

```
mylib/
    __init__.py
    utils.py
    io/
        __init__.py
        file_reader.py    ← 我们在这里
        file_writer.py
```

```python
# file_reader.py 里导入 utils.py

# 绝对导入（推荐）
from mylib import utils
from mylib.utils import greet

# 相对导入（.表示当前包，..表示上级包）
from .. import utils          # 上级包的 utils
from ..utils import greet     # 上级包的 utils 里的 greet
from . import file_writer     # 同级的 file_writer
```

:bulb: **相对导入的场景：** 包内部互相引用时，相对导入更灵活——重命名包时不需要改所有内部引用。但绝对导入更易读，优先用绝对导入。

### 循环导入：最常见的坑

:exclamation: **循环导入**是 Python 新手（和 AI）最常遇到的问题：

```python
# a.py
from b import func_b  # a 导入 b

def func_a():
    return "a"
```

```python
# b.py
from a import func_a  # b 导入 a → 循环！

def func_b():
    return func_a()
```

运行时会报 `ImportError: cannot import name 'func_a' from partially initialized module 'a'`。

**解决方法：**

1. **重构代码** — 最根本，把共同依赖提取到第三个模块
2. **延迟导入** — 在函数内部 import，而不是模块顶层
3. **只导入模块，不导入名字** — `import a` 而不是 `from a import func_a`

```python
# 方法2：延迟导入
def func_b():
    from a import func_a  # 调用时才导入
    return func_a()
```

:thinking: **C++ 类比：** 循环 `#include` 有 header guard 保护，但循环依赖的设计问题是一样的——说明模块边界划分有问题。

---

## 扩展Python

### Python 扩展的几种方式

有时候 Python 不够快，或者需要调用已有的 C/C++ 代码。Python 提供了多种扩展方式：

**1. ctypes — 调用动态库（最简单）**

```python
import ctypes

# 加载动态库
lib = ctypes.CDLL("./mylib.so")  # Linux/Mac
# lib = ctypes.CDLL("./mylib.dll")  # Windows

# 调用函数
lib.add.argtypes = [ctypes.c_int, ctypes.c_int]
lib.add.restype = ctypes.c_int
result = lib.add(3, 4)
print(result)  # 7
```

**2. cffi — 更现代的 C 扩展接口**

```python
from cffi import FFI
ffi = FFI()

ffi.cdef("int add(int a, int b);")
lib = ffi.dlopen("./mylib.so")
print(lib.add(3, 4))
```

**3. Cython — Python 转 C 的编译器**

```python
# mymodule.pyx (Cython 文件)
def fast_sum(list numbers):
    cdef double total = 0
    for n in numbers:
        total += n
    return total
```

编译后性能接近纯 C，语法是 Python 超集。

**4. pybind11 — C++ 扩展的现代方案**

```cpp
// mymodule.cpp
#include <pybind11/pybind11.h>
namespace py = pybind11;

int add(int a, int b) { return a + b; }

PYBIND11_MODULE(mymodule, m) {
    m.def("add", &add, "A function that adds two numbers");
}
```

```python
# Python 端
import mymodule
print(mymodule.add(3, 4))
```

:star: **pybind11 是 C++ 程序员最熟悉的方式**，可以直接暴露 C++ 类、函数、枚举到 Python，性能无损。

---

## 虚拟环境

### 为什么需要虚拟环境？

想象一下：

- 项目 A 需要 `numpy 1.20`
- 项目 B 需要 `numpy 1.24`
- 你的系统只能安装一个版本

:exclamation: 这就是虚拟环境要解决的问题。每个项目一个隔离的 Python 环境，互不干扰。

:thinking: **C++ 类比：**

| C++ | Python |
|-----|--------|
| vcpkg / Conan | pip + venv |
| CMake find_package | import |
| build 目录隔离 | venv 目录隔离 |
| 每个项目独立编译依赖 | 每个项目独立安装依赖 |

### venv 基础用法

```shell
# 创建虚拟环境
python -m venv .venv          # .venv 是约定的目录名

# 激活虚拟环境
source .venv/bin/activate     # Linux/Mac
.venv\Scripts\activate        # Windows

# 此后所有 pip install 都只影响这个环境
pip install requests numpy

# 退出虚拟环境
deactivate
```

激活后，命令行提示符前会多一个 `(.venv)` 标记：

```
(.venv) $ python --version
(.venv) $ pip list
```

### 项目标准工作流

```shell
# 新项目开始
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 开发过程中安装新包
pip install new-package
pip freeze > requirements.txt  # 更新依赖清单

# 提交代码（不要提交 .venv 目录！）
echo ".venv/" >> .gitignore
git add requirements.txt
git commit -m "Update dependencies"
```

:bulb: **进阶：pyenv + virtualenv**

- `pyenv` — 管理多个 Python 版本（类似 nvm for Node.js）
- `virtualenv` — 比内置 venv 功能更多

```shell
pyenv install 3.11.0
pyenv local 3.11.0  # 当前目录用 3.11.0
```

**更现代的选择：`uv`**

`uv` 是用 Rust 写的 Python 包管理器，速度快 10-100 倍：

```shell
# 安装 uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# 创建虚拟环境并安装依赖
uv venv
uv pip install -r requirements.txt

# 甚至可以直接管理 Python 版本
uv python install 3.12
```

---

## 打包发布

### 现代 Python 打包：pyproject.toml

:star: Python 打包方式经历了多次演变。现代做法是用 `pyproject.toml`（PEP 517/518 标准），取代老式的 `setup.py`。

**项目结构：**

```
mypackage/
    pyproject.toml       ← 项目元数据和构建配置
    README.md
    src/
        mypackage/
            __init__.py
            utils.py
            core.py
    tests/
        test_utils.py
```

**pyproject.toml 示例：**

```toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.backends.legacy:build"

[project]
name = "mypackage"
version = "0.1.0"
description = "A sample Python package"
readme = "README.md"
license = {text = "MIT"}
requires-python = ">=3.9"
dependencies = [
    "requests>=2.28",
    "numpy>=1.24",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "black",
    "mypy",
]

[project.scripts]
mypackage-cli = "mypackage.cli:main"  # 命令行工具入口点
```

### 构建和安装

```shell
# 安装构建工具
pip install build

# 构建 wheel 和 sdist
python -m build

# 生成：
# dist/mypackage-0.1.0-py3-none-any.whl  ← 二进制发行包
# dist/mypackage-0.1.0.tar.gz            ← 源码发行包

# 本地安装（开发模式，修改立即生效）
pip install -e .
```

:thinking: **wheel 类比 C++ 的预编译库**——用户安装时不需要编译。对于纯 Python 包，wheel 就是个打包好的 zip。对于含 C 扩展的包，wheel 包含了编译后的 `.so`/`.pyd` 文件。

### 发布到 PyPI

```shell
# 安装发布工具
pip install twine

# 发布到测试 PyPI（先在这里试试）
twine upload --repository testpypi dist/*

# 发布到正式 PyPI（需要先注册账号）
twine upload dist/*

# 别人就可以安装你的包了
pip install mypackage
```

:book: PyPI 地址：[https://pypi.org](https://pypi.org)

**发布前检查清单：**

- [ ] `README.md` 写清楚了
- [ ] 版本号更新了
- [ ] 在 test PyPI 测试过安装
- [ ] LICENSE 文件存在
- [ ] 敏感信息（密钥、密码）没有打包进去

### 打包成可执行文件

有时候需要给不懂 Python 的用户分发工具，可以打包成单个可执行文件：

**PyInstaller（最流行）：**

```shell
pip install pyinstaller

# 打包成单个文件
pyinstaller --onefile main.py

# 生成：dist/main（Linux/Mac）或 dist/main.exe（Windows）
```

**zipapp（Python 自带，轻量）：**

```shell
# 把整个包打包成 .pyz 文件（需要目标机器有 Python）
python -m zipapp mypackage -m "mypackage.main:main" -o myapp.pyz

# 运行
python myapp.pyz
```

:thinking: **C++ 类比：** PyInstaller 类似静态链接——把 Python 解释器和所有依赖都打包进去，文件大但完全独立。zipapp 类似动态链接——需要目标机器有 Python 运行时。

---

## 代码组织的哲学

最后，回到开头的问题：**模块该怎么划分？**

:star: **几个实用原则：**

**1. 单一职责**  
每个模块只做一件事。`utils.py` 里什么都放是反模式。

```
# 差的设计
utils.py  ← 数据库、网络、文件操作全在里面

# 好的设计
db.py        ← 数据库相关
http_client.py  ← 网络请求
file_io.py   ← 文件操作
```

**2. 按变化频率划分**  
经常变化的和稳定的分开。核心逻辑和 UI 逻辑分开。

**3. 公共接口最小化**  
`__init__.py` 里只暴露用户需要的，内部实现细节不要泄露出去。

**4. 避免深层嵌套**  
`mylib.utils.helpers.string.format.advanced` 这种路径说明结构有问题。

:thinking: **思考题：**

1. 你有一个项目，包含数据库操作、HTTP API、命令行界面三个部分。你会怎么划分模块结构？
2. 如果两个模块都需要一个辅助函数，该放在哪里？
3. `__init__.py` 里应该有多少代码？什么情况下应该保持为空文件？

---

## 小结

:book: **本章要点回顾：**

| 概念 | Python 方式 | C++ 类比 |
|------|-------------|----------|
| 模块 | `.py` 文件 | `.h` + `.cpp` |
| 包 | 含 `__init__.py` 的目录 | namespace + 目录 |
| 导入 | `import x` / `from x import y` | `#include` |
| 依赖管理 | pip + requirements.txt | vcpkg / Conan |
| 环境隔离 | venv | 独立 build 目录 |
| 发布 | wheel + PyPI | 库文件 + 包管理器 |

:star: **最重要的三件事：**

1. **虚拟环境是标配**——每个项目都用，别装全局
2. **`requirements.txt` 要提交**——团队协作的基础
3. **模块边界要想清楚**——这是 AI 替代不了你的地方

下一章我们聊 Python 的面向对象——你会发现它和 C++ 差异很大，但又有一些熟悉的感觉。
