# PyTorch 基础 API 梳理：面向读推理引擎源码（重点：权重加载）

> 目标读者：已有 C++ 基础、没有系统学过 Python/PyTorch，要读 nano-vllm / vLLM 源码的人。 约定：标注 **【引擎】** 的 API 在 nano-vllm 源码里出现过；其余是理解它们所需的背景。 代码示例以 CPU 可运行为主，没有写明"运行结果"的地方请自己跑一遍确认。

---

## 0. 全文地图

|章节|内容|读源码时对应|
|---|---|---|
|1|Tensor 的创建与属性|到处都是|
|2|形状操作（view / split / chunk / narrow …）|`Attention`、`Linear`|
|3|索引、切片、赋值|`run_model`、`prepare_*`|
|4|原地操作（带下划线的方法）|`RMSNorm`、`Sampler`|
|5|常用数学与归约算子|`RMSNorm`、`SiluAndMul`、`Sampler`|
|6|内存、视图、stride、pin_memory|KV cache、CUDA Graph|
|7|`nn.Module` / `Parameter` / buffer|模型定义|
|**8**|**权重加载（本文重点）**|`loader.py`、`weight_loader`|
|9|设备与 dtype|`ModelRunner.__init__`|
|10|CUDA 相关|显存统计、CUDA Graph|
|11|推理模式|`@torch.inference_mode()`|
|12|`torch.compile`|`RMSNorm`、`RoPE`、`Sampler`|
|13|`torch.distributed`|`RowParallelLinear`、`ParallelLMHead`|
|14|自测练习||
|15|速查：引擎里出现的 API 出处||

---

## 1. Tensor 的创建与属性

### 1.1 创建

|API|作用|备注|
|---|---|---|
|`torch.tensor(data, dtype=, device=)`|从 Python 列表/标量**拷贝**出张量|【引擎】`prepare_*` 里把 Python 列表变张量|
|`torch.empty(*shape)`|只分配内存，**不初始化**，内容是垃圾值|【引擎】创建参数壳、KV cache|
|`torch.zeros / ones(*shape)`|全 0 / 全 1|【引擎】`capture_cudagraph` 的静态缓冲区、RMSNorm 的 weight 初值|
|`torch.arange(start, end, step)`|等差序列|【引擎】RoPE 的位置与频率|
|`torch.empty_like(x)` / `zeros_like(x)`|与 x 同形状、同 dtype、同设备|【引擎】`Sampler`、`ParallelLMHead`|
|`torch.full(shape, value)`|填充常数||
|`torch.randn(*shape)`|标准正态随机|自测时造数据|

**`empty` 的陷阱（权重加载的核心背景）**：`torch.empty` 返回的内容是内存里原有的垃圾值，可能是 0，也可能是任意数、甚至 NaN。模型参数在构造时就是这样创建的，所以**没被加载的参数不会报错，只会让输出变成乱码**（见 8.9 的检查方法）。

### 1.2 常用属性

```python
x = torch.randn(2, 3, 4)
x.shape          # torch.Size([2, 3, 4])，可当元组用
x.size()         # 同上
x.size(0)        # 第 0 维大小 = 2；x.size(-1) 是最后一维
x.ndim           # 维数 = 3
x.numel()        # 元素总数 = 24
x.dtype          # torch.float32
x.device         # device(type='cpu')
x.stride()       # 各维步长（见第 6 节）
x.is_contiguous()
x.requires_grad  # 是否参与自动求导
```

`dtype.itemsize`（较新版本的 PyTorch 才有）给出每个元素的字节数，【引擎】`allocate_kv_cache` 用它算块大小：`torch.bfloat16.itemsize == 2`。

### 1.3 dtype 与设备转换

```python
x.float()            # → float32
x.to(torch.bfloat16) # → 指定 dtype
x.to("cuda")         # → 搬到 GPU
x.cuda()             # 同上（简写）
x.cpu()
x.tolist()           # → Python 列表（GPU 张量会触发一次 GPU→CPU 同步）
x.item()             # 单元素张量 → Python 数
```

**`.to()` 的行为**：如果 dtype 和设备都没变，返回的是**同一个张量对象**（不拷贝）。【引擎】`RMSNorm` 里的 `x.float()` 在输入已是 fp32 时就是这种情形，这是它原地操作 `mul_` 在 fp32 输入下会改掉调用者数据的原因。

常见 dtype：`torch.float32 / float16 / bfloat16 / int32 / int64 / bool`。bf16 与 fp16 都是 2 字节；bf16 指数位与 fp32 相同（动态范围大、精度低），推理中更常用。

---

## 2. 形状操作

### 2.1 view / reshape / flatten

```python
x = torch.arange(12)          # [12]
x.view(3, 4)                  # → [3, 4]，共享内存（视图）
x.view(-1, 4)                 # -1 表示"自动推断这一维"→ [3, 4]
x.reshape(3, 4)               # 能 view 就 view，不能就拷贝
y = torch.randn(6, 4, 2)
y.flatten(1, -1)              # 把第 1 维到最后一维合并 → [6, 8]
```

【引擎】`q.view(-1, self.num_heads, self.head_dim)` 把 `[N, heads*dim]` 拆成 `[N, heads, dim]`；`o.flatten(1, -1)` 反过来合并。

