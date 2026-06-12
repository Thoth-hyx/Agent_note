## python的上下文管理器

**上下文管理器（Context Manager）** 是 Python 中一种用于**自动管理资源**的机制。

最直观的体现就是我们经常使用的 `with` 语句。它的核心作用是：**确保在代码块执行前，先进行必要的“准备工作”；在代码块执行后，无论是否发生异常，都必定执行“清理工作”。**

### 1. 为什么需要上下文管理器？（解决什么问题）

在编程中，我们经常需要处理外部资源，比如**打开文件、建立数据库连接、获取线程锁**等。这些资源使用完毕后，**必须手动释放**，否则会造成内存泄漏或文件被一直占用。

在没有上下文管理器之前，我们通常用 `try...finally` 来确保资源释放：

**传统写法（又臭又长）：**
```python
file = open('example.txt', 'r')
try:
    # 执行业务逻辑，如果这里抛出异常，依然能执行到 finally
    data = file.read()
finally:
    # 无论成功还是失败，必须关闭文件
    file.close() 
```

**有了上下文管理器（with 语句）后：**
```python
with open('example.txt', 'r') as file:
    data = file.read()
# 缩进结束，此时 Python 会自动帮你调用 file.close()
```
可以看到，代码不仅变得更简洁优雅（Pythonic），而且彻底杜绝了程序员“忘记关闭资源”的低级错误。


### 2. 上下文管理器的工作原理（底层魔法）

一个对象之所以能放在 `with` 语句后面，是因为它的内部实现了两个“魔法方法”（Dunder methods）：

1.  **`__enter__(self)`**：在进入 `with` 语句块**之前**调用。它的返回值会被赋值给 `as` 后面的变量（如上例中的 `file`）。
2.  **`__exit__(self, exc_type, exc_val, exc_tb)`**：在离开 `with` 语句块**之后**调用。无论中间代码是正常结束还是抛出了异常，这个方法都会被执行。它主要用来写清理资源的逻辑。

### 3. 如何自定义上下文管理器？

有两种常见的方式来编写自己的上下文管理器：

#### 方式一：基于类（实现魔法方法）
这是最正统的写法，适合逻辑复杂的场景。

```python
class MyFileOpen:
    def __init__(self, filename, mode):
        self.filename = filename
        self.mode = mode
        self.file = None

    def __enter__(self):
        print(f"-> 准备工作：正在打开文件 {self.filename}")
        self.file = open(self.filename, self.mode)
        return self.file  # 这个返回值就是 as 后面的变量

    def __exit__(self, exc_type, exc_val, exc_tb):
        print(f"-> 清理工作：正在关闭文件 {self.filename}")
        if self.file:
            self.file.close()
        # 注意：如果这三个 exc_ 变量有值，说明代码块里发生了异常。
        # 如果这里返回 True，异常会被吞掉，不会向外抛出；返回 False 或 None，异常会正常抛出。

# 使用自定义的上下文管理器
with MyFileOpen('test.txt', 'w') as f:
    print("写入数据...")
    f.write("Hello World!")
    
# 执行结果：
# -> 准备工作：正在打开文件 test.txt
# 写入数据...
# -> 清理工作：正在关闭文件 test.txt
```

#### 方式二：基于生成器（使用 `contextlib` 装饰器）
这是更 Pythonic 的简写方式，不需要写一个完整的类，只需要写一个使用了 `yield` 的函数。

```python
from contextlib import contextmanager

@contextmanager
def my_timer():
    import time
    start_time = time.time()
    print("-> 计时开始")
    
    yield  # yield 前面的是 __enter__ 逻辑，yield 后面的是 __exit__ 逻辑
    
    end_time = time.time()
    print(f"-> 计时结束，耗时：{end_time - start_time} 秒")

# 使用：
with my_timer():
    print("正在执行一段耗时的代码...")
    import time
    time.sleep(2)
    
# 执行结果：
# -> 计时开始
# 正在执行一段耗时的代码...
# -> 计时结束，耗时：2.002... 秒
```


### 4. 常见的应用场景

在实际开发中，你会在这些地方大量看到上下文管理器：

1.  **文件 I/O 操作**：`with open('file.txt') as f:`
2.  **线程/进程并发锁**：确保临界区代码执行完后一定释放锁，防止死锁。
    ```python
    import threading
    lock = threading.Lock()
    with lock:
        # 安全地修改共享变量
        pass 
    ```
