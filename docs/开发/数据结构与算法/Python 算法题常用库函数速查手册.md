# Python 算法题常用库函数速查手册
> 刷 LeetCode / 面试写算法必备。所有内容都是 Python 标准库自带，**无需 pip 安装**。
## 目录
- [一、collections 模块](https://app.notion.com/p/3e2f447951a480b3b300cb92c1f3337c?p=3e2f447951a4807d8697dc8faa4c0599&t=new&showMoveTo=true#一collections-模块)
- [二、heapq 模块（堆 / 优先队列）](https://app.notion.com/p/3e2f447951a480b3b300cb92c1f3337c?p=3e2f447951a4807d8697dc8faa4c0599&t=new&showMoveTo=true#二heapq-模块堆--优先队列)
- [三、itertools 模块（迭代器工具）](https://app.notion.com/p/3e2f447951a480b3b300cb92c1f3337c?p=3e2f447951a4807d8697dc8faa4c0599&t=new&showMoveTo=true#三itertools-模块迭代器工具)
- [四、functools 模块（函数工具）](https://app.notion.com/p/3e2f447951a480b3b300cb92c1f3337c?p=3e2f447951a4807d8697dc8faa4c0599&t=new&showMoveTo=true#四functools-模块函数工具)
- [五、bisect 模块（二分查找）](https://app.notion.com/p/3e2f447951a480b3b300cb92c1f3337c?p=3e2f447951a4807d8697dc8faa4c0599&t=new&showMoveTo=true#五bisect-模块二分查找)
- [六、math 模块](https://app.notion.com/p/3e2f447951a480b3b300cb92c1f3337c?p=3e2f447951a4807d8697dc8faa4c0599&t=new&showMoveTo=true#六math-模块)
- [七、内置数据结构常用技巧](https://app.notion.com/p/3e2f447951a480b3b300cb92c1f3337c?p=3e2f447951a4807d8697dc8faa4c0599&t=new&showMoveTo=true#七内置数据结构常用技巧)
- [八、内置函数（Built-in）](https://app.notion.com/p/3e2f447951a480b3b300cb92c1f3337c?p=3e2f447951a4807d8697dc8faa4c0599&t=new&showMoveTo=true#八内置函数built-in)
- [九、字符串处理](https://app.notion.com/p/3e2f447951a480b3b300cb92c1f3337c?p=3e2f447951a4807d8697dc8faa4c0599&t=new&showMoveTo=true#九字符串处理)
- [十、综合速查表](https://app.notion.com/p/3e2f447951a480b3b300cb92c1f3337c?p=3e2f447951a4807d8697dc8faa4c0599&t=new&showMoveTo=true#十综合速查表)
---
## 一、collections 模块
### 1. `deque` —— 双端队列
**用途**：BFS、滑动窗口、单调队列。**两端 O(1) 增删**。
```python
from collections import deque

dq = deque([1, 2, 3])
dq.append(4)         # 尾部加 → [1,2,3,4]
dq.appendleft(0)     # 头部加 → [0,1,2,3,4]
dq.pop()             # 尾部出 → 4
dq.popleft()         # 头部出 → 0
dq.extend([5, 6])    # 尾部批量加
dq.rotate(1)         # 循环右移 1 位

# 限长队列（超出自动挤掉另一端）
dq = deque(maxlen=3)
```
**典型场景**：
```python
# BFS 模板
from collections import deque
def bfs(start):
    q = deque([start])
    visited = {start}
    while q:
        node = q.popleft()
        for nxt in neighbors(node):
            if nxt not in visited:
                visited.add(nxt)
                q.append(nxt)
```
---
### 2. `defaultdict` —— 带默认值的字典
**用途**：分组、计数、建图，省去了 `if key not in dict` 判断。
```python
from collections import defaultdict

# 默认值是 list（分组）
d = defaultdict(list)
d['a'].append(1)     # 不用先初始化
d['a'].append(2)     # {'a': [1, 2]}

# 默认值是 int（计数，从 0 开始）
cnt = defaultdict(int)
for c in "aabbc":
    cnt[c] += 1      # {'a': 2, 'b': 2, 'c': 1}

# 默认值是 set（去重分组）
groups = defaultdict(set)

# 建图
graph = defaultdict(list)
for u, v in edges:
    graph[u].append(v)
    graph[v].append(u)
```
---
### 3. `Counter` —— 计数器
**用途**：统计频次、找出现最多的元素、多重集运算。
```python
from collections import Counter

c = Counter("abracadabra")
# Counter({'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1})

c.most_common(2)       # [('a', 5), ('b', 2)] 前 K 高频
c['a']                 # 5
c['z']                 # 0（不存在返回 0，不报错）

# 多重集运算
c1 = Counter("aab")
c2 = Counter("abc")
c1 + c2                # 相加 {'a':3, 'b':2, 'c':1}
c1 - c2                # 相减（负数丢弃）{'a':1}
c1 & c2                # 交集（取 min）{'a':1, 'b':1}
c1 | c2                # 并集（取 max）{'a':2, 'b':1, 'c':1}

# 判断字母异位词（anagram）
Counter(s1) == Counter(s2)

# 从列表统计
Counter([1,1,2,3,3,3])  # {3:3, 1:2, 2:1}
```
---
### 4. `OrderedDict` —— 有序字典
> Python 3.7+ 普通 dict 已经保序，OrderedDict 主要用它的 `move_to_end`。
```python
from collections import OrderedDict

od = OrderedDict()
od['a'] = 1
od['b'] = 2
od.move_to_end('a')          # 移到末尾
od.move_to_end('a', last=False)  # 移到开头
od.popitem(last=False)       # 弹出第一个（普通 dict 只能弹最后）

# 应用：LRU 缓存
class LRUCache:
    def __init__(self, capacity):
        self.cache = OrderedDict()
        self.cap = capacity

    def get(self, key):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.cap:
            self.cache.popitem(last=False)
```
---
### 5. `namedtuple` —— 具名元组
**用途**：让 tuple 有字段名，代码更可读。
```python
from collections import namedtuple

Point = namedtuple('Point', ['x', 'y'])
p = Point(3, 4)
p.x, p.y             # 3, 4
p[0], p[1]           # 也能按下标访问

# 图算法里表示带权边
Edge = namedtuple('Edge', ['u', 'v', 'w'])
```
---
## 二、heapq 模块（堆 / 优先队列）
**特点**：Python 只有小顶堆。堆顶是 `heap[0]`，`push/pop` 是 O(log n)。
```python
import heapq

# 建堆
h = [3, 1, 4, 1, 5, 9, 2, 6]
heapq.heapify(h)           # 原地建堆 O(n)

# 增删
heapq.heappush(h, 0)       # 插入 O(log n)
heapq.heappop(h)           # 弹出最小 O(log n)
heapq.heappushpop(h, 5)    # 先 push 再 pop（比分开调更快）
heapq.heapreplace(h, 5)    # 先 pop 再 push

# Top K
heapq.nlargest(3, [1,5,2,8,3])   # [8,5,3]
heapq.nsmallest(3, [1,5,2,8,3])  # [1,2,3]
heapq.nlargest(2, students, key=lambda s: s['score'])  # 支持 key
```
### 大顶堆技巧
Python 没有大顶堆，用**取负数**模拟：
```python
max_heap = []
for x in [3, 1, 4, 5]:
    heapq.heappush(max_heap, -x)
print(-heapq.heappop(max_heap))    # 5
```
### 堆里放元组（按第一个元素比较）
```python
# 常见套路：(priority, task)
heapq.heappush(pq, (2, 'task_A'))
heapq.heappush(pq, (1, 'task_B'))
heapq.heappop(pq)   # (1, 'task_B')

# Dijkstra 模板
def dijkstra(graph, start):
    dist = {start: 0}
    pq = [(0, start)]
    while pq:
        d, node = heapq.heappop(pq)
        if d > dist.get(node, float('inf')):
            continue
        for nxt, w in graph[node]:
            nd = d + w
            if nd < dist.get(nxt, float('inf')):
                dist[nxt] = nd
                heapq.heappush(pq, (nd, nxt))
    return dist
```
---
## 三、itertools 模块（迭代器工具）
### 组合数学
```python
import itertools

# 排列 P(n, r)
list(itertools.permutations([1,2,3]))       # 全排列
list(itertools.permutations([1,2,3], 2))    # 长度 2

# 组合 C(n, r)
list(itertools.combinations([1,2,3,4], 2))  # [(1,2),(1,3),(1,4),(2,3),(2,4),(3,4)]

# 可重复组合（允许重复选同一个元素）
list(itertools.combinations_with_replacement([1,2,3], 2))

# 笛卡尔积（等价于多层 for 循环）
list(itertools.product([1,2], ['a','b']))   # [(1,'a'),(1,'b'),(2,'a'),(2,'b')]
list(itertools.product([0,1], repeat=3))    # 000-111 所有二进制串
```
### 累加与分组
```python
# 前缀和 / 前缀积
list(itertools.accumulate([1,2,3,4]))              # [1,3,6,10]
list(itertools.accumulate([1,2,3,4], initial=0))   # [0,1,3,6,10]
list(itertools.accumulate([1,2,3,4], max))         # [1,2,3,4] 前缀最大
import operator
list(itertools.accumulate([1,2,3,4], operator.mul)) # [1,2,6,24] 前缀积

# 分组（注意：仅对相邻相同的分组，一般先排序）
for key, group in itertools.groupby("aaabbc"):
    print(key, list(group))
# a ['a','a','a']
# b ['b','b']
# c ['c']
```
### 无限迭代器
```python
itertools.count(10, 2)     # 10, 12, 14, ... 无限
itertools.cycle([1,2,3])   # 1,2,3,1,2,3,... 无限循环
itertools.repeat('x', 3)   # 'x','x','x'
```
### 链接与切片
```python
list(itertools.chain([1,2], [3,4], [5]))  # [1,2,3,4,5] 拼接多个 iterable
list(itertools.chain.from_iterable([[1,2],[3,4]]))  # 展平一层

list(itertools.islice(range(10), 2, 8, 2))  # 迭代器切片 [2,4,6]

list(itertools.zip_longest([1,2,3], ['a','b'], fillvalue='-'))
# [(1,'a'),(2,'b'),(3,'-')]
```
---
## 四、functools 模块（函数工具）
### 1. `lru_cache` / `cache` —— 记忆化搜索神器
**用途**：DP、递归优化，直接把子问题结果缓存。
```python
from functools import lru_cache, cache

# 斐波那契记忆化
@lru_cache(maxsize=None)
def fib(n):
    if n < 2:
        return n
    return fib(n-1) + fib(n-2)

# Python 3.9+ 更简洁
@cache
def fib(n):
    return n if n < 2 else fib(n-1) + fib(n-2)

# 清缓存
fib.cache_clear()

# 应用：DP 状态记忆
@cache
def dp(i, j):
    if 到达边界: return 边界值
    return min(dp(i-1, j), dp(i, j-1)) + cost[i][j]
```
**注意**：参数必须可哈希（不能传 list，要传 tuple）。
---
### 2. `reduce` —— 累积
```python
from functools import reduce

reduce(lambda x, y: x + y, [1,2,3,4])       # 10
reduce(lambda x, y: x * y, [1,2,3,4], 1)    # 24（第 3 参为初始值）

# 求列表所有元素的 GCD
from math import gcd
reduce(gcd, [12, 18, 24])   # 6

# 异或和
reduce(lambda a, b: a ^ b, [1,2,3,1,2])     # 3
```
---
### 3. `cmp_to_key` —— 自定义排序
**用途**：排序规则不是"取某个 key"，而是"比较两个元素"。
```python
from functools import cmp_to_key

# 拼接最大数：[3,30,34,5,9] → "9534330"
def compare(a, b):
    if a + b > b + a: return -1
    if a + b < b + a: return 1
    return 0

nums = ["3","30","34","5","9"]
nums.sort(key=cmp_to_key(compare))
print("".join(nums))    # "9534330"
```
---
## 五、bisect 模块（二分查找）
**用途**：在有序数组中查找/插入位置，O(log n)。
```python
import bisect

arr = [1, 3, 4, 4, 4, 6, 8]

# 查找位置
bisect.bisect_left(arr, 4)   # 2  第一个 >= 4 的位置
bisect.bisect_right(arr, 4)  # 5  第一个 > 4 的位置
bisect.bisect(arr, 4)        # 5  等价于 bisect_right

# 插入（保持有序）
bisect.insort_left(arr, 5)   # 原地插入

# 应用：统计区间元素个数
def count_in_range(arr, low, high):
    return bisect.bisect_right(arr, high) - bisect.bisect_left(arr, low)

# 应用：最长递增子序列 LIS O(n log n)
def LIS(nums):
    tails = []
    for x in nums:
        i = bisect.bisect_left(tails, x)
        if i == len(tails):
            tails.append(x)
        else:
            tails[i] = x
    return len(tails)
```
---
## 六、math 模块
```python
import math

math.inf, -math.inf         # 正负无穷（也可用 float('inf')）
math.nan                    # NaN

math.gcd(12, 18)            # 6 最大公约数
math.lcm(4, 6)              # 12 最小公倍数（3.9+）
math.gcd(12, 18, 24)        # 支持多参数

math.factorial(5)           # 120 阶乘
math.comb(5, 2)             # 10  C(5,2)（3.8+）
math.perm(5, 2)             # 20  P(5,2)

math.isqrt(15)              # 3   整数平方根（避免浮点误差）
math.log(8, 2)              # 3.0
math.log2(8), math.log10(100)

math.ceil(3.2), math.floor(3.8)   # 4, 3
math.dist([0,0], [3,4])           # 5.0 欧氏距离
math.hypot(3, 4)                  # 5.0 斜边

# 位运算相关
n.bit_length()              # n 的二进制位数
n.bit_count()               # n 的 1 的个数（3.10+）
bin(5), oct(8), hex(255)    # '0b101', '0o10', '0xff'
```
---
## 七、内置数据结构常用技巧
### list 常用操作
```python
arr = [3, 1, 4, 1, 5]

# 排序
arr.sort()                          # 原地升序
arr.sort(reverse=True)              # 降序
arr.sort(key=lambda x: -x)          # 按 key 排序
sorted(arr)                         # 返回新列表
sorted(arr, key=lambda x: (x[0], -x[1]))  # 多关键字

# 切片
arr[::-1]                # 反转
arr[::2]                 # 隔一个取
arr[1:4]                 # 子数组

# 查找
arr.index(4)             # 找 4 的第一个下标
arr.count(1)             # 统计 1 出现次数

# 二维数组初始化（坑！）
dp = [[0] * n for _ in range(m)]     # ✅ 正确
dp = [[0] * n] * m                    # ❌ 陷阱！行是同一个引用
```
### dict 常用
```python
d = {'a': 1, 'b': 2}

d.get('c', 0)                # 不存在返回默认值 0
d.setdefault('c', []).append(1)   # 不存在则设默认值

# 遍历
for k, v in d.items(): pass
for k in d.keys():   pass
for v in d.values(): pass

# 排序（按值）
sorted(d.items(), key=lambda x: x[1])

# 反转字典
{v: k for k, v in d.items()}

# 合并（3.9+）
d1 | d2                      # 合并成新字典
d1 |= d2                     # 就地合并
```
### set 常用
```python
s = {1, 2, 3}
s.add(4)
s.discard(2)     # 不存在不报错（remove 会报错）

# 集合运算
a & b            # 交
a | b            # 并
a - b            # 差
a ^ b            # 对称差
a <= b           # a 是否 b 的子集
```
### frozenset —— 可哈希的集合
**用途**：作为 dict 的 key 或放入 set 里（记录"无序组合"状态）。
```python
seen = set()
seen.add(frozenset([1, 2, 3]))    # ✅
seen.add({1, 2, 3})               # ❌ 报错
```
### 字符串常用
```python
s = "hello world"

s.split()                    # ['hello', 'world']
s.split(',')                 # 按逗号
' '.join(['a','b','c'])      # 'a b c'

s.strip(), s.lstrip(), s.rstrip()
s.replace('l', 'L')
s.startswith('he'), s.endswith('ld')

s.isdigit(), s.isalpha(), s.isalnum(), s.islower()
s.lower(), s.upper()

# 字符 ↔ ASCII
ord('a')            # 97
chr(97)             # 'a'
ord('c') - ord('a') # 2 （常用于 26 字母哈希）
```
---
## 八、内置函数（Built-in）
### 高频
```python
# 数学
abs(-5), min(1,2,3), max([1,2,3])
min(students, key=lambda s: s.age)  # 支持 key
sum([1,2,3])
sum([1,2,3], 10)                    # 起始值 10 → 16
round(3.14159, 2)                   # 3.14
pow(2, 10), pow(2, 10, 1000)        # 快速幂 (mod)
divmod(17, 5)                       # (3, 2) 商和余数

# 类型
int('10', 2)                        # 2 二进制转十进制
bin(10)[2:]                         # '1010'
int(bin(10)[2:], 2)                 # 10

# 逻辑
all([1, 2, 3])                      # True 全真
any([0, 0, 1])                      # True 有真

# 迭代
enumerate(['a','b','c'])            # [(0,'a'),(1,'b'),(2,'c')]
enumerate(arr, start=1)             # 下标从 1 开始
zip([1,2], ['a','b'])               # [(1,'a'),(2,'b')]
list(zip(*matrix))                  # 矩阵转置

# 反转
reversed([1,2,3])                   # 返回迭代器
list(reversed(range(5)))            # [4,3,2,1,0]

# 排序（返回新列表）
sorted(arr)
sorted(arr, key=..., reverse=True)

# 过滤 / 映射
list(filter(lambda x: x > 0, arr))
list(map(lambda x: x*2, arr))
# 更 Pythonic 的写法（推荐）
[x for x in arr if x > 0]
[x * 2 for x in arr]
```
### range 技巧
```python
range(5)              # 0..4
range(1, 6)           # 1..5
range(0, 10, 2)       # 0,2,4,6,8
range(10, 0, -1)      # 10..1 倒序

# 常用循环
for i in range(len(arr)):           pass
for i, v in enumerate(arr):         pass  # 更 Pythonic
for i in range(len(arr) - 1, -1, -1): pass  # 倒序遍历
```
---
## 九、字符串处理
```python
# f-string 格式化
name, score = "Tom", 95.5
f"{name} scored {score:.2f}"        # 'Tom scored 95.50'
f"{42:05d}"                         # '00042' 补零
f"{42:b}", f"{42:o}", f"{42:x}"     # 二/八/十六进制
f"{0.1234:.2%}"                     # '12.34%' 百分比

# 正则
import re
re.findall(r'\d+', 'abc123def456')  # ['123', '456']
re.sub(r'\s+', ' ', text)           # 多空格合并
re.split(r'[,;]', 'a,b;c')          # ['a','b','c']
re.match(r'^\d{3}', '123abc')       # 从头匹配
```
---
## 十、综合速查表
### 按场景选择
| 场景 | 推荐工具 |
| --- | --- |
| 计数统计 | `Counter` |
| 分组、建图 | `defaultdict(list)` |
| BFS、滑动窗口 | `deque` |
| Top K / 优先队列 | `heapq` |
| 有序数组查找 | `bisect` |
| 记忆化搜索 / DP | `@cache` / `@lru_cache` |
| 全排列 / 组合 / 笛卡尔积 | `itertools` |
| 累加 / 前缀和 | `itertools.accumulate` |
| 自定义比较排序 | `functools.cmp_to_key` |
| 数学函数 | `math` |
| LRU 缓存 | `OrderedDict` |
| 状态放进 set | `frozenset` |
### 常用 import 模板（刷题一键 copy）
```python
from collections import deque, defaultdict, Counter, OrderedDict
from heapq import heappush, heappop, heapify, nlargest, nsmallest
from itertools import permutations, combinations, product, accumulate, chain
from functools import cache, lru_cache, reduce, cmp_to_key
from bisect import bisect_left, bisect_right, insort
from math import inf, gcd, lcm, comb, perm, isqrt, factorial
```
### 常见陷阱
```python
# 1. 二维数组浅拷贝
dp = [[0] * n] * m           # ❌ 所有行是同一个引用
dp = [[0] * n for _ in range(m)]  # ✅

# 2. 默认可变参数
def f(x, arr=[]):            # ❌ arr 在多次调用间共享
    arr.append(x)
    return arr

# 3. lru_cache 参数必须可哈希
@cache
def dp(state):
    # state 用 tuple 而不是 list
    pass

# 4. heapq 是小顶堆，要大顶堆取负
# 5. dict 不能在遍历时修改
# 6. 整数除法用 // 而不是 /
# 7. -1 // 2 == -1（向下取整，不是向 0 取）
```
### 复杂度对照
| 操作 | list | deque | dict/set | heapq | bisect |
| --- | --- | --- | --- | --- | --- |
| 头部增删 | O(n) | **O(1)** | — | — | — |
| 尾部增删 | O(1) | O(1) | — | — | — |
| 中间插入 | O(n) | O(n) | — | — | O(n) |
| 查找元素 | O(n) | O(n) | **O(1)** | O(n) | O(log n) |
| 最小值 | O(n) | O(n) | O(n) | **O(1)** | O(1) |
| 排序 | O(n log n) | — | — | 建堆 O(n) | — |
---
## 🎯 一句话总结
> **计数用 Counter，分组用 defaultdict，队列用 deque，堆用 heapq；DP 加 @cache，二分找 bisect，全排列组合找 itertools，比较排序 cmp_to_key。**