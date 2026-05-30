### 🪜 配置优先级：命令行 > 项目级 > 全局

Codex CLI 的配置遵循一个清晰的优先级规则：**命令行参数**优先级最高，会覆盖**项目级配置**（ `./.codex/` ），而项目级配置会覆盖**全局配置**（ `~/.codex/` ）。简单理解，越靠近你当前执行目录的配置，权力越大。

### 📁 配置文件解析

Codex CLI 的核心是 `~/.codex/` 目录下的两个文件，建议用 `mkdir -p ~/.codex` 创建好。

*   **`config.toml`** (主配置文件): 用标准 **TOML** 格式，定义核心行为、模型、提供商和安全策略等。
*   **`auth.json`** (认证文件): 存API Key等敏感凭证，**切记不要提交到任何版本控制系统 (Git) 中**。

另外，你也可以把以上两种配置统一放在项目根目录的 `./.codex/` 文件夹下（项目级配置），也可以使用 `config.json` 或 `config.yaml` 格式。

#### **`config.toml` 核心配置详解**

你可以把它当成Codex CLI的控制中心，以下是常用配置项：

| 配置类别 | 配置项 (TBD) | 说明与示例 |
| :--- | :--- | :--- |
| **🔑 API基础设置** | `openai_base_url` | **国内用户必改**：设置兼容OpenAI协议的国内API中转地址。<br>`openai_base_url = "https://your-api-proxy.com/v1"` |
| | `model` | 指定使用的模型。示例：`model = "gpt-4o"` 或 `model = "o3-mini"`。 |
| | `provider` | 指定大模型提供商，默认为"openai"。示例：`provider = "azure"`。 |
| **⚙️ 运行模式与沙盒** | `approval_mode` | 设置审批模式。<br>`approval_mode = "auto-edit"` (自动批准文件修改) |
| | `sandbox_mode` | 设置沙盒策略，限制操作范围。<br>`sandbox_mode = "workspace-write"` (允许写工作区) |
| **🕰️ 历史记录** | `history.maxSize` | 会话历史保存的最大条目数。示例：`maxSize = 1000`。 |
| | `history.saveHistory` | 是否保存历史记录。示例：`saveHistory = true`。 |
| **🧪 实验性功能** | `web_search` | 启用或禁用网络搜索能力。<br>`web_search = "enabled"` |

#### **`config.toml` 高级配置: 自定义Provider**

如果你想用非OpenAI官方服务（如Azure, Ollama等），可以在 `config.toml` 中添加 `[model_providers]` 部分。

*   **接入Ollama**: 连接你本地运行的Ollama服务。
    ```toml
    [model_providers.ollama]
    name = "Ollama"
    base_url = "http://localhost:11434/v1"
    ```
*   **接入Azure OpenAI**: 通过Azure的API访问OpenAI模型。
    ```toml
    [model_providers.azure]
    name = "Azure"
    base_url = "https://YOUR_PROJECT_NAME.openai.azure.com/openai"
    env_key = "AZURE_OPENAI_API_KEY"
    ```

### 🌍 环境变量与环境文件

环境变量是快速设置API密钥等敏感信息的方式，可通过 `export` 命令或 `.env` 文件管理。同时，你可以在 `~/.codex/` 目录下放以下文件：
*   **`.env`**: 项目级环境变量，优先级高于全局配置。
*   **`~/.codex.env`**: 用户级环境变量。

Codex支持以下常见环境变量:
*   **`OPENAI_API_KEY`**: 通用API密钥。
*   **`DEBUG`**: 设为 `true` 可开启DEBUG模式，便于排查网络或请求问题。
*   **`<provider>_API_KEY`**: 其他提供商API密钥，如 `ANTHROPIC_API_KEY`。
*   **`<provider>_BASE_URL`**: 其他提供商Base URL，如 `DEEPSEEK_BASE_URL`。

### 🚀 命令行参数速查

启动Codex CLI时，可以通过添加参数来覆盖配置文件和环境变量的设置，获得最高优先级。常用参数如下:

| 参数                | 简写   | 说明与示例                                                          |
| :---------------- | :--- | :------------------------------------------------------------- |
| `--help`          | `-h` | 显示帮助信息。                                                        |
| `--version`       |      | 显示版本信息。                                                        |
| `--model`         | `-m` | 指定模型，如 `--model o4-mini`。                                      |
| `--provider`      | `-p` | 指定提供商，如 `--provider openrouter`。                               |
## 三个推荐的安全自动化级别

根据你的风险承受能力，可以使用不同的组合：

| 使用场景               | 推荐命令                                                                                 | 说明                                                                |
| ------------------ | ------------------------------------------------------------------------------------ | ----------------------------------------------------------------- |
| **日常开发（较安全）**      | `codex -a on-request -s workspace-write`                                             | 模型可自动执行常见命令，但遇到高风险操作（如删除文件、修改系统配置）时会询问你。**网络默认关闭**，需要网络可手动加 `-c`。 |
| **完全自动（CI/CD/容器）** | `codex -a never -s workspace-write -c 'sandbox_workspace_write.network_access=true'` | 从不询问，但仍在沙箱内运行。适合你完全信任的任务，如测试、构建、代码生成。                             |
| **极度危险（仅隔离环境）**    | `codex --dangerously-bypass-approvals-and-sandbox`                                   | **绕过所有审批和沙箱**，相当于无限制的 root 权限。仅用于一次性 Docker 容器或完全离线的虚拟机。          |

> 注意：`-a on-request` 是让模型自己决定是否需要请求批准（例如它觉得不确定时会问你），这是默认行为。如果你想**完全手动**（每个命令都问），可以用 `-a untrusted` 或不加 `-a` 参数。

### 💡 补充设置（斜杠命令、会话管理）

除了配置文件和启动参数，Codex CLI在交互会话中也提供了一些命令和快捷方式，方便你动态调整设置：

*   **会话内 `/` 命令**: 在会话中输入 `/` 可访问内置命令，例如 `/model` 可动态切换模型， `/permissions` 可调整权限模式， `/login` 可重新进行身份验证。
*   **会话管理**: 使用 `codex resume --last` 可快速恢复上一次会话。
*   **项目文档 (`AGENTS.md`)**: 在项目根目录创建 `AGENTS.md` 或 `codex.md`，Codex会在每次会话开始时读取它，让你可以用自然语言定义项目规范。