![测试物理帧分配](./assets/file-2026-06-15-19--06-83.png)

## 多级页表管理

实现页表数据结构

![](./assets/file-2026-06-15-20--06-15.png)

建立和拆除虚实地址之间的联系

![](./assets/file-2026-06-15-20--06-96.png)

实现手动查询地址映射

![](./assets/file-2026-06-15-20--06-27.png)

## 内核与应用的地址空间

### 实现地址空间抽象

首先，以逻辑段MapArea描述一段连续地址的虚拟内存。

![](./assets/file-2026-06-15-20--06-53.png)

接着，用MapType描述逻辑段内所有虚拟页号映射到物理页帧的方式

![](./assets/file-2026-06-15-20--06-77.png)

同时，利用MapPermisssion控制逻辑段的访问方式，其是页表项标志位PTEFlags的子集

![](./assets/file-2026-06-15-20--06-19.png)

然后，就可以实现地址空间了，也就是一系列有关联的逻辑段，用MemorySet来表示

![](./assets/file-2026-06-15-20--06-19%201.png)

同时，实现MemorySet的方法。

![](./assets/file-2026-06-15-20--06-91.png)

上述实现用到了MapArea提供的一些方法，具体实现如下：

![](./assets/file-2026-06-15-20--06-42.png)
### 实现内核地址空间

实现创建内核地址空间的方法new_kernel

![](./assets/file-2026-06-15-20--06-35.png)

### 实现应用地址空间

修改linker

![](./assets/file-2026-06-15-20--06-85.png)

精简loader

![](./assets/file-2026-06-15-20--06-38.png)

解析elf格式数据

![](./assets/file-2026-06-15-20--06-98.png)

在cargo中添加依赖

![](./assets/file-2026-06-15-20--06-20.png)

实现memory_set子模块所添加的代码

在main中添加各类use
![](./assets/file-2026-06-15-20--06-99.png)

在config中配置参数
![](./assets/file-2026-06-15-20--06-69.png)

## 基于地址空间的分时多任务

### 建立基于分页模式的虚拟地址空间

首先，创建内核地址空间

![](./assets/file-2026-06-15-21--06-90.png)

然后，在rust_main中进行内存管理子系统的初始化

![](./assets/file-2026-06-15-21--06-03.png)

### 实现跳板机制

首先，扩展Trap上下文

![](./assets/file-2026-06-15-21--06-89.png)

实现地址空间切换

![](./assets/file-2026-06-15-21--06-15.png)

将 trap.S 中的整段汇编代码放置在 .text.trampoline 段，并在调整内存布局的时候将它对齐到代码段的一个页面中

![](./assets/file-2026-06-15-21--06-04.png)

### 加载和执行应用程序

修改任务子模块，更新任务控制块管理

![](./assets/file-2026-06-15-21--06-73.png)

![](./assets/file-2026-06-15-21--06-83.png)

在内核初始化的时候，需要将所有的应用程序加载到全局应用管理器中，同时，也要修改TaskManager的实现

![](./assets/file-2026-06-15-21--06-16.png)

修改switch.S

![](./assets/file-2026-06-15-21--06-40.png)

修改switch.rs

![](./assets/file-2026-06-15-21--06-73%201.png)
### 改进trap的管理

首先修改init函数。然后再trap_handler的开头增加set_kernel_trap_entry
同时，在处理完trap后还要调用trap_return 返回用户态。

![](./assets/file-2026-06-15-21--06-58.png)

同时，在每一个应用程序第一次获得CPU权限时，内核栈顶放置在内核加载应用的时候构造的一个任务上下文

![](./assets/file-2026-06-15-21--06-36.png)

### 改进sys_write的实现

由于地址空间的隔离，sys_write无法直接方法应用空间的数据。为此，页表page_table提供一个将应用地址空间的缓冲区转化为内核地址空间可以直接访问的辅助函数

![](./assets/file-2026-06-15-22--06-53.png)

