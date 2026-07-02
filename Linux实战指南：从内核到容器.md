# Linux实战指南：从内核到容器

> 摒弃过时老命令，基于 Linux 6.6 LTS / Ubuntu 24.04 / RHEL 9 编写，面向现代云原生环境的 Linux 实操手册。
> 12章核心内容 + 附录A（Shell脚本生产实战），涵盖系统基础、文件系统、权限模型、进程管理、网络、systemd、性能调优、日志、安全审计、容器化。

---

## 序言：在云原生时代，重新认识 Linux

### 被"僵尸教程"困住的新一代

很多初学者在接触 Linux 时，都会经历一种强烈的"割裂感"：在书里或博客上学会了 `service iptables save`、`ifconfig`、`chkconfig`，满心欢喜地登录上公司最新的 Ubuntu 24.04 或 RHEL 9 服务器，却发现命令全报错，服务根本起不来。

时代变了。当 Kubernetes 和容器化已经成为基础设施的绝对标配，当 eBPF 和 cgroups v2 正在重塑操作系统的边界，当 Linux 6.x 内核开始引入 Rust 驱动与热替换调度器时，市面上却仍有大量教程停留在十年前的 SysVinit 时代。这种技术断层，让无数新人在"学了一堆废弃命令"和"面对生产故障束手无策"之间痛苦挣扎。这正是动笔写下本书的初衷。

### 摒弃过时，直击底层魔法

本书的 Slogan 是：**"摒弃过时老命令，面向现代云原生环境"**。你不会看到冗长的 `net-tools` 教学，而是直接拥抱 `iproute2` 和 `ss`；你不会去死记硬背复杂的 `iptables` 规则，而是掌握更现代的 `nftables`；我们不再讨论如何手动编写 init 脚本，而是深入剖析 `systemd` 的 Unit 架构与资源隔离（Slice）。

更重要的是，今天的 Linux 已经不再仅仅是一个"跑在物理机上的操作系统"——它是 Docker 的基石，是 Kubernetes 的底座，是所有云原生魔法的幕后引擎。本书将带你自底向上构建认知闭环：

* 当你理解了 **Namespace 和 Cgroups**，你就会明白容器究竟为何物，而不只是把它当成一个"轻量级虚拟机"；
* 当你掌握了 **VFS 和 OverlayFS**，你就能看透镜像分层的本质，秒懂容器启动为何如此迅速；
* 当你熟悉了 **Capabilities 和 Seccomp**，你就能在容器安全审计时，精准揪出那些企图越权的危险配置。

### 从生产中来，到生产中去

本书中的每一个知识点，都严格遵循 **"一句话定义 + 实战代码 + 避坑指南"** 的黄金结构。那些关于 `chattr +i` 防御勒索病毒的终极防线、关于 `sysctl` 调优 `tcp_tw_reuse` 在 NAT 网关上引发的血泪教训、关于 `xfs_growfs` 和 `resize2fs` 参数混淆导致的 LVM 扩容翻车——这些都不是从官方手册里抄来的干瘪条文，而是无数个凌晨三点排查故障时换来的"肌肉记忆"。

我试图把一线 SRE 和运维工程师的"排障直觉"倾囊相授。希望当你面对"磁盘有空间却报 No space left"、"进程卡死但 CPU 内存正常"等诡异现象时，能条件反射般地掏出 `df -i`、`strace` 和 `perf`，而不是对着屏幕发呆。

### 给读者的建议

不要只用眼睛看，请打开终端，敲下那些命令。去搞坏一个虚拟机，去弄丢一次 root 密码，去体验一次 `/etc/fstab` 写错导致无法开机的绝望，然后再用救援模式把它救回来。**在安全的环境里多踩坑，是为了在生产环境里不摔跤。**

技术在狂奔，但那些关于权限、进程、网络栈的底层逻辑，在过去的三十年里未曾改变，在未来的三十年依然坚如磐石。愿这本书，能成为你案头那本翻得最破、贴满标签的实战字典。

<br>
<p align="right"><b>作者</b></p>
<p align="right">2026 年 夏</p>

---

# 第一章：系统基础与架构

> **本章定位**：从零开始认识Linux的内部构造——内核如何与硬件对话、Shell如何成为人机桥梁、系统如何从按下电源到登录Shell。理解本章，你就掌握了与Linux对话的"语法"。

---

## 1.1 内核（Kernel）：Linux的心脏

**一句话定义**：内核是直接附着在硬件之上的程序，负责CPU调度、内存分配、设备驱动、文件系统挂载等核心功能，是用户程序与硬件之间唯一的中间人。

**为什么重要**：
- 性能问题的根因，70%在内核（调度、IO、内存）
- 容器、安全、网络都依赖内核特性
- Linux 6.x LTS是当前生产环境主流（6.6/6.10/6.12）

### 内核版本怎么看

```bash
$ uname -r
6.6.21-linuxkit        # 当前内核版本
$ uname -a
Linux host 6.6.21-linuxkit #1 SMP PREEMPT_DYNAMIC ... x86_64 GNU/Linux
$ cat /proc/version
Linux version 6.6.21-linuxkit (root@build) ...
```

版本号含义：`主版本.次版本.补丁号`
- **偶数次版本**：稳定版（如 6.6、6.10）
- **奇数次版本**：开发版（如 6.7、6.11）
- **后缀**：`-generic`（通用）、`-rt`（实时）、`-aws`（云定制）

### 内核的三大核心职责

```bash
# 1. 进程管理 - 查看当前进程树
$ ps -ef --forest
init─┬─systemd───sshd───sshd───bash───pstree
     └─containerd-shim───nginx───nginx

# 2. 内存管理 - 查看内存使用
$ free -h
              total        used        free      shared  buff/cache   available
Mem:           7.7Gi       1.2Gi       4.8Gi        12Mi       1.7Gi       6.3Gi
Swap:          2.0Gi          0B       2.0Gi

# 3. 设备管理 - 查看块设备
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    0   120G  0 disk
├─sda1        8:1    0   512M  0 part /boot/efi
└─sda2        8:2    0 119.5G  0 part /
```

### Linux 6.x的新特性（生产相关）

**Linux 6.6 LTS（2024年发布）**：
- **io_uring增强**：异步IO性能翻倍，数据库场景显著受益
- **eBPF调度器**：可热替换CPU调度器（如`sched_ext`）
- **BBR v3网络拥塞控制**：跨数据中心吞吐量提升15-20%
- **影子栈（Shadow Stack）**：用户态控制流完整性，缓解ROP攻击

**Linux 6.10/6.12 新增**：
- **sched_ext调度器正式可用**：用BPF写自己的调度策略
- **Rust内核代码占比超3%**：驱动可用Rust写，安全性提升
- **实时内核改进**：`PREEMPT_RT` 接近主线合并

### 实战：查看内核模块

```bash
$ lsmod | head -10                    # 列出已加载的模块
Module                  Size  Used by
nf_tables             311296  0
nft_ct                 16384  1
nft_limit             16384  7

$ modinfo ext4                        # 查看模块详情
filename:       /lib/modules/6.6.21-linuxkit/kernel/fs/ext4/ext4.ko
license:        GPL
description:    Fourth Extended Filesystem
depends:        mbcache,jbd2
```

---

## 1.2 发行版（Distribution）：内核之上的"全家桶"

**一句话定义**：发行版 = Linux内核 + GNU工具 + 软件包管理器 + 桌面环境 + 应用软件的完整套装。

### 主流发行版速览

| 发行版 | 包管理器 | 适合场景 | 2024-2026现状 |
|--------|----------|----------|---------------|
| **Ubuntu 24.04 LTS** | apt | 服务器/桌面新手 | 服务器市场占有率第一 |
| **RHEL 9.x** | dnf/rpm | 企业生产环境 | 商业订阅，CentOS Stream是免费替代 |
| **Debian 12** | apt | 稳定服务器 | Ubuntu的上游 |
| **Rocky Linux 9** | dnf/rpm | CentOS替代品 | CentOS停更后崛起 |
| **Fedora 41** | dnf | 技术尝鲜 | RHEL的上游，新特性首发地 |
| **Arch Linux** | pacman | 学习者/高级用户 | 滚动更新，最新软件 |
| **openEuler 22.03** | dnf | 国产化/信创 | 华为主导，国产服务器主流 |
| **Anolis OS 8** | dnf | 国产化替代 | 龙蜥社区，阿里云主导 |

### 如何选择发行版？

**新人推荐路径**：
```
学习曲线平缓 ──────────────────────── 学习曲线陡峭
Ubuntu Desktop → Ubuntu Server → Debian → RHEL/CentOS → Arch
                                  ↓
                                生产环境
```

**场景匹配**：
- **学习Linux**：Ubuntu Desktop（文档多、桌面友好）
- **学服务器**：Ubuntu Server（与生产一致）
- **找工作**：CentOS Stream / RHEL（国内企业主流）
- **性能调优**：Arch（最新内核，方便测试）

### 查看当前发行版

```bash
$ cat /etc/os-release
PRETTY_NAME="Ubuntu 24.04 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
VERSION="24.04 LTS (Noble Numbat)"

$ lsb_release -a            # 更详细信息（部分发行版支持）
$ hostnamectl               # 同时看主机名
 Static hostname: web-server-01
       Icon name: computer-vm
         Chassis: vm
      Machine ID: abc123...
         Boot ID: def456...
  Virtualization: kvm
Operating System: Ubuntu 24.04 LTS
          Kernel: Linux 6.6.21-linuxkit
    Architecture: x86-64
```

---

## 1.3 Shell：与内核对话的翻译官

**一句话定义**：Shell是接收用户命令、解析后调用内核执行的命令行解释器。Linux默认Shell是Bash（`/bin/bash`）。

### Shell的历史脉络

```
sh (Bourne Shell, 1977)         — 经典
 └─ bash (Bourne Again, 1989)   — GNU重写版，Linux默认
     ├─ zsh (1990)              — macOS Catalina后默认
     ├─ fish (2005)             — 智能提示，默认无需配置
     └─ nushell (Rust重写)      — 结构化数据处理
```

### 查看和切换Shell

```bash
$ echo $SHELL                      # 当前用户默认Shell
/bin/bash

$ cat /etc/shells                  # 系统支持的所有Shell
/bin/sh
/bin/bash
/usr/bin/zsh
/usr/bin/fish

$ chsh -s /usr/bin/zsh             # 修改默认Shell（需注销重新登录）
```

### Bash的核心特性

**1. 通配符**：
```bash
$ ls *.txt                         # 匹配所有.txt文件
$ ls file[0-9].log                 # 匹配file0.log到file9.log
$ ls {a,b,c}.sh                    # 展开为a.sh b.sh c.sh
```

**2. 管道与重定向**：
```bash
$ cat /var/log/syslog | grep "ERROR" > errors.log    # 提取错误到文件
$ command > output.log 2>&1                          # 标准输出+错误都重定向
$ command < input.txt                                # 从文件读取输入
```

**3. 历史命令**：
```bash
$ history                          # 查看历史
$ !!                               # 执行上一条
$ !grep                            # 执行最近一条grep开头的命令
$ Ctrl+R                           # 交互式搜索历史（推荐）
(reverse-i-search)`grep': cat /var/log/syslog | grep "ERROR"
```

**4. 变量与环境**：
```bash
$ name="Linux"
$ echo $name                       # 引用变量
$ export PATH=$PATH:/opt/myapp/bin # 设置环境变量
$ env                              # 查看所有环境变量
```

---

## 1.4 终端（Terminal）：Shell的"窗口"

**一句话定义**：终端是运行Shell的图形化窗口程序，它本身不执行命令，只负责显示和键盘输入。

### 常见终端模拟器

| 终端 | 特点 | 适用 |
|------|------|------|
| **GNOME Terminal** | Ubuntu默认 | 通用 |
| **iTerm2** | macOS下的神级终端 | macOS |
| **Windows Terminal** | Windows 11+推荐 | Windows |
| **Tabby** | 跨平台，支持SSH | 跨平台 |
| **tmux / screen** | 纯文本终端复用器 | 服务器（不依赖图形） |

### SSH登录 = 远程终端

```bash
$ ssh alice@192.168.1.100
alice@192.168.1.100's password:
Welcome to Ubuntu 24.04 LTS

