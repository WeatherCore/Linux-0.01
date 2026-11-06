# 📚 Linux 0.01 逐文件精读打卡清单（STUDY_CHECKLIST）

> 本清单覆盖全部 **83 个源文件**（.c/.s/.h，不含 Makefile），每个文件一个 checkbox，读完打勾。
> 功能说明基于源码通读 + ZHIDAO.md 全景图编写，以每个文件头部的中文总纲注释为最终准。
> 配套文档：`ZHIDAO.md`（项目地图）。两者关系：ZHIDAO 是地图，本清单是逐页书签。

---

## 一、使用说明

### 分级标记

| 标记 | 层级 | 判据 | 读法 | 单文件耗时 |
|---|---|---|---|---|
| ⭐⭐ | 必读·核心 | 全内核机制的承重墙（调度器、页管理、系统调用、路径解析） | 逐行精读 | 60–120 分钟 |
| ⭐ | 必读 | 启动链、进程生命周期、核心数据结构 | 逐行精读 | 30–75 分钟 |
| · | 选读 | 有完整逻辑但非主线（分发表、工具函数、辅助驱动） | 读文件头总纲+关键函数 | 10–40 分钟 |
| ○ | 薄文件 | 常量表、POSIX 声明、4~20 行的桩文件 | 扫一眼注释 | 1–5 分钟 |

### 勾选规则

- [ ] 精读完该文件（含对照 ZHIDAO 手绘/复述该文件机制）→ 改成 `[x]`
- 每个模块末尾有**自检题**，答不出 → 回到该模块 ⭐⭐ 文件重读对应函数
- 路径均相对项目根目录；行数是原始代码行数（注释版会更长）

### 时间预算总表

| 模块 | 文件数 | ⭐⭐ | ⭐ | · | ○ | 合计耗时 |
|---|---|---|---|---|---|---|
| boot/ | 2 | 0 | 2 | 0 | 0 | ~2 小时 |
| init/ | 1 | 0 | 1 | 0 | 0 | ~40 分钟 |
| kernel/ 核心与进程 | 8 | 2 | 5 | 1 | 0 | ~6.5 小时 |
| kernel/ 工具函数 | 4 | 0 | 0 | 4 | 0 | ~1.5 小时 |
| kernel/ 驱动 | 6 | 0 | 4 | 2 | 0 | ~4.5 小时 |
| mm/ | 2 | 1 | 1 | 0 | 0 | ~1.75 小时 |
| fs/ | 18 | 4 | 3 | 5 | 6 | ~9 小时 |
| include/ | 29 | 2 | 2 | 3 | 22 | ~4 小时（按需） |
| lib/ | 11 | 0 | 0 | 0 | 11 | ~30 分钟 |
| tools/ | 1 | 0 | 0 | 1 | 0 | ~15 分钟 |
| **合计** | **83** | **9** | **22** | **16** | **39** | **~30 小时精读** |

---

## 二、阅读路线骨架（与 ZHIDAO.md 第 4 章全景图对应）

| 步 | 目标 | 对应全景图阶段 | 模块 |
|---|---|---|---|
| 0 | 建立全局心智（纯文档，零代码） | — | ZHIDAO.md 全文 + sched.h/fs.h 扫结构体 |
| 1 | 操作系统如何醒来 | 阶段①②③ | boot/ + init/ |
| 2 | 进程的诞生与调度 | 阶段④ + ⑥环 | include/linux/sched.h + kernel/sched.c |
| 3 | 系统调用机器 | 阶段⑥环 | unistd.h + system_call.s + sys.h |
| 4 | 进程一生：fork 与 exit | 阶段⑥环 | fork.c + exit.c |
| 5 | 内存：分页与 COW | 阶段⑥环 | mm/ |
| 6 | 异常与信号 | 阶段⑥环 | asm.s + traps.c + signal.h |
| 7 | 文件系统四层楼 | 阶段⑤起 | fs.h + super/buffer/inode/bitmap |
| 8 | 路径与目录操作 | — | namei.c |
| 9 | 打开、读写、执行 | — | open/read_write/file_dev/exec/pipe |
| 10 | 终端与驱动 | 阶段⑥环 | tty.h/tty_io/console/keyboard/serial/hd |
| 11 | 杂项收尾 | — | sys.c/mktime/printk/vsprintf/lib/tools |

---

## 三、第 0 步：全局认知（零代码门槛，~1 小时）

- [ ] ○ ZHIDAO.md — 全文通读，重点：第 2 章七个核心概念、第 4 章全景图
- [ ] ○ include/linux/sched.h — 只扫 task_struct 字段名（不深究）
- [ ] ○ include/linux/fs.h — 只扫 buffer_head/m_inode/file/super_block 字段名

