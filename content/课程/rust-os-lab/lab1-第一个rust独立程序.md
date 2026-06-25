---
categories:
  - 课程
title: lab1-第一个rust程序
---
本实验的主要目的是构建一个独立的不依赖于rust标准库的可执行程序。

### 创建一个Rust项目

首先进入已配置的好环境的容器，并进入/mnt目录

```bash
# 创建一个名为os的新项目， bin后缀说明是可执行程序
cargo new os --bin
# 进入os目录
cd os
# 下载依赖、编译项目、构建项目
cargo build
# 运行项目
cargo run
```
```ad-note
title:关于cargo

现代rust项目的包管理器，相当于maven之于java
- 创建项目
- 管理依赖
- 编译测试发布

--bin后缀表明该项目是一个可执行程序
cargo把项目区分为可执行程序`bin`和库程序`lib`，区别在于是否需要被别人加载才能运行

```

查看运行结果如下

![[Pasted image 20260511110020.png|500]]

### 移除标准库依赖

#### 修改target为riscv64

```ad-note
title:什么是'修改target为riscv64'

将编译目标改为生成能在riscv64架构cpu上跑的机器指令代码
否则会生成本地cpu指令架构的代码

```

在os目录下创建.cargo目录：

```bash
mkdir .cargo
```

然后，在os/.cargo目录下创建config文件，并增加如下内容：

```config
# os/.cargo/config
[build]
target = "riscv64gc-unknown-none-elf"
```
- **unknown**:不绑定厂商
- **none**：不指定操作系统，如果指定了操作系统，机器码将就不一样，一些敏感操作不会直接操作硬件，而是进行系统调用的请求
#### 修改main.rs文件

在 main.rs 的开头分别加入如下内容：

```rust
#![no_std]
#![no_main]
```

同时，因为标准库 std 中提供了 panic 的处理函数 #[panic_handler]，所以还需要实现panic handler。
具体增加如下内容：

```rust
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

**注意还需要删除main函数**

修改完后执行cargo build进行编译。如若出现编译错误，可以先执行cargo clean。其他错误也请尝试执行如下命令安装相关软件包。

```bash
rustup target add riscv64gc-unknown-none-elf
cargo install cargo-binutils
rustup component add llvm-tools-preview
rustup component add rust-src
```

![[Pasted image 20260511115457.png|400]]

#### 分析独立的可执行程序

执行如下命令分析移除标准库后的独立可执行程序

```bash
# 查看文件类型和基本信息
file target/riscv64gc-unknown-none-elf/debug/os
# 查看elf文件头
rust-readobj -h target/riscv64gc-unknown-none-elf/debug/os
# 反汇编和源码对照
rust-objdump -S target/riscv64gc-unknown-none-elf/debug/os
```

代码运行截图

**查看文件基本信息**
![[Pasted image 20260512103446.png]]
**查看elf文件头**
![[Pasted image 20260512103557.png]]
**反汇编和源码对照**
![[Pasted image 20260512103712.png]]

通过分析可以发现编译生成的二进制程序是一个空程序，这是因为编译器找不到入口函数，所以没有生成后续的代码
```ad-note
title:如何看出没有入口函数

entry:0x0,说明找不到入口函数，可能是_start符号没有被正确导出
反汇编中没有输出汇编，说明后续的代码都没有被编译
```
### 用户态可执行的环境

#### 增加入口函数
我们还需要增加入口函数，rust编译器要找的入口函数是 _start() 。
因此，我们可以在main.rs中增加如下内容：

```rust
#[no_mangle]
extern "C" fn _start() {
    loop{};
}
```

然后重新编译。

接着，通过如下命令：

```bash
qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/os
```

执行编译生成的程序，可以发现是在执行一个死循环，也即无任何输出，程序也不结束

![[Pasted image 20260512134348.png]]

如果把loop注释掉，然后重新编译执行的话，会发现出现了Segmentation fault。这是因为目前程序还缺少一个正确的退出机制

![[Pasted image 20260512134843.png]]

接着，我们实现程序的退出机制。

#### 实现退出机制

实现应用程序退出，在main.rs中增加如下代码：

```rust
use core::arch::asm;

const SYSCALL_EXIT: usize = 93;

fn syscall(id: usize, args: [usize; 3]) -> isize {
    let mut ret: isize;
    unsafe {
        asm!("ecall",
             in("x10") args[0],
             in("x11") args[1],
             in("x12") args[2],
             in("x17") id,
             lateout("x10") ret
        );
    }
    ret
}

pub fn sys_exit(xstate: i32) -> isize {
    syscall(SYSCALL_EXIT, [xstate as usize, 0, 0])
}

#[no_mangle]
extern "C" fn _start() {
    sys_exit(9);
}
```

修改完后，再重新编译和执行就可以发现程序能够正常退出了

![[Pasted image 20260512135117.png]]

#### 实现输出支持
首先，封装一下对SYSCALL_WRITE系统调用
这个是Linux操作系统内核提供的系统调用，其ID就是SYSCALL_WRITE

```rust
const SYSCALL_WRITE: usize = 64;

pub fn sys_write(fd: usize, buffer: &[u8]) -> isize {
  syscall(SYSCALL_WRITE, [fd, buffer.as_ptr() as usize, buffer.len()])
}
```

然后，实现基于 Write Trait 的数据结构，并完成 Write Trait 所需要的 write_str 函数，并用 print 
函数进行包装。

```rust
struct Stdout;

impl Write for Stdout {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        sys_write(1, s.as_bytes());
        Ok(())
    }
}

pub fn print(args: fmt::Arguments) {
    Stdout.write_fmt(args).unwrap();
}
```

最后，实现基于 print 函数，实现Rust语言 格式化宏 ( formatting macros )

```rust
use core::fmt::{self, Write};

#[macro_export]
macro_rules! print {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!($fmt $(, $($arg)+)?));
    }
}

#[macro_export]
macro_rules! println {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
    }
}
```

同时在入口函数_start增加println输出
```rust
println!("Hello, world!");
```

编译并通过如下命令执行，就可以看到独立的可执行程序已经支持输出显示了。

```bash
qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/os
```

![[Pasted image 20260512141650.png]]
### 思考并回答问题
#### 为什么称最后实现的程序为独立的可执行程序，它和标准的程序有什么区别？

标准程序调用了OS的依赖库，但是独立可执行程序自己实现依赖库
#### 实现和编译独立可执行程序的目的是什么？

学会如何脱离标准库编写程序，如何和运行环境交互

### git 提交截图

![[Pasted image 20260512143134.png|500]]
![[Pasted image 20260512143215.png|500]]