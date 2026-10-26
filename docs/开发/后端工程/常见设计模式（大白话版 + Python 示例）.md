# 常见设计模式（大白话版 + Python 示例）
---
## 一、创建型模式
### 1. 单例模式（Singleton）
**大白话：** 全局只允许有一个实例，谁来要都给同一个。比如数据库连接池，整个应用只需要一个。
```python
class Database:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.connection = "connected"
        return cls._instance

db1 = Database()
db2 = Database()
print(db1 is db2)  # True，同一个对象
```
---
### 2. 工厂模式（Factory）
**大白话：** 你告诉我要什么类型，我帮你创建对应的对象。你不用关心具体怎么造出来的。
```python
class Dog:
    def speak(self):
        return "汪汪"

class Cat:
    def speak(self):
        return "喵喵"

def animal_factory(animal_type):
    animals = {"dog": Dog, "cat": Cat}
    return animals[animal_type]()

pet = animal_factory("dog")
print(pet.speak())  # 汪汪
```
---
### 3. 建造者模式（Builder）
**大白话：** 一步一步拼装复杂对象，像点奶茶一样——选大小、选糖度、选加料，最后出一杯完整的。
```python
class Pizza:
    def __init__(self):
        self.size = None
        self.cheese = False
        self.pepperoni = False

class PizzaBuilder:
    def __init__(self):
        self.pizza = Pizza()

    def set_size(self, size):
        self.pizza.size = size
        return self

    def add_cheese(self):
        self.pizza.cheese = True
        return self

    def add_pepperoni(self):
        self.pizza.pepperoni = True
        return self

    def build(self):
        return self.pizza

pizza = PizzaBuilder().set_size("large").add_cheese().add_pepperoni().build()
```
---
## 二、结构型模式
### 4. 适配器模式（Adapter）
**大白话：** 接口不兼容？加个转接头。就像出国旅行带的插头转换器。
```python
class OldPrinter:
    def print_old(self, text):
        return f"[旧打印机] {text}"

class PrinterAdapter:
    def __init__(self, old_printer):
        self.old_printer = old_printer

    def print(self, text):
        return self.old_printer.print_old(text)

adapter = PrinterAdapter(OldPrinter())
print(adapter.print("hello"))  # [旧打印机] hello
```
---
### 5. 装饰器模式（Decorator）
**大白话：** 给对象动态加功能，像穿衣服一样一层一层往上套，不改原来的东西。
```python
def bold(func):
    def wrapper(text):
        return f"<b>{func(text)}</b>"
    return wrapper

def italic(func):
    def wrapper(text):
        return f"<i>{func(text)}</i>"
    return wrapper

@bold
@italic
def format_text(text):
    return text

print(format_text("hello"))  # <b><i>hello</i></b>
```
---
### 6. 代理模式（Proxy）
**大白话：** 找了个中间人帮你干活，可以在前后加点逻辑（鉴权、缓存、日志等）。
```python
class RealDatabase:
    def query(self, sql):
        return f"执行: {sql}"

class DatabaseProxy:
    def __init__(self):
        self._db = RealDatabase()
        self._cache = {}

    def query(self, sql):
        if sql not in self._cache:
            self._cache[sql] = self._db.query(sql)
        return self._cache[sql]

db = DatabaseProxy()
print(db.query("SELECT * FROM users"))  # 第一次真查
print(db.query("SELECT * FROM users"))  # 第二次走缓存
```
---
## 三、行为型模式
### 7. 观察者模式（Observer）
**大白话：** 订阅机制——你关注了某个公众号，它一发文章，所有粉丝都能收到通知。
```python
class EventSystem:
    def __init__(self):
        self._listeners = {}

    def subscribe(self, event, callback):
        self._listeners.setdefault(event, []).append(callback)

    def emit(self, event, data):
        for callback in self._listeners.get(event, []):
            callback(data)

events = EventSystem()
events.subscribe("order_paid", lambda d: print(f"发短信: 订单{d}已付款"))
events.subscribe("order_paid", lambda d: print(f"减库存: 订单{d}"))
events.emit("order_paid", "10086")
# 发短信: 订单10086已付款
# 减库存: 订单10086
```
---
### 8. 策略模式（Strategy）
**大白话：** 同一件事有多种做法，把每种做法封装起来，运行时选一个用。比如支付方式——支付宝、微信、银行卡。
```python
def pay_by_alipay(amount):
    return f"支付宝支付 {amount} 元"

def pay_by_wechat(amount):
    return f"微信支付 {amount} 元"

class PaymentContext:
    def __init__(self, strategy):
        self.strategy = strategy

    def pay(self, amount):
        return self.strategy(amount)

ctx = PaymentContext(pay_by_wechat)
print(ctx.pay(100))  # 微信支付 100 元
```
---
### 9. 模板方法模式（Template Method）
**大白话：** 老爸定好做事的步骤框架，具体每一步怎么做让儿子自己决定。
```python
from abc import ABC, abstractmethod

class DataExporter(ABC):
    def export(self):
        data = self.fetch_data()
        formatted = self.format(data)
        self.save(formatted)

    @abstractmethod
    def fetch_data(self): ...

    @abstractmethod
    def format(self, data): ...

    @abstractmethod
    def save(self, data): ...

class CSVExporter(DataExporter):
    def fetch_data(self):
        return [{"name": "张三", "age": 20}]

    def format(self, data):
        return "name,age\n张三,20"

    def save(self, data):
        print(f"保存CSV: {data}")

CSVExporter().export()
```
---
### 10. 责任链模式（Chain of Responsibility）
**大白话：** 请求像流水线一样，一个处理不了就传给下一个，直到有人能处理。比如审批流程：组长→经理→总监。
```python
class Handler:
    def __init__(self, successor=None):
        self.successor = successor

    def handle(self, amount):
        if self.successor:
            return self.successor.handle(amount)
        return "无人能审批"

class Manager(Handler):
    def handle(self, amount):
        if amount <= 1000:
            return "经理审批通过"
        return super().handle(amount)

class Director(Handler):
    def handle(self, amount):
        if amount <= 10000:
            return "总监审批通过"
        return super().handle(amount)

chain = Manager(Director())
print(chain.handle(500))    # 经理审批通过
print(chain.handle(5000))   # 总监审批通过
print(chain.handle(50000))  # 无人能审批
```
---
## 速查表
| 模式 | 一句话总结 | 典型场景 |
| --- | --- | --- |
| 单例 | 全局就一个 | 配置、连接池、日志 |
| 工厂 | 告诉我类型，给你对象 | 多种支付渠道、消息类型 |
| 建造者 | 一步步拼装 | 复杂配置、SQL拼接 |
| 适配器 | 加转接头 | 对接老系统/第三方 |
| 装饰器 | 套娃加功能 | 日志、权限、缓存 |
| 代理 | 找中间人 | 缓存、懒加载、权限控制 |
| 观察者 | 订阅通知 | 事件系统、消息队列 |
| 策略 | 多种做法换着用 | 算法选择、支付方式 |
| 模板方法 | 框架定步骤，子类填细节 | 数据导出、ETL流程 |
| 责任链 | 一个个往后传 | 审批流、中间件、过滤器 |