修改sys_write实现

![](./assets/file-2026-06-15-22--06-75.png)

### 修改应用程序

#### 删除user/src/lib.rs中的clear_bss()

除了删除clear_bss()的实现外，注意删除_start()中调用的clear_bss()

![](./assets/file-2026-06-15-22--06-55.png)

#### 删除build.py

因为应用程序的起始地址是相同的了，所以不在需要build.py，直接删除即可

![](./assets/file-2026-06-15-22--06-05.png)

同时，修改Makefile文件

![](./assets/file-2026-06-15-22--06-10.png)

### 修改main.rs

![](./assets/file-2026-06-15-22--06-95.png)

修改build.rs

![](./assets/file-2026-06-15-22--06-15.png)


## 10.思考并回答问题

### 分析虚拟地址和物理地址的设计与实现

本项目中虚拟地址与物理地址的设计基于 RISC-V Sv39 分页机制，核心代码位于 os/src/mm/address.rs 和 os/src/mm/page_table.rs。

**核心类型定义**
在 address.rs 中定义了四个基本类型：
- PhysAddr(usize)：物理地址，表示物理内存中的实际位置。
- VirtAddr(usize)：虚拟地址，表示 CPU 发出的内存访问地址。
- PhysPageNum(usize)：物理页号，物理地址除以页面大小（4KB）后的编号。
- VirtPageNum(usize)：虚拟页号，虚拟地址除以页面大小后的编号。

这些类型都实现了与 usize 的双向转换（From trait），使得地址计算和页号计算可以在类型安全的框架下进行。

**地址与页号的转换关系**
- 页面大小：PAGE_SIZE = 0x1000（4KB），PAGE_SIZE_BITS = 12。
- 虚拟地址 → 虚拟页号：VirtAddr.floor() 向下取整（addr / 4096），VirtAddr.ceil() 向上取整。
- 虚拟页号 → 虚拟地址：VirtPageNum → VirtAddr（vpn << 12）。
- 物理地址 → 物理页号：同理，phys / 4096。
- 页内偏移：addr & 0xFFF，即低 12 位。

**Sv39 虚拟地址结构**
在 Sv39 分页方案中，虚拟地址被拆分为：
  | VPN[2] (9bit) | VPN[1] (9bit) | VPN[0] (9bit) | offset (12bit) |
共 39 位有效虚拟地址。VirtPageNum::indexes() 方法将 VPN 分解为三级页表的索引：
  idx[2] = (vpn >> 18) & 0x1FF   // 一级页表索引
  idx[1] = (vpn >> 9) & 0x1FF    // 二级页表索引  
  idx[0] = vpn & 0x1FF           // 三级页表索引

**页表项（PTE）结构**
PageTableEntry 是一个 64 位结构（实际使用 54 位）：
  | PPN[43:10] (44bit) | flags (8bit) |
PTE::new(ppn, flags) 将物理页号左移 10 位，低 10 位存放标志位。
标志位定义（PTEFlags）：
  V(Valid)=bit0, R(Read)=bit1, W(Write)=bit2, X(Execute)=bit3,
  U(User)=bit4, G(Global)=bit5, A(Accessed)=bit6, D(Dirty)=bit7

虚拟地址到物理地址的转换流程：
1. satp 寄存器指向根页表物理页号（root_ppn）。
2. 以 VPN[2] 为索引查一级页表，得到二级页表的物理页号。
3. 以 VPN[1] 为索引查二级页表，得到三级页表的物理页号。
4. 以 VPN[0] 为索引查三级页表，得到最终物理页号。
5. 物理地址 = (PPN << 12) | page_offset。

地址空间的抽象
- VPNRange = SimpleRange<VirtPageNum>：表示一段连续的虚拟页号范围，实现 IntoIterator 可迭代 
- MapType::Identical：虚拟地址等于物理地址（用于内核映射）。
- MapType::Framed：动态分配物理帧（用于用户程序映射）。

