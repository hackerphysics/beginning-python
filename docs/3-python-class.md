# 类和面向对象

!!! tip "AI 时代的思考"
    在 AI 能秒写出任何类定义的今天，理解面向对象的核心不在于记住 `class` 的语法，而在于知道**什么时候该用类、什么时候不该用**。这一章的目标不是让你背会 Python 的 OOP 语法——AI 比你背得好——而是让你理解 OOP 背后的设计思想，以及 Python 和 C++ 在这件事上截然不同的哲学。

    Python 的 OOP 更像是一种"约定"，而非强制约束。理解这些约定背后的原理，才是你真正需要掌握的东西。

## 定义类

在Python中定义一个类：

```python
class Bird:
    pass # 当你什么也不想做的时候，可以使用pass
```

在Python3中，所有类默认继承自object类，所以虽然我们这个类看起来什么都没有，但它也包含了丰富的内容，我们可以用 dir 这个方法来查看一个类或者一个函数的内部细节，打印 dir(Bird) 的结果，你会看到如下内容：

```shell
['__class__',
 '__delattr__',
 '__dict__',
 '__dir__',
 '__doc__',
 '__eq__',
 '__format__',
 '__ge__',
 '__getattribute__',
 '__gt__',
 '__hash__',
 '__init__',
 '__init_subclass__',
 '__le__',
 '__lt__',
 '__module__',
 '__ne__',
 '__new__',
 '__reduce__',
 '__reduce_ex__',
 '__repr__',
 '__setattr__',
 '__sizeof__',
 '__str__',
 '__subclasshook__',
 '__weakref__']
```

可以看出，这些默认提供的方法名前后都有两个下划线，这些方法在Python内被称为**魔法方法**，用来实现一些特殊的功能，会在一些特殊的时机被系统调用。

## 给类增加成员变量

现在我们给这个鸟类增加一点东西：

```python
class Bird:
    color = "red"
    age = 1
    childrens = []
```

Python中并没有常量的概念，所以一切成员都是变量。这意味着，我们可以随意对他们进行修改。

```python
b = Bird()
b.color = "green"
b.color # green
```

## 给类增加方法

```python
class Bird:
    color = "red"
    age = 1
    childrens = []
    def eat(self): # self 可以类比于 c++ 中的this
        print("eat")
    def grow(self):
        self.age += 1 # 注意，在类方法中引用类成员必须用 self.
```

在类中定义的方法（method）的第一个参数必须为self，这反应了类中方法和普通函数的区别，这么约定的原因后面会讲到，接下来我们就可以调用这些方法了：

```python
b = Bird()
b.eat()
b.grow()
print(b.age)
# eat
# 2
```

## 构造方法

在Python里面，构造方法的名字统一为__init__， 这是个魔法方法，在上面魔法方法列表中你能找到它，这说明定义类的时候默认就会从object父类继承一个__init__方法，只不过这个方法基本什么都不会做。如果我们想自己定义一个构造方法，那么直接覆盖它就好，也就是重载。

```python
class Bird:
    color = "red"
    age = 1
    childrens = []
    def __init__(self): 
        self.age = 0 # 进行一些初始化的操作
        self.color = ""
    def eat(self): # self 可以类比于 c++ 中的this
        print("eat")
    def grow(self):
        self.age += 1 # 注意，在类方法中引用类成员必须用 self.
```

现在，构造函数帮助我们做了一些初始化的工作，我们可以试一下效果：

```python
b = Bird()
print(b.age)
# 0
```

可以看出，初始化会覆盖初始定义的变量值。所以看起来最开始的定义没有什么用，对不对？不如把它去掉呢？

```python
class Bird:
    color = "red"
    childrens = []
    def __init__(self): 
        self.age = 0 # 进行一些初始化的操作
        self.color = ""
    def eat(self): # self 可以类比于 c++ 中的this
        print("eat")
    def grow(self):
        self.age += 1 # 注意，在类方法中引用类成员必须用 self.
        
b = Bird()
print(b.age)
# 0
```

果然可以。**在C++中，我们使用一个变量的时候总是需要预先定义，但是在Python中并不需要这样，你只需要在用的时候直接用就可以，使用的同时也完成了定义。**

## 多个构造方法？

在C++中，我们通常会给一个类定义多个构造方法，来满足不同场景下的需求，不同的构造方法拥有不同的参数类型或参数个数（这是一种多态的技术）。在Python里面可以这样吗？我们试一下，看看会发生什么：

```python
class Bird:
    color = "red"
    childrens = []
    def __init__(self): 
        self.age = 0 # 进行一些初始化的操作
        self.color = ""
    def __init__(self, age): # 另外一个构造方法
        self.age = age 
        self.color = ""
    def eat(self): # self 可以类比于 c++ 中的this
        print("eat")
    def grow(self):
        self.age += 1 # 注意，在类方法中引用类成员必须用 self.
        
b = Bird()
print(b.age)
```

这一次，我们失败了：