| `view`    | `reshape`    |       |
| --------- | ------------ | ----- |
| 内存是否连续的要求 | 要求兼容（不兼容会报错） | 无要求   |
| 是否可能拷贝    | 永不拷贝         | 必要时拷贝 |

### 2.2 转置与换轴

```python
x = torch.randn(2, 3, 4)
x.transpose(0, 1)             # 交换两个维 → [3, 2, 4]，视图
x.permute(2, 0, 1)            # 任意重排 → [4, 2, 3]，视图
x.T                           # 仅二维：转置
x.contiguous()                # 若不连续，拷贝成连续布局
```

转置后张量**不再连续**，随后直接 `view` 会报错，要先 `.contiguous()` 或用 `reshape`。【引擎】`x[last_indices].contiguous()`：整数索引的结果本来就是新张量，这里 `.contiguous()` 是保证后续 kernel 拿到连续内存。

### 2.3 增删长度为 1 的维

```python
x = torch.randn(3, 4)
x.unsqueeze(0)    # → [1, 3, 4]；unsqueeze(1) → [3, 1, 4]
x.unsqueeze(-1)   # → [3, 4, 1]
y = torch.randn(1, 3, 1)
y.squeeze()       # 去掉所有长度为 1 的维 → [3]
```

【引擎】`temperatures.unsqueeze(dim=1)`：`[num_seqs] → [num_seqs, 1]`，用来广播除到 `[num_seqs, vocab]` 的 logits 上；`q.unsqueeze(1)` 给 decode 补出 `seqlen=1` 这一维。

### 2.4 拆分与拼接

|API|作用|返回|
|---|---|---|
|`x.split([a, b, c], dim)`|按**指定长度**切|视图的元组|
|`x.split(size, dim)`|按**固定大小**切（最后一块可更小）|视图的元组|
|`x.chunk(n, dim)`|切成 **n 块**（整除时等大）|视图的元组|
|`x.narrow(dim, start, length)`|取一段连续切片|视图|
|`torch.cat([a, b], dim)`|沿**已有维**拼接|新张量（拷贝）|
|`torch.stack([a, b], dim)`|沿**新维**堆叠|新张量（拷贝）|

```python
x = torch.arange(24).view(4, 6)
x.split([2, 4], dim=1)        # 列：2 + 4
x.chunk(2, dim=0)             # 行：切两块，每块 [2, 6]
x.narrow(1, 2, 3)             # 第 1 维从下标 2 起取 3 个 → [4, 3]，即列 2..4
```

**`chunk` 与 `split` 的区别**：`chunk(n)` 的参数是**块数**，`split(size)` 的参数是**块大小**；元素不能整除时，`chunk` 返回的块数可能少于 n，这是引擎在构造时先 `divide()`（带 assert 整除）的原因。

【引擎】`SiluAndMul` 里 `x.chunk(2, -1)` 把门控部分和上投影部分对半切；`qkv.split([q_size, kv_size, kv_size], -1)` 切 q/k/v。

### 2.5 广播（broadcast）

形状从**最后一维向前**对齐，每一维要么相等，要么其中一个是 1（或缺失），则小的那个被"虚拟复制"。

```python
a = torch.randn(4, 1000)      # [num_seqs, vocab]
t = torch.randn(4, 1)         # [num_seqs, 1]
a / t                         # → [4, 1000]，t 沿最后一维广播
```

广播不会真的拷贝数据，但会**悄悄掩盖形状错误**，8.7 会讲它在 `copy_` 里的危害。

---

## 3. 索引、切片、赋值

### 3.1 基本切片：得到视图

```python
x = torch.arange(20).view(4, 5)
x[1]            # 第 1 行 → [5]
x[:, 2]         # 第 2 列 → [4]
x[1:3]          # 第 1、2 行 → [2, 5]
x[:2, 1:4]      # 前两行的第 1..3 列
x[-1]           # 最后一行
```

**基本切片返回视图，与原张量共享内存**：修改切片会改动原张量。【引擎】`graph_vars["input_ids"][:bs]` 是 CUDA Graph 静态缓冲区的前 bs 个，视图与底层缓冲区同址，这就是改缓冲区内容后 graph 能读到新数据的原因。

### 3.2 整数张量索引（gather）：得到拷贝

```python
table = torch.randn(10, 4)        # 比如 cos/sin 表 [max_pos, dim]
pos = torch.tensor([3, 0, 7])
table[pos]                        # → [3, 4]，新张量（拷贝）
```

【引擎】`cos_sin_cache[positions]`、`x[last_indices]`、`F.embedding` 的本质都是这种"按下标取行"。

**区分两种索引**：整数/布尔张量索引（高级索引）**拷贝**；切片 `a:b`（基本索引）**视图**。

### 3.3 布尔掩码

```python
x = torch.tensor([5, 12, 8, 20])
mask = (x >= 8) & (x < 15)        # [False, True, True, False]，& 是逐元素"与"，不是 and
x[mask]                           # → tensor([12, 8])
mask * (x - 8)                    # bool 参与乘法时当 0/1 用
```

【引擎】`VocabParallelEmbedding` 用 `mask * (x - start)` 把不属于本 rank 的 token id 置 0，防止越界查表。

### 3.4 切片赋值

```python
buf = torch.zeros(8, 3)
buf[:4] = torch.ones(4, 3)        # 对视图赋值，写进了 buf
buf.fill_(-1)                     # 整体填充
buf[:2, :2] = other[:, :2]
```

