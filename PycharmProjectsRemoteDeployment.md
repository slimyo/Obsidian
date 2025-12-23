PyCharm 本地项目迁移到远程服务器操作指南

# **1️⃣ 本地生成 conda 环境 YAML 文件**

1. 打开本地终端或 Anaconda Prompt。  
2. 激活你的项目环境，例如：  
   ```
   conda activate TimeSeries_env  
   ```
3. 导出环境为 YAML 文件：

	- 在 Windows 上执行
	
	```
	conda activate your_env_name
	
	conda env export --no-builds --from-history > environment.yml
	```
	# 导出所有包（包含版本）
	
	```
	pip freeze > requirements.txt
	```
	
	# 或者只导出顶级包（推荐）
	
	```
	pip list --format=freeze > requirements.txt
	```
	
	- 创建对应的 environment.yml  
	   (conda env export > environment.yml会导出Windows版本，不兼容)  
4. 确认 environment.yml 文件已生成在本地项目根目录下。  
注：此文件包含当前环境所有依赖包信息，但不会包含本地路径或虚拟环境本身。

# **2️⃣ PyCharm 启动 SFTP 远程连接**

1. 打开 PyCharm。  
2. 进入：文件 → 设置 → 构建、执行、部署 → Deployment  
3. 点击“+” → 选择 SFTP。  
4. 配置远程服务器信息：  
   - 名称（Name）：随意，例如 RemoteServer  
   - SFTP 主机（SFTP host）：远程服务器 IP 或域名  
   - 用户名（User name）：远程服务器用户名  
   - 认证方式（Password / Key file）：SSH 密码或私钥  
   - 根路径（Root path）：远程项目存放路径，例如 `/home/username/TimeSeries  `
1. 点击 OK 保存配置。

# **3️⃣ 设置映射（Mapping）**

1. 在 Deployment 配置界面，点击映射（Mappings）标签。  
2. 设置映射关系：  
   - 本地路径（Local path）：本地项目根目录，例如 `C:\Users\zhouhuang\Projects\TimeSeries  `
   - 部署路径（Deployment path on server）：远程服务器对应目录，例如 `/home/username/TimeSeries  `
1. 保存设置。  
注：映射设置正确后，PyCharm 上传文件时会自动放到远程指定目录。

# **4️⃣ 上传配置文件或整个项目**

1. 上传单个文件：
	```
	在项目中右键 environment.yml → Deployment → Upload to RemoteServer  
	```
2. 上传整个项目：在项目根目录右键 → Deployment → Upload to RemoteServer  
3. 上传完成后，可通过远程 SSH 登录确认文件是否存在：  
```
   cd /home/username/TimeSeries  
   ls  
```
   应该能看到 environment.yml 文件已上传成功。

# **5️⃣ 在远程服务器创建 conda 环境**

1. SSH 登录远程服务器：  
   ```
   ssh username@remote_server  
   cd /home/username/TimeSeries  
   ```
2. 使用上传的 environment.yml 创建环境：  
  ```
   conda env create -f environment.yml
  ```
---
- 强烈建议：改用 mamba（成功率最高）

	如果你在服务器 / 校内网环境，这一步几乎是刚需。
	
	1️⃣ 安装 mamba
	
	```
	conda install -n base -c conda-forge mamba
	```
	
	2️⃣ 用 mamba 创建环境（替代 conda）
	
	```
	mamba env create -f environment.yml
	```
	
	�� 优点：自动重试下载并行解析依赖快几乎不会再出现 IncompleteRead

---
3. 激活远程环境：  
   conda activate TimeSeries_env  
4. 验证 PyTorch 是否可用 GPU：  
   ```
   python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"  
   ```
   输出示例：True NVIDIA GeForce RTX 3060

# **6️⃣ 绑定远程解释器并在 PyCharm 运行**

1. 打开：文件 → 设置 → 项目 → Python 解释器  
2. 点击右上角齿轮 → 添加（Add）  
3. 选择 SSH 解释器  
4. 填写远程服务器信息，并选择远程 conda 环境的 Python 路径：  
```
   /home/username/anaconda3/envs/TimeSeries_env/bin/python  
```
1. 点击 OK 保存。  
2. 创建 Run/Debug 配置：  
   - Script path：项目入口文件（如 myrun.py）  
   - Python interpreter：选择远程解释器  
   - Working directory：远程项目路径  
7. 点击 Run 或 Debug，代码会在远程服务器上运行，GPU 自动可用。

# **7️⃣ 注意事项**

- 环境同步：不要直接上传本地 conda 环境，必须在远程重建。  
- 文件上传：可以开启 Deployment → Options → “保存文件时自动上传”。  
- 显存/依赖：远程 GPU 显存和 CUDA 版本必须满足 PyTorch 要求。  
- 调试：绑定远程解释器后可直接在本地 PyCharm 调试远程代码。