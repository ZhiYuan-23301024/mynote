---
title: termux安装代码辅助CLI（goose）
categories:
  - 杂项
---
```bash
# 拉取镜像
udocker pull ubuntu:22.04
# 创建容器并挂载到工作目录
udocker run -i -t --name=aider -v /data/data/com.termux/files/home/Downloads/MyProjects:/mnt python:3.11
# 退出容器
exit
# 重新进入容器
udocker run -v /data/data/com.termux/files/home/Downloads/MyProjects:/mnt -i -t aider /bin/bash
# 下载curl
apt update && apt install -y curl
# 安装 goose
curl -fsSL https://github.com/block/goose/releases/download/stable/download_cli.sh | bash
# 验证goose版本
goose --version
# 启动可视化配置向导
goose configure
# 启动goose
goose start
```

在 Termux 里直接跟各种 Python 包甩锅的底层依赖作斗争，往往是一场极其折磨的“依赖地狱”之旅。因为你不仅要面对手机芯片（ARM64）特有的兼容性问题，还要应对 Termux 非标准文件系统结构带来的各种奇葩报错。

为了让你少走弯路，我为你量身定制了两个维度的破局方案。**强烈推荐你直接看方案一**，一劳永逸。

### 🎯 方案一：降维打击 —— 退守 udocker，用正统 Linux 环境（强烈推荐）

既然你手头有 `udocker`，这就是你最好的逃生舱。不要在 Termux 裸机上死磕了，我们直接在 `udocker` 里拉取一个正统的、支持 ARM64 的 Python 镜像，把 Aider 装进一个纯净的 Linux 沙盒里！

这样不仅能完美避开 Termux 所有的底层路径报错，还能利用容器自带的编译环境顺利装上 `uv`。

**1. 拉取支持 ARM64 的官方 Python 镜像**

打开 Termux，执行：

```bash
udocker pull python:3.11
```

**2. 创建并运行容器（附带终极排雷参数）**

结合你上次遇到的 `resolv.conf` 路径报错，这次我们直接加上 `--novol` 阻止它乱挂载，并把你的项目目录映射进去：

```bash
# 创建容器实例
udocker create --name aider-box python:3.11

# 启动容器，挂载你的代码目录
udocker run --novol -v /data/data/com.termux/files/home/Downloads/MyProjects:/mnt -i -t aider-box /bin/bash
```

**3. 在容器内部丝滑安装 Aider**

现在你已经进入了模拟的 Linux 环境，直接用它自带的 Python 和 pip 安装，绝对顺畅：

```bash
pip install aider-chat
```

_(注：如果这里下载慢，可以先执行 `pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple` 换国内源)_

**4. 运行 Aider**

```bash
cd /mnt
export OPENAI_API_KEY="sk-your-key"
aider
```

🎉 **大功告成！** 这种方式既干净又稳定，卸磨杀驴（删除容器）也毫无心理负担。

---

### 🥷 方案二：硬核破解 —— 在 Termux 裸机强行绕过 uv

如果你就是不想用容器，非要在 Termux 本地跑，那我们只能用“瞒天过海”的战术，强行把 `uv` 这个依赖剔除掉。

Aider 的核心功能其实并不完全依赖 `uv`，只是它的打包配置强制要求了。我们可以通过以下步骤强行通关：

**1. 强行跳过依赖检查安装 Aider**

```bash
pip install aider-chat --no-deps --break-system-packages
```

**2. 手动补齐 Aider 真正需要的运行依赖**

刚才的命令跳过了所有依赖，我们现在把除了 `uv` 之外的核心库补上：

```bash
pip install aiohttp anthropic gitpython tiktoken watchfiles yarl --break-system-packages
```

**3. 制作一个“假”的 uv 命令（Shim）**

Aider 启动时会去系统路径里找 `uv`，既然我们装不上，就给它造一个假装存在的。