alice@server:~$ 
```

**关键理解**：SSH登录后看到的是**虚拟终端**（伪终端PTY），不是真正的硬件终端。SSH客户端 = 终端，Shell在服务器上。

---

## 1.5 控制台（Console）：系统底层的"安全通道"

**一句话定义**：控制台是系统启动时直接输出的文本界面（tty1-tty6），不依赖图形环境。

**为什么关键**：
- 显卡驱动崩溃 → GUI进不去 → 只能靠控制台
- 修改了错误的`/etc/X11/*` → 图形系统挂掉 → 控制台救场
- 远程VNC断连 → 但物理机仍可控制台操作

### 切换控制台

```bash
# 物理机键盘操作
Ctrl+Alt+F1  ~  F6    # 切换到tty1-tty6（6个虚拟终端）
Ctrl+Alt+F7  /  F2    # 回到图形界面（具体看发行版）

# 远程登录
$ who                        # 查看谁登录在哪个tty
alice    tty1         2026-07-02 09:00
bob      pts/0        2026-07-02 09:30
```

> 注意：`tty`是物理/虚拟控制台，`pts/N`是SSH等伪终端（pseudo-terminal）。

---

## 1.6 根用户（Root）：超级管理员的代价

**一句话定义**：UID=0的用户，拥有对系统的完全控制权，能删除任何文件、修改任何配置。

### 风险警示

```bash
$ whoami
root
$ rm -rf /                         # ⚠️ 千万别执行！会删除整个系统
$ dd if=/dev/zero of=/dev/sda      # ⚠️ 千万别执行！会擦除硬盘
```

这些命令一旦执行，**无法撤销**。生产环境应避免直接用root。

### 为什么UID固定是0？

```bash
$ cat /etc/passwd | grep ':0:'
root:x:0:0:root:/root:/bin/bash
     ↑  ↑  
     UID GID

# 内核硬编码：UID 0 = 超级权限
# 任何用户只要UID=0，就是root
# 这就是为什么"创建一个UID=0的非root用户名"同样危险
```

### 实际场景

```bash
# 查看当前用户
$ id
uid=1000(alice) gid=1000(alice) groups=1000(alice),27(sudo)

# 临时提权
$ sudo cat /etc/shadow             # 普通用户+sudo能看shadow文件
$ sudo -i                           # 切换到root shell（需谨慎）

# 危险操作前确认身份
$ rm -rf /opt/old/
# 暂停：当前用户是谁？文件所有者是谁？确认了再回车
```

---

## 1.7 sudo：精细化授权的安全之道

**一句话定义**：sudo允许授权用户以其他用户（通常是root）身份执行特定命令，所有操作被记录到日志。

### sudo vs su对比

| 维度 | `su -` | `sudo cmd` |
|------|--------|------------|
| 认证方式 | 输入root密码 | 输入**自己**密码 |
| 权限粒度 | 完全切换 | 可限定特定命令 |
| 日志记录 | 无 | 完整记录到`/var/log/auth.log` |
| 风险 | 高（长会话） | 低（单次授权） |

### 配置`/etc/sudoers`

```bash
$ sudo visudo                       # 必须用visudo（语法检查）

# 1. 允许wheel组用户执行所有命令（RHEL/CentOS默认）
%wheel  ALL=(ALL)       ALL

# 2. 允许特定用户无需密码执行特定命令
alice   ALL=(ALL)       NOPASSWD: /usr/bin/systemctl restart nginx

# 3. 别名（管理大量规则时很有用）
User_Alias WEBADMINS = alice, bob, charlie
Cmnd_Alias NGIX_CMDS = /usr/bin/systemctl start nginx, \
                       /usr/bin/systemctl stop nginx
WEBADMINS ALL=(ALL) NOPASSWD: NGIX_CMDS
```

### sudo实战技巧

```bash
# 1. 查看自己能执行哪些sudo命令
$ sudo -l
Matching Defaults entries for alice:
    env_reset, mail_badpass, secure_path=...
User alice may run the following commands on this host:
    (ALL : ALL) NOPASSWD: ALL

# 2. 切换到其他用户执行命令
$ sudo -u postgres psql            # 以postgres身份执行psql

# 3. 编辑文件（默认编辑器）
$ sudoedit /etc/hosts              # 比sudo vim更安全（创建临时副本）

# 4. 5分钟内免密
$ sudo -v                          # 延长密码缓存

# 5. 危险操作二次确认
$ sudo rm -rf /opt/old-project/
# 不会问你"are you sure"，sudo是授权不是确认
# 真正的"确认"是养成敲命令前的三秒停顿
```

### sudo日志审计

```bash
$ grep sudo /var/log/auth.log        # Ubuntu/Debian
$ grep sudo /var/log/secure          # RHEL/CentOS
Jul  2 10:30:15 server sudo: alice : TTY=pts/0 ; PWD=/home/alice ; USER=root ; COMMAND=/usr/bin/systemctl restart nginx
```

**生产环境**：
- 给审计团队配置日志转发到SIEM
- 监控`sudo COMMAND=/bin/bash`（提权到shell往往意味着入侵）

---

## 1.8 系统调用（System Call）：用户态进入内核态的独木桥

**一句话定义**：系统调用是用户程序请求内核服务的唯一合法入口，每一次文件操作、网络IO、进程创建最终都通过它实现。

### 为什么需要系统调用？

```
用户态（User Space）        内核态（Kernel Space）
┌────────────────┐         ┌────────────────┐
│ 应用程序         │         │  内核           │
│ - nginx         │         │  - 进程调度      │
│ - mysql         │ ──syscall──>  - 内存管理     │
│ - python        │         │  - 设备驱动      │
└────────────────┘         └────────────────┘
    不能直接访问硬件                  ↑
                            唯一特权区域
```

**没有系统调用**，应用程序什么都做不了——不能读文件、不能发网络包、不能创建进程。

### 常用系统调用速查

| 系统调用 | 用途 | 典型命令/库函数 |
|----------|------|-----------------|
| `open`/`close` | 打开/关闭文件 | `fopen()` |
| `read`/`write` | 读写文件 | `fread()`/`fwrite()` |
| `fork` | 复制进程 | shell启动新进程 |
| `exec` | 加载新程序 | `system()` |
| `wait` | 等待子进程结束 | `waitpid()` |
| `mmap` | 内存映射文件 | 高级IO |
| `socket` | 创建网络连接 | `connect()` |
| `ioctl` | 设备控制 | `ifconfig` |

### 用strace看系统调用

**`strace`是排查问题的神器**，能跟踪进程调用的所有系统调用：

```bash
# 1. 查看ls命令调用了哪些系统调用
$ strace ls /tmp 2>&1 | head -10
execve("/bin/ls", ["ls", "/tmp"], ...) = 0       # 加载ls程序
brk(NULL) = 0x55f4...                            # 内存分配
access("/etc/ld.so.preload", R_OK) = -1          # 检查预加载库
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY) = 3  # 打开动态库缓存
# ...省略...
stat("/tmp", {st_mode=S_IFDIR|0775, ...}) = 0    # 读取/tmp属性

# 2. 跟踪一个进程的系统调用（用于排查"卡在哪了"）
$ strace -p 1234                                  # 跟踪PID 1234
# 输出实时刷新，看进程在等什么资源

# 3. 统计系统调用次数
$ strace -c ls /tmp
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ------------
100.00    0.000012         1           9           read
  0.00    0.000000         0          12           openat
  0.00    0.000000         0          11           close
  ...

# 4. 排查"为什么程序卡住"
$ strace -e trace=network -p <pid>
# 看进程在等哪个网络连接
```

### 性能分析：`perf`系统调用统计

```bash
# 统计系统调用热点
$ perf top -e syscalls:sys_enter_read
# 实时显示哪些read()调用最频繁
```

---

## 1.9 POSIX：Unix世界的"普通话"

**一句话定义**：POSIX是IEEE制定的操作系统接口标准，定义了系统调用、Shell命令、工具行为等规范，Linux和macOS都遵循它。

**为什么重要**：
- 你在Linux写的脚本，**理论上**在macOS/BSD也能跑
- C语言程序用POSIX API写的，跨Unix移植性最强
- 招聘JD常写"熟悉POSIX标准"，说明对方在乎兼容性

### POSIX包含什么

```
POSIX.1  - 核心服务（系统调用、进程、文件）
POSIX.2  - Shell和工具（ls, grep, awk标准行为）
POSIX.1b  - 实时扩展（定时器、信号量）
POSIX.1c  - 线程
POSIX.1d  - 进一步实时扩展
```

### POSIX路径最大值

```bash
# POSIX要求：
# - 文件名最长 255 字节
# - 完整路径最长 4096 字节
$ getconf NAME_MAX /                # 文件名最大长度
255
$ getconf PATH_MAX /                # 路径最大长度
4096
```

### 实战：写POSIX兼容的Shell脚本

```bash
#!/bin/sh
# 文件第一行用#!/bin/sh（POSIX标准Shell），而不是#!/bin/bash
# 避免bash独有特性如[[ ]]、数组、$RANDOM

# 错误示范（bash特有）
if [[ -f /tmp/file ]]; then
    arr=(a b c)
    echo ${arr[0]}
fi

# POSIX兼容写法
if [ -f /tmp/file ]; then
    set -- a b c
    echo $1
fi
```

> **生产环境的Shell脚本应该尽量POSIX兼容**，因为不同系统的`/bin/sh`可能链接到不同的Shell（dash、ash、ksh），不一定有bash。

---

## 1.10 GNU工具集：自由软件的基石

**一句话定义**：GNU（GNU's Not Unix）是1983年由Richard Stallman发起的项目，目标是创建一个完全自由的Unix兼容系统。Linux内核出现后与GNU工具集结合，形成"GNU/Linux"完整系统。

### GNU核心组件

| 工具 | 替代了什么 | 用途 |
|------|-----------|------|
| **GCC** | cc | C/C++编译器 |
| **glibc** | libc | C标准库 |
| **coreutils** | ls, cp, mv, cat | 基础命令 |
| **Bash** | sh | Shell |
| **GNU Make** | make | 构建工具 |
| **GDB** | dbx | 调试器 |
| **Binutils** | ld, as | 链接器、汇编器 |

### 验证：当前系统是否GNU/Linux？

```bash
$ gcc --version
gcc (Ubuntu 13.2.0) 13.2.0     # GNU Compiler Collection
$ ld --version
GNU ld (GNU Binutils) 2.42
$ /lib/x86_64-linux-gnu/libc.so.6  # glibc版本信息
GNU C Library (Ubuntu GLIBC 2.39-0ubuntu8) stable release version 2.39.
```

### 关键GNU工具的"非GNU"替代品

| GNU工具 | 非GNU替代 | 场景 |
|---------|-----------|------|
| `grep` | `ripgrep (rg)` | 大文件搜索更快 |
| `find` | `fd` | 语法更友好 |
| `sed/awk` | `jq` (JSON) | JSON处理 |
| `ls/cat` | `eza/bat` | 现代化显示 |
| `top/htop` | `btop` | 漂亮的系统监控 |

```bash
# 安装现代化替代品（Ubuntu）
$ sudo apt install ripgrep fd-find bat eza btop jq

# ripgrep 替代 grep
$ rg "TODO" --type py                    # 递归搜Python文件
$ rg -i "error" /var/log/                # 忽略大小写

# bat 替代 cat
$ bat /etc/nginx/nginx.conf              # 带语法高亮和行号

# btop 替代 top
$ btop                                   # 全屏漂亮的资源监控
```

---

## 1.11 引导流程：从按下电源到登录Shell

**一句话定义**：Linux启动 = 硬件自检 → 引导加载程序 → 内核 → init进程 → 用户登录。

### 完整引导流程图

```
电源开启
  ↓
[1] BIOS/UEFI              # 硬件自检 + 加载引导设备
  ↓
[2] GRUB (引导加载程序)     # 选择内核/启动项
  ↓
[3] Linux内核               # 初始化硬件、挂载根FS
  ↓
[4] initramfs               # 临时根FS，加载必要驱动
  ↓
[5] systemd (PID=1)         # 启动所有系统服务
  ↓
[6] getty/login             # 显示登录提示符
  ↓
[7] 用户输入密码 → 启动Shell
```

### 各阶段详解

**1. UEFI vs BIOS（启动源头）**
- **UEFI**：现代标准，支持GPT分区、2TB+硬盘、安全启动（Secure Boot）
- **BIOS**：传统老旧，MBR分区，最大2TB硬盘

```bash
$ ls /sys/firmware/efi/                 # 存在=UEFI启动
efi/                                    # 目录存在说明是UEFI
```

**2. GRUB引导加载程序**
- 默认配置文件：`/boot/grub/grub.cfg`
- 内核位置：`/boot/vmlinuz-*`
- initramfs：`/boot/initrd.img-*`

```bash
$ ls /boot/
vmlinuz-6.6.21-linuxkit
initrd.img-6.6.21-linuxkit
grub/
config-6.6.21-linuxkit
System.map-6.6.21-linuxkit
```

**3. 内核与initramfs**

initramfs = 早期内存中的临时根文件系统，**含加载真实根FS所需的所有驱动**。

```bash
# 查看initramfs内容
$ lsinitramfs /boot/initrd.img-6.6.21-linuxkit | head -20
bin
sbin
etc
lib
lib64
usr
...
# 它是cpio+gzip压缩，解压后看到的就是一个小Linux根目录

# 内核命令行参数
$ cat /proc/cmdline
BOOT_IMAGE=/boot/vmlinuz-6.6.21-linuxkit root=UUID=abc... ro quiet splash
        ↑                                            ↑
        内核文件                                      根分区
```

**4. systemd启动**
- PID=1，第一个用户空间进程
- 并行启动服务（不等A完才启B）
- 启动失败自动重启

```bash
# 查看启动耗时
$ systemd-analyze
Startup finished in 2.345s (kernel) + 5.678s (userspace) = 8.023s
graphical.target reached after 5.890s in userspace

# 找出启动最慢的服务
$ systemd-analyze blame
8.123s NetworkManager-wait-online.service
1.234s snapd.service
0.567s apparmor.service
...
```

### 故障排查：启动卡住

```bash
# 1. 临时编辑GRUB参数
# 启动时按'e'进入编辑模式
# 找到 linux/linux16 行，删除 quiet splash 加 systemd.unit=rescue.target
# Ctrl+X 启动到救援模式

# 2. 救援模式常见操作
$ mount -o remount,rw /        # 重新挂载根为可写
$ vi /etc/fstab                # 修改挂载配置
$ systemctl default            # 回到默认target
```

---

## 1.12 init系统：第一个用户态进程

**一句话定义**：init是PID=1的进程，是所有其他进程的祖先，负责启动、管理和关闭系统服务。

### init系统演化史

```
SysV init (1983-)              — 串行启动，慢
  └─ Upstart (Ubuntu 2006-)    — 事件驱动，过渡方案
       └─ systemd (2010-)      — 并行启动，现代标准
```

### systemd核心概念

| 概念 | 说明 |
|------|------|
| **Unit** | 配置单元（service, socket, target, mount等） |
| **Service** | 系统服务（nginx, mysql） |
| **Target** | 一组Unit的集合（类似runlevel） |
| **Socket** | 网络套接字，按需激活服务 |
| **Timer** | 定时器（替代cron的部分功能） |

### 常用systemctl命令

```bash
# 1. 服务管理
$ sudo systemctl start nginx            # 启动
$ sudo systemctl stop nginx             # 停止
$ sudo systemctl restart nginx          # 重启
$ sudo systemctl reload nginx           # 重新加载配置（不中断）
$ sudo systemctl status nginx           # 查看状态
$ sudo systemctl enable nginx           # 开机自启
$ sudo systemctl disable nginx          # 禁止开机自启
$ sudo systemctl is-active nginx        # 是否运行
$ sudo systemctl is-enabled nginx       # 是否开机自启

# 2. 查看所有服务状态
$ systemctl list-units --type=service --state=running
UNIT                       LOAD   ACTIVE SUB     DESCRIPTION
nginx.service              loaded active running A high performance web server
ssh.service                loaded active running OpenBSD Secure Shell server

# 3. 重启系统
$ sudo systemctl reboot
$ sudo systemctl poweroff
```

### system v init的兼容模式

```bash
# 有些老脚本还在用service命令
$ service nginx status                  # = systemctl status nginx
$ service nginx restart                 # = systemctl restart nginx

# 实际上service脚本在现代系统是systemctl的包装
$ cat /usr/sbin/service | head -5
#!/bin/sh
### BEGIN INIT INFO
...
systemctl_redirect ...                  # 最终调用systemctl
```

---

## 1.13 cgroups（控制组）：容器时代的资源基石

**一句话定义**：cgroups是Linux内核特性，用于限制、统计、隔离进程组的资源使用（CPU、内存、IO、网络）。

**为什么关键**：Docker/Kubernetes/Podman的底层就是cgroups+namespace。

### cgroups v1 vs v2

| 维度 | cgroups v1 | cgroups v2 |
|------|-----------|-----------|
| 引入时间 | 2007 | 2016（Linux 4.5+） |
| 控制器 | 每个资源独立子系统 | 统一层级树 |
| 容器支持 | 可用但有限 | **推荐**（Docker 20.10+/Podman默认） |

> ⚠️ **cgroups v2 兼容性注意**：现代发行版（RHEL 9, Ubuntu 22.04+）已全面默认使用 cgroups v2（统一层级树，解决 v1 嵌套冲突）。关键在于 cgroup 驱动配置——Docker 默认 cgroup 驱动为 `cgroupfs`，但 systemd 期望使用 `systemd` 驱动。驱动不匹配会导致资源限制不生效或容器启动失败。
> 
> **解决方案**：
> ```json
> // /etc/docker/daemon.json
> {
>   "exec-opts": ["native.cgroupdriver=systemd"]
> }
> ```
> 或直接使用 Podman（天生兼容 cgroups v2 + systemd）。
| 优势 | 兼容老系统 | 简单一致，资源控制更精准 |

```bash
# 查看系统用的是哪个版本
$ mount | grep cgroup
cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nsdelegate)
# ↑ cgroup2 = v2

# 旧系统看到的是多个cgroup挂载点 = v1
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,...)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,...)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,...)
```

### 实战：限制进程CPU使用

```bash
# 1. 创建cgroup（v2语法）
$ sudo cgcreate -g cpu:/test_group              # v1
$ sudo mkdir /sys/fs/cgroup/test_group         # v2 直接建目录

# 2. 设置CPU限制（最多使用1个CPU的50%）
$ echo 50000 > /sys/fs/cgroup/test_group/cpu.max
# 第二个数字是周期（默认100000=100ms），第一个是配额
# 50000/100000 = 50% 单核

# 3. 把进程加入cgroup
$ echo <PID> > /sys/fs/cgroup/test_group/cgroup.procs

# 4. 验证
$ cat /sys/fs/cgroup/test_group/cpu.max
50000 100000
```

### 实战：限制进程内存

```bash
# 限制最多使用512MB内存
$ echo 536870912 > /sys/fs/cgroup/test_group/memory.max
# 字节数：512 * 1024 * 1024 = 536870912

# 进程超过限制会被OOM Killer杀掉
$ cat /sys/fs/cgroup/test_group/memory.events
high 0
max 1                        # ← 触发过限制
oom 0
oom_kill 0
```

### 用systemd限制服务资源

systemd用`--slice`自动管理cgroups，更简单：

```bash
# 编辑服务文件
$ sudo systemctl edit nginx.service
[Service]
MemoryMax=512M
CPUQuota=50%

$ sudo systemctl daemon-reload
$ sudo systemctl restart nginx

# 查看资源限制
$ systemctl show nginx | grep -i memory
MemoryCurrent=134217728
MemoryMax=536870912
```

---

## 1.14 命名空间（Namespace）：容器的"视角隔离"

**一句话定义**：Namespace让进程拥有独立的系统资源视图（进程、网络、文件系统等），感觉像在独立的OS里运行。

### 八种Namespace类型

| Namespace | 隔离内容 | 应用场景 |
|-----------|----------|----------|
| **PID** | 进程ID | 容器内进程看不到主机进程 |
| **Network** | 网络栈（IP、端口、路由） | 容器有独立IP |
| **Mount** | 挂载点 | 容器有独立文件系统 |
| **UTS** | 主机名/域名 | 容器有独立hostname |
| **IPC** | 进程间通信 | 容器有独立信号量/共享内存 |
| **User** | 用户ID | 容器内root=主机普通用户 |
| **Cgroup** | cgroup根 | 容器看到的cgroup层级 |
| **Time** | 系统时间 | 容器可独立调整时间（Linux 5.6+） |

### 实战：手动用unshare创建Namespace

```bash
# 1. 创建新的PID namespace（子进程看不到主机进程）
$ sudo unshare --pid --fork --mount-proc /bin/bash
$$                                       # 容器内的PID
1
# 主机上还是能看到这个进程
# 但容器内看自己像PID 1

# 2. 创建网络namespace（独立网络栈）
$ sudo unshare --net /bin/bash
$ ip addr show lo
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
# 只有loopback，连主机网络都看不到

# 3. 创建User namespace（UID映射）
$ unshare --user --map-root-user /bin/bash
$$ id
uid=0(root) gid=0(root) groups=0(root)
# 在容器内是root，主机是普通用户
```

### 容器=Namespace+cgroups

```bash
# 简单"容器"原理
$ sudo unshare --pid --fork --mount-proc \
              --net --uts --ipc \
              --cgroup \
              /bin/bash
# + 用cgroups限制资源
# = 一个"原始容器"
```

Docker/Podman的`docker run`本质上就是自动化的`unshare`+cgroups+chroot。

---

## 1.15 虚拟文件系统（VFS）：一切皆文件的实现

**一句话定义**：VFS是Linux内核的抽象层，定义统一接口，让用户用同一套系统调用（open/read/write）操作所有类型的"文件"——普通文件、目录、设备、管道、网络套接字。

### "一切皆文件"举例

```bash
# 1. 普通文件
$ cat /etc/hostname

# 2. 设备文件
$ cat /dev/zero                          # 永远读出0
$ echo "hello" > /dev/null               # 写入的数据消失
$ cat /dev/random | head -c 16 | xxd    # 读16字节随机数

# 3. 进程信息
$ cat /proc/1234/cmdline                  # 读取进程命令行

# 4. 内核参数
$ cat /proc/sys/net/ipv4/ip_forward      # 0=关闭 1=开启

# 5. 网络套接字
$ cat /proc/net/tcp                      # 查看所有TCP连接

# 6. 设备信息
$ cat /sys/class/net/eth0/address        # 网卡MAC地址
```

### VFS的四种主要实现

| 文件系统类型 | 挂载点示例 | 特点 |
|--------------|-----------|------|
| **磁盘FS** (ext4, xfs) | `/`, `/home` | 持久化存储 |
| **虚拟FS** (proc, sysfs) | `/proc`, `/sys` | 内核导出信息，内存中动态生成 |
| **网络FS** (nfs, cifs) | `/mnt/nfs` | 远程服务器挂载 |
| **伪FS** (tmpfs, devpts) | `/dev/shm`, `/dev/pts` | 内存/特殊用途 |

### 实战：从/proc读取系统信息

```bash
# /proc是脚本获取系统信息的宝库
$ #!/bin/bash
# 获取系统基本信息
echo "CPU核心数: $(nproc)"
echo "CPU型号: $(grep -m1 'model name' /proc/cpuinfo | cut -d: -f2)"
echo "总内存: $(awk '/MemTotal/ {printf "%.1f GB", $2/1024/1024}' /proc/meminfo)"
echo "运行时间: $(awk '{d=$1/86400; h=($1%86400)/3600; printf "%d天%d小时", d, h}' /proc/uptime)"
echo "内核版本: $(uname -r)"
```

---

## 1.16 sysctl：内核参数的运行时调优

**一句话定义**：sysctl是在运行时查看/修改内核参数的命令，通过`/proc/sys/`虚拟文件系统实现。

### 三大类常用参数

**1. 文件系统参数**
```bash
# 最大文件句柄数（高并发服务必调）
$ sysctl fs.file-max
fs.file-max = 9223372036854775807

# 临时修改
$ sudo sysctl -w fs.file-max=2097152

# 永久生效：写入/etc/sysctl.d/99-custom.conf
$ echo "fs.file-max = 2097152" | sudo tee /etc/sysctl.d/99-custom.conf
$ sudo sysctl -p /etc/sysctl.d/99-custom.conf
```

**2. 网络参数（高并发必调）**
```bash
# TCP连接队列上限（防拒绝服务）
$ sudo sysctl -w net.core.somaxconn=4096

# TIME_WAIT连接复用（短连接服务）
$ sudo sysctl -w net.ipv4.tcp_tw_reuse=1
# ⚠️ 适用场景：Web服务器、反向代理、API网关等出站连接较多的节点
# ⚠️ 慎用场景：NAT网关、负载均衡器、特殊中间件——可能引发连接混乱

# 允许路由转发（做路由器或容器主机必开）
$ sudo sysctl -w net.ipv4.ip_forward=1

# TCP缓冲区最大（高速网络）
$ sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 6291456"
$ sudo sysctl -w net.ipv4.tcp_wmem="4096 65536 6291456"

# SYN洪水防御
$ sudo sysctl -w net.ipv4.tcp_syncookies=1
```

**3. 内核参数**
```bash
# 最大PID数
$ sysctl kernel.pid_max
kernel.pid_max = 4194304

# 内核panic后自动重启（生产环境推荐）
$ sudo sysctl -w kernel.panic=10           # panic后10秒自动重启

# 启用BBR拥塞控制（Linux 4.9+）
$ sudo sysctl -w net.core.default_qdisc=fq
$ sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
```

### sysctl配置文件规范

```
/etc/sysctl.conf             — 传统单一文件
/etc/sysctl.d/               — 目录，按文件名排序加载
  └── 99-custom.conf         — 数字越大越后加载
```

```bash
# 应用所有配置
$ sudo sysctl --system
# 应用特定文件
$ sudo sysctl -p /etc/sysctl.d/99-custom.conf
```

### 实战：Web服务器调优模板

```bash
# /etc/sysctl.d/99-webserver.conf

# 文件描述符
fs.file-max = 2097152
fs.nr_open = 2097152

# 网络核心
net.core.somaxconn = 4096
net.core.netdev_max_backlog = 65536
net.ipv4.tcp_max_syn_backlog = 65536

# TCP优化
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 60
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_rmem = 4096 87380 6291456
net.ipv4.tcp_wmem = 4096 65536 6291456

# BBR
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```

---

## 1.17 initramfs：内核启动的"小跟班"

**一句话定义**：initramfs是内核启动早期加载到内存的临时根文件系统，含有挂载真实根FS所需的驱动和工具。

### 为什么需要initramfs？

**问题**：内核要挂载根FS（比如`/`在LVM上），但LVM驱动在内核外（作为模块加载）。内核怎么加载LVM驱动？→ 鸡生蛋问题。

**解决**：initramfs预先把驱动打包到内存中的小根FS，内核用它加载驱动后再切换到真实根。

### initramfs结构

```bash
# 看看initramfs里面有什么
$ lsinitramfs /boot/initrd.img-6.6.21-linuxkit 2>/dev/null | head -20
bin
sbin
etc
lib
lib64
usr
scripts
usr/lib/modules/6.6.21-linuxkit/kernel/drivers/...  # 内核模块
usr/sbin/cryptsetup                                    # LUKS加密
usr/sbin/lvm                                           # LVM
init                                                    # 入口脚本

# init脚本就是它的工作流程：
# 1. 探测硬件（加载对应驱动）
# 2. 处理LVM/RAID/加密
# 3. pivot_root切换到真实根
```

### 实战：重新生成initramfs

```bash
# Ubuntu/Debian
$ sudo update-initramfs -u                              # 重建当前内核
$ sudo update-initramfs -c -k 6.6.21-linuxkit           # 创建指定内核
$ sudo update-initramfs -d -k 5.15.0-100               # 删除指定内核

# RHEL/CentOS
$ sudo dracut -f                                        # 强制重建当前内核
$ sudo dracut --force /boot/initramfs-$(uname -r).img $(uname -r)

# 重建时机
# 1. 修改了/etc/fstab（磁盘布局）
# 2. 升级了内核
# 3. 改了LVM/RAID/加密配置
# 4. 安装了新驱动模块
```

### 故障：initramfs损坏导致无法启动

```bash
# 1. 启动时进入dracut emergency shell（黑屏+命令提示符）
# 2. 手动检查
dracut> ls /dev/sd*                       # 确认磁盘存在
dracut> blkid                             # 看分区UUID
dracut> mount /dev/sda2 /sysroot          # 尝试挂载

# 3. 如果挂载成功，重建initramfs
dracut> chroot /sysroot
# 此时进入了原系统
# chroot下：
# dracut -f
# exit
# reboot
```

---

## 1.18 运行时目录（/run）：现代Linux的运行时信息中枢

**一句话定义**：`/run`是tmpfs（内存文件系统）挂载的目录，存放系统启动以来的运行时数据，重启后自动清空。

### /run vs /var/run

```bash
$ ls -la /var/run
lrwxrwxrwx 1 root root 4 /var/run -> /run
# /var/run是符号链接，指向/run
# 历史包袱：旧版本叫/var/run，现在是/run
```

### /run里有什么

```bash
$ ls /run/
nginx.pid                            # nginx主进程PID文件
sshd.pid                             # sshd PID
user/1000/                            # 用户运行时数据
systemd/                             # systemd状态
  ├── system/                        # 服务运行时
  └── users/                         # 用户级服务
log/                                 # 部分服务的日志
lock/                                # 锁文件
utmp                                 # 当前登录用户（who命令读它）
```

### 为什么用tmpfs？

1. **速度快**：内存访问比磁盘快1000倍
2. **自动清理**：重启数据消失，不需要定期清理
3. **早期可用**：系统启动早期`/var`还没挂载，`/run`已就绪

### 实战：PID文件位置

```bash
# 传统服务会在/var/run/写PID文件
# 现在统一在/run/

# 自定义服务写PID
$ cat > /etc/systemd/system/myapp.service <<EOF
[Service]
ExecStart=/usr/local/bin/myapp
PIDFile=/run/myapp.pid          # 显式指定
EOF

# 启动后能看到
$ ls -la /run/myapp.pid
-rw-r--r-- 1 root root 4 Jul  2 10:30 /run/myapp.pid
$ cat /run/myapp.pid
12345
```

### /run的容量问题

`/run`默认只占内存的50%：
```bash
$ df -h /run
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           3.9G  4.2M  3.9G   1% /run
# 3.9G一般够用
# 但如果/tmp也用tmpfs，且大量写小文件，可能不够
```

调整方法（一般不需要）：
```bash
# /etc/fstab 添加
tmpfs /run tmpfs defaults,size=1G 0 0
```

---

## 本章小结

### 核心概念图谱

```
┌─────────── 用户空间 ───────────┐
│  Terminal → Shell → 工具(GNU)   │
│         ↘ sudo                  │
│  cgroups + namespace = 容器     │
└──────────────┬──────────────────┘
               │ 系统调用 (syscall)
┌──────────────▼──────────────────┐
│  内核 (Kernel 6.x LTS)          │
│  - 进程/内存/IO/网络管理         │
│  - VFS (一切皆文件)             │
│  - sysctl (运行时调优)          │
└──────────────┬──────────────────┘
               │ 设备驱动
┌──────────────▼──────────────────┐
│  硬件 (CPU/内存/磁盘/网卡)       │
└─────────────────────────────────┘
```

### 从电源到Shell的时间线

```
[电源] → UEFI/BIOS → GRUB → 内核 → initramfs → systemd → login
                                                                  ↓
                                                              [Shell]
```

### 给新人的学习路径

1. **第一周**：熟练使用Bash，掌握`ls/cd/cp/mv/grep/find/ps/kill`
2. **第二周**：理解用户权限，掌握`chmod/chown/sudo`
3. **第三周**：理解进程与服务，掌握`systemctl`
4. **第四周**：理解文件系统，掌握`mount/df/du`
5. **持续**：阅读`man`页（`man systemctl`、`man sysctl`）

### 推荐命令

```bash
$ man bash                    # Shell最完整的文档
$ man sudoers                 # sudo权限配置
$ man systemd.service         # systemd unit文件格式
$ man sysctl                  # 内核参数
$ man proc                    # /proc详解
```

### 下一章预告

**第二章：文件系统与目录结构** — FHS标准、inode、硬链接/软链接、ext4/XFS/Btrfs对比、LVM/RAID、Swap调优、/proc /sys /dev深度使用。

---

> **本章字数**：约 18,500 字
> **涉及命令**：uname, ps, free, lsblk, strace, sudo, systemctl, sysctl, dracut, update-initramfs
> **内核版本基准**：Linux 6.6 LTS (2024年) + 6.10/6.12 进展

# 第二章：文件系统与目录结构

> **本章定位**：Linux的"一切皆文件"哲学不是口号，而是从根目录到设备节点的统一设计。理解文件系统层次、inode机制、链接原理、挂载流程，你就掌握了Linux存储系统的完整图谱。

---

## 2.1 一切皆文件（Everything is a File）

**一句话定义**：Unix/Linux的核心哲学——将所有系统资源抽象为文件，统一通过open/read/write/ioctl接口访问。

### 五大资源类型，都按文件操作

```bash
# 1. 普通文件
$ echo "hello" > /tmp/test.txt
$ cat /tmp/test.txt
hello

# 2. 设备文件
$ cat /dev/null                    # 永远读出空
$ echo "junk" > /dev/null          # 写入即丢弃
$ head -c 32 /dev/urandom | xxd   # 读32字节随机数

# 3. 进程信息（伪文件）
$ cat /proc/1/cmdline              # PID 1的命令行
/usr/lib/systemd/systemd
$ cat /proc/1/status | head -5     # 进程状态
Name:   systemd
Umask:  0000
State:  S (sleeping)

# 4. 管道（pipe）
$ ls /tmp | grep ".log" | wc -l
$ mkfifo /tmp/myfifo
$ echo "msg" > /tmp/myfifo          # 阻塞直到有人读

# 5. 网络套接字
$ cat /proc/net/tcp | head -3       # 所有TCP连接
```

### 设备文件的两大类

```bash
$ ls -la /dev/sda /dev/null /dev/tty
brw-rw---- 1 root disk      8,  0 Jul  2 10:00 /dev/sda    # 块设备 (b)
crw-rw-rw- 1 root root     1,  3 Jul  2 10:00 /dev/null   # 字符设备 (c)
crw-rw-rw- 1 root tty      5,  0 Jul  2 10:00 /dev/tty     # 字符设备 (c)

# 数字含义：主设备号(major), 次设备号(minor)
# sda:  major=8 (SCSI/SATA磁盘), minor=0 (第1块)
# null: major=1 (mem), minor=3
# tty:  major=5, minor=0
```

**字符设备 vs 块设备**：
- **块设备**（b）：以固定大小块为单位读写，可随机访问 → 磁盘
- **字符设备**（c）：以流式字节读写 → 终端、串口

### 为什么设计成"文件"？

1. **统一接口**：同一套`open/read/write`处理所有IO
2. **权限统一**：用rwx权限模型控制所有资源
3. **编程简化**：写文件 = 写网络 = 写设备，都是fd
4. **组合管道**：`cat file | nc host`就是经典的"文件经管道发送"

### 实战：/dev下的奇怪设备

```bash
# /dev/zero - 无限零字节流
$ dd if=/dev/zero of=bigfile bs=1M count=1024    # 创建1G空文件

# /dev/urandom - 伪随机数
$ tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 16   # 16位随机密码

# /dev/shm - 共享内存tmpfs
$ df -h /dev/shm
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           3.9G     0  3.9G   0% /dev/shm
# 进程间高速共享数据用，速度比/tmp快10倍

# /dev/stdin /dev/stdout /dev/stderr - 三个标准流的设备文件
$ echo "test" > /dev/stderr                       # 输出到错误流
$ cat < /dev/stdin                                # 从输入读
```

---

## 2.2 根目录（/）：文件系统的起点

**一句话定义**：根目录`/`是Linux文件系统树状结构的顶层起点，所有文件、目录、设备都在它之下。

```bash
$ ls -la /
total 64
drwxr-xr-x   1 root root 4096 Jul  2 10:00 .
drwxr-xr-x   1 root root 4096 Jul  2 10:00 ..
drwxr-xr-x   2 root root 4096 Jun 15 09:00 bin         # 基本命令
drwxr-xr-x   3 root root 4096 Jul  2 09:00 boot        # 引导文件
drwxr-xr-x  15 root root 3260 Jul  2 09:00 dev         # 设备文件
drwxr-xr-x 154 root root 12288 Jul  2 10:00 etc        # 配置文件
drwxr-xr-x   3 root root 4096 Jul  2 09:00 home        # 用户家目录
drwxr-xr-x  19 root root 4096 Jul  2 09:00 lib         # 32位库（x86_64下是指向lib64的链接）
lrwxrwxrwx   1 root root    4 Jun 15 09:00 lib64 -> /lib
drwxr-xr-x   2 root root 4096 Jun 15 09:00 media       # 可移动设备
drwxr-xr-x   2 root root 4096 Jun 15 09:00 mnt         # 临时挂载点
drwxr-xr-x   2 root root 4096 Jun 15 09:00 opt         # 可选软件
dr-xr-xr-x 312 root root     0 Jul  2 09:00 proc        # 进程信息
drwx------   2 root root 4096 Jul  2 09:00 root        # root家目录
drwxr-xr-x  33 root root 1180 Jul  2 10:00 run         # 运行时数据
drwxr-xr-x   2 root root 4096 Jun 15 09:00 sbin        # 系统管理命令
dr-xr-xr-x  13 root root     0 Jul  2 09:00 sys        # 设备/驱动信息
drwxrwxrwt   8 root root  240 Jul  2 10:00 tmp         # 临时文件
drwxr-xr-x  14 root root 4096 Jun 15 09:00 usr         # 用户程序
drwxr-xr-x  11 root root 4096 Jul  2 09:00 var         # 可变数据
```

**根目录的inode编号固定为2**（ext4/xfs通用），0号inode是保留的，1号inode是ext4的lost+found。

### 根目录的硬性约束

```bash
# 1. 根文件系统必须最先挂载
#    内核命令行指定 root=/dev/sda2 或 root=UUID=xxx

# 2. 根目录在系统启动完成前不能umount
#    强行umount根目录 = 系统崩溃

# 3. 单根 vs 多根（Linux是单根）
#    Windows是C:\ D:\多根
#    Linux只有/，其他盘都挂载到子树
```

---

## 2.3 FHS：文件系统层次结构标准

**一句话定义**：FHS（Filesystem Hierarchy Standard）规定Linux目录的布局和用途，让不同发行版保持兼容。

### 必知目录速查表

| 目录 | 用途 | 是否可写 | 典型内容 |
|------|------|----------|----------|
| `/bin` | 基本命令 | 否 | ls, cp, mv, cat |
| `/sbin` | 系统管理命令 | 否 | fdisk, ifconfig, iptables |
| `/etc` | 配置文件 | 是 | passwd, hosts, nginx/ |
| `/boot` | 引导文件 | 否 | vmlinuz, grub/, initrd |
| `/home` | 用户家目录 | 是 | /home/alice |
| `/root` | root家目录 | 是 | /root/.bashrc |
| `/var` | 可变数据 | 是 | /var/log/, /var/lib/mysql |
| `/tmp` | 临时文件 | 是 | 应用临时文件 |
| `/usr` | 用户程序 | 否 | /usr/bin, /usr/lib |
| `/opt` | 第三方软件 | 是 | /opt/google/chrome |
| `/dev` | 设备文件 | 内核管理 | /dev/sda, /dev/null |
| `/proc` | 进程/内核信息 | 内核生成 | /proc/cpuinfo |
| `/sys` | 设备/驱动信息 | 内核生成 | /sys/class/net |
| `/run` | 运行时数据 | tmpfs | /run/nginx.pid |
| `/mnt` | 临时挂载点 | 是 | 手动挂载的临时盘 |
| `/media` | 自动挂载的可移动设备 | 是 | /media/cdrom |

### /usr vs /var vs /opt的本质区别

```
/usr  = 静态只读（系统级程序，几乎不变）
/var  = 动态变化（日志、数据库、缓存）
/opt  = 第三方独立软件（每个子目录一个应用）
```

```bash
# /usr 结构
/usr/bin           # 用户命令
/usr/sbin          # 系统管理命令
/usr/lib           # 库文件
/usr/local         # 本地编译安装的软件（不通过包管理器）
  ├── bin
  ├── lib
  └── share
/usr/share         # 共享数据（man页、文档）

# /var 结构
/var/log           # 日志（syslog, nginx/, mysql/）
/var/lib           # 状态数据（数据库文件、Docker镜像）
/var/cache         # 缓存（apt缓存、pip缓存）
/var/spool         # 队列（邮件、打印）
/var/tmp           # 临时文件（比/tmp保留更久）
/var/lock          # 锁文件

# /opt 结构（每个应用一个子目录）
/opt/google/chrome/
/opt/idea/
```

### 现代变化（systemd时代）

```bash
# 1. /bin和/sbin变成了/usr/bin和/usr/sbin的符号链接
$ ls -la /bin /sbin
lrwxrwxrwx /bin -> usr/bin
lrwxrwxrwx /sbin -> usr/sbin
# 这是systemd推的"统一usr"（usrmerge）改革

# 2. /var/run -> /run（已链接）
$ ls -la /var/run
lrwxrwxrwx /var/run -> /run

# 3. /etc已成为最大的"静态"配置目录
#    实际上很多软件往/etc写运行时状态，违反了FHS精神
#    正确的应该是/etc放配置，运行时状态放/run或/var/lib
```

### 实战：找文件最快的方式

```bash
# 1. 找命令的二进制文件
$ which python3
/usr/bin/python3
$ whereis nginx
nginx: /usr/sbin/nginx /etc/nginx /usr/share/man/man8/nginx.8.gz

# 2. 找配置文件
$ locate nginx.conf             # 需安装mlocate
# 或
$ find /etc -name "nginx.conf" 2>/dev/null

# 3. 找手册页
$ man -w ls
/usr/share/man/man1/ls.1.gz

# 4. 按功能反查命令
$ whatis copy
$ man -k network                 # 模糊搜索所有网络相关man
```

---

## 2.4 inode：文件系统的"身份证"

**一句话定义**：inode（index node）是文件系统中存储文件元数据的数据结构，每个文件/目录都有唯一inode。

### inode存储的信息

```bash
# 查看inode信息
$ stat /etc/hostname
  File: /etc/hostname
  Size: 8               Blocks: 8          IO Block: 4096   regular file
Device: 802h/2050d      Inode: 131142      Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-07-02 09:00:00.000000000 +0800
Modify: 2026-06-15 14:30:00.000000000 +0800
Change: 2026-06-15 14:30:00.000000000 +0800
 Birth: 2026-06-15 14:30:00.000000000 +0800

# 关键字段：
# Inode: 131142        唯一编号
# Links: 1             硬链接数
# Access/Modify/Change: atime/mtime/ctime
```

**inode不包含**：
- 文件名（文件名存在目录项里）
- 文件内容（内容存在数据块里，inode只存数据块的指针）

### 文件名 vs inode

```bash
# 文件名是给人看的，inode是给内核看的
# 同一个inode可以对应多个文件名（硬链接）

# 1. 查看文件的inode号
$ ls -i /etc/hostname
131142 /etc/hostname

# 2. 按inode找文件（删除文件名后也能找）
$ find / -inum 131142 2>/dev/null

# 3. 为什么有时删了文件磁盘空间没释放？
#    因为有进程在打开它，inode还在
$ lsof | grep deleted        # 找到已删除但被进程占用的文件
nginx  1234 root   5w  REG  8,2  500000  131142 /var/log/nginx/access.log (deleted)
# inode 131142还在被nginx持有，所以空间不释放
# 解决：让nginx重新打开日志文件（logrotate就是这样工作的）
```

### inode数量限制

```bash
# 1. 查看文件系统inode使用情况
$ df -i /
Filesystem      Inodes  IUsed   IFree IUse% Mounted on
/dev/sda2      7864320 234567 7629753    3% /

# 2. 大量小文件场景（如邮件服务器）会耗尽inode
#    即使磁盘还有空间，也无法创建新文件
#    "No space left on device" 但 df -h 还有空间 → 检查 inode
$ df -h /
$ df -i /                     # 这才是真相

# 3. 解决：重建文件系统用更大的inode密度
#    或使用xfs（xfs的inode按需分配，不预先固定）
```

### 实战：inode耗尽排查

```bash
# 场景：磁盘还有空间，但无法创建文件
$ touch testfile
touch: cannot touch 'testfile': No space left on device

# 1. 看磁盘空间
$ df -h /
/dev/sda2       50G   30G   18G  63% /         # 还有18G！

# 2. 看inode使用
$ df -i /
/dev/sda2      100000 99999     1  99% /        # 满了！

# 3. 找哪个目录小文件最多
$ for dir in /var /tmp /home; do
    echo "$dir: $(find $dir -xdev -type f 2>/dev/null | wc -l) files"
done
/var: 50000 files
/tmp: 45000 files               # ← 大部分在/tmp

# 4. 清理
$ sudo rm -rf /tmp/*.tmp        # 删除临时文件
```

---

## 2.5 硬链接 vs 软链接：链接的本质

**一句话定义**：
- **硬链接**：多个文件名指向**同一个inode**（共享数据，删除一个不影响）
- **软链接（符号链接）**：一个文件存储**另一个文件的路径**（类似Windows快捷方式）

### 硬链接

```bash
# 1. 创建硬链接
$ echo "Hello" > original.txt
$ ln original.txt hardlink.txt

# 2. 验证是同一个inode
$ ls -i original.txt hardlink.txt
131200 original.txt
131200 hardlink.txt          # ← inode相同

# 3. 统计信息
$ stat original.txt
  ...
  Inode: 131200   Links: 2    # 链接数=2

# 4. 删除原文件
$ rm original.txt
$ cat hardlink.txt
Hello                         # 数据还在
$ ls -i hardlink.txt
131200 hardlink.txt          # inode没变，链接数变1
```

**硬链接特点**：
- ✅ 节省磁盘（共享数据块）
- ✅ 修改同步（任一处修改，所有链接都看到）
- ❌ 不能跨文件系统（inode在不同FS不同）
- ❌ 不能链接目录（防止循环引用）

### 软链接

```bash
# 1. 创建软链接
$ echo "Hello" > original.txt
$ ln -s original.txt symlink.txt

# 2. 软链接是独立文件
$ ls -la symlink.txt
lrwxrwxrwx 1 root root 12 Jul  2 10:00 symlink.txt -> original.txt
                                            ↑ 12字节，存的是路径

# 3. inode不同
$ ls -i original.txt symlink.txt
131200 original.txt
131201 symlink.txt            # 不同的inode

# 4. 删除原文件
$ rm original.txt
$ cat symlink.txt
cat: symlink.txt: No such file or directory    # 软链接断开了

# 5. 软链接可指向目录
$ ln -s /var/log logs
$ ls logs                     # 等同于 ls /var/log
```

**软链接特点**：
- ✅ 可跨文件系统
- ✅ 可链接目录
- ✅ 可链接不存在的文件
- ❌ 删除原文件会"悬空"（dangling）

### 对比表

| 维度 | 硬链接 | 软链接 |
|------|--------|--------|
| inode | 相同 | 不同 |
| 跨FS | ❌ | ✅ |
| 链接目录 | ❌ | ✅ |
| 链接不存在 | ❌ | ✅ |
| 原文件删除 | 数据还在 | 软链接失效 |
| 占用空间 | 几乎0（仅一个目录项） | 路径字符串长度 |
| 实际场景 | 备份/防误删 | 版本切换/快捷方式 |

### 实战：find按链接类型

```bash
# 查找所有软链接
$ find /usr -type l | head -10
/usr/bin/python3 -> python3.12
/usr/bin/vi -> vim
/usr/bin/awk -> gawk

# 找悬空的软链接（broken）
$ find /usr -xtype l | head -5
# -xtype l 评估后类型（如果是悬空链接）
# 修复：删除或重新建立

# 实战：Python版本管理
$ ls -la /usr/bin/python*
lrwxrwxrwx /usr/bin/python3 -> python3.12
lrwxrwxrwx /usr/bin/python3.12 -> python3.12
-rwxr-xr-x /usr/bin/python3.12

# 切换默认版本
$ sudo ln -sf python3.11 /usr/bin/python3
$ python3 --version
Python 3.11.x
```

---

## 2.6 挂载（Mount）：让存储可访问

**一句话定义**：挂载是将一个设备或文件系统附加到目录树的过程，使数据通过挂载点可访问。

### 挂载基础

```bash
# 1. 查看已挂载的文件系统
$ mount | head -5
/dev/sda2 on / type ext4 (rw,relatime)
proc on /proc type proc (rw,relatime,hidepid=2)
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec)
tmpfs on /run type tmpfs (rw,nosuid,nodev,size=392208k,nr_inodes=819200)

# 2. 手动挂载
$ sudo mount -t ext4 /dev/sdb1 /mnt/data
# -t ext4    文件系统类型
# /dev/sdb1  设备文件
# /mnt/data  挂载点

# 3. 查看磁盘和分区
$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0   120G  0 disk
├─sda1   8:1    0   512M  0 part /boot/efi
└─sda2   8:2    0 119.5G  0 part /
sdb      8:16   0   500G  0 disk
└─sdb1   8:17  0   500G  0 part
# sdb1还没挂载

# 4. 卸载
$ sudo umount /mnt/data
$ sudo umount /dev/sdb1              # 二选一都行
```

### /etc/fstab：开机自动挂载

```bash
# /etc/fstab格式：6列
# 设备                挂载点    FS类型   选项      dump  fsck
$ cat /etc/fstab
UUID=abc-123-...     /          ext4    defaults    0     1
UUID=def-456-...     /boot      ext4    defaults    0     2
UUID=ghi-789-...     /boot/efi  vfat    umask=0077  0     1
tmpfs                /tmp       tmpfs   defaults,size=1G  0 0
/dev/sdb1            /mnt/data  ext4    defaults,nofail  0 2
```

**字段详解**：
1. **设备**：用UUID更稳定（磁盘位置变也不影响）
2. **挂载点**：目录必须存在
3. **FS类型**：ext4, xfs, nfs, tmpfs, iso9660等
4. **选项**：逗号分隔
   - `defaults`：rw, suid, dev, exec, auto, nouser, async
   - `noexec`：禁止执行二进制
   - `nosuid`：忽略setuid
   - `nofail`：设备不存在时不报错（云环境友好）
   - `ro`：只读
5. **dump**：0=不备份，1=备份（基本淘汰）
6. **fsck顺序**：0=不检查，1=根，2=其他

```bash
# 1. 获取UUID
$ sudo blkid /dev/sdb1
/dev/sdb1: UUID="1234-5678" TYPE="ext4"

# 2. 测试fstab配置
$ sudo mount -a              # 应用所有未挂载的fstab项
# 不报错 = 配置正确

# 3. 重新挂载（修改选项后）
$ sudo mount -o remount,ro /mnt/data    # 重挂为只读
```

### 特殊挂载

```bash
# 1. 挂载ISO镜像（不需要刻录）
$ sudo mount -o loop ubuntu-24.04.iso /mnt/iso

# 2. 只读挂载关键目录（应急修复）
$ sudo mount -o remount,ro /        # 把根挂为只读，修复配置

# 3. 绑定挂载（bind mount）
$ sudo mount --bind /var/log /mnt/log-view      # 两个路径共享内容

# 4. 内存挂载（tmpfs）
$ sudo mount -t tmpfs -o size=100M tmpfs /mnt/ramdisk
```

### 实战：排查挂载问题

```bash
# 场景：fstab配置错误导致无法启动
# 解决：进入救援模式

# 1. 启动时内核命令行加上 single 或 init=/bin/bash
#    或GRUB按e编辑，去掉 rhgb quiet 看错误

# 2. 救援模式下：
$ mount -o remount,rw /                  # 根可写
$ vi /etc/fstab                          # 注释掉错误行
$ reboot

# 3. 避免"nofail"配置失误
#    nofail只在系统启动过程中不阻塞
#    但即使有nofail，systemd也会等待设备一段时间
```

---

## 2.7 ext4：Linux的"瑞士军刀"文件系统

**一句话定义**：ext4是Linux最主流的日志文件系统，平衡了性能、稳定性和特性，是Ubuntu/Debian默认。

### 关键特性

| 特性 | 说明 |
|------|------|
| **最大文件** | 16TB（理论上 1EB） |
| **最大分区** | 1EB |
| **日志（journaling）** | 崩溃后快速恢复，避免长时间fsck |
| **extents** | 用"起始块+长度"描述数据，减少碎片化 |
| **多块分配** | 一次分配多个块，提高大文件性能 |
| **延迟分配** | 写入时分配空间，提高连续性 |
| **在线扩容** | resize2fs 在线扩到最大 |

### 创建和检查ext4

```bash
# 1. 创建ext4文件系统
$ sudo mkfs.ext4 /dev/sdb1
mke2fs 1.47.0 (5-Feb-2024)
Creating filesystem with 131072000 4k blocks and 32768000 inodes
Filesystem UUID: 1234-5678-...
Superblock backups stored on blocks: ...
Allocating group tables: done
Writing inode tables: done
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done

# 2. 挂载
$ sudo mkdir /mnt/data
$ sudo mount /dev/sdb1 /mnt/data

# 3. 调整保留空间（默认5%给root）
$ sudo tune2fs -m 1 /dev/sdb1        # 改保留1%

# 4. 调整文件系统参数
$ sudo tune2fs -l /dev/sdb1          # 查看所有参数
$ sudo tune2fs -C 100 /dev/sdb1      # 设置挂载次数

# 5. 文件系统检查
$ sudo umount /dev/sdb1
$ sudo fsck.ext4 /dev/sdb1
$ sudo e2fsck -p /dev/sdb1           # -p自动修复

# 6. 调整ext4特性
$ sudo tune2fs -O has_journal,extents,uninit_bg /dev/sdb1
```

### 实战：扩容ext4

```bash
# 场景：LVM逻辑卷扩容后，扩展ext4

# 1. LVM扩容（假设逻辑卷是/dev/vg0/data）
$ sudo lvextend -L +10G /dev/vg0/data

# 2. 扩容ext4（在线，无需卸载）
$ sudo resize2fs /dev/vg0/data
resize2fs 1.47.0 ...
Resizing the filesystem on /dev/vg0/data to 5242880 (4k) blocks.
The filesystem on /dev/vg0/data is now 5242880 (4k) blocks long.

# 3. 验证
$ df -h /mnt/data
Filesystem             Size  Used Avail Use% Mounted on
/dev/mapper/vg0-data    20G  5.0G   14G  27% /mnt/data    # 容量增加
```

---

## 2.8 XFS：高性能的"老兵"

**一句话定义**：XFS是SGI 1994年开发的64位日志文件系统，RHEL 7+默认，擅长大文件和高并发。

### 关键特性

| 特性 | ext4 | XFS |
|------|------|-----|
| 最大文件 | 16TB | 9EB |
| 最大分区 | 1EB | 8EB |
| 默认日志 | 有 | 有 |
| 在线扩容 | ✅ | ✅ |
| **在线缩容** | ❌ | ❌ |
| 延迟分配 | ✅ | ✅ |
| **分配组** | ❌ | ✅（多线程并发写入） |
| 性能调优 | 一般 | **丰富**（xfs_db, xfs_fsr） |
| 修复 | e2fsck慢 | xfs_repair快 |
| 适合场景 | 通用 | 大文件/数据库/视频 |

### 创建XFS

```bash
# 1. 创建xfs文件系统
$ sudo mkfs.xfs /dev/sdb1
meta-data=/dev/sdb1              isize=512    agcount=4, agsize=3276800 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=13107200, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=6400, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

# 2. 挂载（带优化选项）
$ sudo mount -o noatime,largeio,inode64 /dev/sdb1 /mnt/data
# noatime:   不更新访问时间，IO优化
# largeio:   大IO请求
# inode64:   inode分散到所有AG（防止单AG耗尽）

# 3. 配额
$ sudo mount -o uquota,gquota /dev/sdb1 /mnt/data
$ xfs_quota -x -c 'limit bsoft=10g bhard=12g alice' /mnt/data
```

### XFS工具集

```bash
# 1. 碎片整理
$ sudo xfs_fsr /dev/sdb1                   # 全盘整理
$ sudo xfs_fsr -v /mnt/data/file.dat        # 单文件整理

# 2. 性能调优
$ sudo xfs_db -c "lsattr" /dev/sdb1
$ sudo xfs_db -c "stat" /dev/sdb1

# 3. 修复
$ sudo xfs_repair /dev/sdb1
# 比ext4的e2fsck快很多（基于元数据日志）

# 4. 查看使用情况
$ xfs_info /dev/sdb1
$ df -hT /mnt/data
```

### XFS vs ext4实战选择

| 场景 | 推荐 |
|------|------|
| 服务器根分区 | 都行（RHEL默认xfs） |
| 数据库（MySQL/PG） | **xfs**（大文件+并发） |
| 大量小文件 | ext4（小文件更友好） |
| 视频/媒体存储 | **xfs**（顺序IO + 8EB支持） |
| 容器overlay | ext4（Docker默认） |
| 个人电脑 | ext4（兼容性好） |
| NAS存储 | **xfs**（性能 + 容量） |

---

## 2.9 Btrfs：写时复制的"未来"文件系统

**一句话定义**：Btrfs（B-tree FS）借鉴ZFS思想，支持CoW（写时复制）、快照、子卷、压缩等高级特性，是Fedora和SUSE系列的默认根分区文件系统。⚠️ **RHEL 9/10不支持Btrfs**（默认及推荐均为XFS），仅Fedora/openSUSE/SLE/Ubuntu(可选)使用。

### 关键特性

| 特性 | 说明 |
|------|------|
| **CoW** | 写时复制，修改时复制原数据再写新数据 |
| **快照** | 秒级创建，可写或只读 |
| **子卷** | 独立挂载的B树根 |
| **压缩** | 透明压缩（zstd, lzo, zlib） |
| **RAID** | 内置RAID 0/1/5/6/10 |
| **校验和** | 数据+元数据checksum，自我修复 |
| **透明大页删除** | 高效处理稀疏文件 |
| **在线扩容** | ✅ |
| **在线缩容** | ❌（理论支持但工具未成熟） |
| **去重** | 需要时启用（默认关闭） |
| **send/receive** | 增量快照传输 |

### Btrfs生产现状（2024-2026）

| 发行版 | 状态 |
|--------|------|
| **RHEL 9.x / RHEL 10** | ❌ 不支持（默认及推荐均为 XFS） |
| **Fedora 33+** | ✅ 默认根分区（企业桌面/开发者首选） |
| **openSUSE Leap 15.4+** | ✅ 默认（配合 snapper 自动系统快照） |
| **SUSE Linux Enterprise 15 SP4+** | ✅ 默认 |
| **Debian 12** | 可选安装 |
| **Ubuntu 24.04** | 可选安装（默认仍是ext4） |
| **Arch** | 可选安装（Wiki详细支持文档） |

> **2026年生产判断**：Btrfs在Fedora/SUSE生态已成熟用于通用服务器/桌面。RHEL系列不支持，用XFS。极高并发OLTP数据库仍建议XFS或ext4。

### Btrfs实战

```bash
# 1. 创建Btrfs文件系统
$ sudo mkfs.btrfs -L "data" /dev/sdb1
btrfs-progs v6.7
See http://btrfs.wiki.kernel.org for more information.

Label:              data
UUID:               abc-def-ghi
Node size:          16384
Sector size:        4096
Filesystem size:    500.00GiB
Block group profiles:
  Data:             single            8.00MiB
  Metadata:         DUP              16.00MiB
  System:           DUP               8.00MiB
SSD detected:       yes
Incompat features:  extref, skinny-metadata, no-holes
Runtime features:   free-space-tree
Checksum:           crc32c
Number of devices:  1
...

# 2. 挂载
$ sudo mkdir /mnt/data
$ sudo mount /dev/sdb1 /mnt/data

# 3. 启用压缩（透明zstd压缩）
$ sudo mount -o remount,compress=zstd:3 /mnt/data
# 创建文件测试
$ ls -la /mnt/data/bigfile.dat
-rw-r--r-- 1 root root 1.0G Jul  2 10:00 /mnt/data/bigfile.dat
$ du -h /mnt/data/bigfile.dat
500M /mnt/data/bigfile.dat         # 实际占用500M（压缩了50%）

# 4. 创建子卷
$ sudo btrfs subvolume create /mnt/data/projects
Create subvolume '/mnt/data/projects'

# 5. 独立挂载子卷
$ sudo mount -o subvolid=257 /dev/sdb1 /mnt/projects

# 6. 快照（秒级）
$ sudo btrfs subvolume snapshot /mnt/data /mnt/data/snap-20260702
# 仅几GB数据的快照仅占几MB

# 7. 查看子卷
$ sudo btrfs subvolume list /mnt/data
ID 256 gen 30 top level 5 path @
ID 257 gen 31 top level 5 path @home
ID 258 gen 32 top level 5 path projects
ID 259 gen 33 top level 5 path snap-20260702
```

### Btrfs RAID

```bash
# 多盘Btrfs RAID（不需要mdadm）
$ sudo mkfs.btrfs -d raid1 -m raid1 /dev/sdb1 /dev/sdc1
# -d raid1: 数据raid1
# -m raid1: 元数据raid1

# 后续添加磁盘
$ sudo btrfs device add /dev/sdd1 /mnt/data
$ sudo btrfs balance start -dconvert=raid1 -mconvert=raid1 /mnt/data

# 查看
$ sudo btrfs filesystem show
Label: 'data'  uuid: abc-def
    Total devices 2 FS bytes used 100.00GiB
    devid    1 size 500.00GiB used 100.00GiB path /dev/sdb1
    devid    2 size 500.00GiB used 100.00GiB path /dev/sdc1
```

### Btrfs自动修复

```bash
# 自动检测并修复损坏数据（依赖RAID1/10/5/6）
$ sudo btrfs scrub start /mnt/data
$ sudo btrfs scrub status /mnt/data
scrub status for /mnt/data
    scrub started at Thu Jul  2 10:00:00 2026
    scrub finished at Thu Jul  2 10:30:00 2026
    duration: 30 minutes
    errors: 0
    corrected: 0
```

---

## 2.10 LVM：逻辑卷管理

**一句话定义**：LVM（Logical Volume Manager）将物理磁盘抽象为逻辑卷，支持在线扩容、缩容、快照，是企业存储的标配。

### LVM三层架构

```
物理卷 (PV)            卷组 (VG)              逻辑卷 (LV)
/dev/sdb1 ─┐                                       ┌─ /dev/vg0/data  → /mnt/data
/dev/sdc1 ─┼─→   vg0 (100G)   ───────────────────┼─ /dev/vg0/db    → /var/lib/mysql
/dev/sdd1 ─┘                                      └─ /dev/vg0/swap  → [swap]
```

| 概念 | 英文 | 说明 |
|------|------|------|
| **PV** | Physical Volume | 物理磁盘或分区（初始化后） |
| **VG** | Volume Group | 一个或多个PV组成的存储池 |
| **LV** | Logical Volume | 从VG划分出的逻辑卷，相当于"虚拟分区" |
| **PE** | Physical Extent | VG的最小分配单位，默认4MB |
| **LE** | Logical Extent | LV的最小分配单位 |

### LVM实战

```bash
# 1. 创建PV
$ sudo pvcreate /dev/sdb1 /dev/sdc1
  Physical volume "/dev/sdb1" successfully created.
  Physical volume "/dev/sdc1" successfully created.

# 2. 创建VG
$ sudo vgcreate vg0 /dev/sdb1 /dev/sdc1
  Volume group "vg0" successfully created

# 3. 创建LV
$ sudo lvcreate -L 50G -n data vg0          # 50GB
$ sudo lvcreate -l 100%FREE -n all vg0      # 全部剩余
$ sudo lvcreate -l 50%VG -n half vg0         # VG的50%

# 4. 创建文件系统并挂载
$ sudo mkfs.ext4 /dev/vg0/data
$ sudo mkdir /mnt/data
$ sudo mount /dev/vg0/data /mnt/data

# 5. 查看
$ sudo pvs
  PV         VG   Fmt  Attr PSize   PFree
  /dev/sdb1  vg0  lvm2 a--  100.00g 50.00g
  /dev/sdc1  vg0  lvm2 a--  100.00g 100.00g

$ sudo vgs
  VG   #PV #LV #SN Attr   VSize   VFree
  vg0    2   1   0 wz--n- 199.99g 150.00g

$ sudo lvs
  LV   VG   Attr       LSize   
  data vg0  -wi-ao----  50.00g
```

### 在线扩容（生产环境核心场景）

```bash
# 1. 物理增加磁盘
$ sudo pvcreate /dev/sdd1
$ sudo vgextend vg0 /dev/sdd1
  Volume group "vg0" successfully extended

# 2. LV扩容（在线）
$ sudo lvextend -L +50G /dev/vg0/data
  Extending logical volume data to 100.00 GiB
  Logical volume data successfully resized

# 3. 扩容文件系统（在线）
$ sudo resize2fs /dev/vg0/data           # ext4
# 或
$ sudo xfs_growfs /mnt/data              # xfs（注意是挂载点不是设备）

# 4. 验证
$ df -h /mnt/data
Filesystem             Size  Used Avail Use% Mounted on
/dev/mapper/vg0-data   100G  5.0G   90G   6% /mnt/data

# 一步到位（lvextend带-r选项）
$ sudo lvextend -L +50G -r /dev/vg0/data
# 自动检测FS类型并扩容
```

### LVM快照（备份/回滚）

```bash
# 1. 创建快照（占用空间很小，CoW机制）
$ sudo lvcreate -L 5G -s -n snap-data /dev/vg0/data
  Logical volume "snap-data" created

# 2. 挂载快照（只读备份）
$ sudo mkdir /mnt/snap
$ sudo mount -o ro /dev/vg0/snap-data /mnt/snap

# 3. 备份快照
$ sudo tar czf backup-$(date +%F).tar.gz -C /mnt/snap .

# 4. 还原（合并回原LV，危险操作！）
$ sudo umount /mnt/data /mnt/snap
$ sudo lvconvert --merge /dev/vg0/snap-data
# 合并完成后快照消失，data回到创建快照时的状态

# 5. 删除快照
$ sudo lvremove /dev/vg0/snap-data
```

### LVM灾难恢复

```bash
# 场景1：误删LVM分区 → 用LiveUSB恢复
# 1. 从Ubuntu LiveUSB启动
# 2. 激活LVM
$ sudo vgchange -ay                    # 激活所有VG
$ sudo lvs                              # 确认LV可见

# 3. 挂载并修复文件系统
$ sudo fsck.ext4 -f /dev/vg0/data
$ sudo mount /dev/vg0/data /mnt

# 4. 备份数据后重建
$ sudo tar czf /backup/data.tar.gz -C /mnt .
$ sudo umount /mnt

# 场景2：PV故障 → 强制移除
$ sudo vgreduce --removemissing vg0
$ cat /etc/lvm/archive/vg0_*.vg         # 查看历史VG元数据
# 如果PV可恢复：sudo pvcreate --uuid <原UUID> --restorefile <元数据> /dev/sdc1

# 场景3：XFS文件系统损坏
$ sudo xfs_repair -L /dev/vg0/data      # -L强制清理日志（⚠最后手段）
$ sudo xfs_repair -n /dev/vg0/data      # 先试只读检查

# 场景4：ext4文件系统损坏
$ sudo e2fsck -f -y /dev/vg0/data       # -f强制检查 -y自动修复
$ sudo e2fsck -p /dev/vg0/data          # -p自动修复（安全）

# 场景5：根分区满了（在线扩容）
$ sudo pvcreate /dev/sdb1
$ sudo vgextend vg0 /dev/sdb1
$ sudo lvextend -l +100%FREE -r /dev/vg0/root
$ df -h /                                # 验证
```

### 实战：LVM扩容踩坑（生产教训）

```bash
# 坑1：xfs_growfs 和 resize2fs 参数不同
$ sudo lvextend -L +10G /dev/vg0/data
# ext4：
$ sudo resize2fs /dev/vg0/data           # 用设备名
# xfs：
$ sudo xfs_growfs /mnt/data              # 用挂载点！（不是设备名）

# 坑2：扩容前未确认文件系统类型
$ df -hT /mnt/data                       # -T显示类型
Filesystem       Type  Size  Used Avail Use% Mounted on
/dev/vg0/data    xfs    50G   40G   11G  79% /mnt/data

# 坑3：lvextend -r 失败时手动处理
$ sudo lvextend -L +10G /dev/vg0/data    # 不跟-r
$ sudo fsadm resize /dev/vg0/data        # fsadm自动检测FS类型
```

```bash
# 场景：安装Linux时把整个系统盘用LVM
# 优点：后续扩容方便（特别是LVM上的/home、/var）

# /etc/fstab 会看到：
$ cat /etc/fstab | grep lvm
/dev/mapper/vg0-root    /     ext4  defaults  0 1
/dev/mapper/vg0-home    /home ext4  defaults  0 2
/dev/mapper/vg0-swap    swap  swap  defaults  0 0

# 调整：把/home扩容
$ sudo lvextend -l +100%FREE /dev/vg0/home
$ sudo resize2fs /dev/vg0/home
```

---

## 2.11 RAID：独立磁盘冗余阵列

**一句话定义**：RAID（Redundant Array of Independent Disks）通过多块磁盘组合，实现性能提升或数据冗余。

### 主流RAID级别对比

| RAID级别 | 冗余 | 性能 | 容量 | 最少磁盘 | 场景 |
|----------|------|------|------|----------|------|
| **RAID 0** | ❌ | 最高 | 100% | 2 | 临时数据，追求速度 |
| **RAID 1** | ✅（镜像） | 写慢，读快 | 50% | 2 | 系统盘，重要数据 |
| **RAID 5** | ✅（单盘） | 读快，写一般 | (N-1)/N | 3 | 通用服务器 |
| **RAID 6** | ✅（双盘） | 读快，写慢 | (N-2)/N | 4 | 大容量存储 |
| **RAID 10** | ✅（多盘） | 高 | 50% | 4 | 数据库，高性能 |
| **RAID 50** | ✅ | 较高 | (N-m)/N | 6 | 大容量+性能 |

### 软RAID实战（mdadm）

```bash
# 1. 安装mdadm
$ sudo apt install mdadm          # Debian/Ubuntu
$ sudo yum install mdadm          # RHEL/CentOS

# 2. 创建RAID 1（镜像）
$ sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb1 /dev/sdc1
mdadm: Note: this array has metadata at the start and
    may not be suitable as a boot device.  If you plan to
    store '/boot' on this device, please ensure that
    your boot-loader understands md/v1.x metadata, or use
    --metadata=0.90
Continue creating array? y
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.

# 3. 查看进度
$ cat /proc/mdstat
Personalities : [raid1]
md0 : active raid1 sdc1[1] sdb1[0]
      104857600 blocks super 1.2 [2/2] [UU]
      [>....................]  resync =  1.5% (1600000/104857600) finish=5.0min speed=320000K/sec
# 同步完成后变成 [UU]

# 4. 创建文件系统
$ sudo mkfs.ext4 /dev/md0
$ sudo mount /dev/md0 /mnt/data

# 5. 保存配置
$ sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
ARRAY /dev/md0 metadata=1.2 UUID=abc123:def456:...
$ sudo update-initramfs -u          # Debian/Ubuntu

# 6. 写入fstab自动挂载
$ echo "UUID=$(blkid -s UUID -o value /dev/md0) /mnt/data ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
```

### RAID运维：故障恢复

```bash
# 1. 模拟磁盘故障（标记为失败）
$ sudo mdadm --fail /dev/md0 /dev/sdb1
mdadm: set /dev/sdb1 faulty in /dev/md0

# 2. 查看状态
$ cat /proc/mdstat
md0 : active raid1 sdc1[1] sdb1[0](F)
#         ↑          ↑        ↑
#         同步盘      备用?     失败

# 3. 移除故障盘
$ sudo mdadm --remove /dev/md0 /dev/sdb1
mdadm: hot removed /dev/sdb1 from /dev/md0

# 4. 插入新盘（物理）+ 添加到RAID
$ sudo mdadm --add /dev/md0 /dev/sdd1
mdadm: added /dev/sdd1
# 自动开始同步（rebuild）

# 5. 监控同步进度
$ watch -n 1 cat /proc/mdstat
# 同步期间IO性能会下降

# 6. 查看详情
$ sudo mdadm --detail /dev/md0
/dev/md0:
        Version : 1.2
  Creation Time : ...
     Raid Level : raid1
     Array Size : 104857600 (100.00 GiB 107.37 GB)
  Used Dev Size : 104857600 (100.00 GiB)
   Raid Devices : 2
  Total Devices : 2
    Persistence : Superblock is persistent
    Update Time : ...
          State : clean
 Active Devices : 2
Working Devices : 2
 Failed Devices : 0
  Spare Devices : 0
```

### 软RAID vs 硬RAID vs LVM RAID

| 维度 | 软RAID (mdadm) | 硬RAID (RAID卡) | LVM RAID |
|------|---------------|----------------|----------|
| 性能 | 一般（吃CPU） | 最好（专用芯片） | 一般 |
| 成本 | 免费 | 贵（硬件） | 免费 |
| 灵活性 | 高 | 低（卡绑定） | 中等 |
| 跨多OS | 通用 | 需同型号卡 | Linux only |
| 适用 | 中小企业 | 大型数据库 | 已用LVM的场景 |

> **现代推荐**：LVM on top of mdadm，或直接用LVM的raid1/raid5/raid6（Linux 3.10+）。

---

## 2.12 Swap：内存不足的"应急位"

**一句话定义**：Swap是磁盘上的交换空间（分区或文件），当物理内存不足时，内核将不活跃的内存页换出到Swap。

### Swap类型

| 类型 | 性能 | 灵活性 | 推荐场景 |
|------|------|--------|----------|
| **Swap分区** | 最佳 | 低（创建后难调整） | 安装时规划 |
| **Swap文件** | 略差 | 高（随时调大小） | 动态调整 |
| **zram** | 内存压缩 | - | 容器/无盘系统 |

### 创建Swap

**方式1：Swap分区（传统）**

```bash
# 1. 创建分区
$ sudo fdisk /dev/sdb
# n (new) -> p (primary) -> 回车 -> 回车 -> t (type) -> 82 (Linux swap) -> w

# 2. 格式化
$ sudo mkswap /dev/sdb2

# 3. 启用
$ sudo swapon /dev/sdb2

# 4. 永久生效：写入fstab
$ echo "UUID=$(blkid -s UUID -o value /dev/sdb2) none swap sw 0 0" | sudo tee -a /etc/fstab
```

**方式2：Swap文件（推荐，更灵活）**

```bash
# 1. 创建4GB的swap文件
$ sudo fallocate -l 4G /swapfile
# 或
$ sudo dd if=/dev/zero of=/swapfile bs=1M count=4096

# 2. 设置权限（仅root可访问）
$ sudo chmod 600 /swapfile

# 3. 格式化
$ sudo mkswap /swapfile

# 4. 启用
$ sudo swapon /swapfile

# 5. 永久生效
$ echo "/swapfile none swap sw 0 0" | sudo tee -a /etc/fstab

# 6. 在线调整大小
$ sudo swapoff /swapfile
$ sudo fallocate -l 8G /swapfile       # 改为8G
$ sudo mkswap /swapfile
$ sudo swapon /swapfile
```

**方式3：zram（内存压缩）**

```bash
# Ubuntu 22.04+默认启用
$ sudo apt install systemd-zram-generator

# /etc/systemd/zram-generator.conf
[zram0]
zram-size = ram * 2
compression-algorithm = zstd
swap-priority = 100
fs-type = swap

$ sudo systemctl start systemd-zram-setup@zram0
$ zramctl
NAME       ALGORITHM DISKSIZE DATA COMPR TOTAL STREAMS MOUNTPOINT
/dev/zram0 zstd           8G   1G  320M  400M       8 [SWAP]
```

### 调优参数：swappiness

```bash
# 查看当前值
$ cat /proc/sys/vm/swappiness
60
# 范围0-200，值越大越倾向使用swap

# 含义：
# 0:   不到万不得已不用swap
# 10:  适合数据库服务器
# 60:  默认值
# 100: 积极使用swap

# 调整（临时）
$ sudo sysctl vm.swappiness=10

# 永久：写入/etc/sysctl.d/99-swap.conf
$ echo "vm.swappiness=10" | sudo tee /etc/sysctl.d/99-swap.conf
$ sudo sysctl -p /etc/sysctl.d/99-swap.conf
```

### Swap配置建议（2026年生产环境）

| 物理内存 | 建议Swap | 备注 |
|----------|----------|------|
| < 2GB | 2x RAM | 内存太小 |
| 2-8GB | 1x RAM | 传统建议 |
| 8-64GB | 4-8GB | 足够应急 |
| > 64GB | 0-4GB | Swap几乎不需要 |

**数据库服务器**：
- 关闭Swap或设置swappiness=1
- 或分配小Swap防OOM（Nginx类反而可以多Swap）

```bash
# MySQL推荐：swappiness=1
$ sudo sysctl -w vm.swappiness=1
```

---

## 2.13 /proc：内核的眼睛

**一句话定义**：`/proc`是内核导出的虚拟文件系统，不占磁盘空间，提供进程和内核的实时信息。

### 关键路径速查

```bash
# CPU信息
$ cat /proc/cpuinfo
processor   : 0
vendor_id   : GenuineIntel
model name  : Intel(R) Xeon(R) CPU E5-2680 v4 @ 2.40GHz
cpu MHz     : 2400.000
cache size  : 35840 KB
...

# 内存信息
$ cat /proc/meminfo
MemTotal:       8092852 kB
MemFree:        5023144 kB
MemAvailable:   6820000 kB
Buffers:          51200 kB
Cached:         2017280 kB
SwapTotal:      2097148 kB
SwapFree:       2097148 kB
...

# 系统运行信息
$ cat /proc/uptime
86400.00 84320.00
# ↑运行时间(秒)  ↑空闲时间(秒)

$ cat /proc/loadavg
0.50 0.40 0.30 1/234 5678
# 1分钟/5分钟/15分钟负载 / 运行进程数/总进程数 / 最近PID

# 内核版本
$ cat /proc/version
Linux version 6.6.21-linuxkit ...

# 已加载文件系统
$ cat /proc/filesystems
nodev   sysfs
nodev   tmpfs
nodev   bdev
        ext4
        xfs
nodev   overlay
...

# 网络接口
$ cat /proc/net/dev
Inter-|   Receive                                                |  Transmit
 face |bytes    packets errs drop fifo frame compressed multicast|bytes    packets errs drop ...
  eth0: 1234567  10000    0    0    0    0          0          0  1234567  10000    0    0 ...

# 进程PID 1信息
$ ls /proc/1/
cmdline   cwd      fd/      maps      net/      root      statm
comm      environ  fdinfo/  mem       ns/       stat      status
task/

# 读进程命令行
$ cat /proc/1/cmdline
/usr/lib/systemd/systemd\x00
# 注意：每个参数用\x00分隔，不是空格

# 读进程工作目录（是个符号链接）
$ readlink /proc/1/cwd
/
```

### 实战：通过/proc调优内核

```bash
# 1. 临时改TCP连接队列上限
$ echo 4096 | sudo tee /proc/sys/net/core/somaxconn
# 永久：写入/etc/sysctl.d/

# 2. 查看进程打开的文件
$ ls /proc/1234/fd/ | head
0
1
2
3
4
# 数字是文件描述符

$ readlink /proc/1234/fd/3
/var/log/nginx/access.log

# 3. 查看进程内存映射
$ cat /proc/1234/maps | head
55f4a3c00000-55f4a3c20000 r--p 00000000 08:02 131072 /usr/bin/nginx

# 4. 查进程当前状态
$ cat /proc/1234/status
Name:   nginx
State:  S (sleeping)
Pid:    1234
PPid:   1
VmRSS:    50000 kB       # 实际物理内存
VmSize:   200000 kB      # 虚拟地址空间
Threads:  4
...
```

---

## 2.14 /sys：设备的管家

**一句话定义**：`/sys`是sysfs虚拟文件系统，提供设备、驱动、电源、总线等硬件信息的统一视图，是udev的基础。

### 关键路径

```bash
# 网卡信息
$ ls /sys/class/net/
eth0  lo
$ cat /sys/class/net/eth0/address
00:16:3e:5a:7b:1c
$ cat /sys/class/net/eth0/speed
10000                          # Mbps，10Gbps
$ cat /sys/class/net/eth0/operstate
up

# 块设备
$ ls /sys/block/
sda  sdb
$ cat /sys/block/sda/queue/rotational
1                              # 1=HDD，0=SSD
$ cat /sys/block/sda/size
234441648                     # 512字节扇区数

# CPU信息
$ ls /sys/devices/system/cpu/ | grep cpu | head
cpu0  cpu1  cpu2  cpu3

# 在线调节CPU频率
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
performance

# 改成节能模式
$ echo powersave | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# USB设备
$ ls /sys/bus/usb/devices/
1-1  1-1.1  usb1  ...

# 内核模块
$ ls /sys/module/ext4/
```

### 实战：通过/sys调优

```bash
# 1. 修改块设备调度器
$ cat /sys/block/sda/queue/scheduler
mq-deadline kyber [none] bfq
# 当前是[none]，意味着多队列无调度（SSD推荐）

# 改用deadline
$ echo mq-deadline | sudo tee /sys/block/sda/queue/scheduler
mq-deadline kyber [mq-deadline] bfq

# 2. 禁用磁盘写缓存（防数据丢失）
$ cat /sys/block/sdb/queue/write_cache
write back
$ echo "write through" | sudo tee /sys/block/sdb/queue/write_cache
# 性能降低但更安全

# 3. 查看设备电源状态
$ cat /sys/class/scsi_host/host*/link_power_management_policy
med_power_with_dipm

