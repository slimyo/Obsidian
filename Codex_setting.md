## Codex CLI 常用设置指南（基于官方最新命令行参考）

> 已对照 OpenAI 官方文档（developers.openai.com/codex/cli/reference）核对。 注意：Codex 迭代很快，模型名与个别行为以你本机 `codex --help` 实际输出为准。

### 🔑 核心概念：审批策略 + 沙箱策略

- **审批策略**（`-a, --ask-for-approval`）：控制在执行模型生成的命令前，何时暂停等你批准。
- **沙箱策略**（`-s, --sandbox`）：限制命令能访问的文件系统范围（读 / 写）以及是否能联网。

二者**独立组合**使用。要正确理解“自动化程度”，必须同时看这两个轴 —— 只看审批策略是常见误区。

---

### ⚙️ 审批策略 `-a` 可选值

| 值                | 行为                                                       |
| ---------------- | -------------------------------------------------------- |
| `untrusted`      | 除一小部分受信任的只读命令外，**每条命令都请求批准**（最保守）                        |
| `on-request`     | 默认在沙箱内**自动**执行；当需要**越出沙箱边界**（如联网、写工作区外）时才请求批准。**交互式推荐值** |
| `never`          | **从不**请求批准（完全不打断），但仍**受沙箱限制**。适合非交互 / CI                 |
| ~~`on-failure`~~ | **已弃用**。交互式请改用 `on-request`，非交互改用 `never`                |

> ⚠️ 你原文档把 `on-failure` 列为常规选项是过时的；现在用它会打印弃用警告。

---

### 📁 沙箱策略 `-s` 可选值

|值|权限范围|
|---|---|
|`read-only`|只能读文件，禁止任何写入|
|`workspace-write`|可写入工作区（`-C` 指定目录及 `--add-dir` 额外目录）。**默认禁止联网**|
|`danger-full-access`|允许访问整个文件系统并联网（谨慎使用）|

> ⚠️ **关键易错点**：`workspace-write` 默认**网络是关闭的**。若命令需要联网（如装依赖、跑迁移），要显式开启： `-c sandbox_workspace_write.network_access=true`

---

### 🛡️ 日常推荐组合

| 场景                    | 命令                                                                                      | 说明                                      |
| --------------------- | --------------------------------------------------------------------------------------- | --------------------------------------- |
| **默认 / 交互式（Auto 预设）** | `codex -a on-request -s workspace-write`                                                | 沙箱内自行执行低风险操作，越界时询问你。直接 `codex` 即等价于该预设。 |
| **交互式 + 允许联网**        | `codex -a on-request -s workspace-write -c sandbox_workspace_write.network_access=true` | 在上面基础上放开网络访问。                           |
| **全自动 / CI**          | `codex -a never -s workspace-write`                                                     | 从不打断，写入限制在工作区（仍默认禁网，需要联网同样加上面的 `-c`）。   |
| **只读审查**              | `codex -a untrusted -s read-only`                                                       | 每条命令都需确认，且无法修改任何文件。                     |
| **极端危险（仅隔离环境）**       | `codex --dangerously-bypass-approvals-and-sandbox`（别名 `--yolo`）                         | 绕过**所有审批和沙箱**，完整系统权限。仅用于一次性容器 / 隔离虚拟机。  |

---

### 📋 常用指令速查表

|指令|说明|注意|
|---|---|---|
|`codex`|启动交互式 TUI，使用默认（Auto）预设|等价于 `-a on-request -s workspace-write`|
|`codex -a never -s workspace-write`|全自动执行，只写工作区|适合脚本；需确保命令不破坏工作区|
|`codex -a untrusted -s read-only`|只读 + 每步确认|最安全，但每条命令都要人工同意|
|`codex --dangerously-bypass-approvals-and-sandbox`|无限制权限（**极度危险**）|**切勿在本地或敏感系统使用**；仅在容器 / 虚拟机内|
|`codex -m gpt-5.4`|指定模型|模型名取决于账号 / 服务方，以 `codex --help` 或 `codex debug models` 为准（`gpt-4o` 已过时）|
|`codex --search`|启用**实时**网络搜索|把 `web_search` 由默认 `cached` 切到 `live`|
|`codex -C /path/to/project`|设置工作目录（沙箱写范围）|与 `-s workspace-write` 配合限制写入路径|
|`codex --add-dir /extra/path`|额外授予某目录写权限|比直接放开 `danger-full-access` 更安全|
|`codex resume --last`|恢复上一次会话|默认限当前目录，加 `--all` 跨目录|
|`codex fork --last`|从上次会话**分叉**出新线程|保留原始记录|
|`codex exec "任务"`|非交互式单次执行（别名 `codex e`）|适合脚本 / CI；可配 `--json`、`-o file` 捕获结果|

> 💡 会话中途想换权限模式，用 **`/permissions`** 斜杠命令即可，无需重启 codex。 另外 `--full-auto` 已**弃用**（会打印警告），官方建议直接用 `-s workspace-write` 代替。

---

### 💡 一句话总结

- **日常 / 交互式**：`codex`（即默认 `-a on-request -s workspace-write`）即可；需要联网时加 `-c sandbox_workspace_write.network_access=true`。
- **全自动任务**：`codex -a never -s workspace-write`。
- **不要碰** `--dangerously-bypass-approvals-and-sandbox`，除非你在隔离容器 / 虚拟机里并清楚后果。