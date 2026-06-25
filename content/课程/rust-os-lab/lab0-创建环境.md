---
title: lab0-创建环境
categories:
  - 课程
---
本次实验内容基于udocker配置实验环境

```ad-note
title:udocker简介

termux官方仓库没有针对docker进行重编译的版本，因此如果要在android+termux环境上实现类似docker的功能，要使用udocker。
**udocker** 是一款轻量级容器工具，**无需 root 权限**。
它让普通用户在无法获取 sudo 权限的服务器或超算集群上，也能像使用普通软件一样，自由拉取和运行 Docker 镜像。
```
### 基于udocker拉取实验镜像

```bash
# 下载udocker
pkg install udocker
# 拉取实验镜像
docker pull openeuler/openeuler
# 检查是否拉取成功
udocker images
```

观察到终端输出如下，成功拉取

![[Pasted image 20260509150826.png]]

### 创建本地开发目录并挂载

#### 创建工作目录
本地机器指的是启动镜像时所在的终端环境
在本实例中指的是termux的bash终端环境，如果使用win系统，win的文件管理环境就是本地环境，Linux同理。如果使用虚拟机，则虚拟机环境里的文件管理环境就是本地环境

```bash
# 进入存放工作目录的文件夹
cd ～/Downloads/MyProjects
# 创建工作文件夹
mkdir MyOS
```

#### 创建自己的docker镜像

在自己创建的实验目录下执行如下命令：
注意：执行如下命令需要在自己创建的实验目录下
```bash
# 创建容器，并将容器的mnt目录挂载在本地实验目录下
udocker run -i -t --name=OS -v /data/data/com.termux/files/home/Downloads/MyProjects/MyOS:/mnt openeuler/openeuler
# 退出容器
exit
# 重新进入容器
udocker run --nosysdirs -v /data/data/com.termux/files/home/Downloads/MyProjects/MyOS:/mnt -i -t OS bash
```

成功进入容器后可看到终端输出如下

![[Pasted image 20260509153440.png]]

在容器环境中安装一些必要的工具：

```bash
# 下载
dnf install curl vim gcc

# 检查是否安装成功

# 检查 curl
curl --version
# 检查 vim
vim --version
# 检查 gcc
gcc --version
```

若安装成功控制台应输出对应软件的版本号，以gcc为例

![[Pasted image 20260509154351.png]]

### 配置Rust开发环境

#### 在环境变量中配置rust下载源

```bash
# 进入用户目录
cd ~
# 编辑环境配置文件
vim .bashrc
```

在vim中输入一下内容后保存

```vim
export RUSTUP_DIST_SERVER=https://mirrors.ustc.edu.cn/rust-static
export RUSTUP_UPDATE_ROOT=https://mirrors.ustc.edu.cn/rust-static/rustup
```

![[Pasted image 20260509155442.png]]

#### 安装rust

```bash
# 安装rust语言工具链
curl https://sh.rustup.rs -sSf | sh
# 安装环境启用。
source "$HOME/.cargo/env"
# 安装nightly版本
rustup install nightly
# 将rust默认设置为nightly
rustup default nightly
# 切换cargo软件包镜像为tuna
vim ~/.cargo/config
# 添加如下内容
[source.crates-io]
replace-with = 'ustc'

[source.ustc]
registry = "git://mirrors.ustc.edu.cn/crates.io-index"

# 安装一些Rust相关的软件包
rustup target add riscv64gc-unknown-none-elf
cargo install cargo-binutils
rustup component add llvm-tools-preview
rustup component add rust-src
```

**限制rust的版本**
在工作目录下创建一个名为 rust-toolchain 的文件，以限制rust的版本，文件内容如下：
```
nightly-2022-10-19
```

**检查是否安装成功**
```bash
# 检查软件是否安装成功
rustc --version
cargo --version
rustup --version
# 检查是否为nightly工具链
rustup show
# 检查RISC-V目标是否安装成功
rustup target list --installed
# 检查cargo-binutils是否可用
rustup run nightly cargo install --list | grep binutils
# 检查LLVM工具 & rust-src
rustup component list --installed
```

![[Pasted image 20260509163054.png]]

![[Pasted image 20260509163126.png]]
### 安装qemu

```bash
dnf groupinstall "Development Tools"
dnf install autoconf automake gcc gcc-c++ kernel-devel curl libmpc-devel mpfr-devel gmp-devel \
              glib2 glib2-devel make cmake gawk bison flex texinfo gperf libtool patchutils bc \
              python3 ninja-build wget xz

# 通过源码安装qemu 5.2
wget https://download.qemu.org/qemu-5.2.0.tar.xz
# 解压源码包
tar xvJf qemu-5.2.0.tar.xz
cd qemu-5.2.0
./configure --target-list=riscv64-softmmu,riscv64-linux-user
# 编译源码包
make -j$(nproc) install
```

安装完成后可以通过如下命令验证qemu是否安装成功。

```bash
qemu-system-riscv64 --version
qemu-riscv64 --version
```

![[Pasted image 20260509171717.png]]

### git 提交截图

![[Pasted image 20260509174532.png]]
![[Pasted image 20260509174556.png]]