**自检**：能不看书画出"电源→shell"六阶段流程图吗？能说出 task_struct 里 state/counter/pid/filp/ldt/tss 各管什么吗？

---

## 四、第 1 步：启动链（boot/ + init/）

- [ ] ⭐ boot/boot.s (329行) — 512字节引导扇区：自搬运0x90000、BIOS读盘、下移内核、临时GDT、A20、重编程8259、lmsw进保护模式
- [ ] ⭐ boot/head.s (179行) — 32位起步：setup_idt/gdt、A20校验、协处理器探测、setup_paging恒等映射8MB、伪造main栈帧ret跳入C世界
- [ ] ⭐ init/main.c (147行) — 内核C入口：time_init→五个init→sti→move_to_user_mode→fork；init()挂根FS、开tty、fork shell

**自检**：
1. boot.s 为什么要先把自己搬到 0x90000 再读内核？
2. head.s 的代码会被页目录覆盖，它怎么保证被覆盖前已跳走？
3. move_to_user_mode 不用任何中断，凭什么能从内核态降到用户态？

---

## 五、第 2~3 步：进程与系统调用（kernel/ 核心）

### 5.1 数据结构与调度

- [ ] ⭐⭐ include/linux/sched.h (230行) — task_struct/tss_struct/i387_struct、INIT_TASK、switch_to宏(ljmp TSS硬件切换)、TSS/LDT选择子计算、set_base/set_limit段操作宏
- [ ] ⭐⭐ kernel/sched.c (254行) — schedule()主调度器(counter时间片+充能公式)、sleep_on/interruptible_sleep_on链式睡眠、wake_up、do_timer计账、sched_init(TSS/LDT登记+8253定时器+时钟门+系统调用门)
- [ ] ⭐ kernel/asm.s (157行) — CPU异常低级入口：no_error_code/error_code两套现场保存模板，全部异常最终调traps.c的do_xxx

### 5.2 系统调用机器

- [ ] ⭐⭐ include/unistd.h (247行) — _syscall0/1/2/3宏(int $0x80 + errno转换)、67个系统调用号__NR_xxx、POSIX函数声明
- [ ] ⭐⭐ kernel/system_call.s (219行) — _system_call入口(参数寄存器入栈+查sys_call_table)、ret_from_sys_call信号分发(改EIP重入信号处理函数!)、_timer_interrupt、_sys_fork、_hd_interrupt
- [ ] ⭐ include/linux/sys.h (80行) — 67个sys_xxx原型 + sys_call_table[]函数指针表（0号setup→66号setsid）

### 5.3 进程生命周期

- [ ] ⭐⭐ kernel/fork.c (136行) — find_empty_process(找PID+槽位)、copy_process(页级task_struct、TSS全字段填充、eax=0的魔法、copy_mem→copy_page_tables)、verify_area
- [ ] ⭐ kernel/exit.c (135行) — do_exit清理序列(页表→孤儿过继→关文件→放inode→ZOMBIE+SIGCHLD)、sys_waitpid收尸、do_kill四种pid语义、release
- [ ] ⭐ kernel/sys.c (216行) — 一堆ENOSYS占位调用+真实现：setuid/setgid权限模型、time/stime、times、brk、setpgid、setsid、uname、umask

**自检**：
1. schedule() 里 `counter = counter/2 + priority` 这行为什么对 IO 密集型进程有利？
2. copy_process 里哪一行决定了"子进程 fork 返回 0"？
3. sleep_on 里 `tmp=*p; *p=current;` 之后醒来 `tmp->state=0` —— 这是哪种数据结构思想？
4. ret_from_sys_call 为什么要检查 CS&3 和 OLDSS==0x17？

---

## 六、第 4~5 步：内存管理（mm/）

- [ ] ⭐⭐ mm/memory.c (264行) — mem_map物理页位图、get_free_page(反向扫描+内联汇编清页)、free_page、copy_page_tables(fork页表复制+COW清写位+内核区160页特例)、free_page_tables、put_page、un_wp_page/do_wp_page(COW缺页核心)、write_verify、do_no_page、calc_mem
- [ ] ⭐ mm/page.s (34行) — 缺页中断低级入口：保存现场→cr2取线性地址→按错误码bit0分发do_no_page/do_wp_page→iret

**自检**：
1. copy_page_tables 里 `this_page &= ~2` 意味着什么？谁会因此挨缺页中断？
2. get_free_page 为什么从位图末尾向前扫？
3. un_wp_page 里 `mem_map[nr]==1` 时直接恢复写位——这一步省了什么？

