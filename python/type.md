# Python 中的 `type` 类完全指南

## 1. `type` 的核心身份

在 Python 中，`type` 是一个**内置的元类**（metaclass），即**所有类的类**。它是 Python 对象系统的核心组件。

```python
# type 是一切类的源头
print(type(int))      # <class 'type'>
print(type(str))      # <class 'type'>
print(type(UOp))      # <class 'type'>
print(type(type))     # <class 'type'>  (自己是自己的实例)
```

任何一个类（包括 `type` 自身）都是 `type` 的一个实例。

---

## 2. `type` 的两种完全不同的用法

### 2.1 作为内置函数：获取对象的类型

这是最常见、最简单的用法。

```python
num = 42
text = "hello"
print(type(num))    # <class 'int'>
print(type(text))   # <class 'str'>
```

此时 `type(obj)` 返回对象所属的类。

### 2.2 作为元类：动态创建类

`type(name, bases, dict)` 可以**动态地创建一个新的类**，等价于 `class` 语句。

```python
# 以下两种写法完全等价

# 写法1：class 语句
class MyClass:
    x = 10
    def foo(self):
        return self.x

# 写法2：type 动态创建
MyClass = type('MyClass',          # 类名
               (),                 # 父类元组
               {                   # 类属性和方法字典
                   'x': 10,
                   'foo': lambda self: self.x
               })
```

三个参数的含义：
- `name`：字符串，新类的名称。
- `bases`：元组，新类的父类（基类）。
- `dict`：字典，包含类属性、方法的定义。

动态创建在实际项目中并不常用，但它是理解元类的基础。

---

## 3. `type` 作为元类时的内部机制：`__call__` 方法

当 `type`（或其子类）作为一个**元类**时，它的 `__call__` 方法负责**实例化一个类**。

**核心链条**：
1. 定义类 `class MyClass: ...` 时，Python 使用元类（默认为 `type`）创建一个类对象。
2. 当执行 `obj = MyClass()` 时：
   - Python 调用元类的 `__call__` 方法。
   - 对于默认元类 `type`，`type.__call__` 会依次调用：
     - `MyClass.__new__(MyClass, ...)` 分配实例
     - `MyClass.__init__(obj, ...)` 初始化实例
     - 返回最终的对象

```python
# 简化的 type.__call__ 伪代码
def __call__(cls, *args, **kwargs):
    obj = cls.__new__(cls, *args, **kwargs)  # 1. 分配实例
    if isinstance(obj, cls):
        cls.__init__(obj, *args, **kwargs)   # 2. 初始化
    return obj
```

因此，当我们自定义元类时，重写 `__call__` 可以在**创建实例之前/之后**插入逻辑（比如缓存、单例、参数验证）。

---

## 4. 元类继承体系

```
object  (所有类的最终基类)
   ↑
type   (所有类的元类基类)
   ↑
UOpMetaClass   (自定义元类)
   ↑
UOp            (普通类，元类是 UOpMetaClass)
```

- `type` 继承自 `object`（因为 `type` 也是一个类）。
- 自定义元类必须直接或间接继承 `type`。
- 普通类的元类可以指定为自定义元类，从而改变类的创建行为。

```python
# 指定自定义元类
class UOp(metaclass=UOpMetaClass):
    ...
```

---

## 5. 典型应用：享元模式（Flyweight）缓存

### 场景
需要确保**相同参数构造的对象是同一个实例**，节省内存且方便比较。

### 实现方式
1. 定义元类，重写 `__call__`。
2. 在全局缓存中查找是否已存在相同键的对象。
3. 若存在则直接返回缓存对象，否则调用 `super().__call__` 真正创建并存入缓存。
4. 使用弱引用避免内存泄漏。

---

## 6. 典型案例：`tinygrad.UOp` 的元类实现

下面是基于 `tinygrad` 源码简化的完整示例，展示如何用元类实现全局结构化去重（单例效果）。

```python
import weakref

class UOpMetaClass(type):
    # 全局缓存：key -> weakref(实例)
    ucache: dict = {}

    def __call__(cls, op, dtype, src, arg, tag):
        # 1. 生成缓存键（所有影响唯一性的参数）
        key = (op, dtype, src, arg, tag)

        # 2. 尝试从缓存中获取
        weak_ref = cls.ucache.get(key)
        if weak_ref is not None:
            existing = weak_ref()
            if existing is not None:
                # 命中！直接返回已有实例（全局单例）
                return existing

        # 3. 未命中：调用父类 type.__call__ 真正创建实例
        new_obj = super().__call__(op, dtype, src, arg, tag)

        # 4. 存入缓存（弱引用，不阻止垃圾回收）
        cls.ucache[key] = weakref.ref(new_obj)

        return new_obj


class UOp(metaclass=UOpMetaClass):
    __slots__ = ('op', 'dtype', 'src', 'arg', 'tag')

    def __init__(self, op, dtype, src, arg, tag):
        self.op = op
        self.dtype = dtype
        self.src = src
        self.arg = arg
        self.tag = tag

    def __del__(self):
        # 对象销毁时从缓存中移除（可选）
        key = (self.op, self.dtype, self.src, self.arg, self.tag)
        UOpMetaClass.ucache.pop(key, None)


# ---------------- 使用示例 ----------------
class Ops:
    ADD = 1
    MUL = 2

class DType:
    float32 = 1

# 创建两个完全相同参数的 UOp
x = UOp(Ops.ADD, DType.float32, (None, None), None, None)
y = UOp(Ops.ADD, DType.float32, (None, None), None, None)

print(x is y)      # True（同一个对象实例，全局单例）
print(id(x) == id(y))  # True
```

### 为什么这样设计？

- **内存效率**：大型计算图中重复的子图（如相同操作、常量）只会存储一次。
- **比较效率**：判断两个 UOp 是否结构相等只需 `is` 比较，无需递归遍历 `src`。
- **自动清理**：弱引用使得不再被引用的节点可以被 GC 回收，缓存自动失效。
- **对用户透明**：使用者仍然用 `UOp(...)` 正常构造，无需调用工厂方法。

---

## 7. 总结

| 概念 | 说明 |
|------|------|
| `type` | 内置元类，所有类的类，也是动态创建类的工具 |
| `type(obj)` | 获取对象类型的函数形式 |
| `type(name, bases, dict)` | 动态创建类的元类形式 |
| `type.__call__` | 负责调用 `__new__` 和 `__init__` 的真正实例化入口 |
| 自定义元类 | 继承 `type`，重写 `__new__` 或 `__call__` 改变类的创建行为 |
| 享元模式 | 通过元类缓存相同参数的对象，保证全局唯一性 |

元类是 Python 区别于 C++/Java 的深度元编程能力。`UOp` 案例展示了如何在保持优雅 API 的同时，利用元类实现工业级的性能优化。