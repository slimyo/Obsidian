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