---

## 七、第 6 步：异常与信号

- [ ] ⭐ kernel/traps.c (199行) — die()打印完整现场(EIP/EFLAGS/栈/指令字节)、17个do_xxx异常处理函数、trap_init登记(idt 0~16+reserved)
- [ ] ○ include/signal.h (65行) — 32个信号编号、SIG_DFL/SIG_IGN、sigaction结构(声明)

**自检**：die() 打印的 esp[0..4] 分别是栈帧里的什么？（提示：对照 asm.s 压栈顺序）

---

## 八、第 7~8 步：文件系统地基（include/linux/fs.h + fs/ 核心）

- [ ] ⭐⭐ include/linux/fs.h (185行) — buffer_head(缓存头)、d_inode(盘上32字节)/m_inode(内存inode)、file、super_block、dir_entry、PIPE_宏组、设备主次号宏、全量函数原型
- [ ] ⭐ fs/super.c (102行) — do_mount(读1号块验magic 0x137F+装位图缓存)、mount_root(根inode计数+4的深意、统计空闲块/inode)
- [ ] ⭐⭐ fs/buffer.c (254行) — buffer_init(内存顶端反向切buffer+跳过640K~1M空洞)、hash(dev,block)查找、getblk(空闲链扫描+脏块回写+竞态重试goto)、bread/brelse、sys_sync/sync_dev、remove/insert_into_queues
- [ ] ⭐⭐ fs/inode.c (288行) — inode_table[32]、lock/wait_on_inode、iget(查找+复用+读盘)、iput(引用递减+管道特例+nlinks==0时truncate+free_inode)、_bmap(0~6直接/7一重间接/8二重间接三级映射+create模式)、read/write_inode、get_empty_inode(轮转分配)、get_pipe_inode
- [ ] · fs/bitmap.c (158行) — set/clear_bit/find_first_zero内联汇编、free_block/new_block(位图+清零+计数校验)、free_inode/new_inode(同一模式的inode版)

**自检**：
1. m_inode 和 d_inode 的区别是什么？各在什么时候存在？
2. _bmap 处理 block=600 时走哪条路径？block=100000 呢？
3. mount_root 里 `mi->i_count += 3` 为什么是 +3？

---

## 九、第 8~9 步：路径与文件操作（fs/ 主体）

- [ ] ⭐⭐ fs/namei.c (678行) — permission(root/uid/gid三级判定)、match(fs段比较)、find_entry(逐块扫描)、add_entry(找空槽或扩目录)、get_dir(逐级下行)、dir_namei/namei、open_namei(O_CREAT全流程)、sys_mkdir(自举.与..)、empty_dir、sys_rmdir、sys_unlink(nlinks--)、sys_link(跨设备EXDEV检查)
- [ ] ⭐ fs/open.c (188行) — sys_open(三层资源:fd/file_table/inode+tty特殊接管逻辑)、sys_creat、sys_close(引用计数+inode释放)、chdir/chroot/chmod/chown/access/utime
- [ ] ⭐ fs/read_write.c (97行) — sys_lseek(三种origin+管道ESPIPE)、sys_read/sys_write(按inode类型四路分发:管道/字符/块/普通)
- [ ] ⭐ fs/exec.c (306行) — MAX_ARG_PAGES参数页机制、copy_strings(从栈顶向下塞参数)、create_tables(构造argc/argv/envp栈布局)、change_ldt(重设代码数据段+参数页put_page)、do_execve(ZMAGIC校验+权限判定+不可返回点+换eip/esp魔法)
- [ ] · fs/pipe.c (92行) — read/write_pipe(一页环形缓冲+i_count==2在场检查+SIGPIPE)、sys_pipe(分配file表+fd对+put_fs_long返回)
- [ ] · fs/file_dev.c (84行) — file_read(file_read空洞填0)、file_write(O_APPEND语义+size推进)
- [ ] ○ fs/stat.c (51行) — cp_stat逐字段拷贝+put_fs_byte写出、sys_stat/sys_fstat
- [ ] ○ fs/fcntl.c (69行) — dupfd(找最低空fd)、sys_dup/dup2、sys_fcntl(F_GETFD/SETFD/GETFL/SETFL)
- [ ] ○ fs/ioctl.c (40行) — ioctl_table[8]分发表、sys_ioctl(设备类型校验)
- [ ] ○ fs/tty_ioctl.c (166行) — termios/termio双向拷贝、TCGETS/TCSETS家族、flush队列、一堆-EINVAL未实现项
- [ ] ○ fs/truncate.c (59行) — free_ind/free_dind(递归释放间接块)、truncate
- [ ] ○ fs/file_table.c (3行) — struct file file_table[NR_FILE] 全局定义
- [ ] · fs/block_dev.c (86行) — block_read/write(按1KB块搬运+get_fs_byte跨段)、rd_blk[]主设备号分发表、ll_rw_block
- [ ] · fs/char_dev.c (50行) — crw_table[]分发表、rw_ttyx/rw_tty(current->tty路由)、rw_char

