# Python 高频考点速查

> 全行业通用（字节/阿里/腾讯/美团/快手/百度/小厂 均高频覆盖）

---

## 一、Python 语言特性

```
解释型语言:
  ├── 源码 → 字节码(.pyc) → 虚拟机执行
  ├── CPython: 官方实现，C 语言编写
  ├── PyPy: JIT 编译，性能更高
  └── GIL: 全局解释器锁（CPython）

动态类型:
  ├── 变量无类型声明，运行时推断
  ├── duck typing: "如果它走路像鸭子，叫像鸭子 → 它就是鸭子"
  └── type hints (3.5+): 提供类型标注，不做强制检查

内存管理:
  ├── 引用计数（主要）+ 标记清扫（循环引用兜底）
  ├── gc 模块: gc.collect() 手动触发
  ├── 小对象复用: PyObject 内存池（<256 Bytes）
  └── sys.getrefcount(obj) 查看引用计数
```

---

## 二、Python 2 vs Python 3 区别

```
核心差异:
  ├── print 语句 → print() 函数
  ├── 整数除法: 3/2=1(Py2) → 1.5(Py3)
  ├── Unicode: str(bytes)+unicode → str(unicode)+bytes
  ├── range: 返回 list → 返回迭代器（性能更好）
  ├── raise: raise Exception, msg → raise Exception(msg)
  ├── except: except Exc, e → except Exc as e
  ├── metaclass: __metaclass__ → metaclass=...
  └── async/await: 3.5+ 原生支持

面试高频:
  Q: 为什么 Python 3 不向后兼容？
     └── 故意 breaking change 清理历史包袱（如 Unicode 统一）
  Q: Python 2 何时停止维护？
     └── 2020 年 1 月 1 日正式终止
```

---

## 三、数据结构与内置类型

```
可变 vs 不可变:
  ├── 可变: list, dict, set, bytearray
  └── 不可变: int, float, str, tuple, frozenset, bytes

list:
  ├── 底层: 动态数组（连续内存，超容量时 ~1.125 倍扩容）
  ├── 索引 O(1)，插入/删除 O(n)
  ├── list comprehension: [x*2 for x in range(10)]
  └── 注意: 循环中修改 list → 用切片副本 for x in lst[:]

dict:
  ├── 底层: 哈希表（3.6+ 有序！保持插入顺序）
  ├── 3.7+: 插入顺序成为语言规范
  ├── key 必须可哈希（不可变类型）
  ├── dict comprehension: {k:v for k,v in ...}
  └── 性能: O(1) 平均，哈希冲突退化 O(n)

set:
  ├── 底层: 同 dict 的 key 集合
  ├── 去重: list(set(lst))
  ├── 集合运算: & | - ^
  └── frozenset: 不可变，可作为 dict key

tuple:
  ├── 不可变但内部元素可变:
      t = ([1], 2); t[0].append(3)  # 可以！tuple 存的引用不变
  └── 单元素: (1,) ← 逗号必需

str:
  ├── 底层: 3.3+ 灵活表示（ASCII→1B, Latin→2B, 全部→4B）
  ├── 不可变，拼接用 ''.join()
  ├── 编码: .encode() / .decode()
  └── f-string (3.6+): f"value={x}"  ← 面试最爱
```

---

## 四、函数与装饰器

```
函数是一等公民:
  ├── 可赋值、传参、返回
  ├── 闭包: 函数 + 外部变量引用
  └── lambda: lambda x: x*2（单表达式）

装饰器原理:
  def decorator(func):
      @functools.wraps(func)  # 保留原函数元信息
      def wrapper(*args, **kwargs):
          # before
          result = func(*args, **kwargs)
          # after
          return result
      return wrapper

  @decorator
  def f(): pass
  # 等价: f = decorator(f)

参数装饰器:
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
  def say(): print("hi")

类装饰器:
  class CountCalls:
      def __init__(self, func):
          self.func = func
          self.count = 0
      def __call__(self, *args, **kwargs):
          self.count += 1
          return self.func(*args, **kwargs)

面试高频:
  Q: @staticmethod vs @classmethod 区别？
     ├── @staticmethod: 无 self/cls 参数，纯工具函数
     └── @classmethod: 接收 cls，可访问类属性、用于工厂方法
  Q: functools.wraps 作用？
     └── 复制 __name__, __doc__ 等元信息到 wrapper
```

