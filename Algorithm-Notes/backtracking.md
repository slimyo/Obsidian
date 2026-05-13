- 经常对nums先排序，方便去重
# 1.for穷举
```cpp
int size = nums.size();
vector<int> path;
vector<vector<int>> ans;

1.集合类型
auto dfs = [&](this auto&&dfs,int i)->void{
	//下面没有加入答案，就这里：ans.push_back(path);
	if(i==size){
		// 有些可以在这里 ans.push_back(path);
		return;
	}
	
	//本层内选择[i,size-1]的一个
	for(int start=i;start<size;start++){
		int x = nums[start];
		if(x条件){
		path.push_back(x);//选择了
		ans.push_back(path);// 集合类型题目，每次选择都加入答案
		dfs(start+1);//下一层，从[start+1,size-1]选择
		path.pop_back();//回溯
		}
	}
};

2. 排列类型--显示使用i的方法更好，减少push、pop操作
vector<bool> used(size);//是否选择了
auto dfs = [&](this auto&&dfs,(可以加上int i))->void{
	//if(i==size){//这里i表示层次，第几次选择了
	//或者
	if(path.size()==size){
		ans.push_back(path);
		return;
	}
	
	//本层内选择
	for(int start=0;start<size;start++){
		int x = nums[start];
		if(!used[start]){//没有选择过才进入
		used[start] = true;
		path.push_back(x);//选择了--path[i] = x;
		dfs();//下一层dfs(i+1);
		path.pop_back();//回溯
		used[start] = false;
		}
	}
};
```

# 2.选或不选
```cpp
int size = nums.size();
vector<int> path;
vector<vector<int>> ans;
auto dfs = [&](this auto&&dfs,int i)->void{//常有标志
	if(i==size){
		// 结束
		ans.push_back(path);
		return;
	}
	
	int x = nums[start];
	
	//选择
	// 满足条件则可选i
	if(x条件){
		path.push_back(x);//选择了
		//ans.push_back(path);// 集合类型题目，每次选择都加入答案
		dfs(i+1);//下一次
		path.pop_back();//回溯
	}
	
	//不选择
	//条件去重--用上一次的选择和这次待选进行比较
	//如果不一样则可以不选择--4种情况：都选，都不选，选上一个，选i
	//去掉选i的这一个重复选项:
	//都选:last == nums[i]，如果选了last，则表明自动都选了
	//都不选：last不会等于这个数值，则这里不选即可
	//只选一个：last没选，则不会等于nums[i]，则这里选即可
	if(条件){//last!=nums[i]
		dfs(i+1);
	}
};
```