```bash
# 创建一个假的 uv 脚本
echo '#!/bin/sh' > /data/data/com.termux/files/usr/bin/uv
echo 'echo "uv is disabled in termux"' >> /data/data/com.termux/files/usr/bin/uv
chmod +x /data/data/com.termux/files/usr/bin/uv
```

**4. 修改 Aider 源码中的 uv 调用（进阶）**

这一步有点硬核。你需要用文本编辑器（如 `vim` 或 `nano`）打开 Aider 的安装目录（通常在 `~/.../site-packages/aider/` 下），搜索所有调用 `uv` 的 Python 文件（比如涉及环境检查或 MCP 工具调用的部分），将相关的 `subprocess.run(["uv", ...])` 逻辑注释掉或改为直接报错退出。

**坦白说，方案二就是在拆东墙补西墙**，后续大概率还会遇到其他底层 C++ 库（如 `faster-whisper` 或 `onnxruntime`）在 Termux 上编译失败的问题。

💡 **最终建议**：听我一句劝，老老实实选**方案一**。在手机上搞开发环境，容器化才是唯一的出路



没问题！Aider 其实是一个纯粹在命令行里运行的工具。只要掌握了“在哪开”、“怎么关”以及“怎么连”，配合几个防烧钱的小开关，你就能把它拿捏得死死的。

这里我继续以咱们刚才配置好的 **DeepSeek** 为例来教你。

### 1. 怎么挂载到工作区？（指定项目文件夹）

Aider 没有图形界面，它只会“接管”你当前所在的文件夹。

- **第一步**：打开你的终端（Windows 的 CMD/PowerShell，或 Mac 的 Terminal）。
    
- **第二步**：通过 `cd` 命令进入到你的代码项目文件夹。
    
    - _例如：你的项目放在桌面的 `my_code` 文件夹里，你就输入 `cd Desktop/my_code`_。
        
- **第三步**：在这个目录下启动 Aider，它就自动“挂载”好了。
    

### 2. 怎么进入并开启“省钱模式”？

不要直接敲 `aider` 就回车！加上这几个参数，能帮你挡掉 60% 以上的无效开销：

```bash
aider --model deepseek/deepseek-chat --no-auto-commits --no-gitignore
```

- 💰 **`--no-auto-commits`**（最核心的省钱开关）：默认情况下，Aider 每次跟你对完话都会自动把代码存进 Git 存档。这会耗费大量 Token 去读取变更记录。**关掉它，只在你自己满意的时候手动 `git commit`。**
    
- 🛡️ **`--no-gitignore`**：防止 Aider 去读取那些被你设置为“忽略”的庞大依赖包文件夹（比如 `node_modules`），避免一上来就吃光你的额度。
    

### 3. 怎么退出 Aider？

非常直观，在 Aider 的交互界面里直接输入：

```bash
/exit
```

或者直接用键盘快捷键 `Ctrl + C` 强制中断也可以。

---

### 💡 进阶：在 Aider 内部的“抠门”绝招

进去之后，你还可以用这几招把性价比拉满：

- **精准投喂，拒绝全家桶**：
    
    不要用 `/add .`（添加当前目录全部文件），而是用 `/add 具体的文件名.py`。Aider 每次对话都会把上下文发给 API，你喂得越精简易懂，花钱越少。
    
- **让 AI 只动口不动手（超级省钱）**：
    
    如果你只是想找个思路，不想让它真的去改代码，在提问前加上 `/thinking` 或者直接问：“**请只给出修改建议，不要直接写完整代码。**” 这样能大幅缩减它的输出长度（输出比输入贵 4 倍！）。
    
- **深夜冲浪**：
    
    还记得刚才说的吗？DeepSeek 在 **凌晨 00:30 到 早上 08:30** 有极大幅度优惠。如果你有那种需要一次性扔几十个文件让它帮忙重构的大工程，留到半夜跑，简直便宜到哭。