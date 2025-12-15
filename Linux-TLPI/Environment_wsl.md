# ✅ **方案 1（强烈推荐）—— Windows + WSL2 + GPU 完整步骤**

## **📌 第 0 步：确认你的系统**

- Windows 10（21H2+）或 Windows 11
    
- NVIDIA 独显
    
- BIOS 开启虚拟化（一般默认已开）
    

---

# **🔵 第 1 步：在 Windows 安装必要组件**

---

## **1. 安装 NVIDIA GPU 驱动（必须）**

你要安装的是 **正常的 Windows NVIDIA 驱动，不要安装 CUDA Toolkit！**

下载地址：

> NVIDIA GeForce / Studio Driver（按你显卡型号选择）

安装完成后：

`nvidia-smi`

如果能看到，就 OK。

---

## **2. 安装 VSCode**

- 下载 VSCode
    
- 安装 “Remote - WSL” 插件  
    用于直接编辑 WSL 的代码。
    

---

# **🔵 第 2 步：安装 WSL2（Ubuntu 22.04）**

在 PowerShell 以管理员运行：

`wsl --install -d Ubuntu-22.04`

等待 Ubuntu 安装完成后创建用户。

---

# **🔵 第 3 步：WSL2 内查看 GPU 是否可用**

进入 WSL：

`nvidia-smi`

如果能看到 GPU 信息 → GPU 直通已经正常。

---

# **🔵 第 4 步：在 WSL 安装 CUDA Toolkit（WSL 专用）**

⚠ 注意：  
WSL CUDA Toolkit **与 Windows CUDA Toolkit 不同**，不要混合安装！

执行：

## ① 更新系统 + 安装依赖
sudo apt update
sudo apt install -y wget gnupg lsb-release

## ② 添加官方 apt key & repo (network 方式更稳)