```shell
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
~\AppData\Local\Temp/ipykernel_47416/476329091.py in <module>
     13         self.age += 1 # 注意，在类方法中引用类成员必须用 self.
     14 
---> 15 b = Bird()
     16 print(b.age)

TypeError: __init__() missing 1 required positional argument: 'age'
```

仔细看错误信息（**这很重要，每次出错的时候记得都要先这样做**），Python在告诉我们少输入了age这个参数，看起来两个构造函数Python只接受第二个：

```python
b = Bird(3)
print(b.age)
# 3
```
这样调用之后果然成功了，你会发现，虽然我们写了两个构造函数，但是Python只实现了第二个，其实第一个是被第二个偷偷覆盖了，并且Python不会给我们报任何错误，除非你去调用那个并不存在的构造函数。

> Python语言的特性就是这样，只有当你执行到具体的代码时，才会暴露出错误。有的时候开发很迅速，测试也很顺畅，但依然有一些问题被隐藏起来了，只有在运行的时候才会暴露。Python作为一门脚本语言，动态解释执行，固然有足够多的优点，但是相比于编译语言还是有一些缺点的，使用的时候需要注意。

那么，我们如何实现多个构造函数呢？ 仔细想想，我们需要的并不一定是多个构造函数，我们需要的是在不同场景下完成不同的构造。因为Python可接受的参数非常灵活，所以很多需求完全可以在一个函数内实现。

比如，我们可以用默认参数：

```python
class Bird:
    color = "red"
    childrens = []
    def __init__(self, age = 0): 
        self.age = age # 进行一些初始化的操作
        self.color = ""
    def eat(self): # self 可以类比于 c++ 中的this
        print("eat")
    def grow(self):
        self.age += 1 # 注意，在类方法中引用类成员必须用 self.
```


还有更灵活的方式，

```python
class Bird:
    color = "red"
    childrens = []
    def __init__(self,  *args, **kwargs): 
        if "age" in kwargs:
            self.age = kwargs["age"] # 进行一些初始化的操作
        else:
            self.age = 0
        self.color = ""
    def eat(self): # self 可以类比于 c++ 中的this
        print("eat")
    def grow(self):
        self.age += 1 # 注意，在类方法中引用类成员必须用 self.
b = Bird(age = 3)
print(b.age)
# 3
```

所以在Python中有很多灵活的方式来实现这些需求。

## 静态成员变量

C++中有静态成员变量，可以在不对类进行实例化的情况下调用，Python中有没有呢？我们重新看一下这个类的定义，

```python
class Bird:
    color = "red"
    age = 1
    childrens = []
```
我们试一下直接访问这些变量：

```python
Bird.color
# red
Bird.age
# 1
```

所以，在类中直接定义的成员变量默认都与可以通过类名访问，等价于C++中的静态变量。

```python
class Bird:
    color = "red"
    childrens = []
    def __init__(self): 
        self.age = 0 # 进行一些初始化的操作
        self.color = "green"
    def eat(self): # self 可以类比于 c++ 中的this
        print("eat")
    def grow(self):
        self.age += 1 # 注意，在类方法中引用类成员必须用 self.
        
b = Bird()
print(b.color)
print(Bird.color)
print(b.age)
print(Bird.age)
```

```shell
green
red
0
---------------------------------------------------------------------------
AttributeError                            Traceback (most recent call last)
~\AppData\Local\Temp/ipykernel_47416/519595942.py in <module>
     14 print(Bird.color)
     15 print(b.age)
---> 16 print(Bird.age)

AttributeError: type object 'Bird' has no attribute 'age'

```

注意观察，这里我们看到了两个有意思的现象：
- 通过类名访问color和通过对象访问color，得到的值不一样
- 在类方法中定义的变量age，通过类名是无法访问的

## 类对象和实例对象

Python的代码在编译时，无论是函数，还是类，都生成了相应的对象，无论这个类是否实例化，都生成了类对象。这也是为什么在不创建新对象的前提下你可以直接访问类里面的内容。那么这个类对象和手动创建的新对象之间有什么区别呢？

```python
class Bird:
    pass
b = Bird()
# type
print(type(Bird))
print(type(b))

# <class 'type'>
# <class '__main__.Bird'>

```
我们看到他们的类型（type）是不一样的，在没有实例化之前，类对象的类型统一为 "type"。我们再直接打印他们，看看他们会输出什么：

```python
class Bird:
    pass
b = Bird()

print((Bird))
print((b))

# <class '__main__.Bird'>
# <__main__.Bird object at 0x000001DEAD20D6A0>

```
> 打印对象实际上调用了对象的魔法方法：\_\_str\_\_(), str(obj)函数也会调用这一魔法方法，如果你希望将object转成自定义的字符串，你可以重新定义这一方法。

打印类名得到一个class的描述（和type(b)的输出一样！，想想为什么？）， 打印创建的对象带有额外的地址信息，这个地址就是对象在内存中的地址。

总之，类对象和实例化的对象之间还是有区别的。 类对象就是一个静态对象，可以类比于C++中的静态类。所以，在Python中你不需要创建静态成员和静态方法，类中定义的所有变量和方法都是静态的，听起来是不是有点不可思议？我们自己验证一下是否确实如此：