3.  **数据库事务处理**：如果代码块执行成功就 Commit，抛出异常就 Rollback。
4.  **网络请求与会话维持**：例如 `requests.Session()` 或前面那段代码里的 `websockets.connect()`：
    ```python
    async with websockets.connect(wss_url) as ws:
        await ws.send(...)
    ```

**总结：**
上下文管理器是一种**优雅处理资源分配与释放的语法糖**。只要是有“借用 -> 使用 -> 归还”这种生命周期的资源，都应该首选 `with` 语句和上下文管理器来处理。

---

## python函数中的*args 和 **kwargs

在 Python 的库函数（以及我们自己编写的函数）中，`*args` 和 `**kwargs` 是用来**处理可变数量参数**的特殊语法。

简单来说：它们允许你向函数传递**任意数量**的参数，而不需要在定义函数时提前写死所有参数的名字。

注意：`args` 和 `kwargs` 只是 Python 社区的**约定俗成的命名**，真正起作用的是前面的星号 `*` 和 `**`。你完全可以写成 `*a` 和 `**b`，但为了代码可读性，强烈建议保持原样。

下面为你详细拆解这两个参数的意思和用法：

### 1. `*args` (位置参数收集器)

*   **全称**：Arguments
*   **作用**：用于接收不定数量的**位置参数**（Positional Arguments），并将它们打包成一个**元组（Tuple）**。
*   **场景**：当你不知道调用者会传入多少个参数，或者参数数量本身就是动态的时候。

**代码示例：**
```python
def my_sum(*args):
    # args 此时是一个元组 (tuple)
    print("接收到的参数 args:", args)
    total = 0
    for num in args:
        total += num
    return total

# 你可以传任意个参数进去
print(my_sum(1, 2))          # args 是 (1, 2)
print(my_sum(1, 2, 3, 4, 5)) # args 是 (1, 2, 3, 4, 5)
```

### 2. `**kwargs` (关键字参数收集器)

*   **全称**：Keyword Arguments
*   **作用**：用于接收不定数量的**关键字参数**（Keyword Arguments，也就是带有名字的参数），并将它们打包成一个**字典（Dictionary）**。
*   **场景**：当你想让函数接受任意的键值对参数配置时。

**代码示例：**
```python
def print_user_info(**kwargs):
    # kwargs 此时是一个字典 (dict)
    print("接收到的参数 kwargs:", kwargs)
    for key, value in kwargs.items():
        print(f"{key} = {value}")

# 传入带有名称的参数
print_user_info(name="Alice", age=25, city="Beijing")
# kwargs 会变成：{'name': 'Alice', 'age': 25, 'city': 'Beijing'}
```

### 3. 混合使用：大一统的参数接收

在很多高级库（如 Django、Flask、Requests、Pandas）的源码中，你经常会看到这种终极形态的函数签名：

```python
def universal_func(a, b, *args, **kwargs):
    pass
```

这种写法的**规则和顺序是固定**的，必须按照：
1. 确定的普通参数（如 `a`, `b`）
2. `*args`（收集剩下的位置参数）
3. `**kwargs`（收集关键字参数）

**代码示例：**
```python
def do_something(action, *args, **kwargs):
    print(f"Action: {action}")
    print(f"Args: {args}")
    print(f"Kwargs: {kwargs}")

do_something("Download", "file1.txt", "file2.txt", timeout=30, retries=3)
# 输出：
# Action: Download
# Args: ('file1.txt', 'file2.txt')
# Kwargs: {'timeout': 30, 'retries': 3}
```

### 为什么第三方库非常喜欢用它们？

当你阅读 Python 开源库源码时，`*args` 和 `**kwargs` 无处不在，主要有以下 3 个原因：

#### ① 极大的灵活性（透传参数）
很多时候，一个函数只是个“中间商”，它需要把接收到的参数原封不动地传递给底层的另一个函数。
比如 Python 中非常著名的 `requests` 库：
```python
# 你调用 requests.get 时，除了 url，还可以传 headers, timeout, cookies 等等
def get(url, params=None, **kwargs):
    # kwargs 收集了你传入的诸如 timeout=5 之类的额外参数
    # 然后把它透传给了底层的 request 方法
    return request('get', url, params=params, **kwargs) 
```