---

## 五、迭代器与生成器

```
迭代器协议:
  ├── __iter__() 返回 self
  └── __next__() 返回下一个值 / raise StopIteration

可迭代对象 (Iterable):
  ├── 实现了 __iter__() 或 __getitem__()
  ├── for 循环本质: iter(obj) → 不断 next() 直到 StopIteration
  └── 常用: list, dict, str, range, file

生成器 (Generator):
  └── 函数中包含 yield 关键字 → 返回生成器对象

  def gen():
      for i in range(10):
          yield i

生成器表达式:
  g = (x*2 for x in range(10))  # 小括号 vs 列表推导式中括号

特点:
  ├── 惰性求值（按需生成，节省内存）
  ├── 协程（3.5+ async/await）
  ├── .send() 向生成器传值
  └── .throw() / .close()

面试高频:
  Q: 生成器 vs 列表性能？
     ├── 内存: 生成器 O(1) vs 列表 O(n)
     └── 速度: 列表遍历更快（预生成缓存）
  Q: yield from 的作用？
     └── 委托给另一个生成器（嵌套生成器扁平化）
```

---

## 六、面向对象

```
类定义:
  class MyClass(Base):
      class_var = 1  # 类属性（所有实例共享）

      def __init__(self, x):
          self.x = x   # 实例属性

      @property
      def x_squared(self):
          return self.x ** 2

      @x_squared.setter
      def x_squared(self, value):
          self.x = int(value ** 0.5)

MRO (方法解析顺序):
  ├── Python 2.3+: C3 线性化算法
  ├── class.mro() 查看顺序
  ├── super(): 按 MRO 调用下一个类的方法
  └── 钻石继承: super() 解决重复调用问题

魔法方法 (Magic Methods):
  ├── __str__: str(obj) / print(obj)
  ├── __repr__: 调试表示，repr(obj)
  ├── __eq__: == (默认比较 id)
  ├── __hash__: hash(obj)（与 __eq__ 同时定义）
  ├── __enter__ / __exit__: 上下文管理器 with
  ├── __getattr__: 属性查找失败时调用
  └── __call__: 使实例可调用 obj()

面试高频:
  Q: __new__ vs __init__？
     ├── __new__(cls): 创建实例（静态方法，返回实例）
     └── __init__(self): 初始化实例（有实例后调用）
     └── 典型: 单例模式在 __new__ 中控制
  Q: 浅拷贝 vs 深拷贝？
     ├── copy.copy(): 最外层拷贝，内部引用共享
     ├── copy.deepcopy(): 递归全部拷贝
     └── 列表切片 [:] = 浅拷贝
```

---

## 七、GIL 与并发编程

```
GIL (Global Interpreter Lock):
  ├── 同一时刻只有一个线程执行 Python 字节码
  ├── 原因: CPython 内存管理非线程安全
  ├── 影响: CPU 密集型多线程无加速
  └── 规避:
      ├── 多进程: multiprocessing (每个进程独立 GIL)
      ├── C 扩展: 释放 GIL（如 numpy, Cython）
      └── asyncio: 协程（单线程并发）

threading 模块:
  ├── threading.Thread(target=fn)
  ├── 锁: threading.Lock / RLock / Semaphore
  └── 事件: threading.Event（等待/通知）

multiprocessing 模块:
  ├── Process(target=fn)  # 类似 Thread API
  ├── Pool: 进程池
  ├── Queue / Pipe: 进程间通信
  └── Manager: 共享状态

concurrent.futures:
  ├── ThreadPoolExecutor / ProcessPoolExecutor
  ├── submit(fn) → Future
  └── as_completed / wait

面试高频:
  Q: GIL 会不会被去掉？
     └── 有尝试（GIL-free CPython），但因单线程性能回退太大未合入
  Q: 多线程适合什么场景？
     └── I/O 密集型（网络请求、文件读写），GIL 在 I/O 时释放
  Q: 如何选择多线程/多进程/协程？
     ├── I/O 密集型、高并发连接 → asyncio
     ├── CPU 密集型 → multiprocessing
     └── 简单 I/O 并发 → threading
```

---

## 八、asyncio 协程