```python
class Bird:
    color = "red"
    childrens = []
    def __init__(self): 
        self.age = 0 # 进行一些初始化的操作
        self.color = "green"
    def eat(self): # self 可以类比于 c++ 中的this
        print("eat")
    def grow(self):
        self.age += 1 # 注意，在类方法中引用类成员必须用 self.

print(Bird.color)
# red
print(Bird.childrens)
# []
print(Bird.age)
# AttributeError: type object 'Bird' has no attribute 'age' ! 注意，思考一下为什么不能访问 age？
Bird.eat()
# TypeError: eat() missing 1 required positional argument: 'self'  ？ 为什么会失败
```

不是说好了类对象中的所有方法都是静态方法吗，为什么调用eat方法会失败呢？ 仔细阅读错误提示！Python告诉我们，我们少输入了一个参数。那如果我们随便传入一个参数呢？

```python
Bird.eat(1)
# eat
```

居然调用成功了！

实际上，因为eat方法中并没有使用self这个参数，所以你传入任何东西都可以，这个方法还能正常工作。 那么请思考一下，为什么在实例化后的对象上面调用eat方法就不需要传入参数呢？ 其实很简单，是Python偷偷帮助我们给方法传入了这一个参数。请记住，这正是方法和函数的重要区别之一！**对象在调用方法的时候会偷偷传入一个参数放到第一个位置。**

那么这个self参数究竟是什么呢？ 我们在上面提到过，可以把它类比于C++的this。为了探究这一点，我们不妨再做点实验：

```python
Bird.grow(1) # 如法炮制，尝试调用grow方法
# AttributeError: 'int' object has no attribute 'age'
```

这一次我们失败了，Python告诉我们 int 对象没有age属性。因为我们在grow函数中使用了self这个参数，并且调用了self的属性 self.age。我们传入了一个整数1，于是报了上面的错误。而当你调用 b.grow() 的时候，情况会有所不同，首先，在创建对象b的时候会调用构造方法__init__， 在那里会创建age这个属性，然后Python会把对象本身作为参数传入到grow，self自然也就具有了这个属性。调用self.age就好比c++中调用this->age。

到这里还没结束…… 我们再想想，Python给我们的错误提示只是说： 'int' object has no attribute 'age'， 可没有说必须要传入一个Bird的对象进来啊？ 看一下下面的代码：

```python
class VirtualBird:
    age = 100

class Bird:
    color = "red"
    childrens = []
    def __init__(self): 
        self.age = 0 # 进行一些初始化的操作
        self.color = "green"
    def eat(self): # self 可以类比于 c++ 中的this
        print("eat")
    def grow(self):
        self.age += 1 # 注意，在类方法中引用类成员必须用 self.
        print(self.age)

Bird.grow(VirtualBird)
# 101
```

我们用一个"假鸟"成功欺骗了Python！看起来，self这个参数仅仅是一个参数，Python约定会在调用方法的时候把对象本身传进来，当然你也可以传进来别的东西，只要保证方法本身能正常工作！**Python里面有很多的约定，你可以遵守，也可以不遵守**，甚至self这个参数命也是约定! 不信你看：

```python
class VirtualBird:
    age = 100

class Bird:
    color = "red"
    childrens = []
    def __init__(self): 
        self.age = 0 # 进行一些初始化的操作
        self.color = "green"
    def eat(self): # self 可以类比于 c++ 中的this
        print("eat")
    def grow(you):
        you.age += 1 # 注意，在类方法中引用类成员必须用 self.
        print(you.age)

Bird.grow(VirtualBird)
# 101
```

Python真是一门自由的语言，如果和C++相比，在这里你仿佛完全不受束缚了。不过要小心，自由也是有代价的，他会让有些错误更加隐蔽，写代码的时候可不要大意了。

于是，类对象下面的方法都是静态方法（严格来说应该叫函数，下面会有说明），即使是带self参数的也是如此。
Python类对象
1. Python是脚本语言，Pythonn的代码在编译时，无论是函数，还是类，都生成了相应的对象，无论这个类是否实例化，都生成了类对象
2. Python类对象是一个静态对象,一旦生成，就不再变化
3. 类对象生成时，init构造函数没有运行,所以调用类方法时，init构造方法的实例属性不存在，不可以调用init方法中self.xxx实例属性
4. 类属性是类对象的静态变量
5. 类方法/函数是类对象的静态方法

当一个类被实例化后，会调用init方法，然后在内存中生成一个新的对象，拥有一份属于自己的属性，和类对象之间相互独立。

```python
class Bird:
    color = "red"
    age = 1
    childrens = []
    def __init__(self): 
        self.age = 0 # 进行一些初始化的操作
        self.color = "green"
    def eat(self): # self 可以类比于 c++ 中的this
        print("eat")
    def grow(self):
        self.age += 1 # 注意，在类方法中引用类成员必须用 self.
        
b = Bird()
print(b.color)
print(Bird.color)

# green
# red

```

可以看出，改变实例对象的属性值并不会影响类对象的属性值。

## 静态方法