【引擎】`run_model` 里 `graph_vars["slot_mapping"][:bs] = context.slot_mapping`：把本步数据写进静态缓冲区的前 bs 个。

---

## 4. 原地操作（in-place）

**约定**：方法名以 `_` 结尾 = **原地修改调用者本身**，不创建新张量。

|原地版|非原地版|说明|
|---|---|---|
|`x.add_(y)`|`x + y` / `x.add(y)`|加|
|`x.mul_(y)`|`x * y`|乘|
|`x.div_(y)`|`x / y`|除|
|`x.fill_(v)`|—|填充|
|`x.zero_()`|—|清零|
|`x.copy_(src)`|—|**把 src 的值拷进 x**（权重加载的核心，见第 8 节）|
|`x.clamp_min_(v)`|`x.clamp(min=v)`|下限截断|
|`x.unsqueeze_(d)`|`x.unsqueeze(d)`|原地插维|
|`x.exponential_(rate)`|—|用指数分布随机数原地填充|

**注意两点**：

1. 原地操作让"别名"也看到改动。`y = x.float()` 在 x 已是 fp32 时返回**同一个对象**，`y.mul_(2)` 就会改掉 x。
2. 在需要梯度的叶子张量（`requires_grad=True` 的参数）上做原地操作，PyTorch 会报错：`a leaf Variable that requires grad is being used in an in-place operation`。这是 8.3 为什么要用 `param.data` 的原因。

---

## 5. 常用数学与归约算子

### 5.1 矩阵运算

```python
import torch.nn.functional as F
F.linear(x, W, b)       # = x @ W.T + b，W 形状 [out, in]
x @ W                   # 矩阵乘，= torch.matmul
torch.einsum("i,j->ij", t, inv_freq)   # 外积（RoPE 里的角度表）
```

**`F.linear` 的权重形状是 `[out_features, in_features]`**，不是 `[in, out]`，这决定了 TP 里"列并行切 dim 0（输出），行并行切 dim 1（输入）"。

### 5.2 逐元素与归约

```python
x.pow(2)                          # 平方
x.mean(dim=-1, keepdim=True)      # 沿最后一维求均值，保留该维（长度 1）以便广播
torch.rsqrt(v)                    # 1/sqrt(v)
torch.softmax(x, dim=-1)
F.silu(x)                         # x * sigmoid(x)
x.argmax(dim=-1)                  # 最大值的下标
x.cos(); x.sin()
```

`keepdim=True` 的意义：`x` 是 `[N, hidden]`，`mean(dim=-1)` 得 `[N]`，不能直接和 `x` 相除；`keepdim=True` 得 `[N, 1]`，可以广播。【引擎】`RMSNorm` 里 `var = x.pow(2).mean(dim=-1, keepdim=True)`。

### 5.3 RMSNorm 的五行（读源码时的参照）

```python
x = x.float()                                   # 升到 fp32 算
var = x.pow(2).mean(dim=-1, keepdim=True)       # 均方
x = x * torch.rsqrt(var + eps)                  # 缩放
x = x.to(orig_dtype) * weight                   # 转回，乘可学习的 weight
```

---

## 6. 内存模型：存储、视图、stride、pin_memory

### 6.1 存储与视图

一个张量 = **一块一维存储（storage）+ 形状 + stride + 偏移**。多个张量可以共享同一块存储，这就是"视图"。

```python
a = torch.arange(12)
b = a.view(3, 4)
b[0, 0] = 100
a[0]                # tensor(100)，a 也变了
a.data_ptr() == b.data_ptr()   # True：起始地址相同
```

### 6.2 stride

`stride[d]` 表示沿第 d 维前进 1，在底层存储里要跨多少个元素。

```python
x = torch.arange(12).view(3, 4)
x.stride()                 # (4, 1)
x.T.stride()               # (1, 4)  转置：只改 stride，没动数据 → 不再连续
```

【引擎】`store_kvcache` 里的断言 `key.stride(1) == head_dim`：`k` 是 `qkv.split(...)` 切出来的视图，行与行之间的间距是 `q_size + 2*kv_size`，不等于 `D`，所以 kernel 要单独接收 `key.stride(0)`。

### 6.3 `.data` 与 `detach()`

```python
p = nn.Parameter(torch.randn(3))
p.data          # 与 p 共享存储，但脱离自动求导系统（不会触发 leaf 原地操作报错）
p.detach()      # 同样共享存储，脱离计算图；区别细微（版本计数）
```

权重加载里用 `param.data.copy_(...)`，是图方便且不触发 autograd 检查的惯用写法，见 8.3。

### 6.4 锁页内存与异步拷贝

```python
t = torch.tensor([1, 2, 3], pin_memory=True)   # 锁页（pinned）CPU 内存
g = t.cuda(non_blocking=True)                   # 异步 H2D，不阻塞 CPU
```

只有源张量在锁页内存里，`non_blocking=True` 才真正异步。【引擎】`prepare_prefill / prepare_decode` 里每个张量都这样搬运，减少 CPU 等待。

---

## 7. `nn.Module` 体系

### 7.1 Module 与 Parameter

