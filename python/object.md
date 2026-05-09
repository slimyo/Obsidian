# Python 中的 `object` 类完全指南

## 1. `object` 的核心身份

在 Python 中，`object` 是**所有类的根基类**（root base class）。它是 Python 对象体系的**最顶层**。

- 每一个类（无论是内置类如 `int`、`str`，还是用户自定义类）都直接或间接地继承自 `object`。
- 如果定义一个类时没有指定父类，Python 会默认让其继承 `object`。

```python
class MyClass:
    pass

# 等价于
class MyClass(object):
    pass

print(issubclass(MyClass, object))   # True
print(issubclass(int, object))        # True
print(issubclass(str, object))        # True
print(issubclass(type, object))       # True   (元类也是类，也继承 object)
```

---

## 2. `object` 与 `type` 的关系

这是 Python 对象模型中**最容易混淆**的地方。两条重要的“循环”事实：

1. **`object` 是 `type` 的实例**（因为 `type` 是所有类的元类，`object` 也是一个类）。
2. **`type` 继承自 `object`**（因为 `type` 也是一个类，所有类都继承 `object`）。

```python
print(isinstance(object, type))   # True  (object 是 type 的实例)
print(issubclass(type, object))   # True  (type 是 object 的子类)
```

形象地说：
- `type` 负责“创造”类（也包括 `object` 这个类）。
- `object` 负责“成为”所有类的最终父类。

它们形成了一种特殊的循环依赖，但这种设计是 Python 对象系统的基础。

---

## 3. `object` 提供的默认行为

任何类继承 `object` 后，都会获得一组默认的方法和属性。其中最常用的包括：

| 方法 | 作用 | 默认行为 |
|------|------|----------|
| `__new__(cls, ...)` | **静态方法**，创建实例 | 分配内存，返回一个新实例 |
| `__init__(self, ...)` | 初始化实例 | 无操作，可以重写 |
| `__repr__(self)` | 可读的字符串表示 | `'<__main__.MyClass object at 0x...>'` |
| `__str__(self)` | 用户友好的字符串表示 | 同 `__repr__`（如果未重写） |
| `__eq__(self, other)` | 相等判断 | 比较对象 id (`is`) |
| `__hash__(self)` | 哈希值 | 基于对象 id 生成 |
| `__dir__(self)` | 属性列表 | 返回所有属性和方法名 |
| `__class__` | 属性 | 指向对象的类 |

### 重要：`__new__` 与 `__init__`

`__new__` 是真正创建实例的静态方法，`__init__` 则是初始化已创建实例的方法。它们的关系是：

```python
obj = MyClass(*args, **kwargs)
# 等价于：
obj = MyClass.__new__(MyClass, *args, **kwargs)
if isinstance(obj, MyClass):
    MyClass.__init__(obj, *args, **kwargs)
```

`__new__` 可以被重写以实现**单例**或**不可变类型**（如 `int`、`str`）的实例创建。

---

## 4. `object` 在元类机制中的位置

回顾之前的元类笔记：

```
object
   ↑
type
   ↑
UOpMetaClass  (自定义元类)
   ↑
UOp           (普通类)
```

- `object` 处于整个继承链的最顶端。
- `type` 继承自 `object`。
- 任何自定义元类（继承 `type`）间接继承 `object`，因此元类实例（即普通类）也最终继承 `object`。

```python
# 验证 UOp 继承自 object
print(issubclass(UOp, object))        # True

# 验证 UOpMetaClass 也继承自 object
print(issubclass(UOpMetaClass, object)) # True
```

---

## 5. 典型应用：单例模式的两种实现

### 使用 `__new__`（适合简单单例）

```python
class Singleton:
    _instance = None
    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

a = Singleton()
b = Singleton()
print(a is b)   # True
```

### 使用元类 `__call__`（更灵活，如前面 UOp 示例）

```python
class SingletonMeta(type):
    _instances = {}
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class MyClass(metaclass=SingletonMeta):
    pass
```

元类方式适用于需要**“按参数去重”**的场景（如 UOp 的缓存机制），而 `__new__` 适用于**“类级别只创建一个实例”**的场景。

---

## 6. 常见误区

### 误区1：`object` 是空类，什么都没有。
实际上 `object` 提供了所有 Python 对象共有的基础设施（`__new__`、`__init__`、`__repr__` 等）。

### 误区2：`type` 和 `object` 没有区别。
区别巨大：`object` 是实例和类的父类，`type` 是元类。用 `isinstance` 和 `issubclass` 对比：

```python
print(isinstance(object, type))   # True  (object 是 type 的实例)
print(isinstance(type, object))   # True  (type 是 object 的实例)
print(issubclass(object, type))   # False (object 不是 type 的子类)
print(issubclass(type, object))   # True  (type 是 object 的子类)
```

### 误区3：自定义类如果不继承 `object` 就会更轻量。
在 Python 3 中，所有类都隐式继承 `object`，显式写出 `class MyClass(object):` 只是为了兼容 Python 2。Python 3 中两者完全等价。

---

## 7. 总结

| 概念 | 说明 |
|------|------|
| `object` | 所有类的根基类，提供默认的对象行为 |
| `object.__new__` | 静态方法，实际创建实例 |
| `object.__init__` | 初始化实例 |
| `type` 与 `object` | `type` 继承 `object`，`object` 是 `type` 的实例 |
| 单例实现 | 可重写 `__new__` 或使用元类 `__call__` |

理解 `object` 是理解 Python 对象模型的基石。结合前文的 `type` 笔记，可以完整把握 Python 中“类 – 元类 – 实例”的三层关系。