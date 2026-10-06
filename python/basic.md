## `self.data: np.ndarray`基本语法解析

```python
self.data: np.ndarray = data
#  ^      ^         ^
# 变量名  类型注解符号 类型注解   赋值
```

**等价于**：

```python
# 1. 声明一个实例属性 data
# 2. 预期它的类型应该是 np.ndarray
# 3. 将参数 data 赋值给它
self.data = data
# 但添加了类型提示：self.data 应该是 np.ndarray 类型
```

---

## 一、函数参数：默认值、`*args`、`**kwargs`、解包

```python
def __init__(self, model, **kwargs):          # **kwargs：收集所有多余的"关键字参数"成 dict
    config = Config(model, **config_kwargs)   # 调用时 ** 反过来：把 dict 展开成关键字参数

def call(self, method_name, *args):           # *args：收集多余的"位置参数"成 tuple
    method(*args)                             # 调用时 * 把 tuple/list 展开成位置参数
```

```python
def f(a, b, c): ...
f(1, 2, 3)
f(*[1, 2, 3])               # 等价
f(**{"a": 1, "b": 2, "c": 3})   # 等价
```

**星号解包赋值**也很常见（`ModelRunner.read_shm`）：

```python
method_name, *args = pickle.loads(...)   # 第一个给 method_name，剩下的全进 args(list)
a, b = b, a                               # 元组解包，交换变量
x1, x2 = torch.chunk(x, 2, dim=-1)        # 函数返回多个值，其实就是返回一个 tuple
```

**链式赋值**：`prefill_throughput = decode_throughput = 0.` 两个名字指向同一个值（对数字没问题，对列表要小心，见第十三节）。

## 二、类型注解

```python
def add_request(self, prompt: str | list[int], sampling_params: SamplingParams): ...
def schedule(self) -> tuple[list[Sequence], bool]: ...
self.waiting: deque[Sequence] = deque()
```

- `str | list[int]`：联合类型，"字符串或整数列表"（Python 3.10+ 写法，旧写法是 `Union[str, list[int]]`）。
- `list[int]`、`dict[int, int]`、`tuple[list[Sequence], bool]`：泛型标注，3.9+ 可直接用内置类型。
- `-> list[int]`：返回值标注。
- **关键点：标注在运行时完全不强制**。写错了（比如 `generate` 标 `list[str]` 实际返回 `list[dict]`）程序照样跑，只是给人和 IDE/类型检查器看的。

## 三、类的基础

```python
class Sequence:
    block_size = 256              # 类属性：所有实例共享
    counter = count()

    def __init__(self, token_ids):
        self.seq_id = next(Sequence.counter)   # 实例属性：每个对象自己一份
```

- `self` 就是"当前对象"，方法第一个参数必须写，调用时不用传。
- **类属性 vs 实例属性**：`Sequence.block_size = config.kvcache_block_size`（`LLMEngine.__init__`）是修改**类属性**，之后所有实例读到的都是新值。如果实例上没有同名属性，`self.block_size` 会回退去找类属性。
- `__init__` 是构造函数，`Sequence(...)` 时自动调用。

### 三种方法

```python
class BlockManager:
    @classmethod
    def compute_hash(cls, token_ids, prefix=-1): ...   # 第一个参数是类 cls，不是实例

    @staticmethod
    def helper(x): ...                                  # 没有 self/cls，就是放在类里的普通函数

    def can_append(self, seq): ...                      # 普通方法，操作实例
```

`compute_hash` 不需要访问实例状态，所以用 `@classmethod`，调用写 `self.compute_hash(...)` 或 `BlockManager.compute_hash(...)` 都行。

### `@property`：把方法伪装成属性

```python
@property
def num_blocks(self):
    return (self.num_tokens + self.block_size - 1) // self.block_size

seq.num_blocks        # 注意：不加括号，像读属性一样；每次读取都会重新计算
```