```python
import torch
from torch import nn

class Tiny(nn.Module):
    def __init__(self):
        super().__init__()                    # 必须先调用，初始化内部字典
        self.weight = nn.Parameter(torch.empty(4, 3))   # 注册为可训练参数
        self.register_buffer("table", torch.arange(4.0), persistent=False)  # 非参数的张量状态

    def forward(self, x):
        return x @ self.weight.T
```

|概念|是否在 `parameters()` 里|是否在 `state_dict()` 里|典型用途|
|---|---|---|---|
|`nn.Parameter`|是|是|权重|
|buffer（`persistent=True`）|否|是|BN 的均值等|
|buffer（`persistent=False`）|否|**否**|【引擎】RoPE 的 `cos_sin_cache`，不存入权重文件|
|普通张量属性|否|否|不会随 `.to()` 移动设备|

**`nn.Module.__setattr__`**：给 Module 赋值时，PyTorch 根据值的类型登记它：`Parameter` 进参数字典，`Module` 进子模块字典，其他值走普通属性。所以 `MergedColumnParallelLinear.__init__` 里先 `self.output_sizes = [...]`（普通 list）再 `super().__init__()` 能行，赋的是普通值；但先赋 Parameter/Module 会报错，因为内部字典还没创建。

### 7.2 forward 与 `__call__`

```python
m = Tiny()
y = m(x)            # 调用 __call__，内部处理 hooks 后再调 forward
y = m.forward(x)    # 不推荐：绕过 hooks
```

### 7.3 遍历与按名字取

|API|返回|
|---|---|
|`m.parameters()`|所有参数（迭代器）|
|`m.named_parameters()`|`(完整点分名, 参数)`，名字形如 `model.layers.0.self_attn.qkv_proj.weight`|
|`m.named_modules()` / `m.modules()`|所有子模块（含自身）|
|`m.get_parameter("a.b.c.weight")`|**按点分名取参数**，不存在抛 `AttributeError`|
|`m.get_submodule("a.b.c")`|按点分名取子模块|
|`m.state_dict()`|参数与持久 buffer 的有序字典|
|`m.load_state_dict(sd, strict=True)`|按 key 加载（见第 8 节）|

【引擎】`load_model` 的核心就是 `model.get_parameter(param_name)`。

### 7.4 容器

```python
self.layers = nn.ModuleList([Block() for _ in range(28)])   # 能被 parameters() 找到
# 普通 Python list 里的子模块不会被登记！
```

`nn.ModuleList` 的下标是名字的一部分：第 0 层的路径是 `layers.0`，所以权重名里有 `model.layers.0.self_attn...`。

### 7.5 Hook（调试利器）

```python
h = layer.register_forward_hook(lambda m, args, out: print(out.shape))
# 带关键字参数的模块（新版 HF 常见）用 with_kwargs=True：
h = layer.register_forward_hook(lambda m, args, kwargs, out: ..., with_kwargs=True)
h.remove()
```

### 7.6 `train()` / `eval()`

`eval()` 切换 Dropout、BatchNorm 的行为，**不会关闭梯度计算**，推理时关梯度要用第 11 节的 `no_grad` / `inference_mode`。

---

## 8. 权重加载（重点）

### 8.1 全貌：从磁盘到 GPU 参数

```
磁盘 *.safetensors
   │  safe_open(...).get_tensor(name)      读出一个张量（CPU）
   ▼
loaded_weight（完整权重，未切分）
   │  按 tp_rank 取属于本卡的那一片  ← narrow / chunk
   ▼
本卡切片（视图）
   │  param.data.copy_(切片)              拷进 GPU 上已存在的参数（同时完成 dtype 转换与设备搬运）
   ▼
GPU 上的 nn.Parameter（构造时用 torch.empty 创建的"空壳"）
```

两个前提要先成立：

1. **参数的壳先建好**：模型在 GPU 上以"本 rank 的切分后形状"构造出来，内容是未初始化的垃圾值。
2. **checkpoint 里存的是未切分的完整权重**：切分发生在加载时，不存进文件。

### 8.2 方式一：标准的 `state_dict` 加载

```python
sd = torch.load("model.pt", map_location="cpu")     # pickle 格式的旧式 checkpoint
missing, unexpected = model.load_state_dict(sd, strict=False)
```

|参数|含义|
|---|---|
|`strict=True`（默认）|key 不匹配就报错|
|`strict=False`|不报错，返回 `missing_keys`（模型有、文件没有）和 `unexpected_keys`（文件有、模型没有）|

**为什么引擎不用它**：`load_state_dict` 要求文件里的 key 与模型参数**逐个对应、形状完全一致**。引擎的参数有两处不一致：(1) 文件里 q/k/v 是三个张量，模型里是合并的一个；(2) TP 下文件是完整形状，模型参数是切片形状。所以引擎自己写了 `load_model`，给参数挂 `weight_loader` 来处理。

`torch.load` 基于 pickle，**加载不可信文件有任意代码执行风险**，这是 safetensors 格式出现的动机之一。

### 8.3 方式二：safetensors

```python
from safetensors import safe_open

with safe_open("model.safetensors", framework="pt", device="cpu") as f:
    names = list(f.keys())                    # 所有张量名
    t = f.get_tensor("model.embed_tokens.weight")     # 读出完整张量（CPU）
    sl = f.get_slice("model.layers.0.self_attn.q_proj.weight")
    sl.get_shape(); sl.get_dtype()            # 只读文件头，不加载数据
    part = sl[0:128]                          # 只读前 128 行
```

