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
		- 系统调用`vfork`
		- `fork()`之后的竞争条件:
			- 调用`fork()`后，无法确定父、子进程间谁率先访问CPU（父子进程顺序依据系统可能不一样）
- 进程的终止
	- `_exit()`和`exit()`：
		- `_exit(int status)`总是终止该进程，无返回值，父进程可通过`wait()`获取`status`终止状态（仅低8位有效）
		- `exit()`是将其包装的C语言库函数，含有调用退出处理程序、刷新stdio缓冲区
		- 从main函数中return也是终止程序的方法（只有在退出处理过程中要用到main函数本地变量的情况下，其与exit调用不同）
	- 进程终止的细节：
		- 关闭文件描述符......
	- 退出处理程序
		- `atexit()`:注册无传参、无返回值的处理函数
		- `on_exit()`:注册有exit参数status和参数args的处理函数
		- 先执行后注册的处理程序
		- `fork()`创建的子进程会继承父进程的处理函数(`exec()`函数会替换掉)
	- `fork()`,`stdio`缓冲区`_exit()`间的交互:
		- `fork()`创建的进程会复制缓冲区
		- `fflush()`可以用来刷新缓冲区，在fork前使用避免重复输出
		- `setvbuf()`,`setbuf()`可以用来关闭stdio流缓冲
		- 子进程调用`_exit()`可以不再刷新缓冲区：一个设计方法，创建子进程的应用中，只有一个进程调用`exit()`,其他调用`_exit()`
- 监控子进程
	- 