```
基本概念:
  ├── event loop: 事件循环，调度协程
  ├── coroutine: async def 定义的协程函数
  ├── await: 挂起当前协程，等待另一个协程完成
  └── Future / Task: 封装异步操作的结果

async def f():
    await asyncio.sleep(1)  # 休眠但不阻塞线程
    return 42

运行:
  asyncio.run(f())                # 3.7+ 标准入口
  await asyncio.gather(f(), g())  # 并发执行多个

底层原理:
  ├── 生成器增强: async/await 基于 yield from
  ├── event loop: 注册 callback → I/O 多路复用(epoll/select)
  └── 协程切换: 在 await 点主动让出控制权

注意事项:
  ├── 不要在 async 函数中调用阻塞函数（用 asyncio.to_thread）
  ├── 异步代码中慎用同步锁（用 asyncio.Lock）
  ├── asyncio.run() 每个进程只调用一次
  └── 大量协程不影响性能（~微秒级切换）

面试高频:
  Q: 协程 vs 线程？
     ├── 协程用户态切换，线程内核态切换
     ├── 协程 ~1μs 切换，线程 ~1μs（但线程更多上下文）
     └── 协程数量可达百万，线程通常几千
  Q: await 和 yield 区别？
     └── await: 等待另一个协程，yield: 返回给调用方
```

---

## 九、深拷贝、浅拷贝与引用

```
a = [1, 2, [3, 4]]
b = a           # 引用（完全共享）
c = a.copy()    # 浅拷贝（内层 [3,4] 共享）
d = copy.deepcopy(a)  # 深拷贝（全部独立）

默认参数陷阱:
  def f(lst=[]):          # ← 默认参数在函数定义时求值！
      lst.append(1)
      return lst
  f()  # [1]
  f()  # [1, 1]  ← 同一个 list 对象！

  修复: def f(lst=None):
            if lst is None: lst = []
```

---

## 十、上下文管理器

```
with 语句:
  with open('file.txt') as f:
      content = f.read()

自定义上下文管理器:
  class MyContext:
      def __enter__(self):
          print("enter")
          return self   # 绑定到 as 后面
      def __exit__(self, exc_type, exc_val, exc_tb):
          print("exit")
          return False  # True 则压制异常

  contextlib 工具:
  from contextlib import contextmanager
  @contextmanager
  def my_context():
      print("enter")
      try:
          yield
      finally:
          print("exit")
```

---

## 十一、元类 (Metaclass)

```
元类 = 创建类的类
  ├── 类: type 的实例
  ├── type(name, bases, dict) 动态创建类
  └── 自定义元类: class Meta(type): pass

使用场景:
  ├── ORM: SQLAlchemy 模型声明
  ├── 单例: 元类控制实例创建
  └── 注册: 自动注册子类到工厂

class Singleton(type):
    _instances = {}
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class MyClass(metaclass=Singleton):
    pass
```

---

## 十二、常用标准库

```
os / sys:
  ├── os.path: 路径操作
  ├── os.environ: 环境变量
  ├── sys.argv: 命令行参数
  └── sys.path: 模块搜索路径

re 正则:
  ├── re.search / re.match / re.findall / re.sub
  ├── 编译: re.compile(pattern) 提高性能
  └── 分组: r'(\d{4})-(\d{2})-(\d{2})'

datetime:
  ├── datetime.now() / datetime.strptime / strftime
  ├── timedelta: 时间运算
  ├── timezone: 时区处理（3.9+ zoneinfo）
  └── 格式: "%Y-%m-%d %H:%M:%S"

json:
  ├── json.dumps / json.loads（含 indent 美化）
  ├── 自定义序列化: default 参数或实现 __json__ 方法
  └── 注意: json 不支持 set/bytes/datetime

collections:
  ├── namedtuple: 轻量对象
  ├── defaultdict: 默认值字典
  ├── Counter: 计数器
  ├── deque: 双端队列 O(1) 两端操作
  ├── OrderedDict: 有序字典（3.7+ 普通 dict 也有序）
  └── ChainMap: 多个字典合并视图

itertools:
  ├── chain: 扁平化多个可迭代对象
  ├── product: 笛卡尔积
  ├── permutations / combinations: 排列组合
  ├── groupby: 相邻元素分组
  └── cycle / repeat / count: 无限迭代器

functools:
  ├── partial: 偏函数（固定部分参数）
  ├── lru_cache: 缓存函数结果
  ├── reduce: 累积运算
  └── wraps: 装饰器保留元信息

logging:
  ├── 级别: DEBUG < INFO < WARNING < ERROR < CRITICAL
  ├── 组件: Logger → Handler → Formatter
  └── 配置: basicConfig / dictConfig / fileConfig
```