|特点|说明|
|---|---|
|纯数据格式|不含可执行代码，没有 pickle 的安全问题|
|文件头是 JSON|记录每个张量的名字、dtype、形状、偏移，所以能**只读头不读数据**|
|按需读取|单个张量惰性读出（我的理解是内部用内存映射），不会一次把整个文件读进内存|
|大模型分片|多个 `model-0000x-of-0000y.safetensors`，加上 `model.safetensors.index.json` 记录"某个张量在哪个分片"|

【引擎】`loader.py`：

```python
for file in glob(os.path.join(path, "*.safetensors")):
    with safe_open(file, "pt", "cpu") as f:
        for weight_name in f.keys():
            ...f.get_tensor(weight_name)...
```

用 `with` 保证文件句柄关闭；`glob` 遍历目录下所有分片。

### 8.4 逐行拆解 `weight_loader`

```python
def weight_loader(self, param: nn.Parameter, loaded_weight: torch.Tensor):
    param_data = param.data                              # ①
    shard_size = param_data.size(self.tp_dim)            # ②
    start_idx = self.tp_rank * shard_size                # ③
    loaded_weight = loaded_weight.narrow(self.tp_dim, start_idx, shard_size)   # ④
    param_data.copy_(loaded_weight)                      # ⑤
```

|行|做什么|为什么|
|---|---|---|
|① `param.data`|取参数底层张量，与 `param` 共享存储，脱离自动求导|`param` 是 `requires_grad=True` 的叶子，直接对它原地写会报错；`.data` 绕开这个检查|
|② `param_data.size(tp_dim)`|本 rank 的参数在被切分维度上的长度|参数在构造时就已经是**切分后的形状**，所以这个长度就是"每片多大"|
|③ `tp_rank * shard_size`|本 rank 的片在完整权重里的起点|各 rank 的片首尾相接|
|④ `narrow(tp_dim, start, length)`|从完整权重里切出本 rank 的那一片（**视图**）|不拷贝，只是一个窗口|
|⑤ `copy_`|把这片数据写进参数的存储|同时完成 CPU→GPU 搬运和 dtype 转换|

**用数字走一遍**（列并行，完整权重 `[8, 4]`，TP=2，`tp_dim=0`）：

- 本 rank 的参数形状是 `[4, 4]`（构造时 `output_size / tp`）。
- ② `shard_size = 4`；③ rank 0 的 `start = 0`，rank 1 的 `start = 4`。
- ④ rank 0 取完整权重的第 0..3 行，rank 1 取第 4..7 行。
- ⑤ 各自拷进自己的 `[4, 4]` 参数。

### 8.5 `narrow` 与 `chunk` 与切片，三者对比

```python
W = torch.arange(32).view(8, 4)

W.narrow(0, 4, 4)            # 第 0 维从下标 4 起取 4 个 → 行 4..7，视图
W[4:8]                       # 等价的切片写法
W.chunk(2, 0)[1]             # 切成两块，取第 2 块 → 行 4..7，视图
```

三者在整除时结果相同，都是视图。区别在"表达意图"：

|写法|需要的信息|引擎里谁在用|
|---|---|---|
|`narrow(dim, start, length)`|起点和长度（要自己算偏移）|`ColumnParallelLinear`、`RowParallelLinear`、`VocabParallelEmbedding`（用 `tp_rank * shard_size` 算起点）|
|`chunk(n, dim)[rank]`|总块数和 rank|`MergedColumnParallelLinear`、`QKVParallelLinear` 的**来源**一侧|

`narrow` 对**越界**会抛错（起点加长度超过该维大小），这是个有用的安全网：切分维度不匹配时能立刻发现。

### 8.6 完整可运行的例子：单进程模拟两个 rank

```python
import torch

class FakeColumn:
    """模拟 ColumnParallelLinear 的加载逻辑（单进程，rank 手动指定）"""
    def __init__(self, in_f, out_f, tp_size, tp_rank):
        self.tp_dim, self.tp_rank = 0, tp_rank
        self.weight = torch.nn.Parameter(torch.empty(out_f // tp_size, in_f))  # 本 rank 的切片形状

    def weight_loader(self, param, loaded_weight):
        param_data = param.data
        shard_size = param_data.size(self.tp_dim)
        start_idx = self.tp_rank * shard_size
        loaded_weight = loaded_weight.narrow(self.tp_dim, start_idx, shard_size)
        param_data.copy_(loaded_weight)

full = torch.arange(32, dtype=torch.float32).view(8, 4)       # 完整权重
shards = []
for rank in range(2):
    layer = FakeColumn(in_f=4, out_f=8, tp_size=2, tp_rank=rank)
    layer.weight.weight_loader = layer.weight_loader          # 把方法挂到 Parameter 上
    layer.weight.weight_loader(layer.weight, full)            # 模拟 load_model 的调用
    shards.append(layer.weight.data)

print(shards[1])                                  # 应为 full 的行 4..7
print(torch.equal(torch.cat(shards, 0), full))    # True：两片拼起来 = 完整权重
```

**验证 TP 切分对不对的通用手法**：把各 rank 的切片沿切分维拼回去，应当等于完整权重。

### 8.7 行并行：切输入维（`tp_dim=1`）