用途是"这个值由别的字段推导出来，不单独存"。`Sequence` 里的 `num_blocks`、`is_finished`、`completion_token_ids`、`last_block_num_tokens` 都是这样。好处是不用手动同步，`append_token` 改了 `num_tokens`，`num_blocks` 自然跟着变。

## 四、装饰器 `@`

### 1. 本质：函数套函数

`@` 只是语法糖：

```python
@deco
def f(): ...

# 完全等价于
def f(): ...
f = deco(f)
```

装饰器就是一个**接收函数、返回新函数**的函数。手写一个计时装饰器：

```python
from time import perf_counter

def timer(func):
    def wrapper(*args, **kwargs):          # 用 *args/**kwargs 接住任意参数
        t = perf_counter()
        result = func(*args, **kwargs)     # 调用原函数
        print(f"{func.__name__} took {perf_counter() - t:.3f}s")
        return result
    return wrapper

@timer
def run(n): ...
```

### 2. 带参数的装饰器

```python
@lru_cache(1)
def get_rope(...): ...
# 等价于：get_rope = lru_cache(1)(get_rope)
```

`lru_cache(1)` 先执行，返回一个真正的装饰器，再去装饰函数。多一层嵌套而已。

### 3. 叠加

```python
@a
@b
def f(): ...
# 等价于 f = a(b(f))，离函数最近的先生效
```

### 4. 源码里出现的装饰器

| 装饰器                       | 出现位置                            | 作用                                                                   |
| ------------------------- | ------------------------------- | -------------------------------------------------------------------- |
| `@property`               | `Sequence`                      | 方法变属性                                                                |
| `@classmethod`            | `BlockManager.compute_hash`     | 方法绑定到类                                                               |
| `@lru_cache(1)`           | `get_rope`                      | 缓存函数结果：同样参数再调用直接返回上次的对象。要求参数可哈希（int/float/str/tuple 可以，list/dict 不行） |
| `@torch.inference_mode()` | `run_model`、`capture_cudagraph` | 关闭梯度和 autograd 记录，省显存提速                                              |
| `@torch.compile`          | `RotaryEmbedding.forward`       | 把函数编译成融合后的 GPU kernel                                                |
| `@triton.jit`             | `store_kvcache_kernel`          | 这个函数**不再按普通 Python 执行**，而是被编译成 GPU kernel                            |
| `@dataclass`              | `Config`（推断）                    | 自动生成 `__init__`、`__repr__` 等                                         |

## 五、魔法方法（dunder）

名字前后各两个下划线的方法，由 Python 在特定语法下**自动调用**。你不会直接写 `seq.__len__()`，而是用语法触发它。

|你写的|Python 实际调用|
|---|---|
|`len(seq)`|`seq.__len__()`|
|`seq[3]`、`seq[a:b]`|`seq.__getitem__(3)`、`seq.__getitem__(slice(a, b))`|
|`Sequence(...)`|`__init__`|
|`obj(x)`|`obj.__call__(x)`|
|`print(obj)`|`__str__` / `__repr__`|
|`pickle.dumps(obj)`|`__getstate__`|
|`pickle.loads(...)`|`__setstate__`|

### `__len__` 和 `__getitem__`

```python
class Sequence:
    def __len__(self):
        return self.num_tokens
    def __getitem__(self, key):
        return self.token_ids[key]
```

- 定义了 `__len__`，`len(seq)` 才能用，`ModelRunner` 里的 `len(seq) - 1` 就是这样。
- 定义了 `__getitem__`，`seq[start:end]` 才能用。切片时 `key` 收到的是一个 `slice` 对象，这里直接转交给内部 list，所以切片自动可用。
- 这就是 Python 的"鸭子类型"：对象不需要继承什么，只要实现了对应的魔法方法，就能被当作序列使用。

### `__call__`：让对象可以像函数一样调用

```python
self.sampler(logits, temperatures)    # Sampler 是 nn.Module
```

