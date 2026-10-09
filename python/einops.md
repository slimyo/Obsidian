# einops 基础 API 教程（对比原生张量操作）

## 0. 为什么用 einops

原生操作靠"下标 + 位置"（`permute(0,2,3,1)`、`view(b,-1)`），读代码时要自己推断每一维的含义。einops 用**带命名的维度表达式**描述变换，代码即文档，还会自动检查形状。

```bash
pip install einops
```

```python
import torch
from einops import rearrange, reduce, repeat, einsum, pack, unpack
from einops.layers.torch import Rearrange, Reduce  # 作为 nn.Module 层使用

x = torch.randn(8, 3, 32, 32)  # b c h w
```

支持 numpy / torch / jax / tf 等后端，API 完全一致。

---

## 1. 表达式语法速览

|语法|含义|示例|
|---|---|---|
|`a b c`|为每个轴命名|`'b c h w'`|
|`->`|左边是输入模式，右边是输出模式|`'b c h w -> b h w c'`|
|`( )`|合并（右侧）或拆分（左侧）轴|`'b c h w -> b (c h w)'`|
|`...`|匹配任意数量的其余轴|`'... h w -> ... w h'`|
|`1`|长度为 1 的匿名轴|`'c -> 1 c'`|
|`k=数值`|拆分时指定某个轴的长度|`h2=2`|

**核心规则**：括号内的轴顺序决定内存排布，`(h w)` 与 `(w h)` 不同，等价于 row-major 展开顺序。

---

## 2. `rearrange`：重排 / 合并 / 拆分

对应原生的 `permute` / `transpose` / `reshape` / `view` / `squeeze` / `unsqueeze`。

### 2.1 转置

```python
# 原生
y = x.permute(0, 2, 3, 1)
# einops
y = rearrange(x, 'b c h w -> b h w c')
```

### 2.2 展平

```python
# 原生
y = x.reshape(x.shape[0], -1)
# einops
y = rearrange(x, 'b c h w -> b (c h w)')
```

### 2.3 拆分轴（如分组）

```python
# 原生：要自己算 channel // groups
g = 3
y = x.reshape(x.shape[0], g, x.shape[1] // g, *x.shape[2:])
# einops：只需给出一个已知长度，另一个自动推断
y = rearrange(x, 'b (g c) h w -> b g c h w', g=g)
```

### 2.4 增删长度为 1 的轴

```python
# 原生
y = x.unsqueeze(1)          # 插入
z = y.squeeze(1)            # 删除
# einops
y = rearrange(x, 'b c h w -> b 1 c h w')
z = rearrange(y, 'b 1 c h w -> b c h w')   # 若该轴长度不是 1 会报错，起到断言作用
```

### 2.5 `...` 省略号

```python
# 交换最后两维，不关心前面有几维
y = rearrange(x, '... h w -> ... w h')     # 原生: x.transpose(-1, -2)
```

### 2.6 经典案例：ViT 的 patchify

```python
# 原生：unfold/reshape/permute 一长串，难以验证
p = 8
b, c, H, W = x.shape
y = (x.reshape(b, c, H // p, p, W // p, p)
       .permute(0, 2, 4, 3, 5, 1)
       .reshape(b, (H // p) * (W // p), p * p * c))
# einops
y = rearrange(x, 'b c (h p1) (w p2) -> b (h w) (p1 p2 c)', p1=p, p2=p)
```

### 2.7 经典案例：多头注意力拆头

```python
qkv = torch.randn(8, 100, 3 * 4 * 64)    # b n (3*h*d)
# 原生
b, n, _ = qkv.shape
q, k, v = qkv.reshape(b, n, 3, 4, 64).permute(2, 0, 3, 1, 4)
# einops
q, k, v = rearrange(qkv, 'b n (three h d) -> three b h n d', three=3, h=4)
# 合并头
out = rearrange(q, 'b h n d -> b n (h d)')   # 原生: q.transpose(1,2).reshape(b,n,-1)
```

---

## 3. `reduce`：归约 / 池化

对应 `mean` / `sum` / `max` / `min` / `prod` / `any` / `all` 以及池化操作。第三个参数是归约方式，**出现在左侧、没有出现在右侧的轴就被归约掉**。

### 3.1 全局平均池化

```python
# 原生
y = x.mean(dim=(2, 3))
# einops
y = reduce(x, 'b c h w -> b c', 'mean')
```

### 3.2 保留维度（keepdim）

```python
# 原生
y = x.mean(dim=(2, 3), keepdim=True)
# einops
y = reduce(x, 'b c h w -> b c 1 1', 'mean')
```

### 3.3 2×2 最大池化（用括号拆分再归约）

```python
# 原生
y = torch.nn.functional.max_pool2d(x, kernel_size=2)
# einops
y = reduce(x, 'b c (h h2) (w w2) -> b c h w', 'max', h2=2, w2=2)
```

同一思路可以做任意窗口的平均池化、按组求和等，不需要查各种 `pool` 函数的参数。

### 3.4 归约方式

`'min' | 'max' | 'sum' | 'mean' | 'prod' | 'any' | 'all'`，也可传入自定义函数（可调用对象）。

---

## 4. `repeat`：复制 / 平铺

对应 `unsqueeze + expand`、`repeat_interleave`、`tile`、`repeat`。

