# Python 下划线与 dunder 方法
Python 中下划线的使用分为四种约定，先理清命名，再深入 dunder（双下划线）方法的机制。
## 一、四种下划线命名约定
| 形式 | 名称 | 含义 |
| --- | --- | --- |
| `_name` | 单前导下划线 | 「弱私有」约定，`from m import *` 不会导入，但仍可访问 |
| `name_` | 单后置下划线 | 避免与关键字冲突，如 `class_`、`type_` |
| `__name` | 双前导下划线 | **名称改写（name mangling）**，编译期改为 `_ClassName__name` |
| `__name__` | 双下划线包围 | **dunder**，Python 保留给解释器调用的协议方法 |
关键点：`__name` 不是"更私有"，而是防止子类意外覆盖父类属性。它由**编译器**在类作用域内静态改写。
```python
class A:
    def __init__(self):
        self.__x = 1   # 实际存为 self._A__x

class B(A):
    def peek(self):
        return self.__x  # 会去找 self._B__x，报 AttributeError
```
## 二、Python 的「协议」思想
Python 的哲学是**协议 \> 继承**。任何对象只要实现了对应 dunder 方法，就"是"某种类型。
- 实现 `__iter__` → 就是可迭代对象
- 实现 `__len__` 和 `__getitem__` → 就是序列
- 实现 `__enter__`/`__exit__` → 就是上下文管理器
- 实现 `__call__` → 就是可调用对象
内置函数的调用是**间接分发**的：`len(x)` 实际执行 `type(x).__len__(x)`，注意是**从类型上查找**，不是从实例上，这叫 **implicit lookup**。
```python
class Foo:
    pass

f = Foo()
f.__len__ = lambda: 5
len(f)  # TypeError！实例上的 __len__ 不算数
```
## 三、核心 dunder 分类精讲
### 1. 对象生命周期
```python
class Point:
    def __new__(cls, *a, **kw):     # 构造：返回实例
        print("new")
        return super().__new__(cls)

    def __init__(self, x, y):        # 初始化：填充实例
        print("init")
        self.x, self.y = x, y

    def __del__(self):               # 析构：引用计数归零时触发
        print("del")
```
理解要点：
- `__new__` 是**类方法**（隐式），负责分配对象；`__init__` 负责初始化。想返回单例、缓存实例、或继承 `int`/`str`/`tuple` 等不可变类型时，必须用 `__new__`。
- `__del__` **不等于** C++ 的析构函数。循环引用、解释器退出、异常都可能让它不执行。**不要依赖它释放资源**，用 `with`。
### 2. 表示协议：`__repr__` vs `__str__`
```python
class Money:
    def __init__(self, amount, currency):
        self.amount, self.currency = amount, currency

    def __repr__(self):
        return f"Money({self.amount!r}, {self.currency!r})"  # 面向开发者，力求可 eval

    def __str__(self):
        return f"{self.amount} {self.currency}"              # 面向用户
```
- `repr(x)` / 交互式回显 → `__repr__`
- `str(x)` / `print(x)` / f-string → `__str__`，缺失时回退到 `__repr__`
- 只写一个的话，**优先写 ****`__repr__`**
### 3. 属性访问协议（最容易踩坑）
```python
class Proxy:
    def __getattribute__(self, name):   # 所有属性访问都走这里
        print(f"get {name}")
        return super().__getattribute__(name)

    def __getattr__(self, name):        # 仅当正常查找失败时触发
        return f"fallback:{name}"

    def __setattr__(self, name, value):
        super().__setattr__(name, value)

    def __delattr__(self, name):
        super().__delattr__(name)
```
深度要点：
- `__getattribute__` 是**无条件拦截**，重写它极易死循环，必须用 `super().__getattribute__` 或 `object.__getattribute__` 绕开。
- `__getattr__` 是**兜底**，只在 MRO 和实例字典都找不到时才调用。ORM、RPC 客户端、Mock 对象大量使用。
- 属性查找顺序（简化）：`type(obj)` 的数据描述符 → `obj.__dict__` → `type(obj)` 的非数据描述符和普通属性 → `__getattr__`。
### 4. 描述符协议（Python 的隐藏 boss）
`property`、`classmethod`、`staticmethod`、`__slots__` 中的槽位、绑定方法……**全部**基于描述符。
```python
class Typed:
    def __init__(self, type_):
        self.type_ = type_

    def __set_name__(self, owner, name):  # 3.6+ 自动获知属性名
        self.name = name

    def __get__(self, instance, owner):
        if instance is None:
            return self
        return instance.__dict__[self.name]

    def __set__(self, instance, value):
        if not isinstance(value, self.type_):
            raise TypeError(f"{self.name} must be {self.type_}")
        instance.__dict__[self.name] = value

class User:
    age = Typed(int)
    name = Typed(str)
```
规则：
- 同时有 `__get__` 和 `__set__`/`__delete__` → **数据描述符**，优先级高于实例字典
- 只有 `__get__` → **非数据描述符**（如函数），优先级低于实例字典
- 这就是为什么 `obj.method = something` 能覆盖普通方法但不能覆盖 `property`
### 5. 运算符重载与反射
```python
class Vec:
    def __init__(self, x, y): self.x, self.y = x, y
    def __add__(self, other):  return Vec(self.x+other.x, self.y+other.y)
    def __radd__(self, other): return self.__add__(other)  # 3 + vec
    def __iadd__(self, other):                             # vec += ...
        self.x += other.x; self.y += other.y
        return self
```
分派规则：`a + b` → 先试 `type(a).__add__(a, b)`；若返回 `NotImplemented`，再试 `type(b).__radd__(b, a)`。**特殊情况**：若 `type(b)` 是 `type(a)` 的**子类**，`__radd__` 反而先调用（让子类有机会覆盖）。
比较运算符：`__lt__`、`__le__`、`__eq__`、`__ne__`、`__gt__`、`__ge__`。定义了 `__eq__` 应同时定义 `__hash__`（或显式设为 `None` 表示不可哈希）。用 `functools.total_ordering` 可以只写两个方法。
### 6. 容器与迭代协议
```python
class Range:
    def __init__(self, n): self.n = n
    def __len__(self):     return self.n
    def __getitem__(self, i):
        if i >= self.n: raise IndexError
        return i * i
    def __contains__(self, x): return x in {i*i for i in range(self.n)}
    def __iter__(self):
        for i in range(self.n): yield i*i
    def __reversed__(self):
        for i in range(self.n-1, -1, -1): yield i*i
```
冷知识：只实现 `__getitem__`（不需要 `__iter__`）也能被 `for` 循环，Python 会传 0, 1, 2… 直到 `IndexError`。这是"老式"迭代协议。
### 7. 上下文管理器
```python
class Timer:
    def __enter__(self):
        self.t = time.perf_counter()
        return self
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.elapsed = time.perf_counter() - self.t
        return False  # 返回 True 会吞掉异常，通常返回 False/None
```
对应 `async with` 有 `__aenter__` / `__aexit__`；对应 `async for` 有 `__aiter__` / `__anext__`。
### 8. 可调用与哈希
```python
class Adder:
    def __init__(self, n): self.n = n
    def __call__(self, x): return x + self.n   # 让实例像函数

add5 = Adder(5)
add5(10)  # 15
```
`__hash__` 契约：`a == b ⟹ hash(a) == hash(b)`。反过来不成立（哈希冲突正常）。可变对象通常不该实现 `__hash__`，否则放进 `dict` 后修改字段，会永远找不到。
### 9. 类创建协议（元编程）
```python
class Meta(type):
    def __new__(mcs, name, bases, namespace, **kw):
        # 类创建时执行
        return super().__new__(mcs, name, bases, namespace)

class Base(metaclass=Meta):
    def __init_subclass__(cls, **kw):  # 3.6+，每个子类创建时触发
        super().__init_subclass__(**kw)
        print(f"subclass {cls.__name__} created")
```
现代 Python 中，`__init_subclass__` 和 `__set_name__` 组合能覆盖 90% 曾经需要元类的场景，优先用它们。
## 四、性能相关的下划线
### `__slots__`
```python
class Point:
    __slots__ = ('x', 'y')  # 禁用 __dict__，用固定槽位
    def __init__(self, x, y):
        self.x, self.y = x, y
```
- 内存占用降低 40-50%（无 `__dict__`）
- 属性访问更快（走描述符协议直达槽位）
- 代价：不能动态加属性；多继承时槽位规则复杂；子类若不声明 `__slots__` 又会带回 `__dict__`
## 五、调试用的内省属性
| 属性 | 含义 |
| --- | --- |
| `__class__` | 实例的类型（可赋值，能"变身"） |
| `__dict__` | 实例或类的属性字典 |
| `__mro__` | 类的方法解析顺序（C3 线性化） |
| `__bases__` / `__subclasses__()` | 父类 / 直接子类 |
| `__module__` / `__qualname__` | 定义所在模块 / 限定名 |
| `__annotations__` | 类型标注字典 |
| `__doc__` | docstring |
## 六、建议的学习路径
1. **先掌握** `__init__`、`__repr__`、`__eq__`、`__hash__`、`__len__`、`__iter__` — 覆盖 80% 场景
2. **再深入**描述符和属性查找机制 — 这是理解 Python 对象模型的钥匙
3. **最后碰**元类和 `__new__` — 通常你不需要，但读框架源码（Django ORM、Pydantic、SQLAlchemy）时必须懂
推荐动手实验：用描述符实现一个类型校验器；用 `__getattr__` 写个属性懒加载；用 `__init_subclass__` 实现插件自动注册。写出来就真懂了。
要不要挑其中某一节（比如描述符、元类、或属性查找顺序）我再展开讲，配上 CPython 层面的实现细节？