`nn.Module` 实现了 `__call__`，它内部做了一些钩子处理，然后调用你写的 `forward`。所以**写模型时定义 `forward`，用模型时直接 `model(x)`**，不要手动调 `model.forward(x)`。

### 一个有意思的组合：`kernel[(N,)](...)`

```python
store_kvcache_kernel[(N,)](key, key.stride(0), ...)
```

这是两个魔法方法连用：`[(N,)]` 触发 `__getitem__`（把 grid 大小存起来，返回一个启动器对象），后面的 `(...)` 触发 `__call__`（真正启动 kernel）。

### `__getstate__` / `__setstate__`

控制对象怎么被 `pickle` 序列化：`getstate` 决定存什么，`setstate` 决定怎么恢复。`Sequence` 里用它在 decode 阶段少传 `token_ids`。要点：恢复时**不会调用 `__init__`**，所以恢复出来的对象只有 `setstate` 里赋值过的字段。

## 六、`dataclass` 与反射

```python
from dataclasses import fields
config_fields = {field.name for field in fields(Config)}
```

`@dataclass` 装饰的类，只需要写字段声明：

```python
@dataclass
class Config:
    model: str
    max_num_seqs: int = 512
    enforce_eager: bool = False
```

Python 自动生成 `__init__`、`__repr__`、`__eq__`。`fields(Config)` 返回所有字段的描述，可以拿来做"过滤合法参数"。

**反射**：运行时按字符串操作对象。

```python
getattr(self, method_name, None)   # 按名字取属性/方法，取不到返回 None
hasattr(module, "k_cache")         # 有没有这个属性
isinstance(prompt, str)            # 类型判断
setattr(obj, "x", 1)               # 按名字设置
```

`ModelRunner.call` 用 `getattr(self, method_name)` 实现了"RPC"：主进程发来字符串 `"run"`，worker 就去调自己的 `self.run`。

## 七、推导式与生成器表达式

```python
config_fields = {field.name for field in fields(Config)}                    # 集合推导式
config_kwargs = {k: v for k, v in kwargs.items() if k in config_fields}     # 字典推导式
temperatures  = [seq.temperature for seq in seqs]                           # 列表推导式
outputs = [(seq.seq_id, seq.completion_token_ids) for seq in seqs if seq.is_finished]  # 带过滤
```

统一格式是 `[表达式 for 变量 in 可迭代 if 条件]`。把 `[]` 换成 `{}` 就是集合/字典，换成 `()` 就是**生成器表达式**（惰性求值，不先造出整个列表）：

```python
num_tokens = sum(seq.num_scheduled_tokens for seq in seqs)             # 生成器，省内存
graph = self.graphs[next(x for x in self.graph_bs if x >= bs)]         # 取第一个满足条件的元素
```

`next(生成器)` 取第一个值。上面这句的意思是"在 `[1,2,4,8,16,...]` 里找第一个 ≥ bs 的桶"。

## 八、真值判断与短路

Python 里很多东西可以直接当条件：

```python
if not seq.block_table:     # 空列表、空字符串、0、None、空 dict 都是 False
while self.waiting and len(scheduled_seqs) < self.max_num_seqs:   # waiting 非空才继续
```

- `and` / `or` **短路**：`a and b` 里如果 `a` 为假就不再算 `b`。
- **`bool` 是 `int` 的子类**，`True == 1`、`False == 0`。`BlockManager.can_append` 里：

```python
return len(self.free_block_ids) >= (len(seq) % self.block_size == 1)
# 右边是 True/False，当 1/0 用：需要新块 → 至少 1 个空闲块；否则 ≥ 0 恒成立
```

- 三元表达式：`a if 条件 else b`。例：

```python
num_tokens = sum(...) if is_prefill else -len(seqs)
```

## 九、循环特性：`while...else`、`zip`、`reversed`

### `while ... else` / `for ... else`

**循环没有被 `break` 打断，正常结束时执行 `else`**：