#### ② 编写装饰器 (Decorators)
装饰器经常需要包裹各种形形色色的函数，但装饰器本身不知道原函数有多少个参数，所以必须用 `*args, **kwargs` 来全盘接收。
```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print("函数执行前...")
        result = func(*args, **kwargs) # 把接收到的参数原样传给真实函数
        print("函数执行后...")
        return result
    return wrapper
```

#### ③ 类的继承和重写 (`super()`)
在重写父类的方法时（比如 Django/DRF 的视图函数），为了防止未来父类的参数发生改变导致子类报错，或者子类不想逐个写满父类的参数，通常会这么写：
```python
class ChildClass(ParentClass):
    def __init__(self, my_param, *args, **kwargs):
        self.my_param = my_param
        # 把剩下的参数全部丢给父类的初始化函数处理
        super().__init__(*args, **kwargs)
```

### 总结
*   `*args`：把**没名字**的多余参数装进**元组**里。
*   `**kwargs`：把**有名字**的多余参数装进**字典**里。
*   它们的存在让 Python 函数变得像一个具有极高兼容性的“黑洞”，能够优雅地接收和传递未知数量的参数。
  
---

## python的装饰器和生成器

**装饰器Decorators**和**生成器Generators**是 Python 中非常强大、优雅的高级特性。

我们刚好可以把你之前问的 `*args`、`**kwargs` 以及“上下文管理器”串联起来，因为它们经常搭配使用！

### 一、 装饰器 (Decorators)

**1. 是什么？**
装饰器本质上是一个**函数**。它的作用是：**在不修改原函数代码的情况下，为原函数添加新的功能。**
Python 中使用 `@` 符号作为语法糖来调用装饰器。

**2. 通俗比喻**
你可以把“原函数”想象成一部裸机手机，“装饰器”就是手机壳。套上手机壳（装饰器）后，手机还是那个手机（原函数），照样能打电话，但额外多出了“防摔”或“防水”的新功能。你不需要拆开手机去改造它，只需要套个壳就行了。

**3. 核心原理与代码演示**
装饰器的实现依赖于闭包（在一个函数内部定义另一个函数）。这里完美用到了你之前问的 `*args` 和 `**kwargs`！

```python
import time

# 定义一个计算函数执行时间的装饰器
def timer_decorator(func):  # func 就是被套壳的原函数
    # wrapper 就是那个“壳”
    def wrapper(*args, **kwargs): 
        # 1. 增加的新功能（执行前）
        start_time = time.time()
        
        # 2. 原封不动地调用原函数（使用 *args 和 **kwargs 完美透传参数）
        result = func(*args, **kwargs) 
        
        # 3. 增加的新功能（执行后）
        end_time = time.time()
        print(f"[{func.__name__}] 执行耗时: {end_time - start_time:.4f} 秒")
        
        return result  # 把原函数的结果返回出去
    return wrapper

# 使用 @ 语法糖给普通函数套上“壳”
@timer_decorator
def do_heavy_work(name, sleep_time):
    print(f"{name} 正在辛勤工作...")
    time.sleep(sleep_time)
    return "工作完成"

# 测试
print(do_heavy_work("Alice", 1.5))
```
**输出：**
```text
Alice 正在辛勤工作...
[do_heavy_work] 执行耗时: 1.5012 秒
工作完成
```

**4. 常见应用场景**
*   **权限校验**：比如 Django 中常用的 `@login_required`，用户没登录直接拦截。
*   **日志记录**：自动记录函数每次被调用的参数和返回值。
*   **缓存**：比如 `@lru_cache`，如果同样的参数传进来，直接返回缓存结果，不再执行函数。

### 二、 生成器 (Generators)

**1. 是什么？**
生成器是一种特殊的迭代器，它最大的特点是：**它使用 `yield` 关键字而不是 `return` 来返回值。**
*   `return` 会彻底结束函数，并一次性把结果交出来。
*   `yield` 会**暂停**函数，交出一个值，并记住当前运行到了哪里，下次调用时接着往下跑。

**2. 通俗比喻**
*   **普通函数（List/Return）**：就像是**一盆水**。你要多少水，它一次性装满一盆端给你。如果这盆水超级大（比如读取100GB的文件），你的电脑内存会直接爆炸。
*   **生成器（Generator/Yield）**：就像是**自来水龙头**。它每次只滴一滴水给你（Yield）。它不占用庞大的内存（哪怕水源有100GB），你需要一滴，它就给你一滴（惰性计算/延迟加载）。