```python
full = torch.arange(32, dtype=torch.float32).view(4, 8)    # [out=4, in=8]
# TP=2：每 rank 参数 [4, 4]，tp_dim=1
full.narrow(1, 0, 4)    # rank 0：列 0..3
full.narrow(1, 4, 4)    # rank 1：列 4..7，即 [[4,5,6,7],[12,13,14,15],[20,21,22,23],[28,29,30,31]]
```

`RowParallelLinear.weight_loader` 多一个分支：

```python
if param_data.ndim == 1:      # bias 是一维，不切，整份拷贝
    param_data.copy_(loaded_weight)
    return
```

因为 Row 的 bias 形状是 `[out]`，它在所有 rank 上是全量的（前向时只有 rank 0 加，见引擎笔记）。

### 8.8 合并层：为什么 `param_data` 要再 `narrow` 一次

```python
# MergedColumnParallelLinear.weight_loader（简化）
param_data = param.data
shard_offset = sum(self.output_sizes[:loaded_shard_id]) // self.tp_size   # 目的偏移
shard_size   = self.output_sizes[loaded_shard_id] // self.tp_size
param_data = param_data.narrow(self.tp_dim, shard_offset, shard_size)     # ① 目的：参数里的一段
loaded_weight = loaded_weight.chunk(self.tp_size, self.tp_dim)[self.tp_rank]   # ② 来源：完整权重的本 rank 片
param_data.copy_(loaded_weight)
```

两侧各取一段，再拷贝：

```
目的（本 rank 的合并参数）：[ gate 段 | up 段 ]
                              ↑ narrow(offset, size)
来源（完整的 gate_proj.weight）→ chunk(tp)[rank] 
```

**关键性质**：`param_data.narrow(...)` 返回的是**视图**，所以后面对它 `copy_`，数据写进的是原参数的那一段。如果这里不小心用了会产生拷贝的操作（如整数索引、`.clone()`），写入的就是临时副本，原参数什么都没变，且不会报错。

用数字（`output_sizes=[8, 8]`，TP=2，hidden 随意）：

|加载|`shard_offset`|`shard_size`|目的（参数的行）|来源|
|---|---|---|---|---|
|gate（id=0）|`0 // 2 = 0`|`8 // 2 = 4`|0..3|gate 权重 `chunk(2, 0)[rank]`|
|up（id=1）|`8 // 2 = 4`|4|4..7|up 权重 `chunk(2, 0)[rank]`|

QKV 同理，只是 q 段与 k、v 段大小不同（GQA），偏移按段累加，`shard_id` 是字符串 `"q"/"k"/"v"`。

### 8.9 把函数挂到 Parameter 上：Python 的动态属性

```python
self.weight = nn.Parameter(torch.empty(out, in_))
self.weight.weight_loader = self.weight_loader      # 给对象动态添加属性
```

- Python 允许给对象随时添加属性（除非用了 `__slots__`），`Parameter` 也是普通对象。
- `self.weight_loader` 是**绑定方法**：它已经捕获了 `self`，所以之后被调用时 `self.tp_rank`、`self.tp_dim` 都可用。
- 加载器调用 `param.weight_loader(param, tensor)`：虽然 `param` 就是 `self.weight`，仍然显式传入，是为了让 loader 接口通用，一个函数可以服务 weight 和 bias 两个参数（`self.bias.weight_loader = self.weight_loader`）。
- 加载器通过 `getattr(param, "weight_loader", default)` 询问"这个参数有没有自带的加载方法"，没有就回退到默认的整份 `copy_`。这种"问对象有没有某个属性"的风格叫鸭子类型。

### 8.10 `loader.py` 的路由逻辑

```python
def load_model(model, path):
    packed = getattr(model, "packed_modules_mapping", {})     # 没有这个属性就当空 dict
    for file in glob(os.path.join(path, "*.safetensors")):
        with safe_open(file, "pt", "cpu") as f:
            for weight_name in f.keys():
                for k in packed:                               # 子串匹配
                    if k in weight_name:
                        v, shard_id = packed[k]
                        param_name = weight_name.replace(k, v)          # q_proj → qkv_proj
                        param = model.get_parameter(param_name)
                        param.weight_loader(param, f.get_tensor(weight_name), shard_id)
                        break
                else:                                           # for...else：没有 break 才执行
                    param = model.get_parameter(weight_name)
                    loader = getattr(param, "weight_loader", default_weight_loader)
                    loader(param, f.get_tensor(weight_name))
```

两个分支：

|分支|触发条件|例子|
|---|---|---|
|packed|权重名**包含** mapping 的某个 key|`...q_proj.weight`、`...q_proj.bias`、`...gate_proj.weight`|
|普通|其余|`o_proj.weight`、`input_layernorm.weight`、`embed_tokens.weight`|

注意：匹配用的是**子串**（`k in weight_name`），所以 `q_proj.bias` 与 `q_proj.weight` 都会被改写，bias 也能走合并加载。

### 8.11 常见坑与排查