回顾一下我们上面调用类对象静态方法的方式：在调用eat方法的时候必须传入一个dummy的参数，这看起来非常不方便，如果我们确定这是一个类的静态，方法，是否可以不要self这个参数呢？

```python
class Bird:
    def eat():
        print("eat")

Bird.eat()
# eat
```
确实可以，这样看起来自然多了。不过这带来了新的问题：

```python
class Bird:
    def fly(self):
        print("bird fly")
    def eat():
        print("eat")

b= Bird()
b.fly()
b.eat()
# bird fly
# TypeError: eat() takes 0 positional arguments but 1 was given

```

如果这样定义，实例对象就无法调用这一方法了，因为实例对象在调用的时候总是偷偷传入了一个参数。在C++里面，静态方法不管用类名还是新建的对象都可以正常调用，在Python里面就不能有类似的方法吗？当然有,而且实现起来很简单:

```python
class Bird:
    def fly(self):
        print("bird fly")
    @staticmethod
    def eat():
        print("eat")

Bird.eat()
b= Bird()
b.fly()
b.eat()
# eat
# bird fly
# eat
```
可以看到,现在不管怎么调用鸟儿都可正常吃了.

## 装饰器/注解

这个 `@staticmethod` 看起来有点神秘，它究竟是如何工作的呢？

在 C++ 中，你可能用过属性（`[[nodiscard]]`、`[[deprecated]]`）或者模板元编程来给函数附加"元信息"。Python 的装饰器（Decorator）做的事情类似，但更加直接：**装饰器本质上是一个函数，它接收一个函数作为参数，返回一个新的函数**。

### 装饰器的本质

先看一个最简单的例子：

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print("调用前")
        result = func(*args, **kwargs)
        print("调用后")
        return result
    return wrapper

# 使用装饰器语法
@my_decorator
def say_hello():
    print("Hello!")

say_hello()
# 调用前
# Hello!
# 调用后
```

`@my_decorator` 这个语法只是语法糖，等价于：

```python
say_hello = my_decorator(say_hello)
```

所以 `@staticmethod` 也不神秘，`staticmethod` 就是一个内置的"装饰器类"，它把普通函数包装成一个描述符对象，使得通过实例或类访问时都不会自动传入 `self`。

### 写一个有用的装饰器

来写一个记录函数执行时间的装饰器，这在 C++ 里需要手动插桩，在 Python 里可以非常优雅：

```python
import time
import functools

def timer(func):
    @functools.wraps(func)  # 保留原函数的元信息（名字、文档等）
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} 耗时: {elapsed:.4f}s")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(0.1)
    return 42

slow_function()
# slow_function 耗时: 0.1003s
```

!!! note ":bulb: `@functools.wraps` 是什么？"
    没有它，`slow_function.__name__` 会变成 `"wrapper"`，调试起来很困惑。`@functools.wraps(func)` 把原函数的元数据复制过来，这本身也是一个装饰器——装饰装饰器的装饰器 😄

### 带参数的装饰器

装饰器还可以带参数，只需要再包一层函数：

```python
def repeat(n):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(n):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(3)
def greet(name):
    print(f"Hello, {name}!")

greet("Python")
# Hello, Python!
# Hello, Python!
# Hello, Python!
```

### :thinking: Python 装饰器 vs C++ 的类似机制

| Python 装饰器 | C++ 对应机制 |
|---|---|
| `@staticmethod` | `static` 关键字 |
| `@property` | getter/setter 方法 |
| `@functools.lru_cache` | 手动实现 memoization 或第三方库 |
| 自定义计时器 | RAII 计时对象 + 宏包裹 |
| AOP（面向切面编程） | 需要大量模板元编程 |

C++ 的 `[[deprecated]]`、`[[nodiscard]]` 等属性是编译期注解，只影响编译器行为；Python 的装饰器是运行期包装，直接改变函数行为。两者哲学不同，Python 更灵活但有运行时开销。

!!! tip "AI 时代思考 🤖"
    装饰器是 Python 中少数几个 AI **写得出但你必须理解**的特性之一。当你让 AI 写一个"带缓存的函数"时，它会给你 `@lru_cache`，但你需要知道：
    
    - 缓存是基于参数的哈希，参数必须是可哈希的
    - 缓存不会自动过期，内存可能无限增长
    - 在类方法上使用 `@lru_cache` 有 `self` 的坑
    
    理解装饰器的原理，才能判断 AI 给的方案是否适合你的场景。

---

## 方法和函数的区别

前面我们反复提到"方法偷偷传入 self"，现在来彻底搞清楚这件事。

### 函数和绑定方法

在 Python 中，定义在类里的函数，根据访问方式的不同，会有不同的行为：

```python
class Bird:
    def fly(self):
        print(f"Bird at {id(self)} is flying")

b = Bird()

# 通过类访问：得到的是普通函数
print(type(Bird.fly))   # <class 'function'>

# 通过实例访问：得到的是绑定方法（bound method）
print(type(b.fly))      # <class 'method'>