**3. 代码演示**
```python
# 这是一个生成器函数
def my_number_generator():
    print("准备生成第一步")
    yield 1
    
    print("准备生成第二步")
    yield 2
    
    print("准备生成第三步")
    yield 3

# 获取生成器对象（此时内部代码还一行都没执行！）
gen = my_number_generator()

# 使用 next() 或者 for 循环来获取值
print(next(gen))  # 打印 "准备生成第一步" 和 1，然后暂停。
print(next(gen))  # 接着上一次的位置，打印 "准备生成第二步" 和 2，然后暂停。
print(next(gen))  # ...
```

如果在实际开发中，比如你要生成斐波那契数列（无限长）：
```python
def fibonacci(n):
    a, b = 0, 1
    for _ in range(n):
        yield a
        a, b = b, a + b

# 如果 n 是 1 亿，用 return 返回一个 list，内存会爆掉。
# 用 yield 返回，内存占用几乎为 0，因为它每次只计算和返回当前那个数。
for num in fibonacci(5):
    print(num)  # 输出: 0, 1, 1, 2, 3
```

**4. 常见应用场景**
*   **读取超大文件**：每次 `yield` 文件的一行，防止内存溢出。
*   **流式响应（Streaming）**：**这正是你第一个问题那段代码中的核心！**
    你之前发的代码里有一段：
    ```python
    def event_stream(self, app, inputs, friend, message):
        # ...
        yield f'data:{json.dumps(...) }\n\n'
        # ...
        yield 'data: [DONE]\n\n'
    ```
    这段代码就是一个典型的**生成器**。它没有把 AI 所有的文字全生成完再 return，而是只要 AI 说了一句话，就立刻通过 `yield` 推送给前端（SSE），从而实现了类似 ChatGPT 那样打字机般的流式效果。

### 与上个知识点串联！

还记得你上一个问题问的**上下文管理器**吗？

Python 标准库提供了一个极度优雅的方法来快速写上下文管理器。它竟然**同时用到了装饰器和生成器**：

```python
from contextlib import contextmanager

@contextmanager   # 这是一个装饰器！
def my_timer():
    print("开始工作")
    yield         # 这是一个生成器！yield 把进入(enter)和退出(exit)的逻辑切成了两半！
    print("结束工作")

with my_timer():  # 使用上下文管理器！
    print("正在执行中...")
```

通过这三个概念的结合，你就能理解 Pythonic 代码的真正魅力：简洁、高效、富有逻辑之美。

---

## python的深拷贝和浅拷贝

在 Python 中，**深拷贝（Deep Copy）**和**浅拷贝（Shallow Copy）**是用来复制对象的两种不同方式。它们的核心区别在于：**当你复制一个包含“嵌套可变对象”（比如列表里面套列表，或者字典里面套列表）的数据时，底层的子对象是“完全独立”还是“共享引用”。**

为了让你透彻理解，我们先从最基础的“赋值”说起。

### 0. 铺垫：直接赋值 (`=`) 只是贴标签

在 Python 中，变量名只是贴在对象上的“标签”。
当你用 `=` 赋值时，**根本没有发生复制**，只是给同一个对象多贴了一个标签。

```python
a = [1, 2, [3, 4]]
b = a  # 这叫直接赋值，不叫拷贝

b[0] = 99       # 修改外层元素
b[2][0] = 88    # 修改内层（嵌套）元素

print("a:", a)  # 结果: a: [99, 2, [88, 4]]
print("b:", b)  # 结果: b: [99, 2, [88, 4]]
```
你看，修改 `b` 会让 `a` 也跟着变，因为它们实际上是**同一个东西**在内存里的两个名字。

为了真正复制出一个新对象，我们就需要用到**浅拷贝**或**深拷贝**。它们都属于 Python 内置的 `copy` 模块。

### 1. 浅拷贝 (Shallow Copy)

**1. 是什么？**
浅拷贝会创建一个新的最外层对象，但是它里面包含的子对象，**仅仅是拷贝了原来子对象的引用（内存地址）**。

**2. 通俗比喻**
假设有一份包含各种超链接的网页文档。浅拷贝就是新建了一个网页文件，把原网页排版抄了过来，但是里面的**超链接依然指向原来的那些网址**。

**3. 代码演示**
使用 `copy.copy()`、列表的切片 `[:]` 或者 `list()` 都可以实现浅拷贝。