|现象|原因|怎么查|
|---|---|---|
|模型能跑但输出乱码，无报错|有参数没被加载（`empty` 的垃圾值）|8.12 的 NaN 哨兵检查|
|`AttributeError: ... has no attribute 'xxx'`|checkpoint 里有模型没有的参数名|对照 safetensors 的 key 与 `named_parameters()`|
|`RuntimeError: The size of tensor a must match ...`|`copy_` 两侧形状不匹配且不可广播|打印 `param_data.shape` 与 `loaded_weight.shape`|
|**加载"成功"但结果错**|`copy_` 的**广播**悄悄掩盖了形状错误，例如目的 `[8, 4]`、来源 `[1, 4]`，会把一行复制 8 次|加载前断言 `param_data.shape == loaded_weight.shape`|
|合并层数值错|`shard_id` 或偏移错位，q/k/v 或 gate/up 互相覆盖|对比 TP=1 时 HF 对应张量与合并参数的相应行|
|TP=2 错，TP=1 对|切分维度、Row 的 bias、头数整除|8.6 的"切片拼回应等于完整权重"|
|tie 的 `lm_head` 全乱|赋值顺序错：必须先让 `lm_head.weight.data = embed.weight.data`，再加载 embedding|加载后比较 `data_ptr()` 是否相同|
|dtype 不对|`copy_` 会静默转换（fp32 → bf16 有精度损失）|打印 `param.dtype` 与文件里张量的 dtype|

**`copy_` 的广播危害**是最隐蔽的一类。给自己的加载代码加一行防线：

```python
assert param_data.shape == loaded_weight.shape, (param_data.shape, loaded_weight.shape)
param_data.copy_(loaded_weight)
```

### 8.12 验证加载是否完整（NaN 哨兵）

```python
for p in model.parameters():
    p.data.fill_(float("nan"))               # 全部填 NaN
load_model(model, path)
missing = [n for n, p in model.named_parameters() if torch.isnan(p).any()]
print("未被加载的参数:", missing)             # 期望 []
```

任何被正确加载的参数都不会含 NaN。tie 的 `lm_head.weight` 与 embedding 共享存储，加载 embedding 时已一并被填好，不会被误报。

### 8.13 验证数值是否正确

```python
# 与 HF 的同名参数逐个比较（TP=1 时形状一致）
hf_sd = hf_model.state_dict()
for name, p in model.named_parameters():
    if ".qkv_proj." in name:            # 合并参数：拼接 q/k/v 后比较
        base = name.replace("qkv_proj", "{}_proj")
        ref = torch.cat([hf_sd[base.format(s)] for s in "qkv"], 0)
    elif ".gate_up_proj." in name:
        base = name.replace("gate_up_proj", "{}_proj")
        ref = torch.cat([hf_sd[base.format(s)] for s in ("gate", "up")], 0)
    else:
        ref = hf_sd.get(name)
    if ref is not None:
        print(name, torch.equal(p.data.cpu(), ref.to(p.dtype).cpu()))
```

---

## 9. 设备与 dtype

```python
torch.set_default_dtype(torch.bfloat16)       # 之后 torch.empty/randn 等默认用 bf16
torch.set_default_device("cuda")              # 之后创建的张量默认在 GPU 上
torch.get_default_dtype()                     # 查询

with torch.device("meta"):                    # "meta" 设备：只记录形状，不分配内存
    model = SomeModel(cfg)
```

【引擎】`ModelRunner.__init__` 先把默认 dtype/设备设成目标值，构造完模型后**恢复**，避免污染调用方的其他代码。这是一种"借用窗口"的写法。

`meta` 设备适合"只看结构、不要内存"的场景（比如打印模块树、查看参数形状），上面的结构追踪脚本用到它。

---

## 10. CUDA 相关

### 10.1 设备与同步

```python
torch.cuda.set_device(rank)       # 设置当前线程默认 GPU
torch.cuda.synchronize()          # 等 GPU 上所有已提交的任务做完
torch.cuda.is_available()
torch.cuda.device_count()
```

GPU 操作是**异步**的：Python 调用返回时 kernel 可能还没跑完。`.item()`、`.tolist()`、打印 GPU 张量会触发隐式同步。

### 10.2 显存统计

```python
free, total = torch.cuda.mem_get_info()                       # 当前设备的空闲/总字节数
stats = torch.cuda.memory_stats()
stats["allocated_bytes.all.peak"]                              # PyTorch 分配的峰值
stats["allocated_bytes.all.current"]                           # 当前已分配
torch.cuda.reset_peak_memory_stats()
torch.cuda.empty_cache()                                       # 把缓存的空闲块还给驱动
```

【引擎】`warmup_model` 先 `reset_peak_memory_stats()` 再跑一次最大规模的前向，之后读 `peak` 得到激活峰值；`allocate_kv_cache` 用 `total * utilization - used - peak + current` 估算剩余可用显存。

### 10.3 CUDA Graph

```python
g = torch.cuda.CUDAGraph()
with torch.cuda.graph(g, pool):          # 录制：期间的 kernel 被记下来
    out[:] = model(inp)
g.replay()                               # 重放：一次 launch 重放整串 kernel
```

规则：

1. 录制前必须先**热身**一次（让懒初始化、编译、cuBLAS 句柄创建发生在录制之外）。
2. 录制的是 kernel 与**固定的内存地址**，所以输入输出要用预分配的**静态缓冲区**，重放前把新数据拷进去。
3. 多个 graph 可以共享一个内存池（`pool`），省显存。

---

## 11. 推理模式

```python
with torch.no_grad():          # 不记录计算图
    y = model(x)

@torch.inference_mode()        # 更彻底：连"版本计数"等开销也省
def run(...): ...
```

