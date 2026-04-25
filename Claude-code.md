配置WSL+VSCode+Claude-code
先进行网络配置，让wsl使用windows代理镜像
### 阶段一：配置 `.wslconfig`（Windows 侧）

在 PowerShell 里执行：

powershell

```powershell
notepad $env:USERPROFILE\.wslconfig
```

粘贴以下内容保存：

ini

```ini
[wsl2]
networkingMode=mirrored
dnsTunneling=true
autoProxy=true
firewall=true
memory=8GB          # 按你实际内存调整，建议不超过物理内存一半
processors=4        # 分配的 CPU 核心数
```

然后重启 WSL：

powershell

```powershell
wsl --shutdown
wsl
```

---

### 阶段二：WSL2 Ubuntu 环境配置

#### 3.1 系统更新

bash

```bash
sudo apt update && sudo apt upgrade -y
```

#### 3.2 安装基础开发工具

bash

```bash
sudo apt install -y \
  build-essential git curl wget \
  cmake ninja-build \
  python3-pip python3-venv \
  tmux htop tree
```

#### 3.3 安装 Node.js（用 NVM，便于管理版本）

bash

```bash
# 安装 NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# 重新加载配置
source ~/.bashrc

# 安装最新 LTS 版本
nvm install --lts
nvm use --lts

# 验证
node --version   # 应该显示 v22.x.x
npm --version
```

#### 3.4 配置代理环境变量（把端口改成你实际的代理端口）

bash

```bash
nano ~/.bashrc
```

在文件末尾添加：

bash

```bash
# ── Proxy ──────────────────────────────────────
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
export all_proxy="socks5://127.0.0.1:7890"
export no_proxy="localhost,127.0.0.1,::1"
# ───────────────────────────────────────────────
```

保存后：

bash

```bash
source ~/.bashrc

# 验证代理生效
curl -I https://api.anthropic.com  # 应该看到 cloudflare 的响应
```

#### 3.5 安装 Claude Code

bash

```bash
npm install -g @anthropic-ai/claude-code

# 验证安装
claude --version
```

#### 3.6 登录 Claude Code

bash

```bash
claude
```

会自动弹出浏览器，用你的 Claude 账号登录（Pro/Max/Team 任一即可）。登录完成后回到终端，出现对话提示符即成功。

---

### 阶段四：VSCode 集成配置

#### 4.1 安装 VSCode 扩展

在 Windows 的 VSCode 里安装：

- **WSL**（微软官方，扩展 ID：`ms-vscode-remote.remote-wsl`）

#### 4.2 从 WSL 内打开 VSCode

进入你的项目目录，执行：

bash

```bash
mkdir -p ~/projects/test-project
cd ~/projects/test-project
code .
```

VSCode 会自动在 Windows 侧打开，左下角显示 `WSL: Ubuntu` 绿色标志，说明已连接到 WSL 环境。

#### 4.3 在 VSCode 集成终端里使用 Claude Code

VSCode 打开后，用快捷键 **Ctrl+shift+\`** 打开集成终端，此终端就是 WSL 的 bash，直接运行：

bash

```bash
claude
```

#### 4.4 配置 VSCode 默认终端为 WSL bash（可选）

按 `Ctrl+Shift+P`，搜索 `Terminal: Select Default Profile`，选择 `WSL Bash`。

---

### 阶段五：创建项目 CLAUDE.md（AI Infra 学习用）

进入任意项目目录后：

bash

```bash
cd ~/projects/你的项目
nano CLAUDE.md
```

参考模板：

markdown

```markdown
# 项目说明

## 技术栈
- C++ / CUDA 12.x
- Python 3.11, PyTorch 2.x
- 构建：CMake + setuptools

## 常用命令
- `make build` — 编译 CUDA 扩展
- `pytest tests/` — 单元测试
- `python train.py` — 启动训练

## 编码规范
- CUDA kernel 必须注释 warp/block 布局
- Python 全部使用 type hints
- 提交前运行 black 和 clang-format

## 注意
- 不要修改 CMakeLists.txt 除非明确说明
- 大型模型训练移至远程服务器
```

---

### 验证清单

全部完成后，在 VSCode 集成终端里逐一确认：

bash

```bash
node --version        # v22.x.x
claude --version      # @anthropic-ai/claude-code x.x.x
curl -I https://api.anthropic.com  # HTTP/2 404 + cloudflare
git --version         # git version 2.x.x
python3 --version     # Python 3.x.x
```

都通过之后，在项目目录运行 `claude`，环境就完全就绪了。