```python
import copy

a = [1, 2, [3, 4]]
b = copy.copy(a)  # 进行了浅拷贝

# 场景一：修改最外层（互不影响）
b[0] = 99
print(a) # [1, 2, [3, 4]]
print(b) # [99, 2, [3, 4]]
# （由于 1 和 99 是不可变类型的数字，替换 b[0] 只是让它指向了新的数字 99，不影响 a）

# 场景二：修改内层的嵌套列表（会互相影响！）
b[2][0] = 88
print(a) # [1, 2, [88, 4]]  <-- a 居然也被修改了！
print(b) # [1, 2, [88, 4]]
```

**为什么会这样？**
因为浅拷贝只拷贝了外层的壳子，`a[2]` 和 `b[2]` 实际上指向的是内存里的**同一个**小列表 `[3, 4]`。牵一发而动全身。

### 2. 深拷贝 (Deep Copy)

**1. 是什么？**
深拷贝是一个**递归**的过程。它不仅复制了最外层的对象，还会一直深入下去，**把里面嵌套的所有可变对象全部复制一份，放到新的内存地址里**。彻底斩断和原对象的任何联系。

**2. 通俗比喻**
还是网页文档的例子。深拷贝不仅新建了网页，还顺着里面的每一个超链接爬过去，把目标网站的内容也都克隆了一份，放到了你自己的服务器上，然后把你新网页里的超链接替换成了你自己的网址。**从此以后，原网站怎么变，都跟你的网站毫无关系。**

**3. 代码演示**
使用 `copy.deepcopy()` 实现深拷贝。

```python
import copy

a = [1, 2, [3, 4]]
c = copy.deepcopy(a)  # 进行了深拷贝

# 无论怎么修改内层嵌套对象，都绝对互不干扰！
c[2][0] = 88

print("a:", a)  # 结果: [1, 2, [3, 4]]  <-- a 毫发无损！
print("c:", c)  # 结果: [1, 2, [88, 4]]
```
此时，`a[2]` 和 `c[2]` 在内存里已经是两个长得一模一样，但毫无瓜葛的独立列表了。

### 3. 核心总结对决

| 拷贝类型 | 语法示例 | 发生改变的对象 | 对外层不可变元素的影响（如数字、字符串） | 对内层嵌套可变元素的影响（如列表套列表） |
| :--- | :--- | :--- | :--- | :--- |
| **直接赋值** | `b = a` | 没有新对象产生，完全是同一个。 | 修改一处，另一处**跟着变**。 | 修改一处，另一处**跟着变**。 |
| **浅拷贝** | `b = copy.copy(a)`<br>`b = a[:]` | 复制了最外层的一层皮。 | 修改一处，另一处**互不影响**。 | 修改一处，另一处**跟着变**。 |
| **深拷贝** | `b = copy.deepcopy(a)` | 从外到内彻彻底底复制出全新的对象。 | 修改一处，另一处**互不影响**。 | 修改一处，另一处**互不影响**。 |

**开发中的建议：**
1. 大多数情况下，如果你只是处理简单的一维列表 `[1, 2, 3]` 或简单的字典 `{"name": "Alice"}`，**浅拷贝** `list[:]` 或 `dict.copy()` 已经完全足够了，速度快且节省内存。
2. 只有在处理诸如 JSON 数据、复杂的配置文件、多维数组等**嵌套层级深且需要独立修改**的复杂数据结构时，为了防止出 Bug，才需要用到**深拷贝 `copy.deepcopy()`**。（深拷贝因为要递归遍历，耗时和内存消耗相对较高）。

---

## python的内存管理机制

Python 的内存管理机制非常经典，它是一套**自动化**的系统，意味着开发者不需要像在 C/C++ 中那样手动分配（`malloc`）和释放（`free`）内存。

Python 的内存管理主要由**四大核心机制**组成。我们可以用大白话和比喻来逐一拆解：

### 一、 引用计数（核心主力，占据 80% 的工作量）

这是 Python 垃圾回收（Garbage Collection, GC）的**最基本、最主要**的方式。

**1. 原理**
Python 中的每一个对象，内部都有一个“计分板”（叫做引用计数器 `ob_refcnt`）。
*   当这个对象被创建、被赋值给变量、或被放进列表里时，它的计分板就 **+1**。
*   当变量被删除（`del`）、变量被赋予了新值、或者离开了函数作用域时，它的计分板就 **-1**。
*   **一旦计分板归零（变成 0），Python 就会立刻无情地把这个对象从内存中销毁，释放空间。**

