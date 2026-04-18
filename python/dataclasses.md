# Python 内置 dataclasses 详解

`dataclasses`是 Python 3.7+ 内置的标准库，用于自动为类添加特殊方法，减少编写样板代码的工作量。

## 基本用法

### 最简单的数据类

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

# 自动获得：
# 1. __init__(self, x: float, y: float)
# 2. __repr__(): Point(x=1.0, y=2.0)
# 3. __eq__(): 按值比较

p1 = Point(1.0, 2.0)
p2 = Point(1.0, 2.0)
print(p1)        # Point(x=1.0, y=2.0)
print(p1 == p2)  # True
print(p1.x)      # 1.0
```

### 带默认值

```python
@dataclass
class User:
    name: str
    age: int = 0
    active: bool = True

user1 = User("Alice")      # User(name='Alice', age=0, active=True)
user2 = User("Bob", 25)    # User(name='Bob', age=25, active=True)
user3 = User("Charlie", 30, False)
```

## 核心装饰器参数

```python
@dataclass(
    init=True,      # 生成 __init__ 方法（默认 True）
    repr=True,      # 生成 __repr__ 方法（默认 True）
    eq=True,        # 生成 __eq__ 方法（默认 True）
    order=False,    # 生成比较方法（<, <=, >, >=）（默认 False）
    unsafe_hash=False,  # 生成 __hash__ 方法
    frozen=False,   # 创建不可变实例（默认 False）
    match_args=True,  # 生成 __match_args__ 用于模式匹配
    kw_only=False,  # 所有字段必须关键字参数（Python 3.10+）
    slots=False,    # 生成 __slots__（Python 3.10+）
)
class MyClass:
    ...
```

## 字段控制：`field()`

`field()`函数用于精细控制每个字段：

```python
from dataclasses import dataclass, field
from typing import List, ClassVar

@dataclass
class InventoryItem:
    # 基本用法
    name: str
    
    # 默认值
    unit_price: float = 0.0
    quantity: int = 0
    
    # 复杂的默认值（使用工厂函数）
    tags: List[str] = field(default_factory=list)
    
    # 不包含在 __init__ 中
    computed_value: float = field(init=False, default=0.0)
    
    # 不包含在 repr 中
    internal_id: int = field(default=1, repr=False)
    
    # 不参与比较
    timestamp: float = field(default=0.0, compare=False)
    
    # 类变量（不是实例字段）
    tax_rate: ClassVar[float] = 0.1
    
    def __post_init__(self):
        """初始化后自动调用"""
        self.computed_value = self.unit_price * self.quantity
        
    @property
    def total_price(self):
        return self.computed_value * (1 + self.tax_rate)

# 使用示例
item = InventoryItem("Laptop", 999.99, 2)
print(item)  # InventoryItem(name='Laptop', unit_price=999.99, quantity=2, tags=[])
print(item.total_price)  # 2199.978
```

## 高级特性

### 1. 不可变类（frozen）

```python
@dataclass(frozen=True)
class ImmutablePoint:
    x: float
    y: float

point = ImmutablePoint(1.0, 2.0)
try:
    point.x = 3.0  # ❌ FrozenInstanceError
except:
    print("不能修改不可变对象")

# 替代方案：replace() 创建新实例
from dataclasses import replace
new_point = replace(point, x=3.0)  # ✅
```

### 2. 排序（order）

```python
@dataclass(order=True)
class Person:
    # 排序按字段定义顺序
    name: str = field(compare=False)  # 不参与排序
    age: int
    height: float

p1 = Person("Alice", 25, 1.65)
p2 = Person("Bob", 30, 1.75)
p3 = Person("Charlie", 25, 1.70)

print(p1 < p2)   # True (比较 age: 25 < 30)
print(p1 < p3)   # True (age 相同，比较 height: 1.65 < 1.70)
```

### 3. 模式匹配（Python 3.10+）

```python
@dataclass
class Command:
    action: str
    value: int

def handle_command(cmd: Command):
    match cmd:
        case Command(action="start", value=v):
            print(f"Starting with {v}")
        case Command(action="stop", _):
            print("Stopping")
        case _:
            print("Unknown command")
```

### 4. 槽位优化（Python 3.10+）

```python
@dataclass(slots=True)
class Point3D:
    x: float
    y: float
    z: float
    
# 等价于：
# class Point3D:
#     __slots__ = ('x', 'y', 'z')
#     ...dataclass 生成的方法...
```

## 实用函数

```python
from dataclasses import dataclass, fields, asdict, astuple, is_dataclass, replace

@dataclass
class Product:
    id: int
    name: str
    price: float

# 创建实例
product = Product(1, "Laptop", 999.99)

# 1. 获取字段信息
for f in fields(product):
    print(f"字段: {f.name}, 类型: {f.type}, 默认值: {f.default}")

# 2. 转换为字典/元组
print(asdict(product))   # {'id': 1, 'name': 'Laptop', 'price': 999.99}
print(astuple(product))  # (1, 'Laptop', 999.99)