| `no_grad`       | `inference_mode` |                |
| --------------- | ---------------- | -------------- |
| 不构建计算图          | ✓                | ✓              |
| 额外省开销           |                  | ✓              |
| 产出的张量之后能否参与自动求导 | 能                | **不能**（"推理张量"） |

纯推理引擎用 `inference_mode`。加载权重时不用它，而是用 `param.data`（见 8.4）。

---

## 12. `torch.compile`

```python
@torch.compile
def forward(self, x): ...
```

- 首次调用会**追踪并编译**，之后相同形状直接用编译结果；形状变化会触发重编译（之后会尝试动态形状）。
- 对"很多小的逐元素算子串在一起"的函数收益最大：这些算子都是访存受限的，融合成一个 kernel 就省掉了中间结果的显存读写。
- 【引擎】`RMSNorm`、`RotaryEmbedding.forward`、`SiluAndMul`、`Sampler` 都靠它自动融合，不需要手写 kernel。
- 首次调用的编译耗时放在启动阶段（`warmup_model`）里完成，避免第一个请求变慢。

---

## 13. `torch.distributed`

### 13.1 基本概念

|概念|含义|
|---|---|
|进程组|一组协同工作的进程，每个进程管一张 GPU|
|`world_size`|进程总数|
|`rank`|本进程的编号（0 .. world_size-1）|
|后端|`nccl`（GPU 间通信的标准选择）|

### 13.2 API

```python
import torch.distributed as dist
dist.init_process_group("nccl", "tcp://localhost:2333", world_size=2, rank=0)
dist.get_rank(); dist.get_world_size()

dist.all_reduce(t)                     # 各 rank 的 t 逐元素求和，结果**原地**写回每个 rank 的 t
dist.gather(t, gather_list, dst=0)     # 各 rank 的 t 汇到 dst；gather_list 只在 dst 上提供，其他 rank 传 None
dist.barrier()                         # 等所有 rank 到齐
dist.destroy_process_group()
```

|操作|谁拿到结果|引擎里的用处|
|---|---|---|
|`all_reduce`|所有 rank|`RowParallelLinear` 汇总部分和；`VocabParallelEmbedding` 汇总各自查到的向量|
|`gather`|只有 `dst`|`ParallelLMHead` 把各 rank 的 logits 分片汇到 rank 0|
|`barrier`|—|共享内存创建/退出对齐|

**进程组必须先于任何并行层的构造**：`LinearBase.__init__` 里就调用了 `dist.get_rank()`。

---

## 14. 自测练习

1. **手写加载**：用 8.6 的 `FakeColumn` 改写出 `FakeRow`（`tp_dim=1`，参数 `[4, 4]`，完整权重 `[4, 8]`），验证两片沿 dim 1 拼回等于完整权重。
2. **合并层**：写 `FakeMerged(output_sizes=[8, 8], tp=2)`，对 gate、up 各调用一次 loader，打印每个 rank 的参数，核对前 4 行来自 gate、后 4 行来自 up。
3. **广播陷阱**：构造目的 `[8, 4]`、来源 `[1, 4]` 的 `copy_`，观察不报错的结果，再加上形状断言看它如何拦住。
4. **视图陷阱**：把 8.8 里的 `narrow` 换成 `param_data[idx_tensor]`（整数张量索引），说明为什么写入不再进入原参数。
5. **NaN 哨兵**：对 `nn.Linear` 做一次"全部填 NaN，只加载 weight，不加载 bias"，确认 bias 被查出。
6. **读真代码**：对照 `linear.py`，给 `QKVParallelLinear.weight_loader` 的 q、k、v 三个分支各算一次偏移与长度（Qwen2.5-1.5B，TP=2：q 偏移 0、长 768，k 偏移 768、长 128，v 偏移 896、长 128）。

---

## 15. 速查：引擎里出现的 API

|API / 语法|出现位置|本文章节|
|---|---|---|
|`torch.empty`、`empty_like`|参数壳、KV cache、`Sampler`|1.1|
|`.view` / `.flatten` / `.unsqueeze`|`Attention`、`Sampler`|2|
|`.split` / `.chunk` / `.narrow`|`qkv`、`SiluAndMul`、`weight_loader`|2.4、8.5|
|`x[idx]`、`x[mask]`|RoPE 查表、embedding 掩码、`last_indices`|3|
|`copy_` / `fill_` / `zero_` / `mul_`|`weight_loader`、`run_model`、`RMSNorm`|4|
|`F.linear`、`F.silu`、`F.embedding`|各层|5|
|`.stride()`|`store_kvcache` 断言|6.2|
|`pin_memory` + `non_blocking`|`prepare_*`|6.4|
|`nn.Parameter`、`register_buffer`|线性层、RoPE|7.1|
|`get_parameter`|`load_model`|7.3、8.10|
|`safe_open`、`get_tensor`|`load_model`|8.3|
|`param.data.copy_`|`weight_loader`|8.4|
|`set_default_dtype` / `set_default_device`|`ModelRunner.__init__`|9|
|`mem_get_info`、`memory_stats`|`allocate_kv_cache`|10.2|
|`CUDAGraph`、`torch.cuda.graph`|`capture_cudagraph`|10.3|
|`inference_mode`|`run_model`|11|
|`torch.compile`|`RMSNorm`、RoPE、`Sampler`|12|
|`all_reduce`、`gather`、`barrier`|并行层、`ModelRunner`|13|