[CUDA Toolkit 12.4 Downloads | NVIDIA Developer](https://developer.nvidia.com/cuda-12-4-0-download-archive?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=22.04&target_type=deb_local)
`wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb`
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update

## ③ 安装 CUDA（toolkit + driver）
sudo apt install -y cuda        # 或 sudo apt install -y nvidia-cuda-toolkit

## ④ 配置环境变量
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc

## ⑤ 验证
nvcc --version
nvidia-smi

安装后：

`nvcc --version`

看到 CUDA 版本即成功。

---

# **🔵 第 5 步：安装 Nsight (可选，但建议)**

执行代码若有输出表明已经安装好了:
```bash
nsys --version
ncu --version
```

安装命令:
```bash
sudo apt install -y nvidia-nsight 
sudo apt install -y nvidia-nsight-compute
```

---

# **🔵 第 6 步：安装开发工具链**

### **CMake / Clang / Ninja**

## 🧩 **1. Clang 是什么？有什么作用？**

Clang 是 LLVM 项目的 **C/C++/Objective-C 编译器**，相当于 GCC 的替代品。

### ✔ **它是一个编译器**

作用和 `gcc` 一样：

- 编译 C / C++ 代码
    
- 生成目标文件和可执行文件
    
- 提供静态分析、警告信息
    
- 编译时优化（LLVM 优化非常强）
    

### ✔ **为什么很多项目更推荐 Clang？**

|特点|Clang|GCC|
|---|---|---|
|编译速度|快|较慢|
|错误信息|非常清晰、人类友好|相对不友好|
|支持最新标准|领先|略慢|
|和 LLVM 工具链集成|完美|无|
|静态分析能力|强（clang-tidy 等）|弱|

尤其在 **大型 C++ 项目（如 PyTorch、LLVM、Chrome、Eigen 开发者）** 中，Clang 编译速度更快，错误提示更友好，因此受到欢迎。

---

## 🏗️ **2. Ninja 是什么？有什么作用？**

Ninja 是一个 **极快的构建系统（build system）**，主要用于替代 GNU Make。

但它不是用来写构建脚本的，而是：

### ✔ **以“执行构建任务”为核心，速度极快**

特点：

- 多线程构建特别快
    
- 极少的 I/O 与开销
    
- 和 CMake 配合得很好
    

Ninja 不是 Make 的直接替代，而是一个 **低层次、速度优先** 的工具。

### ✔ 为什么 CMake + Ninja 是最佳组合？

因为 CMake 不直接编译，只生成构建文件。  
你可以让 CMake 为你生成：

- Makefile（给 make）
    
- Ninja build.ninja（给 ninja）
    
- Visual Studio 项目
    
- Xcode 项目…
    

但 **Ninja 比 Make 快得多**，例如：

|项目|Make 构建|Ninja 构建|
|---|---|---|
|PyTorch|慢很多|2-5 倍更快|
|TensorRT|更慢|快|
|Chromium|非常慢|只能用 Ninja|

因此几乎所有大型项目默认是：

👉 **cmake -G Ninja**  
意思是让 CMake 生成 Ninja 构建系统。

---

### 🧪 **简短总结**

|工具|作用|
|---|---|
|**Clang**|C/C++ 编译器，替代 GCC，语法错误提示友好，速度快，现代 C++ 支持好|
|**Ninja**|构建执行器，替代 Make，速度极快，和 CMake 配合最佳|

---
## 安装 CMake / Clang / Ninja（推荐一次性安装）

```bash
sudo apt update 
sudo apt install -y cmake clang ninja-build
```

安装内容：

- `cmake`
    
- `clang`, `clang++`
    
- `ninja`（包名：`ninja-build`）
    

---


## 🧪 **2. 验证安装成功**

### ✔ 验证 CMake

```
cmake --version
```

示例输出：

`cmake version 3.22.1`

---

### ✔ 验证 Clang

```
clang --version clang++ --version
```

示例输出：

`clang version 14.0.0 Target: x86_64-pc-linux-gnu`

---

### ✔ 验证 Ninja

```
ninja --version
```

示例输出：

`1.10.1`

---

## 🧪 **3. 测试三者是否协同工作（强烈推荐）**

测试一下：**CMake + Clang + Ninja 是否能一起运行**。

### ① 创建一个测试工程

`mkdir test_build cd test_build`

### ② 创建源文件

```
echo '#include <iostream> int main(){ std::cout << "Hello from Clang + Ninja!" << std::endl; }' > main.cpp
```

### ③ 创建 CMakeLists.txt

```
echo `cmake_minimum_required(VERSION 3.10) 
project(TestNinjaClang) 
set(CMAKE_CXX_COMPILER clang++) 
add_executable(test main.cpp)' > CMakeLists.txt
````

### ④ 使用 Ninja 生成构建文件

```
cmake -G Ninja .
```

输出（正确时）：

`-- The CXX compiler identification is Clang -- Build files have been written to: /path/test_build`

### ⑤ 构建

```
ninja
```

输出：

`[1/1] Linking CXX executable test`

### ⑥ 运行

`./test`

输出：

`Hello from Clang + Ninja!`

🎉 说明你的配置完全正确！

---

## **(1) 在的本地 Windows VSCode 安装 "clangd" 插件（最重要）**

clangd 是 VSCode 的语言服务器（LSP），必须通过 **VSCode 插件** 启动。

### ✔ 安装步骤（本地 Windows VSCode 操作）

1. 打开 VSCode（Windows 上）
    
2. 左侧 Extensions（扩展）面板
    
3. 搜索：**clangd**
    
4. 选择 `clangd`（作者 LLVM）
    
5. 点击 **Install**
    

安装后，VSCode 会自动在 WSL 中使用 clangd。

---

## ✅ **(2) 在 WSL 中安装 clangd（真正执行代码分析的地方）**

VSCode 只是 UI，**clangd 真正运行在你的 WSL Linux 里**。

在 WSL Terminal 执行：

`sudo apt update sudo apt install -y clangd`

验证：

`clangd --version`

输出类似：

`clangd version 14.0.0`

---

## 🧠 为什么 VSCode 需要双侧安装？

### 📌 Windows 安装

- VSCode 插件
    
- 提供 UI / 跳转 / 自动补全界面
    

VSCode 本身不编译 C++。

---

### 📌 WSL 安装

- 真正执行 clangd 后台服务器
    
- 分析你的 Linux 项目
    
- 理解 include、宏、CMake 构建等
    

当你打开 VSCode → 使用 “WSL: Ubuntu” 打开文件夹时：

VSCode 会自动调用：

`/usr/bin/clangd`

---

## ⚙️ **检查 VSCode 是否正确连接 WSL 的 clangd**

在 VSCode（打开 WSL 工程后）按：

`Ctrl + Shift + P`

输入：

`Developer: Show Running Extensions`

你应该看到：

`clangd language server (WSL)`

如果不是 WSL 版本，表示你打开了“Windows 文件夹”，需要正确打开。

---

## 📌 **如何正确打开工程？（非常重要）**

在 Windows VSCode 左下角：

✔ 显示绿色 “WSL: Ubuntu-22.04” 才是正确  
❌ 显示“Windows”或无绿色标签 → clangd 不会生效

如果不是绿色的：

1. `Ctrl + Shift + P`
    
2. 输入：**WSL: Reopen folder in WSL**
    

---

## 🧪 测试 clangd 是否工作

你的工程根目录必须有：

`compile_commands.json`

如何生成？

在你的 CMake 构建目录：

`cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..`

然后 clangd 能自动：

- 跳转到定义（F12）
    
- 智能补全
    
- 实时错误提示
    
- 自动 include
    
- 搜索符号
    
- 模板推导跳转

---

# **🔵 第 7 步：安装 Python + PyTorch（GPU 版）**

### 安装 Python 环境

`sudo apt install -y python3 python3-pip python3-venv`

### 创建虚拟环境

`python3 -m venv ~/env source ~/env/bin/activate`

### 安装 PyTorch（CUDA 12.1）

`pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121`

测试：

`import torch torch.cuda.is_available() torch.cuda.get_device_name(0)`

---

# **🔵 第 8 步：安装性能分析工具**

### perf（Linux 原生 perf）

`sudo apt install linux-tools-common linux-tools-$(uname -r)`

### strace

`sudo apt install -y strace`

### ftrace（内核自带）

无需安装，使用方式：

`sudo trace-cmd record ls`

### flamegraph

`sudo apt install -y linux-perf git clone https://github.com/brendangregg/FlameGraph ~/FlameGraph`

---

# **大功告成 🎉**

现在你拥有一个等价于原生 Linux 95% 的 GPU 计算环境：

- ✔ PyTorch + CUDA
    
- ✔ CMake / clang / ninja
    
- ✔ Nsight Compute / Systems
    
- ✔ perf / ftrace / strace / flamegraph
    
- ✔ VSCode + clangd 自动补全
    
- ✔ CUDA 能调用 GPU（nvidia-smi 正常）
    
- ✔ 能编译 C++/CUDA 项目