# 绑定方法已经把 self 绑定进去了
print(b.fly.__self__)   # <__main__.Bird object at 0x...>
print(b.fly.__func__)   # <function Bird.fly at 0x...>
```

**绑定方法（bound method）**就是函数 + 实例的组合体。调用 `b.fly()` 时，Python 自动把 `b` 作为第一个参数传给 `Bird.fly`，这就是为什么方法签名里有 `self`。

### 验证一下

```python
class Bird:
    def fly(self):
        print(f"Flying! self is: {self}")

b = Bird()

# 这两种调用完全等价
b.fly()               # 绑定方法调用
Bird.fly(b)           # 非绑定函数调用，手动传 self
```

### :thinking: 和 C++ 的对比

C++ 的成员函数在编译时就被绑定了，`this` 指针由编译器隐式处理，你无法把成员函数当普通函数传递（除非用函数指针或 `std::bind`）。

Python 的绑定方法是运行期对象，可以到处传递：

```python
# 可以把方法存起来，像回调一样传递
fly_func = b.fly    # 保存绑定方法
fly_func()          # 调用时不需要再传 self
# Flying! self is: <__main__.Bird object at 0x...>
```

这在 C++ 里需要 `std::function` + `std::bind` 才能做到：

```cpp
// C++
auto fly_func = std::bind(&Bird::fly, b);
fly_func();
```

Python 内置支持，更自然。

---

## 魔法方法：`__get__`

现在来解释"方法如何从函数变成绑定方法"这个魔法背后的机制：**描述符协议**。

### 什么是描述符

如果一个对象定义了 `__get__`、`__set__`、`__delete__` 中的任意一个，它就是一个**描述符**。Python 的属性访问机制会在特定时机自动调用这些方法。

函数对象就是描述符！它实现了 `__get__`：

```python
def fly(self):
    print("flying")

# 函数对象有 __get__ 方法
print(hasattr(fly, '__get__'))  # True

# 模拟类的属性访问过程
class Bird:
    pass

# 直接访问描述符的 __get__：
bound_method = fly.__get__(Bird(), Bird)
print(type(bound_method))  # <class 'method'>
bound_method()             # 等同于 b.fly()（不过 self 是随机的 Bird 实例）
```

当你访问 `b.fly` 时，Python 内部大概做了这件事：

```python
# Python 内部近似实现
def __getattribute__(self, name):
    attr = type(self).__dict__[name]  # 从类里找到函数
    if hasattr(attr, '__get__'):
        return attr.__get__(self, type(self))  # 调用 __get__，绑定 self
    return attr
```

### 用描述符实现 `@property`

`@property` 就是一个内置描述符，让你用属性语法调用方法：

```python
class Circle:
    def __init__(self, radius):
        self._radius = radius
    
    @property
    def radius(self):
        return self._radius
    
    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("半径不能为负数")
        self._radius = value
    
    @property
    def area(self):
        import math
        return math.pi * self._radius ** 2

c = Circle(5)
print(c.radius)  # 10  — 像属性一样访问，实际调用了方法
print(c.area)    # 78.539...
c.radius = 10    # 触发 setter，有验证逻辑
c.radius = -1    # ValueError: 半径不能为负数
```

### :thinking: 和 C++ 的对比

C++ 里的 getter/setter 需要显式定义方法并调用：

```cpp
// C++
circle.getRadius();
circle.setRadius(10);
```

Python 的 `@property` 让你保持"像访问属性"的语法，同时后端有方法逻辑。

### 自定义描述符

你也可以写自己的描述符，实现类型检查：

```python
class PositiveFloat:
    """只允许正数的描述符"""
    def __set_name__(self, owner, name):
        self.name = name
    
    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return obj.__dict__.get(self.name, 0)
    
    def __set__(self, obj, value):
        if not isinstance(value, (int, float)) or value <= 0:
            raise ValueError(f"{self.name} 必须是正数，got {value}")
        obj.__dict__[self.name] = float(value)

class Rectangle:
    width = PositiveFloat()
    height = PositiveFloat()
    
    def __init__(self, width, height):
        self.width = width
        self.height = height
    
    @property
    def area(self):
        return self.width * self.height

r = Rectangle(3, 4)
print(r.area)   # 12.0
r.width = -1    # ValueError: width 必须是正数，got -1
```

这在 C++ 里需要大量模板代码才能实现，Python 的描述符协议让这变得非常简洁。

---

## 类方法 `@classmethod`

前面我们学了 `@staticmethod`，现在来看它的兄弟：`@classmethod`。

### 区别一眼看清

```python
class Bird:
    species = "Aves"  # 类变量
    
    def __init__(self, name):
        self.name = name
    
    @staticmethod
    def info():
        # 静态方法：不接收任何隐式参数
        # 不能访问 cls 或 self
        print("我是一只鸟")
    
    @classmethod
    def create(cls, name):
        # 类方法：第一个参数是类本身（cls）
        # 可以访问类变量，可以创建实例
        print(f"创建 {cls.species} 类的实例")
        return cls(name)

# 两种方法都可以通过类名或实例调用
Bird.info()           # 我是一只鸟
Bird.create("鹦鹉")   # 创建 Aves 类的实例

