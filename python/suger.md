# **`@property

Python 中的 **`@property`装饰器**，它把一个方法变成"属性"，让我们可以用访问属性的语法来调用方法。这是一种语法糖，让代码更简洁、更 Pythonic。

## 基本概念

```python
@property
def shape(self) -> Tuple[int, ...]:
    return self.data.shape
```

**翻译**：创建一个名为 `shape`的**只读属性**，当访问 `obj.shape`时，实际上调用的是这个方法，返回 `self.data.shape`。

## 传统写法 vs @property 写法

### 传统写法（需要显式调用方法）

```python
class Tensor:
    def __init__(self, data):
        self.data = data
    
    def get_shape(self):  # 需要调用 get_shape() 方法
        return self.data.shape

# 使用
tensor = Tensor(some_data)
shape = tensor.get_shape()  # 需要加括号调用
```

### @property 写法（像属性一样访问）

```python
class Tensor:
    def __init__(self, data):
        self.data = data
    
    @property
    def shape(self):  # 变成属性
        return self.data.shape

# 使用
tensor = Tensor(some_data)
shape = tensor.shape  # 像属性一样访问，不需要括号
```