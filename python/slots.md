在 Python 中，`__slots__`是一个特殊的类属性，用于**显式声明类的实例可以拥有哪些属性**，从而限制动态添加新属性，以节省内存。

## 基本用法

```python
class Person:
    # 声明这个类只能有 name 和 age 两个实例属性
    __slots__ = ('name', 'age')
    
    def __init__(self, name, age):
        self.name = name
        self.age = age

# 正常使用
p = Person("Alice", 25)
print(p.name)  # Alice
print(p.age)   # 25

# 不能添加新属性
try:
    p.email = "alice@example.com"  # ❌ AttributeError
except AttributeError as e:
    print(e)  # 'Person' object has no attribute 'email'
```

## 为什么需要 **slots**？

### 1. **内存优化**（主要用途）

普通类使用字典（`__dict__`）来存储实例属性，每个实例都有一个字典，会占用较多内存。`__slots__`会：

- 为每个属性创建固定的存储位置（描述符）
    
- 不使用 `__dict__`字典
    
- 适用于创建大量小对象（如数据库记录、科学计算）
    

**内存对比示例**：

```python
import sys

class WithDict:
    def __init__(self, x, y):
        self.x = x
        self.y = y

class WithSlots:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x = x
        self.y = y

obj_dict = WithDict(1, 2)
obj_slots = WithSlots(1, 2)

print(f"WithDict 实例大小: {sys.getsizeof(obj_dict) + sys.getsizeof(obj_dict.__dict__)} bytes")
print(f"WithSlots 实例大小: {sys.getsizeof(obj_slots)} bytes")
# 通常 WithSlots 小 30-50%
```

### 2. **属性访问速度稍快**

由于属性位置固定，访问速度比字典查找稍快。

## 重要特性

### 1. 继承行为

- 子类如果没有定义 `__slots__`，会自动拥有 `__dict__`
    
- 子类定义自己的 `__slots__`时，只包含自己声明的属性
    
- 要继承父类的 slots，需要显式包含：
    

```python
class Base:
    __slots__ = ('x',)
    
class Child(Base):
    __slots__ = ('y',)  # 不会继承 x
    # 实际只有 y 属性

class ChildWithAll(Base):
    __slots__ = Base.__slots__ + ('y',)  # 包含父类的 slots
    # 实际有 x 和 y 属性
```

### 2. 与 `__dict__`共存

如果在 `__slots__`中包含 `'__dict__'`，实例会同时拥有两种存储方式：

```python
class Mixed:
    __slots__ = ('x', '__dict__')
    
    def __init__(self, x, y):
        self.x = x
        self.y = y  # 可以动态添加，因为存在 __dict__

m = Mixed(1, 2)
print(m.x)  # 1 (在 slots 中)
print(m.y)  # 2 (在 __dict__ 中)
m.z = 3     # 也可以，在 __dict__ 中
```

### 3. 弱引用支持

默认使用 `__slots__`的类不支持弱引用，需要在 `__slots__`中包含 `'__weakref__'`：

```python
import weakref

class NoWeakref:
    __slots__ = ('x',)
    # 不支持 weakref.ref(obj)

class WithWeakref:
    __slots__ = ('x', '__weakref__')
    # 支持弱引用
```

## 限制和注意事项

1. **不能动态添加属性**（除非包含 `'__dict__'`）
    
2. **所有实例共享同一个 slots 定义**
    
3. **不能对 slots 属性使用默认值**（需要在 `__init__`中初始化）
    
4. **可能会破坏 pickle 序列化**（需要额外处理）
    

```python
class Problematic:
    __slots__ = ('x', 'y')
    
    # 错误：不能在类级别给 slots 属性赋值
    # x = 10  # ❌
    
    def __init__(self):
        self.x = 10  # ✅ 必须在 __init__ 中初始化
        # self.y 未初始化，访问会报错
```

## 使用场景

### 推荐使用：

- 内存敏感的应用程序（数百万个实例）
    
- 作为简单的数据容器（类似 C 的结构体）
    
- 需要严格控制实例属性的场景
    

### 不建议使用：

- 需要动态添加属性的类
    
- 大多数普通业务逻辑类
    
- 复杂的继承层次结构
    

```python
# 适合使用 slots 的场景
class Point3D:
    """三维点，创建数百万个这样的点"""
    __slots__ = ('x', 'y', 'z')
    
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

# 不适合使用 slots 的场景
class DynamicConfig:
    """需要动态添加配置项"""
    # 不使用 slots，保留 __dict__
    def __init__(self):
        self._data = {}
    
    def set(self, key, value):
        setattr(self, key, value)  # 需要动态添加属性
```

## 总结

|特性|有 `__slots__`|无 `__slots__`|
|---|---|---|
|内存使用|较小|较大（每个实例有 `__dict__`）|
|属性访问速度|稍快|稍慢|
|动态添加属性|不允许（除非包含 `'__dict__'`）|允许|
|实例大小|固定|可变|
|内存布局|紧凑数组|哈希表（字典）|
|适用场景|大量小对象、数据结构|普通业务对象|

**简单来说**：`__slots__`是用空间换灵活性的优化工具。在不需要动态添加属性且关注内存时使用，否则让 Python 使用默认的 `__dict__`机制更简单方便。