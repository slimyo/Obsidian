# ✅ TLPI 指标体系


| 章节号 (Chapter)                                                                                                   | 内容 / 模块                                                                                                     | 推荐理由                                                                      |
| --------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| **4–5**                                                                                                         | 文件 I/O 基础 & 深入 (open/read/write/close/fcntl / pread/pwrite / scatter-gather / nonblocking I/O / large file) | 基本 I/O，是所有系统 + 数据加载/存储 + 日志 + 文件交互基础                                      |
| **6**                                                                                                           | 进程 (process basics)                                                                                         | 理解进程模型、虚拟地址空间、命令行／环境／堆栈布局，对任何系统级/服务级都基础                                   |
| **24–28**                                                                                                       | 进程创建 / exec / fork / clone / process termination / wait / 子进程管理                                             | dataloader、worker pool、多进程服务 / 分布式启动等，都强依赖 fork/exec/execve/clone         |
| **29–33**                                                                                                       | 线程 (pthread)、同步、thread local storage                                                                        | 多线程 I/O、异步任务、并发 kernel、线程池等必须                                             |
| **49–50**                                                                                                       | Memory mapping / Virtual memory operations (mmap, mprotect, mlock, madvise, msync, shared memory mapping)   | 对大模型 / GPU pinned memory / 数据预处理 / zero-copy IO 非常关键                      |
| **56–60**                                                                                                       | Socket API + TCP/UDP 网络基础 + Server 设计                                                                       | 用于 RPC / inference server / distributed training / parameter server 等网络通信 |
| **60 (I/O 多路复用 / 服务器设计)**                                                                                       | epoll / select / poll / server model                                                                        | 构建高性能网络服务 (如 LLM inference 服务) 的基础                                        |
| **20–22 / 23**                                                                                                  | 信号 (signal) + 计时器 / sleep /定时器                                                                              | 对后台任务、异步 I/O、超时、资源管理、定时任务管理重要                                             |
| **43–48 / IPC / Pipe / FIFO / System V / POSIX IPC / Shared Memory / Semaphore / Message Queue / File Locking** | 进程间通信 + 共享内存 + 锁机制                                                                                          | 对多进程 / 多 worker / IPC / 数据共享 /锁机制有用 (尤其在 HPC、训练/推理系统)                     |


---

## **📘 Chapter 4–5：文件 I/O 基础 + 深入**

### 🎯 学习指标（你最终要做到）

- 熟练使用 `open/read/write/close`
    
- 熟练区分
    
    - 阻塞 / 非阻塞 open
        
    - O_APPEND / O_SYNC / O_DIRECT
        
- 能解释文件描述符表、打开文件描述、inode table 的关系
    
- 能使用：
    
    - `pread/pwrite`（无移动文件偏移）
        
    - `lseek`
        
    - `fcntl` 管理 fd flag、锁
        
    - `readv/writev` scatter–gather I/O
        

### 📝 输出（学习笔记）

- 文件描述符的生命周期（进程表 → 内核表 → inode）
    
- 画一张“fd -> open file desc -> inode”图
    
- 常用 flag 总表
    
- pread/pwrite 的实际使用场景
    

### 🧪 实操项目（必须会写）

1. **实现 cat 的简化版**
    
    - 使用 `read/write`
        
    - 支持 `-n` 输出行号
        
2. **实现一份二进制日志写入器**
    
    - 使用 O_APPEND + O_SYNC
        
    - 支持 rotate（大小超过 1MB 自动重命名）
        
3. **实现 scatter-gather 文件复制**
    
    - 用 `readv/writev` 读写文件
        

---

## **📘 Chapter 6：进程基础**

### 🎯 学习指标

- 能解释进程的：
    
    - PID / PPID
        
    - 命令行参数
        
    - 环境变量布局
        
    - 栈 & 堆的创建时机
        
- 能操作进程表环境（`getenv`, `setenv`, `environ`）
    

### 📝 输出

- 写出“典型 Linux 进程内存布局图”
    
- 写出 argc/argv/environ 是如何从 exec 传入的
    

### 🧪 项目

1. **实现 printenv**
    
    - 列出所有环境变量
        
2. **实现 which**
    
    - 用 `getenv("PATH")`
        
    - 搜索可执行文件
        

---

## **📘 Chapter 24–28：进程创建 / exec / fork / clone / wait**

### 🎯 学习指标

- 熟悉 `fork`、COW 原理
    
- 能解释：`fork → exec → wait`
    
- 能使用 `clone` 创建 share-mem 或 share-file-descriptor 轻量进程
    
- 熟练进程回收，避免僵尸进程
    

### 📝 输出

- fork 前后虚拟内存布局对比图
    
- 僵尸进程产生 & 回收机制总结
    

### 🧪 项目

1. **实现迷你 Shell**
    
    - 支持：
        
        - 输入命令
            
        - fork + exec 执行
            
        - 管道：`cmd1 | cmd2`
            
        - 后台任务：`cmd &`  
            ——这是学习 TLPI 最经典的项目。
            
