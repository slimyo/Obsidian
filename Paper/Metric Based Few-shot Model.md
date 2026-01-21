# **Prototypical Networks for Few-shot Learning**

---
## 思想
- 使用*episodes*：样本的一个小批量子集，包含数个数据点，支撑集（`support set`)和测试集(`querry set`)。进行训练。
- 存在嵌入空间（*Embedding Space*）:在这个空间上，每一类的数据点都包括了一个原型表征(*prototype presentation*)
- 模型学习嵌入函数，寻找如何将数据点映射到嵌入空间
- 让支撑集中每个类的原型表征为类内所有点在嵌入空间的平均值
- 分类任务就是:寻找测试集合的点在嵌入空间上最接近的一个原型表征，其类别为该原型表征所属类

---
## 模型

设episode集：$$S=\{(X_i,y_1),...,(X_N,y_N)\}$$$$X_i\in \mathbb R^D,y_i\in\mathbb R^k$$模型计算一个M维的特征表示(*prototype*):$$c_k\in \mathbb R^M$$通过(模型参数为$\phi$):$$f_{\phi}:\mathbb R^D\to \mathbb R^M$$则：$$c_k=\frac{1}{|S_k|}\sum_{(X_i,y_i)\in S_k}f_\phi(X_i)$$
给定距离度量:$$d:\mathbb R^M\times \mathbb R^M\to[0,+\infty)$$网络根据距离计算输入（*querry*）在各类的概率：$$p_\phi(y=k|X)=\frac{exp(-d(f_\phi(X),c_k))}{\sum_{k'}exp(-d(f_\phi(X),c_{k'}))}$$训练目标:$$min J(\phi)=-log p_\phi(y=k|x)$$

---