b = Bird("麻雀")
b.info()              # 我是一只鸟
b.create("鸽子")      # 创建 Aves 类的实例
```

### 工厂方法模式 :star:

`@classmethod` 最常见的用途是实现**工厂方法**——提供多种创建对象的方式（记得上面说 Python 没有构造函数重载吗？这就是优雅的解决方案）：

```python
class Date:
    def __init__(self, year, month, day):
        self.year = year
        self.month = month
        self.day = day
    
    @classmethod
    def from_string(cls, date_string):
        """从字符串 '2024-01-15' 创建日期"""
        year, month, day = map(int, date_string.split('-'))
        return cls(year, month, day)
    
    @classmethod
    def today(cls):
        """创建今天的日期"""
        import datetime
        now = datetime.date.today()
        return cls(now.year, now.month, now.day)
    
    def __repr__(self):
        return f"Date({self.year}, {self.month}, {self.day})"

# 三种创建方式
d1 = Date(2024, 1, 15)
d2 = Date.from_string("2024-01-15")
d3 = Date.today()

print(d1)  # Date(2024, 1, 15)
print(d2)  # Date(2024, 1, 15)
```

### 继承时的神奇之处

`@classmethod` 的 `cls` 参数在子类中会自动变成子类本身，这在 `@staticmethod` 中是做不到的：

```python
class Animal:
    name = "动物"
    
    @classmethod
    def introduce(cls):
        return f"我是 {cls.name}"
    
    @staticmethod
    def static_intro():
        return "我是动物（静态）"

class Dog(Animal):
    name = "狗"

print(Animal.introduce())  # 我是 动物
print(Dog.introduce())     # 我是 狗  ← cls 变成了 Dog！
print(Dog.static_intro())  # 我是动物（静态）  ← 静态方法不感知子类
```

### :thinking: 和 C++ 的对比

C++ 的 `static` 方法相当于 Python 的 `@staticmethod`，没有对应 `@classmethod` 的概念（因为 C++ 在编译期就确定了类型，不需要运行期的 `cls`）。Python 的 `@classmethod` + 继承组合非常强大，是多态工厂方法的利器。

---

## 类继承

Python 支持继承，语法比 C++ 简洁很多：

```python
# C++
// class Parrot : public Bird { ... };

# Python
class Parrot(Bird):
    pass
```

### 单继承

```python
class Animal:
    def __init__(self, name):
        self.name = name
    
    def speak(self):
        return f"{self.name} 发出声音"
    
    def __repr__(self):
        return f"{type(self).__name__}(name={self.name!r})"

class Dog(Animal):
    def speak(self):  # 重写父类方法
        return f"{self.name} 说：汪汪！"
    
    def fetch(self):  # 子类独有方法
        return f"{self.name} 去捡球了"

class Cat(Animal):
    def speak(self):
        return f"{self.name} 说：喵～"

dog = Dog("旺财")
cat = Cat("咪咪")

print(dog.speak())   # 旺财 说：汪汪！
print(cat.speak())   # 咪咪 说：喵～
print(dog.fetch())   # 旺财 去捡球了

# isinstance 检查继承关系
print(isinstance(dog, Dog))     # True
print(isinstance(dog, Animal))  # True  ← 继承链上的都是 True
```

### 多继承

Python 支持多继承，C++ 也支持，但 Python 用 MRO（Method Resolution Order，方法解析顺序）来解决菱形继承问题：

```python
class Flyable:
    def move(self):
        return "飞行中"
    
    def ability(self):
        return "会飞"

class Swimmable:
    def move(self):
        return "游泳中"
    
    def ability(self):
        return "会游泳"

class Duck(Flyable, Swimmable):  # 多继承，顺序很重要！
    def __init__(self, name):
        self.name = name

duck = Duck("唐老鸭")
print(duck.move())     # 飞行中  ← 用了 Flyable 的，因为它在前面
print(duck.ability())  # 会飞
```

### MRO：方法解析顺序

Python 用 **C3 线性化算法**计算 MRO，你可以直接查看：

```python
print(Duck.__mro__)
# (<class 'Duck'>, <class 'Flyable'>, <class 'Swimmable'>, <class 'object'>)

# 或者
print(Duck.mro())
```

MRO 决定了在多继承中，当多个父类有同名方法时，Python 选哪个。规则是：**深度优先，但按列表顺序，且保证每个类只出现一次**。

菱形继承（最经典的多继承问题）：

```python
class A:
    def hello(self):
        return "A"

class B(A):
    def hello(self):
        return "B"

class C(A):
    def hello(self):
        return "C"

class D(B, C):  # 菱形继承
    pass