---

## 十三、文件 I/O 与异常

```
文件操作:
  with open('f.txt', 'r', encoding='utf-8') as f:
      content = f.read()         # 全部读取
      line = f.readline()        # 单行
      lines = f.readlines()      # 全部行（列表）
      for line in f:             # 迭代器（推荐大文件）

  r / w / a / rb / wb / r+ / w+

异常处理:
  try:
      risky()
  except ValueError as e:
      handle()
  except (TypeError, KeyError):
      pass
  else:
      no_error()   # 无异常时执行
  finally:
      cleanup()    # 始终执行

自定义异常:
  class MyError(Exception):
      def __init__(self, msg, code):
          super().__init__(msg)
          self.code = code

断言:
  assert condition, "message"  # 调试用，可被 -O 禁用
```

---

## 十四、模块与包

```
模块导入:
  import module
  from module import name
  from module import name as alias

__name__ == '__main__':
  # 直接运行该文件时执行
  # 被 import 时不执行

包结构:
  mypkg/
      __init__.py    # 包初始化（可为空）
      module_a.py
      subpkg/
          __init__.py
          module_b.py

绝对引入 vs 相对引入:
  from mypkg import module_a   # 绝对
  from . import module_a       # 相对（同级）
  from ..subpkg import module_b # 上级

搜索路径:
  ├── 当前目录
  ├── PYTHONPATH 环境变量
  └── 标准库路径 + site-packages
```

---

## 十五、Python 性能优化

```
性能分析:
  python -m cProfile script.py       # 函数级性能
  python -m profile script.py
  import timeit; timeit.timeit('...')

优化手段:
  ├── 使用内置函数（C 实现）: map, filter, sum, any, all
  ├── 列表推导式 > for 循环 > .append()
  ├── 局部变量加速: local_var = global_var（减少 LOAD_GLOBAL）
  ├── 字符串拼接: ''.join(list) > +=
  ├── 选择合适数据结构: set 查重 O(1), deque 两端操作
  ├── 预分配: [None] * n
  └── PyPy: JIT 编译加速 CPU 密集型

C 扩展:
  ├── Cython: Python-like → C 扩展
  ├── ctypes: 调用 C 库
  ├── CFFI: 更友好的 C 接口
  └── pybind11: C++ 绑定

面试高频:
  Q: Python 慢怎么办？
     ├── 先用 cProfile 找瓶颈
     ├── 算法优化 > 语言优化
     └── 最后考虑 C 扩展 / PyPy
```

---

## 十六、Python 测试

```
unittest（内置）:
  import unittest
  class TestMyFunc(unittest.TestCase):
      def setUp(self): ...    # 每个测试前
      def tearDown(self): ... # 每个测试后
      def test_case(self):
          self.assertEqual(got, want)

pytest（第三方主流）:
  ├── 更简洁: def test_case(): assert got == want
  ├── fixture: @pytest.fixture 依赖注入
  ├── parametrize: @pytest.mark.parametrize 多组参数
  ├── mock: unittest.mock / pytest-mock
  └── coverage: pytest-cov

Mock:
  from unittest.mock import Mock, patch
  @patch('module.function')
  def test(mock_fn):
      mock_fn.return_value = 42

运行:
  python -m pytest tests/
  python -m pytest -v -k "keyword"
  python -m pytest --cov=src tests/
```

---

## 十七、常见面试输出题