**satp 寄存器格式**
PageTable::token() 返回：
  MODE(8) << 60 | root_ppn
其中 MODE=8 表示 Sv39。内核通过写 satp 寄存器和执行 sfence.vma 来切换地址空间。


### 分析物理帧是如何管理与分配的


物理帧管理器实现于 os/src/mm/frame_allocator.rs，核心设计如下：

**StackFrameAllocator 结构**
pub struct StackFrameAllocator {
    current: usize,    // 下一个可分配的空闲物理页号
    end: usize,        // 物理内存结束页号
    recycled: Vec<usize>,  // 回收的空闲页号栈
}

这是一个基于栈的物理帧分配器（Stack-based Allocator），其初始化方法为：
  init(l: PhysPageNum, r: PhysPageNum)
  设置 current = l，end = r。
  初始化时打印 "last N Physical Frames." 显示可用的物理帧数量。

**物理帧分配流程**
alloc() 方法按以下优先级分配：
1. 优先从 recycled 向量中弹出（pop）之前释放的帧——实现空闲帧复用。
2. 若 recycled 为空，则从 current 指针取帧，current += 1。
3. 若 current == end，说明物理内存已耗尽，返回 None。

这种策略优先复用释放的帧，减少内存碎片，类似"空闲链表"的栈实现。

**物理帧回收流程**
dealloc(ppn) 方法：
1. 合法性检查：ppn 必须 < current（已分配过），且不能在 recycled 中重复出现。
2. 通过检查后，将 ppn 推入 recycled 向量待复用。

**FrameTracker —— RAII 风格的帧管理**
pub struct FrameTracker { pub ppn: PhysPageNum }

FrameTracker::new(ppn) 在分配时自动清零整个 4KB 页面：
  for i in bytes_array { *i = 0; }

FrameTracker 实现了 Drop trait，在离开作用域时自动调用 frame_dealloc(ppn)，
将物理帧归还给分配器。这种 RAII 模式保证了即使在 panic 或提前返回等异常
情况下，物理帧也能被正确回收，避免了内存泄漏。

**全局单例访问**
通过 lazy_static! 创建全局静态变量：
  FRAME_ALLOCATOR: UPSafeCell<FrameAllocatorImpl>
UPSafeCell 是对 RefCell 的封装，提供 exclusive_access() 方法获取可变引用。
由于当前是单核环境，不需要真正的锁机制，使用 RefCell 即可。

**物理帧的初始化范围**
fn init_frame_allocator() 中：
- 起始地址：ekernel（内核结束地址）向上取整到页边界
- 结束地址：MEMORY_END（0x80800000，即 128MB 处）向下取整到页边界
即从内核镜像结束后到 128MB 物理内存上限之间的所有物理页均可分配。


### 分析内核的地址空间以及应用程序的地址空间是如何实现的

地址空间的实现位于 os/src/mm/memory_set.rs，核心结构是 MemorySet 和 MapArea。

**核心数据结构**
MemorySet {
    page_table: PageTable,  // 多级页表
    areas: Vec<MapArea>,    // 映射区域列表
}

MapArea {
    vpn_range: VPNRange,           // 虚拟页号范围
    data_frames: BTreeMap<VirtPageNum, FrameTracker>,  // framed 映射的帧
    map_type: MapType,             // Identical 或 Framed
    map_perm: MapPermission,       // 访问权限
}

**内核地址空间 —— MemorySet::new_kernel()**
内核地址空间使用 Identical 映射（虚拟地址 == 物理地址），确保内核可以
直接访问所有物理内存。构建步骤如下：

1. 创建空页表（new_bare），分配根页表帧。
2. 映射 Trampoline 页（TRAMPOLINE = 0xFFFF_FFFF_FFFF_F000，虚拟地址最高页）：
   - 使用 Identical 映射到 strampoline 的实际物理地址
   - 权限：R | X（可读可执行）
   - 用途：用户态与内核态切换的跳板代码