# 4. 查看CPU温度
$ ls /sys/class/thermal/thermal_zone*/
$ cat /sys/class/thermal/thermal_zone0/temp
45000                         # 45.0°C（毫摄氏度）
```

---

## 2.15 /dev：设备文件目录

**一句话定义**：`/dev`包含所有设备文件（块设备、字符设备），是用户空间访问硬件的接口。

### 关键设备节点

```bash
# 块设备
$ ls -la /dev/sd* /dev/nvme* /dev/vd*
brw-rw---- 1 root disk 8,   0 Jul  2 10:00 /dev/sda
brw-rw---- 1 root disk 8,  16 Jul  2 10:00 /dev/sdb
brw-rw---- 1 root disk 259, 0 Jul  2 10:00 /dev/nvme0n1

# 字符设备
$ ls -la /dev/null /dev/zero /dev/random /dev/console
crw-rw-rw- 1 root root 1, 3 Jul  2 10:00 /dev/null
crw-rw-rw- 1 root root 1, 5 Jul  2 10:00 /dev/zero
crw-rw-rw- 1 root root 1, 8 Jul  2 10:00 /dev/random
crw-w---- 1 root tty  5, 1 Jul  2 10:00 /dev/console

# 数字含义：主(major) 次(minor)
# 主设备号 = 设备类型（8=SATA，1=mem，5=tty）
# 次设备号 = 同类型设备的编号（0, 1, 2...）
```

### 常用设备节点速查

| 设备 | 类型 | 用途 |
|------|------|------|
| `/dev/null` | 字符 | "黑洞"，丢弃所有写入 |
| `/dev/zero` | 字符 | 无限零字节流 |
| `/dev/random` | 字符 | 真随机数（熵池满时阻塞） |
| `/dev/urandom` | 字符 | 伪随机数（不阻塞） |
| `/dev/tty` | 字符 | 当前进程的终端 |
| `/dev/console` | 字符 | 系统控制台 |
| `/dev/pts/N` | 字符 | 伪终端（SSH登录） |
| `/dev/stdin` | 符号链接 | 标准输入（fd 0） |
| `/dev/stdout` | 符号链接 | 标准输出（fd 1） |
| `/dev/stderr` | 符号链接 | 标准错误（fd 2） |
| `/dev/loopN` | 块 | 回环设备（挂载ISO） |
| `/dev/sdX` | 块 | SATA/SCSI磁盘 |
| `/dev/nvme0n1` | 块 | NVMe SSD |
| `/dev/md0` | 块 | 软件RAID设备 |
| `/dev/dm-N` | 块 | device-mapper（LVM） |
| `/dev/shm` | tmpfs | 共享内存tmpfs |

### udev：动态设备管理

```bash
# 1. 查看udev管理的设备
$ udevadm info --query=all --name=/dev/sda

# 2. 触发设备重扫（插入新硬件或修改规则后）
$ sudo udevadm trigger
$ sudo udevadm control --reload

# 3. 监控设备事件（实时）
$ udevadm monitor
monitor will print the received events for:
UDEV - the event which udev sends out after rule processing
KERNEL - the kernel uevents
KERNEL[12345.6] add /devices/.../sdb1
UDEV  [12345.7] add /devices/.../sdb1

# 4. 编写udev规则（/etc/udev/rules.d/）
$ cat /etc/udev/rules.d/99-usb-backup.rules
# 当USB序列号为12345的设备插入时，创建符号链接
SUBSYSTEM=="block", ATTRS{serial}=="12345", SYMLINK+="usb_backup"
```

### 实战：用dd测试磁盘

```bash
# 1. 顺序写入测试（注意会覆盖目标）
$ sudo dd if=/dev/zero of=/tmp/testfile bs=1M count=1024 oflag=direct
1073741824 bytes (1.1 GB) copied, 5.234 s, 205 MB/s
# oflag=direct 绕过页面缓存，真实磁盘速度

# 2. 顺序读测试
$ sudo dd if=/tmp/testfile of=/dev/null bs=1M iflag=direct
1073741824 bytes (1.1 GB) copied, 4.123 s, 260 MB/s

# 3. 看设备型号
$ cat /sys/block/sda/device/model
Samsung SSD 980 PRO 1TB

# 4. 清理
$ rm /tmp/testfile
```

---

## 2.16 /tmp：临时文件目录

**一句话定义**：`/tmp`用于存放临时文件，任何用户可创建，但只能删除自己的文件（粘滞位保护）。

### 关键属性

```bash
$ ls -ld /tmp
drwxrwxrwt 12 root root 4096 Jul  2 10:00 /tmp
# ↑粘滞位（t）

# 查看挂载
$ df -h /tmp
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        50G  5.0G   42G  11% /
# 或
tmpfs           3.9G  4.0K  3.9G   1% /tmp
# 很多发行版把/tmp挂为tmpfs

# 调整tmpfs大小（默认50% RAM）
# /etc/fstab:
tmpfs /tmp tmpfs defaults,size=2G 0 0
```

### 粘滞位（Sticky Bit）

```bash
# 防止用户互相删除/tmp下的文件
$ ls -ld /tmp
drwxrwxrwt
# ↑最后的t就是粘滞位

# 测试：
$ alice$ touch /tmp/alice.txt
$ bob$ rm /tmp/alice.txt
rm: cannot remove '/tmp/alice.txt': Operation not permitted
# bob是/tmp的"其他用户"，但不能删alice的文件
# 只有文件所有者、目录所有者或root才能删
```

### /tmp vs /var/tmp

| 维度 | /tmp | /var/tmp |
|------|------|----------|
| 文件保留 | 重启清空 | 重启保留（但可能被定期清理） |
| 挂载 | tmpfs（内存） | 真实磁盘 |
| 速度 | 极快 | 一般 |
| 用途 | 应用临时数据 | 需重启保留的临时数据 |
| 清理 | 立即 | 老系统有tmpwatch，新系统有systemd-tmpfiles |

```bash
# systemd-tmpfiles：自动清理
$ cat /etc/tmpfiles.d/tmp.conf
# 清理10天未访问的/tmp文件
D /tmp 1777 root root 10d

$ cat /etc/tmpfiles.d/var-tmp.conf
# 清理30天未访问的/var/tmp
D /var/tmp 1777 root root 30d

