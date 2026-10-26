# 软件工程核心知识（大白话版 + Python 示例）
#
---
## 一、SOLID 原则
### 1. 单一职责原则（SRP）
**大白话：** 一个类只干一件事。厨师只管做菜，不管端盘子。
```python
# 反面：一个类又管用户数据又管发邮件
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email

    def save(self):
        print(f"保存用户 {self.name}")

    def send_email(self, msg):  # 不该在这里
        print(f"发邮件给 {self.email}: {msg}")

# 正面：拆开
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email

    def save(self):
        print(f"保存用户 {self.name}")

class EmailService:
    def send(self, to, msg):
        print(f"发邮件给 {to}: {msg}")
```
---
### 2. 开闭原则（OCP）
**大白话：** 对扩展开放，对修改关闭。加新功能不用改老代码，而是写新代码插进去。
```python
from abc import ABC, abstractmethod

class Discount(ABC):
    @abstractmethod
    def calculate(self, price): ...

class VIPDiscount(Discount):
    def calculate(self, price):
        return price * 0.8

class StudentDiscount(Discount):
    def calculate(self, price):
        return price * 0.7

# 新增折扣类型不需要修改已有代码
def checkout(price, discount: Discount):
    return discount.calculate(price)
```
---
### 3. 里氏替换原则（LSP）
**大白话：** 子类能替代父类出现在任何地方，且行为不出错。儿子得能干老子能干的事。
```python
class Bird:
    def fly(self):
        return "飞行中"

class Sparrow(Bird):
    def fly(self):
        return "麻雀飞行中"

# 反面：企鹅不能飞，继承Bird就违反了LSP
class Penguin(Bird):
    def fly(self):
        raise Exception("企鹅不会飞！")  # 违反LSP

# 正面：重新设计继承体系
class Bird:
    pass

class FlyingBird(Bird):
    def fly(self):
        return "飞行中"

class Penguin(Bird):
    def swim(self):
        return "游泳中"
```
---
### 4. 接口隔离原则（ISP）
**大白话：** 别强迫别人实现用不到的方法。菜单上别放顾客不需要的东西。
```python
from abc import ABC, abstractmethod

# 反面：一个大接口
class Worker(ABC):
    @abstractmethod
    def work(self): ...
    @abstractmethod
    def eat(self): ...  # 机器人不需要吃饭

# 正面：拆成小接口
class Workable(ABC):
    @abstractmethod
    def work(self): ...

class Eatable(ABC):
    @abstractmethod
    def eat(self): ...

class Human(Workable, Eatable):
    def work(self): return "干活"
    def eat(self): return "吃饭"

class Robot(Workable):
    def work(self): return "干活"
```
---
### 5. 依赖倒置原则（DIP）
**大白话：** 高层不依赖底层的具体实现，都依赖抽象。老板不直接指挥工人，而是通过"岗位职责"来管理。
```python
from abc import ABC, abstractmethod

class MessageSender(ABC):
    @abstractmethod
    def send(self, msg): ...

class EmailSender(MessageSender):
    def send(self, msg):
        print(f"邮件: {msg}")

class SmsSender(MessageSender):
    def send(self, msg):
        print(f"短信: {msg}")

class NotificationService:
    def __init__(self, sender: MessageSender):  # 依赖抽象，不依赖具体
        self.sender = sender

    def notify(self, msg):
        self.sender.send(msg)

service = NotificationService(SmsSender())
service.notify("你的快递到了")
```
---
## 二、并发编程模式
### 1. 生产者-消费者模式
**大白话：** 一边生产任务放到队列里，另一边从队列里取任务处理。厨房做菜放窗口，服务员从窗口端菜。
```python
import threading
import queue
import time

q = queue.Queue(maxsize=5)

def producer():
    for i in range(10):
        q.put(f"任务{i}")
        print(f"生产: 任务{i}")
        time.sleep(0.1)

def consumer():
    while True:
        task = q.get()
        if task is None:
            break
        print(f"消费: {task}")
        q.task_done()

t1 = threading.Thread(target=producer)
t2 = threading.Thread(target=consumer)
t1.start()
t2.start()
t1.join()
q.put(None)  # 毒丸信号，通知消费者退出
t2.join()
```
---
### 2. 线程池模式
**大白话：** 提前雇好一批工人，有活就分配，别每次临时招人。
```python
from concurrent.futures import ThreadPoolExecutor
import time

def do_work(task_id):
    time.sleep(0.5)
    return f"任务{task_id}完成"

with ThreadPoolExecutor(max_workers=3) as pool:
    futures = [pool.submit(do_work, i) for i in range(6)]
    for f in futures:
        print(f.result())
```
---
### 3. 异步编程（async/await）
**大白话：** 遇到等待（网络、IO）就先去干别的事，等好了再回来继续。一个人同时泡三杯茶。
```python
import asyncio

async def fetch_data(name, delay):
    print(f"开始获取 {name}")
    await asyncio.sleep(delay)  # 模拟IO等待
    print(f"获取完成 {name}")
    return f"{name}的数据"

async def main():
    results = await asyncio.gather(
        fetch_data("用户", 2),
        fetch_data("订单", 1),
        fetch_data("商品", 1.5),
    )
    print(results)

asyncio.run(main())
# 三个任务并发执行，总耗时约2秒而非4.5秒
```
---
## 三、架构模式
### 1. MVC（Model-View-Controller）
**大白话：** 数据、展示、逻辑三件事分开管。厨师管做菜（Model），服务员管端菜（Controller），餐盘管摆盘（View）。
```python
# Model - 数据层
class UserModel:
    def __init__(self):
        self.users = {"1": "张三", "2": "李四"}

    def get_user(self, uid):
        return self.users.get(uid)

# View - 展示层
class UserView:
    def render(self, user):
        print(f"┌─────────────┐")
        print(f"│ 用户: {user} │")
        print(f"└─────────────┘")

# Controller - 逻辑层
class UserController:
    def __init__(self):
        self.model = UserModel()
        self.view = UserView()

    def show_user(self, uid):
        user = self.model.get_user(uid)
        if user:
            self.view.render(user)
        else:
            print("用户不存在")

ctrl = UserController()
ctrl.show_user("1")
```
---
### 2. 仓储模式（Repository）
**大白话：** 在业务代码和数据库之间加一层，业务代码不直接碰SQL。像图书管理员，你只需要说"帮我找本Python的书"。
```python
from abc import ABC, abstractmethod

class UserRepository(ABC):
    @abstractmethod
    def find_by_id(self, uid): ...
    @abstractmethod
    def save(self, user): ...

class MySQLUserRepository(UserRepository):
    def __init__(self):
        self._db = {}

    def find_by_id(self, uid):
        return self._db.get(uid)

    def save(self, user):
        self._db[user["id"]] = user

# 业务代码只依赖 UserRepository 接口
# 换数据库只需换实现类，业务代码不动
repo = MySQLUserRepository()
repo.save({"id": "1", "name": "张三"})
print(repo.find_by_id("1"))
```
---
### 3. 中间件模式（Middleware）
**大白话：** 请求进来像过安检，一层一层检查/处理，每层各管各的事。
```python
class Middleware:
    def __init__(self):
        self._middlewares = []

    def use(self, fn):
        self._middlewares.append(fn)

    def run(self, request):
        def execute(index):
            if index >= len(self._middlewares):
                return request
            def next_fn():
                return execute(index + 1)
            return self._middlewares[index](request, next_fn)
        return execute(0)

app = Middleware()

def auth_middleware(req, next):
    if req.get("token"):
        print("✓ 鉴权通过")
        return next()
    return "未授权"

def log_middleware(req, next):
    print(f"→ 请求: {req['path']}")
    result = next()
    print(f"← 响应完成")
    return result

app.use(log_middleware)
app.use(auth_middleware)
app.run({"path": "/api/users", "token": "abc123"})
```
---
## 四、容错与重试模式
### 1. 重试模式（Retry）
**大白话：** 失败了就再试几次，别一次失败就放弃。像拨电话占线了，过会儿再拨。
```python
import time
import random

def retry(max_attempts=3, delay=1, backoff=2):
    def decorator(func):
        def wrapper(*args, **kwargs):
            attempt = 0
            current_delay = delay
            while attempt < max_attempts:
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    attempt += 1
                    if attempt == max_attempts:
                        raise
                    print(f"第{attempt}次失败，{current_delay}秒后重试: {e}")
                    time.sleep(current_delay)
                    current_delay *= backoff
        return wrapper
    return decorator

@retry(max_attempts=3, delay=0.1)
def unstable_api():
    if random.random() < 0.7:
        raise ConnectionError("网络抖动")
    return "成功"
```
---
### 2. 熔断器模式（Circuit Breaker）
**大白话：** 下游服务挂了就别一直调了，先断开保护自己，过一会儿试试恢复没。像家里电路跳闸保护电器。
```python
import time

class CircuitBreaker:
    def __init__(self, failure_threshold=3, recovery_timeout=5):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failures = 0
        self.state = "CLOSED"  # CLOSED / OPEN / HALF_OPEN
        self.last_failure_time = None

    def call(self, func, *args):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = "HALF_OPEN"
            else:
                raise Exception("熔断器开启，拒绝请求")

        try:
            result = func(*args)
            self.failures = 0
            self.state = "CLOSED"
            return result
        except Exception as e:
            self.failures += 1
            self.last_failure_time = time.time()
            if self.failures >= self.failure_threshold:
                self.state = "OPEN"
                print(f"熔断器开启！连续失败{self.failures}次")
            raise

breaker = CircuitBreaker(failure_threshold=3, recovery_timeout=5)
```
---
### 3. 限流模式（Rate Limiting）
**大白话：** 控制请求速度，别把下游打爆了。像高速公路收费站，一次只放那么多车。
```python
import time
from collections import deque

class RateLimiter:
    """滑动窗口限流"""
    def __init__(self, max_requests, window_seconds):
        self.max_requests = max_requests
        self.window = window_seconds
        self.requests = deque()

    def allow(self):
        now = time.time()
        while self.requests and self.requests[0] < now - self.window:
            self.requests.popleft()
        if len(self.requests) < self.max_requests:
            self.requests.append(now)
            return True
        return False

limiter = RateLimiter(max_requests=5, window_seconds=1)

for i in range(10):
    if limiter.allow():
        print(f"请求{i}: 通过")
    else:
        print(f"请求{i}: 被限流")
```
---
## 五、数据处理模式
### 1. 管道模式（Pipeline）
**大白话：** 数据像流水线一样，经过一个个加工站，每站干一件事。
```python
class Pipeline:
    def __init__(self):
        self.steps = []

    def add(self, func):
        self.steps.append(func)
        return self

    def run(self, data):
        for step in self.steps:
            data = step(data)
        return data

result = (
    Pipeline()
    .add(lambda text: text.strip())
    .add(lambda text: text.lower())
    .add(lambda text: text.replace(" ", "_"))
    .add(lambda text: f"processed_{text}")
    .run("  Hello World  ")
)
print(result)  # processed_hello_world
```
---
### 2. 批处理模式（Batch）
**大白话：** 别一条一条处理，攒一批一起搞，效率高。像快递员攒一车再送，不是收一个送一个。
```python
def batch_process(items, batch_size, processor):
    results = []
    for i in range(0, len(items), batch_size):
        batch = items[i:i + batch_size]
        print(f"处理批次: {batch}")
        results.extend(processor(batch))
    return results

def insert_to_db(batch):
    # 模拟批量插入
    return [f"inserted_{item}" for item in batch]

data = list(range(10))
results = batch_process(data, batch_size=3, processor=insert_to_db)
```
---
### 3. 缓存模式（Cache-Aside）
**大白话：** 先看缓存有没有，有就直接用；没有就去数据库查，查完放缓存里。
```python
class CacheAside:
    def __init__(self):
        self._cache = {}
        self._db = {"user:1": "张三", "user:2": "李四"}

    def get(self, key):
        # 先查缓存
        if key in self._cache:
            print(f"缓存命中: {key}")
            return self._cache[key]
        # 缓存没有，查数据库
        value = self._db.get(key)
        if value:
            self._cache[key] = value  # 写入缓存
            print(f"数据库查询并缓存: {key}")
        return value

    def update(self, key, value):
        self._db[key] = value
        self._cache.pop(key, None)  # 删除缓存，下次读时重建

cache = CacheAside()
cache.get("user:1")  # 数据库查询并缓存
cache.get("user:1")  # 缓存命中
```
---
## 六、测试相关模式
### 1. 依赖注入（方便测试）
**大白话：** 把依赖从外面传进来，测试时可以传假的（mock）进去。
```python
class OrderService:
    def __init__(self, payment_gateway, inventory):
        self.payment = payment_gateway
        self.inventory = inventory

    def place_order(self, item, amount):
        if not self.inventory.check(item):
            return "库存不足"
        self.payment.charge(amount)
        return "下单成功"

# 测试时传入假实现
class FakePayment:
    def charge(self, amount):
        self.charged = amount

class FakeInventory:
    def check(self, item):
        return True

service = OrderService(FakePayment(), FakeInventory())
assert service.place_order("iPhone", 9999) == "下单成功"
```
---
### 2. Fixture 模式
**大白话：** 测试前准备好数据，测试后清理掉。像考试前发卷子，考完收卷子。
```python
import pytest

@pytest.fixture
def sample_users():
    """准备测试数据"""
    users = [
        {"id": 1, "name": "张三", "age": 25},
        {"id": 2, "name": "李四", "age": 30},
    ]
    yield users
    # 这里可以做清理工作
    print("清理测试数据")

def test_find_user(sample_users):
    result = [u for u in sample_users if u["name"] == "张三"]
    assert len(result) == 1
    assert result[0]["age"] == 25
```
---
## 七、实用编程技巧
### 1. 上下文管理器（资源管理）
**大白话：** 用完自动关，不用操心忘记释放资源。像自动门，进去办事，出来自动关门。
```python
from contextlib import contextmanager

@contextmanager
def timer(name):
    import time
    start = time.time()
    yield
    elapsed = time.time() - start
    print(f"{name} 耗时: {elapsed:.2f}秒")

with timer("数据处理"):
    sum(range(1000000))

# 自定义数据库连接管理
@contextmanager
def db_connection(url):
    conn = f"连接到 {url}"
    print(f"打开: {conn}")
    try:
        yield conn
    finally:
        print(f"关闭: {conn}")

with db_connection("localhost:3306") as conn:
    print(f"使用 {conn} 查询")
```
---
### 2. 生成器（惰性求值）
**大白话：** 数据太多一次性加载不下？那就要一条给一条，用多少算多少。
```python
def read_large_file(path):
    """逐行读大文件，内存只占一行的空间"""
    with open(path) as f:
        for line in f:
            yield line.strip()

def fibonacci():
    """无限序列，要几个取几个"""
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

# 取前10个斐波那契数
from itertools import islice
print(list(islice(fibonacci(), 10)))
# [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```
---
### 3. 注册表模式（Registry）
**大白话：** 搞个花名册，新成员自己来登记，需要时按名字找。插件系统的基础。
```python
class HandlerRegistry:
    _handlers = {}

    @classmethod
    def register(cls, name):
        def decorator(func):
            cls._handlers[name] = func
            return func
        return decorator

    @classmethod
    def get(cls, name):
        return cls._handlers.get(name)

@HandlerRegistry.register("json")
def handle_json(data):
    return f"处理JSON: {data}"

@HandlerRegistry.register("xml")
def handle_xml(data):
    return f"处理XML: {data}"

handler = HandlerRegistry.get("json")
print(handler('{"a": 1}'))  # 处理JSON: {"a": 1}
```
---
## 总结速查表
| 类别 | 模式 | 一句话 |
| --- | --- | --- |
| SOLID | 单一职责 | 一个类只管一件事 |
| SOLID | 开闭原则 | 加功能写新代码，别改旧代码 |
| SOLID | 里氏替换 | 子类能无缝替代父类 |
| SOLID | 接口隔离 | 接口要小而专 |
| SOLID | 依赖倒置 | 依赖抽象，不依赖具体 |
| 并发 | 生产者-消费者 | 队列解耦生产和消费 |
| 并发 | 线程池 | 复用线程，避免频繁创建 |
| 并发 | async/await | IO等待时去干别的事 |
| 架构 | MVC | 数据、展示、逻辑分离 |
| 架构 | 仓储 | 业务和数据库之间加一层 |
| 架构 | 中间件 | 请求像过安检逐层处理 |
| 容错 | 重试 | 失败了多试几次 |
| 容错 | 熔断器 | 下游挂了先断开保护 |
| 容错 | 限流 | 控制速度别打爆下游 |
| 数据 | 管道 | 数据流水线逐步加工 |
| 数据 | 批处理 | 攒一批一起处理更快 |
| 数据 | 缓存旁路 | 先查缓存，没有再查库 |
| 测试 | 依赖注入 | 从外面传依赖，方便换 |
| 技巧 | 上下文管理器 | 用完自动释放资源 |
| 技巧 | 生成器 | 惰性求值省内存 |
| 技巧 | 注册表 | 花名册式插件管理 |