2. **多进程 worker pool**
    
    - 使用 fork 多 worker
        
    - 使用 pipe/master 派发任务
        
    - 使用 wait 回收子进程
        

---

## **📘 Chapter 29–33：线程 + 同步**

### 🎯 学习指标

- 熟练 pthread API：`pthread_create / join`
    
- 熟悉互斥锁、条件变量、读写锁、thread-local-storage
    
- 能解释 data race / memory model
    

### 📝 输出

- 总结每种锁适用场景
    
- 画出生产者-消费者模型图
    

### 🧪 项目

1. **线程安全队列（blocking queue）**
    
    - pthread_mutex + pthread_cond
        
    - 支持 push/pop/timeout
        
2. **多线程日志系统**
    
    - 主线程 enqueue
        
    - worker 线程异步写文件
        

---

## **📘 Chapter 49–50：内存映射 / 虚拟内存**

### 🎯 学习指标

- 熟练使用 `mmap`、`mprotect`、`munmap`
    
- 区分匿名映射/文件映射
    
- 能解释 page fault、copy-on-write、page cache、dirty page flush
    
- 能说明 mlock/madvise 的使用场景
    

### 📝 输出

- 一张 page cache 生命周期图（mmap → page fault → major fault → dirty → flush）
    

### 🧪 项目

1. **实现 mmap 版大文件读写器**
    
    - 比较 mmap vs read/write 性能
        
2. **实现持久化共享内存 KV store**
    
    - 使用 mmap + msync 做 mini-redis
        

---

## **📘 Chapter 56–60：Socket、TCP/UDP、服务器基础**

### 🎯 学习指标

- 熟悉 socket API：`socket/bind/listen/accept/connect`
    
- 理解 TCP 三次握手 + TIME_WAIT
    
- 能写同步/异步网络通信
    

### 📝 输出

- TCP 状态机图
    
- HTTP 请求报文例子与解析流程
    

### 🧪 项目

1. **实现 echo server（TCP）**
    
    - server 多线程或多进程版本都写一遍
        
2. **实现一个简单的 HTTP server**
    
    - 返回静态文件
        
    - 支持 keep-alive
        

---

## **📘 Chapter 60（IO 多路复用）**

### 🎯 学习指标

- 熟悉 select/poll/epoll 区别
    
- 能写**多路复用的高性能服务器**
    
- 理解 “水平触发 / 边缘触发” 差异
    

### 📝 输出

- epoll ET 与 LT 对比表
    
- 描述 Reactor、Proactor 架构
    

### 🧪 项目（推荐必须写）

- **epoll + 非阻塞 I/O 版本的 HTTP 服务器（tiny HTTP server）**
    
    - 支持持久连接
        
    - 解析 HTTP header
        
    - 返回文件内容
        
    - CPU 单核下可以撑 20k QPS 以上
        

这是未来你写**LLM 推理服务器**最核心的能力。

---

## **📘 Chapter 20–23：信号 / 定时器**

### 🎯 学习指标

- 熟悉 signal 机制
    
- 使用 sigaction 建立可靠信号处理器
    
- 使用 timerfd 或 POSIX timer 实现定时任务
    
- 避免 reentrant bugs
    

### 📝 输出

- 可重入函数表整理
    
- signal-disposition 总结图
    

### 🧪 项目

1. **定时器驱动的任务调度器**
    
    - 使用 timerfd
        
    - 支持 N 秒执行一次任务
        
2. **实现 Ctrl+C 安全退出 server**
    
    - graceful shutdown
        

---

## **📘 Chapter 43–48：IPC / 共享内存 / 消息队列 / 信号量**

### 🎯 学习指标

- 熟悉无名管道、命名管道、SysV / POSIX IPC 对比
    
- 熟练使用共享内存 + 信号量构建多进程同步
    
- 能写一个多进程间的生产者-消费者模型
    

### 📝 输出

- 列表：各种 IPC 的延迟/吞吐对比表
    
- 管道缓冲区大小原理（内核可调）
    

### 🧪 项目

1. **共享内存消息队列（多进程版）**
    
    - 使用 shm + sem
        
    - 可测延迟（微秒级）
        
    - 倍速比 pipe 快
        

---

# 🎓 总结（给你一份最终目标）

当你按上面模块读完 TLPI，你应该能**输出与掌握**：

## ✔ 清晰的 TLPI 学习文档

（包含：fd 管理 → fork/exec → mmap → epoll → IPC 全链路）

## ✔ 实作十几个系统编程项目

最终你应该会**从零实现**：

### 🔥 _一个基于 epoll 的高性能 HTTP 服务器_

- 非阻塞 socket
    
- epoll ET
    
- mmap 文件服务
    
- 信号安全退出
    
- 多线程 / 多 worker
    
- 支持 keep-alive
    

> 这个项目是 TLPI 学习的最强证明。