# 手动清理
$ sudo systemd-tmpfiles --clean
```

---

## 2.17 /var：可变数据中枢

**一句话定义**：`/var`存放系统运行过程中持续增长的数据——日志、缓存、数据库、邮件队列、临时文件等。

### 关键子目录

```bash
$ ls /var/
backups  cache  lib  local  lock  log  mail  opt  run  spool  tmp
```

| 目录 | 用途 | 典型内容 |
|------|------|----------|
| `/var/log` | 日志 | syslog, nginx/, mysql/ |
| `/var/lib` | 应用状态数据 | mysql/, docker/, apt/ |
| `/var/cache` | 缓存 | apt缓存, pip缓存, 字体缓存 |
| `/var/spool` | 队列数据 | cron/, mail/, print/ |
| `/var/mail` | 邮件队列 | 用户邮箱 |
| `/var/lock` | 锁文件 | .LCK.. |
| `/var/tmp` | 长期临时文件 | 应用临时数据 |
| `/var/run` | → /run | 已弃用 |
| `/var/backups` | 系统备份 | dpkg备份、shadow备份 |

### 实战：/var/log日志管理

```bash
# 1. 主流日志位置
/var/log/syslog              # Debian/Ubuntu 系统日志
/var/log/messages            # RHEL/CentOS 系统日志
/var/log/auth.log            # Ubuntu 认证日志
/var/log/secure              # RHEL 认证日志
/var/log/kern.log            # 内核日志
/var/log/nginx/              # Nginx
/var/log/mysql/              # MySQL
/var/log/journal/            # systemd journal（结构化日志）

# 2. 日志轮转配置：/etc/logrotate.conf + /etc/logrotate.d/

# 3. 当前日志大小
$ du -sh /var/log/*
8.0M    /var/log/syslog
120M    /var/log/nginx/access.log
50M     /var/log/journal/

# 4. 找大日志
$ find /var/log -type f -size +100M 2>/dev/null

# 5. 强制轮转
$ sudo logrotate -f /etc/logrotate.conf
```

### 实战：/var/lib应用数据

```bash
# Docker
/var/lib/docker/             # 容器镜像、卷
/var/lib/docker/overlay2/    # overlay2存储驱动

# 数据库
/var/lib/mysql/              # MySQL数据文件
/var/lib/postgresql/         # PostgreSQL
/var/lib/redis/              # Redis持久化
/var/lib/mongodb/            # MongoDB

# 包管理器
/var/lib/dpkg/               # Debian已安装包信息
/var/lib/rpm/                # RPM数据库
/var/lib/apt/                # apt状态

# 系统状态
/var/lib/systemd/            # systemd状态
/var/lib/dhcp/               # DHCP租约
```

---

## 2.18 /run 运行时数据

**一句话定义**：`/run`是tmpfs挂载的临时目录，存放系统启动以来的运行时数据。

```bash
# 1. 查看大小（默认50%内存）
$ df -h /run
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           3.9G  4.0K  3.9G   1% /run

# 2. 关键文件
$ ls /run/
nginx.pid                     # 服务PID文件
sshd.pid
utmp                          # 当前登录用户
user/1000/                    # 用户运行时
systemd/                      # systemd状态
log/                          # 部分服务日志
lock/                         # 锁文件

# 3. 读取登录用户
$ who -a
           system boot  2026-07-02 09:00
alice    + tty1         2026-07-02 09:00
bob      + pts/0        2026-07-02 09:30 (192.168.1.100)
```

> 详细内容参见第一章1.18。

---

## 本章小结

### 文件系统选型速查

```
服务器根分区:    xfs (RHEL) | ext4 (Ubuntu) | btrfs (SUSE/Fedora)
数据库:          xfs         | ext4          | btrfs (压测后)
大量小文件:      ext4
大文件/视频:     xfs
NAS/存储:        xfs (优先) | btrfs
容器overlay:     ext4 (兼容)
国产化:          ext4 (兼容性优先)
```

### 关键场景决策树

```
需要冗余？
├─ 是 → RAID 1/5/6/10 (mdadm或硬件)
│       └─ 还要灵活？→ LVM on top of mdadm
└─ 否 → 单盘
        └─ 要快照？→ btrfs / ZFS
            └─ 要扩容？→ LVM
                └─ 简单？→ 纯 ext4/xfs
```

### 故障排查清单

```bash
$ df -h                     # 1. 磁盘空间
$ df -i                     # 2. inode使用
$ mount | grep "ro,"        # 3. 是否只读挂载
$ dmesg | tail -20          # 4. 内核错误
$ cat /proc/mounts          # 5. 实际挂载点
$ lsof | grep deleted       # 6. 已删除但仍占空间的文件
$ du -sh /* 2>/dev/null | sort -h   # 7. 找大目录
$ lsblk                     # 8. 磁盘拓扑
```

### 下一章预告

**第三章：文件操作与权限** — chmod/chown/chgrp/umask/ACL/特殊权限位/Capabilities/ls -l/find/xargs/sort|uniq。深入理解Linux的权限模型。

---

> **本章字数**：约 19,000 字
> **涉及命令**：stat, ln, lsblk, mount, mkfs, blkid, tune2fs, mdadm, pvcreate, vgcreate, lvcreate, resize2fs
> **基准版本**：Linux 6.6 LTS, Btrfs 6.7, xfsprogs 6.5

# 第三章：文件操作与权限

> **本章定位**：Linux的权限模型是安全架构的基石。从`rwx`到ACL，从setuid到Linux Capabilities，理解这些才能真正掌控"谁能对什么做什么"。

---

## 3.1 文件权限（rwx）：三组三位

**一句话定义**：Linux用9位二进制（3组×3位）表示文件权限，分别对应所有者、所属组、其他用户。

### 权限结构详解

```bash
$ ls -l /etc/passwd
-rw-r--r-- 1 root root 2841 Jun 15 09:00 /etc/passwd
↑└┬┘└┬┘└┬┘
 │ │  │  │
 │ │  │  └── 其他用户(o): r-- (4)
 │ │  └───── 所属组(g): r-- (4)
 │ └──────── 所有者(u): rw- (6)
 └────────── 文件类型: - (普通文件)
```

### 文件类型字符

| 字符 | 类型 |
|------|------|
| `-` | 普通文件 |
| `d` | 目录 |
| `l` | 符号链接 |
| `b` | 块设备 |
| `c` | 字符设备 |
| `p` | 命名管道 |
| `s` | 套接字（socket） |

### 三种权限位

| 权限 | 数字 | 含义（文件） | 含义（目录） |
|------|------|-------------|-------------|
| `r` | 4 | 读 | 列目录（ls） |
| `w` | 2 | 写 | 增删文件 |
| `x` | 1 | 执行 | 进入目录（cd） |

### 实战：目录权限的"反直觉"

```bash
# 目录的可读不等于可访问！
$ mkdir /tmp/test && chmod 444 /tmp/test
$ ls /tmp/test/        # 可读 - 列出文件
file1  file2
$ cd /tmp/test         # 不可写 - 拒绝
bash: cd: /tmp/test: Permission denied
# 因为cd需要"执行"权限，不是"读"权限

# 给目录加上x
$ chmod 555 /tmp/test
$ cd /tmp/test         # 现在可进了
$ ls
file1  file2
# 但仍不能创建/删除文件（缺w）
$ touch newfile
touch: cannot touch 'newfile': Permission denied
```

### 八进制转权限

```bash
# 数字模式 = 三组分别求和
# r=4, w=2, x=1

# rwxr-xr-x = 755
#  rwx  = 4+2+1 = 7
#  r-x  = 4+0+1 = 5
#  r-x  = 4+0+1 = 5

# 常见权限
chmod 644 file.txt   # rw-r--r-- (普通文件默认)
chmod 755 script.sh  # rwxr-xr-x (脚本默认)
chmod 600 id_rsa     # rw------- (SSH私钥必须)
chmod 700 ~/.ssh     # rwx------ (SSH目录必须)
chmod 644 /etc/passwd
chmod 640 /etc/shadow  # 重要文件，只有root和shadow组能读
```

---

## 3.2 chmod：修改权限

### 三种修改方式

**1. 八进制（推荐，最常用）**

```bash
$ chmod 755 script.sh
$ chmod 644 config.yaml
$ chmod 600 secret.key
$ chmod -R 755 /opt/myapp/  # 递归
```

**2. 符号模式（精细控制）**

```bash
# u = 所有者 (user)
# g = 所属组 (group)
# o = 其他 (other)
# a = 全部 (all)

# 操作：+ 添加 - 移除 = 设置
# 权限：r w x

$ chmod u+x script.sh           # 所有者加执行
$ chmod g-w file.txt            # 组移除写
$ chmod o=r file.txt            # 其他设为只读
$ chmod a+r file.txt           # 所有人加读
$ chmod u=rwx,g=rx,o= file.sh  # 完整描述
$ chmod -R u+w,g-w directory/   # 递归
```

**3. 特殊模式（copy）**

```bash
# 参考其他文件的权限
$ chmod --reference=file1 file2
# file2的权限 = file1的权限
```

### 实战：脚本部署的标准流程

```bash
# 1. 写脚本
$ cat > deploy.sh <<'EOF'
#!/bin/bash
echo "Deploying..."
EOF

# 2. 默认没有x
$ ls -l deploy.sh
-rw-r--r-- 1 alice alice 30 Jul  2 10:00 deploy.sh
$ ./deploy.sh
bash: ./deploy.sh: Permission denied

# 3. 添加x
$ chmod +x deploy.sh
$ ./deploy.sh
Deploying...

# 4. 安全做法：644+x（所有者可执行，其他只读）
$ chmod 755 deploy.sh
$ ls -l deploy.sh
-rwxr-xr-x 1 alice alice 30 Jul  2 10:00 deploy.sh
```

### 故障排查：权限问题

```bash
# 场景：能进目录但访问文件报错
$ cat /var/log/app.log
cat: /var/log/app.log: Permission denied

# 排查步骤
$ ls -l /var/log/app.log           # 看文件权限
-rw------- 1 root root 1024 Jul  2 10:00 /var/log/app.log
# 文件属主是root，权限600，其他人无任何权限

$ id                                 # 确认当前用户
uid=1000(alice) gid=1000(alice)

# 解决：让alice属于能读的组，或加sudo
$ sudo cat /var/log/app.log

# 或修改权限
$ sudo chmod 644 /var/log/app.log
```

---

## 3.3 chown：修改所有者和组

**一句话定义**：`chown`修改文件/目录的所有者(owner)和所属组(group)。

### 语法

```bash
chown [选项] user[:group] file

# 只改所有者
$ sudo chown alice file.txt

# 改所有者和组
$ sudo chown alice:staff file.txt

# 只改组（等价chgrp）
$ sudo chown :staff file.txt

# 递归
$ sudo chown -R alice:alice /home/alice/

# 引用其他文件的属性
$ sudo chown --reference=ref.txt target.txt
```

### 实战场景

```bash
# 1. 新建用户后初始化家目录
$ sudo useradd -m -d /home/bob bob
$ ls -ld /home/bob
drwxr-x--- 2 bob bob 4096 Jul  2 10:00 /home/bob
# 属主属组都是bob，权限750

# 2. 把整个目录交给新用户
$ sudo cp -r /opt/template /opt/bobapp
$ sudo chown -R bob:bob /opt/bobapp

# 3. 让多个用户共享文件
$ sudo chown :devteam /opt/projects/
$ sudo chmod 2775 /opt/projects/   # 2=setgid，新文件自动继承组
$ ls -ld /opt/projects/
drwxrwsr-x 4 root devteam 4096 Jul  2 10:00 /opt/projects/
#          ↑  s是setgid位
```

### 权限：只有root能改

```bash
$ alice$ chown bob file.txt        # 报错
chown: changing ownership of 'file.txt': Operation not permitted

# 原因：只有root和文件当前所有者能把所有权给别人
# 实际上：
# - 文件所有者可以把"组"改为自己所属的组
# - 任何情况下不能"把文件送给别人"
# - 只有root能改所有者
```

### 实战：网站部署权限

```bash
# 场景：网站文件 /var/www/html 由www-data用户维护
$ ls -l /var/www/html/index.html
-rw-r--r-- 1 www-data www-data 1024 Jul  2 10:00 /var/www/html/index.html

# 部署时上传文件
$ sudo chown -R www-data:www-data /var/www/html/
$ sudo chmod -R 644 /var/www/html/    # 文件644
$ sudo find /var/www/html -type d -exec chmod 755 {} \;  # 目录755
```

---

## 3.4 chgrp：单独改组

**一句话定义**：`chgrp`专门修改文件/目录的所属组（chown的子集）。

```bash
# 语法
chgrp [选项] group file

# 修改单个文件
$ sudo chgrp devteam file.txt

# 递归
$ sudo chgrp -R developers /opt/code/

# 普通用户可以改组（但有限制）
# 1. 必须是当前用户所属的组
$ id alice
uid=1000(alice) gid=1000(alice) groups=1000(alice),1005(devteam)
$ chgrp devteam file.txt              # 可以，因为alice属于devteam
$ chgrp othergroup file.txt           # 失败
chgrp: changing group of 'file.txt': Operation not permitted
```

### chown vs chgrp

| 命令 | 改所有者 | 改组 | 谁能用 |
|------|----------|------|--------|
| chown | ✅ | ✅ | root |
| chgrp | ❌ | ✅ | root + 当前用户（限自身组） |

> **经验法则**：统一用`chown`即可，符号`:`的写法更清晰：`chown user:group file`。

---

## 3.5 umask：默认权限的"减法器"

**一句话定义**：umask是文件创建时的权限掩码，从系统默认权限中"减去"umask值，得到实际权限。

### 基础概念

```
文件默认权限：666 (rw-rw-rw-) - 文件默认没有x
目录默认权限：777 (rwxrwxrwx) - 目录需要x

实际权限 = 默认权限 - umask
```

### 实战

```bash
# 1. 查看当前umask
$ umask
0022
# 八进制0022

# 2. 计算实际权限
#   文件：666 - 022 = 644 (rw-r--r--)
#   目录：777 - 022 = 755 (rwxr-xr-x)

# 3. 验证
$ umask
0022
$ touch testfile && ls -l testfile
-rw-r--r-- 1 alice alice 0 Jul  2 10:00 testfile        # 644 ✓
$ mkdir testdir && ls -ld testdir
drwxr-xr-x 2 alice alice 4096 Jul  2 10:00 testdir      # 755 ✓

# 4. 改变umask（临时）
$ umask 0077
$ touch testfile && ls -l testfile
-rw------- 1 alice alice 0 Jul  2 10:00 testfile        # 600 (更安全)

# 5. 改回
$ umask 0022
```

### 安全的umask

| umask | 文件默认 | 目录默认 | 场景 |
|-------|----------|----------|------|
| `022` | 644 | 755 | 通用，组成员可读 |
| `002` | 664 | 775 | 组内协作 |
| `077` | 600 | 700 | 严格个人 |
| `007` | 660 | 770 | 仅本组成员 |

### 永久设置

```bash
# 1. 全局：/etc/profile 或 /etc/bash.bashrc
# 大多数系统默认：if [ $UID -gt 199 ] && [ "`id -u`" -gt 199 ]; then umask 002; else umask 022; fi

# 2. 用户：~/.bashrc
$ cat >> ~/.bashrc <<'EOF'
umask 022
EOF
$ source ~/.bashrc

# 3. 脚本中：放在脚本开头
#!/bin/bash
umask 022
```

### 实战：跨平台的umask问题

```bash
# 场景：Docker容器中umask可能不是0022
$ docker exec myapp umask
0000
# 这意味着新文件是666，新目录是777
# 在容器中要明确设置：

# Dockerfile
RUN echo "umask 0022" >> /etc/profile

# 或在docker run时
$ docker run -e UMASK=0022 myapp
```

---

## 3.6 ACL：细粒度权限控制

**一句话定义**：ACL（Access Control List）扩展了rwx三组模型，允许为特定用户或组单独设置权限。

### 为什么需要ACL？

```bash
# 场景：/opt/project 目录
# - alice 需要读写
# - bob 只能读
# - devteam组需要读写
# - 其他用户无权限

# 用传统rwx很难满足
# 用ACL可以精细控制
```

### ACL实战

```bash
# 1. 检查文件系统是否支持ACL
$ sudo tune2fs -l /dev/sda1 | grep "Default mount options"
Default mount options:    user_xattr acl
# 看到acl = 支持

# 2. 确认挂载带acl
$ mount | grep sda1
/dev/sda1 on / type ext4 (rw,relatime,acl)        # 看到acl
# 如果没有：sudo mount -o remount,acl /

# 3. 设置ACL
$ setfacl -m u:alice:rw /opt/project/file.txt
# u:alice:rw  = 给alice用户设rw权限

$ setfacl -m u:bob:r /opt/project/file.txt
# u:bob:r     = 给bob用户设r权限

$ setfacl -m g:devteam:rw /opt/project/file.txt
# g:devteam:rw = 给devteam组设rw权限

# 4. 查看ACL
$ getfacl /opt/project/file.txt
# file: /opt/project/file.txt
# owner: root
# group: root
user::rw-
user:alice:rw-
user:bob:r--
group::r--
group:devteam:rw-
mask::rwx
other::r--

# 5. 递归设置
$ setfacl -R -m u:alice:rwX /opt/project/
# X=大写X，表示"目录加x，文件不加x（除非已有）"

# 6. 删除ACL
$ setfacl -x u:bob /opt/project/file.txt      # 删除bob的ACL
$ setfacl -b /opt/project/file.txt             # 删除所有ACL
```

### 关键概念：ACL mask

```bash
# mask是"有效权限上限"，会与所有非所有者非其他的ACL条目相与
# 例：mask=r-x，实际alice的ACL是rw-，但有效只有r-x

# 查看mask
$ getfacl file
user::rwx
user:alice:rw-         # 显式
group::r-x
mask::r-x              # ← mask
other::r--

# 改mask
$ setfacl -m m::rwx file
```

### 实战：默认ACL（目录继承）

```bash
# 在目录上设default ACL，新文件/子目录自动继承

$ setfacl -d -m u:alice:rwX /opt/project/
$ setfacl -d -m g:devteam:rwX /opt/project/

# 查看
$ getfacl /opt/project/
default:user::rwx
default:user:alice:rwX
default:group::r-x
default:group:devteam:rwX
default:mask::rwx
default:other::r-x

# 测试：新建文件自动有ACL
$ touch /opt/project/newfile.txt
$ getfacl /opt/project/newfile.txt
user:alice:rw-
group:devteam:rw-
# 自动继承！
```

### ACL vs chmod对比

| 维度 | chmod | ACL |
|------|-------|-----|
| 粒度 | 三组 | 任意用户/组 |
| 适合 | 简单权限 | 复杂共享 |
| 性能 | 略好 | 略差（额外查找） |
| 兼容性 | POSIX | POSIX.1e（ext4/xfs/btrfs支持） |
| 推荐 | 默认 | 复杂场景 |

---

## 3.7 setuid / setgid：临时借权

**一句话定义**：`setuid`让用户以文件所有者的身份执行程序；`setgid`让用户以文件所属组的身份执行，或让新文件继承目录的组。

### setuid（4xxx）

```bash
# 经典例子：passwd
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 68208 May  2  2023 /usr/bin/passwd
   ↑
   s位 = setuid
# 普通用户执行passwd时，临时获得root权限
# 这样能写/etc/shadow

# 数字设置
$ chmod 4755 /usr/local/bin/myapp
# 4 + 755 = s权限+rwxr-xr-x

# 符号设置
$ chmod u+s /usr/local/bin/myapp
```

### setgid（2xxx）

```bash
# 1. 文件上的setgid：执行时获得文件组身份
$ chmod 2755 /usr/local/bin/group-tool
# 2 + 755

# 2. 目录上的setgid：新建文件自动继承目录组
$ mkdir /opt/shared
$ sudo chown :devteam /opt/shared
$ sudo chmod 2775 /opt/shared
$ ls -ld /opt/shared
drwxrwsr-x 2 root devteam 4096 Jul  2 10:00 /opt/shared
#            ↑ s

# 测试
$ touch /opt/shared/test.txt
$ ls -l /opt/shared/test.txt
-rw-r--r-- 1 alice devteam 0 Jul  2 10:00 /opt/shared/test.txt
# 所属组自动是devteam（继承自目录）
```

### 安全警告：setuid是安全重灾区

```bash
# ⚠️ setuid程序是攻击者的目标
# 1. 避免自制setuid程序
# 2. 定期审计系统中的setuid
$ find / -perm -4000 -type f 2>/dev/null
/usr/bin/passwd
/usr/bin/su
/usr/bin/sudo
/usr/bin/mount
/usr/bin/umount
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/pkexec
# 看到不认识的就要警惕

# 3. 移除不需要的setuid
$ sudo chmod u-s /usr/bin/pkexec    # 比如polkit漏洞多
# 或更彻底：
$ sudo dpkg-statoverride --update --add root root 0755 /usr/bin/pkexec
```

### sticky bit（1xxx）重提

```bash
# 已在第二章2.16讲过，这里看完整形式
$ chmod 1777 /tmp
$ ls -ld /tmp
drwxrwxrwt 12 root root 4096 Jul  2 10:00 /tmp
# 1777 = 1 + 777 = sticky+rwxrwxrwx

# 4+2+1 = 7xxx组合使用
$ chmod 7755 /usr/local/bin/super-tool
# 4+2+1+755 = 4+2+1 + u=rwx,g=rx,o=rx
# 4=setuid, 2=setgid, 1=sticky
```

### 八进制特殊位速记

| 八进制 | 含义 |
|--------|------|
| 0xxx | 无特殊位 |
| 1xxx | sticky bit |
| 2xxx | setgid |
| 4xxx | setuid |
| 7xxx | setuid + setgid + sticky |

---

## 3.8 粘滞位（Sticky Bit）再深入

**应用场景**：
- `/tmp` 防止用户互删文件
- 公共上传目录（如网盘）防止被恶意清理

```bash
# 1. 创建公共目录
$ sudo mkdir /opt/public-uploads
$ sudo chmod 1777 /opt/public-uploads
$ ls -ld /opt/public-uploads
drwxrwxrwt 3 root root 4096 Jul  2 10:00 /opt/public-uploads

# 2. 验证粘滞位效果
$ alice$ touch /opt/public-uploads/alice.txt
$ bob$ touch /opt/public-uploads/bob.txt
$ bob$ rm /opt/public-uploads/alice.txt
rm: cannot remove '/opt/public-uploads/alice.txt': Operation not permitted
# bob删不了alice的文件

# 3. 实战：网盘场景
# 业务上传目录：/var/www/upload/
$ sudo chmod 1777 /var/www/upload/
# 任何用户都能上传，但只能删自己的文件
```

---

## 3.9 chattr / lsattr：文件属性与不可变位

**一句话定义**：`chattr`改变ext2/ext3/ext4文件系统的扩展属性，包括"不可变"、"仅追加"等高级保护。

### 关键属性

| 属性 | 含义 | 典型用途 |
|------|------|----------|
| `a` | 只能追加（append-only） | 日志文件 |
| `i` | 不可变（immutable） | 关键配置文件 |
| `A` | 不更新atime | SSD优化 |
| `c` | 自动压缩 | 冷数据 |
| `d` | 不备份到dump | dump备份 |
| `S` | 同步更新 | 数据安全 |
| `u` | 删除时保留（undeletable） | 重要数据 |

### 实战：防止关键文件被篡改

```bash
# 1. 设置不可变（防御勒索软件）
$ sudo chattr +i /etc/passwd
$ sudo chattr +i /etc/shadow
$ sudo chattr +i /etc/sudoers
$ sudo chattr +i /etc/ssh/sshd_config

# 验证
$ sudo rm /etc/passwd
rm: cannot remove '/etc/passwd': Operation not permitted
# ⚠️ root也删不了！

# 修改前必须先解除
$ sudo chattr -i /etc/passwd
# 修改
# 重新锁定
$ sudo chattr +i /etc/passwd

# 2. 日志文件：只允许追加
$ sudo chattr +a /var/log/secure
$ echo "test" >> /var/log/secure       # OK
$ echo "test" > /var/log/secure        # 失败
bash: /var/log/secure: Operation not permitted
$ sudo rm /var/log/secure              # 失败
rm: cannot remove '/var/log/secure': Operation not permitted

# 3. 查看属性
$ lsattr /etc/passwd
----i---------e------- /etc/passwd
#    ↑ i是不可变

$ lsattr /var/log/secure
-----a--------e------- /var/log/secure
#      ↑ a是append-only
```

### 批量锁定关键文件

```bash
#!/bin/bash
# /usr/local/bin/lock-critical-files.sh
# 锁定关键配置文件，防篡改

CRITICAL_FILES=(
    /etc/passwd
    /etc/shadow
    /etc/group
    /etc/gshadow
    /etc/sudoers
    /etc/ssh/sshd_config
    /etc/pam.d/*
    /etc/security/*
)

for f in "${CRITICAL_FILES[@]}"; do
    if [ -e "$f" ]; then
        sudo chattr +i "$f"
        echo "Locked: $f"
    fi
done

# 用法：
# 1. 部署前运行
# 2. 修改前先unlock
# 3. 改完再lock
# 4. 加入systemd timer定期执行
```

### chattr的限制

```bash
# chattr仅适用于ext2/ext3/ext4
# XFS不支持chattr（用xfs_io的相关功能）
# Btrfs用chattr +C（禁用CoW）

# Btrfs相关属性
$ sudo chattr +C /var/lib/mysql/        # 禁用CoW（数据库需要）
# CoW对数据库的双写机制是冗余的，反而降低性能

# 验证
$ lsattr /var/lib/mysql/
---------------C-- /var/lib/mysql/
#              ↑ C = no copy-on-write
```

---

## 3.10 Linux Capabilities：精细化root权限

**一句话定义**：Linux Capabilities将root的"超级权限"拆分为独立的位（capability），程序可以只获得所需权限，最小化风险。

### 为什么需要Capabilities？

```bash
# 传统问题：让普通用户绑定80端口
# 方法1：setuid给root权限（太危险，整个root权限）
$ sudo chmod u+s /usr/bin/nginx
# 攻击者拿下nginx = 拿到整个root

# 方法2：Capabilities（精确授权）
$ sudo setcap cap_net_bind_service+ep /usr/bin/nginx
# 只给"绑定<1024端口"的能力，没有其他root权限
```

### 常用Capabilities

| Capability | 权限 |
|------------|------|
| `CAP_NET_BIND_SERVICE` | 绑定<1024端口 |
| `CAP_NET_RAW` | 使用RAW套接字（ping, tcpdump） |
| `CAP_NET_ADMIN` | 网络管理（iptables, 路由） |
| `CAP_SYS_ADMIN` | 大量管理操作（mount等） |
| `CAP_SYS_TIME` | 设置系统时间 |
| `CAP_SYS_PTRACE` | ptrace其他进程（gdb attach） |
| `CAP_DAC_OVERRIDE` | 绕过文件权限检查 |
| `CAP_CHOWN` | 改变文件属主 |
| `CAP_FOWNER` | 绕过属主权限检查 |
| `CAP_KILL` | 发送信号给任意进程 |
| `CAP_AUDIT_WRITE` | 写审计日志 |
| `CAP_SYS_RESOURCE` | 资源限制覆盖 |
| `CAP_MKNOD` | 创建设备文件 |
| `CAP_SYS_RAWIO` | I/O权限操作 |
| `CAP_SYS_BOOT` | 重新启动系统 |

完整列表：`man 7 capabilities`

### 实战

```bash
# 1. 让非root用户启动80端口的nginx
$ id www-data
uid=33(www-data) gid=33(www-data) groups=33(www-data)

$ sudo setcap cap_net_bind_service+ep /usr/sbin/nginx
$ getcap /usr/sbin/nginx
/usr/sbin/nginx cap_net_bind_service=ep
#             ↑ 已经设置

# 验证
$ sudo -u www-data nginx -g 'daemon off;'
# 现在能用www-data用户启动监听80端口的nginx

# 2. python绑定80端口
$ setcap cap_net_bind_service=+ep $(which python3)
$ cat > bind80.py <<'EOF'
import socket
s = socket.socket()
s.bind(('0.0.0.0', 80))
s.listen()
print("Bound to 80")
EOF
$ python3 bind80.py
Bound to 80
# OK！普通用户也能绑80

# 3. tcpdump让普通用户能用
$ sudo setcap cap_net_raw,cap_net_admin+ep /usr/bin/tcpdump
$ getcap /usr/bin/tcpdump
/usr/bin/tcpdump cap_net_admin,cap_net_raw=ep

# 4. 查看进程当前能力
$ getpcaps <pid>
# 0 1 2 3 4 ... 等等，对应bit位

# 5. 删除能力
$ setcap -r /usr/sbin/nginx
# 恢复成无能力
```

### 完整文件capability（自Linux 2.6.24）

```bash
# Per-document capabilities（文件能力）的格式
# cap_<NAME>=<effective,permitted,inheritable>
# +ep  = 设置 effective + permitted
# +epx = + effective + permitted + inheritable

# 实际例子
setcap cap_net_bind_service,cap_net_raw+ep /usr/bin/ping
# 让ping能创建RAW socket，无需setuid

# 用法实践
# /usr/bin/ping 多数发行版已经不带setuid
# 而是设置了cap_net_raw能力
$ ls -l /bin/ping
-rwxr-xr-x 1 root root 64496 ... /bin/ping
# 注意没有s位！
$ getcap /bin/ping
/bin/ping cap_net_raw=ep
```

### 系统所有能力集合

```bash
# 当前shell的能力
$ cat /proc/self/status | grep Cap
CapInh: 0000000000000000
CapPrm: 00000000a80425fb
CapEff: 00000000a80425fb
CapBnd: 00000000a80425fb
CapAmb: 0000000000000000

# 解码capability位
$ capsh --decode=00000000a80425fb
0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap

# 完整能力列表
$ capsh --print
```

### 实战：容器中Capabilities

```bash
# Docker默认丢弃大量能力
$ docker run --rm alpine cat /proc/self/status | grep Cap
CapInh: 0000000000000000
CapPrm: 00000000a80425fb       # 默认12个
CapEff: 00000000a80425fb
CapBnd: 00000000a80425fb

# 完整能力（不安全！）
$ docker run --rm --cap-add=ALL alpine cat /proc/self/status | grep Cap
CapEff: 0000003fffffffff        # 全部37个

# 减少能力（更安全）
$ docker run --rm --cap-drop=NET_RAW alpine sh -c "ping 8.8.8.8"
ping: permission denied (are you running as root?)

# 实战：sys容器需要SYS_ADMIN
$ docker run --cap-add=SYS_ADMIN ...
```

---

## 3.11 ls -l：看文件必备

**一句话定义**：`ls -l`（长格式列出）是Linux查看文件信息的"瑞士军刀"。

### 输出字段详解

```bash
$ ls -l /etc/passwd
-rw-r--r-- 1 root root 2841 Jun 15 09:00 /etc/passwd
└┬┘ └┬┘  │  └┬┘ └┬┘ │  └────┬────┘ └────┬────┘
 │   │   │   │   │   │       │           │
 │   │   │   │   │   │   修改时间      文件名
 │   │   │   │   │   └──── 文件大小（字节）
 │   │   │   │   └──── 所属组
 │   │   │   └──── 所有者
 │   │   └──── 硬链接数
 │   └──── 权限（9位）
 └──── 文件类型
```

### 常用ls选项

```bash
# 长格式
$ ls -l

# 显示隐藏文件
$ ls -la
$ ls -A    # 不显示 . 和 ..

# 显示inode
$ ls -i
131200 file.txt
$ ls -li file.txt
131200 -rw-r--r-- 1 alice alice 0 Jul  2 10:00 file.txt

# 按大小排序
$ ls -lhS                      # 降序
$ ls -lhSr                     # 升序

# 按时间排序
$ ls -lt                       # 修改时间
$ ls -ltr                      # 逆序（最新在底）

# 显示目录树
$ ls -R /opt/

# 人类可读
$ ls -lh
total 60K
-rw-r--r-- 1 root root  2.8K Jun 15 09:00 passwd

# 显示ACL标记（+）或SELinux
$ ls -lZ
-rw-r--r--. root root unconfined_u:object_r:etc_t:s0 file

# 多种排序
$ ls -l --sort=size
$ ls -l --sort=time

# 颜色（默认开启）
$ ls --color=auto
# 不同类型不同颜色：目录蓝色，链接青色，压缩文件红色等
```

### 实战：批量文件分析

```bash
# 1. 找最大的10个文件
$ ls -lhS /var/log/ | head -10

# 2. 找最近修改的5个文件
$ ls -lt /var/log/ | head -5

# 3. 找所有隐藏文件
$ ls -ld .??*                  # 文件名以.开头
$ ls -A | grep '^\.'            # 同上

# 4. 找777权限的文件（潜在风险）
$ ls -l | awk '$1 ~ /rwxrwxrwx/ {print}'

# 5. 找带ACL的文件
$ ls -l | awk '$1 ~ /\+/ {print}'
#         ↑ 最后是+表示有ACL

# 6. 按扩展名统计
$ ls -1 /opt/myapp/logs/ | awk -F. '{print $NF}' | sort | uniq -c | sort -rn
   1230 log
    456 json
    234 txt
```

---

## 3.12 find：Linux文件搜索之王

**一句话定义**：`find`是Linux最强大的文件搜索工具，支持按名称/类型/大小/时间/权限等条件递归查找。

### 基础语法

```bash
find [起始路径] [条件] [动作]
```

### 常用条件

```bash
# 按名称
$ find / -name "*.log" 2>/dev/null
$ find . -iname "*.JPG"           # -i 忽略大小写

# 按类型
$ find / -type f                  # 普通文件
$ find / -type d                  # 目录
$ find / -type l                  # 符号链接
$ find / -type b                  # 块设备

# 按大小
$ find / -size +100M              # 大于100M
$ find / -size -10k               # 小于10K
$ find / -size 50M                # 正好50M
# 单位：c=字节 k=KB M=MB G=TB

# 按时间
$ find /var/log -mtime -7         # 7天内修改
$ find /tmp -mtime +30            # 30天前修改
$ find / -atime -1                # 1天内访问
$ find / -ctime -1                # 1天内状态变更
# -mmin, -amin, -cmin（用分钟）

# 按权限
$ find / -perm 777                # 权限777
$ find / -perm -4000              # 有setuid位
$ find / -perm -u+s               # 同上（符号）
$ find / -perm /u+w               # 所有者有w（不管其他）

# 按所有者和组
$ find / -user alice
$ find / -group devteam
$ find / -uid 1000
$ find / -nouser                  # 属主已删除
$ find / -nogroup                 # 属组已删除

# 组合条件
$ find / -name "*.log" -size +10M
$ find / \( -name "*.log" -o -name "*.tmp" \) -mtime +7
# 多个-name用-o
# 注意：\( \) 要转义

# 排除路径
$ find / -path "/proc" -prune -o -name "*.conf" -print
# 在/中搜索但跳过/proc
```

### 常用动作

```bash
# 默认：-print（打印到屏幕）

# -exec：对结果执行命令
$ find /tmp -name "*.tmp" -exec rm {} \;
# {} 是占位符，\; 是结束符
# 每找到一个执行一次，效率一般

# -exec +：合并执行（更高效）
$ find /tmp -name "*.tmp" -exec rm {} +
# 类似 xargs，把参数批量传给命令

# -ok：exec的确认版
$ find /tmp -name "*.tmp" -ok rm {} \;
# 每个操作前询问 y/n

# -delete：直接删除
$ find /tmp -name "*.tmp" -delete
# 慎用！

# -ls：详细列出
$ find / -size +1G -ls 2>/dev/null
# 类似ls -l
```

### 实战：find高级用法

```bash
# 1. 找大文件（>1GB）
$ find / -type f -size +1G 2>/dev/null | xargs ls -lhS 2>/dev/null | head -10

# 2. 清理7天前的日志
$ find /var/log/myapp -name "*.log" -mtime +7 -delete

# 3. 找30分钟前修改的PHP文件（疑似被篡改）
$ find /var/www -name "*.php" -mmin -30 -ls

# 4. 找可执行文件
$ find /opt/myapp -type f -executable

# 5. 找空目录
$ find /data -type d -empty

# 6. 找空文件
$ find /data -type f -empty

# 7. 找被删除但仍被打开的文件
$ find /proc/*/fd -ls 2>/dev/null | grep deleted
# 或
$ lsof | grep deleted | awk '{print $4}' | sort -u

