
```mathematica
Layer 4 —— 高级接口（Matrix / Array / Block / Map）
Layer 3 —— Expression（CwiseUnaryOp / Product / etc）
Layer 2 —— Evaluator（CoreEvaluators）
Layer 1 —— 内核（SIMD/PacketMath, GEMM kernels）
Layer 0 —— Traits 系统（NumTraits, flags, scalar types）

```
## 1、存储+基础 API
---
- [ ] 1. DenseStorage.h
- 阅读重点：
- 动态矩阵如何管理内存
- m_data 指针的布局
- static/dynamic/partial-dynamic 三种配置方式

- [ ] 2.MatricBase.h
- 阅读重点：
- CRTP 结构
- coeff / coeffRef
- rows() / cols()- operator+ operator- operator*
- 一般算子接口的框架
你需要掌握：
- 如何为你的 Matrix/Tensor 设计统一 API
- 如何让操作分发到 Derived

- [ ] 3. PlainObjectBase.h
- 阅读重点：
- PlainObject 如何持有存储（DenseStorage）
- 构造/resize 的设计方式
你需要掌握：
- 区分“表达式”与“真正存储数据的对象” 

- [ ] 4.Matrix.h
阅读重点：
- Matrix 继承结构
- 默认构造 / 初始化方式
- Resize 与维度管理
你需要掌握：
- 如何写自己的 Matrix 类接口

- [ ] 4.DenseBase.h

## 2、表达式系统(真正构建矩阵计算)
---
- [ ] 5.CwiseUnaryOp.h / CwiseBinaryOp.h
- 阅读重点：
- operator+ 如何返回表达式对象
- 表达式节点如何存 lhs/rhs
- coeff(i,j) 的 lazy 计算方式
你需要掌握：
- 如何写自己的 Expression Template
- 如何返回轻量级表达式类型而不立即计算

- [ ] 6.Map.h-Mapbase.h
- 阅读重点：
- 如何将外部内存包装成 Matrix
- 引用视图，而不是复制你可以应用在：
- TensorMap / ArrayView 等设计

- [ ] 7.Block.h
- coeff(i,j) 如何做偏移
你可以借鉴：
- 你项目中的 Slice/Block 设计

- [ ] 8.Assign.h（必须）

- [ ] 9.Product.h(略)
- 略去Product 内核（GEMM micro-kernel）
## 3、Evaluator 执行系统

---
- CoreEvaluators.h
    
- BinaryOpEvaluator.h
    
- UnaryOpEvaluator.h
    
- ProductEvaluator.h
    
- MapEvaluator.h
    
- BlockEvaluator.h


## 4、优化内核，可选但建议
---
18. PacketMath.h（SIMD 内核）
    
19. GeneralBlockPanelKernel.h（矩阵乘法微内核）
    
20. Parallelizer.h


## ELSE:底层Traits 系统（最底层）— 类型与编译期属性
---
- [ ] 9.NumTraits.h（最低限度阅读）
- 阅读重点：
- 标量类型 Traits 设计哲学
你只需理解：
- 如何通过 traits 抽象标量的行为

- [ ] XprTraits.h / Meta.h（可选）

--- 
【完成阅读后的能力目标】

- 能完全理解 Matrix 是如何存储、访问、构建的
- 能写自己的 Matrix/Tensor 类
- 能写自己的 Unary/Binary 表达式模板
- 能用 CRTP 设计基类 API
- 能借鉴 Map/View 方式设计非拷贝视图类
- 为未来阅读 DenseBase 高级系统打下基础

------------------------------------------

【适合在你当前阶段重点精读的文件总结】

（按重要度排序）
1. MatrixBase.h
2. DenseStorage.h
3. PlainObjectBase.h
4. Matrix.h
5. CwiseBinaryOp.h / CwiseUnaryOp.h
6. Map.h
7. Block.h / Transpose.h（任选）

### OR
---
1. DenseStorage.h
2. PlainObjectBase.h
3. Matrix.h
4. DenseBase.h
5. MatrixBase.h
6. CwiseUnaryOp.h
7. CwiseBinaryOp.h
8. Block.h（选读）
9. Map.h（选读）
--------------------------
10. Assign.h（以后再看）


------------------------------------------