**自检**：
1. open("/tmp/a", O_CREAT) 但 /tmp 目录不存在——在哪一步返回什么错误？
2. do_execve 哪一行之后"不可返回"？之后失败为什么只能 sys_exit(-1)？
3. sys_write 遇到 i_mode=0020000(字符设备) 会路由到哪里？

---

## 十、第 10 步：终端与驱动（kernel/ 驱动组）

- [ ] ⭐ include/linux/tty.h (74行) — tty_queue(1024环形缓冲+头尾指针+睡眠队列)、INC/DEC/EMPTY/FULL/CHARS/GETCH/PUTCH宏、tty_struct(termios+pgrp+三队列+write函数指针)、INIT_C_CC控制字符默认值
- [ ] ⭐ kernel/tty_io.c (306行) — tty_table[3]初始化(控制台+2串口)、copy_to_cooked(行规程全逻辑:CR/NL转换、退格回显、^S/^Q、^C信号、EOF计数)、tty_read(VTIME/VMIN超时模型+signal打断)、tty_write(OPOST输出处理+满则睡)、tty_intr
- [ ] ⭐ kernel/console.c (550行) — 显存0xB8000直写、gotoxy/set_origin(6845 CRTC寄存器)、scrup/scrdown(显存搬移滚屏)、CSI转义序列状态机(state 0~4:字符/ESC/参数收集/执行)、csi_J/K/L/M/P/@、con_init(从0x90510恢复光标+挂键盘中断)
- [ ] ⭐ kernel/keyboard.s (409行) — 扫描码分发key_table[256]、key_map/shift_map/alt_map三张ASCII表、ctrl/alt/shift/caps状态位维护、put_queue(往read_q塞字节)、cursor键(含Ctrl-Alt-Del重启!)、func功能键、kb_wait、reboot
- [ ] ⭐ kernel/hd.c (413行) — hd_info[]几何参数、request[32]请求队列、IN_ORDER电梯排序宏、rw_hd(块号→CHS三维换算divl)、sys_setup(读0扇区分区表0x55AA+挂根)、hd_out(7步命令端口序列)、read_intr/write_intr(中断续传256字/次)、do_request、add_request(排序链插入+sorting锁)、bad_rw_intr重试、hd_init
- [ ] · kernel/serial.c (53行) + rs_io.s (141行) — 8250初始化(DLAB除数2400bps)、rs1/rs2中断四路分发表(modem/write/read/line)、write_char(队列剩余<256唤醒写者)

**自检**：
1. 你敲一个 'A'，从键帽到屏幕一共经过哪些队列和函数？（draw it!）
2. copy_to_cooked 里 ^H(退格) 是怎么让屏幕上的字符消失的？
3. hd.c 为什么敢说"电梯算法不需要关中断"？（看 sorting 标志的使用方式）

---

## 十一、第 11 步：杂项与工具