# 8. 找硬链接数>1的文件
$ find / -type f -links +1 2>/dev/null

# 9. 复制查找到的文件
$ find /src -name "*.conf" -exec cp {} /dest/ \;

# 10. 批量修改权限
$ find /var/www -type f -exec chmod 644 {} \;
$ find /var/www -type d -exec chmod 755 {} \;
```

### find vs locate

| 维度 | find | locate |
|------|------|--------|
| 数据库 | 无（实时扫描） | 每天更新的索引库 |
| 速度 | 慢 | 极快 |
| 实时性 | 实时 | 略滞后 |
| 资源占用 | 高 | 低 |
| 适合 | 精确搜索 | 模糊快速搜索 |

```bash
# locate使用
$ sudo apt install mlocate           # 安装
$ sudo updatedb                     # 手动更新数据库
$ locate nginx.conf
$ locate -i "*.conf"                # 忽略大小写
$ locate -c "*.log"                 # 只统计
$ locate -e nginx                   # 仅存在的文件
```

---

## 3.13 xargs：管道的"终结者"

**一句话定义**：`xargs`把标准输入的数据转换成命令行参数，解决"参数列表过长"问题。

### 为什么需要xargs？

```bash
# 1. 直接管道的问题
$ find /tmp -name "*.log" | rm
# xargs: 报错或行为异常
# 实际：rm rm /tmp/a.log /tmp/b.log
# 多个参数会被rm当成文件名

# 2. 正确做法：用xargs把stdin转参数
$ find /tmp -name "*.log" | xargs rm
# xargs把stdin拼接成 rm /tmp/a.log /tmp/b.log ...

# 3. 极端案例：参数超过ARG_MAX
$ find / -name "*.conf" | xargs ls
# 直接用管道可能超长
```

### 核心选项

```bash
# -n N：每次传N个参数
$ echo "a b c d e" | xargs -n 2 echo
a b
c d
e
# 一行传2个

# -I {}：占位符（每行一个）
$ find . -name "*.txt" | xargs -I {} cp {} /backup/
# {} 代表每一行的输入

# -P N：并行N个进程
$ find . -name "*.jpg" -print0 | xargs -0 -P 8 -I {} convert {} -resize 50% {}.small.jpg
# 8个进程并行处理图片

# -0：用null字符分隔（处理带空格/特殊字符的文件名）
$ find . -name "*.txt" -print0 | xargs -0 rm
# 防止文件名含空格导致错误

# -d DELIM：自定义分隔符
$ echo "a:b:c" | xargs -d: echo
a b c

# -t：打印执行的命令（调试用）
$ find /tmp -name "*.log" | xargs -t rm
```

### 实战：xargs经典案例

```bash
# 1. 批量重命名
$ ls *.jpg | xargs -I {} mv {} {}.bak
# a.jpg → a.jpg.bak

# 2. 统计代码行数
$ find . -name "*.py" | xargs wc -l
# 总行数

# 3. 搜索内容
$ grep -l "TODO" *.c | xargs sed -i 's/TODO/FIXME/'
# 在所有含TODO的.c文件里替换

# 4. 杀进程
$ ps aux | grep "python app.py" | grep -v grep | awk '{print $2}' | xargs kill

# 5. 并行下载
$ cat urls.txt | xargs -P 4 -I {} wget -q {}
# 4个并发下载

# 6. 安全删除（处理空格文件名）
$ find . -name "*.tmp" -print0 | xargs -0 rm -f

# 7. 压缩所有日志
$ find /var/log/myapp -name "*.log" | xargs -I {} gzip {}
```

### 实战：xargs与find的-exec区别

```bash
# 1. xargs：所有参数一次传给命令
$ find . -name "*.txt" | xargs rm
# 等同：rm file1 file2 file3 ...

# 2. -exec：每个文件启动一个新命令
$ find . -name "*.txt" -exec rm {} \;
# 等同：rm file1; rm file2; rm file3 ...
# 慢

# 3. -exec +：类似xargs
$ find . -name "*.txt" -exec rm {} +
# 等同：rm file1 file2 file3 ...

# 性能对比（10000个文件）：
# find -exec \;      : 30秒
# find -exec +        : 5秒
# find | xargs        : 5秒
```

---

## 3.14 sort / uniq：文本处理双剑客

**一句话定义**：`sort`对文本行排序；`uniq`去重（必须先排序）。

### sort基础

```bash
# 1. 默认字典序
$ sort file.txt

# 2. 数字排序
$ sort -n file.txt                # 把"10"看作10而非"1,0"

# 3. 逆序
$ sort -r file.txt

# 4. 按字段（-k）
$ sort -k2 file.txt               # 按第2列
$ sort -k2,2 file.txt             # 仅按第2列
$ sort -k2n file.txt              # 第2列按数字
$ sort -t: -k3n /etc/passwd       # 用:分隔，按第3列UID数字排序

# 5. 去重
$ sort -u file.txt                # 排序+去重

# 6. 检查是否已排序
$ sort -c file.txt
# 没排序会报错

# 7. 大小写处理
$ sort -f file.txt                # 忽略大小写
$ sort -d file.txt                # 仅字母数字，忽略标点
```

### uniq基础

```bash
# 1. 必须先排序（uniq只比较相邻行）
$ sort file.txt | uniq

# 2. 统计重复次数
$ sort file.txt | uniq -c
   3 apple
   2 banana
   1 cherry

# 3. 只显示重复行
$ sort file.txt | uniq -d

# 4. 只显示唯一的行
$ sort file.txt | uniq -u

# 5. 忽略前N个字段比较
$ sort file.txt | uniq -f1         # 跳过第1字段
$ sort file.txt | uniq -s5         # 跳过前5字符

# 6. 不区分大小写
$ sort file.txt | uniq -i
```

### 实战：日志分析黄金组合

```bash
# 1. 访问最多的10个IP
$ awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
   1234 192.168.1.100
    567 10.0.0.5
    ...

# 2. 状态码统计
$ awk '{print $9}' access.log | sort | uniq -c | sort -rn
 12345 200
   567 404
   234 500

# 3. URL访问Top 10
$ awk '{print $7}' access.log | sort | uniq -c | sort -rn | head -10

# 4. 当前活跃用户会话
$ who | awk '{print $1}' | sort | uniq -c
   2 alice
   1 bob
   1 charlie

# 5. 进程按CPU使用排序
$ ps aux | sort -k3 -rn | head
# 按第3列（CPU）数字降序

# 6. 内存占用Top 10
$ ps aux | sort -k4 -rn | head

# 7. 用户登录次数统计
$ last | awk '{print $1}' | sort | uniq -c | sort -rn

# 8. 找出文件中出现次数>100的单词
$ cat text.txt | tr ' ' '\n' | sort | uniq -c | awk '$1>100' | sort -rn
```

### 实战：多文件对比

```bash
# 1. 两个文件的并集
$ sort file1 file2 | uniq
# 或
$ cat file1 file2 | sort -u

# 2. 两个文件的交集
$ comm -12 <(sort file1) <(sort file2)
# comm需要排序过的输入

# 3. 文件1有但文件2没有
$ comm -23 <(sort file1) <(sort file2)

# 4. 文件2有但文件1没有
$ comm -13 <(sort file1) <(sort file2)
```

### sort/uniq vs awk/python性能

```bash
# 处理1GB文件时：
# sort + uniq: 10-30秒
# awk: 20-60秒
# python: 60-180秒

# sort是用C写的，性能最强
# 但处理复杂逻辑时awk更合适
```

---

## 本章小结

### 权限模型速查

```
+---------------------------+
| 文件类型  所有者  组  其他 |  ls -l
|   -      rwx   rwx  rwx   |
+---------------------------+
| 9位 rwx + 3位特殊位         |
| setuid (4) setgid (2) sticky (1)
+---------------------------+

普通文件: 644
脚本:    755
私钥:    600
目录:    755
共享目录: 2775 (setgid + 775)
公共临时: 1777 (sticky + 777)
```

### 决策树

```
需要细粒度控制？
├─ 是 → ACL (setfacl/getfacl)
└─ 否 → rwx 够用

需要临时借权？
├─ 是 → Capabilities (setcap)  ← 推荐
│       或 setuid (chmod u+s)  ← 谨慎
└─ 否 → 正常使用

需要防篡改？
├─ 是 → chattr +i (关键文件)
│       chattr +a (日志)
└─ 否 → 常规

需要搜索文件？
├─ 实时 → find
├─ 快速 → locate
└─ 批量处理 → find -exec 或 xargs
```

### 应急工具

```bash
# 误改权限恢复（恢复常见权限）
$ sudo chmod 644 /etc/passwd
$ sudo chmod 640 /etc/shadow
$ sudo chmod 644 /etc/group
$ sudo chmod 640 /etc/gshadow
$ sudo chmod 600 /etc/ssh/ssh_host_*

# 误删/篡改应急
$ sudo chattr -i /etc/passwd
$ sudo cp /etc/passwd- /etc/passwd   # 从备份恢复

# 排查777权限
$ find / -perm 0777 -type f 2>/dev/null
$ find / -perm 0777 -type d 2>/dev/null
```

### 下一章预告

**第四章：进程与作业管理** — 进程生命周期、守护进程、PID/PPID、僵尸进程、ps/top/htop、信号、前后台作业、nice/renice、systemd-cgtop。

---

> **本章字数**：约 18,500 字
> **涉及命令**：chmod, chown, chgrp, umask, setfacl, getfacl, chattr, lsattr, setcap, getcap, find, xargs, sort, uniq
> **基准版本**：Linux 6.6 LTS, libcap 2.69, acl 2.3.1

# 第四章：进程与作业管理

> **本章定位**：进程是Linux运行的实体。理解进程的生命周期、父子关系、状态转换，掌握ps/top/htop/kill的实战用法，你就从"会用Linux"升级为"懂Linux运行机制"。

---

## 4.1 进程（Process）：运行的程序实例

**一句话定义**：进程是程序在计算机上的一次执行活动，是系统进行资源分配和调度的基本单位，拥有独立地址空间、文件描述符、环境变量等资源。

### 进程的"身份证"

```bash
# 查看进程基本信息
$ cat /proc/1/status
Name:   systemd
Umask:  0000
State:  S (sleeping)        # ← 状态：S=sleeping
Tgid:   1
Ngid:   1
Pid:    1                   # ← 进程ID
PPid:   0                   # ← 父进程ID（0=内核）
TracerPid:      0
Uid:    0   0   0   0       # 真实/有效/保存/fs UID
Gid:    0   0   0   0
FDSize: 128                  # 文件描述符表大小
Threads:        1            # 线程数
SigQ:   0/63706
SigPnd: 0000000000000000
ShdPnd: 0000000000000000
SigBlk: 0000000000000000
SigIgn: 0000000000000000
SigCgt: fffffffe7f3bfeff   # 捕获的信号
CapInh: 0000000000000000   # 继承能力
CapPrm: 00000000a80425fb   # 许可能力
CapEff: 00000000a80425fb   # 有效能力
CapBnd: 00000000a80425fb   # 边界能力
CapAmb: 0000000000000000   # 模糊能力
Seccomp:        2           # seccomp模式
NoNewPrivs:     0
Speculation_Store_Bypass:   thread vulnerable
Cpus_allowed:   ffffffff
Cpus_allowed_list:      0-31
Mems_allowed:   00000001
Mems_allowed_list:      0
voluntary_ctxt_switches:        1
nonvoluntary_ctxt_switches:     0

# 关键字段说明：
# Pid:     进程唯一ID
# PPid:    父进程ID
# State:   R=运行 S=睡眠 D=不可中断睡眠 Z=僵尸 T=停止 I=空闲
# Uid:     真实/有效/保存/fs四组
# Threads: 进程内的线程数
# Cap*:    能力（Capabilities）
```

### 进程的一生

```
创建 (fork)
  ↓
就绪 (Ready) ──── 被调度选中
  ↓              ↓
   ←── 被切换出去
  ↓
运行 (Running)
  ↓
睡眠 (Sleeping) ── 等待资源/事件
  ↓
  ├─ 唤醒 → 继续运行
  └─ 停止 (Stopped) ── 收到SIGSTOP
                    ├─ 收到SIGCONT → 继续
                    └─ 收到SIGKILL → 终止
  ↓
终止 (Terminated)
  ↓
  ├─ 父进程wait() → 释放资源
  └─ 父进程不wait() → 僵尸进程
```

### 进程创建：fork + exec

```bash
# 1. 查看进程树
$ pstree -p
systemd(1)─┬─NetworkManager(789)─┬─dhclient(891)
           │                      └─{NetworkManager}(792)
           ├─sshd(1056)───sshd(2345)───bash(2367)───pstree(2400)
           ├─nginx(1200)─┬─nginx(1201)
           │              ├─nginx(1202)
           │              └─nginx(1203)
           └─cron(1100)

# 2. 看进程命令行
$ cat /proc/1234/cmdline | tr '\0' ' '
/usr/sbin/nginx -g daemon on; master_process on;
# 每个参数用\0分隔，需要转换

# 3. 看进程环境变量
$ cat /proc/1234/environ | tr '\0' '\n'
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
HOME=/var/www
LANG=en_US.UTF-8
# 注意：environ权限是-r--------，只有进程属主能读
```

---

## 4.2 守护进程（Daemon）：后台服务

**一句话定义**：守护进程是脱离终端控制、在后台持续运行的服务进程（如sshd、nginx、cron）。

### 守护进程的"出生"流程

```c
// 经典daemon化代码（简化版）
1. fork()              // 父进程退出，子进程成为孤儿
2. setsid()            // 创建新会话，脱离原终端
3. fork() again        // 防止重新获得终端
4. chdir("/")          // 切换工作目录
5. umask(0)            // 重设umask
6. close(0/1/2)        // 关闭标准IO
7. open /dev/null as stdin/stdout/stderr
```

### systemd时代的守护进程

```bash
# 现代daemon几乎都由systemd管理
# 它们不fork两次，而是systemd fork+exec它们

# 1. 查看所有daemon
$ systemctl list-units --type=service --state=running
UNIT                       LOAD   ACTIVE SUB     DESCRIPTION
nginx.service              loaded active running A high performance web server
ssh.service                loaded active running OpenBSD Secure Shell server
systemd-journald.service   loaded active running Journal Service
...

# 2. 自己编写daemon（systemd风格）
$ cat > /etc/systemd/system/myapp.service <<EOF
[Unit]
Description=My Application
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/bin/myapp --daemon
ExecReload=/bin/kill -HUP $MAINPID
PIDFile=/run/myapp.pid
User=myapp
Group=myapp
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

$ sudo systemctl daemon-reload
$ sudo systemctl start myapp
```

### 实战：传统方式启动后台进程

```bash
# 1. 用&后台运行
$ ./myapp &               # 后台启动
$ jobs                     # 查看当前shell的后台作业
[1]+ Running   ./myapp &

# 2. 用nohup不挂起
$ nohup ./myapp > /var/log/myapp.log 2>&1 &
# nohup忽略SIGHUP，关闭终端不退出

# 3. 用setsid完全脱离终端
$ setsid ./myapp < /dev/null > /var/log/myapp.log 2>&1 &
# 进程属于新会话，关闭终端不影响

# 4. 用disown把作业从job table移除
$ ./myapp &
$ disown                  # 即使关掉shell，进程继续跑
```

### 守护进程的特征

```bash
# 1. PPID=1（被init收养）
$ ps -o pid,ppid,cmd -p $(pgrep nginx)
  PID  PPID CMD
 1200     1 nginx: master process
 1201  1200 nginx: worker process
 1202  1200 nginx: worker process

# 2. TTY=?（无终端）
$ ps -o pid,tty,cmd -p $(pgrep nginx)
  PID TT       CMD
 1200 ?        nginx: master process
#              ↑ TTY是?表示无终端

# 3. SESSION ID=1（独立会话）
$ ps -o pid,sid,cmd -p $(pgrep nginx)
  PID  SID CMD
 1200    1 nginx: master
```

---

## 4.3 PID：进程的身份证号

**一句话定义**：PID（Process ID）是系统为每个进程分配的唯一正整数，从1开始。

```bash
# 1. PID范围
$ cat /proc/sys/kernel/pid_max
4194304
# 最多约420万个PID
# 32位系统：32768
# 64位系统：默认4194304

# 2. PID 1 = init（systemd）
$ ps -p 1 -o pid,cmd
    PID CMD
      1 /usr/lib/systemd/systemd --system --deserialize 22

# 3. 查看自己的PID
$ echo $$
1234
# 当前shell的PID

# 4. 按PID操作进程
$ kill 1234               # 发送SIGTERM
$ kill -9 1234            # 发送SIGKILL
$ renice -n 5 -p 1234     # 调整优先级
$ cat /proc/1234/status   # 查看状态
```

### PID复用

```bash
# PID不会立即复用
# 但系统耗尽PID后，会从最低未用号开始
# 进程频繁创建-退出会快速消耗PID

# 查看PID使用情况
$ ls /proc | grep -E '^[0-9]+$' | sort -n | tail
# 或
$ ps -e --no-headers | awk '{print $1}' | sort -n | tail
```

---

## 4.4 PPID：父进程关系

**一句话定义**：PPID（Parent PID）是创建当前进程的父进程ID，所有进程形成树状结构，根是init（PID=1）。

```bash
# 1. 查看进程的PPID
$ ps -o pid,ppid,cmd -p 1234
  PID  PPID CMD
 1234  1100 /usr/bin/python3 app.py
#                 ↑ PPID是cron(1100)

# 2. 用pstree看完整层级
$ pstree -p | head
systemd(1)─┬─NetworkManager(789)
           ├─cron(1100)───sh(1101)───python3(1234)
           ├─sshd(1056)
           └─nginx(1200)─┬─nginx(1201)
                          ├─nginx(1202)
                          └─nginx(1203)

# 3. 找进程的祖先链
$ pstree -sp 1234
systemd,cron,sh,python3
# 从init到当前进程的完整路径
```

### 实战：找僵尸进程的父进程

```bash
# 1. 找僵尸进程
$ ps -eo pid,ppid,stat,cmd | awk '$3~/^Z/'
  PID  PPID STAT CMD
 5678  5000 Z    [defunct] <defunct>
# Z表示僵尸

# 2. 找父进程
$ ps -p 5000 -o pid,ppid,cmd
  PID  PPID CMD
 5000     1 /usr/bin/parent_app
# 父进程是5000，PPID是1（被init收养？不对，是parent_app）

# 3. 看父进程状态
$ cat /proc/5000/status | grep State
State:  S (sleeping)
# 父进程在sleep，没wait子进程

# 4. 解决
# - 让父进程wait：通常重启父进程
# - kill父进程：僵尸会被init收养并清理
$ kill 5000                # 优雅退出
$ kill -9 5000             # 强制
```

---

## 4.5 僵尸进程（Zombie）：已死未埋

**一句话定义**：僵尸进程是已终止但未被父进程`wait()`回收的进程，仍占PID资源但不占CPU/内存。

### 产生原因

```
进程终止
  ↓
内核发送SIGCHLD给父进程
  ↓
父进程没装SIGCHLD处理，且没wait()
  ↓
内核保留进程的task_struct（包括PID、退出状态）
  ↓
进程变成僵尸
```

### 排查与解决

```bash
# 1. 找僵尸
$ ps aux | grep 'Z'
USER  PID   %CPU %MEM VSZ   RSS  TTY  STAT  START  TIME  COMMAND
root  5678  0.0  0.0    0    0   ?    Z     10:00  0:00  [python3] <defunct>

# 2. 看父进程
$ ps -o pid,ppid,stat,cmd -p 5678
  PID  PPID STAT CMD
 5678  5000 Z    [python3] <defunct>
# 父进程是5000

# 3. 看父进程行为
$ cat /proc/5000/wchan
do_wait
# 父进程正在wait子进程（奇怪）

$ cat /proc/5000/status | grep -E 'State|Threads'
State:  S (sleeping)
Threads: 1
# 单线程，正常运行

# 4. 解决
# a. 给父进程发SIGCHLD，让它处理（最优雅）
$ kill -SIGCHLD 5000

# b. 父进程如果是bug，修复代码加signal handler

# c. 杀父进程（最后手段）
$ kill 5000
# 僵尸进程会被init(PID=1)收养并清理
```

### 避免产生僵尸

```python
# Python示例：父进程处理SIGCHLD
import signal
import os

def reap_zombies(signum, frame):
    while True:
        try:
            pid, status = os.waitpid(-1, os.WNOHANG)
            if pid == 0:
                break
            print(f"Reaped {pid}")
        except ChildProcessError:
            break

signal.signal(signal.SIGCHLD, reap_zombies)
```

```c
// C示例
struct sigaction sa;
sa.sa_handler = sigchld_handler;
sa.sa_flags = SA_RESTART | SA_NOCLDSTOP;
sigemptyset(&sa.sa_mask);
sigaction(SIGCHLD, &sa, NULL);
```

---

## 4.6 孤儿进程：被init收养的娃

**一句话定义**：孤儿进程是父进程先于子进程终止的进程，会被init（PID=1）自动收养。

```bash
# 1. 演示
$ cat > orphan.c <<'EOF'
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

int main() {
    pid_t pid = fork();
    if (pid == 0) {
        // 子进程
        sleep(5);
        printf("子进程PID=%d, PPID=%d\n", getpid(), getppid());
        return 0;
    } else {
        // 父进程立即退出
        printf("父进程退出，子进程PID=%d\n", pid);
        return 0;
    }
}
EOF
$ gcc orphan.c -o orphan
$ ./orphan
父进程退出，子进程PID=1234
$ 等待5秒...
子进程PID=1234, PPID=1      # ← PPID=1，被init收养

# 2. 孤儿进程无害
# - 不会变僵尸（init会wait）
# - 仍然正常运行
# - 在长生命周期进程里常见（如daemon）
```

### 实战：避免意外孤儿

```bash
# 场景：长运行的Web服务，每个请求启动子进程处理
# 问题：父进程重启时，正在处理的子进程变孤儿

# 解决：用进程组/PID文件跟踪子进程
$ cat /etc/systemd/system/myapp.service
[Service]
Type=notify
NotifyAccess=main
# systemd会跟踪主进程+子进程
# 主进程重启前会先kill子进程
```

---

## 4.7 ps：进程状态查看器

**一句话定义**：`ps`（process status）是查看进程信息的"瑞士军刀"。

### 常用选项组合

```bash
# 1. ps aux - BSD风格（最常用）
$ ps aux
USER  PID   %CPU %MEM VSZ    RSS    TTY  STAT  START  TIME  COMMAND
root  1     0.0  0.0  169548  11880  ?    Ss    09:00  0:01  /sbin/init
root  789   0.0  0.1 654320  12300  ?    Ssl   09:00  0:05  /usr/sbin/...
# a = 所有进程（含其他用户）
# u = 用户友好格式
# x = 含无终端进程

# 2. ps -ef - SystemV风格
$ ps -ef
UID   PID  PPID  C STIME TTY  TIME     CMD
root  1    0     0 09:00  ?   00:00:01 /sbin/init
# e = 所有进程
# f = 完整格式

# 3. 自定义列
$ ps -eo pid,ppid,pcpu,pmem,rss,vsz,stat,start,cmd --sort=-pcpu | head
  PID  PPID %CPU %MEM   RSS    VSZ STAT  STARTED CMD
 1234     1 25.5  5.0 200000 500000  Sl   10:00 python3 heavy_job.py
 5678  5000 15.2  2.0 80000  300000  R    10:05 ./data_processor

# 4. 按进程名过滤
$ ps -C nginx
  PID TTY          STAT   TIME COMMAND
 1200 ?            Ss     0:00 nginx: master process
 1201 ?            S      0:01 nginx: worker process

# 5. 按用户
$ ps -u alice
$ ps -U root -u root          # 真实UID是root，有效UID也是root

# 6. 按进程树
$ ps -e --forest
$ ps axjf                     # 树状

# 7. 按线程
$ ps -L -p 1234               # 进程的所有线程
  PID   LWP  C NLWP STIME TTY  TIME     CMD
 1234  1234  0    4 10:00  ?   00:00:00 python3 app.py
 1234  1235  0    4 10:00  ?   00:00:00 python3 app.py
#  LWP = Light Weight Process = 线程

# 8. 找进程的PID（多种方式）
$ pgrep nginx                 # 命令名匹配
$ pidof nginx                 # 全路径匹配
$ ps -C nginx -o pid=         # ps方式
```

### 进程状态码

```bash
$ ps -eo pid,stat,cmd
  PID STAT CMD
 1234 Sl   python3 app.py
 5678 R+   ./data_processor
 9012 Ss   nginx
# 第一个字符：R=运行 S=睡眠 D=不可中断 Z=僵尸 T=停止 I=空闲
# 第二个字符：< = 高优先级 N = 低优先级 s = 会话首进程 l = 多线程
# 第三个字符：+ = 前台进程组
```

| 状态 | 含义 |
|------|------|
| `R` | Running，运行或就绪 |
| `S` | Sleeping，可中断睡眠（等待事件） |
| `D` | Disk sleep，不可中断睡眠（通常在等IO） |
| `Z` | Zombie，僵尸 |
| `T` | Stopped，停止（SIGSTOP） |
| `t` | Tracing，被调试器暂停 |
| `I` | Idle，内核空闲进程 |
| `X` | Dead，已死 |
| `K` | Wakekill，wakeup kill |
| `W` | Waking，唤醒中 |
| `P` | Parked，parked |

### 实战：ps高级过滤

```bash
# 1. 找CPU占用最高的进程
$ ps -eo pid,ppid,pcpu,pmem,cmd --sort=-pcpu | head -10

# 2. 找内存占用最高
$ ps -eo pid,ppid,pcpu,pmem,rss,cmd --sort=-rss | head

# 3. 找状态异常的进程
$ ps -eo pid,stat,cmd | awk '$2~/[ZTDI]/'   # 异常状态

# 4. 找父进程是1的（已无父的孤儿）
$ ps -eo pid,ppid,cmd | awk '$2==1' | head

# 5. 找耗时最长的
$ ps -eo pid,etime,cmd --sort=-etime | head

# 6. 找子进程最多的进程
$ ps -eo pid,ppid,cmd | awk '{a[$2]++} END {for (k in a) print a[k], k}' | sort -rn | head

# 7. 找PHP-FPM worker数量
$ ps -C php-fpm --no-headers | wc -l

# 8. 找所有用户的进程
$ ps -eo user,pid,cmd | awk '$1=="alice"'
```

---

## 4.8 top / htop / btop：实时监控

### top：经典工具

```bash
# 1. 启动
$ top
top - 10:00:00 up  1:00,  3 users,  load average: 0.50, 0.40, 0.30
Tasks: 234 total,   1 running, 233 sleeping
%Cpu(s):  5.0 us,  2.0 sy,  0.0 ni, 92.0 id,  0.5 wa,  0.0 hi,  0.5 si
MiB Mem :   7890.0 total,   4500.0 free,   1200.0 used,   2190.0 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   6000.0 avail Mem

  PID USER      PR  NI    VIRT    RES  SHR  S  %CPU  %MEM     TIME+ COMMAND
 1234 alice     20   0  500000 200000  10000 R  25.5   5.0   0:30.00 python3
 5678 root      20   0  300000  80000   5000 S  15.2   2.0   0:15.00 nginx
 ...
```

**关键快捷键**：
```
P - 按CPU排序（默认）
M - 按内存排序
T - 按时间排序
k - kill进程（输入PID）
r - renice（调整优先级）
1 - 显示每个CPU核心
H - 显示线程
c - 显示完整命令行
u - 按用户过滤
V - 树状显示
f - 字段管理
W - 保存配置
q - 退出
```

**top输出解读**：
- `load average: 0.50 0.40 0.30` - 1/5/15分钟平均负载
  - 1.0 = 满载（1个CPU）
  - 4核CPU的"健康"上限是4.0
  - 持续 > CPU数 = 过载
- `%Cpu(s): 5.0 us` - 用户态CPU
  - `us` 用户态、`sy` 内核态、`ni` nice进程、`id` 空闲、`wa` IO等待
  - `wa` 高 = 磁盘瓶颈
- `MiB Mem: ... buff/cache` - 缓冲缓存，可回收

```bash
# top批量模式（脚本用）
$ top -bn1 -p 1234
# -b 批量模式（不进入交互）
# -n 1 只采样1次
# -p 指定PID

# 监控特定进程
$ top -p $(pgrep -d',' nginx)
# 监控所有nginx进程
```

### htop：top的增强版

```bash
# 安装
$ sudo apt install htop

# 启动
$ htop
# 优势：
# - 颜色高亮
# - 鼠标操作
# - 进程树视图（F5）
# - 搜索/过滤（F3/F4）
# - 直接strace/lsof（F2设置）
# - 杀进程F9
# - 调整nice F7/F8
```

### btop：最漂亮的监控工具

```bash
# 安装
$ sudo apt install btop
# 或
$ sudo snap install btop

# 启动
$ btop
# 优势：
# - 全屏ASCII图形（CPU/内存/网络/磁盘）
# - 鼠标操作
# - 主题切换
# - 详细进程列表
# - 内置iotop/iftop/NetHops功能
```

### atop：长期数据收集

```bash
# 安装
$ sudo apt install atop

# 启动
$ atop
# 优势：
# - 默认每10分钟记录一次快照
# - 保留30天历史
# - 可回看"当时为什么慢了"
# - 支持CGroup级别分析

# 配置文件
$ cat /etc/default/atop
INTERVAL=60                     # 采样间隔(秒)
LOGPATH="/var/log/atop"
LOGGENERATIONS=30
```

---

## 4.9 信号（Signal）：进程间通信的基础

**一句话定义**：信号是Linux进程间异步通信机制，1-64个信号用于通知进程发生特定事件（如Ctrl+C = SIGINT）。

### 常用信号速查（详见`kill -l`）

| 信号 | 编号 | 默认动作 | 用途 |
|------|------|----------|------|
| **SIGHUP** | 1 | 终止 | 终端挂起 / 守护进程重载配置 |
| **SIGINT** | 2 | 终止 | Ctrl+C |
| **SIGQUIT** | 3 | Core | Ctrl+\\ |
| **SIGKILL** | 9 | 终止 | **强制杀（不可捕获）** |
| **SIGTERM** | 15 | 终止 | 优雅终止（kill默认） |
| **SIGCHLD** | 17 | 忽略 | 子进程退出 |
| **SIGCONT** | 18 | 继续 | 恢复暂停 |
| **SIGSTOP** | 19 | 暂停 | **强制暂停（不可捕获）** |
| **SIGTSTP** | 20 | 暂停 | Ctrl+Z |
| **SIGUSR1** | 10 | 终止 | 用户自定义1 |
| **SIGUSR2** | 12 | 终止 | 用户自定义2 |
| **SIGPIPE** | 13 | 终止 | 管道破裂 |
| **SIGALRM** | 14 | 终止 | alarm()超时 |
| **SIGSEGV** | 11 | Core | 段错误 |

### 实战：信号处理

```python
# Python：优雅处理SIGTERM
import signal
import sys

def graceful_exit(signum, frame):
    print(f"Received {signum}, cleaning up...")
    # 关闭连接、刷盘、释放资源
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_exit)
signal.signal(signal.SIGINT, graceful_exit)
signal.signal(signal.SIGHUP, lambda s, f: reload_config())  # 重载配置

while True:
    do_work()
```

```c
// C：相同逻辑
void handler(int signum) {
    printf("Got %d\n", signum);
    cleanup();
    exit(0);
}