3. 依次映射内核各段（使用 Identical 映射）：
   - .text 段（stext~etext）：   权限 R|X（代码段，可读可执行）
   - .rodata 段（srodata~erodata）：权限 R（只读数据）
   - .data 段（sdata~edata）：   权限 R|W（可读写数据）
   - .bss 段（sbss~ebss）：      权限 R|W（未初始化数据）
   - 剩余物理内存（ekernel~MEMORY_END）：权限 R|W

4. 全局单例 KERNEL_SPACE 通过 lazy_static! 创建，类型为 Arc<UPSafeCell<MemorySet>>。
   Arc 允许在任务创建时共享内核页表引用。

内核地址空间的特点：
- 所有映射都是 Identical，VA == PA，内核可以直接访问物理地址。
- 没有 U（User）标志位，用户态不能访问内核空间。
- 内核可以访问全部物理内存（0x80000000 ~ 0x80800000）。

**应用程序地址空间 —— MemorySet::from_elf()**
应用程序地址空间使用 Framed 映射（动态分配物理帧），构建步骤如下：

1. 创建空页表。
2. 映射 Trampoline 页（与内核相同）。
3. 解析 ELF 文件：
   - 遍历所有 LOAD 类型的 program header
   - 为每个段创建 MapArea，使用 Framed 映射
   - 权限从 ELF ph_flags 读取，并附加 U（User）标志
   - 分配物理帧并复制 ELF 数据内容到帧中
   - 跟踪 max_end_vpn（程序最大虚拟页号）

4. 映射用户栈：
   - 起始位置 = max_end_vpn + 1 页（guard page）
   - 大小 = USER_STACK_SIZE（8KB）
   - 权限：R | W | U
   - 使用 Framed 映射

5. 映射 TrapContext 页：
   - 位置：TRAP_CONTEXT（= TRAMPOLINE - PAGE_SIZE）
   - 权限：R | W（仅内核可访问，无 U 标志）
   - 用于保存用户态进入内核态时的寄存器上下文

6. 返回值：(memory_set, user_stack_top, entry_point)

应用程序地址空间的特点：
- 使用 Framed 映射，物理帧动态分配，每个应用拥有独立的物理帧。
- 设置了 U 标志位，允许用户态 CPU 模式访问。
- 用户栈在程序段之后，有 guard page 保护。
- TrapContext 不设 U 标志，防止用户程序篡改。

**两种映射类型的对比**
  Identical：  VA == PA，页表直接映射，不分配新帧。
  Framed：     通过 frame_alloc() 分配新帧，VA 和 PA 不同。

**地址空间切换**
MemorySet::activate() 方法：
1. 获取页表 token（satp 值）
2. 写入 satp CSR 寄存器
3. 执行 sfence.vma 刷新 TLB

### 分析基于地址空间的分时多任务是如何实现的

分时多任务基于独立地址空间实现，核心代码位于 os/src/task/ 和 os/src/trap/。

**任务控制块 —— TaskControlBlock**
pub struct TaskControlBlock {
    task_status: TaskStatus,   // UnInit / Ready / Running / Exited
    task_cx: TaskContext,      // 内核态任务上下文（保存内核栈和返回地址）
    memory_set: MemorySet,     // 独立的地址空间（包含自己的页表）
    trap_cx_ppn: PhysPageNum,  // TrapContext 所在物理页号
    base_size: usize,          // 用户栈顶
}

每个任务拥有独立的 MemorySet，即独立的虚拟地址空间和页表。

任务创建 —— TaskControlBlock::new()
1. 调用 MemorySet::from_elf(elf_data) 解析 ELF，创建地址空间。
2. 获取 TrapContext 的物理页号（用于快速访问）。
3. 在内核地址空间中分配内核栈：调用 KERNEL_SPACE.insert_framed_area()
   为每个应用在 TRAMPOLINE 下方分配独立的 8KB 内核栈。
   kernel_stack_position(app_id):
     top = TRAMPOLINE - app_id * (KERNEL_STACK_SIZE + PAGE_SIZE)
     bottom = top - KERNEL_STACK_SIZE