```python
while not self.block_manager.can_append(seq):
    if self.running:
        self.preempt(self.running.pop())
    else:
        self.preempt(seq)
        break                    # 走了 break → 不执行 else
else:
    seq.num_scheduled_tokens = 1 # 条件变为 False 正常退出 → 执行 else
```

### 常用迭代工具

```python
for prompt, sp in zip(prompts, sampling_params):   # 并行遍历多个序列，以最短的为准
for block_id in reversed(seq.block_table):         # 倒序
for i in range(start, end):                        # 左闭右开 [start, end)
sorted(outputs.keys())                             # 返回排好序的新 list
for seq_id, token_ids in output:                   # 遍历时直接解包元组
```

## 十、切片与负索引

```python
a[2:5]     # 下标 2,3,4（左闭右开）
a[:3]      # 前 3 个
a[3:]      # 从第 3 个到末尾
a[-1]      # 最后一个
a[-2]      # 倒数第二个
a[start - 1]
```

源码里的例子：

```python
seq.block_table[-1]              # 最后一个物理块
cu_seqlens_q[-1]                 # 累计长度的最后一项，即总 token 数
self.shm.buf[4:n+4]              # 共享内存的第 4 到 n+4 字节
graph_vars["input_ids"][:bs]     # 取前 bs 个（padding 之前的部分）
```

**切片的拷贝语义因类型而异：**

- Python `list` 切片：得到一个**新的拷贝**。
- PyTorch `tensor` 切片：得到**视图**，和原张量共享内存。这正是 CUDA Graph 能工作的前提：`slot_mapping[:bs]` 和底层大张量同一块地址，改底层数据，graph 读到的就变了。

## 十一、容器：`list`、`dict`、`set`、`deque`

```python
d = {}
d.get(h, -1)           # 取不到返回 -1，不会抛 KeyError（对比 d[h] 会报错）
del d[h]               # 删除键
h in d                 # 判断键是否存在
```

|类型|特点|源码里的用法|
|---|---|---|
|`list`|有序，尾部追加快；头部操作慢|`block_table`、`token_ids`|
|`dict`|哈希表，查找 O(1)|`hash_to_block_id`|
|`set`|无序去重，`in` 判断 O(1)|`used_block_ids`、`config_fields`|
|`deque`|双端队列，两端增删 O(1)|`waiting`、`running`、`free_block_ids`|

**为什么调度器用 `deque`**：`list.pop(0)` 要把后面所有元素前移，O(n)；`deque.popleft()` 是 O(1)。

```python
dq.append(x)         dq.appendleft(x)
dq.pop()             dq.popleft()
dq.extendleft(xs)    # 注意：会把 xs 逐个左插，结果顺序是反的 → 源码里先 reversed(...) 抵消
dq.remove(x)         # 按值删除，O(n)
```

`list.append(x)` 追加一个元素，`list.extend(xs)` 把可迭代对象的元素逐个追加：

```python
input_ids.extend(seq[start:end])    # 把一段 token 展开加进去
slot_mapping.extend(range(a, b))    # range 也可以直接 extend
```

列表乘法：`[-1] * n` 得到 n 个 -1，`[0] * seq_len` 得到长度 seq_len 的零列表。`seq.block_table + [-1] * k` 是列表拼接。

## 十二、枚举与 `itertools`

```python
class SequenceStatus(Enum):
    WAITING = auto()      # auto() 自动分配值 1, 2, 3
    RUNNING = auto()
    FINISHED = auto()

seq.status = SequenceStatus.RUNNING
seq.status == SequenceStatus.FINISHED
```

枚举比直接写字符串 `"running"` 安全：拼写错了会立刻报错，IDE 也能补全。

```python
counter = count()      # 无限迭代器：0, 1, 2, 3, ...
next(counter)          # 取下一个值，每调用一次递增
```

这就是 `seq_id` 的来源。

## 十三、引用语义与可变默认参数