int main() {
    signal(SIGTERM, handler);
    signal(SIGINT, handler);
    while (1) {
        // ...
    }
}
```

### 关键原则

```bash
# 1. 永远先SIGTERM（15），再SIGKILL（9）
$ kill 1234                    # 优雅退出（给进程清理机会）
$ sleep 5                      # 等5秒
$ kill -0 1234 && kill -9 1234 # 如果还在，强制

# 2. 重要服务的"reload"通常用SIGHUP
$ sudo nginx -s reload         # 内部发SIGHUP给master
$ sudo systemctl reload nginx  # 同上
$ kill -HUP $(cat /run/nginx.pid)

# 3. SIGKILL的代价
# - 数据库：可能导致数据损坏
# - 消息队列：消息丢失
# - 事务：未提交回滚
# - 文件写入：部分写入
# 永远作为最后手段

# 4. SIGSTOP暂停的应用
# - 调试：用gdb attach后SIGSTOP，再检查状态
# - CPU：让进程暂停以"踩刹车"
# - 临时：避免OOM
```

---

## 4.10 kill 与 killall / pkill

### kill：发送信号

```bash
# 1. 默认发SIGTERM
$ kill 1234

# 2. 发具体信号
$ kill -9 1234                 # SIGKILL
$ kill -15 1234                # SIGTERM（默认）
$ kill -HUP 1234               # SIGHUP（重载配置）
$ kill -USR1 1234              # 自定义信号

# 3. 名字发送（杀进程组）
$ kill -USR1 -1234             # 杀进程组（负号=组ID）
# 整个进程组都会收到信号

# 4. 同时杀多个
$ kill 1234 5678 9012

# 5. 检查信号是否成功
$ kill -0 1234                 # 0号信号不实际发送，只检查进程存在
$ echo $?
0                              # 存在
$ kill -0 9999
$ echo $?
1                              # 不存在
```

### killall：按名字杀

```bash
# 1. 安装
$ sudo apt install psmisc

# 2. 用法
$ killall nginx                # 杀所有nginx进程
$ killall -9 nginx             # 强制
$ killall -HUP nginx           # 重载所有nginx
$ killall -u alice python3     # 杀alice的所有python3
$ killall -r '^python'         # 正则匹配

# 3. 实战：重启某个服务
$ sudo killall -HUP sshd       # sshd重载配置
$ sudo killall -9 uwsgi        # 强制重启uwsgi
```

### pkill：高级过滤

```bash
# 1. 按进程名
$ pkill nginx
$ pkill -9 python

# 2. 按用户
$ pkill -u alice
$ pkill -u alice python3

# 3. 按终端
$ pkill -t pts/0               # 杀pts/0的进程

# 4. 按命令行
$ pkill -f "python.*app.py"   # 匹配完整命令行
$ pkill -f "uwsgi"             # 包含uwsgi字样的命令行

# 5. 按cgroup
$ pkill -P 1234                # 杀PID 1234的所有子进程
$ pkill --pgroup 1234

# 6. 按其他属性
$ pkill -G devteam             # 按进程组ID
$ pkill -s 9 -f "python app"  # 发SIGKILL给匹配的命令行

# 7. 实战：杀所有用户进程
$ pkill -u guest
# 用户guest的所有进程被清理
```

### pgrep：按名字查PID

```bash
# 1. 基础
$ pgrep nginx
1200
1201
1202

# 2. 多列
$ pgrep -l nginx
1200 nginx
1201 nginx
1202 nginx

# 3. 详细信息
$ pgrep -a nginx
1200 nginx: master process /usr/sbin/nginx -g 'daemon on; master_process on;'
1201 nginx: worker process

# 4. 最新创建的
$ pgrep -n nginx                # newest
# 最老的
$ pgrep -o nginx                # oldest

# 5. 统计数量
$ pgrep -c nginx
3
```

---

## 4.11 nice / renice：进程优先级

**一句话定义**：nice值是进程的"礼貌程度"，值越高越"谦让"（优先级越低）；值越低越"贪婪"（优先级越高）。

### 取值范围

```bash
# nice值：-20 到 19
# -20 = 最高优先级
#  0  = 默认
#  19 = 最低优先级

# 优先级计算：PR = 20 + NI
# 所以：PR 0 (NI=-20) 到 PR 39 (NI=19)
```

### nice：启动时设优先级

```bash
# 1. 启动低优先级任务（备份/压缩等）
$ nice -n 19 tar czf backup.tar.gz /data
# nice值19，最低优先级

# 2. 启动高优先级任务（需root）
$ sudo nice -n -20 ffmpeg -i input.mkv output.mkv
# nice值-20，最高优先级

# 3. 普通用户只能调高nice（0-19）
$ nice -n 10 long_task
# 范围0-19
$ nice -n -5 long_task
nice: cannot set niceness: Permission denied
```

### renice：调整运行中进程

```bash
# 1. 调整PID
$ renice -n 10 -p 1234
1234 (process ID) old priority 0, new priority 10

# 2. 调整进程组
$ renice -n 5 -g 1234
# 整个进程组的nice都设为5

# 3. 按用户
$ sudo renice -n 10 -u alice
# alice的所有进程nice=10

# 4. 实战
# - 跑批任务降级
$ sudo renice -n 19 -p $(pgrep batch_job)

# - 数据库服务升级（需root）
$ sudo renice -n -5 -p $(pgrep mysqld)

# - 监控某个用户
$ ps -o pid,nice,cmd -u alice
```

### 实战：nice的合理使用

```bash
# 场景1：服务器跑批时影响前端
# 1. 前端优先级保持默认
# 2. 跑批任务nice=19

$ cat /etc/cron.daily/backup.sh
#!/bin/bash
nice -n 19 tar czf /backup/$(date +\%F).tar.gz /var/lib/mysql
nice -n 19 rsync -a /data/ /backup/data/

# 场景2：编译时不影响开发
$ nice -n 19 make -j$(nproc)
# 并行编译，但优先级最低

# 场景3：CI/CD构建
$ sudo systemd-run --nice=19 ./build.sh
# systemd风格的nice
```

---

## 4.12 前台/后台作业（Jobs）

**一句话定义**：作业是Shell的逻辑单位，单个命令或管道；前台作业占用终端，后台作业不占用。

### 基础操作

```bash
# 1. & 后台运行
$ long_task &
[1] 1234                    # 作业号1，PID 1234

# 2. jobs - 查看当前shell的后台作业
$ jobs
[1]+ Running   long_task &
[2]- Stopped  vim file.txt

# 3. fg - 前台
$ fg %1                      # 把作业1提到前台
# 或
$ fg                         # 把当前作业（+标记）提到前台

# 4. bg - 后台运行
$ bg %1                      # 让已停止的作业1继续后台运行

# 5. Ctrl+Z - 暂停当前前台作业
$ long_task
^Z
[1]+  Stopped   long_task

# 6. Ctrl+C - 终止前台作业

# 7. kill %1 - 杀后台作业
$ kill %1
# 作业号%开头
```

### 实战

```bash
# 1. 启动3个后台任务
$ (for i in 1 2 3 4 5; do echo $i; sleep 1; done) &
$ (while true; do date; sleep 2; done) &
$ (yes > /dev/null) &
$ jobs
[1]   Done    (for i in 1 2 3 4 5; do echo $i; sleep 1; done)
[2]-  Running  (while true; do date; sleep 2; done) &
[3]+  Running  (yes > /dev/null) &

# 2. 中间查看输出
$ fg %2                      # 提到前台看date
# Ctrl+Z 暂停
$ bg %2                      # 继续后台

# 3. 杀某个
$ kill %3                    # 杀yes进程

# 4. 全部等待
$ wait                       # 等所有后台完成
[2]-  Done    (while true; do date; sleep 2; done)
```

### 进程组与会话

```bash
# 1. 查看进程所属会话
$ ps -o pid,sid,pgid,cmd -p $$
  PID  SID PGID CMD
 2345 2345 2345 bash
# 当前shell是会话首进程，进程组ID=PID

# 2. 后台进程与shell的关系
$ long_task &
$ ps -o pid,ppid,pgid,sid,cmd -p $(pgrep long_task)
  PID  PPID PGID  SID CMD
 1234  2345 1234 2345 long_task
# 父进程是shell(PID 2345)
# 进程组ID=自己的PID（独立组）
# 会话ID=shell的SID

# 3. 关闭shell时发生了什么？
# - shell发送SIGHUP给所有子进程
# - 没nohup/setsid的进程会死
# - 控制终端关闭，所有前台进程收到SIGHUP
```

---

## 4.13 nohup：免疫SIGHUP

**一句话定义**：`nohup`让进程忽略SIGHUP信号，关闭终端不退出。

```bash
# 1. 基本用法
$ nohup long_task &
[1] 1234
$ nohup: ignoring input and appending output to 'nohup.out'
# 默认把stdout/stderr重定向到nohup.out

# 2. 重定向到指定文件
$ nohup long_task > /var/log/task.log 2>&1 &
# 1+2 合并到指定文件

# 3. 与&的区别
$ long_task &               # 后台，但关shell会SIGHUP
$ nohup long_task &         # 后台，免疫SIGHUP

# 4. 与disown的区别
$ long_task &
$ disown                    # 从job表移除，shell退出不杀
# disown不忽略SIGHUP，只是不让shell追踪

# 5. 完整组合
$ nohup long_task > /var/log/task.log 2>&1 &
$ disown
# 完全脱离shell
```

### 实战：长期运行任务

```bash
# 1. 数据库备份（不因关shell中断）
$ nohup pg_dump -h db-server -U backup dbname > /backup/db-$(date +%F).sql 2>/tmp/db-err.log &
$ disown
# 退出shell也继续

# 2. 远程同步（防网络断）
$ nohup rsync -avz -e ssh /data/ backup@nas:/backup/ > /tmp/rsync.log 2>&1 &
$ disown

# 3. 训练任务（几天不间断）
$ nohup python3 train.py --epochs 100 > train.log 2>&1 &
$ disown
$ exit                       # 关掉SSH，训练继续
```

### 为什么systemd不再用nohup？

```bash
# systemd service比nohup更靠谱
# 1. 自动重启
# 2. 日志集成（journald）
# 3. 资源限制（cgroups）
# 4. 启动顺序（依赖管理）
# 5. 标准化的配置

# 长期服务用systemd
$ cat /etc/systemd/system/longtask.service
[Service]
ExecStart=/usr/local/bin/long_task
Restart=always
RestartSec=10

# 一次性任务用nohup或tmux
```

---

## 4.14 screen / tmux：终端复用器

**一句话定义**：终端复用器允许多个"伪终端"在单个连接中运行，断线重连不断作业。

### tmux（现代推荐）

```bash
# 1. 安装
$ sudo apt install tmux

# 2. 启动新会话
$ tmux new -s dev               # 创建名为"dev"的会话
# 底部出现绿条表示在tmux中

# 3. 退出会话（会话在后台运行）
Ctrl+B, d                       # Prefix d (detach)

# 4. 查看会话
$ tmux ls
dev: 1 windows (created Wed Jul  2 10:00:00 2026)

# 5. 重连会话
$ tmux attach -t dev

# 6. 在会话中新建窗口
Ctrl+B, c                       # create window

# 7. 切换窗口
Ctrl+B, 0/1/2...                # 切到第N个窗口
Ctrl+B, w                       # 列表选择

# 8. 分割窗口
Ctrl+B, %                       # 垂直分割
Ctrl+B, "                       # 水平分割
Ctrl+B, 方向键                    # 切换窗格
Ctrl+B, x                       # 关闭窗格

