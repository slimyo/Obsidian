- 进程的创建
	- `fork()`创建新进程：
		- 调用创建子进程，其拥有相同程序文本段，不同栈段、数据段及堆段（初始为父进程的拷贝）。
		- 习惯调用方法：
			```cpp
			pid_t childPid;
			switch(childPid=fork()){
			case -1://调用失败
				//error
			case 0://fork 在子进程中返回0
				//child action;
			default://fork 在父进程中返回子进程pid
				//parent action;
			}
			//child and parent actions;
			```
		- 父子进程的文件共享：执行`fork()`时，子进程会获得父进程所有文件描述符的副本
		- 