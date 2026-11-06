# 📖 Linux 0.01 项目导读指南

> 本文件是 `Linux 0.01` 项目的中文导读，帮助你从零开始理解这个 **1991 年诞生的第一个 Linux 内核** 的架构、代码和运行方式。
> 全文约 10,000 行源码，是学习"一个操作系统如何从裸机长出来"的最好教材——麻雀虽小，五脏俱全。

---

## 目录

1. [这个项目是干什么的？](#1-这个项目是干什么的)
2. [核心概念速览](#2-核心概念速览)
3. [项目目录结构详解](#3-项目目录结构详解)
4. [运行流程全景图](#4-运行流程全景图)
5. [逐文件代码导读](#5-逐文件代码导读)
6. [关键设计模式解析](#6-关键设计模式解析)
7. [配置系统详解](#7-配置系统详解)
8. [如何运行和测试](#8-如何运行和测试)
9. [复刻建议与学习路线](#9-复刻建议与学习路线)
10. [常见问题](#10-常见问题)
- [附录：关键术语对照表](#附录关键术语对照表)

---

## 1. 这个项目是干什么的？

**一句话总结**：Linux 0.01 是 Linus Torvalds 在 1991 年发布的 Linux 内核第一个版本——一个能跑在 Intel 386 保护模式下、支持多任务、内存分页和 MINIX 文件系统的完整操作系统内核，总共只有约 1 万行代码。

它**不是一个**现代 Linux 发行版，而**是一个**"操作系统的全部骨架"：从按下电源到出现 shell 提示符的每一步，都能在这 1 万行里找到答案。现代 Linux 内核超过 3000 万行，而它的祖先只需要 1 万行就能开机、跑 shell、执行命令——因为骨架完整，血肉可以慢慢长。

**类比**：如果说现代 Linux 是一座超大城市，Linux 0.01 就是这座城市的**第一张施工图**：一条主路（启动流程）、一套水电管网（进程调度 + 内存分页）、一个邮局（文件系统）——小，但每根管道你都能摸得到。

```
按下电源 → BIOS → boot.s(引导扇区) → head.s(建页表/中断表) → main.c(内核初始化)
     → move_to_user_mode(进程0降权) → fork → init() → 挂载根文件系统 → 启动 /bin/sh
     → 用户在 shell 里敲命令 → write() 系统调用 → int 0x80 → 内核 TTY/控制台 → 屏幕出字
```

**技术上你必须先知道的几个硬约束**（读代码时处处会碰到）：

| 约束 | 数值 | 影响 |
|---|---|---|
| 目标 CPU | Intel 386 保护模式 | 代码里全是 GDT/LDT/TSS/页表这些 386 术语 |
| 内存模型 | 最多 8MB（`HIGH_MEMORY=0x800000`） | 页表只建了 2 页（pg0/pg1），够映射 8MB |
| 编译器 | gcc 1.40（1991 年版） | 只认 C89，**注释必须用 `/* */`**，没有 `//` |
| 文件系统 | MINIX FS（Linux 自己的 ext2 要 0.96 才有） | inode 32 字节、块 1KB、文件名最长 14 字符 |
| 任务数上限 | 64 个（`NR_TASKS`） | task[] 数组硬编码 64 项 |
| 系统调用 | 67 个（0~66） | sys_call_table 有 67 个表项，多数是空壳 |

**要点**：
- Linux 0.01 能完成：多进程（fork/exit/wait）、分页内存管理、COW 写时复制、MINIX 文件系统读写、控制台/键盘/串口/硬盘驱动、信号机制（雏形）。
- 它**没有**：网络、动态链接、虚拟终端切换、swap、管道虽有但相当简陋、现代符号链接。
- 代码里保留了 Linus 大量原版英文注释（包括著名的 "Linus fuck it up" 风格吐槽），这些注释本身也是历史文献，本项目的中文注释会完整保留它们并在其上叠加意图解释。

---

## 2. 核心概念速览

读 Linux 0.01 的代码前，有 7 个概念必须先建立。每个概念给出"是什么 + 把它理解为 + 在代码里长什么样"。

### 2.1 实模式与保护模式（Real Mode vs Protected Mode）

- **是什么**：x86 CPU 的两种运行状态。实模式（8086 时代遗产）只能寻址 1MB、没有内存保护；保护模式（386 起）支持 4GB 寻址、分段保护、分页。
- **把它理解为**：实模式是"自行车时代"（没有红绿灯），保护模式是"汽车时代"（有交规、有交警）。CPU 上电永远从自行车时代开始，操作系统要在头几百行代码里完成"换挡"。
- **在代码里**：`boot/boot.s` 末尾的 `mov ax,#0x0001` + `lmsw ax` 就是换挡动作；换挡前必须先备好 GDT（全局描述符表），否则一进保护模式就撞墙。

### 2.2 GDT / LDT / TSS（三张"户口本"）

- **是什么**：386 保护模式的三张核心表。GDT 是全局段描述符表（全系统共用）；LDT 是每个进程私有的段描述符表；TSS 是任务状态段（保存 CPU 寄存器全套现场，任务切换的硬件依据）。
- **把它们理解为**：GDT 是"全国户籍总册"，LDT 是每个进程的"个人户口本"（登记你的代码段、数据段住哪、有多大），TSS 是"任务交接登记表"（换班时把 CPU 里所有寄存器抄在上面）。
- **在代码里**：`head.s` 建 GDT 雏形；`include/linux/sched.h` 定义 `tss_struct`；`kernel/sched.c` 的 `sched_init()` 给每个任务槽位预登记 TSS/LDT 条目；任务切换靠 `switch_to` 宏里的一句 `ljmp`（长跳转到目标 TSS 段，CPU 硬件自动完成全部现场交换）。

### 2.3 分页与页表（Paging）

- **是什么**：把线性地址切成 4KB 一页，通过"页目录 → 页表 → 物理页"两级映射到物理内存。这是现代虚拟内存的地基。
- **把它理解为**：页目录是"省地图"（1024 项，每项指向一本县地图），页表是"县地图"（1024 项，每项指向一个 4KB 的街区）。你手里的地址是"逻辑门牌号"，CPU 查两级地图找到真实街区。
- **在代码里**：`mm/memory.c` 是全部实现。`head.s` 的 `setup_paging` 把最初 8MB 做"恒等映射"（虚拟地址=物理地址，内核专用）；`copy_page_tables()` 实现 fork 的页表复制；`do_wp_page()` 实现**写时复制（COW）**——父子进程先共享同一物理页并标记只读，谁写谁挨缺页中断，中断处理程序再复制新页。

### 2.4 进程与 task_struct（进程是"人"，task_struct 是"人事档案"）

- **是什么**：进程 = 运行中的程序。task_struct 是描述进程全部状态的结构体（寄存器现场、PID、打开的文件、信号等），Linux 0.01 里一个进程的 task_struct 恰好占一页（4KB），页的低端是结构体、高端是该进程的内核栈。
- **把它理解为**：人事档案 + 工位合租——档案在上半页，下半页是他干活时用的桌子（内核栈）。一个页既存"这个人是谁"又存"他此刻手头的东西"，切换任务时整页带走。
- **在代码里**：`include/linux/sched.h` 的 `struct task_struct` + `INIT_TASK` 宏；`kernel/sched.c` 的 `union task_union`（这就是"档案+桌子合一"的 union）；`task[64]` 数组是全系统的"员工名册"。

### 2.5 系统调用与 int 0x80（用户态敲内核大门的门铃）

- **是什么**：用户程序不能直接碰硬件，必须通过系统调用陷入内核。Linux 0.01 用 `int $0x80` 软中断：功能号放 eax，参数放 ebx/ecx/edx，返回值在 eax。
- **把它理解为**：银行柜台。用户态是柜台外的客户（权限低），内核态是柜员（权限高），int 0x80 是按铃，eax 里的号码牌决定办哪种业务。
- **在代码里**：`include/unistd.h` 的 `_syscall0/1/2/3` 宏（用 C 宏展开生成完整的系统调用桩函数）；`kernel/system_call.s` 的 `_system_call` 入口；`include/linux/sys.h` 的 `sys_call_table[67]` 函数指针表（号码牌 → 柜员窗口的映射表）。

### 2.6 缓冲区高速缓存 Buffer Cache（磁盘的"临时货架"）

- **是什么**：内核把磁盘数据按 1KB 块缓存到内存里，读写都先过缓存，避免反复磨盘。
- **把它理解为**：仓库门口的临时货架。要拿货先看货架上有没有（hash 表查找）；没有就叫叉车去仓库搬（ll_rw_block 发起硬盘中断 I/O）并放上货架；货架满了按 LRU 精神腾位置（free list 双向链表）。
- **在代码里**：`fs/buffer.c` 全部实现；`struct buffer_head`（`include/linux/fs.h`）是每个货架格子的登记卡（数据指针 + 设备号 + 块号 + 脏标记 + 引用计数）。

### 2.7 MINIX 文件系统（Linux 的第一位"房客"）

- **是什么**：Tanenbaum 教学用的文件系统，Linux 0.01 直接采用它（磁盘格式与 MINIX 完全兼容）。
- **把它理解为**：一栋楼的管理架构——1 层是"楼栋管理处的登记总册"（超级块，记整栋楼有多少房间、多少住户），2-3 层是"房间占用表"（imap 管住户登记号，zmap 管房间占用），之后是"住户档案柜"（inode 区，每个 inode 32 字节记一个文件的大小/权限/数据块位置），最后才是"真正放东西的房间"（数据区 zone）。
- **在代码里**：`fs/super.c`（读超级块、挂根）、`fs/inode.c`（inode 读写、iget/iput）、`fs/bitmap.c`（两个位图的分配回收）、`fs/namei.c`（按路径 "/usr/bin/sh" 一节节找 inode）。

**要点**：这 7 个概念在后面所有文件里反复出现。第一遍读代码时可以边读边回来查；建议把 `include/linux/sched.h` 和 `include/linux/fs.h` 当"字典"放在手边。

---

## 3. 项目目录结构详解

```
linux-0.01/
├── Makefile                  # 顶层构建：make 产出 Image（可引导内核镜像）
├── boot/                     # ⭐ 引导阶段（汇编，最先执行）
│   ├── boot.s                # ⭐ 512字节引导扇区：BIOS 加载它，它加载内核、进保护模式
│   └── head.s                # ⭐ 32位起步代码：建 IDT/GDT/页表，跳入 main()
├── init/
│   └── main.c                # ⭐ 内核 C 入口：初始化全部子系统 → 降权 → fork 出 init
├── kernel/                   # ⭐ 内核核心（调度/系统调用/驱动 全在这）
│   ├── sched.c               # ⭐⭐ 调度器总纲：schedule()/sleep_on/wake_up/时钟处理
│   ├── system_call.s         # ⭐⭐ int 0x80 总入口 + 时钟中断 + 硬盘中断（汇编）
│   ├── fork.c                # ⭐⭐ 进程复制：copy_process + 地址空间复制
│   ├── exit.c                # ⭐  进程退出与 waitpid（僵尸进程回收机制）
│   ├── sys.c                 # ⭐  简单系统调用集合（time/uid/brk/setsid…）
│   ├── traps.c               # ⭐  CPU 异常处理登记（除0/缺页保护…全走 die 打印现场）
│   ├── asm.s                 # ⭐  CPU 异常的低级汇编入口（保存现场 → 调 traps.c）
│   ├── mktime.c              # ·  CMOS 时间 → Unix 时间戳换算
│   ├── panic.c               # ·  内核死刑宣告（打印后死循环）
│   ├── printk.c              # ·  内核版 printf（绕过 fs 段寄存器的坑）
│   ├── vsprintf.c            # ·  格式化字符串实现（%d %s %x…）
│   ├── console.c             # ⭐  VGA 文本控制台 + VT102 转义序列解析
│   ├── tty_io.c              # ⭐  TTY 统一读写层（终端行规程：回显/退格/^C）
│   ├── keyboard.s            # ⭐  键盘扫描码 → ASCII 翻译（纯汇编状态机）
│   ├── serial.c              # ·  串口初始化（2400bps）
│   ├── rs_io.s               # ·  串口中断读写（纯汇编）
│   └── hd.c                  # ⭐  AT 硬盘驱动 + 请求队列电梯排序
├── mm/                       # ⭐ 内存管理
│   ├── memory.c              # ⭐⭐ 分页核心：get_free_page/页表复制/COW 缺页处理
│   └── page.s                # ⭐  缺页中断低级入口（区分"页不在"与"页只读"）
├── fs/                       # ⭐ 文件系统（MINIX FS）
│   ├── namei.c               # ⭐⭐ 路径解析 + mkdir/rmdir/unlink/link（最大文件 678 行）
│   ├── inode.c               # ⭐⭐ inode 缓存与读写（iget/iput/_bmap 三级块映射）
│   ├── buffer.c              # ⭐⭐ 缓冲区高速缓存（hash 表 + 空闲链表）
│   ├── exec.c                # ⭐⭐ a.out 格式可执行文件加载（execve 的全部实质）
│   ├── open.c                # ⭐  open/close/chmod/chown/chdir 等文件系统调用
│   ├── read_write.c          # ⭐  read/write/lseek 分发（按 inode 类型路由）
│   ├── pipe.c                # ·  管道实现（一页内存当环形缓冲区）
│   ├── file_dev.c            # ·  普通文件读写的块级搬运
│   ├── block_dev.c           # ·  块设备读写分发（ll_rw_block）
│   ├── char_dev.c            # ·  字符设备读写分发（TTY 路由）
│   ├── bitmap.c              # ·  inode/块位图分配回收
│   ├── super.c               # ·  超级块管理 + mount_root（挂根文件系统）
│   ├── stat.c                # ○  stat/fstat 系统调用
│   ├── fcntl.c               # ○  fcntl/dup/dup2
│   ├── ioctl.c               # ○  ioctl 分发表
│   ├── tty_ioctl.c           # ○  终端 ioctl（termios 设置）
│   ├── truncate.c            # ○  文件截断（释放全部数据块）
│   └── file_table.c          # ○  3行：全局 file 表定义
├── include/                  # ⭐ 头文件（读 .c 前先扫这里的结构体）
│   ├── linux/sched.h         # ⭐⭐ task_struct/tss_struct/INIT_TASK/switch_to（最重要的头）
│   ├── linux/fs.h            # ⭐⭐ buffer_head/m_inode/file/super_block（文件系统数据结构总纲）
│   ├── linux/sys.h           # ⭐  sys_call_table[67] 系统调用分发表
│   ├── linux/tty.h           # ⭐  tty_queue 环形队列宏（PUTCH/GETCH）
│   ├── linux/config.h        # ⭐  编译期配置（内存大小/根设备/硬盘型号）
│   ├── linux/hdreg.h         # ·  AT 硬盘控制器寄存器定义
│   ├── linux/head.h          # ○  desc_struct 描述符类型
│   ├── linux/mm.h            # ○  页大小与三个页函数原型
│   ├── linux/kernel.h        # ○  printk/panic 原型
│   ├── unistd.h              # ⭐⭐ _syscallN 宏 + 67 个系统调用号（用户态/内核态共用）
│   ├── asm/system.h          # ⭐  move_to_user_mode/set_gate 等关键内联汇编宏
│   ├── asm/io.h              # ○  inb/outb 端口读写宏
│   ├── asm/segment.h         # ○  get_fs_byte/put_fs_long 跨段读写
│   ├── asm/memory.h          # ○  memcpy 宏
│   ├── string.h              # ·  全套字符串函数的内联汇编实现（strcpy/strcmp…）
│   ├── a.out.h               # ·  可执行文件格式定义（exec 结构体）
│   ├── termios.h             # ○  终端参数常量（termios/termio 结构）
│   ├── signal.h              # ○  信号编号与原型
│   ├── const.h/ctype.h/errno.h/fcntl.h/stdarg.h/stddef.h/time.h/utime.h  # ○  通用常量
│   └── sys/{stat,times,types,utsname,wait}.h                              # ○  POSIX 兼容声明
├── lib/                      # ·  用户态系统调用桩（_syscallN 宏展开产物）
│   ├── write.c/close.c/dup.c/execve.c/setsid.c/_exit.c/wait.c/open.c  # ○  每个文件4~19行
│   ├── ctype.c               # ○  字符分类表 _ctype[256]
│   ├── errno.c               # ○  1行：int errno;
│   └── string.c             # ○  8行：把 string.h 的内联函数"实例化"成库
└── tools/
    └── build.c               # ⭐  构建工具：boot扇区(补0x55AA) + 内核 → 拼成 Image
```

**要点**：
- ⭐⭐ = 必须逐行精读（6 个），⭐ = 必读，· = 选读，○ = 薄文件扫一眼即可。
- 总量最大的是 `fs/`（2879 行），但**复杂度最高**的是 `mm/memory.c` 和 `kernel/sched.c`——它们分别对应 OS 课程的"内存管理"和"进程调度"两章。
- `include/` 不是配角：Linux 0.01 的核心数据结构几乎全在头文件里，`sched.h`、`fs.h`、`unistd.h` 三个头文件的含金量超过一半的 .c 文件。

---

## 4. 运行流程全景图

```
┌─────────────────────────────────────────────────────────────────────┐
│ 阶段① BIOS 加电自检（硬件时代，不属于本项目代码）                      │
│   职责: POST 自检 → 读软盘第1扇区(512B)到 0x7C00 → 跳过去执行          │
│   退出条件: 控制权交给 boot.s                                         │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 阶段② boot/boot.s —— 引导扇区（实模式，16位）                         │
│   职责: ① 把自己从 0x7C00 搬到 0x90000                               │
│         ② BIOS int 13h 把内核(软盘后续扇区)读到 0x10000              │
│         ③ 把内核整体下移到 0x00000（为页目录腾地方）                   │
│         ④ 装临时 IDT/GDT → 开 A20 地址线 → 重编程 8259 中断控制器      │
│         ⑤ 置 PE 位进入保护模式 → 长跳转到 0x00000（head.s）            │
│   退出条件: jmpi 0,8 执行后 boot.s 使命结束                           │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 阶段③ boot/head.s —— 32位起步（刚进保护模式，无页表）                  │
│   职责: ① 重装段寄存器，设内核栈（lss _stack_start）                  │
│         ② setup_idt: 256 个中断门全指向 ignore_int（占位）            │
│         ③ setup_gdt: 重装正式 GDT                                    │
│         ④ 验证 A20 真的开了、探测 387 协处理器                        │
│         ⑤ setup_paging: 清零页目录+pg0+pg1，恒等映射 8MB，置 PG 位     │
│         ⑥ 在栈上压 main 的假参数和返回地址 → ret 跳进 main()           │
│   退出条件: setup_paging 的 ret —— 注意：head.s 自身代码被页目录覆盖，  │
│            这是"用完即焚"的绝妙设计（页目录必须放 0 地址处）            │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 阶段④ init/main.c :: main() —— 内核初始化（特权级0）                  │
│   职责: time_init(CMOS时钟) → tty_init(控制台+串口) → trap_init       │
│         (17个CPU异常门) → sched_init(任务0登记+8253定时器+时钟门)      │
│         → buffer_init(缓冲区初始化) → hd_init(硬盘门) → sti(开中断)    │
│         → move_to_user_mode(伪造iret栈帧降权到用户态,进程0诞生)        │
│         → fork() → 子进程进 init(),父进程0永远 pause() 当闲逛任务      │
│   退出条件: main 永不返回（for(;;) pause()）                           │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 阶段⑤ init/main.c :: init() —— 第一个真正的用户进程                   │
│   职责: setup()(读分区表→mount_root挂根FS) → fork出update守护进程      │
│         → 开/dev/tty0,dup×2得到stdin/stdout/stderr                    │
│         → 打印缓冲区信息 → fork出shell(/bin/sh) → wait收尸             │
│         → sync落盘 → _exit(0)（init 死后由内核接管）                   │
│   退出条件: shell 运行后,系统进入正常服务状态                          │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 阶段⑥ 常态运行：中断驱动的循环世界                                     │
│   时钟中断(20Hz…实际100Hz): jiffies++ → do_timer 计账 → 时间片耗尽     │
│       则 schedule() 挑下一个进程 → switch_to 硬件任务切换              │
│   键盘中断: keyboard.s 扫描码→ASCII → read_q → copy_to_cooked 行规程   │
│       → secondary 队列 → 唤醒 tty_read 里睡着的 shell                 │
│   系统调用: 用户程序 int $0x80 → _system_call → sys_call_table[n]     │
│       → 返回前查信号位图 → iret 回用户态                              │
│   缺页中断: page.s → do_no_page(分配新页) / do_wp_page(COW复制)        │
│   磁盘中断: hd.c read_intr/write_intr → 唤醒等缓冲区的进程             │
└─────────────────────────────────────────────────────────────────────┘
```

**要点**：
- 前 5 个阶段是"一条线"（每步只走一次），阶段⑥ 是"一个环"（中断 + 调度循环，操作系统的常态）。
- 最容易被忽视但最精妙的一跳：`move_to_user_mode()`——用手工伪造的中断返回栈帧执行 iret，让进程 0 "假装从中断里返回"从而降级到用户态。理解它就理解了"特权级切换"的本质。
- 全景图里每个箭头都对应具体代码，第 5 章的逐文件导读会给出精确行号。

---

## 5. 逐文件代码导读

> 阅读顺序总原则：**按系统启动的时间轴读**（引导 → 初始化 → 调度 → 内存 → 文件系统 → 驱动），每个新文件都建立在前一个的概念上。
> 耗时按精读（逐行+理解）估计；"扫读"指只看结构和注释。

### 5.1 启动链（建议第 1 天读完）

| 顺序 | 文件 | 行数 | 作用 | 关键内容 | 耗时 |
|---|---|---|---|---|---|
| 1 | `boot/boot.s` | 329 | 引导扇区 | 自搬运、int 13h 读盘、GDT 雏形、A20、8259 重编程、lmsw 进保护模式 | ~60min |
| 2 | `boot/head.s` | 179 | 32位起步 | setup_idt/setup_gdt/setup_paging、A20 校验、协处理器探测、伪造 main 栈帧 | ~60min |
| 3 | `init/main.c` | 147 | 内核 C 入口 | time_init、五个 xxx_init、move_to_user_mode、fork+pause | ~40min |

**先读这 3 个再往下走**——它们回答"操作系统是怎么醒来的"。

### 5.2 进程与调度（第 2~3 天）

| 顺序 | 文件 | 行数 | 作用 | 关键内容 | 耗时 |
|---|---|---|---|---|---|
| 4 | `include/linux/sched.h` | 230 | 调度数据结构总纲 | task_struct 全字段、tss_struct、INIT_TASK、switch_to 宏、TSS/LDT 选择子计算 | ~60min |
| 5 | `kernel/sched.c` | 254 | 调度器核心 | schedule() 轮转+counter 时间片、sleep_on/interruptible_sleep_on 链式睡眠、do_timer、sched_init | ~60min |
| 6 | `kernel/system_call.s` | 219 | 系统调用+时钟中断汇编 | _system_call 栈布局、sys_call_table 间接调用、ret_from_sys_call 信号分发、_timer_interrupt | ~60min |
| 7 | `kernel/fork.c` | 136 | 进程创建 | find_empty_process、copy_process（TSS/LDT 填充、copy_mem 页表复制）、verify_area | ~45min |
| 8 | `kernel/exit.c` | 135 | 进程终止 | do_exit 清理序列、ZOMBIE 状态、sys_waitpid 收尸、do_kill 四种 pid 语义 | ~45min |
| 9 | `kernel/traps.c` | 199 | CPU 异常处理 | die() 打印现场（栈/寄存器/指令字节）、trap_init 登记 17 个异常门 | ~30min |
| 10 | `kernel/asm.s` | 157 | 异常低级入口 | no_error_code/error_code 两套现场保存模板 | ~30min |
| 11 | `kernel/sys.c` | 216 | 杂项系统调用 | setuid/setgid 的权限模型、sys_time/stime、sys_brk、sys_setsid | ~30min |

### 5.3 内存管理（第 4 天，本项目的"明珠"）

| 顺序 | 文件 | 行数 | 作用 | 关键内容 | 耗时 |
|---|---|---|---|---|---|
| 12 | `mm/memory.c` | 264 | 分页管理核心 | mem_map 物理页位图、get_free_page（从高往低扫）、copy_page_tables（COW 关键：子页清写位）、un_wp_page/do_wp_page、put_page | ~90min |
| 13 | `mm/page.s` | 34 | 缺页中断入口 | 区分 page-not-present 与 write-protect，分发到 do_no_page/do_wp_page | ~15min |

**要点**：`copy_page_tables` 里 `this_page &= ~2`（清写位）就是整个 COW 机制的"扣扳机"动作，与 `do_wp_page` 前后呼应，务必连起来读。

### 5.4 文件系统（第 5~7 天，代码量最大）

| 顺序 | 文件 | 行数 | 作用 | 关键内容 | 耗时 |
|---|---|---|---|---|---|
| 14 | `include/linux/fs.h` | 185 | FS 数据结构总纲 | buffer_head/m_inode/d_inode/file/super_block/dir_entry、MINIX 常量 | ~40min |
| 15 | `fs/super.c` | 102 | 超级块与挂根 | do_mount 读 1 号块校验 magic 0x137F、mount_root 初始化位图统计 | ~30min |
| 16 | `fs/buffer.c` | 254 | 缓冲区缓存 | hash(dev,block) 定位、getblk 的免费块搜索+竞态重试、bread/brelse、buffer_init 巧用内存末端 | ~60min |
| 17 | `fs/inode.c` | 288 | inode 管理 | inode_table[32]、iget/iput 引用计数、_bmap 三级块映射（直接/一重/二重间接）、get_pipe_inode | ~75min |
| 18 | `fs/namei.c` | 678 | 路径解析+目录操作 | permission、match/find_entry/add_entry、get_dir/dir_namei/namei/open_namei、mkdir/rmdir/unlink/link | ~120min |
| 19 | `fs/open.c` | 188 | 文件打开关闭 | sys_open 三层资源分配（fd/file_table/inode）、tty 特殊处理、sys_close | ~45min |
| 20 | `fs/read_write.c` | 97 | 读写分发 | sys_read/write 按 inode 类型路由到管道/字符/块/普通文件 | ~25min |
| 21 | `fs/exec.c` | 306 | 程序加载 | ZMAGIC 校验、copy_strings 参数页、change_ldt 重设 LDT、read_area 加载代码数据、换 eip/esp "魔法时刻" | ~75min |
| 22 | `fs/bitmap.c` | 158 | 位图分配 | free_block/new_block/free_inode/new_inode（inode 位图与块位图同一套模式） | ~40min |
| 23 | `fs/pipe.c` | 92 | 管道 | 一页内存做环形缓冲、i_count==2 读写双方在场的检查、SIGPIPE | ~30min |
| 24 | `fs/file_dev.c` | 84 | 普通文件读写 | file_read/file_write 的块搬运循环、O_APPEND 处理 | ~25min |
| 25 | 其余 8 个薄文件 | ~530 | 各类辅助 | block_dev/char_dev（分发表）、stat/fcntl/ioctl/tty_ioctl（小系统调用）、truncate、file_table（3行） | ~90min 合计 |

### 5.5 驱动与终端（第 8~9 天）

| 顺序 | 文件 | 行数 | 作用 | 关键内容 | 耗时 |
|---|---|---|---|---|---|
| 26 | `include/linux/tty.h` | 74 | TTY 结构 | tty_queue 环形队列与 PUTCH/GETCH 宏、tty_struct 三队列模型 | ~20min |
| 27 | `kernel/tty_io.c` | 306 | TTY 统一层 | tty_table[3]（控制台+两串口）、copy_to_cooked 行规程（回显/退格/^S/^Q）、tty_read 的 VTIME/VMIN 超时模型 | ~75min |
| 28 | `kernel/console.c` | 550 | VGA 控制台 | 0xb8000 显存直写、CSI 转义序列状态机（state 0→4）、滚屏 scrup 的显存搬移 | ~90min |
| 29 | `kernel/keyboard.s` | 409 | 键盘驱动 | 扫描码→key_map/shift_map/alt_map 三张表、ctrl/alt/caps 状态位、Ctrl-Alt-Del 重启 | ~60min |
| 30 | `kernel/hd.c` | 413 | AT 硬盘 | hd_out 命令序列、read_intr/write_intr 中断续传、add_request 电梯排序、sys_setup 读分区表 | ~75min |
| 31 | `kernel/serial.c`+`rs_io.s` | 194 | 串口 | 8250 初始化序列、rs 中断四路分发表 | ~40min |

### 5.6 工具链与库（第 10 天，扫读为主）

| 顺序 | 文件 | 行数 | 作用 | 耗时 |
|---|---|---|---|---|
| 32 | `include/unistd.h` | 247 | _syscallN 宏 + 系统调用号（回看 2.5 节） | ~30min |
| 33 | `include/asm/system.h` | 66 | move_to_user_mode/set_gate 宏精读 | ~30min |
| 34 | `kernel/vsprintf.c` | 227 | printf 格式化实现 | ~40min |
| 35 | `tools/build.c` | 68 | 镜像拼接（boot 扇区补 0x55AA 签名） | ~15min |
| 36 | `kernel/mktime.c`/`panic.c`/`printk.c` | 96 | 时间换算/内核死循环/内核打印 | ~20min |
| 37 | `lib/` 11 个文件 | 94 | _syscall 宏展开的桩函数 | ~20min |
| 38 | `include/` 其余头文件 | ~1500 | 常量表/POSIX 兼容声明/内联汇编 string 函数 | 按需查阅 |

**quick_start 快速上手 3 文件**（骨架分析结果 + 人工语义修正后的建议）：
1. **入口**：`init/main.c` —— 从名字就能看出是主入口，所有初始化序列一目了然（骨架算法选了 fs/exec.c，但那是依赖图视角；对学习者 main.c 更符合"入口"直觉，两者可以互相印证：main.c fork 出的进程最终通过 exec.c 变成 shell）。
2. **中间**：`kernel/sched.c` —— 被 boot/init/fork/system_call 多方引用的"调度中枢"，连接启动代码与运行时。
3. **核心**：`include/linux/sched.h` —— 全项目入度最高的文件（几乎每个 .c 都包含它），task_struct 是所有内核代码的公共语言。

**要点**：
- 38 个文件 ≈ 38 小时精读 ≈ 每天 4 小时读 10 天。快的话 6 天（薄文件全扫读）。
- 读每个 .c 之前先把它 include 的 linux/*.h 扫一遍，结构体不认识就读不下去。
- 本项目注释完成后，每个文件的文件头都会有一段"本文件在全景图中的位置"总纲注释，可对照阶段①~⑥的图。

---

## 6. 关键设计模式解析

Linux 0.01 里反复出现的 6 个设计手法——它们不是教科书模式，而是内核工程的"惯用手筋"，读懂它们读懂一切。

### 6.1 函数指针分发表（整个内核的"交换机"）

```
                 ┌──────────────────────────────┐
   int 0x80 ───► │ sys_call_table[67] (sys.h)   │──► sys_fork / sys_read / ...
                 └──────────────────────────────┘
                 ┌──────────────────────────────┐
   中断号 0x21 ─► │ idt[256] (head.s 建)          │──► keyboard_interrupt / timer...
                 └──────────────────────────────┘
                 ┌──────────────────────────────┐
   设备类型 ───► │ crw_table / rd_blk / ioctl_table │──► rw_ttyx / rw_hd / ...
                 └──────────────────────────────┘
```
- **意图**：把"编号 → 处理函数"的映射收敛成一张表，入口代码只负责查表跳转。
- **代码**：`include/linux/sys.h:sys_call_table`、`fs/char_dev.c:crw_table`、`fs/block_dev.c:rd_blk`、`fs/ioctl.c:ioctl_table`。
- **为什么重要**：现代内核的 file_operations 结构体就是它的直系后代。

### 6.2 TSS 长跳转任务切换（硬件帮忙换班）

```
switch_to(n):
    cmpl %ecx,_current    # 目标就是当前任务? 什么都不用做
    je 1f
    xchgl %ecx,_current   # 换 current 指针
    movw %%dx,%1          # 填好 48 位远跳目标 (TSS 选择子,偏移0)
    ljmp %0               # ★ 一句 ljmp，CPU 硬件自动：把全部寄存器存旧TSS、从新TSS装载
    ...
```
- **意图**：把进程切换完全托付给 386 硬件——软件只需一句远跳转到目标进程的 TSS 段。
- **代码**：`include/linux/sched.h` 的 `switch_to` 宏。
- **历史注**：后来的 Linux 放弃了硬件 TSS 切换（移植性差、慢），改用软件保存寄存器，但 TSS 仍用于特权级切换的 esp0。

### 6.3 写时复制 COW（fork 的省钱之道）

```
fork() 时:  copy_page_tables() 把父页表项复制给子进程，
            两边都 this_page &= ~2 (清写位) + mem_map[]++  → 只读共享
    ↓
任何一方写:  CPU 检测只读页写入 → 缺页中断(错误码 bit1=1)
    ↓
page.s → do_wp_page → un_wp_page:
    if (物理页引用计数==1) 直接恢复写位 return;   # 只有自己在用，白送的写权限
    else 新分配一页 + copy_page + 原页引用计数--    # 真正复制
```
- **意图**：fork 不复制内存，只复制页表并标记只读；等真有人写时才复制那一页。
- **代码**：`mm/memory.c:copy_page_tables / do_wp_page / un_wp_page`。
- **这是现代操作系统 COW 的完整原型**，虽然 0.01 的 fork 实际还是复制页表映射（不复制页内容），但 COW 触发链已经完整。

### 6.4 睡眠队列与唤醒（最朴素的同步原语）

```
sleep_on(p):                    # p 指向"等待队列头"（其实是单个指针）
    tmp = *p;                   # 记住当前排队者
    *p = current;               # 自己顶到队头
    current->state = UNINTERRUPTIBLE;
    schedule();                 # 让出 CPU
    if (tmp) tmp->state = 0;    # 醒来后把前任也唤醒 —— 链式唤醒！

wake_up(p):
    (**p).state = 0; *p = NULL; # 只唤醒队头一个
```
- **意图**：用**一个指针**实现的隐式等待队列：新的睡眠者把老的"记下来"，醒来时连锁叫醒。没有链表节点、没有 mutex，全靠调度时序保证正确。
- **代码**：`kernel/sched.c:sleep_on / interruptible_sleep_on / wake_up`。
- **精髓**：`interruptible_sleep_on` 的 repeat 循环——被信号唤醒后发现队头不是自己，就替真正的队头叫醒后继续睡，保证队头优先。

### 6.5 环形缓冲区宏（TTY 的每秒百万次心跳）

```
struct tty_queue { data; head; tail; proc_list; buf[1024]; }
#define INC(a) ((a) = ((a)+1) & (TTY_BUF_SIZE-1))   # 加1后按1024取模(位运算)
#define GETCH(q,c)  c = q.buf[q.tail]; INC(q.tail)
#define PUTCH(c,q)  q.buf[q.head] = c; INC(q.head)
```
- **意图**：键盘→行规程→设备，三级队列全是 1024 字节环形缓冲；size 是 2 的幂，取模用 & 完成，中断里也要跑得飞快。
- **代码**：`include/linux/tty.h`，消费方 `kernel/tty_io.c`、`keyboard.s`（汇编里 head=4/tail=8 偏移量**硬编码**，改结构体必须同步改汇编——header 注释里 Linus 特别警告过）。

### 6.6 用编译期配置 + 宏内联汇编（C 与硬件的缝合线）

```
move_to_user_mode():  伪造 iret 栈帧，从内核态"返回"到用户态（进程0 降权的唯一通道）
set_intr_gate/set_trap_gate/set_system_gate:  一个宏填 IDT 门，只差 DPL 一个参数
get_fs_byte/put_fs_long:  内核访问用户内存必须借道 fs 段（fs 指向用户数据段）
```
- **意图**：386 的特权模型决定了"内核→用户"不能直接跳，只能借中断返回的"下楼梯"；内核读用户指针也必须用带段前缀的指令，否则读到内核自己的内存。
- **代码**：`include/asm/system.h`、`include/asm/segment.h`、`include/asm/io.h`。
- **要点**：读不懂这些宏时，手画一下栈布局，或者对照 4.全景图阶段④。

---

## 7. 配置系统详解

Linux 0.01 没有 Kconfig，所有配置**写死在 `include/linux/config.h`**，改配置要重编译。

| 配置项 | 值 | 位置 | 说明 |
|---|---|---|---|
| `LINUS_HD` / `LASU_HD` | 二选一 | config.h | 硬盘类型预设（Linus 的盘 vs 他的朋友 Lars 的盘） |
| `HIGH_MEMORY` | 0x800000 (8MB) | config.h | 物理内存总量上限，页表常量的计算基准 |
| `BUFFER_END` | 0x200000 (2MB) | config.h | 缓冲区可用的最高物理地址 |
| `ROOT_DEV` | 0x306 | config.h | 根文件系统设备（0x306 = 第二硬盘第1分区，major=3 minor=6） |
| `HD_TYPE` | {5,17,980,300,980,0}×2 | config.h | 硬盘几何参数：磁头/扇区/柱面/写预补偿/着陆区/控制字 |
| `HARD_DISK_TYPE` | 17 | hdreg.h | BIOS 硬盘类型表编号（17 型 = 977柱/5头/17扇区） |
| `NR_TASKS` | 64 | sched.h | 最大进程数 |
| `HZ` | 100 | sched.h | 时钟中断频率（每秒 100 次） |
| `NR_OPEN/NR_INODE/NR_FILE/NR_SUPER` | 20/32/64/8 | fs.h | 每进程打开文件数/inode 缓存数/file 表大小/可挂载 FS 数 |
| `NR_HASH/NR_BUFFERS` | 307/动态 | fs.h + buffer.c | 缓冲 hash 桶数 / 缓冲头数（buffer_init 按内存余量算） |
| `BLOCK_SIZE` | 1024 | fs.h | 文件系统块大小 |
| `NAME_LEN` | 14 | fs.h | 文件名最大长度（MINIX 限制） |
| `PAGE_SIZE` | 4096 | mm.h | 页大小（386 硬性规定） |

**加载顺序**：编译期写死 → 无运行时配置 → 启动后唯一的"动态配置"是 CMOS 时钟（main.c:time_init）读出的系统时间。

**要点**：想改内存大小（如 4MB→8MB 以上）要同时改 config.h 和 boot/head.s 里的页表数量，Linus 在两处都留了注释互相提醒——这是理解"编译期配置系统"局限性的活教材。

---

## 8. 如何运行和测试

> 原版需要 1991 年的工具链（gcc 1.40 + bin86），在现代机器上**编译原样代码已不可行**。学习者的三种实践路径：

**路径 A：QEMU 直接跑现成镜像（推荐，10 分钟见效）**

1. 准备：安装 QEMU（Windows: qemu.org 下载；Linux: `apt install qemu-system-x86`）。
2. 获取一份已构建好的 linux-0.01 Image + MINIX 根文件系统软盘/硬盘镜像（oldlinux.org 等存档站有历史镜像）。
3. 运行：
   ```
   qemu-system-i386 -fda bootimage -hda minix.rootimage -boot a
   ```
   （具体参数随镜像作者说明略有差异，原理是：软盘放引导+内核，硬盘放 MINIX 根文件系统。）
4. 验收点：能看到 "Loading system..." → 内核打印 "Partition table ok" → shell 提示符，敲 `ls` 看到目录列表，就说明这条 1 万行的链路全部活着。

**路径 B：现代工具链重编译（进阶，需打补丁）**

- gcc 1.40 早已绝迹，社区有移植补丁集（如 Github 上多个 "linux-0.01-compilable" 项目用 gcc 3.x/4.x + bin86 编译通过）。思路：把 `as86/ld86` 换成 bin86 包（仍可安装），C 代码补 `volatile`、修复原型告警即可。适合想动手魔改的读者。
- 注意：`tools/build.c` 是**宿主机**程序（用现代 gcc 编译没问题），它把 boot 扇区和内核拼成 Image。

**路径 C：只读精读 + 手绘（零环境成本）**

- 按 ZHIDAO 第 9 章路线精读，配一张纸画启动栈变化/页表映射/任务切换时序。读内核最大的成本不是跑不起来，而是"以为自己懂了"。

**验收测试点**（无论哪条路径，这些现象能对应回具体代码）：

| 现象 | 对应代码 |
|---|---|
| 屏幕左上角 "Loading system ..." | boot.s: msg1 + int 0x10 |
| 无数个小白点闪烁 | head.s: ignore_int（未注册中断被触发） |
| "Partition table ok." | hd.c: sys_setup |
| "xxx buffers = xxx bytes buffer space" | init/main.c: init() |
| shell 提示符与回显 | console.c + tty_io.c: copy_to_cooked |
| Ctrl-Alt-Del 重启 | keyboard.s: cursor → reboot |

---

## 9. 复刻建议与学习路线

**总路线**：先通读（本指南第 5 章顺序）→ 再提笔复刻。复刻不是抄写，是"先背下骨架再默写，默写不出来=没读懂"。

### 阶段 1：通读期（1~2 周）

| 周 | 内容 | 产出 |
|---|---|---|
| 第1周前半 | 启动链 3 文件 + sched.h/sched.c（速览路线核心） | 手绘：BIOS→main 的完整栈变化图 |
| 第1周后半 | fork/exit/system_call.s + mm/memory.c | 手绘：fork 时页表变化图；COW 缺页时序图 |
| 第2周 | fs 四大件（buffer/inode/namei/exec）+ 你最感兴趣的驱动 | 手绘：open("/bin/sh") 的调用链图 |

### 阶段 2：最小复刻期（2~4 周）——推荐顺序

1. **引导**：写 boot.s（自己搬到 0x90000、读盘、进保护模式）+ head.s（建页表）。验收：能从 0 跳进你自己的 main 并打印字符。耗时 ~3 天。
2. **内存**：实现 get_free_page/put_page/free_page + 缺页处理。验收：main 里 malloc 一段内存能读写不崩。耗时 ~4 天。
3. **多任务**：task_struct + schedule + switch_to + 时钟中断。验收：两个进程轮流在屏幕上打印 A/B。耗时 ~5 天。
4. **系统调用**：int 0x80 + sys_call_table + write。验收：用户态程序 printf 一句话。耗时 ~2 天。
5. **文件系统**（可选进阶）：MINIX FS 只读实现。验收：ls 列出根目录。耗时 ~1 周。

### 阶段 3：扩展期（按兴趣）

- 加 COW 完善（0.01 的 fork 是全页表复制+COW 雏形混合体，可以做成纯 COW）
- 加 ELF 支持（替代 a.out）
- 加第二个虚拟控制台（Alt-F1/F2）
- 移植到现代编译器并跑 QEMU（工程量最大但简历最亮）

### 学习资源表

| 资源 | 类型 | 用途 |
|---|---|---|
| 《Linux内核完全注释》(赵炯) | 中文注释书 | 与本项目逐行对应，第 5 章卡壳时的"参考答案" |
| 《操作系统导论》(OSTEP) | 教材 | 三大主题（虚拟化/并发/持久化）的理论框架 |
| 《Linux内核设计与实现》 | 教材 | 现代内核对照，看同一机制 30 年后长什么样 |
| Intel 80386 Programmer's Reference Manual | 硬件手册 | 分段/分页/中断的权威定义（读 head.s 必备） |
| oldlinux.org | 网站 | 原版镜像、历史邮件存档 |
| Linus 1991 comp.os.minix 发帖 | 历史文献 | "just a hobby, won't be big..." |

---

## 10. 常见问题

**Q1：为什么 task_struct 用 union 和内核栈挤在一页里？**
A：`union task_union { struct task_struct task; char stack[PAGE_SIZE]; }` —— 一页 4KB，低端是档案、高端是栈，栈向下生长。好处：切换任务时只需换一个页地址，档案和栈"打包带走"，且天然隔离各进程的内核栈（不用动态分配）。现代 Linux 仍沿用此布局（thread_info + kernel stack）。

**Q2：schedule() 里 `c = -1; if (c) break;` 是什么套路？**
A：先在 64 个任务里找 counter 最大的可运行任务。若最大 counter>0（c>0）直接切过去；若全为 0（c==0），说明时间片耗尽，进入 for 循环把所有任务的 counter 重新充能：`counter = counter/2 + priority`（运行中的进程衰减慢、睡眠的进程衰减快，形成"IO 密集型进程优先"的隐性奖励），然后重新挑选。这是最早的"多级反馈"思想雏形。

**Q3：为什么内核要 set 但用户不能 set 的门（trap gate vs system gate）？**
A：IDT 门描述符的 DPL 字段决定"谁有权触发"。trap gate DPL=0 只有内核能触发（如缺页，用户态触发不了是自己撞上的）；system gate DPL=3 允许用户态主动触发（int3 断点、int 0x80 系统调用）。看 traps.c 里 `set_trap_gate` 与 `set_system_gate` 的分布就懂了。

**Q4：fork 返回后"一次调用两次返回"是怎么做到的？**
A：copy_process 里 `p->tss.eax = 0`——子进程的 eax 强制清零；父进程的 eax 是 copy_process 的返回值 last_pid（>0）。iret 之后两个进程从同一条指令继续，但各自 eax 不同，于是 `if (!fork())` 一个走 if 一个走 else。这是"用寄存器传两份答案"的经典手法。

**Q5：head.s 为什么敢把页目录放在 0 地址覆盖自己？**
A：因为 setup_paging 是"最后一句 ret 跳进 main"的，页目录一旦建好，head.s 的代码就再也不需要了。0 地址是页目录的法定位置（CR3=0），与其把启动代码搬走，不如直接覆盖——省 4KB 还少一次拷贝。Linus 的注释也明说了 "will be overwritten by the page tables"。

**Q6：mm/memory.c 的 put_page 为什么注释说 "uses the fact that _pg_dir=0"？
A：页目录在物理地址 0，而内核线性地址=物理地址（恒等映射），所以 `(address>>20)&0xffc` 算出的页目录项偏移可以直接当指针用，不用加任何基址。若页目录不放 0，这套指针运算全部失效——这是全内核最隐晦的隐含约定。

**Q7：为什么 syscall 返回前要检查 `CS & 3` 和 `OLDSS == 0x17`？**
A：ret_from_sys_call 只处理"从用户态进来的"信号分发。CS 低 2 位=3 表示之前在用户态（ring3），0x17 是用户数据段选择子。内核态任务自己调系统调用（如 init 进程内部）不需要信号分发，直接返回——这个判断避免了在错误的栈上重建信号帧。

**Q8：boot.s 里那么多 `.word 0x00eb` 是什么？**
A：编码后的 `jmp $+2`（跳过下一条指令）。8259A 中断控制器对连续写端口之间的时序有要求，插入这两字节相当于"喘口气"的延时，比计数循环省空间。

**Q9：为什么 exit.c 的 do_exit 结尾 `return (-1)` 永远不会执行还要写？**
A：do_exit 内部最后 schedule() 后该进程已是 ZOMBIE 或被释放，不可能回来。这行只为压掉 gcc 的"控制到达非 void 函数末尾"警告——1991 年的编译器没有 noreturn 属性。

**Q10：想给这个内核加个功能，从哪下手？**
A：小步走：① 先加系统调用（unistd.h 加号 → sys.h 表加指针 → 实现函数），② 再碰调度策略（改 schedule 的挑选条件），③ 最后动内存管理。每步都用 QEMU 跑通再进行下一步。

---

## 附录：关键术语对照表

| 英文 | 中文 | 说明 |
|---|---|---|
| Boot sector | 引导扇区 | 磁盘第一个 512 字节，末尾必须 0x55AA |
| Real mode / Protected mode | 实模式 / 保护模式 | 16位无保护 / 32位分段+分页 |
| GDT (Global Descriptor Table) | 全局描述符表 | 全系统段描述符总册 |
| LDT (Local Descriptor Table) | 局部描述符表 | 每进程私有段表 |
| TSS (Task State Segment) | 任务状态段 | 硬件任务切换的现场保存区 |
| Selector | 段选择子 | 段寄存器里的值，指向描述符表条目 |
| Page Directory / Page Table | 页目录 / 页表 | 两级地址翻译结构，各 1024 项 |
| Page Fault | 缺页异常 | int 14，COW 与按需分配的触发器 |
| COW (Copy-On-Write) | 写时复制 | 共享只读，写时才复制 |
| Task switch | 任务切换 | 换 CPU 现场，0.01 用 ljmp TSS 实现 |
| Time slice | 时间片 | counter 字段，每 tick 减 1 |
| Jiffy | 时钟节拍 | jiffies 变量，HZ=100 即 10ms 一个 |
| System call | 系统调用 | int 0x80，用户进内核的唯一正门 |
| IDT (Interrupt Descriptor Table) | 中断描述符表 | 256 个门：异常+硬件中断+系统调用 |
| Gate (Trap/Interrupt/System) | 门 | IDT 表项类型，DPL 决定触发权限 |
| PIC 8259A | 可编程中断控制器 | 硬件中断汇聚器，主从两级 |
| Timer 8253/8254 | 定时器芯片 | 产生 100Hz 时钟节拍的硬件源 |
| Buffer cache | 缓冲区高速缓存 | 磁盘块的内存缓存 |
| Inode | 索引节点 | 文件元数据（大小/权限/块位置） |
| Super block | 超级块 | 文件系统全局信息（magic/位图位置） |
| Zone | 区块 | MINIX FS 的数据块单位（1KB） |
| i_zone[9] | 块索引数组 | 7直接 + 1一重间接 + 1二重间接 |
| TTY | 终端 | 控制台/串口的统一抽象 |
| Line discipline | 行规程 | 终端输入的"编辑层"（回显/退格/^C） |
| Scan code | 扫描码 | 键盘硬件编码，需查表变 ASCII |
| VGA text mode | VGA 文本模式 | 0xB8000 显存，80×25 字符，每字符2字节 |
| a.out | 旧可执行格式 | exec 结构体 + ZMAGIC 魔数 0413 |
| Zombie | 僵尸进程 | 已死未收尸，TASK_ZOMBIE 状态 |
| Orphan | 孤儿进程 | 父先亡，被 reparent 给进程0 |
| SIGCHLD | 子进程终止信号 | waitpid 的唤醒源 |
| Schedule | 调度 | schedule()，选下一个可运行任务 |
| Idle task | 闲逛任务 | 进程0，无活可干时的兜底 |
| Kernel panic | 内核恐慌 | panic()，打印后死循环等人工复位 |

---

*本导读基于对全部 83 个源文件的完整通读编写。配合逐文件中文注释阅读效果最佳——注释会精确到每个分支、每次端口操作、每个字段。*