# 9. 滚动模式
Ctrl+B, [                       # 进入
# PgUp/PgDn 滚动
# q 退出
```

### 实战：tmux + 长期任务

```bash
# 1. 开始长任务
$ tmux new -s train
# 在tmux里：
$ python3 train.py --epochs 100
# Ctrl+B, d 退出（训练继续）
$ tmux attach -t train  # 随时回来查看

# 2. 关掉SSH再回来
$ ssh user@server
$ tmux attach -t train
# 训练继续，输出在

# 3. 多任务
$ tmux new -s dev -d           # 后台启动
$ tmux send-keys -t dev 'vim file.py' C-m
# 发送命令到dev会话
$ tmux send-keys -t dev 'python3 server.py' C-m

# 4. 全部会话一键恢复
$ tmux new -s monitor
$ while true; do date; sleep 5; done
Ctrl+B, d
# 重连后还在
```

### tmux配置文件

```bash
# ~/.tmux.conf
# Prefix改为Ctrl+A（更顺手）
unbind C-b
set -g prefix C-a
bind C-a send-prefix

# 鼠标支持
set -g mouse on

# 256色
set -g default-terminal "screen-256color"

# 状态栏
set -g status-bg green
set -g status-fg black
set -g status-interval 60
set -g status-left-length 30
set -g status-left "#S:#[fg=cyan]#[bg=black] #W"
set -g status-right "#[fg=cyan]%H:%M"

# 重新加载
$ tmux source-file ~/.tmux.conf
# 或在tmux中：Ctrl+B, :source-file ~/.tmux.conf
```

### screen（经典备选）

```bash
# 1. 安装
$ sudo apt install screen

# 2. 启动
$ screen -S dev                 # 创建会话
# Ctrl+A, d 退出
# screen -r dev 重连

# 3. 窗口
Ctrl+A, c                       # 新建
Ctrl+A, n/p                     # 下/上一个
Ctrl+A, "                       # 列表

# 4. 共享会话（两人都看同一屏）
$ screen -S shared
# 别人：
$ screen -x shared              # -x 是附加而非创建
```

### 实战：tmux + SSH保活

```bash
# 场景：SSH连接不稳定，tmux保住作业
# 1. SSH连上后先开tmux
$ tmux new -s main

# 2. 任何工作都在tmux里做
# 网络断了？再连上
$ tmux attach -t main
# 工作继续，输出在

# 3. 关键配置（~/.ssh/config）
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
# 每60秒发keepalive，超时3次才断

# 4. 自动恢复tmux
$ cat ~/.bashrc
if [[ -z "$TMUX" ]] && [ -n "$SSH_CONNECTION" ]; then
    tmux attach -t main || tmux new -s main
fi
# SSH登录后自动连到main会话
```

---

## 本章小结

### 进程管理决策树

```
需要查进程？
├─ 一次性 → ps
├─ 实时 → top/htop/btop
├─ 长期数据 → atop
└─ 找特定 → pgrep

需要杀进程？
├─ 优雅 → kill -15 / killall
├─ 强制 → kill -9
└─ 过滤 → pkill

需要后台运行？
├─ 短期 → &
├─ 长期 → nohup + disown
├─ 调试 → tmux
└─ 服务 → systemd
```

### 必备快捷键

| 场景 | 命令 |
|------|------|
| 启动后台 | `cmd &` |
| 暂停前台 | `Ctrl+Z` |
| 继续后台 | `bg` |
| 提到前台 | `fg` |
| 看作业 | `jobs` |
| 等所有 | `wait` |
| 杀当前 | `Ctrl+C` |
| 退出shell | `exit` |

### 应急流程

```bash
# 进程卡住
$ ps -o pid,stat,wchan,cmd -p <pid>
# 看wchan知道在等什么

# CPU占满
$ top -bn1 | head -20
$ ps -eo pid,pcpu,cmd --sort=-pcpu | head
# 找到凶手

# 内存泄漏
$ ps -eo pid,pmem,rss,cmd --sort=-rss | head
# 持续观察RSS增长

# 杀不掉（僵尸）
$ ps -eo pid,ppid,stat,cmd | awk '$3~/Z/'
# 杀父进程
```

### 下一章预告

**第五章：网络基础与配置** — IP/子网掩码、TCP/UDP四层模型、DNS解析、netstat/ss、iptables/nftables、curl/wget、网络调试（ping/traceroute/mtr）。

---

> **本章字数**：约 18,800 字
> **涉及命令**：ps, top, htop, btop, kill, killall, pkill, pgrep, nice, renice, nohup, tmux, screen
> **基准版本**：procps-ng 4.0, tmux 3.4, htop 3.3, btop 1.4

# 第五章：网络基础与配置

> **本章定位**：网络是Linux服务器的命脉。从IP子网到TCP握手，从ss/netstat到防火墙规则，本章覆盖运维必须掌握的网络诊断和配置技能。

---

## 5.1 IP地址与子网掩码

**一句话定义**：IP地址是网络设备的"门牌号"，子网掩码定义了"哪个范围是邻居、哪个范围要出门找路由"。

### IPv4 vs IPv6

```bash
# IPv4：32位，约43亿地址
$ ip addr show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP
    inet 192.168.1.100/24 brd 192.168.1.255 scope global eth0
         ↑           ↑        ↑
         本机IPv4     网络前缀  广播地址
    inet6 fe80::216:3eff:fe5a:7b1c/64 scope link
         ↑
         本地链路IPv6

# 关键字段：
# /24 = 子网掩码 255.255.255.0
# brd = 广播地址
# scope global = 全球可路由
```

### 子网掩码速查

| CIDR | 子网掩码 | 可用地址数 | 典型场景 |
|------|----------|-----------|----------|
| `/8` | 255.0.0.0 | 16,777,214 | 大型ISP |
| `/16` | 255.255.0.0 | 65,534 | 大型企业 |
| `/24` | 255.255.255.0 | 254 | 标准子网 |
| `/25` | 255.255.255.128 | 126 | 中型子网 |
| `/28` | 255.255.255.240 | 14 | 小型子网 |
| `/30` | 255.255.255.252 | 2 | 点对点链路 |
| `/32` | 255.255.255.255 | 1 | 单主机 |

### 理解CIDR

```bash
# 192.168.1.100/24
# 网络部分：192.168.1    (前24位)
# 主机部分：.100          (后8位)
# 网络地址：192.168.1.0   (主机部分全0)
# 广播地址：192.168.1.255 (主机部分全1)
# 可用范围：192.168.1.1 - 192.168.1.254

# 手动验证
$ ipcalc 192.168.1.100/24
Address:   192.168.1.100   
Netmask:   255.255.255.0 = 24
Wildcard:  0.0.0.255      
Network:   192.168.1.0/24  
HostMin:   192.168.1.1     
HostMax:   192.168.1.254   
Broadcast: 192.168.1.255   
Hosts/Net: 254             
```

### 实战：IP配置

```bash
# 1. 查看所有接口IP
$ ip addr show
# 或
$ ip a

# 2. 临时添加IP
$ sudo ip addr add 192.168.1.200/24 dev eth0

# 3. 删除IP
$ sudo ip addr del 192.168.1.200/24 dev eth0

# 4. 永久配置（netplan方式，Ubuntu 18.04+）
$ cat /etc/netplan/00-installer-config.yaml
network:
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
  version: 2

$ sudo netplan apply

# 5. 永久配置（NetworkManager，主流桌面）
$ nmcli con show                  # 列出连接
$ nmcli con mod "Wired connection 1" ipv4.addresses 192.168.1.100/24
$ nmcli con mod "Wired connection 1" ipv4.gateway 192.168.1.1
$ nmcli con mod "Wired connection 1" ipv4.dns "8.8.8.8 1.1.1.1"
$ nmcli con mod "Wired connection 1" ipv4.method manual
$ nmcli con up "Wired connection 1"

# 6. 查看路由表
$ ip route show
default via 192.168.1.1 dev eth0 proto static   # 默认网关
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.100
```

---

## 5.2 TCP/UDP：传输层双雄

**一句话定义**：TCP面向连接、可靠；UDP无连接、快速。理解它们的区别是网络编程和故障排查的基础。

### 对比表

| 维度 | TCP | UDP |
|------|-----|-----|
| 连接 | 需要三次握手 | 无连接 |
| 可靠性 | 确认+重传+顺序 | 无 |
| 速度 | 慢（有开销） | 快 |
| 流控 | 拥塞控制 | 无 |
| 头部大小 | 20字节 | 8字节 |
| 场景 | HTTP、SSH、MySQL | DNS、视频流、游戏 |

### TCP状态机（运维必需）

```bash
# 查看TCP状态统计
$ ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
   1234 ESTAB
    567 LISTEN
     89 TIME-WAIT
     12 SYN-SENT
      3 CLOSE-WAIT
      1 FIN-WAIT-2

# 关键状态
# LISTEN      — 服务端等待连接
# ESTABLISHED — 连接已建立，正常通信
# TIME_WAIT   — 主动关闭方等待2MSL（2x最大段生命周期）
# CLOSE_WAIT  — 被动关闭方等待应用关闭（⚠ 如果多说明应用bug）
# FIN_WAIT_2  — 主动关闭方等待对方FIN
# SYN_SENT    — 客户端已发SYN，等待SYN-ACK

# TIME_WAIT过多（>10000）的问题
$ sysctl net.ipv4.tcp_fin_timeout
net.ipv4.tcp_fin_timeout = 60           # 默认60秒
# 调低：sysctl -w net.ipv4.tcp_fin_timeout=30
# 启用复用：sysctl -w net.ipv4.tcp_tw_reuse=1
```

### 实战：用tcpdump看三次握手

```bash
# 三次握手
$ sudo tcpdump -i eth0 -nn 'tcp and host 93.184.216.34' | grep -E 'Flags'
# 1. 客户端 -> 服务器: SYN
12:00:00.100 IP 192.168.1.100.54321 > 93.184.216.34.80: Flags [S], seq 123456789
# 2. 服务器 -> 客户端: SYN-ACK
12:00:00.200 IP 93.184.216.34.80 > 192.168.1.100.54321: Flags [S.], seq 987654321, ack 123456790
# 3. 客户端 -> 服务器: ACK
12:00:00.201 IP 192.168.1.100.54321 > 93.184.216.34.80: Flags [.], ack 987654322

# 四次挥手（关闭连接）
# 1. 客户端 -> 服务器: FIN
# 2. 服务器 -> 客户端: ACK
# 3. 服务器 -> 客户端: FIN
# 4. 客户端 -> 服务器: ACK
# → 客户端进入TIME_WAIT 2MSL（≈60秒）
```

### UDP实战

```bash
# DNS查询就是UDP的经典场景
$ dig +short google.com @8.8.8.8
142.250.80.46
# 一个UDP包发过去，一个UDP包回来，极快

# UDP没有"连接"，所以：
# - 没有ESTABLISHED状态
# - 不保证对方收到
# - 应用层需要自己处理丢包（如DNS重试）
```

---

## 5.3 ss / netstat：连接查看器

**一句话定义**：`ss`（socket statistics）是现代Linux查看网络连接的工具，比`netstat`快10倍（读取内核直接数据结构，无需遍历/proc）。

### ss常用组合

```bash
# 1. 所有TCP连接（最常用）
$ ss -tan
State    Recv-Q  Send-Q  Local Address:Port   Peer Address:Port
LISTEN   0       128     0.0.0.0:22           0.0.0.0:*
LISTEN   0       511     0.0.0.0:80           0.0.0.0:*
ESTAB    0       0       192.168.1.100:22     10.0.0.1:54321
# -t TCP  -a 所有  -n 数字格式

# 2. 监听端口
$ ss -tlnp
State   Recv-Q Send-Q Local Address:Port  Peer Address:Port  Process
LISTEN  0      128    0.0.0.0:22          0.0.0.0:*          sshd(pid=1056)
LISTEN  0      511    0.0.0.0:80          0.0.0.0:*          nginx(pid=1200)
LISTEN  0      128    127.0.0.1:3306      0.0.0.0:*          mysqld(pid=3000)
# -p 显示进程

# 3. 查看特定端口
$ ss -tlnp sport = :80
$ ss -tlnp sport = :443

# 4. 查看特定进程
$ ss -tnp | grep sshd

# 5. 统计连接数
$ ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
   1234 ESTAB
    567 LISTEN
     89 TIME-WAIT

# 6. 按状态过滤
$ ss -tan state established
$ ss -tan state time-wait
$ ss -tan state listening

# 7. 每个源IP的连接数（找异常）
$ ss -tan state established | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -10

# 8. 实时监控
$ watch -n 1 'ss -tan | head -20'
```

### ss vs netstat

| 维度 | ss | netstat |
|------|-----|---------|
| 速度 | 很快（内核直接读） | 慢（遍历/proc） |
| 输出 | 简洁 | 详细 |
| 推荐 | ✅ 现代标准 | ❌ 已淘汰但还在用 |
| 安装 | 内置（iproute2） | 需net-tools包 |

```bash
# netstat习惯用法（新系统可能没装）
$ sudo apt install net-tools
$ netstat -tulnp                # 等价ss -tulnp
$ netstat -an | grep 80         # 等价ss -tan | grep :80
```

---

## 5.4 DNS解析

**一句话定义**：DNS（Domain Name System）将域名翻译为IP地址，是互联网的"电话簿"。

### 解析链路

```
应用 (www.google.com)
  ↓
/etc/nsswitch.conf: hosts: files dns
  ↓
/etc/hosts 文件               → 找到？返回
  ↓ 没找到
/etc/resolv.conf DNS服务器     → 查询DNS
  ↓
DNS递归查询：
  . → com. → google.com. → www.google.com. → 142.250.80.46
```

### 排障工具

```bash
# 1. dig - DNS查询（首选）
$ dig +short google.com
142.250.80.46

$ dig google.com               # 详细模式
;; ANSWER SECTION:
google.com.    300    IN    A    142.250.80.46
# ↑TTL=300秒

# 2. 查特定记录
$ dig MX google.com             # 邮件记录
$ dig NS google.com             # 域名服务器
$ dig TXT google.com            # 文本记录
$ dig AAAA google.com           # IPv6记录
$ dig @8.8.8.8 google.com      # 指定DNS服务器

# 3. 反向解析（IP→域名）
$ dig -x 142.250.80.46 +short
lga34s47-in-f14.1e100.net.

# 4. nslookup - 简单查询
$ nslookup google.com
Server:     127.0.0.53
Address:    127.0.0.53#53

Non-authoritative answer:
Name:   google.com
Address: 142.250.80.46

# 5. 追踪解析过程
$ dig +trace google.com
# 从根服务器到最终答案的完整过程

# 6. host - 最简
$ host google.com
google.com has address 142.250.80.46
```

### DNS配置文件

```bash
# /etc/resolv.conf - DNS解析器配置
$ cat /etc/resolv.conf
nameserver 127.0.0.53           # systemd-resolved
options edns0 trust-ad
search localdomain

# /etc/hosts - 静态映射
$ cat /etc/hosts
127.0.0.1       localhost
192.168.1.100   server01.example.com server01
10.0.0.5        db-master.internal

# /etc/nsswitch.conf - 名称服务切换
$ cat /etc/nsswitch.conf | grep hosts
hosts:          files dns
# 先查/etc/hosts，再查DNS
```

### 实战：DNS问题排查

```bash
# 场景：ping不通域名但能ping通IP

# 1. 确认DNS服务器可达
$ nslookup google.com 8.8.8.8          # 用Google DNS
# OK → DNS服务本身没问题

# 2. 确认本地DNS配置
$ cat /etc/resolv.conf                  # 看nameserver行
nameserver 127.0.0.53                  # systemd-resolved
$ resolvectl status                     # 看实际用哪个DNS

# 3. 刷新DNS缓存（如果有）
$ sudo systemd-resolve --flush-caches   # systemd-resolved
$ sudo resolvectl flush-caches          # 新版命令

# 4. 看域名是否被block
$ dig +short google.com @114.114.114.114  # 换国内DNS
$ curl -v --dns-servers 8.8.8.8 https://google.com  # curl指定DNS
```

---

## 5.5 curl / wget：网络客户端

### curl：全能URL传输器

```bash
# 1. GET请求（默认）
$ curl https://api.example.com/v1/status

# 2. 详细输出
$ curl -v https://httpbin.org/get
* Connected to httpbin.org (34.192.121.172) port 443
* TLSv1.3 handshake
* Server certificate: *.httpbin.org
> GET /get HTTP/1.1
> Host: httpbin.org
> User-Agent: curl/8.5.0
< HTTP/1.1 200 OK
< Content-Type: application/json
{...}

# 3. POST请求
$ curl -X POST -d '{"name":"test"}' -H "Content-Type: application/json" https://httpbin.org/post

# 4. 上传/下载文件
$ curl -O https://example.com/file.tar.gz       # 下载（同名保存）
$ curl -o output.tar.gz https://example.com/file.tar.gz  # 下载（改名）
$ curl -T file.tar.gz https://example.com/upload          # 上传

# 5. SSL/证书
$ curl -k https://self-signed.example.com        # 忽略证书验证（危险）
$ curl --cert client.pem https://mtls.example.com  # 双向认证

# 6. 认证
$ curl -u user:pass https://api.example.com          # Basic Auth
$ curl -H "Authorization: Bearer token123" https://api.example.com  # Bearer

# 7. 跟随重定向
$ curl -L https://short.link/abc              # 自动跟随302

# 8. 性能指标
$ curl -w "\ntime_total: %{time_total}s\ntime_connect: %{time_connect}s\ntime_starttransfer: %{time_starttransfer}s\n" -o /dev/null -s https://example.com
time_total: 0.234s
time_connect: 0.045s
time_starttransfer: 0.200s

# 9. 限速下载
$ curl --limit-rate 1M -O https://example.com/bigfile.iso
```

### wget：批量下载器

```bash
# 1. 断点续传
$ wget -c https://example.com/bigfile.iso

# 2. 镜像网站
$ wget -m -k -p https://example.com/docs/
# -m 镜像 -k 转换本地链接 -p 下载所有资源

# 3. 递归下载
$ wget -r -np -nH --cut-dirs=2 https://example.com/pub/linux/kernel/v6.x/

# 4. 后台下载
$ wget -b -o wget.log https://example.com/bigfile.iso
# -b 后台 -o 日志文件

# 5. 限速
$ wget --limit-rate=1M https://example.com/bigfile.iso

# 6. 重试
$ wget --tries=5 --retry-connrefused https://unstable-server.com/file
```

### curl vs wget

| 场景 | curl | wget |
|------|------|------|
| API交互 | ✅ 首选 | ❌ |
| 大文件下载 | 一般 | ✅ 续传 |
| 镜像网站 | ❌ | ✅ |
| 查看HTTP头 | ✅ `-I` | ❌ |
| 递归下载 | ❌ | ✅ |
| 默认安装 | ✅ (大多数) | ❌ 需安装 |

---

## 5.6 ping / traceroute / mtr / tcpdump：网络诊断四件套

### ping：最基础的连通性测试

```bash
# 1. 基础用法
$ ping -c 4 google.com
PING google.com (142.250.80.46): 56 data bytes
64 bytes from 142.250.80.46: icmp_seq=0 ttl=118 time=15.123 ms
64 bytes from 142.250.80.46: icmp_seq=1 ttl=118 time=14.987 ms
64 bytes from 142.250.80.46: icmp_seq=2 ttl=118 time=15.234 ms
64 bytes from 142.250.80.46: icmp_seq=3 ttl=118 time=15.001 ms

--- google.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss
round-trip min/avg/max/stddev = 14.987/15.086/15.234/0.106 ms

# 2. 关键指标
# icmp_seq    - 包序号（顺序乱=路由波动）
# ttl         - Time To Live（经过路由数）
# time        - 往返延迟
# packet loss - 丢包率（>0%要关注）

# 3. 测试本地网络栈
$ ping 127.0.0.1                        # TCP/IP栈正常？
$ ping 192.168.1.100                    # 本机IP正常？
$ ping 192.168.1.1                      # 网关正常？
$ ping 8.8.8.8                          # 外网正常？
$ ping google.com                       # DNS正常？

# 4. 其他选项
$ ping -i 0.2 google.com               # 间隔0.2秒
$ ping -s 1000 google.com              # 大包（1000字节）
$ ping -f google.com                   # 洪水模式（需root，慎用！）
$ ping -c 100 -q google.com            # 安静模式，最后显示统计
```

### traceroute：看数据包走哪条路

```bash
# 1. 追踪路由
$ traceroute google.com
 1  192.168.1.1 (192.168.1.1)  1.123 ms  0.987 ms  0.856 ms
 2  10.0.0.1 (10.0.0.1)  5.234 ms  5.123 ms  5.098 ms
 3  192.0.2.1 (192.0.2.1)  10.456 ms  10.234 ms  10.123 ms
 ...
 8  142.250.80.46 (142.250.80.46)  15.123 ms  14.987 ms  15.234 ms

# 2. mtr - 实时traceroute+ping
$ mtr google.com
# 交互式显示，按包丢失率排序
# ? 表示路由器没回ICMP（星号*）

# 3. traceroute vs mtr
# traceroute = 一次路径探查
# mtr = 持续监测路径变化

# 4. 跳过DNS（更快）
$ mtr -n google.com
```

### tcpdump：高级抓包过滤

```bash
# 1. 基础抓包
$ sudo tcpdump -i eth0 -nn port 80
# -nn = 不解析域名和端口名（快）

# 2. 按主机/端口范围
$ sudo tcpdump -i eth0 -nn host 192.168.1.100
$ sudo tcpdump -i eth0 -nn portrange 8000-9000

# 3. 按TCP标志位（排障利器）
$ sudo tcpdump -i eth0 -nn 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0'
# 抓所有SYN或FIN包

# 4. 抓HTTP GET请求（排查API调用）
$ sudo tcpdump -i eth0 -A -s 0 'tcp port 80 and (tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420)'
# 0x47455420 = "GET " 十六进制

# 5. 写pcap文件（后可用Wireshark分析）
$ sudo tcpdump -i eth0 -nn -w capture.pcap port 443

# 6. 抓包快速统计：谁在连接我？
$ sudo tcpdump -i eth0 -nn -c 1000 port 80 | awk '{print $3}' | sort | uniq -c | sort -rn | head

# 7. 抓特定大小包（排查MTU问题）
$ sudo tcpdump -i eth0 -nn 'greater 1400'

# 8. 只抓新连接（SYN包）
$ sudo tcpdump -i eth0 -nn 'tcp[tcpflags] & tcp-syn != 0'
```

### ip route：高级路由排查

```bash
# 1. 查特定目标走哪条路
$ ip route get 8.8.8.8
8.8.8.8 via 192.168.1.1 dev eth0 src 192.168.1.100 uid 1000

# 2. 添加静态路由（双网卡场景）
$ sudo ip route add 10.0.0.0/8 via 192.168.1.254 dev eth0

# 3. 策略路由：根据源IP选出口
$ sudo ip rule add from 192.168.1.100 table 100
$ sudo ip route add default via 192.168.1.2 table 100

# 4. 排查"能ping通但不能传大包"的MTU问题
$ tracepath 8.8.8.8                       # 自动探测路径MTU
$ ping -M do -s 1472 8.8.8.8             # 禁止分片测试
# 1500(MTU) - 20(IP头) - 8(ICMP头) = 1472
```

### 实战：网络故障排查流程

```bash
#!/bin/bash
# /usr/local/bin/netcheck.sh
# 一键网络诊断

echo "=== 网卡状态 ==="
ip link show

echo -e "\n=== IP配置 ==="
ip addr show | grep -E 'inet '

echo -e "\n=== 路由表 ==="
ip route show

echo -e "\n=== DNS配置 ==="
cat /etc/resolv.conf | grep -v '^#'

echo -e "\n=== 网关连通性 ==="
GATEWAY=$(ip route show | awk '/default/ {print $3}')
if [ -n "$GATEWAY" ]; then
    ping -c 2 -W 2 $GATEWAY && echo "网关可达" || echo "⚠ 网关不通！"
fi

echo -e "\n=== 外网连通性 ==="
ping -c 2 -W 2 8.8.8.8 && echo "外网可达" || echo "⚠ 外网不通！"

echo -e "\n=== DNS解析 ==="
nslookup google.com > /dev/null 2>&1 && echo "DNS正常" || echo "⚠ DNS异常！"

echo -e "\n=== 监听端口 ==="
ss -tlnp | awk '{print $4}' | sort | uniq -c

echo -e "\n=== 连接统计 ==="
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn | head -5
```

---

## 5.7 iptables / nftables：Linux防火墙

**一句话定义**：`iptables`是传统Linux防火墙；`nftables`是新标准（Linux 3.13+），语法更现代。

### 关键概念

```
数据包进入 → PREROUTING → 路由决策 → FORWARD → POSTROUTING → 离开
                                ↓
                           本机进程？ → INPUT → 本机 → OUTPUT
```

### iptables速查

```bash
# 1. 查看规则
$ sudo iptables -L -n -v
Chain INPUT (policy ACCEPT 1234 packets, 567890 bytes)
 pkts bytes target     prot opt in     out     source               destination
  100 10000 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22
  200 20000 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
Chain OUTPUT (policy ACCEPT 5000 packets, 100000 bytes)

# -L 列出规则
# -n 不解析域名（快）
# -v 详细统计

# 2. 添加规则
$ sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT      # 允许80端口
$ sudo iptables -A INPUT -p tcp --dport 22 -s 192.168.1.0/24 -j ACCEPT  # 只允许特定网段SSH
$ sudo iptables -A INPUT -j DROP                           # 默认拒绝（⚠ 先加到日志）

# 3. 删除规则
$ sudo iptables -D INPUT 1                                 # 删第1条
$ sudo iptables -D INPUT -p tcp --dport 80 -j ACCEPT      # 按内容删

# 4. 默认策略
$ sudo iptables -P INPUT DROP                              # 默认拒绝（⚠慎用）
$ sudo iptables -P FORWARD DROP

# 5. 保存/恢复
$ sudo iptables-save > /etc/iptables/rules.v4              # 保存
$ sudo iptables-restore < /etc/iptables/rules.v4           # 恢复

# 6. 常用组合：安全服务器
$ sudo iptables -P INPUT DROP
$ sudo iptables -A INPUT -i lo -j ACCEPT                   # 允许回环
$ sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT  # 允许已知连接
$ sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT      # 允许SSH
$ sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT      # 允许HTTP
$ sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT     # 允许HTTPS
$ sudo iptables -A INPUT -p icmp -j ACCEPT                 # 允许ping
# 其余全拒绝
```

### nftables：新一代防火墙

```bash
# 1. 查看规则
$ sudo nft list ruleset

# 2. 创建表
$ sudo nft add table inet myfilter

# 3. 添加链和规则
$ sudo nft 'add chain inet myfilter input { type filter hook input priority 0; policy drop; }'
$ sudo nft add rule inet myfilter input iif "lo" accept
$ sudo nft add rule inet myfilter input ct state established,related accept
$ sudo nft add rule inet myfilter input tcp dport 22 accept
$ sudo nft add rule inet myfilter input tcp dport {80, 443} accept
$ sudo nft add rule inet myfilter input icmp type echo-request accept

# 4. 保存
$ sudo nft list ruleset > /etc/nftables.conf

# 5. 永久生效（systemd）
$ sudo systemctl enable nftables
$ sudo systemctl start nftables
```

### 实战：防DDoS限速规则

```bash
# iptables limit模块
$ sudo iptables -A INPUT -p tcp --dport 80 -m limit --limit 25/minute --limit-burst 100 -j ACCEPT

# nftables
$ sudo nft add rule inet myfilter input tcp dport 80 limit rate 1/second burst 4 packets accept
```

---

## 5.8 网络配置文件速查

```bash
# Debian/Ubuntu
/etc/netplan/*.yaml              # netplan（Ubuntu 18.04+）
/etc/network/interfaces          # ifupdown（传统）
/etc/NetworkManager/system-connections/  # NetworkManager

# RHEL/CentOS/Fedora
/etc/sysconfig/network-scripts/ifcfg-eth0  # 传统 RHEL6-
/etc/NetworkManager/system-connections/    # NetworkManager
/etc/NetworkManager/NetworkManager.conf

# 通用
/etc/hosts                       # 主机名静态映射
/etc/resolv.conf                 # DNS解析器
/etc/nsswitch.conf               # 名称服务切换
/etc/hostname                    # 主机名

# 主机名管理
$ hostnamectl set-hostname server01.example.com
$ hostnamectl status
```

---

## 本章小结

### 网络诊断四步法

```
1. ip addr → 确认IP配置
2. ping 网关 → 确认局域网通
3. ping 8.8.8.8 → 确认外网通
4. dig google.com → 确认DNS正常
```

### 连接分析三步法

```
1. ss -tlnp → 确认监听端口
2. ss -tan → 确认连接状态
3. ss -tan | awk '$1=="ESTAB"' | head → 正常连接
```

### 下一章预告

**第六章：用户与组管理** — useradd/usermod/userdel/groups/passwd/shadow/PAM/NSS。

---

> **本章字数**：约 11,000 字
> **涉及命令**：ip, ss, netstat, dig, curl, wget, ping, traceroute, mtr, iptables, nftables

# 第六章：用户与组管理

> **本章定位**：用户和组是多用户系统的基石。从创建用户到PAM认证，从密码策略到sudo权限，本章覆盖系统管理员最核心的日常工作。

---

## 6.1 用户（User）：系统的"居民"

**一句话定义**：Linux中的用户 = UID + home目录 + shell + 密码 + 组成员。

### 用户文件

```bash
# /etc/passwd - 用户数据库
$ cat /etc/passwd | head -5
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
alice:x:1000:1000:Alice:/home/alice:/bin/bash
# 格式：用户名:密码占位:UID:GID:GECOS:主目录:Shell
# ↑ x表示密码存在/etc/shadow

# /etc/shadow - 加密密码（仅root可读）
$ sudo cat /etc/shadow | head -3
root:$y$j9T$Kj8...:19561:0:99999:7:::
daemon:*:19561:0:99999:7:::
alice:$y$j9T$Ab3...:19570:0:99999:7:::
# 格式：用户名:加密密码:最后修改:最小寿命:最大寿命:警告:不活动:过期
```

### UID分类

| UID范围 | 类型 | 说明 |
|---------|------|------|
| 0 | root | 超级管理员 |
| 1-999 | 系统用户 | 服务账号（不可登录） |
| 1000-60000 | 普通用户 | 可登录系统 |
| 65534 | nobody | 无特权用户 |

```bash
# 查看用户ID
$ id alice
uid=1000(alice) gid=1000(alice) groups=1000(alice),27(sudo),1005(devteam)

# 查看所有用户
$ cat /etc/passwd | cut -d: -f1 | sort
$ getent passwd | cut -d: -f1           # 同上，支持NIS/LDAP
```

---

## 6.2 用户管理命令

### useradd：创建用户

```bash
# 1. 默认创建
$ sudo useradd bob
# 创建UID≥1000的用户
# 默认组=bob（USERGROUPS_ENAB=yes时）

# 2. 指定属性
$ sudo useradd -u 2000 -g devteam -G sudo,docker -s /bin/zsh -c "Bob Smith" -m bob
# -u 2000      → UID=2000
# -g devteam   → 主组=devteam
# -G sudo,docker → 附加组
# -s /bin/zsh  → 登录shell
# -c "..."     → 注释
# -m           → 创建家目录

# 3. 系统用户（不可登录）
$ sudo useradd -r -s /usr/sbin/nologin myapp
# -r           → 系统用户(UID<1000)
# -s nologin   → 无法登录

# 4. 查看默认值
$ useradd -D
GROUP=100
HOME=/home
INACTIVE=-1
EXPIRE=
SHELL=/bin/sh
SKEL=/etc/skel
CREATE_MAIL_SPOOL=no

# 5. 创建后检查
$ grep bob /etc/passwd /etc/shadow /etc/group
$ ls -la /home/bob/
```

### usermod：修改用户

```bash
# 1. 改Shell
$ sudo usermod -s /bin/zsh bob

# 2. 改主目录
$ sudo usermod -d /home/bob_new -m bob
# -m 移动现有文件

# 3. 改附加组
$ sudo usermod -aG docker bob         # 追加
$ sudo usermod -G staff,docker bob     # 覆盖

# 4. 锁定/解锁
$ sudo usermod -L bob                  # 锁定
$ sudo usermod -U bob                  # 解锁

# 5. 改UID
$ sudo usermod -u 2001 bob

# 6. 改用户名
$ sudo usermod -l bob_new bob          # 文件名不变
$ sudo usermod -d /home/bob_new -m -l bob_new bob  # 一起改
```

### userdel：删除用户

```bash
$ sudo userdel bob                      # 删除用户，保留家目录
$ sudo userdel -r bob                   # 删除用户+家目录+邮件
$ sudo userdel -f bob                   # 强制（即使登录中也杀）

# 确认已删除
$ grep bob /etc/passwd /etc/shadow /etc/group
# 无输出 = 删干净
```

### 实战：批量创建用户

```bash
#!/bin/bash
# /usr/local/bin/batch-add-users.sh
# 从CSV创建用户：name,comment,shell

while IFS=',' read -r name comment shell; do
    sudo useradd -m -c "$comment" -s "$shell" "$name"
    echo "Created: $name ($comment)"
done < users.csv

# csv内容：
# alice,Alice Admin,/bin/bash
# bob,Bob Developer,/bin/zsh
# carl,Carl DevOps,/bin/bash
```

---

## 6.3 组（Group）：权限共享的基础

### 组文件

```bash
# /etc/group - 组数据库
$ cat /etc/group | head -5
root:x:0:
daemon:x:1:
sudo:x:27:alice,bob
docker:x:999:bob,charlie
devteam:x:1005:alice,bob,charlie,dan
# 格式：组名:密码占位:GID:成员列表

# /etc/gshadow - 组密码（几乎不用）
$ sudo groupadd shared
$ sudo groupadd -g 3000 mygroup     # 指定GID
$ sudo groupmod -n newname oldname  # 改名
$ sudo groupdel mygroup              # 删组

# 查看用户的所有组
$ groups alice
alice : alice sudo docker devteam
$ id alice
```

### 主组 vs 附加组

```bash
# 主组 (primary group)：创建文件时的默认组
$ id bob
uid=1001(bob) gid=1001(bob) groups=1001(bob),27(sudo)
#              ↑ 主组

# 附加组 (supplementary groups)：权限继承
# bob在sudo组，所以能用sudo

# 临时切换主组
$ newgrp devteam                      # 登录到devteam
# 新创建的shell，主组变成devteam
```

---

## 6.4 passwd / chage：密码管理

### passwd

```bash
# 1. 用户自己改密码
$ passwd
Changing password for alice.
Current password: 
New password: 
Retype new password: 
passwd: password updated successfully

# 2. root修改其他用户密码
$ sudo passwd bob

# 3. 锁定/解锁密码
$ sudo passwd -l bob                  # 锁定
$ sudo passwd -u bob                  # 解锁

# 4. 查看状态
$ sudo passwd -S bob
bob P 2026-07-02 0 99999 7 -1
# ↑状态(P=密码已设,L=锁定,NP=无密码)

# 5. 强制下次登录改密码
$ sudo passwd -e bob

# 6. 删除密码（危险！）
$ sudo passwd -d bob
```

### chage：密码过期策略

```bash
# 1. 查看密码过期信息
$ sudo chage -l bob
Last password change                    : Jul 02, 2026
Password expires                       : never
Password inactive                      : never
Account expires                        : never
Minimum number of days between password change      : 0
Maximum number of days between password change      : 99999
Number of days of warning before password expires   : 7

# 2. 设置过期规则
$ sudo chage -M 90 bob                # 90天后过期
$ sudo chage -m 7 bob                 # 最少7天才能改
$ sudo chage -W 14 bob                # 提前14天警告
$ sudo chage -I 7 bob                 # 过期后7天锁定

# 3. 生产环境推荐配置
$ sudo chage -M 90 -m 1 -W 7 -I 30 bob

# 4. 强制改密码
$ sudo chage -d 0 bob                 # 立即过期
```

### 实战：密码强度检查

```bash
# 安装libpam-pwquality
$ sudo apt install libpam-pwquality

# /etc/security/pwquality.conf
minlen = 12                           # 最小12位
dcredit = -1                          # 至少1位数字
ucredit = -1                          # 至少1位大写
lcredit = -1                          # 至少1位小写
ocredit = -1                          # 至少1位特殊字符
minclass = 3                          # 至少3类字符
maxrepeat = 3                         # 最多重复3次
```

---

## 6.5 实战：标准用户管理流程

```bash
#!/bin/bash
# /usr/local/bin/add-deploy-user.sh
# 创建部署用户的标准化流程

set -euo pipefail

USERNAME=${1:?Usage: $0 <username>}

# 1. 创建用户
sudo useradd -m -s /bin/bash "$USERNAME"

# 2. 设置初始密码（随机生成）
PASSWORD=$(openssl rand -base64 24)
echo "$USERNAME:$PASSWORD" | sudo chpasswd

# 3. 立即要求修改密码
sudo chage -d 0 "$USERNAME"

# 4. 设置密码策略
sudo chage -M 90 -m 1 -W 7 "$USERNAME"

# 5. 创建.ssh目录
sudo mkdir -p "/home/$USERNAME/.ssh"
sudo chmod 700 "/home/$USERNAME/.ssh"

# 6. 输出信息
echo "=== 用户 $USERNAME 已创建 ==="
echo "临时密码: $PASSWORD"
echo "已强制首次登录修改密码"
echo "已设置90天密码过期"
echo ""
echo "下一步:"
echo "1. 把SSH公钥放入 /home/$USERNAME/.ssh/authorized_keys"
echo "2. 如需要,添加到附加组: sudo usermod -aG docker $USERNAME"
```

---

## 6.6 PAM：可插拔认证模块

**一句话定义**：PAM（Pluggable Authentication Modules）是Linux认证框架，所有需要认证的服务（SSH、sudo、login）都通过它。

### PAM配置结构

```bash
# /etc/pam.d/ 目录下的文件
$ ls /etc/pam.d/
common-auth          # 认证相关（密码）
common-account       # 账号相关（过期、锁定）
common-password      # 密码变更
common-session       # 会话管理（环境、资源限制）

# 每条规则格式：type control module-path arguments
# 例：
auth    required    pam_unix.so    nullok_secure
# type    control     module          arguments
```

### 关键配置

```bash
# 1. 限制root SSH登录
$ sudo grep -E '^auth.*pam_securetty' /etc/pam.d/login
auth       required   pam_securetty.so
# /etc/securetty 定义root可登录的终端

# 2. 密码复杂度
$ cat /etc/pam.d/common-password
password    requisite   pam_pwquality.so retry=3
password    [success=1 default=ignore]  pam_unix.so obscure use_authtok try_first_pass yescrypt

# 3. 登录失败锁定（防暴力破解）
$ cat /etc/pam.d/common-auth
auth    required    pam_tally2.so deny=5 unlock_time=600
# 5次失败锁定600秒

# 查看锁定状态
$ sudo pam_tally2 -u bob
Login           Failures Latest failure     From
bob             3        07/02/26 10:00:00 192.168.1.50

# 解锁
$ sudo pam_tally2 -u bob --reset

# 4. 资源限制
$ cat /etc/pam.d/common-session
session    required   pam_limits.so
# 配合 /etc/security/limits.conf 使用
```

### /etc/security/limits.conf

```bash
# 格式: <domain> <type> <item> <value>

# 全局
*           soft    nofile      65536
*           hard    nofile      65536

# 特定用户
mysql       hard    nofile      65536
www-data    soft    nofile      65536

# 生效需要重启或重新登录
$ ulimit -n                         # 当前soft limit
65536
```

---

## 本章小结

### 用户管理命令矩阵

| 操作 | 命令 |
|------|------|
| 创建 | useradd | adduser(交互式) |
| 修改 | usermod |
| 删除 | userdel -r |
| 密码 | passwd / chage |
| 查看 | id / groups / getent |
| 锁定 | usermod -L / passwd -l |
| 登录审计 | last / lastb |

### 安全基线

```bash
# 1. 定期审计用户
$ sudo awk -F: '$3 >= 1000 {print $1}' /etc/passwd     # 可登录用户
$ sudo awk -F: '$7 !~ /nologin|false/ {print $1}' /etc/passwd  # 有shell的

# 2. 清理无主文件
$ sudo find / -nouser -o -nogroup 2>/dev/null

# 3. 检查空密码
$ sudo awk -F: '($2 == "" || $2 == "!") && $3 >= 1000' /etc/shadow

# 4. 检查UID重复
$ sudo awk -F: '{print $3}' /etc/passwd | sort -n | uniq -d
```

### 下一章预告

**第七章：服务管理与systemd** — Unit文件编写、target/runlevel、journalctl日志、定时器、资源限制。

---

> **本章字数**：约 8,500 字
> **涉及命令**：useradd, usermod, userdel, passwd, chage, groupadd, id, getent

# 第七章：服务管理与systemd

> **本章定位**：systemd是现代Linux的服务管理器，理解Unit文件编写和systemctl/journactl命令是运维基本功。

---

## 7.1 systemd概述

**一句话定义**：systemd是PID=1的初始化系统，管理服务、套接字、定时器、挂载点等一切"Unit"。

```bash
# 系统启动链条
$ systemd-analyze
Startup finished in 2.34s (kernel) + 5.67s (userspace) = 8.01s

$ systemd-analyze blame | head
8.123s NetworkManager-wait-online.service
1.234s snapd.service
0.567s apparmor.service

$ systemd-analyze critical-chain
graphical.target @5.890s
└─multi-user.target @5.890s
  └─nginx.service @4.500s +1.234s
    └─network.target @4.200s
```

---

## 7.2 Unit文件编写

### Unit类型

| 后缀 | 类型 | 说明 |
|------|------|------|
| `.service` | 服务 | 守护进程 |
| `.socket` | 套接字 | 网络端口激活 |
| `.target` | 目标 | 一组Unit的集合 |
| `.timer` | 定时器 | 替代cron |
| `.mount` | 挂载 | 文件系统挂载 |
| `.device` | 设备 | 内核设备 |

### 实战：完整的Service文件

```bash
$ cat > /etc/systemd/system/myapp.service <<'EOF'
[Unit]
Description=My Application
Documentation=https://myapp.io/docs
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple                           # simple | forking | oneshot | notify
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure                    # no | always | on-failure | on-abnormal
RestartSec=10
TimeoutStartSec=30
TimeoutStopSec=30
LimitNOFILE=65536
LimitNPROC=4096
PrivateTmp=true                       # 隔离/tmp
ProtectSystem=full                    # 只读/system
ProtectHome=true                      # 隔离/home
NoNewPrivileges=true                  # 禁止提权

[Install]
WantedBy=multi-user.target
EOF

$ sudo systemctl daemon-reload
$ sudo systemctl enable --now myapp
```

### Type详解

| Type | 行为 | 实践 |
|------|------|------|
| `simple`（默认） | ExecStart启动=就绪 | 占用前台，不fork |
| `forking` | 父进程退出=就绪 | 传统daemon（nginx） |
| `oneshot` | 进程退出=完成 | 一次性任务 |
| `notify` | sd_notify()通知=就绪 | 支持systemd通知 |

---

## 7.3 systemctl实战

```bash
# 启停
$ sudo systemctl start/stop/restart/reload nginx
$ sudo systemctl enable/disable nginx
$ sudo systemctl is-active/is-enabled nginx

# 查看状态
$ systemctl status nginx
$ systemctl list-units --type=service --state=running

# 依赖关系
$ systemctl list-dependencies nginx
$ systemctl list-dependencies --reverse nginx   # 谁依赖我

# 屏蔽/取消屏蔽
$ sudo systemctl mask nginx            # 彻底禁用（连手动都不让启）
$ sudo systemctl unmask nginx

# 编辑服务
$ sudo systemctl edit nginx            # 创建override文件
$ sudo systemctl cat nginx             # 查看完整配置

# 重启/关机
$ sudo systemctl reboot
$ sudo systemctl poweroff
```

### Target = 运行级别

```bash
$ systemctl get-default                # 当前默认target
graphical.target

$ systemctl list-units --type=target
$ sudo systemctl set-default multi-user.target  # 改默认

# 紧急模式
$ sudo systemctl rescue                # 单用户模式（保留启动服务）
$ sudo systemctl emergency             # 最小模式（仅启动shell）
$ sudo systemctl isolate multi-user.target  # 临时切换
```

---

## 7.4 systemd定时器（替代cron）

```bash
# 1. 定时器文件
$ cat /etc/systemd/system/backup.timer
[Unit]
Description=Daily backup

[Timer]
OnCalendar=*-*-* 02:00:00             # 每天凌晨2点
Persistent=true                        # 补偿启动后错过的任务

[Install]
WantedBy=timers.target

# 2. 对应的service
$ cat /etc/systemd/system/backup.service
[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh

# 3. 启用
$ sudo systemctl enable --now backup.timer

# 4. 查看
$ systemctl list-timers
NEXT                        LEFT    LAST                        PASSED  UNIT
Wed 2026-07-03 02:00:00 CST 5h left Tue 2026-07-02 02:00:00 CST 18h ago backup.timer
```

---

## 7.5 journalctl：systemd日志

```bash
# 1. 基础查看
$ journalctl
# 翻页查看，按q离开

# 2. 按服务过滤
$ journalctl -u nginx
$ journalctl -u nginx --since "2026-07-02 08:00" --until "2026-07-02 12:00"
$ journalctl -u nginx -f               # follow实时

# 3. 按时间
$ journalctl --since today
$ journalctl --since "30 minutes ago"
$ journalctl --since "1 hour ago" --until "5 minutes ago"

# 4. 按优先级
$ journalctl -p err                     # 只看ERROR及以上
$ journalctl -p emerg..warning          # 紧急到警告

# 5. 本次启动日志
$ journalctl -b
$ journalctl -b -1                      # 上次启动（排查崩溃原因）

# 6. 空间管理
$ journalctl --disk-usage               # 当前占用
$ sudo journalctl --vacuum-size=500M    # 限制500M
$ sudo journalctl --vacuum-time=7d      # 保留7天

# 7. 持久化配置
$ sudo mkdir -p /var/log/journal
$ sudo systemd-tmpfiles --create --prefix /var/log/journal
$ sudo systemctl restart systemd-journald
```

### grep vs journalctl

```bash
# journalctl -g 是systemd自己的grep
$ journalctl -g "failed|error|timeout" --case-insensitive
# 等同于 grep -iE "failed|error|timeout" 但对结构化日志更有效
```

---

## 7.6 Slice与Scope：systemd的资源层级

**一句话定义**：Slice是systemd对cgroups的组织方式，将服务分组为层级树，统一施加资源限制。

### 默认Slice层级

```bash
$ systemd-cgls
Control group /:
-.slice
├─user.slice
│ └─user-1000.slice
│   ├─session-2.scope
│   │ ├─1234 sshd: alice@pts/0
│   │ ├─1235 -bash
│   │ └─1456 systemd-cgls
│   └─user@1000.service
├─init.scope
├─system.slice
│ ├─nginx.service
│ ├─sshd.service
│ ├─mysqld.service
│ └─cron.service
└─machine.slice
  └─docker-abc123.scope
```

**三层结构**：
- **Slice**：资源池（system.slice、user.slice、machine.slice）
- **Scope**：临时进程组（SSH会话、Docker容器）
- **Service**：持久服务（nginx、mysql）

### 实战：按Slice限制资源

```bash
# 1. 查看某个服务的slice归属
$ systemctl show nginx | grep Slice
Slice=system.slice

# 2. 限制整个用户slice（所有登录用户加起来最多50% CPU）
$ sudo systemctl set-property user.slice CPUQuota=50%

# 3. 限制某服务的CPU
$ sudo systemctl set-property nginx.service CPUQuota=200%
# 200% = 2个核心

# 4. 限制内存
$ sudo systemctl set-property mysql.service MemoryMax=2G

# 5. 查看已设置的属性
$ systemctl show nginx | grep -E 'CPUQuota|MemoryMax'
CPUQuota=200%
MemoryMax=

# 6. 按Slice限制（特定服务组）
$ sudo systemctl set-property system.slice MemoryHigh=4G
# MemoryHigh = 软限制（尽力而为）
# MemoryMax = 硬限制（达到后OOM）
```

### systemd-run：一次性任务的资源隔离

```bash
# 等价于 docker run --memory=512m --cpus=1，但直接作用于进程
$ sudo systemd-run --user --scope \
    -p MemoryMax=512M \
    -p CPUQuota=100% \
    -p IOWeight=10 \
    ./heavy_batch_job.sh

# 在systemd-cgls中会看到：
# └─session-2.scope
#   └─run-u12345.scope
#     └─heavy_batch_job.sh
```

## 7.7 依赖分析与调试

```bash
# 1. 查看服务依赖树
$ systemctl list-dependencies nginx.service
nginx.service
├─system.slice
├─network.target
├─sysinit.target
│ ├─local-fs.target
│ ├─swap.target
│ └─systemd-journald.service
└─-.mount

# 2. 反向：谁依赖我？
$ systemctl list-dependencies --reverse network.target
network.target
├─nginx.service
├─sshd.service
├─docker.service
└─mysql.service

# 3. 图形化依赖（生成SVG）
$ systemd-analyze dot nginx.service | dot -Tsvg > nginx-deps.svg

# 4. 启动关键路径
$ systemd-analyze critical-chain nginx.service
multi-user.target @5.890s
└─nginx.service @4.500s +1.234s
  └─network.target @4.200s +300ms
    └─network-pre.target @4.190s +10ms
      └─firewalld.service @3.500s +680ms
        └─basic.target @3.200s

# 5. 启动耗时最长的服务
$ systemd-analyze blame | head
8.123s NetworkManager-wait-online.service
1.234s snapd.service
0.567s firewalld.service
0.345s docker.service
```

### 依赖配置实战

```bash
# 合理使用After/Requires/Wants/BindsTo
$ cat /etc/systemd/system/myapp.service
[Unit]
Description=MyApp
Requires=postgresql.service          # 必须启动，否则自己启动失败
Wants=redis.service                   # 最好启动，但没启动也能启动
After=network.target postgresql.service redis.service
BindsTo=caddy.service                 # 同生同死

[Service]
...
```

| 指令 | 含义 | 带启动 |
|------|------|--------|
| `Requires=` | 硬依赖，对方必须启动 | ✅ |
| `Wants=` | 软依赖，尽力而为 | ✅ |
| `After=` | 排序关系，不做启动保证 | ❌ |
| `Before=` | 先于对方启动 | ❌ |
| `BindsTo=` | 生命周期绑定 | ✅ |
| `Conflicts=` | 不能同时运行 | ❌ |

---

## 本章小结

| 命令 | 用途 |
|------|------|
| systemctl start/stop/restart | 服务生命周期 |
| systemctl enable/disable | 开机自启 |
| systemctl status | 运行状态 |
| systemctl edit | 覆盖配置 |
| journalctl -u | 服务日志 |
| journalctl -b | 本次启动 |
| systemd-analyze blame | 启动耗时 |

### 下一章预告：性能监控与调优

---

> **本章字数**：约 5,500 字

# 第八章：性能监控与调优

> **本章定位**：从top到perf，从iostat到iotop，本章覆盖运维必须掌握的性能监控工具和调优参数。

---

## 8.1 CPU监控

```bash
# 1. top / htop / btop（实时）
$ top -bn1 | head -5

# 2. mpstat - 每核CPU统计
$ mpstat -P ALL 2 5
Average:  CPU   %usr  %sys  %iowait   %idle
Average:  all   5.20  2.10     0.50   92.20
# %iowait 高 = 磁盘瓶颈，%steal 高 = 虚拟化资源争抢

# 3. 负载均值
$ uptime
10:00:00 up 30 days, load average: 0.50, 0.40, 0.30

# 4. 进程CPU排行
$ ps -eo pid,ppid,pcpu,cmd --sort=-pcpu | head -10

# 5. strace - 追踪系统调用（排查进程卡住的原因）
$ sudo strace -p 1234                     # 实时看进程在做什么
$ sudo strace -c -p 1234                  # 统计系统调用频率
$ sudo strace -e trace=network -p 1234    # 只看网络调用
# 场景：应用无响应但CPU和内存正常 → strace看卡在哪个系统调用

# 6. perf - 定位CPU热点函数
$ sudo perf top                           # 实时热点函数
$ sudo perf stat -p 1234 sleep 5          # 5秒性能计数
$ sudo perf record -p 1234 -g sleep 30    # 30秒采样（含调用栈）
$ sudo perf report                        # 查看报告
$ sudo perf script | flamegraph-tool > flame.svg  # 生成火焰图（需额外工具）
# 场景：CPU 100%但不知道哪行代码 → perf定位热点函数

# 7. pidstat - 进程级CPU监控
$ pidstat 2 5                             # 每2秒，5次
# 比top更适合脚本化监控
```

## 8.2 内存监控

```bash
# 1. free
$ free -h
# available = free + 可回收cache（真实可用内存）

# 2. vmstat - 全方位系统状态
$ vmstat 2 5
# si/so列 → swap换入/换出（非0说明内存不足）
# bi/bo列 → 磁盘读写（高=IO瓶颈）

# 3. smem - 更准确的进程内存
$ sudo smem -tk
# PSS = 私有内存 + 共享内存/共享进程数（最准确）

# 4. 排查内存泄漏
$ while true; do ps -o pid,rss,cmd -p $(pgrep myapp) | tail -1; sleep 5; done
# 看RSS是否持续增长

# 5. OOM分析
$ dmesg | grep -i "out of memory\|oom-killer"
$ grep -i "killed process" /var/log/syslog
# 找出被内核OOM Killer杀掉的进程
```

## 8.3 磁盘IO监控

```bash
# 1. iostat - 磁盘级IO
$ iostat -x 2 3
# await = 平均等待时间（>10ms要关注，>50ms瓶颈）
# %util = 利用率（SSD不适用此指标，看await）

# 2. iotop - 进程级IO
$ sudo iotop -o -b -n 1                   # 只看有IO的进程

# 3. ioping - 磁盘延迟测试
$ sudo ioping /data
# <1ms = NVMe, 1-5ms = SATA SSD, 5-15ms = HDD

# 4. fio - 磁盘基准测试
$ sudo fio --name=randwrite --ioengine=libaio --rw=randwrite --bs=4k --numjobs=1 --size=1G --runtime=60 --time_based --filename=/data/test

# 5. 大文件清理：找磁盘占用大户
$ du -sh /* 2>/dev/null | sort -h | tail -10
$ ncdu /                                   # 交互式磁盘分析（需安装）
```

## 8.4 网络IO监控

```bash
# 1. sar - 历史数据回看
$ sar -n DEV 1 5                           # 每1秒网卡流量，共5次
$ sar -n TCP,ETCP 1 5                      # TCP连接统计

# 2. iftop - 实时带宽
$ sudo iftop -i eth0

# 3. nethogs - 按进程显示流量
$ sudo nethogs eth0

# 4. 找网络瓶颈
$ ethtool eth0 | grep Speed                # 网卡速率
$ ss -s                                     # 总连接数统计
$ nstat -az | grep -E 'TcpExt.*Timeouts|TcpExt.*Retrans'
# 查看TCP超时和重传统计
```

## 8.5 实战：一键性能采集脚本

```bash
#!/bin/bash
echo "=== $(date) ==="
echo "CPU: $(top -bn1 | awk '/Cpu/ {print $2}')% us"
echo "MEM: $(free -m | awk '/Mem/ {printf "%d/%d MB (%.0f%%)", $3, $2, $3*100/$2}')"
echo "LOAD: $(uptime | awk -F'load average:' '{print $2}')"
echo "DISK: $(df -h / | awk 'NR==2{print $5}') used"
echo "TOP_CPU: $(ps -eo pcpu,cmd --sort=-pcpu --no-headers | head -1)"
echo "TOP_MEM: $(ps -eo pmem,cmd --sort=-pmem --no-headers | head -1)"
```

### 下一章预告：日志与定时任务

---

> **本章字数**：约 3,000 字

# 第九章：日志管理与定时任务

> **本章定位**：日志是故障排查的眼睛，定时任务是自动化的基础。

---

## 9.1 rsyslog与journald

```bash
# rsyslog（传统，仍在用）
$ tail -f /var/log/syslog             # Debian/Ubuntu
$ tail -f /var/log/messages           # RHEL/CentOS
$ tail -f /var/log/auth.log           # 认证日志
$ sudo ls -la /var/log/

# journald（现代，systemd自带）
$ journalctl -u nginx -f              # follow实时
$ journalctl -u nginx --since "1 hour ago"
$ journalctl -u nginx -p err          # 只看ERROR及以上
$ journalctl -b -1                    # 上次启动日志（查崩溃原因）
$ journalctl -k                        # 内核日志
$ journalctl --disk-usage              # 磁盘占用
$ sudo journalctl --vacuum-size=500M   # 限制500M
```

## 9.2 日志轮转

```bash
$ cat /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily                              # 每天轮转
    missingok                          # 文件不存在不报错
    rotate 14                          # 保留14份
    compress                           # 压缩旧日志
    delaycompress                      # 延迟1天压缩
    notifempty                         # 空文件不轮转
    create 640 nginx adm
    sharedscripts
    postrotate
        [ -f /run/nginx.pid ] && kill -USR1 $(cat /run/nginx.pid)
    endscript
}

$ sudo logrotate -f /etc/logrotate.conf
$ sudo logrotate -d /etc/logrotate.conf   # 调试模式（不实际执行）
```

## 9.3 结构化日志与集中管理

```bash
# 应用输出到syslog（Python示例）
import logging
import logging.handlers
logger = logging.getLogger('myapp')
handler = logging.handlers.SysLogHandler(address='/dev/log')
handler.setFormatter(logging.Formatter('myapp: %(message)s'))
logger.addHandler(handler)
logger.error("Payment failed: order_id=1234, reason=timeout")

# 查询：
$ journalctl -t myapp | grep "Payment failed"
# 通过结构化日志（JSON），用jq快速过滤
$ journalctl -t myapp -o json | jq 'select(.MESSAGE | contains("timeout"))'

# 生产环境集中日志方案：
# 轻量级：filebeat/fluent-bit → Elasticsearch/Grafana Loki
# 全功能：Promtail → Loki（配合Prometheus指标+日志关联）
# 传统：rsyslog → syslog-ng → 中心服务器
```

## 9.4 cron定时任务

### crontab格式

```
# 分钟 小时 日 月 星期 命令
  0    2    *  *   *    /usr/local/bin/backup.sh

# 常用频率
* * * * *   每分钟
0 * * * *   每小时整点
0 2 * * *   每天凌晨2点
0 9 * * 1-5 工作日9:00
0 0 1 * *   每月1日
```

### crontab命令

```bash
$ crontab -l                          # 列出
$ crontab -e                          # 编辑
$ sudo crontab -l -u bob              # 查看bob的

# 系统级
$ ls /etc/cron.d/                     # 系统任务目录
$ ls /etc/cron.daily/                 # 每天
$ ls /etc/cron.hourly/                # 每小时
```

### 实战：cron的四个坑

```bash
# 坑1：环境变量（cron环境极度干净）
0 2 * * * . /home/alice/.profile && /usr/local/bin/script.sh
# 或用绝对路径

# 坑2：重定向输出（不重定向会发邮件塞满/var/mail）
0 2 * * * /backup.sh >> /var/log/backup.log 2>&1

# 坑3：防并发
0 2 * * * flock -n /tmp/backup.lock -c /usr/local/bin/backup.sh

# 坑4：调试
$ grep CRON /var/log/syslog
$ echo "PATH=$PATH" | crontab -
```

### systemd timer（现代替代cron）

见第七章7.4，timer优于cron的原因：
- 自动记录日志到journald（无需手动重定向）
- 支持单调时钟（`OnBootSec`、`OnUnitActiveSec`）
- 可设随机延迟（`RandomizedDelaySec`）防惊群
- 失败的timer自动记录状态

---

## 本章小结

| 工具 | 用途 |
|------|------|
| journalctl | systemd日志查看 |
| rsyslog | 传统syslog |
| logrotate | 日志轮转 |
| crontab -e | 定时任务 |
| systemd timer | 现代化定时器 |

### 下一章预告：软件包管理

---

> **本章字数**：约 3,500 字

# 第十章：软件包管理

> **本章定位**：软件包管理是系统管理的基础操作——安装、升级、卸载、锁定版本、管理仓库。

---

## 10.1 apt / dpkg（Debian/Ubuntu）

```bash
# apt（高层，推荐）
$ sudo apt update && sudo apt upgrade
$ sudo apt install nginx
$ sudo apt purge nginx                  # 完全卸载（含配置）
$ sudo apt autoremove                   # 清理孤儿依赖
$ apt search keyword

# 锁定版本
$ sudo apt-mark hold nginx              # 锁定
$ sudo apt-mark unhold nginx            # 解锁
$ sudo apt-mark showhold                # 查看所有锁定的包

# dpkg（底层）
$ dpkg -l | grep nginx                  # 列表
$ dpkg -L nginx                         # 包包含的文件
$ dpkg -S /etc/nginx/nginx.conf         # 文件属于哪个包
$ dpkg --get-selections | grep hold     # 所有锁定包
```

## 10.2 dnf / rpm（RHEL/CentOS/Fedora）

```bash
# dnf（高层）
$ sudo dnf update
$ sudo dnf install nginx
$ sudo dnf remove nginx
$ sudo dnf versionlock list             # 版本锁

# rpm（底层）
$ rpm -qa | grep nginx                  # 全部已安装
$ rpm -ql nginx                         # 包文件列表
$ rpm -qf /etc/nginx/nginx.conf         # 文件属于哪个包
$ rpm -q --changelog nginx | head       # 更新日志
$ rpm -V nginx                          # 验证包完整性
```

## 10.3 仓库管理

```bash
# apt仓库（Debian/Ubuntu）
$ cat /etc/apt/sources.list.d/myrepo.list
deb [signed-by=/usr/share/keyrings/myrepo.gpg] https://repo.example.com stable main

$ sudo apt update

# dnf仓库（RHEL/CentOS）
$ cat /etc/yum.repos.d/myrepo.repo
[myrepo]
name=My Repository
baseurl=https://repo.example.com/$releasever/
gpgcheck=1
enabled=1

$ sudo dnf makecache
```

## 10.4 实践：生产环境安全更新

```bash
# 1. 仅安装安全更新（最小变更）
$ sudo unattended-upgrade                # Debian自动安全更新
$ sudo dnf update --security             # RHEL

# 2. 自动更新配置
$ cat /etc/apt/apt.conf.d/20auto-upgrades
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";

# 3. 更新后检查
$ sudo needrestart                       # 哪些服务需要重启
$ checkrestart                           # 同上（Debian）
$ sudo needs-restarting -r               # RHEL

# 4. 内核热修补（无需重启，Ubuntu Pro）
$ sudo pro enable livepatch
$ canonical-livepatch status
```

## 10.5 实战：创建自定义deb包

```bash
$ mkdir -p myapp_1.0/DEBIAN
$ cat myapp_1.0/DEBIAN/control
Package: myapp
Version: 1.0
Architecture: amd64
Maintainer: Alice <alice@example.com>
Depends: libc6 (>= 2.35)
Description: My Application

$ mkdir -p myapp_1.0/usr/local/bin
$ cp /path/to/myapp myapp_1.0/usr/local/bin/
$ dpkg-deb --build myapp_1.0
$ sudo dpkg -i myapp_1.0.deb
```

---

## 本章小结

| Debian/Ubuntu | RHEL/CentOS | 操作 |
|---------------|-------------|------|
| apt update | dnf check-update | 更新索引 |
| apt install | dnf install | 安装 |
| apt remove | dnf remove | 卸载 |
| apt search | dnf search | 搜索 |
| dpkg -l | rpm -qa | 已安装列表 |

### 下一章预告：安全与审计

---

> **本章字数**：约 3,000 字

# 第十一章：安全与审计

> **本章定位**：Linux安全不只是防火墙——从SSH加固到audit审计，从SELinux到漏洞扫描。

---

## 11.1 SSH安全加固

```bash
# /etc/ssh/sshd_config 重要参数
Port 2222                             # 改端口（减少扫描）
PermitRootLogin no                    # 禁止root登录
PasswordAuthentication no             # 禁用密码（仅密钥）
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
MaxAuthTries 3                        # 最多3次尝试
ClientAliveInterval 300               # 5分钟keepalive
ClientAliveCountMax 2                 # 2次无响应断开
AllowUsers alice bob                  # 白名单
AllowGroups ssh-users                 # 组白名单

$ sudo sshd -t                         # 检查配置
$ sudo systemctl reload sshd
```

### 实战：密钥登录设置

```bash
# 客户机：
$ ssh-keygen -t ed25519 -C "alice@server01"
# 生成 id_ed25519（私钥）和 id_ed25519.pub（公钥）

# 把公钥放到服务器：
$ ssh-copy-id -p 2222 alice@server01
# 或手动：
$ cat ~/.ssh/id_ed25519.pub | ssh alice@server01 "cat >> ~/.ssh/authorized_keys"

# 服务器端权限（严格！）
$ chmod 700 ~/.ssh
$ chmod 600 ~/.ssh/authorized_keys
```

---

## 11.2 auditd审计

```bash
# 1. 安装
$ sudo apt install auditd

# 2. 基本规则
$ sudo auditctl -w /etc/passwd -p wa -k passwd_changes
# -w 监听的文件  -p 权限(r/w/x/a)  -k 标记

$ sudo auditctl -w /etc/shadow -p wa -k shadow_changes
$ sudo auditctl -w /usr/bin/sudo -p x -k sudo_exec
$ sudo auditctl -a always,exit -F arch=b64 -S execve -k exec

# 3. 查看
$ sudo ausearch -k passwd_changes
$ sudo ausearch -ts today -k exec

# 4. 永久规则
$ sudo cat > /etc/audit/rules.d/audit.rules <<'EOF'
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/sudoers -p wa -k sudoers
-w /usr/bin/sudo -p x -k sudo_exec
-a always,exit -F arch=b64 -S execve -k exec
EOF
$ sudo systemctl restart auditd

# 5. 报告
$ sudo aureport --summary
$ sudo aureport -au                    # 认证事件
$ sudo aureport -f                     # 文件事件
```

---

## 11.3 SELinux / AppArmor

```bash
# SELinux（RHEL/CentOS默认）
$ getenforce                           # 查看状态
$ sudo setenforce 0                    # 临时关闭（permissive）
$ sudo setenforce 1                    # 启用（enforcing）
$ sudo ausearch -m avc                 # 查SELinux阻止的

# 问题排查：
$ sudo tail -f /var/log/audit/audit.log | grep AVC
# 常见：type=AVC msg=audit(...): avc:  denied  { write } for  ...
# 解决：
$ sudo audit2allow -a -M mypol         # 自动生成策略
$ sudo semodule -i mypol.pp

# AppArmor（Ubuntu/Debian默认）
$ sudo aa-status
$ sudo aa-complain /usr/sbin/nginx     # 告警模式
$ sudo aa-enforce /usr/sbin/nginx      # 强制模式
```

---

## 11.5 漏洞修补与CVE管理

### 安全更新策略

```bash
# 1. 检查可用的安全更新
# Debian/Ubuntu
$ sudo apt update
$ sudo unattended-upgrade --dry-run -d
$ cat /var/run/reboot-required           # 是否有需重启的内核更新

# RHEL/CentOS
$ sudo dnf updateinfo list --security
$ sudo dnf updateinfo list --cves CVE-2024-0001

# 2. 仅安装安全更新（最小化生产变更）
$ sudo unattended-upgrade                # Debian自动安装安全补丁
$ sudo dnf update --security             # RHEL

# 3. 自动更新配置
$ cat /etc/apt/apt.conf.d/20auto-upgrades
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";

# 4. 内核热修补（无需重启）
$ sudo apt install linux-modules-extra-$(uname -r)
# Canonical Livepatch（Ubuntu Pro）
$ sudo pro enable livepatch
$ canonical-livepatch status
```

### 已知漏洞扫描

```bash
# 1. 查正在使用的软件版本
$ dpkg -l | grep -E 'openssh|openssl|nginx'
$ rpm -qa | grep -E 'openssh|openssl|nginx'

# 2. 用CVSS评分优先级
# 9.0+ 立即修补
# 7.0-8.9 24小时内修补
# 4.0-6.9 计划窗口期修补

# 3. 快速脚本：检查当前系统是否有已知高危CVE
$ cat > /tmp/check-cve.sh <<'EOF'
#!/bin/bash
echo "=== 关键漏洞检查 ==="
echo "SSH版本: $(ssh -V 2>&1 | head -1)"
echo "OpenSSL版本: $(openssl version)"
echo "内核版本: $(uname -r)"
echo ""
echo "建议：访问 https://nvd.nist.gov 交叉验证"
EOF
```

## 11.6 提权防护

### SUID/SGID审计

```bash
# 1. 找所有SUID文件（最具风险）
$ sudo find / -perm -4000 -type f 2>/dev/null
/usr/bin/passwd
/usr/bin/su
/usr/bin/sudo
/usr/bin/pkexec                  # ← 历史上多次爆提权漏洞！
/usr/bin/mount
/usr/bin/umount
/usr/bin/newgrp
/usr/bin/chsh

# 2. 移除不安全的
$ sudo chmod u-s /usr/bin/pkexec
# 或更保险：dpkg-statoverride
$ sudo dpkg-statoverride --update --add root root 0755 /usr/bin/pkexec

# 3. 找所有SGID
$ sudo find / -perm -2000 -type f 2>/dev/null

# 4. 找无属主文件（可能被提权利用）
$ sudo find / -nouser -o -nogroup 2>/dev/null

# 5. 找全局可写文件
$ sudo find / -perm -2 ! -type l 2>/dev/null | grep -v '/proc\|/sys\|/dev'
```

### 限制用户提权路径

```bash
# 1. sudo精准控制（禁止shell逃逸）
$ sudo visudo
# 危险：alice ALL=(ALL) NOPASSWD: ALL        # 可以sudo su变成root
# 安全：alice ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx
#       alice ALL=(ALL) !/bin/bash, !/bin/sh, !/usr/bin/passwd root

# 2. 禁用CTRL+ALT+DELETE
$ sudo systemctl mask ctrl-alt-del.target
$ sudo systemctl daemon-reload

# 3. 限制cron/at
$ echo "alice" | sudo tee -a /etc/cron.allow
$ echo "bob" | sudo tee -a /etc/cron.allow
# 只有alice和bob可以用cron（其他人无法设定时任务提权）

# 4. 内核参数加固
$ sudo sysctl -w kernel.kptr_restrict=2    # 隐藏内核指针
$ sudo sysctl -w kernel.dmesg_restrict=1   # 限制dmesg
$ sudo sysctl -w kernel.yama.ptrace_scope=2 # 限制ptrace
```

---

## 11.7 实战：安全检查清单

```bash
#!/bin/bash
# /usr/local/bin/sec-audit.sh
echo "=== 安全审计 $(date) ==="

echo "1. 可登录用户:"
awk -F: '$7 !~ /nologin|false/ && $3>=1000 {print $1, $7}' /etc/passwd

echo "2. 空密码用户:"
awk -F: '($2==""||$2=="!") && $3>=1000 {print $1}' /etc/shadow

echo "3. 免密sudo用户:"
grep -E 'NOPASSWD' /etc/sudoers /etc/sudoers.d/* 2>/dev/null

echo "4. 监听端口:"
ss -tlnp | awk '{print $4}' | sort -u

echo "5. 失败登录:"
lastb | head -5

echo "6. SSH信任关系:"
find /home -name authorized_keys -exec cat {} \; 2>/dev/null
```

---

> **本章字数**：约 4,500 字

# 第十二章：容器化与云原生基础

> **本章定位**：Docker/Podman容器技术是现代Linux运维的核心，理解namespace+cgroups=容器。

---

## 12.1 容器原理回顾

```bash
# 容器 = namespace(隔离) + cgroups(限制) + rootfs(文件系统)
# namespace：PID、网络、挂载、UTS、IPC、User
# cgroups：CPU、内存、IO限制
# rootfs：overlay2/proxy文件系统

# 手动创建最小容器（理解原理）
$ sudo unshare --pid --fork --mount-proc \
               --net --uts --ipc --cgroup \
               --mount --map-root-user \
               /bin/bash
# 在这个shell里：
$ hostname container1
$ hostname                           # container1
$ ip a                               # 只有lo
$ ps aux                              # 只有自己（PID 1）
# Ctrl+D 退出
```

---

## 12.2 Docker核心命令

```bash
# 1. 镜像
$ docker pull nginx:alpine
$ docker images
$ docker rmi nginx:alpine
$ docker build -t myapp:1.0 .

# 2. 容器生命周期
$ docker run -d --name web -p 8080:80 nginx:alpine
$ docker ps                           # 运行中的
$ docker ps -a                        # 所有
$ docker stop/start/restart web
$ docker rm web
$ docker rm -f $(docker ps -aq)       # ⚠ 删除所有容器

# 3. 日志和调试
$ docker logs -f web
$ docker exec -it web /bin/sh
$ docker inspect web | jq '.[0].NetworkSettings.IPAddress'

# 4. 资源限制
$ docker run -d --name app --memory="512m" --cpus="1.5" myapp:1.0

# 5. 数据卷
$ docker volume create app-data
$ docker run -v app-data:/data myapp
$ docker run -v $(pwd)/config:/config:ro myapp  # 只读
```

### Dockerfile模板

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
USER nobody
CMD ["python", "server.py"]
```

---

## 12.3 Podman：无守护进程的替代品

```bash
# 安装
$ sudo apt install podman

# 和Docker几乎一样的命令
$ podman run -d --name web -p 8080:80 nginx:alpine
$ podman ps
$ podman logs web

# 优势：
# 1. 无守护进程（没有dockerd）
# 2. 不需要root（rootless模式）
# 3. 原生systemd集成
$ podman generate systemd web > ~/.config/systemd/user/container-web.service
$ systemctl --user enable --now container-web.service

# Docker兼容
$ alias docker=podman
```

---

## 12.4 Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - api
  
  api:
    build: ./api
    environment:
      - DB_HOST=db
    depends_on:
      - db
  
  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: secret

volumes:
  pgdata:
```

```bash
$ docker-compose up -d
$ docker-compose ps
$ docker-compose logs -f api
$ docker-compose down
```

---

## 12.4 存储驱动：overlay2详解

**一句话定义**：overlay2是Docker默认的联合文件系统驱动，通过分层挂载实现镜像复用和容器快速启动。

### overlay2工作原理

```
容器层（可写）:  /var/lib/docker/overlay2/abc.../merged/
镜像层（只读）:
  Layer 3:   /var/lib/docker/overlay2/def.../
  Layer 2:   /var/lib/docker/overlay2/ghi.../
  Layer 1:   /var/lib/docker/overlay2/jkl.../ (基础镜像)
```

```bash
# 1. 查看当前驱动
$ docker info | grep Storage
Storage Driver: overlay2

# 2. 查看镜像层
$ docker image inspect nginx:alpine | jq '.[0].RootFS.Layers'
[
  "sha256:abc...",
  "sha256:def...",
  "sha256:ghi..."
]
# 每一层对应一个Dockerfile指令

# 3. 磁盘占用
$ docker system df
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          15        5         2.5GB     1.2GB (48%)
Containers      5         3         500MB     100MB (20%)
Local Volumes   3         2         1GB       0B (0%)
Build Cache     20        0         800MB     800MB

# 4. 清理
$ docker system prune -a -f               # ⚠ 清理所有未使用
$ docker builder prune -f                  # 清理构建缓存
```

## 12.5 镜像优化与多阶段构建

### 优化前 vs 优化后

```dockerfile
# ❌ 传统方式（900MB）
FROM golang:1.21
WORKDIR /app
COPY . .
RUN go build -o server .
EXPOSE 8080
CMD ["./server"]

# ✅ 多阶段构建（15MB）
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server .

FROM alpine:3.19
RUN apk add --no-cache ca-certificates tzdata
COPY --from=builder /app/server .
EXPOSE 8080
USER nobody
CMD ["./server"]
```

### 镜像瘦身技巧

```bash
# 1. .dockerignore（先于COPY生效）
$ cat .dockerignore
.git/
*.md
node_modules/
__pycache__/
*.log

# 2. 合并RUN指令（减少层数）
RUN apt-get update && \
    apt-get install -y --no-install-recommends pkg1 pkg2 && \
    rm -rf /var/lib/apt/lists/*

# 3. 从特定阶段复制
COPY --from=builder /app/dist /dist

# 4. 查看镜像大小和层
$ docker history nginx:alpine
IMAGE          CREATED         SIZE      COMMENT
abc123         2 weeks ago     0B        CMD ["nginx" "-g"]
def456         2 weeks ago     1.5MB     EXPOSE 80
```

## 12.6 网络模型（CNI基础）

```bash
# Docker网络模式
$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
abc123         bridge    bridge    local     # 默认（NAT转发）
def456         host      host      local     # 共享宿主机网络
ghi789         none      null      local     # 无网络

# 1. bridge模式（默认）
$ docker run -d --name web -p 8080:80 nginx
# 容器获得172.17.x.x IP，通过NAT访问外网
# docker-proxy监听宿主机8080转发到容器80

# 2. host模式（高性能，适合网络IO密集型）
$ docker run --network host nginx
# 容器直接使用宿主机IP和端口，无NAT开销
# ⚠ 端口冲突风险

# 3. 自定义网络（容器间通信）
$ docker network create mynet
$ docker run -d --name db --network mynet postgres
$ docker run -d --name web --network mynet -p 8080:80 nginx
# web可以ping db，DNS自动解析
$ docker exec web ping db
PING db (172.18.0.2): 56 data bytes

# 4. 排查网络
$ docker network inspect mynet | jq '.[0].Containers'
$ docker exec web cat /etc/resolv.conf
nameserver 127.0.0.11           # Docker内置DNS
```

### Podman与CNI

```bash
# Podman默认不创建网络，需手动配置
$ podman network create mynet
$ podman run --network mynet nginx

# rootless容器网络限制
# rootless模式下不能用特权端口（<1024）
$ podman run -p 80:80 nginx
Error: rootlessport cannot expose privileged port
# 解决：$ sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80
```

---

## 12.7 实战：生产容器化清单

```bash
#!/bin/bash
# /usr/local/bin/container-sec-check.sh
# 容器安全检查清单

CONTAINER=${1:?Usage: $0 <container_name>}

echo "=== 容器安全检查: $CONTAINER ==="

# 1. 检查运行用户
docker inspect $CONTAINER | jq '.[0].Config.User'

# 2. 检查特权模式
docker inspect $CONTAINER | jq '.[0].HostConfig.Privileged'

# 3. 检查Capabilities
docker inspect $CONTAINER | jq '.[0].HostConfig.CapAdd'

# 4. 检查挂载
docker inspect $CONTAINER | jq '.[0].Mounts'

# 5. 检查资源限制
docker inspect $CONTAINER | jq '.[0].HostConfig | {Memory, CpuShares}'

# 6. 检查网络模式
docker inspect $CONTAINER | jq '.[0].HostConfig.NetworkMode'
```

---

## 本章小结

| 命令 | 用途 |
|------|------|
| docker run -d --name ... | 启动容器 |
| docker exec -it ... sh | 进入容器 |
| docker logs -f ... | 查看日志 |
| docker-compose up -d | 启动服务栈 |
| podman run ... | 免root容器 |

---

> **本章字数**：约 4,000 字
> **全书完成！**

# 附录A：Shell脚本实战：自动化你的日常工作

> **本章定位**：Shell脚本是Linux管理员的"母语"。本章不讲语法字典，而是从中级管理员最常踩的坑出发，串讲变量、条件、循环、正则，最后用五个生产级脚本收尾。附录A 建议在学完第4章（进程管理）后阅读。

---

## S.1 脚本安全底线：set -euo pipefail

**一句话定义**：`set -euo pipefail`是脚本的第一道防线，分别拦截"命令失败→继续执行""未定义变量当空用""管道中间失败→只看最后成功"三类经典bug。

```bash
#!/bin/bash
set -euo pipefail
# -e  任何命令返回非零立即退出（中断继续执行）
# -u  使用未定义变量立即报错（而不是当作空字符串）
# -o pipefail  管道中任一命令失败 = 管道失败（而不是只看最后一个）

# 没有 set -e 的恐怖场景：
# cd /nonexistent_dir
# rm -rf *        ← 这会在当前目录执行！！因为cd失败后脚本继续了
```

### 三个flag的实战对比

```bash
# 1. set -e 的坑：grep 没匹配返回 1
set -e
grep "pattern" file.txt || true        # 加 || true 防止退出
# 或
result=$(grep "pattern" file.txt) || true

# 2. set -u：强制声明变量
set -u
: "${BACKUP_DIR:?BACKUP_DIR must be set}"  # 未设置则退出并报错
echo "Backing up to $BACKUP_DIR"

# 3. set -x：调试模式（打印每条执行的命令）
set -x                                  # 开启
# ... your code ...
set +x                                  # 关闭
# 或运行时：bash -x script.sh
```

---

## S.2 变量与引号：双引号是安全的，单引号是字面的

```bash
# 变量引用规则：双引号包裹 = 安全
name="hello world"
echo "$name"           # hello world（正确）
echo $name             # hello world（碰巧对，但有空格就错）

path="/tmp/my files/"
echo "$path"           # /tmp/my files/（正确）
echo $path             # /tmp/my（错误！空格被当成分隔符）

# 单引号 = 一切原样
echo '$HOME is not expanded'   # $HOME is not expanded

# 命令替换：$() 优于反引号
current_date=$(date +%F)          # 推荐
current_date=`date +%F`           # 不推荐（难嵌套、难读）

# 默认值
echo "${VAR:-default}"             # VAR未设置时用default
echo "${VAR:=default}"             # VAR未设置时设为default

# 长度
echo "${#str}"                     # 字符串长度
echo "${str:0:10}"                 # 前10个字符

# 数组（bash特有）
files=(/var/log/*.log)
echo "${files[0]}"                 # 第一个
echo "${#files[@]}"                # 数组长度
for f in "${files[@]}"; do ...; done
```

---

## S.3 条件判断：test vs [[ ]]

```bash
# 1. test / [ ]（POSIX兼容，通用）
if [ "$a" = "$b" ]; then ...; fi
if [ -f "/etc/passwd" ]; then ...; fi   # 文件存在
if [ -z "$var" ]; then ...; fi          # 字符串为空
if [ "$num" -gt 10 ]; then ...; fi      # 大于

# 2. [[ ]]（bash/zsh特有，更安全）
if [[ "$a" == "$b" ]]; then ...; fi     # 双等号
if [[ "$str" =~ ^[0-9]+$ ]]; then ...; fi  # 正则匹配
if [[ -f "$file" && -r "$file" ]]; then ...; fi  # && 联动

# 关键区别：
# [ ]    要求变量加双引号、不支持正则、= 是字符串比较
# [[ ]]  不需要双引号、支持 =~、== 是模式匹配
```

### 常用文件测试

| 测试 | 含义 |
|------|------|
| `-f file` | 普通文件存在 |
| `-d dir` | 目录存在 |
| `-x file` | 可执行 |
| `-r file` | 可读 |
| `-w file` | 可写 |
| `-s file` | 文件存在且非空 |
| `-nt / -ot` | 比另一个文件新/旧 |

---

## S.4 循环：for / while / until

```bash
# 1. for 遍历文件
for log in /var/log/*.log; do
    echo "Processing $log (size: $(wc -c < "$log") bytes)"
    gzip "$log"
done

# 2. for 遍历列表
for user in alice bob charlie; do
    id "$user" && echo "$user exists" || echo "$user missing"
done

# 3. while 读文件（逐行处理）
while IFS= read -r line; do
    echo "Line: $line"
done < /etc/passwd
# IFS= 保留前导空格/tab，-r 不转义反斜杠

# 4. while 计数
count=0
while [ $count -lt 10 ]; do
    echo "$count"
    ((count++))
done

# 5. until（条件为假时继续，为真时退出）
until ping -c1 -W2 8.8.8.8 > /dev/null 2>&1; do
    echo "Waiting for network..."
    sleep 5
done
echo "Network is up!"
```

---

## S.5 awk / sed / grep：文本处理三件套

### grep：搜索

```bash
$ grep "ERROR" /var/log/syslog
$ grep -c "ERROR" /var/log/syslog        # 计数
$ grep -i "error" file                   # 忽略大小写
$ grep -v "DEBUG" file                   # 排除
$ grep -r "TODO" /opt/code/ --include="*.py"  # 递归搜索
$ grep -A2 -B1 "ERROR" file              # 显示上下文（前1行后2行）
$ grep -E "ERROR|FATAL" file             # 扩展正则（多模式）
$ grep -P '\d{4}-\d{2}-\d{2}' file       # Perl正则（日期格式）
```

### sed：流编辑器

```bash
# 替换（s/查找/替换/标志）
$ sed 's/foo/bar/' file                  # 每行第一个
$ sed 's/foo/bar/g' file                 # 全部
$ sed -i 's/foo/bar/g' file              # 直接修改文件
$ sed -i.bak 's/foo/bar/g' file          # 备份原文件

# 删除行
$ sed '/^$/d' file                       # 删空行
$ sed '1,10d' file                       # 删1-10行
$ sed '/#/d' file                        # 删含#的行

# 打印特定行
$ sed -n '5,10p' file                    # 打印5-10行
$ sed -n '/ERROR/p' file                 # 打印含ERROR的行
```

### awk：列处理

```bash
# awk基本结构：awk '条件 {动作}' 文件
$ awk '{print $1, $3}' access.log        # 打印第1列和第3列
$ awk -F: '{print $1, $3}' /etc/passwd   # 用:分隔
$ awk '$3 > 1000 {print $1}' /etc/passwd # 第3列>1000
$ awk '/root/ {print}' /etc/passwd       # 含root的行

# 高级：统计+排序
$ awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
# 访问最多的10个IP

# 求和
$ awk '{sum += $1} END {print sum}' numbers.txt

# 列操作
$ awk '{print $NF}'                      # 最后一列（NF=列数）
$ awk '{print $(NF-1)}'                  # 倒数第二列
$ awk '{print NR ": " $0}'               # NR=行号, $0=整行
```

---

## S.6 实战：五个生产级脚本

### 脚本1：磁盘空间告警

```bash
#!/bin/bash
set -euo pipefail
THRESHOLD=80

df -h | awk 'NR>1' | while read -r fs size used avail pct mnt; do
    pct_num=${pct%%%}
    if [ "$pct_num" -gt "$THRESHOLD" ]; then
        echo "⚠  $mnt ($fs): ${pct} used (${used}/${size})"
    fi
done
```

### 脚本2：批量创建用户

```bash
#!/bin/bash
set -euo pipefail
INPUT=${1:?Usage: $0 users.csv}
# CSV格式：name,comment,shell

while IFS=',' read -r name comment shell; do
    id "$name" && { echo "$name exists, skip"; continue; }
    password=$(openssl rand -base64 12)
    sudo useradd -m -c "$comment" -s "$shell" "$name"
    echo "$name:$password" | sudo chpasswd
    sudo chage -d 0 "$name"            # 首次登录强制改密码
    echo "Created $name, temp pwd: $password"
done < "$INPUT"
```

### 脚本3：服务健康检查

```bash
#!/bin/bash
set -euo pipefail
SERVICES=(nginx mysql redis)

for svc in "${SERVICES[@]}"; do
    if ! systemctl is-active --quiet "$svc"; then
        echo "⚠  $svc is down! Attempting restart..."
        sudo systemctl restart "$svc"
        sleep 3
        systemctl is-active --quiet "$svc" && echo "✅ $svc recovered" || echo "❌ $svc still down!"
    fi
done
```

### 脚本4：日志清理

```bash
#!/bin/bash
set -euo pipefail
LOG_DIR=${1:-/var/log}
DAYS=${2:-30}

find "$LOG_DIR" -name "*.log" -type f -mtime +"$DAYS" -exec gzip {} \;
find "$LOG_DIR" -name "*.log.gz" -type f -mtime +"$((DAYS * 2))" -delete
echo "Cleaned logs older than $DAYS days in $LOG_DIR"
```

### 脚本5：备份+保留

```bash
#!/bin/bash
set -euo pipefail
SRC=${1:?Usage: $0 /path/to/backup}
RETENTION=${2:-7}

BACKUP_FILE="/backup/$(basename "$SRC")_$(date +%F).tar.gz"
tar czf "$BACKUP_FILE" -C "$(dirname "$SRC")" "$(basename "$SRC")"
echo "Backup: $BACKUP_FILE ($(du -h "$BACKUP_FILE" | cut -f1))"

# 清理旧备份
find /backup -name "$(basename "$SRC")_*.tar.gz" -mtime +"$RETENTION" -delete
echo "Removed backups older than $RETENTION days"
```

---

## 本章小结

### Shell脚本检查清单

```
□ #!/bin/bash
□ set -euo pipefail
□ 变量都用双引号包裹
□ 用户输入校验（:?语法）
□ 危险命令前确认（rm/mv/chmod 等重要操作）
□ 输出重定向到日志
□ function 名字用动词（do_xxx）
□ README写清楚用法
```

### 学习路径

```
新手：bash历史 + 别名 + 简单for循环
初级：grep/sed/awk 基础 + while读写文件
中级：管道组合 + 函数 + 错误处理
高级：set -euo pipefail + 信号陷阱 + 并发
```

---

> **本章字数**：约 7,500 字
> **涉及命令**：set, grep, sed, awk, while, for, if, test, [[ ]], tar, find