```
# 1. 默认参数陷阱
def f(lst=[]):
    lst.append(1)
    return lst
print(f())  # [1]
print(f())  # [1, 1]

# 2. 闭包延迟绑定
def make_funcs():
    funcs = []
    for i in range(3):
        funcs.append(lambda: i)  # 闭包绑定变量 i 的引用
    return funcs
for f in make_funcs():
    print(f())  # 2 2 2（全部是最后一个 i 值）
# 修复: funcs.append(lambda i=i: i)  # 默认参数捕获当前值

# 3. 可变对象作为默认参数
def f(a, b=[]):
    b.append(a)
    return b
print(f(1))  # [1]
print(f(2))  # [1, 2]
print(f(3))  # [1, 2, 3]

# 4. 类变量 vs 实例变量
class A:
    x = []
    def add(self, v):
        self.x.append(v)
a1, a2 = A(), A()
a1.add(1)    # a1.x = [1]
a2.add(2)    # a2.x = [1, 2]
print(A.x)   # [1, 2]  ← 类变量被修改了！

# 5. is vs ==
a = [1,2,3]
b = [1,2,3]
print(a == b)  # True（值相等）
print(a is b)  # False（不同对象）
c = a
print(a is c)  # True（同一对象）

# 6. 小整数缓存
a = 256
b = 256
print(a is b)  # True（-5~256 缓存）
c = 257
d = 257
print(c is d)  # False（大整数不缓存）

# 7. 上下文管理器异常压制
class Tr:
    def __exit__(self, *args):
        return True  # 压制异常
with Tr():
    raise ValueError("test")
print("这里仍会执行")  # 异常被压制！

# 8. 循环中修改列表
lst = [1,2,3,4,5]
for x in lst:
    if x % 2 == 0:
        lst.remove(x)
print(lst)  # [1, 3, 5] ？ 实际上 [1, 3, 4, 5] ← 迭代时修改出问题
# 修复: for x in lst[:]:
```

---

## 十八、设计模式 Python 实现

```
单例模式:
  # 1. 模块单例（最推荐）
  class Singleton: ...
  instance = Singleton()  # 模块导入时创建

  # 2. 元类单例
  class SingletonMeta(type):
      _instances = {}
      def __call__(cls, *args, **kwargs):
          if cls not in cls._instances:
              cls._instances[cls] = super().__call__(*args, **kwargs)
          return cls._instances[cls]

  # 3. 装饰器单例
  def singleton(cls):
      instances = {}
      def get(*args, **kwargs):
          if cls not in instances:
              instances[cls] = cls(*args, **kwargs)
          return instances[cls]
      return get

工厂模式:
  class Animal: ...
  class Dog(Animal): ...
  class Cat(Animal): ...
  def create_animal(t: str) -> Animal:
      return {'dog': Dog, 'cat': Cat}[t]()

观察者模式:
  class Subject:
      def __init__(self):
          self._observers = []
      def attach(self, obs): self._observers.append(obs)
      def notify(self, msg):
          for obs in self._observers:
              obs.update(msg)

策略模式:
  from abc import ABC, abstractmethod
  class Strategy(ABC):
      @abstractmethod
      def execute(self, data): ...
```

---

## 十九、Python 与其他语言对比

```
vs Java:
  ├── Python 动态类型，Java 静态类型
  ├── Python 缩进，Java 大括号
  ├── Python 所有异常可捕获，Java checked/unchecked
  └── Python 无访问控制（全靠自觉）

vs JavaScript:
  ├── Python 强类型，JS 弱类型
  ├── Python 类更好用，JS 原型链
  ├── Python 同步风格（async 也清晰），JS callback 历史
  └── Python 空格/JS 分号

vs Go:
  ├── Python 动态类型 + duck typing，Go 静态 + 接口
  ├── Python 无原生并发（GIL），Go goroutine
  ├── Python 包管理（pip/conda）比 Go module 更成熟
  └── Python 上手更快，Go 执行更快

vs Rust:
  ├── Python 运行时 GC，Rust 编译期所有权
  ├── Python 快速开发，Rust 系统编程
  └── Python 慢但写起来快，Rust 快但写起来慢
```

---

## 二十、Python 版本演进

```
Python 2.7 (2010): 最后一个 2.x 版本
Python 3.5 (2015): async/await, typing, *
Python 3.6 (2016): f-string, dict 有序, 异步生成器
Python 3.7 (2018): dataclass, breakpoint(), asyncio.run()
Python 3.8 (2019): walrus :=, f-string =, match/case
Python 3.9 (2020): str.removeprefix, typing 泛型简化
Python 3.10 (2021): match/case (structural pattern matching)
Python 3.11 (2022): 大幅提速 (~60%), except*, Tomllib
Python 3.12 (2023): f-string 增强, typing 重大改进
Python 3.13 (2024): 实验性 JIT, 无 GIL 构建选项

面试高频:
  Q: Python 3.8 walrus operator := 有什么用？
     └── 赋值表达式: if (n := len(x)) > 10: ...
  Q: Python 3.10 match/case 是什么？
     └── 模式匹配（类似 switch，但更强大）
  Q: dataclass 和普通 class 区别？
     └── 自动生成 __init__, __repr__, __eq__ 等
```