- [ ] ⭐ include/asm/system.h (66行) — move_to_user_mode(伪造iret栈帧!)、_set_gate门描述符宏、set_tss/ldt_desc、sti/cli/nop
- [ ] · kernel/vsprintf.c (227行) — number()格式化核心(零填充/左右对齐/进制)、vsprintf主循环(%%c/s/o/p/x/d/u/n解析)
- [ ] · kernel/printk.c (33行) — 内核打印:fs段保存恢复+内联汇编直调tty_write(为什么不能直接调printf?)
- [ ] · kernel/mktime.c (52行) — month[]累计天数表(含2月29的魔法offset)、kernel_mktime
- [ ] · kernel/panic.c (11行) — printk+死循环，内核的"死刑宣告"
- [ ] ○ include/asm/io.h (24行) — outb/inb/outb_p/inb_p端口读写宏(带_p版有jmp延时)
- [ ] ○ include/asm/segment.h (38行) — get_fs_byte/word/long与put_fs系列(内核跨段访问用户内存)
- [ ] ○ include/asm/memory.h (14行) — memcpy宏(ds=es约定的说明)
- [ ] ○ include/linux/kernel.h (8行) — printk/panic/verify_area原型
- [ ] ○ include/linux/head.h (20行) — desc_struct/desc_table、idt/gdt外部声明、段索引常量
- [ ] ○ include/linux/mm.h (10行) — PAGE_SIZE=4096 + 三个页函数原型
- [ ] ○ include/linux/config.h (53行) — HIGH_MEMORY/BUFFER_END/ROOT_DEV/HD_TYPE编译期配置
- [ ] ○ include/linux/hdreg.h (99行) — AT硬盘控制器全部寄存器地址+状态位+命令码+partition结构
- [ ] ○ include/a.out.h (220行) — exec结构体、N_MAGIC/ZMAGIC=0413、N_TXTOFF等偏移宏、nlist符号表
- [ ] ○ include/termios.h (222行) — TCGETS等ioctl号、termios/termio结构、全部标志位常量
- [ ] ○ include/const.h (15行) — I_REGULAR/I_DIRECTORY等inode类型位
- [ ] ○ include/ctype.h (34行) — _U/_L/_D位掩码+isalpha等宏(查表实现)
- [ ] ○ include/errno.h (60行) — 39个错误码
- [ ] ○ include/fcntl.h (55行) — O_RDONLY/O_CREAT等打开标志、F_DUPFD等fcntl命令
- [ ] ○ include/stdarg.h (28行) — va_list/va_start/va_arg宏(基于栈布局)
- [ ] ○ include/stddef.h (19行) — size_t/NULL/offsetof
- [ ] ○ include/string.h (405行) — 全套字符串函数的内联汇编实现(lodsb/stosb/scasb串指令艺术，选读2~3个:strlen/strcmp/memcpy)
- [ ] ○ include/time.h (42行) — struct tm/time_t
- [ ] ○ include/utime.h (13行) — utimbuf
- [ ] ○ include/sys/types.h (46行) — pid_t/uid_t/size_t等基本类型
- [ ] ○ include/sys/stat.h (58行) — S_IFREG/S_ISDIR等宏
- [ ] ○ include/sys/times.h (15行) — struct tms
- [ ] ○ include/sys/utsname.h (16行) — struct utsname
- [ ] ○ include/sys/wait.h (23行) — WNOHANG/WUNTRACED
- [ ] ○ lib/ 11个文件 — write/open/close/dup/execve/setsid/wait/_exit(每个都是_syscall宏展开或int 0x80桩)、ctype.c(_ctype[256]分类表)、errno.c(1行)、string.c(把string.h内联函数实例化为库)
- [ ] · tools/build.c (68行) — 校验boot扇区Minix头+补0x55AA签名+拼接system→Image
- [ ] ○ 各目录 Makefile — .c.o/.s.o规则、tools/build调用(选读顶层1个即可)

**自检**：
1. printk 为什么要内联汇编调用 tty_write 而不是直接函数调用？
2. _syscall3 宏展开后，write(1,"hi",2) 的三个参数分别躺在哪些寄存器里？

---

## 十二、三条精读路径

### 🚀 速览路径（~3~4 天，面试/考试前）

第0步 → 第1步(boot+init) → 第2~3步的 ⭐⭐（sched.h + sched.c + system_call.s + fork.c）→ 第5步(mm/memory.c) → 第7~9步只读 4 个 ⭐⭐（fs.h + buffer.c + inode.c + namei.c 挑前 300 行）
> 目标：能讲清"启动六阶段、调度公式、int 0x80 全链路、COW 四步、MINIX 五层楼"五件事。

### 🛤 主线路径（~8~10 天，推荐）

第0~11步全部 ⭐⭐ + ⭐ 打勾，· 文件读文件头总纲，○ 文件扫一眼。
> 目标：能向别人把整个内核从上电到 shell 讲一遍不留断点。

### 🏔 全量路径（~14+ 天，深度吃透）

全部 83 文件打勾 + 每个自检题都能口答 + 完成 ZHIDAO 第 9 章阶段 2 的最小复刻（boot→main→双进程轮流打印）。
> 目标：具备读 0.11/0.12/0.95 演进版本的能力，可着手自己的内核实验。

---

## 十三、完成打卡

- [ ] 我已通读全部核心机制，能不看资料画出"电源 → shell 提示符"的完整链路图
- [ ] 我能向朋友解释清楚：int 0x80 发生了什么、fork 一次调用两次返回的原理、COW 的触发链
- [ ] 我已开始构思自己的 OS 项目（下一个里程碑：自己的 boot.s 能跳进自己写的 main）

*完成后建议：对照 ZHIDAO.md 第 9 章开始动手复刻，或进阶阅读 Linux 0.11（多了硬盘交换与更完整的 COW）。*
