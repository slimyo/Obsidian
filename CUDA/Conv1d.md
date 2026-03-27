
# Conv1D 版本解析

## 版本1：基本正确实现思路

### **核心思想：分块计算 + 数据复用**

```markdown
输入数据: [a0, a1, a2, a3, a4, a5, a6, a7, a8, a9, a10, ...]
卷积核:    [b0, b1, b2]  (window=3)
```

### **步骤分解：**

#### 1. **共享内存布局规划**

```cpp
extern __shared__ int smem[];
int* a_shared = smem;                     // 输入数据块
int* b_shared = &smem[TILE_SIZE + window - 1];  // 卷积核
```

内存布局：

```markdown
[ 输入数据块 (TILE_SIZE + window-1) | 卷积核 (window) ]
     ↑                                  ↑
    a_shared                          b_shared
```

#### 2. **为什么需要 `TILE_SIZE + window - 1`？**

考虑卷积计算：

```markdown
输出位置 i 需要: a[i], a[i+1], a[i+2]  (window=3)
线程0计算: a0*b0 + a1*b1 + a2*b2
线程1计算: a1*b0 + a2*b1 + a3*b2
...
线程T计算: aT*b0 + aT+1*b1 + aT+2*b2
```

**问题**：线程T需要`a[T+2]`，但下一个block的第一个线程需要`a[T+1]`计算

**解决方案**：每个block多加载`window-1`个元素（halo区域）

#### 3. **数据加载策略**

```markdown
Block 0 加载: [a0, a1, a2, a3, a4, a5, a6]  (TILE_SIZE=4, window=3)
              ↑ 主区域 (4个)  ↑ halo (2个)
              
Block 1 加载: [a3, a4, a5, a6, a7, a8, a9]  (从a3开始，包含重叠)
              ↑ 重叠的halo
```

**关键**：相邻block有`window-1`个元素重叠，避免全局内存重复访问

#### 4. **计算流程**

```markdown
线程0: shares[0]*b_shared[0] + shares[1]*b_shared[1] + shares[2]*b_shared[2]
线程1: shares[1]*b_shared[0] + shares[2]*b_shared[1] + shares[3]*b_shared[2]
线程2: shares[2]*b_shared[0] + shares[3]*b_shared[1] + shares[4]*b_shared[2]
线程3: shares[3]*b_shared[0] + shares[4]*b_shared[1] + shares[5]*b_shared[2]
```

**共享内存访问模式**：每个线程访问连续地址，合并访问