# 3. 检查是否是数据类
print(is_dataclass(product))  # True
print(is_dataclass(Product))  # True

# 4. 创建修改后的副本
new_product = replace(product, price=899.99)  # Product(id=1, name='Laptop', price=899.99)
```

## 继承

```python
@dataclass
class Base:
    x: int
    y: int = 0

@dataclass
class Derived(Base):
    z: int = 0
    
    # 注意：字段顺序是父类在前
    # __init__ 签名: (x, y=0, z=0)

# 使用
d = Derived(1, 2, 3)  # Derived(x=1, y=2, z=3)
```

## 实际应用示例

### 配置管理

```python
from dataclasses import dataclass, field
from typing import Optional, List
from pathlib import Path
import json

@dataclass
class AppConfig:
    host: str = "localhost"
    port: int = 8080
    debug: bool = False
    database_url: Optional[str] = None
    plugins: List[str] = field(default_factory=list)
    
    @classmethod
    def from_json(cls, path: Path) -> "AppConfig":
        with open(path) as f:
            data = json.load(f)
        return cls(**data)
    
    def to_json(self, path: Path):
        with open(path, 'w') as f:
            json.dump(asdict(self), f, indent=2)

config = AppConfig(port=3000, debug=True)
print(config)  # 清晰的输出
```

### API 响应模型

```python
from dataclasses import dataclass, asdict
from datetime import datetime
from typing import Generic, TypeVar, List
import json

T = TypeVar('T')

@dataclass
class ApiResponse(Generic[T]):
    success: bool
    data: T
    message: str = ""
    timestamp: datetime = field(default_factory=datetime.now)
    
    def to_json(self) -> str:
        return json.dumps({
            **asdict(self),
            "timestamp": self.timestamp.isoformat()
        })

@dataclass
class User:
    id: int
    username: str
    email: str

# 使用
user_data = User(1, "alice", "alice@example.com")
response = ApiResponse(success=True, data=user_data, message="OK")
print(response.to_json())
```

### 科学计算

```python
from dataclasses import dataclass, field
import math
from typing import Tuple

@dataclass
class Vector3D:
    x: float
    y: float
    z: float
    
    def __add__(self, other: "Vector3D") -> "Vector3D":
        return Vector3D(self.x + other.x, self.y + other.y, self.z + other.z)
    
    def norm(self) -> float:
        return math.sqrt(self.x**2 + self.y**2 + self.z**2)
    
    @classmethod
    def from_tuple(cls, t: Tuple[float, float, float]) -> "Vector3D":
        return cls(*t)

@dataclass
class Particle:
    position: Vector3D
    velocity: Vector3D
    mass: float = 1.0
    
    @property
    def kinetic_energy(self) -> float:
        return 0.5 * self.mass * (self.velocity.norm() ** 2)

# 使用
v1 = Vector3D(1, 2, 3)
v2 = Vector3D(4, 5, 6)
p = Particle(v1, v2, mass=2.0)
print(f"动能: {p.kinetic_energy:.2f}")
```

## 与 attrs 对比总结

|特性|dataclasses|attrs|
|---|---|---|
|内置性|Python 3.7+ 标准库|需要安装|
|语法简洁性|很简洁|很简洁|
|验证器|无，需手动或使用 pydantic|内置强大验证|
|类型转换|无，需在 `__post_init__`处理|内置转换器|
|性能|标准|稍快|
|功能完整性|基础功能|更丰富功能|
|社区生态|标准库，广泛使用|成熟，许多项目使用|
|slots 支持|Python 3.10+|一直支持|

## 最佳实践

1. **使用类型注解**：充分利用 Python 类型系统
    
2. **合理使用默认值**：让类更易用
    
3. **善用 `field()`**：精细控制字段行为
    
4. **考虑不可变性**：`frozen=True`可避免意外修改
    
5. **添加业务方法**：dataclass 不只存储数据
    
6. **文档字符串**：为复杂的数据类添加文档
    

```python
@dataclass
class Customer:
    """客户信息"""
    
    id: int
    name: str
    email: str
    created_at: datetime = field(default_factory=datetime.now)
    
    def display_name(self) -> str:
        """返回显示名称"""
        return f"{self.name} <{self.email}>"
    
    def is_valid(self) -> bool:
        """验证客户信息是否有效"""
        return "@" in self.email and len(self.name) > 0
```

## 总结

`dataclasses`的核心优势：

1. **内置**：无需安装第三方库
    
2. **简洁**：大幅减少样板代码
    
3. **类型安全**：与类型注解完美配合
    
4. **功能实用**：覆盖了 80% 的常见需求
    
5. **性能良好**：生成的代码效率高
    

适用场景：

- 数据容器类
    
- 配置对象
    
- API 请求/响应模型
    
- 数据传输对象
    
- 简单的值对象
    

对于复杂验证、类型转换等高级需求，可考虑结合 `pydantic`或使用 `attrs`，但对于大多数简单场景，`dataclasses`完全够用且是 Python 官方推荐的方式。