```python
img = torch.randn(32, 32)   # h w

# 增加新轴并复制（灰度图 -> 3 通道）
# 原生
y = img.unsqueeze(-1).expand(-1, -1, 3)
# einops
y = repeat(img, 'h w -> h w c', c=3)

# 逐元素重复（最近邻上采样）
# 原生
y = img.repeat_interleave(2, dim=0).repeat_interleave(2, dim=1)
# einops：新轴写在括号的右侧 -> 逐元素重复
y = repeat(img, 'h w -> (h h2) (w w2)', h2=2, w2=2)

# 整体平铺
# 原生
y = img.repeat(2, 2)
# einops：新轴写在括号的左侧 -> 整体平铺
y = repeat(img, 'h w -> (h2 h) (w2 w)', h2=2, w2=2)

# 给 batch 复制 CLS token
cls = torch.randn(1, 1, 256)
cls_b = repeat(cls, '1 1 d -> b 1 d', b=8)   # 原生: cls.expand(8, -1, -1)
```

---

## 5. `einsum`：爱因斯坦求和

`einops.einsum` 与 `torch.einsum` 的差别：**支持多字母轴名**，可读性更好。注意它的**最后一个参数是表达式**。

```python
q = torch.randn(8, 4, 100, 64)  # b h i d
k = torch.randn(8, 4, 100, 64)  # b h j d

# 原生：只能单字母
attn = torch.einsum('bhid,bhjd->bhij', q, k)
# einops：可用有意义的名字，空格分隔
attn = einsum(q, k, 'batch head i d, batch head j d -> batch head i j')
```

规则：出现在输入、不出现在输出的轴会被求和。

---

## 6. `pack` / `unpack`：打包与还原

把形状不同的张量沿某个（通配）轴拼接，之后可精确还原，适合处理 `[cls_token, patch_tokens]` 这类场景。

```python
cls   = torch.randn(8, 1, 256)      # b 1 d
patch = torch.randn(8, 196, 256)    # b n d

# 原生：torch.cat([cls, patch], dim=1)，之后用切片还原
tokens, ps = pack([cls, patch], 'b * d')      # tokens: (8, 197, 256)
...
cls2, patch2 = unpack(tokens, ps, 'b * d')    # 还原原形状
```

`*` 可以匹配多个轴，会被展平后拼接，`ps` 保存了还原所需的形状信息。

---

## 7. 作为 `nn.Module` 层使用

可以直接放进 `nn.Sequential`，不用写 lambda 或自定义 `forward`：

```python
import torch.nn as nn

model = nn.Sequential(
    nn.Conv2d(3, 64, 3, padding=1),
    nn.ReLU(),
    Reduce('b c (h h2) (w w2) -> b c h w', 'max', h2=2, w2=2),  # 池化
    Rearrange('b c h w -> b (c h w)'),                           # 展平
    nn.Linear(64 * 16 * 16, 10),
)
```

---

## 8. 其他实用函数

```python
from einops import parse_shape

# 把形状解析成字典，便于复用
parse_shape(x, 'b c h w')       # {'b': 8, 'c': 3, 'h': 32, 'w': 32}
```

---

## 9. 与原生操作对照速查表

|需求|原生|einops|
|---|---|---|
|转置|`x.permute(0,2,3,1)`|`rearrange(x, 'b c h w -> b h w c')`|
|展平|`x.flatten(1)`|`rearrange(x, 'b c h w -> b (c h w)')`|
|拆轴|`x.view(b, g, c//g, h, w)`|`rearrange(x, 'b (g c) h w -> b g c h w', g=g)`|
|全局池化|`x.mean((2,3))`|`reduce(x, 'b c h w -> b c', 'mean')`|
|窗口池化|`F.max_pool2d(x, 2)`|`reduce(x, 'b c (h h2) (w w2) -> b c h w', 'max', h2=2, w2=2)`|
|增维复制|`x.unsqueeze(-1).expand(...)`|`repeat(x, 'h w -> h w c', c=3)`|
|逐元素重复|`repeat_interleave`|`repeat(x, 'h w -> (h r) w', r=2)`|
|整体平铺|`x.repeat(2,2)`|`repeat(x, 'h w -> (r h) (r2 w)', r=2, r2=2)`|
|求和缩并|`torch.einsum('bhid,bhjd->bhij', ...)`|`einsum(q, k, 'b h i d, b h j d -> b h i j')`|
|拼接|`torch.cat([...], dim=1)`|`pack([...], 'b * d')`|

---

## 10. 常见坑

1. **括号内顺序**：`(h w)` 与 `(w h)` 的结果不同，写错不会报错，只会得到错误的数据排布。
2. **拆轴必须可推断**：`'b (g c) h w'` 里 `g` 和 `c` 至少给出一个，且能整除，否则报错。
3. **输出可能是非连续的**：`rearrange` 只做 `permute` 时返回 view，后续需要 `.view()` 时要先 `.contiguous()`，或者直接用 `.reshape()`。
4. **性能**：einops 本身只是对 `reshape/permute/...` 的封装，开销主要在表达式解析，有缓存，通常可忽略。在 `torch.compile` 下也能正常工作。
5. **轴名不能重复**：同一侧同名轴会报错；`einsum` 中同名轴出现在多个输入则表示它们要对齐。
6. **轴名要有意义**：能写 `batch head seq dim` 就别全写单字母，调试时错误信息更清晰。

---

## 11. 小练习（自测）

用 einops 实现：

1. 把 `(b, t, c)` 的时间序列按窗口长度 `w` 切成不重叠窗口，得到 `(b, t//w, w, c)`。
2. 对 `(b, n, d)` 的 token 序列在 `n` 轴上取平均，得到 `(b, d)`。
3. 把 `(b, h, n, d)` 的多头输出合并成 `(b, n, h*d)`。

参考答案：

```python
rearrange(x, 'b (n w) c -> b n w c', w=w)
reduce(x, 'b n d -> b d', 'mean')
rearrange(x, 'b h n d -> b n (h d)')
```