Python 变量保存的是对象的**引用**，赋值不会拷贝：

```python
a = [1, 2]
b = a
b.append(3)      # a 也变成 [1, 2, 3]，因为是同一个 list

from copy import copy
c = copy(a)      # 浅拷贝，得到新 list
```

所以 `Sequence.__init__` 里写 `self.token_ids = copy(token_ids)`，避免用户传进来的 list 之后被外部改动，或者反过来被自己改动。

两个常见陷阱，源码里都碰到了：

1. **默认参数只在定义时创建一次**：

```python
def __init__(self, token_ids, sampling_params = SamplingParams()):
```

这个 `SamplingParams()` 对象被所有不传参的调用共享。只读时没问题；如果有人修改它，所有调用都受影响。**所以别用 `[]`、`{}` 做默认值**，常见写法是默认 `None`，函数里再创建。

2. **`[x] * n` 里 n 个元素是同一个对象**：

```python
sampling_params = [sampling_params] * len(prompts)   # n 个引用指向同一个对象
```

这里 `SamplingParams` 只读，没问题；但 `[[]] * 3` 得到的是三个指向同一个空列表的引用，改一个全变。

## 十四、`with` 上下文管理器

```python
with torch.cuda.graph(graph, self.graph_pool):
    outputs[:bs] = self.model(input_ids[:bs], positions[:bs])
```

`with` 保证"进入时做准备、离开时做收尾"，即使中间抛异常也会收尾。背后是 `__enter__` / `__exit__` 两个魔法方法。这里：进入时开始录制 CUDA 操作，离开时结束录制。常见的还有 `with open(...) as f:`（自动关文件）。

`@torch.inference_mode()` 既能当装饰器，也能写成 `with torch.inference_mode():`，同一个东西两种用法。

## 十五、`assert`、`atexit`

```python
assert config.num_kvcache_blocks > 0
assert key.stride(-1) == 1
assert scheduled_seqs
```

`assert 条件`：条件为假就抛 `AssertionError` 终止程序，用来写"这里不应该出现的情况"。`assert scheduled_seqs` 等价于"非空"。可以加说明：`assert x > 0, "x must be positive"`。

```python
atexit.register(self.exit)
```

注册一个"解释器退出时自动调用"的函数，用来兜底清理子进程和共享内存。

## 十六、模块与导入

```python
import torch.multiprocessing as mp           # 导入模块并起别名
from nanovllm.config import Config           # 从模块里导入某个名字
from multiprocessing.synchronize import Event
```

`nanovllm.engine.scheduler` 对应目录结构 `nanovllm/engine/scheduler.py`，每个目录里要有 `__init__.py`（可以是空文件）才被当作包。`from A import B` 之后直接用 `B`，不用再写 `A.B`。

## 十七、多进程相关（vLLM 特有且重要）

```python
ctx = mp.get_context("spawn")
process = ctx.Process(target=ModelRunner, args=(config, i, event))
process.start()
...
p.join()
```

- **为什么多进程而不是多线程**：CPython 有 **GIL**（全局解释器锁），同一时刻只有一个线程执行 Python 字节码，多线程跑不满多个 CPU 核心。多进程各有各的解释器，没有这个限制；而且一张 GPU 对应一个进程，是推理框架的惯例。
- **`spawn` vs `fork`**：`fork` 复制父进程的内存状态，如果父进程已经初始化过 CUDA，子进程再用会出问题；`spawn` 启动一个全新的解释器，重新 import，干净。代价是传给子进程的参数必须能被 pickle 序列化。
- `Process(target=可调用对象, args=元组)`：子进程启动后执行 `target(*args)`。类也是可调用的（调用类就是构造实例），所以 `target=ModelRunner` 相当于"在子进程里 `ModelRunner(config, i, event)`"。
- `Event`：进程间的信号灯，`set()` 置位、`wait()` 阻塞等待、`clear()` 复位。
- `SharedMemory`：多个进程共同读写的一段内存，比通过管道传数据更快。
- **`pickle`**：把 Python 对象转成字节串（`dumps`）和还原（`loads`）。进程之间传对象都要靠它。能被 pickle 的东西有限制，所以第五节的 `__getstate__` 才有用。