4. 设置 TaskContext::goto_trap_return(kernel_stack_top)：
   ra = trap_return 地址（从内核第一次进入用户态的入口）
   sp = 内核栈顶
5. 初始化用户态 TrapContext：
   设置 sepc = entry_point（用户程序入口地址）
   设置 sp = user_stack_top（用户栈顶）
   设置 kernel_satp、kernel_sp、trap_handler 等内核信息

**任务管理器 —— TaskManager**
全局单例 TASK_MANAGER，初始化时：
1. 读取应用程序数量（_num_app 符号）。
2. 为每个应用创建 TaskControlBlock。
3. 所有任务初始状态为 Ready。

调度状态机：
  Ready → Running → Ready（时间片用尽，被挂起）
  Ready → Running → Exited（程序退出）

**时钟中断驱动的调度**
1. timer.rs 中 set_next_trigger() 每 10ms（CLOCK_FREQ / 100）触发一次时钟中断。
2. trap_handler 收到 SupervisorTimer 中断后：
   - 调用 set_next_trigger() 设置下一次中断
   - 调用 suspend_current_and_run_next() 挂起当前任务并切换

**任务切换流程 —— __switch**
汇编函数 __switch(current_task_cx_ptr, next_task_cx_ptr)：
1. 保存当前任务的 ra、sp、s0~s11 到 current_task_cx_ptr。
2. 从 next_task_cx_ptr 恢复 ra、sp、s0~s11。
3. ret 指令跳转到 ra（即下一个任务的执行点）。

**首次进入用户态**
1. run_first_task() → __switch(&unused, &task0.task_cx)
2. __switch 恢复 task0 的 ra（= trap_return）和 sp（= 内核栈顶）
3. ret 跳转到 trap_return
4. trap_return 中：
   - 设置 stvec 为 TRAMPOLINE（用户态中断入口）
   - 获取 TrapContext 地址和用户 satp
   - 跳转到 __restore（在 trampoline 中）
5. __restore 中：
   - 切换到用户 satp（地址空间切换）
   - 恢复所有通用寄存器
   - sret 跳转到 sepc（用户程序入口）

**任务切换完整流程**
  用户程序执行 → 时钟中断 → __alltraps（trampoline）
  → 保存上下文到 TrapContext → 切换到内核页表
  → trap_handler → suspend_current_and_run_next
  → __switch(当前, 下一个) → trap_return
  → __restore → 切换到用户页表 → sret → 新任务开始执行

**地址空间隔离的意义**
- 每个应用程序拥有独立的页表，虚拟地址 0x0 映射到不同的物理帧。
- 用户程序不能访问内核空间（没有 U 标志）。
- 用户程序不能访问其他程序的物理帧（不在其页表中）。
- 通过 satp 切换实现地址空间隔离，保证了多任务的安全性。

**轮转调度算法**
find_next_task() 从 current+1 开始循环查找状态为 Ready 的任务：
  (current + 1 .. current + num_app + 1).map(|id| id % num_app)
    .find(|id| tasks[id].task_status == TaskStatus::Ready)
这是典型的 Round-Robin 调度，每个任务轮流获得 CPU 时间片。


### 编写新的应用程序并测试验证结果

新增应用程序 04add.rs 以及 05fib.rs，验证虚拟内存和分时多任务功能。
具体实现见 user/src/bin/04add.rs 和 user/src/bin/05fib.rs

![](./assets/file-2026-06-15-23--06-60.png)

![](./assets/file-2026-06-15-23--06-54.png)

测试步骤：
1. 编译用户程序：cd user && make build
2. 编译内核：cd os && make build  
3. 运行 qemu 测试多任务是否正确切换和执行。

## git 截图

![](./assets/file-2026-06-15-23--06-26.png)