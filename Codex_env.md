- 使用deepseek接入codex
[awesome-deepseek-agent/docs/codex.zh-CN.md at main · deepseek-ai/awesome-deepseek-agent · GitHub](https://github.com/deepseek-ai/awesome-deepseek-agent/blob/main/docs/codex.zh-CN.md)

---
## 1：Node.js

安装 Node.js 最推荐的方法是利用其官方版本管理工具 `nvm`，它能让你轻松地安装、切换和管理多个 Node.js 版本，非常灵活。以下是针对不同操作系统，基于官方 `nvm` 方法的安装教程。

---
### 📥 第一步：安装 nvm (Node Version Manager)

`nvm` 是一个跨平台的工具，可以帮你管理多个 Node.js 版本。

*   **Windows 用户**：
    1.  访问 `nvm-windows` 的官方 GitHub 仓库：[https://github.com/coreybutler/nvm-windows/releases](https://github.com/coreybutler/nvm-windows/releases)
    2.  在 "Assets" 区域，下载最新版本的 `nvm-setup.exe` 文件。
    3.  运行下载好的 `.exe` 文件，按照安装向导的提示完成安装。
    4.  安装完成后，执行以下命令使配置生效，或者关闭并重新打开终端：
        ```bash
        source ~/.bashrc
        # 如果你使用的是 zsh，则执行：
        # source ~/.zshrc
        ```

- **macOS / Linux 用户**：
    1. 打开你的终端 (Terminal)。
    2. 执行以下命令下载并运行安装脚本[](https://nodejs.org/en/download/)：
        ```bash
        curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh 
        ```
    3. 安装完成后，执行以下命令使配置生效，或者关闭并重新打开终端：   
```   bash
        source ~/.bashrc
        # 如果你使用的是 zsh，则执行：
        # source ~/.zshrc
```

---

### 💻 第二步：通过 nvm 安装 Node.js

安装好 `nvm` 后，你就可以轻松安装 Node.js 了。以下所有操作都在**终端**（Windows 上为命令提示符或 PowerShell）中进行。

1.  **查看可用的 Node.js 版本**：
    ```bash
    nvm list availabler
    ```
    或者查看所有远程可安装的版本：
    ```bash
    nvm list remote
    ```

2.  **安装最新的长期支持版（LTS - Long Term Support）** ：
    这是推荐给大多数用户的稳定版本。
    ```bash
    nvm install --lts
    # 或安装指定版本，例如 Node.js 24，这也是一个 LTS 版本
    # nvm install 24
    ```

3.  **切换并使用已安装的 Node.js 版本**：
    ```bash
    nvm use 24.16.0
    ```

---

### ✅ 第三步：验证安装

无论你使用哪种操作系统，安装和切换完成后，都可以通过以下命令验证 Node.js 和它自带的包管理器 `npm` 是否安装成功。

```bash
# 查看 Node.js 版本
node -v

# 查看 npm 版本
npm -v
```
如果这两个命令都正确打印出了版本号（例如 `v24.16.0` 和 `11.13.0`），则说明安装成功。

---

### 🛠️ 第四步：后续配置（可选但推荐）

为了在后续开发中能更快地安装依赖，可以考虑配置 npm 镜像。如果安装时速度很慢，可以通过 nvm 设置 Node.js 和 npm 的国内镜像源来加速：

*   **设置 nvm 的 Node.js 镜像**：
    ```bash
    nvm node_mirror https://npmmirror.com/mirrors/node/
    ```
*   **设置 npm 的全局镜像为淘宝镜像**：。
    ```bash
    npm config set registry https://registry.npmmirror.com
    ```
    配置完成后，可以运行 `npm config get registry` 来验证是否设置成功。

---

### 📌 补充说明：为什么不推荐直接用官网安装包？

*   **版本管理困难**：使用官网 `.pkg` (macOS) 或 `.msi` (Windows) 安装包安装的是系统级的 Node.js。这会导致在同一台机器上**难以在不同项目间切换 Node.js 版本**，未来升级版本也需要重新下载安装包。
*   **权限问题**：直接在 macOS 或 Linux 上使用安装包，有时会因为权限问题导致全局安装包时报错。
*   **相比之下**：`nvm` 解决了以上所有痛点，让你能轻松地在多个 Node.js 版本之间切换，完全不受项目环境限制。

---

这份教程将指导你通过 **Moon Bridge** 转发层，将 OpenAI 的 Codex 接入 DeepSeek 模型。

由于不同操作系统的具体命令略有差异，请根据你所用的系统，遵循以下分步指南。

---

## 2.  **安装 Go (版本 1.25+)**：

### windows:

- 前往 [Go 官网](https://go.dev/dl/) 下载并安装。安装完成后，powersehll运行 `go version` 确认版本号。
### linux

- 从官网下载并安装 (推荐)
1. **更新系统并安装工具包**
```bash
    sudo apt update && sudo apt upgrade -y
    sudo apt install wget build-essential
```
    
- `build-essential` 包含了编译和构建 Go 项目时可能会用到的`gcc`、`make`等工具
2. **下载并解压 Go**  
    将以下命令中的 `VERSION_NUMBER` 替换为你想安装的具体版本号（如 1.26.3），你可以在 [Go 官网下载页](https://go.dev/dl/) 查看最新的稳定版版本号：
```bash
    wget https://golang.org/dl/goVERSION_NUMBER.linux-amd64.tar.gz
    sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf goVERSION_NUMBER.linux-amd64.tar.gz
    rm goVERSION_NUMBER.linux-amd64.tar.gz
```
- `wget`：下载 Go 的 Linux 二进制包
- `sudo rm -rf /usr/local/go`：确保之前没有旧版本的 Go，以避免冲突[](https://datasea.cn/go0130445640.html)。
- `sudo tar -C /usr/local -xzf`：将下载的压缩包解压到 `/usr/local` 目录[](https://datasea.cn/go1109264465.html#lwptoc12)。

3. **配置环境变量**  
    将 Go 的安装路径添加到系统的 PATH 中，使其在终端中能全局调用：

```bash
    echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
    source ~/.bashrc
```
    
- `echo ... >> ~/.bashrc`：将配置命令写入 `~/.bashrc` 文件，这样每次打开终端都会自动加载
- `source ~/.bashrc`：立即生效配置，无需重启终端[](https://blog.csdn.net/b1ue_2/article/details/154843829)[](https://tripletech.hashnode.dev/how-to-install-go-on-wsl-ubuntu-a-step-by-step-guide?source=more_articles_bottom_blogs)。
        
4. **验证安装**
```bash
    go version
```
    如果一切顺利，你将看到类似 `go version go1.22.5 linux/amd64` 的输出，表明安装成功

---
## 3.  **安装 Codex CLI**：

- Command-line interface
- 在终端或 PowerShell 中运行以下命令，安装 OpenAI 官方的 Codex 命令行工具：

    ```bash
    npm install -g @openai/codex
    ```
---
## 4.  **获取 DeepSeek API Key**：

- 访问 [DeepSeek 开放平台](https://platform.deepseek.com/api_keys)，登录后创建并复制你的 API Key。请妥善保管，它只显示一次。

---
##  5.核心操作：配置与启动

本方案通过 **Moon Bridge** 作为桥梁，让 Codex 的请求能正确发送给 DeepSeek 模型。请按以下顺序，在**两个独立的终端**中分别操作。

#### 终端一：配置并启动 Moon Bridge

这部分操作将克隆 Moon Bridge 项目，并配置它以将请求转发给 DeepSeek。

1.  **克隆 Moon Bridge 项目**：
    ```bash
    git clone https://github.com/ZhiYi-R/moon-bridge.git
    cd moon-bridge
    ```

2.  **创建配置文件**：
    在 `moon-bridge` 目录下，创建一个名为 `config.yml` 的文件，并将下面的配置内容完整复制进去。**请务必将 `sk-your-deepseek-api-key` 替换为你自己的 DeepSeek API Key**。

    ```yaml
    mode: "Transform"
    server:
      addr: "127.0.0.1:38440"
    models:
      deepseek-v4-pro:
        context_window: 1000000
        max_output_tokens: 384000
        default_reasoning_level: "high"
        supported_reasoning_levels:
          - effort: "high"
            description: "High reasoning effort"
          - effort: "xhigh"
            description: "Extra high reasoning effort"
        supports_reasoning_summaries: true
        default_reasoning_summary: "auto"
        extensions:
          deepseek_v4:
            enabled: true
      deepseek-v4-flash:
        context_window: 1000000
        max_output_tokens: 384000
        default_reasoning_level: "high"
        supported_reasoning_levels:
          - effort: "high"
            description: "High reasoning effort"
          - effort: "xhigh"
            description: "Extra high reasoning effort"
        supports_reasoning_summaries: true
        default_reasoning_summary: "auto"
        extensions:
          deepseek_v4:
            enabled: true
    providers:
      deepseek:
        base_url: "https://api.deepseek.com/anthropic"
        api_key: "sk-your-deepseek-api-key"
        offers:
          - model: deepseek-v4-pro
          - model: deepseek-v4-flash
    routes:
      moonbridge:
        model: deepseek-v4-pro
        provider: deepseek
    defaults:
      model: moonbridge
      max_tokens: 65536
    ```
-1
- key在配置文件中

3.  **启动 Moon Bridge 服务**：
    在 `moon-bridge` 目录下，运行以下命令启动服务。保持这个终端窗口一直运行。
    ```bash
    go run ./cmd/moonbridge --config config.yml
    
    # 报错可设置代理
    # 1. 设置国内最快的七牛云代理
	go env -w GOPROXY=https://goproxy.cn,direct
	# 2. (强烈推荐) 同时设置校验数据库代理，避免校验失败
	go env -w GOSUMDB=sum.golang.google.cn
    ```
    看到类似 `Listening on 127.0.0.1:38440` 的提示，表示启动成功。


#### 终端二：生成 Codex 配置

1.  **进入项目目录并准备环境**：
    打开一个新的终端窗口（或 PowerShell），**首先进入你想要使用 Codex 进行工作的项目目录**。
    ```bash
    cd /path/to/your/project
    ```

2.  **生成 `config.toml`**：
    运行下方对应你操作系统的命令，它会自动为 Codex 生成连接 Moon Bridge 所需的配置文件。

- **macOS / Linux**:
    
    ``` bash
    # 回到 moon-bridge 目录生成配置
    cd /path/to/moon-bridge
    CODEX_HOME_DIR="${CODEX_HOME:-$HOME/.codex}"
    mkdir -p "$CODEX_HOME_DIR"
    cp "$CODEX_HOME_DIR/config.toml" "$CODEX_HOME_DIR/config.toml.bak" 2>/dev/null || true
    MODEL="$(go run ./cmd/moonbridge --config config.yml --print-codex-model)"
    go run ./cmd/moonbridge \
      --config config.yml \
      --print-codex-config "$MODEL" \
      --codex-base-url "http://127.0.0.1:38440/v1" \
      --codex-home "$CODEX_HOME_DIR" \
      > "$CODEX_HOME_DIR/config.toml"
    ```

- **Windows (PowerShell)**:
    *   **Windows (PowerShell)**:
        ```powershell
        # 回到 moon-bridge 目录生成配置
        cd C:\path\to\moon-bridge
        $CODEX_HOME_DIR = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { "$HOME\.codex" }
        New-Item -ItemType Directory -Force -Path $CODEX_HOME_DIR | Out-Null
        if (Test-Path "$CODEX_HOME_DIR\config.toml") { Copy-Item "$CODEX_HOME_DIR\config.toml" "$CODEX_HOME_DIR\config.toml.bak" -Force }
        $MODEL = go run ./cmd/moonbridge --config config.yml --print-codex-model
        go run ./cmd/moonbridge `
          --config config.yml `
          --print-codex-config "$MODEL" `
          --codex-base-url "http://127.0.0.1:38440/v1" `
          --codex-home "$CODEX_HOME_DIR" `
          | Set-Content -Path "$CODEX_HOME_DIR\config.toml"
        ```
    *   **验证配置是否成功**：检查 `~/.codex/config.toml` (Linux/macOS) 或 `$HOME\.codex\config.toml` (Windows) 文件是否已生成。

3.  **启动 Codex**：
    生成配置后，在同一个终端中，确保你仍然在项目目录下，直接运行 `codex` 命令即可启动。
    ```bash
    codex
    ```
    现在，你可以正常使用 Codex 进行编程，它的所有 API 请求都会通过 Moon Bridge 发送给 DeepSeek 模型。

---

### ✅ 验证与测试

为了确保整个链路配置成功，你可以在 Moon Bridge 运行的同时，在另一个终端中使用 `curl` 命令进行测试。

1.  **查看可用模型列表**：
    ```bash
    curl http://127.0.0.1:38440/v1/models
    ```
    应该会返回包含 `"id": "moonbridge"` 的模型列表。

2.  **发送测试请求**：
    ```bash
    curl http://127.0.0.1:38440/v1/responses \
      -H "Content-Type: application/json" \
      -d '{ "model": "moonbridge", "input": "请用一句话打个招呼。", "max_output_tokens": 1024 }'
    ```
    如果一切正常，你将看到来自 DeepSeek 模型的回复。

---

### ⚠️ 常见问题与注意事项

*   **版本兼容性**：新版 Codex（≥ 0.81.0）移除了对旧 `chat` 协议的支持。如果遇到兼容性问题，可以尝试安装 `0.80.0` 版本：`npm install -g @openai/codex@0.80.0`。
*   **配置文件备份**：在生成新的 `config.toml` 前，脚本会自动为旧配置文件创建 `.bak` 备份。如有问题，可轻松恢复。
*   **API Key 安全**：请勿将包含 API Key 的配置文件或代码上传至公开仓库。建议使用环境变量管理敏感信息。
*   **代理网络问题**：若遇网络超时或连接失败，请检查终端一是否仍在运行 Moon Bridge，并确保防火墙允许 `127.0.0.1:38440` 端口的通信。