### 定长滑动窗口

```cpp
[-----|------|------]nums
    i-k+1   i
      |______|length=k
假设：循环体内部每次是待进入一个最前面的数状态
for(int i=0;i<nums.size();i++){
# 1. 进入
window += nums[i];

# window未满
int j= i-k+1;
if(j<0) continue;

# 2. 更新
window = max(window, threldhold);

# 3. 退出
window -= nums[j];
}
```