## 十八、字节与整数转换

```python
n.to_bytes(4, "little")                  # 整数 → 4 字节，小端序
int.from_bytes(self.shm.buf[0:4], "little")   # 4 字节 → 整数
```

这是 `ModelRunner` 在共享内存开头存"数据长度"用的：前 4 字节存长度 n，后面 n 字节存 pickle 数据，读的时候先读长度再读内容。**小端序（little）**指低位字节在前，x86/ARM 的主流约定，读写两边一致即可。

## 十九、PyTorch 里常见的 Python 习惯

```python
x.shape / x.size(0) / x.size(-1)        # 形状；size(0) 是第 0 维大小
x.stride(1)                              # 第 1 维的步长（跳到下一个元素要跨多少个元素）
x[idx]                                   # 用整数张量做索引（gather），cos_sin_cache[positions]
x.unsqueeze(1)                           # 在第 1 维插入大小为 1 的维度
x.tolist()                               # 张量 → 普通 Python list
x.zero_() / x.fill_(-1) / x.unsqueeze_(1)  # 带下划线结尾 = 原地修改，不返回新张量
torch.tensor(list, pin_memory=True).cuda(non_blocking=True)
```

**下划线结尾约定**很重要：`x.fill_(-1)` 原地改；`x.fill(-1)` 这种不带下划线的一般是返回新张量。读 PyTorch 代码时看到下划线，就知道这行改了原对象。

## 二十、速查：特性在源码里的位置

|特性|出现位置|
|---|---|
|`**kwargs` 过滤 + `fields`|`LLMEngine.__init__`|
|`str \| list[int]`、泛型标注|`add_request`、`Scheduler` 字段|
|`@property`|`Sequence.num_blocks` 等|
|`@classmethod`|`BlockManager.compute_hash`|
|`@lru_cache`|`get_rope`|
|`__len__` / `__getitem__`|`Sequence`|
|`__getstate__` / `__setstate__`|`Sequence`（配合 pickle）|
|`__call__`（经 `nn.Module`）|`self.sampler(...)`、`self.model(...)`|
|`kernel[grid](...)`|`store_kvcache`（`__getitem__` + `__call__`）|
|`while ... else`|`Scheduler.schedule` 的 decode 阶段|
|`getattr` 反射|`ModelRunner.call`|
|`*args` 解包|`read_shm`、`call`|
|生成器 + `next`|`run_model` 选 graph 桶|
|`bool` 当整数|`BlockManager.can_append`|
|`deque`|`Scheduler.waiting/running`、`free_block_ids`|
|`Enum` + `auto()`|`SequenceStatus`|
|`itertools.count`|`Sequence.counter`|
|`with`|`torch.cuda.graph`|
|`atexit`、`spawn`、`Event`、`SharedMemory`|`LLMEngine`、`ModelRunner`|
|视图切片|`graph_vars[...][:bs]`|

## 建议的学习路径

你现在是"看懂别人代码"的需求，不需要从头系统学。推荐顺序：

1. 先把上面第三、四、五节（类、装饰器、魔法方法）吃透，这三块占了读源码困难的大头。
2. 然后第十三节（引用语义），这是写 Python 最容易踩坑的地方。
3. 其余的遇到再查。

可以试着做一个小练习巩固：自己写一个 `MiniSequence` 类，实现 `__len__`、`__getitem__`、一个 `@property`，再写一个计时装饰器，然后用 `pickle.dumps/loads` 配合 `__getstate__` 往返一次。写完这些，前面读过的 nano-vllm 代码里的 Python 部分基本就没有盲区了。需要的话，我可以给你出一组带答案的小练习。