print(D().hello())  # B  ← MRO: D -> B -> C -> A
print(D.__mro__)
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
```

### :thinking: Python 多继承 vs C++ 多继承

| 特性 | Python | C++ |
|---|---|---|
| 多继承支持 | ✅ 内置 | ✅ 支持，但复杂 |
| 菱形继承 | MRO 自动解决 | 需要虚继承 `virtual` |
| 方法查找顺序 | C3 线性化，确定性强 | 依赖虚继承和编译器 |
| 接口/抽象类 | `abc.ABC` + `@abstractmethod` | 纯虚函数 `= 0` |

C++ 的菱形继承需要这样处理：

```cpp
// C++
class A { ... };
class B : virtual public A { ... };  // 虚继承
class C : virtual public A { ... };
class D : public B, public C { ... };
```

Python 不需要这些，MRO 帮你搞定了。

!!! tip "AI 时代思考 🤖"
    多继承是一个 AI 特别容易写出但你需要仔细审查的场景。AI 可以给你生成合法的多继承代码，但它不一定知道你的 MRO 顺序是否符合业务逻辑。在 code review 时，看到多继承就要亲手打印一下 `__mro__`，确认方法解析顺序是你期望的。

---

## 调用超类方法

在子类中重写了父类方法后，如果还想调用父类的实现，用 `super()`：

```python
class Animal:
    def __init__(self, name, age):
        self.name = name
        self.age = age
    
    def describe(self):
        return f"{self.name}，{self.age}岁"

class Dog(Animal):
    def __init__(self, name, age, breed):
        super().__init__(name, age)  # 调用父类构造方法
        self.breed = breed
    
    def describe(self):
        base = super().describe()  # 调用父类的 describe
        return f"{base}，{self.breed}犬"

dog = Dog("旺财", 3, "柴")
print(dog.describe())  # 旺财，3岁，柴犬
```

### `super()` 和 MRO 的关系

`super()` 不是简单地"调用父类"，而是**按照 MRO 顺序调用下一个类**。这在多继承中至关重要：

```python
class A:
    def greet(self):
        print("A greet")

class B(A):
    def greet(self):
        print("B greet")
        super().greet()  # 按 MRO 调用下一个

class C(A):
    def greet(self):
        print("C greet")
        super().greet()

class D(B, C):
    def greet(self):
        print("D greet")
        super().greet()

D().greet()
# D greet
# B greet
# C greet   ← 注意：B 的 super() 调用的是 C，而不是 A！
# A greet
```

MRO 是 `D -> B -> C -> A`，所以每个 `super()` 都沿着 MRO 往后走一步。

### :thinking: 和 C++ 的对比

C++ 需要显式指定父类名：

```cpp
// C++
class Dog : public Animal {
public:
    Dog(string name, int age, string breed)
        : Animal(name, age), breed_(breed) {}  // 显式调用父类构造
    
    string describe() override {
        return Animal::describe() + "，" + breed_ + "犬";
    }
};
```

Python 的 `super()` 不需要写父类名，好处是改变继承关系时不用修改 `super()` 调用；C++ 的写法更明确，但改继承关系时需要改所有地方。

---

## Override：方法重写

在 Python 中，重写父类方法只需要在子类中定义同名方法，不需要任何关键字：

```python
class Shape:
    def area(self):
        return 0
    
    def describe(self):
        return f"面积: {self.area()}"

class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius
    
    def area(self):  # 重写，不需要任何关键字
        import math
        return math.pi * self.radius ** 2

class Rectangle(Shape):
    def __init__(self, w, h):
        self.w = w
        self.h = h
    
    def area(self):
        return self.w * self.h

shapes = [Circle(5), Rectangle(3, 4)]
for s in shapes:
    print(s.describe())
# 面积: 78.539...
# 面积: 12
```

Python 的多态是**鸭子类型**：只要对象有对应的方法，就可以调用，不需要继承关系。

### :thinking: Python vs C++ 的多态

| 特性 | Python | C++ |
|---|---|---|
| 重写语法 | 直接定义同名方法 | `override` 关键字（推荐）|
| 虚函数 | 所有方法默认都是"虚的" | 必须显式声明 `virtual` |
| 多态机制 | 鸭子类型，运行期动态 | 虚表（vtable），编译期确定 |
| 性能 | 有运行时开销 | 虚函数调用稍有开销，但比 Python 快 |
| 纯虚函数 | `@abstractmethod` | `= 0` |

在 C++ 里，如果父类方法没有声明 `virtual`，子类重写后通过父类指针调用会调用父类版本（静态分派）。Python 不存在这个问题，所有方法调用都是动态分派。

```cpp
// C++：没有 virtual 的陷阱
class Animal {
public:
    string speak() { return "..."; }           // 非虚
    virtual string name() { return "Animal"; } // 虚
};

class Dog : public Animal {
public:
    string speak() { return "Woof"; }  // 隐藏，不是重写
    string name() override { return "Dog"; }
};

Animal* a = new Dog();
cout << a->speak();  // "..."  ← 调用了 Animal::speak，因为非虚
cout << a->name();   // "Dog"  ← 正确，因为虚函数
```

Python 完全不用担心这个：

```python
# Python：所有方法都是"虚的"
class Animal:
    def speak(self):
        return "..."

class Dog(Animal):
    def speak(self):
        return "Woof"

a = Dog()
print(a.speak())  # "Woof"  ← 永远调用实际类型的方法
```

### 抽象方法

如果你想强制子类必须重写某个方法（类似 C++ 的纯虚函数），用 `abc` 模块：

```python
from abc import ABC, abstractmethod

