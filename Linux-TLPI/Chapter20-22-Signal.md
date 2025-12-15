
- 信号：
	- def：事件发生时对进程的通知机制，软件中断
	- 通常产生自内核，产生信号的事件如下：
		1. 硬件异常
		2. 用户键入特殊字符：如中断字符`Ctrl-C`暂停字符`Crtl-Z`等
		3. 软件事件：如定时器到期，进程执行CPU时间超限等
	- 在`<signal.h>`中以SIGxxx形式对信号进行了定义（从1开始的小整数）
	- 分类：
		1. 用于内核向进程通知事件：编号1-31，标准信号
		2. 实时信号
	- 等待：产生信号和到达期间
	- 信号的到达:
		1. 若进程在运行或内核要调度它运行，则等待信号马上送达
		2. 若要确保一段代码不为传递信号中断，可以将信号加入进程的信号掩码中
	- 信号掩码：阻塞信号的到达，直至将信号从掩码中移除
	- 进程接收信号的默认操作（执行其一）：
		1. 忽略信号
		2. 终止进程
		3. 产生核心转储文件，同时进程终止：包含对进程虚拟内存的镜像，可加载到调试器中检查进程终止时的状态
		4. 暂停进程
		5. 暂停中恢复进程
	- 信号处置设置：改变信号到达的响应行为（下列之一）
		1. 采取默认行为：恢复对信号处置的修改
		2. 忽略信号：适用于默认行为终止进程的信号
		3. 执行信号处理器程序
	- 信号处理器：程序员编写的函数，响应信号
	- （linux）`/proc/PID/status`文件掩码字段可以查看进程对信号的处理
	- 改变信号处置：
		1. `siganl()`:
		```cpp
		# include<signal.h>
		void (*signal(int sig,void (*hander)(int))(int);
		//信号处理程序
		void hander(int sig){/*处理代码*/}
		```
		2. `sigaction()`(signal()`无法在不改变信号处置的同时，获取当前信号处置。)
			```cpp
			int sigaction(int sig,const struct sigaction* act,struct sigaction* oldact);
			struct sigaction{
			void (*sa_handler)(int);//handler地址
			sigset_t sa_mask;//一组信号，调用handler处理器程序时阻塞
			int sa_flags;//位掩码，控制信号处理过程选项
			void (*sa_restorer)(void)//内部调用
			};
			```
	- 发送信号`kill()`:
		- 与shell的kill命令类似，进程能调用向另一进程发送信号。
		- 早期Unix实现多数信号默认终止进程，故称为kill
		- 使用：
			```cpp
			#include<signal.h>
			int kill(pid_t pid, int sig);
			```
		- 参数`pid`标识一个或多个目标进程:查阅4种情况
		- 权限：进程发送信号给另一进程需要适当权限：
			1. 特权级(CAP_KILL)进程可以向任何进程发送
			2. 以root用户和组运行的init进程，仅能接收已安装了处理器函数的信号（防止系统管理员意外杀死）
			3. ID匹配时非特权进程也能向另一进程发送信号![[killid.png]]
			4. `SIGCONT`信号特殊：无论用户ID检查，非特权进程均可向同一会话中任何其他进程发送
			5. 进程无权发送给请求pid将调用kill失败，且设置errno为EPERM
		- 检查进程的存在:
			1. 检查特定进程ID是否存在：使用kill设置参数sig为0（空信号）查看调用是否成功。（不能保证特定程序运行）
			2. `wait()`系统调用
			3. 信号量和排他文件锁
			4. 诸如管道和FIFO类的IPC通道
			5. `/process/PID`接口
		- 其他方式：
			- `raise()`
			- `killpg()`
	- 显示信号描述：每个信号的可打印说明
		- 位于数组`sys_siglist`中，`sys_siglist[SIGNAME]`调用
		- `strsignal()`函数
			```cpp
			#include<signal.h>
			extern const char * const sys_siglist[];
			char* strsignal(int sig);
			```
	- 信号集：
		- 表示多个信号的数据结构
		- 数据类型`sigset_t`
		- 操纵信号集的函数：
			- `int sigemptyset(sigset_t* set)`
			- `int sigfillset(sigset_t* set)`
			- `int sigaddset(sigset_t* set,int sig)`
			- `int sigdekset(sigset_t* set,int sig)`
			- `sigismember,sigandset,sigirset,sigisemptset`等
	- 信号掩码：内核为每个进程维护的一组信号，阻塞这些信号对进程的传递
		- 向掩码中添加信号：
			1. 调用信号处理器程序时，可将引发调用的信号自动添加
			2. 使用`sigaction()`建立信号处理器程序时，可以指定处理器程序的阻塞信号
			3. `sigprocmask()`系统调用随时显式添加或者移除信号
			- `int sigprocmask(int how,const sigset_t* set,sigset_t* oldset)`
			- 注:无法阻塞`SIGKILL,SIGSTOP`信号
	- 处于等待状态的信号：
		- 进程接受了一个进程正在阻塞的信号，会将该信号加入等待信号集中
		- 只是掩码，不会记录信号接受次数
		- 查看等待状态的信号:`int sigpending(sigset_t* set)`
	- 等待信号：
		- `pause()`暂停进程执行，直至信号处理函数中断该调用（或未处理信号终止进程）
	
	- 附：标准信号表：

| 信号名称      | 信号值 | 描述             | 默认行为        | SUS要求 |
| --------- | --- | -------------- | ----------- | ----- |
| SIGHUP    | 1   | 终端挂起或控制进程终止    | 终止进程        | 是     |
| SIGINT    | 2   | 键盘中断（Ctrl+C）   | 终止进程        | 是     |
| SIGQUIT   | 3   | 键盘退出（Ctrl+\）   | 终止进程并生成核心转储 | 是     |
| SIGILL    | 4   | 非法指令           | 终止进程并生成核心转储 | 是     |
| SIGTRAP   | 5   | 跟踪/断点陷阱        | 终止进程并生成核心转储 | 是     |
| SIGABRT   | 6   | 中止信号（abort()）  | 终止进程并生成核心转储 | 是     |
| SIGBUS    | 7   | 总线错误           | 终止进程并生成核心转储 | 是     |
| SIGFPE    | 8   | 浮点异常           | 终止进程并生成核心转储 | 是     |
| SIGKILL   | 9   | 强制终止           | 终止进程（不可捕获）  | 是     |
| SIGUSR1   | 10  | 用户自定义信号1       | 终止进程        | 是     |
| SIGSEGV   | 11  | 段错误（无效内存访问）    | 终止进程并生成核心转储 | 是     |
| SIGUSR2   | 12  | 用户自定义信号2       | 终止进程        | 是     |
| SIGPIPE   | 13  | 管道破裂（向无读端的管道写） | 终止进程        | 是     |
| SIGALRM   | 14  | 定时器信号（alarm()） | 终止进程        | 是     |
| SIGTERM   | 15  | 终止信号           | 终止进程        | 是     |
| SIGSTKFLT | 16  | 协处理器栈错误        | 终止进程        | 否     |
| SIGCHLD   | 17  | 子进程停止或终止       | 忽略          | 是     |
| SIGCONT   | 18  | 继续执行（如果停止）     | 继续/忽略       | 是     |
| SIGSTOP   | 19  | 停止进程           | 停止进程（不可捕获）  | 是     |
| SIGTSTP   | 20  | 终端停止（Ctrl+Z）   | 停止进程        | 是     |
| SIGTTIN   | 21  | 后台进程尝试从终端读     | 停止进程        | 是     |
| SIGTTOU   | 22  | 后台进程尝试向终端写     | 停止进程        | 是     |
| SIGURG    | 23  | 套接字紧急数据        | 忽略          | 是     |
| SIGXCPU   | 24  | CPU时间限制超出      | 终止进程        | 是     |
| SIGXFSZ   | 25  | 文件大小限制超出       | 终止进程        | 是     |
| SIGVTALRM | 26  | 虚拟定时器信号        | 终止进程        | 是     |
| SIGPROF   | 27  | 性能分析定时器信号      | 终止进程        | 是     |
| SIGWINCH  | 28  | 窗口大小改变         | 忽略          | 是     |
| SIGIO     | 29  | I/O就绪          | 终止进程        | 是     |
| SIGPWR    | 30  | 电源故障           | 终止进程        | 否     |
| SIGSYS    | 31  | 系统调用错误         | 终止进程并生成核心转储 | 是     |

---
- 信号处理器函数
	- 可能随时打断主程序；内核代表进程来调用处理器程序，返回时主程序会在断电恢复执行。
	![[信号处理器.png]]