**2. 代码演示**
```python
import sys

a = "hello world"  # 创建对象，计分板 = 1
b = a              # 变量 b 也指向它，计分板 = 2
c = [a]            # 放进列表里，计分板 = 3

print(sys.getrefcount(a)) # 输出 4（注意：getrefcount 本身也会临时把它当参数用一次，所以实际看到的是 3+1=4）

del b              # 计分板 = 2
c.clear()          # 计分板 = 1
a = "new string"   # 原对象的计分板变成 0，瞬间死亡！内存释放。
```

### 二、 标记-清除（专门对付“循环引用”的杀手锏）

引用计数有一个致命的弱点：**循环引用（死锁）**。

**1. 什么是循环引用？**
就像两个互相锁住的箱子，钥匙都在对方里面。
```python
a = []
b = []
a.append(b)  # a 引用 b，b 的计分板 +1
b.append(a)  # b 引用 a，a 的计分板 +1

del a
del b
```
执行 `del` 后，虽然外界再也无法访问 `a` 和 `b` 了，但由于它们互相指着对方，计分板永远都是 `1`，永远不会归零。如果程序一直在跑，这种垃圾就会越来越多，导致**内存泄漏**。

**2. 解决办法：标记-清除 (Mark-and-Sweep)**
Python 针对“容易产生循环引用”的容器对象（如列表、字典、自定义类对象等），会定期进行扫描：
*   **标记阶段**：从根对象（如全局变量、当前运行堆栈）出发，顺藤摸瓜，能被摸到的对象全部打上“有用”的标记。
*   **清除阶段**：把所有没有被打上标记的对象（比如上面互相指着的死循环 `a` 和 `b`，因为外界根本摸不到它们了），强行清除掉。

### 三、 分代回收（提升效率的加速器）

如果 Python 每时每刻都去全局扫描“标记-清除”，程序会变得卡顿无比。为了解决性能问题，Python 引入了“分代回收”。

**1. 核心思想**
基于一个统计学规律：**“如果一个对象活得越久，它就越不可能是垃圾。”**

**2. 怎么做？**
Python 将所有的容器对象分为三代（0代，1代，2代），你可以把它们想象成公司的员工：

*   **0 代（实习生）**：刚创建的对象都在 0 代。Python 会非常频繁地检查 0 代。如果发现有些对象活过了这一次检查（不是垃圾），就会把它“晋升”到 1 代。
*   **1 代（老员工）**：检查频率相对较低。如果再活过检查，就晋升到 2 代。
*   **2 代（核心骨干）**：检查频率极低。Python 认为能活到现在的对象（比如全局配置、长连接对象等）几乎不会被销毁，所以懒得经常去查它们。

这种机制极大地减少了 Python 扫描垃圾的计算量。

### 四、 内存池机制（底层优化的黑科技）

这是 Python 在向操作系统申请内存时做的底层优化（Pymalloc 机制）。

每次创建变量都去向操作系统（OS）申请内存，代价很高、速度很慢。Python 为了变快，采用了“大内存和小内存分开管理”的策略：

1.  **大对象（> 512 字节）**：比如读了一个大文件，Python 会老老实实调用 C 语言的 `malloc()` 和 `free()` 向操作系统借和还。
2.  **小对象（≤ 512 字节）**：Python 内部维护了一个巨大的**“内存池”**。如果你要创建一个小字符串或小列表，Python 不找 OS，而是直接从自己的内存池里切一块给你用。销毁时也不还给 OS，而是放回池子里留给下一个小对象用。
3.  **小整数缓存池**：
    Python 内部连整数都做了预缓存。**从 `-5` 到 `256` 这些极其常用的数字**，Python 在启动时就已经在内存里创建好了。
    ```python
    a = 100
    b = 100
    print(a is b)  # True! 因为它们在内存里是同一个现成的对象
    
    x = 1000
    y = 1000
    print(x is y)  # False! 因为 1000 超出了预设池子，分别创建了两个不同的对象
    ```
    
### 总结（一句话回顾）

Python 的内存管理可以概括为：
以**引用计数**为主，一旦归零当场超度；
辅以**标记清除**，专门解决循环引用的顽疾；
利用**分代回收**，减少扫描次数提升运行速度；
底层有**内存池机制**，缓存小对象避免频繁向系统索要内存。