class Shape(ABC):  # 继承 ABC
    @abstractmethod
    def area(self) -> float:
        """子类必须实现这个方法"""
        pass
    
    def describe(self):
        return f"面积: {self.area()}"

# Shape()  ← TypeError: 不能实例化抽象类

class Circle(Shape):
    def __init__(self, r):
        self.r = r
    
    def area(self):
        import math
        return math.pi * self.r ** 2

c = Circle(5)
print(c.describe())  # 面积: 78.539...
```

---

## 异常

Python 的异常处理和 C++ 的 try/catch 很像，但更简洁、更灵活。

### 基本语法

```python
try:
    x = 1 / 0
except ZeroDivisionError:
    print("除以零了！")

# 除以零了！
```

对比 C++：

```cpp
// C++
try {
    int x = 1 / 0;
} catch (const std::exception& e) {
    std::cout << "异常: " << e.what() << std::endl;
}
```

Python 更简洁，而且异常类型更细分。

### 捕获多种异常

```python
def safe_divide(a, b):
    try:
        result = a / b
    except ZeroDivisionError:
        print("不能除以零")
        return None
    except TypeError as e:
        print(f"类型错误: {e}")
        return None
    else:
        # 没有异常时执行
        print(f"结果: {result}")
        return result
    finally:
        # 无论如何都会执行（类似 C++ 的 RAII 析构）
        print("计算完毕")

safe_divide(10, 2)
# 结果: 5.0
# 计算完毕

safe_divide(10, 0)
# 不能除以零
# 计算完毕
```

### `else` 和 `finally`

这是 Python 独有的：

- `else`：只有 `try` 块**没有**抛出异常时才执行
- `finally`：**无论如何**都会执行，类似 C++ 中 RAII 保证资源释放

```python
def read_file(path):
    f = None
    try:
        f = open(path, 'r')
        content = f.read()
    except FileNotFoundError:
        print(f"文件不存在: {path}")
        return None
    else:
        print(f"成功读取 {len(content)} 个字符")
        return content
    finally:
        if f:
            f.close()  # 保证文件一定被关闭
            print("文件已关闭")
```

!!! note ":bulb: 更 Pythonic 的写法"
    实际上，Python 推荐用 `with` 语句管理资源，更简洁：
    ```python
    with open(path, 'r') as f:
        content = f.read()
    # 离开 with 块后文件自动关闭，相当于 C++ 的 RAII
    ```

### 自定义异常

自定义异常就是继承 `Exception`（或其子类）：

```python
class BirdError(Exception):
    """鸟类相关错误的基类"""
    pass

class TooYoungToFlyError(BirdError):
    """太小了不会飞"""
    def __init__(self, name, age, min_age):
        self.name = name
        self.age = age
        self.min_age = min_age
        super().__init__(
            f"{name} 还不会飞（当前 {age} 岁，至少需要 {min_age} 岁）"
        )

class Bird:
    MIN_FLY_AGE = 2
    
    def __init__(self, name, age):
        self.name = name
        self.age = age
    
    def fly(self):
        if self.age < self.MIN_FLY_AGE:
            raise TooYoungToFlyError(self.name, self.age, self.MIN_FLY_AGE)
        print(f"{self.name} 在飞！")

baby_bird = Bird("小麻雀", 1)

try:
    baby_bird.fly()
except TooYoungToFlyError as e:
    print(f"错误: {e}")
except BirdError as e:
    print(f"鸟类错误: {e}")
# 错误: 小麻雀 还不会飞（当前 1 岁，至少需要 2 岁）
```

### 异常链

Python 支持异常链，可以在捕获一个异常后抛出另一个，保留原始错误信息：

```python
try:
    int("不是数字")
except ValueError as e:
    raise RuntimeError("数据处理失败") from e

# Traceback 会显示两个异常：原始的 ValueError 和新的 RuntimeError
```

### :thinking: Python vs C++ 异常处理对比

| 特性 | Python | C++ |
|---|---|---|
| 基本语法 | `try/except/else/finally` | `try/catch/` (无 else) |
| 捕获所有 | `except Exception` | `catch (...)` |
| 异常类型 | 类的继承体系 | 任意类型都可以 throw |
| 资源管理 | `with` 语句（上下文管理器）| RAII（析构函数） |
| 性能影响 | 异常路径有一定开销 | 零开销异常（ZEH）选项 |
| 检查型异常 | 不存在（类似 C++ 的 noexcept 反面）| `noexcept` 声明 |

C++ 可以 throw 任意类型（包括 `int`），Python 只能 throw 继承自 `BaseException` 的对象。

!!! tip "AI 时代思考 🤖"
    AI 生成的代码经常在异常处理上偷懒——要么 `except Exception: pass`（吞掉所有错误），要么完全不处理。
    
    你需要判断的是：
    
    - 这个错误**应该被捕获并恢复**，还是**应该让它传播**？
    - `finally` 块是否会有副作用（比如关闭一个从未打开的连接）？
    - 自定义异常是否携带了足够的上下文信息方便调试？
    
    异常设计是软件架构层面的判断，AI 做不了——它需要你来决定。
