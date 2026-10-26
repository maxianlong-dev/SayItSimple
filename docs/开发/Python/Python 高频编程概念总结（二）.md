# Python 高频编程概念总结（二）
> 大白话 + 专业定义 + 代码示例，方便快速复习。
## 目录
1. [闭包 Closure](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#1-闭包-closure)
2. [异步 Async](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#2-异步-async)
3. [装饰器 Decorator](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#3-装饰器-decorator)
4. [高阶函数 Higher-order Function](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#4-高阶函数-higher-order-function)
5. [生成器 Generator](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#5-生成器-generator)
6. [迭代器 Iterator](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#6-迭代器-iterator)
7. [GIL 全局解释器锁](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#7-gil-全局解释器锁)
8. [多线程 Threading](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#8-多线程-threading)
9. [多进程 Multiprocessing](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#9-多进程-multiprocessing)
10. [协程 Coroutine](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#10-协程-coroutine)
11. [魔法方法 Magic Method](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#11-魔法方法-magic-method)
12. [@property](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#12-property)
13. [继承 vs 组合](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#13-继承-vs-组合)
14. [元类 Metaclass](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#14-元类-metaclass)
15. [可变 vs 不可变对象](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#15-可变-vs-不可变对象)
16. [深拷贝 vs 浅拷贝](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#16-深拷贝-vs-浅拷贝)
17. [上下文管理器 Context Manager](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#17-上下文管理器-context-manager)
18. [`args`](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#18-args-和-kwargs)[ 和 ](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#18-args-和-kwargs)[`*kwargs`](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#18-args-和-kwargs)
19. [列表推导式 / 生成器表达式](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#19-列表推导式--生成器表达式)
20. [异常处理 Exception Handling](https://app.notion.com/p/ai-3e4f447951a480068a8ad73041acf9be?p=3e5f447951a4800bb0f9f1de5452eea8&t=new&showMoveTo=true#20-异常处理-exception-handling)
---
## 1. 闭包 Closure
**大白话**：一个函数"记住"了它出生时所在的环境，即使离开了那个环境，还能访问里面的变量——就像搬出父母家后，仍能打开随身带走的"传家宝"保险箱。
**定义**：内部函数引用了外部函数的变量，且被返回/传递出去后仍能访问那些外部变量。
```python
def make_counter():
    count = 0

    def counter():
        nonlocal count
        count += 1
        return count

    return counter

my_counter = make_counter()
print(my_counter())  # 1
print(my_counter())  # 2
```
每次调用 `make_counter()` 生成的计数器互相独立，各自"记住"自己的 `count`。
**常见用途**：装饰器、需要"私有状态 + 记忆"的轻量场景。
---
## 2. 异步 Async
**大白话**：点奶茶时不站在柜台死等，而是拿号去做别的事，好了再被叫——等待期间不浪费时间。
**为什么需要**：网络请求、文件读写、数据库查询等"等待型"操作，同步方式会让程序在等待期间什么都干不了。
```python
import asyncio

async def async_task(name, seconds):
    print(f"{name} 开始点奶茶")
    await asyncio.sleep(seconds)   # 让出控制权，不是死等
    print(f"{name} 奶茶做好了")

async def async_main():
    await asyncio.gather(
        async_task("小明", 2),
        async_task("小红", 2),
    )
    # 总耗时约2秒，而不是4秒

asyncio.run(async_main())
```
**关键字**：`async def` 声明协程；`await` 表示"这里要等，先让出去"；`asyncio.gather` 并发跑多个任务。
---
## 3. 装饰器 Decorator
**大白话**：给函数"穿衣服"——不改动函数本身代码，外面包一层加新功能。
```python
def log_time(func):
    def wrapper(*args, **kwargs):
        print(f"开始执行 {func.__name__}")
        result = func(*args, **kwargs)
        print("执行结束")
        return result
    return wrapper

@log_time
def say_hello():
    print("Hello!")

say_hello()
```
装饰器本质是**闭包**的应用。常见场景：登录校验、缓存（`@lru_cache`）、计时、日志。
---
## 4. 高阶函数 Higher-order Function
**大白话**：函数可以像快递包裹一样被当参数传递，或被当"产品"生产出来。
```python
nums = [1, 2, 3, 4, 5]
doubled = list(map(lambda x: x * 2, nums))          # [2, 4, 6, 8, 10]
evens = list(filter(lambda x: x % 2 == 0, nums))    # [2, 4]
sorted_by_len = sorted(["banana", "kiwi", "apple"], key=len)
```
---
## 5. 生成器 Generator
**大白话**：普通函数是"一次性把菜全做好端上桌"，生成器是"边吃边做"，不占一大桌子空间。
```python
def read_big_file_lines():
    for i in range(1000000):
        yield f"第{i}行数据"

gen = read_big_file_lines()
print(next(gen))  # 第0行数据 —— 只算了这一个
```
省内存，适合处理大文件、大数据流。
---
## 6. 迭代器 Iterator
**大白话**：`for` 循环背后有个"发牌员"，喊"下一张"就发一张，没牌了就报错停止。
```python
my_list = [1, 2, 3]
it = iter(my_list)
print(next(it))  # 1
print(next(it))  # 2
```
生成器就是一种特殊的迭代器，写法更简单。
---
## 7. GIL 全局解释器锁
**大白话**：Python 解释器里有个"唯一的麦克风"，同一时刻只能一个线程执行代码，其他线程排队。
**影响**：多线程在 CPU 密集型任务上基本没有加速效果，因为大家轮流抢麦克风。
---
## 8. 多线程 Threading
**大白话**：适合"等待型"任务——比如同时下载多个文件，等网络时不占麦克风，可以趁机切换给别的线程。
```python
import threading, time

def download(name):
    print(f"{name} 开始下载")
    time.sleep(2)
    print(f"{name} 下载完成")

threads = [threading.Thread(target=download, args=(f"文件{i}",)) for i in range(3)]
for t in threads: t.start()
for t in threads: t.join()
```
---
## 9. 多进程 Multiprocessing
**大白话**：一个进程只有一个麦克风，那就开多个"会场"（进程），各自有麦克风，真正利用多核 CPU。
```python
from multiprocessing import Pool

def heavy_compute(n):
    return sum(i * i for i in range(n))

if __name__ == "__main__":
    with Pool(4) as p:
        results = p.map(heavy_compute, [10**7] * 4)
```
**区分**：I/O 密集型 → 多线程/异步；CPU 密集型 → 多进程。
---
## 10. 协程 Coroutine
**大白话**：`async def` 定义出来的函数的正式学名——能自己主动暂停、让出执行权，之后再从暂停处接着跑。
区别于线程/进程（操作系统抢占式调度），协程是程序**自己主动**在 `await` 处让出控制权，更轻量，可同时开几万个。
---
## 11. 魔法方法 Magic Method
**大白话**：`__xxx__` 这种前后双下划线包裹的方法，是 Python 留的"钩子"，让自定义对象支持 `+`、`print()` 等内置操作。
```python
class Point:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __str__(self):
        return f"Point({self.x}, {self.y})"

    def __add__(self, other):
        return Point(self.x + other.x, self.y + other.y)

p1, p2 = Point(1, 2), Point(3, 4)
print(p1 + p2)  # Point(4, 6)
```
---
## 12. `@property`
**大白话**：让"方法"伪装成"属性"用——访问时不用加括号，但背后偷偷跑逻辑（比如校验）。
```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def area(self):
        return 3.14 * self._radius ** 2

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value <= 0:
            raise ValueError("半径必须大于0")
        self._radius = value
```
---
## 13. 继承 vs 组合
**大白话**：
- **继承**："我是XX的一种"（狗是动物的一种）
- **组合**："我里面装了一个XX"（车里装了引擎，车不是引擎的一种）
```python
class Animal:
    def eat(self): print("吃东西")

class Dog(Animal):          # 继承
    def bark(self): print("汪汪叫")

class Engine:
    def start(self): print("引擎启动")

class Car:
    def __init__(self):
        self.engine = Engine()   # 组合
    def drive(self):
        self.engine.start()
```
**经验法则**：能用组合就别用继承，继承层级一多容易变僵硬。
---
## 14. 元类 Metaclass
**大白话**：类是创建对象的模具，元类是创建模具的模具。
```python
print(type(5))     # <class 'int'>
print(type(int))   # <class 'type'>
```
对象由类创建，类由元类（默认是 `type`）创建。99% 场景用不到，了解即可（用于理解 Django ORM 等框架的黑魔法）。
---
## 15. 可变 vs 不可变对象
**大白话**：
- **不可变**：`int`、`str`、`tuple` —— 一次性打印的纸，改不了，只能重印
- **可变**：`list`、`dict`、`set` —— 白板，可以直接涂改
**经典坑**（可变默认参数）：
```python
def add_item(item, my_list=[]):   # 危险！
    my_list.append(item)
    return my_list

print(add_item(1))  # [1]
print(add_item(2))  # [1, 2]  ← 不是[2]！默认参数只创建一次，被复用了
```
正确写法：`my_list=None`，函数内部 `if my_list is None: my_list = []`。
---
## 16. 深拷贝 vs 浅拷贝
**大白话**：文件夹里装着子文件夹——
- **浅拷贝**：只复制最外层，子文件夹还是同一个
- **深拷贝**：连子文件夹也整个复制，彻底独立
```python
import copy

original = [[1, 2], [3, 4]]
shallow = copy.copy(original)
shallow[0][0] = "改了"
print(original)  # [['改了', 2], [3, 4]]  ← 原始也被改了

deep = copy.deepcopy(original)
deep[0][0] = "再改"
print(original)  # 不受影响
```
---
## 17. 上下文管理器 Context Manager
**大白话**：`with ... as ...:` 背后的机制，保证"用完一定自动清理"，像酒店房间自动门锁。
```python
with open("test.txt", "w") as f:
    f.write("hello")
# 出了缩进块，文件自动关闭
```
自定义实现：
```python
class MyContext:
    def __enter__(self):
        print("进入，准备资源")
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        print("退出，自动清理资源")

with MyContext():
    print("干活中...")
```
---
## 18. `args` 和 `*kwargs`
**大白话**：函数参数个数不确定时用的"万能收纳箱"——`*args` 收无名参数（打包成元组），`**kwargs` 收 `名字=值` 参数（打包成字典）。
```python
def show_info(*args, **kwargs):
    print("args:", args)
    print("kwargs:", kwargs)

show_info(1, 2, 3, name="小明", age=18)
# args: (1, 2, 3)
# kwargs: {'name': '小明', 'age': 18}
```
---
## 19. 列表推导式 / 生成器表达式
**大白话**：把三行 `for` 循环压缩成一行，更 Pythonic。
```python
# 传统写法
squares = []
for x in range(10):
    if x % 2 == 0:
        squares.append(x ** 2)

# 列表推导式
squares = [x ** 2 for x in range(10) if x % 2 == 0]

# 生成器表达式（圆括号，懒加载，不占内存）
squares_gen = (x ** 2 for x in range(10) if x % 2 == 0)
```
---
## 20. 异常处理 Exception Handling
**大白话**：程序出意外时不直接崩溃，而是优雅兜底——像走路踩到石头稳住继续走。
```python
try:
    result = 10 / 0
except ZeroDivisionError as e:
    print(f"出错了：{e}")
else:
    print("没出错才执行这里")
finally:
    print("无论如何都会执行")   # 常用来关闭资源
```
---
## 学习路径建议
1. **先巩固**：可变/不可变对象、`args/**kwargs`、异常处理、列表推导式（基础必备）
2. **再进阶**：装饰器、生成器、迭代器、上下文管理器（写出 Pythonic 代码的关键）
3. **最后深入**：GIL、多线程/多进程、协程、元类（面试加分项，理解底层原理）