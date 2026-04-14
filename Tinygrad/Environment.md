Miniforge 是目前最轻量、最推荐的 conda 替代方案。它自带 **conda + mamba**（mamba 比 conda 快很多），默认使用 conda-forge 通道，非常适合你后续做多个 ML/AI 项目（tinygrad、PyTorch、JAX、Hugging Face 等）。

### 1. 在 WSL 中安装 Miniforge（一步一步执行）

在 WSL 的 Ubuntu 终端中，**逐条复制运行**：

Bash

```
# 1. 更新系统
sudo apt update && sudo apt upgrade -y

# 2. 安装必要工具（如果还没装）
sudo apt install -y curl wget git

# 3. 下载最新 Miniforge3（自动适配 x86_64）
cd ~
wget https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh

# 4. 给执行权限并安装（推荐安装到默认 ~/miniforge3）
chmod +x Miniforge3-Linux-x86_64.sh
./Miniforge3-Linux-x86_64.sh

# 安装过程中会询问：
# - Do you accept the license terms? → 输入 yes
# - Install location → 直接按 Enter（默认 ~/miniforge3）
# - Do you wish to update your shell profile? → 输入 yes（重要！）
```

- 注
```
# 先创建存放目录（推荐放在 home 下） 
mkdir -p ~/Downloads

# 把 Windows D 盘的 Miniforge 文件复制到 WSL 的 ~/Downloads cp /mnt/d/wsl/share/Miniforge3-Linux-x86_64.sh ~/Downloads/
```
**注意**：
- /mnt/d/ 就是 Windows 的 **D: 盘**（WSL 会自动挂载所有 Windows 盘符）。

安装完成后，**重启终端**（关闭当前 WSL 窗口，重新打开），或者手动执行：

Bash

```
source ~/.bashrc
```

### 2. 安装后的基础配置（强烈推荐）

运行以下命令进行优化：

Bash

```
# 1. 验证安装成功
conda --version
mamba --version   # 应该都能看到版本号

# 2. 关闭 base 环境自动激活（避免每次打开终端都进入 base）
conda config --set auto_activate_base false

# 3. 设置常用配置（加快下载、优先 conda-forge）
conda config --add channels conda-forge
conda config --set channel_priority strict
conda config --set solver libmamba   # 使用更快的新 solver

# 4. 更新 conda 和 mamba
mamba update --all -y
```

以后每次新开终端，直接用 conda activate xxx 切换环境即可，不会自动进入 base。

### 3. 为 tinygrad 创建专用环境（推荐做法）

Bash

```
# 创建一个名为 tinygrad 的环境，使用 Python 3.12（tinygrad 支持良好）
mamba create -n tinygrad python=3.12 -y

# 激活环境
conda activate tinygrad

# 在这个环境中安装 tinygrad（可编辑模式，方便你修改源码）
cd ~/projects/tinygrad          # 替换成你实际的 tinygrad 目录
pip install -e .

# 额外安装 Jupyter 支持（后面连接 Notebook 很方便）
mamba install jupyterlab ipykernel matplotlib -y
python -m ipykernel install --user --name tinygrad --display-name "Python (tinygrad)"
```

### 4. 后续多个项目如何管理（最佳实践）

为你不同的项目创建独立环境，避免依赖冲突：

Bash

```
# 示例1：另一个 PyTorch 项目
mamba create -n torch-project python=3.12 pytorch torchvision torchaudio pytorch-cuda=12.4 -c pytorch -c nvidia -y

# 示例2：一个 LLaMA / Hugging Face 项目
mamba create -n llm-project python=3.12 -y
conda activate llm-project
pip install transformers accelerate bitsandbytes

# 示例3：纯数据科学项目
mamba create -n data python=3.12 pandas numpy scipy scikit-learn seaborn -y
```

**常用命令速查**：

- 列出所有环境：conda env list
- 激活环境：conda activate 环境名
- 退出环境：conda deactivate
- 删除环境：conda env remove -n 环境名
- 导出环境（方便备份或分享）：conda env export > environment.yml
- 从 yml 文件创建环境：mamba env create -f environment.yml

### 5. 与 Jupyter Notebook / VS Code 配合

- 在对应环境中激活后，直接运行 jupyter lab 即可。
- VS Code + Remote-WSL：打开项目文件夹后，选择对应的 kernel（tinygrad、torch-project 等），会自动识别。

### 6. 小贴士 & 注意事项

- **优先用 mamba**：创建/安装环境时尽量用 mamba 而不是 conda，速度快很多。
- **GPU 支持**：NVIDIA 在 WSL 下用 CUDA 时，不需要在 conda 里装 CUDA Toolkit（用系统驱动即可）。tinygrad 的 GPU 后端会自动检测。
- **磁盘空间**：多个环境会占用空间，定期 mamba clean --all 可以清理缓存。
- **更新 Miniforge**：以后升级时，重新下载最新 .sh 文件安装覆盖即可。
- 如果安装过程中出现网络问题（你在东京），可以临时用清华镜像：
    
    Bash
    
    ```
    conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge
    ```