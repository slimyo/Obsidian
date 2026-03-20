### 二分查找

```cpp
[#######-#####+#####-########]
        |     |     |
        L     M     R
        
```
- 注意：
	1. 看区间，左开右闭？
	2. 看分界条件，大于等于？小于等于？
	3. 看循环不变量：Left，right区间内值
	4. 看收敛：`Left==right？
	5. 看返回值：return left？
	6. 返回得到的0？-1？nums.size()-1？nums.size()?k?
`
- 区间问题：
	- 设left=0;right=len-1;
		- 保持每次待查找区间内值都是要查验的（闭区间）
		- 假设分为的两个区间是**大于等于**和**小于**
		- 查找**小于target**的区间：若n\[m\]>=target,则更新R=M-1(一定要减一，中间位置判断过了)
		- 查找**大于等于target**的区间：若n\[m\]<target,则更新L=M+1(一定要加一，中间位置判断过了)
		- 循环不变量：
			- R+1一定大于等于target
			- L-1一定小于target
			- 循环结束时，R+1=L
			- 循环内L和R仅仅代表要检索的区间
		- 注意：如果 `left`和 `right`都很大，比如接近 32 位整数的最大值（2³¹ − 1），那么 `left + right`可能会超过最大表示范围，导致溢出错误。
			一种安全的写法是：
			```c
			int center = left + (right - left) / 2;
			```
	- 代码：
	- ```cpp
	  int lowerbound(vector<int>&nums,int target){
	  //返回最小nums[i]>=target的i	  
		  int left=0;
		  int right=nums.size()-1;//闭区间
		  while(left<=right){//区间不为空
			  int center = (left+right)/2;
			  if(nums[center]>=target){//关键不在此大于等于号--这是假设分区间的方法
				  //[l,mid-1]
				  right = center-1;//关键在范围更新
			  }
			  else{
				  //[mid+1,r]
				  left = center+1;
			  }
		  }
		  return left;
	  }
	  ```
	- 设left=0;right=len;
		- 左闭右开
			- 若n\[m\]>=target,则更新R=M
			- 若n\[m\]<target,则更新L=M+1