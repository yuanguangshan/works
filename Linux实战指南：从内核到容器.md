# Linux实战指南：从内核到容器

> 摒弃过时老命令，基于 Linux 6.6 LTS / Ubuntu 24.04 / RHEL 9 编写，面向现代云原生环境的 Linux 实操手册。
> 12 章核心内容 + 第 13 章（Linux 6.x+ 内核前沿特性）+ 附录 A（Shell 脚本实战）/B（故障排查速查表）/C（命令速查表）/D（概念索引），涵盖系统基础、文件系统、权限模型、进程管理、网络、systemd、性能调优、日志、安全审计、容器化与内核新特性。

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
| 优势 | 兼容老系统 | 简单一致，资源控制更精准 |

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

---

## 课后练习

1. 概念题：SSH 配置中 PermitRootLogin no 和 PasswordAuthentication no 分别防御什么攻击？

思路：PermitRootLogin no 防止直接暴力破解 root 密码（攻击者不知道用户名，只能盲猜）；PasswordAuthentication no 强制使用密钥，彻底防御密码爆破。

2. 排障题：修改 /etc/ssh/sshd_config 后重启 SSH 失败，如何快速定位语法错误？

思路：sudo sshd -t（测试模式），它会指出错误行号和具体原因。修复后再 systemctl restart sshd。

3. 实操题：使用 auditd 添加一条规则，监控 /etc/sudoers 文件的写入（w）和属性修改（a）操作，并打上标签 sudoers_change。

思路：sudo auditctl -w /etc/sudoers -p wa -k sudoers_change。永久生效需写入 /etc/audit/rules.d/ 下的 .rules 文件。

4. 安全题：SELinux 处于 Enforcing 模式，Nginx 无法写入 /var/www/uploads，ausearch -m avc 显示 denied { write }。请给出两种解决方法。

思路：1. 改标签：chcon -t httpd_sys_rw_content_t /var/www/uploads -R；2. 改布尔值：setsebool -P httpd_unified on（或特定布尔值）。最佳实践是改标签并 restorecon。

5. 对比题：SELinux 和 AppArmor 在策略定义方式上的根本区别是什么？

思路：SELinux 是类型强制（TE）基于 inode 标签，任何文件都有安全上下文；AppArmor 是路径强制，基于程序配置文件允许访问的路径。AppArmor 更简单，SELinux 更精细。

6. 实操题：写一条 Docker 运行命令，要求容器：只读根文件系统、丢弃所有 Capabilities、仅添加 NET_BIND_SERVICE、禁止提权。

思路：docker run --read-only --cap-drop=ALL --cap-add=NET_BIND_SERVICE --security-opt no-new-privileges:true myapp。

7. 排障题：fail2ban 没有封禁任何 IP，但 journalctl -u fail2ban 显示 WARNING 'sshd' not found in 'systemd-journal'。问题出在哪？

思路：fail2ban 的 backend 配置不对。默认 backend=auto 可能选了 pyinotify。应在 /etc/fail2ban/jail.local 中显式设置 backend = systemd 以读取 journald。

8. 综合题：容器中运行 ping 8.8.8.8 报错 Operation not permitted，但宿主机可以。如何在不给容器 --privileged 的情况下修复？

思路：ping 需要 CAP_NET_RAW。启动时添加 --cap-add=NET_RAW 即可。更优雅的做法是不用 ping，用 curl 或 nc 测试连通性。
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

---

## 课后练习

1. 概念题：容器的隔离依赖 Linux 内核的哪两大特性？分别负责什么？

思路：命名空间 (Namespace) 负责“隔离”（看不同的 PID、网络、挂载点等）；Cgroups 负责“限制”（控制 CPU、内存、磁盘 IO 配额）。

2. 排障题：Docker 容器内 top 看到 CPU 使用率，但宿主机 top 看到容器进程占满 2 个核。为什么容器内显示的数字可能和宿主机不一致？

思路：Docker 默认容器内的 top 读取的是容器的 Cgroup 限制（如果没限制则看主机全部核心），而宿主机 top 是物理真实消耗。如果容器内存限制小于物理内存，free -m 显示也可能不一致。

3. 实操题：写出 Dockerfile，使用多阶段构建，将一个 Go 程序编译成静态二进制，并最终放入 scratch 空镜像中运行。

思路：

```dockerfile
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server .
FROM scratch
COPY --from=builder /app/server /server
EXPOSE 8080
CMD ["/server"]
```

4. 对比题：Docker 的 bridge 网络模式和 host 网络模式，在性能和端口管理上有什么优缺点？

思路：bridge 有 NAT 和端口映射开销（docker-proxy），但支持端口复用（宿主机 8080 映射容器 80）。host 无 NAT，性能最高，但容器直接占用宿主机端口，易冲突。

5. 存储题：docker volume create 创建的数据卷和 bind mount（-v /host/path:/container/path）在管理方式上有何不同？

思路：Volume 由 Docker 管理（存储在 /var/lib/docker/volumes/），可通过 docker volume 命令生命周期管理，适合生产数据；Bind mount 直接映射宿主机目录，依赖宿主机文件系统布局，适合开发调试。

6. 实战题：启动一个 Podman 容器，要求以非 root 用户（nobody）运行，并挂载宿主机的 /data 为只读。

思路：podman run --user 65534 -v /data:/data:ro --rm alpine ls /data。注意 rootless 模式下挂载宿主机目录可能需要额外配置用户命名空间映射。

7. 网络题：Docker 容器访问宿主机上的 MySQL（宿主机 IP 是 192.168.1.10，端口 3306），容器内应连接什么地址？

思路：对于 Docker，Linux 容器可连接 172.17.0.1（docker0 网桥的网关）或宿主机实际 IP。Mac/Windows 下需连接 host.docker.internal。最佳实践是不直接用 IP，用容器名称或服务发现。

8. 综合题：如何查看一个运行中容器的 overlay2 存储层在宿主机上的具体路径？

思路：docker inspect <container_id> | jq '.[0].GraphDriver.Data.UpperDir'。这将显示该容器可写层的绝对路径。
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

---

## 课后练习

1. 概念题：执行 `chmod 2755 /shared` 后，权限中的数字 `2` 代表什么？为什么团队共享目录常被设为 `2770`？

思路：`2` 是 setgid（SGID）位。目录设置 SGID 后，其下新建的子文件/子目录会自动继承该目录的属组，而非创建者的主组。共享目录用 `2770` 可确保成员新建的文件都属于共享组，避免因属组错配导致互相无法访问。

2. 实操题：当 `umask=022` 时，新建普通文件和目录的默认权限分别是多少？若希望新建文件默认权限为 `640`，应把 umask 设为多少？

思路：文件默认权限 = `666 & ~umask`，目录 = `777 & ~umask`。umask=022 时，文件为 `644`（rw-r--r--），目录为 `755`（rwxr-xr-x）。要使新文件为 `640`（rw-r-----），需要 `umask=026`（666 & ~026 = 640）。

3. 排障题：用户反馈"明明在 `dev` 组里，却无法读写 `/data/project`"。`ls -ld /data/project` 显示 `drwxrws--- root dev`。请列出至少 3 种可能原因，并说明如何用 `getfacl` 排查。

思路：①存在 ACL 条目压缩了组的有效权限——`getfacl /data/project` 看 `mask` 与 `group:dev` 的实际值；②用户登录会话未刷新组归属，`id` 查看是否真有 dev 组，必要时重新登录或 `newgrp dev`；③SELinux/AppArmor 策略拦截（`ls -Z`、`ausearch -m avc`、`dmesg | grep -i denied`）；④目录被加了不可变等属性（`lsattr /data/project`）。

4. 安全题：用 `chattr +i /etc/passwd` 能否真正防御勒索病毒篡改？它有哪些副作用？如何验证与解除？

思路：`+i`（immutable 位）会让文件对所有人（含 root）变为只读，无法删除、改名、追加或修改，对关键配置确有一定防篡改效果。但副作用明显：连 root 也无法正常写入，会导致 `useradd` 等工具失败；而且多数勒索以普通用户权限加密用户数据（如 `/home`），保护 `/etc/passwd` 并不能挡住对用户文件的破坏。验证：`lsattr /etc/passwd` 看到 `i`；解除：`chattr -i /etc/passwd`。

5. 实操题：用一条命令查找 `/var/log` 下所有大于 100MB、修改时间超过 30 天的 `.log` 文件，并用 `xargs` 配合 `gzip` 压缩（要求正确处理文件名带空格的情况）。

思路：`find /var/log -type f -name '*.log' -size +100M -mtime +30 -print0 | xargs -0 gzip`。`-print0` 与 `xargs -0` 配合，以 NUL 分隔文件名，可安全处理含空格或特殊字符的路径。
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

> **🧭 延伸阅读**：本章讨论的 CPU 调度器（CFS / EEVDF）是写死在内核里的。如果需要修改调度策略，传统方式需要重编译内核并重启服务器——这在生产环境中几乎不可行。**Linux 6.12 引入的 `sched_ext` 框架允许开发者用 eBPF 编写调度策略并热加载**，彻底打破了这一限制。详见**第十三章 13.2 节**。


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

---

## 课后练习

1. 概念题：什么是僵尸进程？僵尸进程占用 CPU 和内存吗？

思路：僵尸进程是已终止但父进程未调用 wait() 回收的进程。它不占用 CPU 和内存（内存已释放），仅占用进程表中的 PID 资源。

2. 排障题：系统出现大量僵尸进程，父进程 PID 为 1234。除了重启父进程，还有什么优雅的方法让父进程回收这些僵尸进程？

思路：向父进程发送 SIGCHLD 信号：kill -SIGCHLD 1234。如果父进程正确捕获该信号并执行 waitpid()，即可回收。

3. 实操题：一个任务在前台运行卡住了（Ctrl+C 无效），你想把它放到后台并暂停，再强制终止它，该怎么做？

思路：Ctrl+Z 暂停当前前台任务 → bg %1 放到后台 → kill %1（或 kill -9 %1）。如果 Ctrl+Z 无效，只能另开终端用 kill -9 <PID>。

4. 对比题：command &、nohup command &、tmux 三者哪个能保证关闭 SSH 终端后任务依然运行且输出可查看？

思路：& 关终端会挂（除非用 disown）。nohup 可以免疫 SIGHUP 并保留输出到 nohup.out，但重连后无法交互。tmux 最完美，重连后直接看到现场。

5. 信号题：kill -15 <PID> 和 kill -9 <PID> 的区别是什么？生产环境优先用哪个？

思路：-15 是 SIGTERM（优雅终止），进程可捕获并执行清理逻辑；-9 是 SIGKILL（强制内核级终止），进程无法捕获。优先用 -15，-9 是最后的杀手锏（防止数据损坏）。

6. 调优题：一个耗时的数据备份脚本在运行，导致前端 Web 响应变慢。如何在不杀进程的情况下降低其 CPU 资源抢占优先级？

思路：用 renice 调高其 nice 值。sudo renice -n 19 -p <PID>（nice 值越高，优先级越低）。

7. 监控题：系统平均负载（load average）长期高于 CPU 核心数，但 top 看到的 CPU 使用率只有 20%。最可能的原因是什么？

思路：高 I/O 等待（磁盘瓶颈）。大量进程处于不可中断睡眠（D 状态）等待磁盘读写。执行 iostat -x 1 查看 %util 和 await 即可验证。

8. 实战题：如何在 tmux 中创建一个名为 train 的会话，在后台运行 AI 训练脚本，然后安全地断开连接？

思路：tmux new -s train → 执行 python train.py → 按 Ctrl+B 再按 D 脱离。重连时 tmux attach -t train。
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

> **🧭 延伸阅读**：本章讨论的 TCP 拥塞控制默认算法是 CUBIC（基于丢包），但现代跨数据中心场景下，**Google 推出的 BBR 算法通过主动探测带宽和延迟**，吞吐量可提升 15%-20%。Linux 4.9+ 已支持 BBR，6.x 版本引入的 BBR v3 进一步优化了多流公平性。详见**第十三章 13.3 节**。


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


---

## 5.9 实战：手工搭建容器网络（veth pair + netns）
**一句话定义**：veth pair 是 Linux 内核中的虚拟以太网对，像一根“网线”的两端——一端在容器命名空间，另一端在宿主机，是 Docker/Podman 容器网络的底层基石。
### 为什么必须手工做一遍？
Docker 帮你自动创建了 veth pair、网桥、路由、NAT 规则——你只看到了结果，没看到过程。手工做一遍，你就能回答三个灵魂问题：
1. 容器为什么能访问外网？
2. 宿主机为什么能 ping 通容器？
3. 为什么容器之间即使不在同一个网桥也能通信？
### 实战：两个 netns 通过 veth pair 通信
# 1. 创建两个网络命名空间（模拟两个容器）
$ sudo ip netns add container1
$ sudo ip netns add container2
# 2. 创建 veth pair（一根虚拟网线的两端）
$ sudo ip link add veth1 type veth peer name veth2
# 3. 把两端分别放进两个命名空间
$ sudo ip link set veth1 netns container1
$ sudo ip link set veth2 netns container2
# 4. 给两端配 IP（现在它们在同一子网）
$ sudo ip netns exec container1 ip addr add 10.0.1.1/24 dev veth1
$ sudo ip netns exec container2 ip addr add 10.0.1.2/24 dev veth2
# 5. 启动两端
$ sudo ip netns exec container1 ip link set veth1 up
$ sudo ip netns exec container2 ip link set veth2 up
# 6. ping 通！
$ sudo ip netns exec container1 ping 10.0.1.2
64 bytes from 10.0.1.2: icmp_seq=1 ttl=64 time=0.035 ms
# ✅ 两个“容器”已经可以通信了
升级：连接外网（添加网桥 + NAT）
# 1. 创建网桥（模拟 docker0 / cni0）
$ sudo ip link add br0 type bridge
$ sudo ip link set br0 up
# 2. 把 container1 的 veth1 从直连改为接入网桥
$ sudo ip netns exec container1 ip link set veth1 down
$ sudo ip link set veth1 netns 1  # 移回宿主机
$ sudo ip link set veth1 master br0
$ sudo ip link set veth1 up
# 3. 为网桥配 IP（容器的网关）
$ sudo ip addr add 172.18.0.1/16 dev br0
# 4. 在容器内设置网关
$ sudo ip netns exec container1 ip route add default via 172.18.0.1
# 5. 开启宿主机 NAT 转发（让容器能访问外网）
$ sudo sysctl -w net.ipv4.ip_forward=1
$ sudo iptables -t nat -A POSTROUTING -s 172.18.0.0/16 ! -o br0 -j MASQUERADE
# 6. 现在容器能访问外网了！
$ sudo ip netns exec container1 ping 8.8.8.8
64 bytes from 8.8.8.8: icmp_seq=1 ttl=118 time=15.2 ms
关键理解
组件 对应 Docker 概念
ip netns add 容器的网络命名空间
veth pair 容器内的 eth0 ↔ 宿主机上的 vethXXXXX
br0 docker0 网桥（或 CNI 网桥）
MASQUERADE 容器访问外网时的 SNAT
清空实验环境：
$ sudo ip netns del container1
$ sudo ip netns del container2
$ sudo ip link del br0


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

---

## 课后练习

1. 概念题：/24 子网掩码的十进制表示是什么？该子网最多容纳多少台有效主机？

思路：255.255.255.0，有效主机数为 2^(32-24) - 2 = 254 个。

2. 排障题：ping 8.8.8.8 能通，但 ping www.baidu.com 显示 Temporary failure in name resolution。你的排查步骤是什么？

思路：DNS 问题。先 cat /etc/resolv.conf 看 nameserver（可能是 127.0.0.53），再 nslookup baidu.com 8.8.8.8 测外网 DNS，最后检查 systemd-resolved 状态：resolvectl status。

3. 实操题：查看系统当前所有监听的 TCP 端口，并显示对应的进程名，请写出 ss 命令。

思路：ss -tlnp（-t TCP，-l 监听，-n 不解析服务名，-p 显示进程）。netstat -tulnp 也行但已淘汰。

4. 抓包题：想抓取 eth0 网卡上源 IP 为 192.168.1.100 且目标端口为 80 的 TCP 包，并保存为 web.pcap，请写出 tcpdump 命令。

思路：sudo tcpdump -i eth0 -nn 'src host 192.168.1.100 and tcp dst port 80' -w web.pcap。

5. 配置题：Ubuntu 24.04 使用 Netplan，请写出将 eth0 配置为静态 IP 10.0.0.10/24，网关 10.0.0.1，DNS 8.8.8.8 的 YAML 配置。

思路：

```yaml
network:
  ethernets:
    eth0:
      dhcp4: no
      addresses: [10.0.0.10/24]
      routes: [{ to: default, via: 10.0.0.1 }]
      nameservers: { addresses: [8.8.8.8] }
  version: 2
```

6. 安全题：为了防止 SSH 暴力破解，请写出两条 nftables 规则：一条放行已建立连接，一条限制 SSH 端口（2222）的新连接频率。

思路：nft add rule inet filter input ct state established,related accept；nft add rule inet filter input tcp dport 2222 meter ssh-limit { ip saddr limit rate 10/minute } accept（配合 drop 策略）。

7. 排障题：curl -v https://example.com 卡在 TLS handshake，ping 通 IP 但延迟较高。可能的原因是什么？

思路：可能是 MTU 问题（PMTUD 黑洞）导致 TLS 握手大包被丢弃。尝试 ping -M do -s 1472 <IP> 测试是否分片被禁止，或检查防火墙是否放行了 ICMP need-frag 包。

8. 对比题：ss 和 netstat 获取连接状态信息的原理有何不同？为何现代推荐用 ss？

思路：netstat 遍历 /proc/net/（慢，文件系统开销）；ss 直接通过 netlink 内核接口读取（快，开销小）。
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

---

## 课后练习

1. 概念题：/etc/passwd 和 /etc/shadow 文件权限分别是什么？为什么 /etc/shadow 只能 root 读取？

思路：/etc/passwd 是 644，/etc/shadow 是 640（属 root 组 shadow）。因为 shadow 存储加密密码哈希和过期策略，泄露后极易被暴力破解。

2. 实操题：创建一个无法交互登录的系统服务用户 myapp，指定家目录为 /opt/myapp，Shell 设为 /usr/sbin/nologin。

思路：sudo useradd -r -d /opt/myapp -s /usr/sbin/nologin myapp（-r 创建系统用户 UID<1000）。

3. 安全题：新入职员工 bob 需要加入 docker 和 dev 组，但不能覆盖其原有的附加组，如何操作？

思路：sudo usermod -aG docker,dev bob（必须加 -a，否则覆盖）。

4. 策略题：公司要求所有普通用户密码必须 90 天修改一次，提前 7 天提醒。请写出对用户 alice 设置此策略的 chage 命令。

思路：sudo chage -M 90 -W 7 alice。

5. 排障题：sudo -l 显示 User alice is not allowed to run sudo on host，但 alice 明确在 sudo 组里。哪里可能出问题了？

思路：sudo 的权限由 /etc/sudoers 决定，通常使用 %sudo ALL=(ALL:ALL) ALL。检查组名是否确实是 sudo（有些是 wheel）。或者 sudo 的 PAM 配置（/etc/pam.d/sudo）限制了一致性。

6. 对比题：主组（Primary Group）和附加组（Supplementary Group）在文件创建时的权限继承上有何区别？

思路：用户创建文件时，默认所属组是主组；附加组只用于判断访问权限，不会成为新文件的默认组（除非父目录有 setgid 位）。

7. 实操题：批量锁定所有一个月内未登录的普通用户（lastlog 日志显示 **Never logged in**）。

思路：lastlog -t 30 列出最近 30 天未登录的，结合 awk 提取用户名，usermod -L 锁定。注意筛选 $3>=1000 避免锁定系统用户。

8. 综合题：如何配置 PAM（/etc/security/limits.conf）限制 www-data 用户的最大进程数为 100，最大打开文件数为 65535？

思路：

```
www-data soft nproc 100
www-data hard nproc 100
www-data soft nofile 65535
www-data hard nofile 65535
```
# 第七章：服务管理与systemd（增强版）

> **本章定位**：systemd是现代Linux的服务管理器，理解Unit文件编写、依赖解析、资源限制和journalctl是运维基本功。本章从"会用systemctl"升级到"精通systemd架构"。

---

## 7.1 systemd概述：PID=1的帝国

**一句话定义**：systemd是PID=1的初始化系统，不仅是服务管理器，更是整个用户空间的"操作系统"——管理挂载、定时器、套接字、设备、交换分区，甚至用户的登录会话。

### 为什么systemd取代了SysVinit？

```bash
# SysVinit的问题：串行启动
# 服务A → 服务B → 服务C → ... → 启动完成（耗时2-3分钟）

# systemd的解决：并行+按需激活
# 所有无依赖的服务同时启动
# socket激活：服务未启动也能接收连接
# D-Bus激活：服务首次被调用时才启动
```

**systemd的核心设计哲学**：
- **并行启动**：通过依赖分析图，无依赖的服务同时启动
- **按需激活**：socket/dbus/path/timer激活，减少常驻进程
- **统一配置**：所有管理对象都是Unit文件，格式一致
- **资源隔离**：原生集成cgroups，无需额外配置
- **结构化日志**：journald替代分散的syslog文件

### 系统启动链条深度解析

```bash
# 1. 启动耗时分析
$ systemd-analyze
Startup finished in 2.345s (kernel) + 5.678s (userspace) = 8.023s
graphical.target reached after 5.890s in userspace

# 2. 最慢的服务（优化重点）
$ systemd-analyze blame | head -10
8.123s NetworkManager-wait-online.service
1.234s snapd.service
0.567s apparmor.service
0.345s docker.service
0.234s mysql.service

# 3. 关键路径分析（为什么启动慢）
$ systemd-analyze critical-chain
graphical.target @5.890s
└─multi-user.target @5.890s
  └─nginx.service @4.500s +1.234s
    └─network.target @4.200s +300ms
      └─network-pre.target @4.190s +10ms
        └─firewalld.service @3.500s +680ms
          └─basic.target @3.200s
# ↑ 这条链上的任何服务慢了，都会拖慢整体启动

# 4. 生成依赖图（可视化）
$ systemd-analyze dot | dot -Tsvg > boot-deps.svg
# 用浏览器打开SVG，可以看到完整的依赖关系图
```

### systemd的"帝国版图"

```bash
# systemd管理的不仅是.service，而是所有Unit类型
$ systemctl --type=help
Available unit types:
  service    # 服务进程（nginx, sshd）
  socket     # 套接字（用于socket激活）
  target     # 目标（类似runlevel，一组Unit的集合）
  device     # 内核设备（/dev/sda等）
  mount      # 挂载点（/home, /var）
  automount  # 自动挂载（按需挂载）
  swap       # 交换分区
  timer      # 定时器（替代cron）
  path       # 路径监控（文件变化触发）
  slice      # 资源切片（cgroups层级）
  scope      # 作用域（外部创建的进程组）
```

---

## 7.2 Unit文件编写：从入门到精通

### Unit文件位置（优先级从高到低）

```bash
/etc/systemd/system/          # 管理员自定义（最高优先级）
/run/systemd/system/          # 运行时生成
/usr/lib/systemd/system/      # 软件包安装（最低优先级）

# 查看Unit文件的完整路径
$ systemctl cat nginx.service
# /lib/systemd/system/nginx.service
[Unit]
Description=A high performance web server
...
```

### 实战：生产级Service文件

```bash
$ sudo cat > /etc/systemd/system/myapp.service <<'EOF'
[Unit]
Description=My Application API Server
Documentation=https://myapp.io/docs
After=network-online.target postgresql.service redis.service
Wants=redis.service
Requires=postgresql.service
BindsTo=caddy.service
StartLimitIntervalSec=60
StartLimitBurst=3

[Service]
Type=notify
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
Environment="RUST_LOG=info"
Environment="DATABASE_URL=postgres://localhost/myapp"
EnvironmentFile=-/etc/myapp/env          # -表示文件不存在不报错
ExecStartPre=/opt/myapp/bin/migrate      # 启动前执行数据库迁移
ExecStart=/opt/myapp/bin/server --config /etc/myapp/config.toml
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/opt/myapp/bin/graceful-shutdown
TimeoutStartSec=30
TimeoutStopSec=60                        # 优雅关闭最多等60秒
Restart=on-failure
RestartSec=5
RestartSteps=5                           # 每次重启间隔增加5秒
RestartMaxDelaySec=60                    # 最大间隔60秒
NotifyAccess=main                        # 只有主进程能发sd_notify

# 资源限制（cgroups v2）
MemoryMax=512M
MemorySwapMax=0                          # 禁用swap
CPUQuota=200%                            # 最多2核
TasksMax=100                             # 最多100个线程
IOWeight=100                             # IO权重

# 安全加固
NoNewPrivileges=true
ProtectSystem=strict                     # 只读挂载/usr, /boot, /etc
ProtectHome=true                         # 不可访问/home
PrivateTmp=true                          # 独立/tmp
PrivateDevices=true                      # 仅保留/dev/null, zero等
ProtectKernelTunables=true               # 不可改/proc/sys
ProtectKernelModules=true                # 不可加载内核模块
ProtectControlGroups=true                # 不可操作cgroups
RestrictRealtime=true                    # 不可创建实时调度进程
RestrictSUIDSGID=true                    # 不可创建setuid文件
LockPersonality=true                     # 不可改变执行域
MemoryDenyWriteExecute=true              # 不可创建可写可执行内存
SystemCallFilter=@system-service         # 只允许系统服务syscall
SystemCallErrorNumber=EPERM

[Install]
WantedBy=multi-user.target
EOF

$ sudo systemctl daemon-reload
$ sudo systemctl enable --now myapp
```

### Type详解：选择正确的服务类型

| Type | 行为 | 适用场景 | 示例 |
|------|------|----------|------|
| `simple` | ExecStart启动=就绪 | 前台运行的现代服务 | 大多数Go/Rust程序 |
| `forking` | 父进程退出=就绪 | 传统daemon（fork两次） | nginx, Apache |
| `oneshot` | 进程退出=完成 | 一次性任务 | 数据库迁移、备份 |
| `notify` | sd_notify()通知=就绪 | 支持systemd通知的服务 | **现代应用最佳实践** |
| `dbus` | D-Bus名称获取=就绪 | D-Bus服务 | 桌面服务 |
| `idle` | 所有作业完成后启动 | 低优先级后台任务 | 延迟启动的监控 |

```bash
# Type=notify 的代码示例（C语言）
#include <systemd/sd-daemon.h>

int main() {
    // 初始化...
    
    // 通知systemd：我已就绪
    sd_notify(0, "READY=1");
    
    // 主循环...
    
    // 发送状态更新
    sd_notify(0, "STATUS=Processing request #1234");
    
    // 优雅关闭时
    sd_notify(0, "STOPPING=1");
    return 0;
}

# Go语言示例
import "github.com/coreos/go-systemd/daemon"
daemon.SdNotify(false, daemon.SdNotifyReady)
```

### 依赖指令深度对比

```bash
# 关键区别：带不带启动保证？

[Unit]
# Requires = 硬依赖，对方必须启动，否则自己启动失败
Requires=postgresql.service
# 但！如果postgresql后来挂了，myapp不会自动停止

# BindsTo = 生命周期绑定，对方停止，自己也停止
BindsTo=caddy.service
# 适合：sidecar模式，主服务停，代理也停

# Wants = 软依赖，尽力启动对方，但失败也能启动
Wants=redis.service
# 适合：缓存服务，有最好，没有也能跑

# After = 排序关系，只在对方启动完成后启动
After=network-online.target
# ⚠ After ≠ Requires！只排序，不保证对方已启动

# Before = 先于对方启动
Before=shutdown.target
# 适合：需要在系统关机前完成清理的服务
```

### 依赖实战：常见错误

```bash
# ❌ 错误：只用After，没用Requires
[Unit]
After=postgresql.service
# 问题：postgresql没启动，myapp也会启动，然后报错

# ✅ 正确：After + Requires/Wants
[Unit]
After=postgresql.service
Requires=postgresql.service

# ❌ 错误：循环依赖
# A.service Requires=B
# B.service Requires=A
# 结果：systemd检测到循环，两个都启动失败

# ✅ 正确：单向依赖
# A.service Requires=B
# B.service 无依赖
```

---

## 7.3 systemctl实战：从基础到高级

### 生命周期管理

```bash
# 基础操作
$ sudo systemctl start nginx
$ sudo systemctl stop nginx
$ sudo systemctl restart nginx          # 先stop再start
$ sudo systemctl reload nginx           # 发送SIGHUP，不中断服务
$ sudo systemctl try-reload-or-restart nginx  # 支持reload则reload，否则restart

# 状态查看（深度）
$ systemctl status nginx
● nginx.service - A high performance web server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-07-02 09:00:00 CST; 2h ago
       Docs: man:nginx(8)
    Process: 1234 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 1235 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 1236 (nginx)
      Tasks: 5 (limit: 9445)
     Memory: 5.2M
        CPU: 123ms
     CGroup: /system.slice/nginx.service
             ├─1236 "nginx: master process /usr/sbin/nginx"
             ├─1237 "nginx: worker process"
             └─1238 "nginx: worker process"

# 关键字段：
# Loaded: enabled = 开机自启, preset = 发行版默认策略
# Active: running = 运行中, (running) = 主进程在跑
# Main PID: 主进程ID（forking类型时特别重要）
# Tasks: 当前线程数 / 限制
# Memory: 实际内存占用
# CGroup: 完整的cgroups路径

# 查看特定属性
$ systemctl show nginx | grep -E 'MainPID|MemoryCurrent|CPUUsageNSec|Restart'
MainPID=1236
MemoryCurrent=5242880
CPUUsageNSec=123000000
Restart=no

# 查看所有属性
$ systemctl show nginx --all | less
```

### 高级操作：mask, edit, isolate

```bash
# 1. mask - 彻底禁用（连手动启动都不行）
$ sudo systemctl mask nginx
# 创建 /etc/systemd/system/nginx.service → /dev/null 的符号链接
# 用途：防止依赖误启动，或临时禁用有问题的服务

$ sudo systemctl unmask nginx          # 恢复

# 2. edit - 安全编辑（创建override文件）
$ sudo systemctl edit nginx
# 自动创建 /etc/systemd/system/nginx.service.d/override.conf
# 只覆盖需要的字段，原文件不动

# 3. 查看合并后的完整配置
$ systemctl cat nginx
# /lib/systemd/system/nginx.service
[Unit]
...
# /etc/systemd/system/nginx.service.d/override.conf
[Service]
Environment="MY_VAR=override"

# 4. isolate - 切换到特定target（类似runlevel）
$ sudo systemctl isolate multi-user.target
# 停止所有不属于multi-user.target的服务
# ⚠ 危险！可能停止图形界面

# 5. 紧急模式（最小化启动）
$ sudo systemctl emergency
# 只启动基本系统+shell，用于修复
$ sudo systemctl rescue
# 单用户模式，保留更多服务
```

### 启动失败排查

```bash
# 1. 查看失败原因
$ systemctl status failed-service
# 看 "Main PID: ... (code=exited, status=1/FAILURE)"
# 和最后的日志行

# 2. 查看完整日志
$ journalctl -u failed-service -n 50 --no-pager

# 3. 检查配置文件语法
$ systemd-analyze verify /etc/systemd/system/myapp.service
# 会报告语法错误和潜在问题

# 4. 手动测试ExecStart
$ sudo -u myapp /opt/myapp/bin/server
# 直接运行看报错

# 5. 检查环境变量
$ systemctl show myapp -p Environment
# 对比手动运行的 env 输出

# 6. 权限问题
$ ls -la /opt/myapp/bin/server
$ getfacl /opt/myapp/config.toml
# 确认用户有读取权限

# 7. 资源限制导致的OOM
$ journalctl -u myapp | grep -i "killed process\|oom"
# 看是否被OOM Killer杀掉
```

---

## 7.4 Target：现代运行级别

### Target vs SysV Runlevel

| Runlevel | Target | 说明 |
|----------|--------|------|
| 0 | poweroff.target | 关机 |
| 1 | rescue.target | 单用户救援 |
| 3 | multi-user.target | 多用户命令行 |
| 5 | graphical.target | 图形界面 |
| 6 | reboot.target | 重启 |

```bash
# 查看当前target
$ systemctl get-default
graphical.target

# 切换默认target
$ sudo systemctl set-default multi-user.target
# 下次启动进入命令行模式

# 临时切换（当前会话）
$ sudo systemctl isolate multi-user.target

# 查看target包含哪些服务
$ systemctl list-dependencies multi-user.target
multi-user.target
● ├─basic.target
● ├─nginx.service
● ├─sshd.service
● ├─mysql.service
● └─network.target

# 自定义target（服务分组）
$ sudo cat > /etc/systemd/system/web-stack.target <<'EOF'
[Unit]
Description=Web Application Stack
After=network.target
Requires=nginx.service php-fpm.service redis.service
Wants=mysql.service
EOF

$ sudo systemctl enable web-stack.target
$ sudo systemctl start web-stack.target
# 一键启动整个Web栈
```

---

## 7.5 systemd定时器：cron的现代替代

### 为什么用Timer替代cron？

| 维度 | cron | systemd timer |
|------|------|----------------|
| 日志 | 分散在/var/log/syslog | 统一在journald |
| 失败处理 | 无内置重试 | OnFailure= 可触发恢复 |
| 环境变量 | 极度干净（几乎为空） | 继承systemd环境 |
| 依赖管理 | 无 | 可依赖其他服务 |
| 精确时间 | 分钟级 | 秒级（.timer文件） |
| 持久化 | 错过就错过 | Persistent=true 补偿执行 |
| 随机延迟 | 无 | RandomizedDelaySec 防惊群 |

### Timer实战：三种触发模式

```bash
# 模式1：日历时间（类似cron）
$ sudo cat > /etc/systemd/system/backup.timer <<'EOF'
[Unit]
Description=Daily backup timer

[Timer]
OnCalendar=*-*-* 02:00:00           # 每天凌晨2点
OnCalendar=Mon *-*-* 03:00:00       # 每周一3点
OnCalendar=*-*-01 04:00:00          # 每月1日4点
Persistent=true                      # 如果错过（关机），开机后补偿执行
RandomizedDelaySec=300               # 随机延迟0-300秒，防惊群

[Install]
WantedBy=timers.target
EOF

# 模式2：单调时钟（基于系统状态）
$ sudo cat > /etc/systemd/system/cleanup.timer <<'EOF'
[Unit]
Description=Cleanup timer

[Timer]
OnBootSec=10min                      # 启动后10分钟
OnUnitActiveSec=1h                   # 上次执行后1小时
OnUnitInactiveSec=30min              # 上次结束后30分钟

[Install]
WantedBy=timers.target
EOF

# 模式3：文件监控（路径触发）
$ sudo cat > /etc/systemd/system/sync-on-change.path <<'EOF'
[Unit]
Description=Watch config for changes

[Path]
PathChanged=/etc/nginx/nginx.conf    # 文件变化时触发
PathModified=/etc/nginx/conf.d/      # 目录内任何文件变化

[Install]
WantedBy=multi-user.target
EOF

$ sudo cat > /etc/systemd/system/sync-on-change.service <<'EOF'
[Service]
Type=oneshot
ExecStart=/usr/local/bin/sync-nginx-config.sh
EOF
```

### Timer管理与监控

```bash
# 查看所有定时器
$ systemctl list-timers
NEXT                        LEFT    LAST                        PASSED  UNIT
Wed 2026-07-03 02:00:00 CST 5h left Tue 2026-07-02 02:00:00 CST 18h ago backup.timer
Wed 2026-07-03 03:00:00 CST 6h left n/a                         n/a     cleanup.timer

# 查看特定timer下次触发时间
$ systemctl show backup.timer -p NextElapseUSecRealtime

# 手动触发（测试用）
$ sudo systemctl start backup.service

# 查看timer日志
$ journalctl -u backup.timer
$ journalctl -u backup.service

# 定时器状态
$ systemctl status backup.timer
```

---

## 7.6 journalctl：结构化日志的瑞士军刀

### journald存储引擎深度解析

```bash
# journald的三种存储模式
$ cat /etc/systemd/journald.conf
[Journal]
Storage=auto                         # auto | persistent | volatile | none
# auto: /var/log/journal/存在则持久化，否则内存
# persistent: 强制持久化到/var/log/journal/
# volatile: 仅内存，/run/log/journal/
# none: 丢弃（但可转发到syslog）

# 生产环境推荐配置
$ sudo mkdir -p /etc/systemd/journald.conf.d/
$ sudo cat > /etc/systemd/journald.conf.d/99-production.conf <<'EOF'
[Journal]
Storage=persistent
SystemMaxUse=2G                    # 最大占用2GB
SystemKeepFree=1G                  # 保留1GB空闲
SystemMaxFileSize=128M             # 单文件128MB
SystemMaxFiles=50                  # 最多50个文件
MaxRetentionSec=30day              # 保留30天
Compress=yes                       # 压缩旧日志
Seal=yes                           # 启用FSS密封（防篡改）
RateLimitIntervalSec=30s           # 速率限制窗口
RateLimitBurst=20000               # 窗口内最大条数
ForwardToSyslog=no                 # 不重复转发（节省IO）
EOF

$ sudo systemctl restart systemd-journald
```

### journalctl高级查询

```bash
# 1. 基础过滤
$ journalctl -u nginx                # 服务日志
$ journalctl -u nginx -f             # 实时跟踪
$ journalctl -k                      # 内核日志
$ journalctl -b                      # 本次启动
$ journalctl -b -1                   # 上次启动（排查崩溃神器）
$ journalctl --since "2026-07-02 08:00" --until "2026-07-02 12:00"

# 2. 优先级过滤
$ journalctl -p err                  # ERROR及以上
# emerg(0), alert(1), crit(2), err(3), warning(4), notice(5), info(6), debug(7)
$ journalctl -p warning..err         # 范围过滤

# 3. 字段过滤（结构化日志的威力）
$ journalctl _SYSTEMD_UNIT=nginx.service
$ journalctl _PID=1234
$ journalctl _UID=1000               # 特定用户
$ journalctl _COMM=python3           # 特定命令
$ journalctl _EXE=/usr/bin/python3   # 特定可执行文件
$ journalctl _AUDIT_SESSION=2        # 特定登录会话

# 4. 组合查询
$ journalctl -u nginx _PID=1234 --since today
$ journalctl -u nginx -g "error|failed|timeout"  # 正则匹配消息内容

# 5. 输出格式
$ journalctl -o short                # 默认
$ journalctl -o verbose              # 完整字段
$ journalctl -o json                 # JSON格式（适合程序处理）
$ journalctl -o json-pretty          # 格式化JSON

# 6. 日志分析
$ journalctl --since today | awk '{print $5}' | sort | uniq -c | sort -rn | head
# 看今天哪个服务日志最多

$ journalctl -p err --since "1 hour ago" | awk '{print $5}' | sort | uniq -c | sort -rn
# 最近1小时错误最多的服务

# 7. 导出与归档
$ journalctl -u nginx --since "2026-06-01" --until "2026-07-01" > nginx-june.log
$ journalctl -u nginx --output=json > nginx-logs.json
$ journalctl --list-boots            # 查看所有启动记录
$ journalctl -b -2                   # 上上次启动的日志
```

### 日志持久化与故障排查

```bash
# 场景：系统崩溃后分析原因
# 1. 确保持久化已启用
$ ls -la /var/log/journal/
drwxr-sr-x 2 root systemd-journal 4096 Jul  2 09:00 .

# 2. 查看上次启动的日志
$ journalctl -b -1 -p err
# 看上次启动时的错误

$ journalctl -b -1 -k | tail -50
# 看上次启动的内核日志最后50行（可能是panic信息）

# 3. 查看特定时间段的日志
$ journalctl --since "2026-07-02 14:00:00" --until "2026-07-02 14:05:00"
# 精确定位故障发生时间

# 4. 磁盘空间管理
$ journalctl --disk-usage
Archived and active journals take up 1.5G in the file system.

$ sudo journalctl --vacuum-size=500M   # 保留最近500MB
$ sudo journalctl --vacuum-time=7d     # 保留最近7天
$ sudo journalctl --vacuum-files=5     # 保留最近5个文件

# 5. 日志完整性验证（防篡改）
$ sudo journalctl --verify
# 检查FSS密封的完整性
```

---

## 7.7 Slice与Scope：cgroups的资源层级

### Slice架构详解

```bash
# 默认的三层Slice结构
$ systemd-cgls
Control group /:
-.slice
├─user.slice
│ └─user-1000.slice
│   ├─session-2.scope
│   │ ├─1234 sshd: alice@pts/0
│   │ ├─1235 -bash
│   │ └─1456 vim
│   └─user@1000.service
│     └─app.slice
│       ├─firefox.service
│       └─code.service
├─system.slice
│ ├─nginx.service
│ ├─sshd.service
│ ├─mysql.service
│ └─cron.service
└─machine.slice
  └─docker-abc123.scope
    ├─container1.service
    └─container2.service

# 层级含义：
# -.slice        根
# ├── system.slice   系统服务（PID 1的子进程）
# ├── user.slice     用户会话
# └── machine.slice  虚拟机/容器（systemd-nspawn, Docker with systemd）
```

### 实战：自定义Slice限制资源

```bash
# 创建数据库服务的专用Slice
$ sudo cat > /etc/systemd/system/db.slice <<'EOF'
[Unit]
Description=Database Slice
Before=slices.target

[Slice]
# CPU限制
CPUQuota=400%                          # 最多4核
CPUWeight=200                          # 相对权重（默认100）

# 内存限制
MemoryMax=8G                           # 硬限制
MemoryHigh=6G                          # 软限制（尽力回收）
MemorySwapMax=2G                       # swap限制

# IO限制
IOWeight=300                           # IO权重
IOReadBandwidthMax=/dev/sda 100M       # 读带宽限制
IOWriteBandwidthMax=/dev/sda 50M       # 写带宽限制

# 任务数限制
TasksMax=500                           # 最多500个线程
EOF

# 将服务放入Slice
$ sudo cat > /etc/systemd/system/mysql.service.d/50-slice.conf <<'EOF'
[Service]
Slice=db.slice
EOF

$ sudo systemctl daemon-reload
$ sudo systemctl restart mysql

# 查看资源使用
$ systemctl show db.slice | grep -E 'Memory|CPU|IO'
$ systemd-cgtop                        # 实时查看各Slice资源使用
```

### systemd-run：临时任务的资源隔离

```bash
# 运行一次性任务，自动创建Scope并限制资源
$ sudo systemd-run --scope \
    -p MemoryMax=1G \
    -p CPUQuota=100% \
    -p IOWeight=10 \
    --user \
    ./heavy_batch_job.sh

# 效果：
# 1. 自动创建 run-u12345.scope
# 2. 应用指定的资源限制
# 3. 任务结束后Scope自动清理
# 4. 日志自动进入journald

# 对比Docker：
# docker run --memory=1g --cpus=1 ...
# systemd-run -p MemoryMax=1G -p CPUQuota=100% ...
# 功能类似，但无需镜像，直接运行宿主机二进制
```

---

## 7.8 Socket激活：按需启动的艺术

### 为什么需要Socket激活？

```bash
# 传统方式：所有服务开机即启动
# sshd, nginx, mysql, postgresql, redis... 全部常驻内存
# 即使没人连接，也占用资源

# Socket激活方式：
# 1. systemd先创建socket（监听端口），但不启动服务
# 2. 第一个连接到达时，systemd才启动服务
# 3. 服务通过socket接收连接，用户无感知

# 优势：
# - 减少常驻进程数量
# - 加快系统启动速度
# - 服务崩溃后自动重启，不丢连接
# - 零停机更新（新旧服务共享socket）
```

### Socket激活实战

```bash
# 1. 创建socket文件
$ sudo cat > /etc/systemd/system/myapp.socket <<'EOF'
[Unit]
Description=MyApp Socket

[Socket]
ListenStream=8080                    # TCP端口
ListenStream=/run/myapp.sock         # Unix域套接字（可选）
Backlog=128                          # 连接队列长度
BindIPv6Only=both                    # 同时监听IPv4和IPv6
NoDelay=true                         # 禁用Nagle算法

[Install]
WantedBy=sockets.target
EOF

# 2. 修改service文件
$ sudo cat > /etc/systemd/system/myapp.service <<'EOF'
[Unit]
Description=MyApp Service
Requires=myapp.socket

[Service]
Type=notify
ExecStart=/opt/myapp/bin/server
# 注意：不需要监听端口，systemd会传递socket（fd 3）
EOF

# 3. 启用（只启用socket，不启用service）
$ sudo systemctl enable myapp.socket
$ sudo systemctl start myapp.socket

# 4. 验证
$ ss -tlnp | grep 8080
LISTEN 0 128 *:8080 *:* users:(("systemd",pid=1,fd=42))
# 端口由systemd(PID=1)持有！

$ curl http://localhost:8080
# 第一次请求触发myapp.service启动
$ systemctl status myapp
# 服务已启动
```

---

## 7.9 依赖分析与调试

### 启动失败深度排查

```bash
# 1. 验证Unit文件
$ systemd-analyze verify /etc/systemd/system/myapp.service
/etc/systemd/system/myapp.service:8: Unknown lvalue 'ExecStartt' in section 'Service'
# 发现拼写错误

# 2. 查看依赖树
$ systemctl list-dependencies myapp.service
myapp.service
● ├─postgresql.service
● ├─redis.service
● └─network.target

# 3. 反向依赖：谁依赖我？
$ systemctl list-dependencies --reverse nginx.service
nginx.service
● ├─web-stack.target
● └─multi-user.target

# 4. 查看启动顺序
$ systemd-analyze critical-chain myapp.service
myapp.service @10.234s
└─postgresql.service @8.500s +1.500s
  └─network.target @8.200s +300ms
    └─NetworkManager.service @5.000s +3.200s

# 5. 调试启动问题
$ sudo systemctl start myapp.service
Job for myapp.service failed because the control process exited with error code.
$ journalctl -u myapp.service -n 20
# 查看具体错误

# 6. 手动模拟systemd环境
$ sudo systemd-run --unit=test --service-type=simple /opt/myapp/bin/server
# 快速测试，无需写Unit文件
```

---

## 7.10 实战：systemd 生产 checklist

```bash
#!/bin/bash
# /usr/local/bin/systemd-audit.sh
# systemd配置审计脚本

echo "=== systemd 生产审计 $(date) ==="

# 1. 检查开机失败的服务
echo -e "\n--- 失败的服务 ---"
systemctl --failed --no-pager

# 2. 检查无限制的服务
echo -e "\n--- 无资源限制的服务 ---"
systemctl show --property=MemoryMax,CPUQuota,TasksMax --all | \
    grep -E "MemoryMax=$|CPUQuota=$|TasksMax=$" | head -20

# 3. 检查未启用持久化日志
echo -e "\n--- journald配置 ---"
grep -E "^Storage=" /etc/systemd/journald.conf /etc/systemd/journald.conf.d/*.conf 2>/dev/null

# 4. 检查定时器状态
echo -e "\n--- 定时器状态 ---"
systemctl list-timers --no-pager

# 5. 检查高权限服务
echo -e "\n--- 以root运行的服务 ---"
systemctl show --property=User,ExecStart --all | grep -B1 "User=root" | grep ExecStart

# 6. 检查未加固的服务
echo -e "\n--- 未设置安全加固的服务 ---"
systemctl show --property=NoNewPrivileges,ProtectSystem,PrivateTmp --all | \
    grep "NoNewPrivileges=no$\|ProtectSystem=no$\|PrivateTmp=no$" | head -20

echo -e "\n=== 审计完成 ==="
```

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

---

## 课后练习

1. 概念题：systemd 中 Wants 和 Requires 依赖指令的核心区别是什么？

思路：Requires 是强依赖，依赖服务启动失败，本服务也启动失败；Wants 是弱依赖/愿望，依赖服务启动失败不影响本服务。

2. 实操题：写一个 myapp.service 单元文件，使服务在 network.target 之后启动，使用用户 myapp 执行，异常退出时自动重启，并限制内存最大 1GB。

思路：

```
[Unit]
After=network.target
[Service]
User=myapp
ExecStart=/opt/myapp/bin/server
Restart=on-failure
MemoryMax=1G
[Install]
WantedBy=multi-user.target
```

3. 日志题：查看 nginx 服务今天上午 10 点到 11 点之间所有 error 级别以上的日志，写出 journalctl 命令。

思路：journalctl -u nginx --since "2026-07-03 10:00" --until "2026-07-03 11:00" -p err。

4. 排障题：systemctl start nginx 卡住不动（一直处于 activating），过了很久才超时失败。你会如何排查？

思路：systemctl status nginx 看进程 PID，然后 strace -p <PID> 看卡在哪个系统调用（如 connect 卡数据库，或 fork 卡资源）。检查 Unit 文件的 Type 是否正确（如果是 forking 但父进程没退出就会卡）。

5. 对比题：systemd timer 相比传统的 cron 有哪些核心优势？

思路：支持单调定时器（开机后多久执行）、支持随机延迟（防惊群）、集成 journald 日志、支持依赖和资源限制（Cgroups）、支持丢失后补偿执行（Persistent=true）。

6. 实操题：创建一个每天凌晨 3 点执行 /usr/local/bin/backup.sh 的 timer，并配置在错过后开机立即补执行。

思路：写 backup.service（Type=oneshot）和 backup.timer。Timer 中设置 OnCalendar=*-*-* 03:00:00 和 Persistent=true，然后 systemctl enable backup.timer。

7. 安全题：如何在 systemd 服务中配置 NoNewPrivileges=true 和 PrivateTmp=true，分别起什么安全作用？

思路：NoNewPrivileges 阻止进程通过 setuid 或 capabilities 提权；PrivateTmp 给服务分配独立的 /tmp 命名空间，防止临时文件信息泄露或冲突。

8. 综合题：systemd-analyze blame 显示 network-online.target 耗时很长，但业务应用不需要等待网络完全就绪。如何优化启动速度？

思路：修改应用服务的 Unit 文件，将 After=network-online.target 改为 After=network.target（network.target 不等待网络就绪，启动更快），并去掉 Wants=network-online.target。
# 第八章：性能监控与调优（增强版）

> **本章定位**：从top到eBPF，从iostat到io_uring，本章覆盖2024-2026年Linux性能监控的完整工具链和调优方法论。

> **🧭 延伸阅读**：io_uring 是 Linux 5.1 引入的异步 I/O 框架，**在 6.x 版本中已全面成熟**，MySQL 8.0.26+、PostgreSQL 16+、Redis 7.0+ 均已默认启用。其性能比传统 libaio 提升 30%-100%，延迟降低 20%-50%。详见**第十三章 13.1 节**。


---

## 8.1 CPU监控：从top到eBPF

### 负载与CPU使用率：两个不同的概念

```bash
# 负载 = 正在运行 + 等待运行的进程数（包括IO等待）
$ uptime
10:00:00 up 30 days, load average: 4.50, 2.30, 1.20
# 1分钟/5分钟/15分钟平均负载

# 关键理解：
# load=4.5 on 4-core = 过载（队列堆积）
# load=4.5 on 16-core = 空闲（还有11.5核可用）
# load=2.0 but CPU 10% = IO瓶颈（大量进程等磁盘）

# 查看逻辑CPU数
$ nproc
16
$ grep -c ^processor /proc/cpuinfo
16
```

### 实时工具：top / htop / btop

```bash
# top 高级用法
$ top -bn1 -p $(pgrep -d',' nginx)    # 批量模式监控特定进程
$ top -H -p 1234                       # 查看进程的所有线程
$ top -d 0.5                           # 0.5秒刷新（更平滑）

# htop 优势
$ htop
# F2 设置：显示CPU频率、显示IO速率、自定义颜色
# F5 树状视图：看进程父子关系
# F6 排序：按MEM%、IO_RATE、TIME+等
# t 显示树状 / u 按用户过滤 / I 反转排序

# btop 最漂亮的监控
$ btop
# 优势：全屏ASCII图形、鼠标操作、内置历史图表
# 配置：~/.config/btop/btop.conf
```

### mpstat：多核CPU深度分析

```bash
$ mpstat -P ALL 2 5
# -P ALL = 所有CPU，2 = 2秒间隔，5 = 5次采样

Average:  CPU   %usr  %nice  %sys  %iowait  %irq  %soft  %steal  %guest  %idle
Average:  all    5.20   0.00  2.10    0.50  0.10   0.20    0.00    0.00  91.90
Average:    0   15.00   0.00  5.00    0.00  0.00   0.50    0.00    0.00  79.50
Average:    1    2.00   0.00  1.00    0.00  0.00   0.00    0.00    0.00  97.00
# ↑ CPU0 忙碌，CPU1 空闲 = 单线程瓶颈

# 关键指标：
# %iowait 高 = 磁盘瓶颈（CPU空闲等IO）
# %steal 高 = 虚拟机CPU被宿主机抢占（云环境关注）
# %soft 高 = 软中断过多（网卡收包太多？）
```

### perf：定位CPU热点函数

```bash
# 1. 实时查看热点函数
$ sudo perf top
# 显示消耗CPU最多的内核/用户态函数
# 例：看到 20% libc-2.35.so  __memcmp_sse4_1
# → 大量字符串比较，优化方向：减少比较、用hash

# 2. 进程级采样（生成报告）
$ sudo perf record -p 1234 -g sleep 30
# -g = 记录调用栈，30秒采样
$ sudo perf report
# 交互式查看，按Enter展开调用栈

# 3. 生成火焰图（需额外工具）
$ sudo perf record -F 99 -a -g -- sleep 60
$ sudo perf script | ./stackcollapse-perf.pl | ./flamegraph.pl > cpu.svg
# 火焰图：宽度=CPU占比，y轴=调用栈深度
# 看"平顶" = 热点函数

# 4. 系统调用统计
$ sudo perf stat -e syscalls:sys_enter_read,syscalls:sys_enter_write -p 1234 sleep 5
# 统计5秒内read/write系统调用次数
```

### eBPF：2024-2026年的性能监控标配

```bash
# 安装bpftrace（eBPF高级语言）
$ sudo apt install bpftrace

# 1. 查看进程打开的文件（实时）
$ sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s: %s\n", comm, str(args->filename)); }'

# 2. 查看TCP连接建立（抓慢连接）
$ sudo bpftrace -e 'kprobe:tcp_v4_connect { @start[tid] = nsecs; } kretprobe:tcp_v4_connect /@start[tid]/ { @latency_us = hist((nsecs - @start[tid]) / 1000); delete(@start[tid]); }'

# 3. 查看文件系统延迟
$ sudo bpftrace -e 'kprobe:vfs_read { @start[tid] = nsecs; } kretprobe:vfs_read /@start[tid]/ { @us = hist((nsecs - @start[tid]) / 1000); delete(@start[tid]); }'

# 4. 查看MySQL慢查询（uprobes）
$ sudo bpftrace -e 'uprobe:/usr/sbin/mysqld:dispatch_command { @start[tid] = nsecs; } uretprobe:/usr/sbin/mysqld:dispatch_command /@start[tid]/ { @latency_ms = hist((nsecs - @start[tid]) / 1000000); delete(@start[tid]); }'

# 5. 使用bcc工具集
$ sudo apt install bpfcc-tools
$ sudo biolatency-bpfcc             # 块设备IO延迟直方图
$ sudo biosnoop-bpfcc               # 每次IO的详细信息
$ sudo ext4slower-bpfcc 10          # 抓ext4上超过10ms的IO
$ sudo tcplife-bpfcc                # TCP连接生命周期
$ sudo tcpconnect-bpfcc             # 实时TCP连接建立
$ sudo runqlat-bpfcc                # CPU调度队列延迟
```

### 实战：CPU问题排查流程

```bash
# 场景：CPU使用率100%，需要定位原因

# 步骤1：确认是用户态还是内核态
$ top -bn1 | head -3
%Cpu(s): 80.0 us, 15.0 sy, 0.0 ni, 0.0 id, 5.0 wa
# us高 = 应用计算密集
# sy高 = 系统调用/内核操作过多
# wa高 = IO等待

# 步骤2：找具体进程
$ ps -eo pid,pcpu,cmd --sort=-pcpu | head -5

# 步骤3：看进程在做什么（用户态高）
$ sudo strace -c -p 1234 -e trace=all - sleep 5

# 步骤4：perf定位热点（用户态高但strace看不出）
$ sudo perf record -p 1234 -g sleep 10
$ sudo perf report

# 步骤5：如果是内核态高
$ sudo perf top
# 常见：_raw_spin_lock（锁竞争）、ext4_file_read_iter（文件系统）

# 步骤6：如果是IO等待高
$ iostat -x 1 5
```

---


---

## 8.1.5 eBPF：内核的可编程“操作系统”
**一句话定义**：eBPF（Extended Berkeley Packet Filter）允许用户在内核中安全地运行沙盒程序，无需修改内核源码或加载内核模块。它是 2024-2026 年云原生可观测性、安全、网络的底层引擎（Cilium、Falco、bpftrace 都依赖它）。
### eBPF 的核心架构（四层）
┌─────────────────────────────────────────────────────────┐
│  用户态：bpftrace / bcc / Cilium / 你的 Go/Python 程序   │
│         ↓ 加载 eBPF 字节码                              │
├─────────────────────────────────────────────────────────┤
│  内核态：eBPF 验证器（Verifier）→ JIT 编译 → 挂载执行    │
│         ↓ 挂载点：kprobe / tracepoint / uprobe / XDP   │
├─────────────────────────────────────────────────────────┤
│  事件源：系统调用、网络包、函数入口/返回、调度事件        │
└─────────────────────────────────────────────────────────┘
### 关键概念速查
| 概念 | 通俗解释 |
|------|----------|
| **kprobe / kretprobe** | 在内核函数入口/出口安插探测点 |
| **tracepoint** | 内核预置的静态探测点（更稳定） |
| **uprobe** | 在用户态函数（如 malloc）安插探测点 |
| **XDP** | 在网卡驱动层处理数据包（比 iptables 快 10 倍） |
| **Map** | eBPF 程序与用户态共享数据的“黑板” |
### 实战：从零到一的 eBPF 感知
# 1. 确认内核支持
$ grep CONFIG_BPF /boot/config-$(uname -r) | grep '=y' | wc -l
# 如果 > 20，说明支持（现代发行版默认开启）
# 2. 安装 bcc 工具集（无需编写代码，直接用）
$ sudo apt install bpfcc-tools
# 3. 看谁在打开文件（tracepoint 方式）
$ sudo opensnoop-bpfcc -T
TIME      PID    COMM               FD ERR PATH
10:00:01  1234   nginx               3   0 /var/log/nginx/access.log
# 4. 查看 TCP 连接建立（kprobe 方式）
$ sudo tcpconnect-bpfcc
PID    COMM         IP SADDR            DADDR            DPORT
5678   curl         4  192.168.1.100    142.250.80.46    443
# 5. 查看哪些进程在消耗 IO（不靠 iostat 猜）
$ sudo biotop-bpfcc
PID    COMM         DISK    READ/s  WRITE/s  WAIT(s)
1234   postgres     sda     12.5    8.2      0.23
自己写一个 eBPF 程序（bpftrace 一行流）
# 追踪所有 execve 系统调用（即每个新进程启动）
$ sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s executed %s\n", comm, str(args->filename)); }'
Attaching 1 probe...
bash executed /bin/ls
nginx executed /usr/sbin/nginx
# 追踪每次文件打开（抓谁在频繁读写）
$ sudo bpftrace -e 'kprobe:do_sys_open { printf("%s: %s\n", comm, str(arg1)); }'
避坑指南
# 坑1：debugfs 未挂载
$ sudo mount -t debugfs none /sys/kernel/debug
# 坑2：内核配置不足
$ cat /boot/config-$(uname -r) | grep -E "CONFIG_BPF|CONFIG_KPROBES"
# 如果缺失，需重新编译内核或换发行版
# 坑3：容器内无法运行 eBPF
# 需要 --privileged 或 --cap-add BPF,SYS_ADMIN
$ docker run --privileged --rm -v /sys/kernel/debug:/sys/kernel/debug:rw ...


## 8.2 内存监控：从free到memleak

### 内存指标详解

```bash
$ free -h
              total        used        free      shared  buff/cache   available
Mem:           62Gi       15Gi       2.0Gi       1.2Gi        45Gi        45Gi
Swap:         8.0Gi          0B       8.0Gi

# 关键字段：
# total     = 物理内存总量
# used      = 已使用（包含缓存）
# free      = 完全未使用
# buff/cache = 文件缓存（可回收）
# available = 真正可用（free + 可回收cache）
# ⚠ 看available，不是free！

# 缓存类型
$ cat /proc/meminfo | grep -E "^(Mem|Buffers|Cached|Swap|Dirty|Writeback|AnonPages|Slab)"
AnonPages:       12345678 kB     # 匿名页（应用数据，不可回收）
Dirty:              12345 kB     # 等待写回磁盘的
Writeback:           1234 kB     # 正在写回的
```

### vmstat：内存+IO+CPU一体化

```bash
$ vmstat 2 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0  2048000 123456 45120000    0    0     0     0  123  456  5  2 93  0  0

# 关键列：
# si/so = swap换入/换出（非0 = 内存不足！）
# bi/bo = 块设备读/写（KB/s）
# r = 运行队列长度（>CPU数 = 过载）
```

### smem：最准确的进程内存

```bash
$ sudo smem -tk
# USS = Unique Set Size（私有内存，进程独占）
# PSS = Proportional Set Size（USS + 共享内存/共享进程数）
# RSS = Resident Set Size（USS + 全部共享内存，会重复计算）
# ⚠ PSS是最准确的指标！
```

### 内存泄漏排查

```bash
# 方法1：持续监控RSS
$ while true; do ps -o pid,rss,cmd -p $(pgrep myapp) | tail -1; sleep 60; done

# 方法2：pmap看内存映射
$ pmap -x 1234 | sort -k3 -n | tail -20

# 方法3：bpftrace跟踪内存分配（生产环境）
$ sudo bpftrace -e 'uprobe:/usr/lib/x86_64-linux-gnu/libc.so.6:malloc { @allocations[comm] = count(); @bytes[comm] = sum(arg0); }'
```

### OOM Killer分析与防护

```bash
# 1. 查看OOM日志
$ dmesg | grep -i "out of memory"

# 2. OOM得分（值越高越容易被杀，范围 -1000 到 +1000）
$ cat /proc/1234/oom_score

# 3. 保护关键服务（永不OOM）
$ sudo cat > /etc/systemd/system/mysql.service.d/50-oom.conf <<'EOF'
[Service]
OOMScoreAdjust=-1000
EOF
```

---

## 8.3 磁盘IO监控：从iostat到io_uring

### iostat：磁盘级IO分析

```bash
$ iostat -x 2 3
# r_await/w_await = 单次IO等待时间(ms)
#   <1ms = NVMe优秀, 1-5ms = SSD正常, >20ms = 瓶颈
# %util = 设备利用率（HDD有意义，SSD看await更准）
```

### 进程级IO：iotop / pidstat

```bash
$ sudo iotop -o -b -n 1        # 只看有IO的进程
$ pidstat -d 2 5                # 更稳定，适合脚本
```

### 磁盘基准测试：fio

```bash
# 顺序读
$ sudo fio --name=seqread --ioengine=libaio --rw=read --bs=1M --direct=1 --size=4G --numjobs=1 --runtime=60 --group_reporting --filename=/data/test

# 随机读（测数据库场景）
$ sudo fio --name=randread --ioengine=libaio --rw=randread --bs=4k --direct=1 --size=4G --numjobs=16 --runtime=60 --group_reporting --filename=/data/test

# 混合读写（70%读+30%写）
$ sudo fio --name=mix --ioengine=libaio --rw=randrw --rwmixread=70 --bs=4k --direct=1 --size=4G --numjobs=16 --runtime=60 --group_reporting --filename=/data/test
```

### io_uring：Linux 5.1+的异步IO革命

```bash
# io_uring是新一代异步IO接口，性能比aio高2-5倍
$ grep CONFIG_IO_URING /boot/config-$(uname -r)
CONFIG_IO_URING=y

# 检查应用是否使用io_uring
$ sudo bpftrace -e 'tracepoint:io_uring:io_uring_submit_sqe { @apps[comm] = count(); }'

# 性能对比：通常IOPS提升30-100%，延迟降低20-50%
```

---

## 8.4 网络IO监控：从ss到eBPF

### ss + sar：基础网络监控

```bash
# 连接状态统计
$ ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
# TIME-WAIT > 10000 → 考虑启用tcp_tw_reuse
# SYN-RECV > 100 → 可能SYN Flood攻击

# 网络流量 + TCP统计
$ sar -n DEV 1 5                  # 每网卡带宽
$ sar -n TCP,ETCP 1 5             # 主动/被动连接、重传数
# retrans/s > 10 = 网络不稳定
```

### iftop / nethogs：实时流量分析

```bash
$ sudo iftop -i eth0 -P           # 按连接显示带宽
$ sudo nethogs eth0               # 按进程显示带宽
```

### eBPF网络监控：tcpconnect / tcplife

```bash
$ sudo tcpconnect-bpfcc           # 实时TCP连接建立（找异常）
$ sudo tcplife-bpfcc              # TCP连接生命周期（找慢连接）
$ sudo tcpretrans-bpfcc           # 重传统计（找网络质量问题）
$ sudo gethostlatency-bpfcc       # DNS解析延迟
```

---

## 8.5 内核调优：sysctl生产模板

```bash
# /etc/sysctl.d/99-production.conf

# ===== 文件描述符 =====
fs.file-max = 2097152

# ===== 网络核心 =====
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65536
net.ipv4.tcp_max_syn_backlog = 65536

# ===== TCP优化 =====
net.ipv4.tcp_tw_reuse = 1              # TIME_WAIT复用
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_congestion_control = bbr  # BBR拥塞控制
net.core.default_qdisc = fq

# ===== 内存 =====
vm.swappiness = 10                     # 数据库推荐
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5
```

---

## 8.6 实战：一键性能采集脚本

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

---

> **本章字数**：约 7,500 字
> **涉及工具**：top, htop, btop, mpstat, perf, bpftrace, bcc, strace, smem, iostat, iotop, fio, ss, sar, iftop, nethogs


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

---

## 课后练习

1. 概念题：top 中的 load average 为 8.00 4.00 2.00，服务器是 4 核 CPU。这代表系统过去 1 分钟、5 分钟、15 分钟的负载状况如何？

思路：1 分钟负载 8 远超 4 核（严重过载），5 分钟 4（刚好满载），15 分钟 2（空闲）。说明系统在最近 1 分钟突然涌入大量任务，可能是突发的请求高峰或定时任务。

2. 实操题：定位 CPU 用户态（%us）占用最高的函数热点，请写出 perf 的实时监控命令。

思路：sudo perf top -g（-g 显示调用链）。如果是追踪特定进程：sudo perf top -p <PID>。

3. 排障题：free -h 显示 available 只有 100MB，但 buff/cache 占了 20GB。系统是否内存不足？

思路：是。available 是应用真正可用的内存（含可回收的 cache），如果 available 过低且 swap 开始使用（si/so > 0），说明内存紧张。需要排查 ps aux --sort=-%mem 找内存泄漏进程。

4. 监控题：磁盘 I/O 延迟高，iostat -x 1 中主要看哪两列来判断磁盘是否瓶颈？正常 SSD 的 await 应小于多少？

思路：看 %util（设备繁忙度，接近 100% 说明饱和）和 await（平均 I/O 响应时间）。普通 SATA SSD 的 await 应在 1-5ms 以内，超过 20ms 说明严重瓶颈。

5. 调优题：写一个生产环境适用的 sysctl 配置片段，开启 TCP BBR 拥塞控制。

思路：

```
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```

6. 安全题：如何在容器或非 root 环境下使用 eBPF 追踪程序？需要哪些 capabilities？

思路：需要 CAP_BPF（Linux 5.8+）和 CAP_SYS_ADMIN，或者 CAP_PERFMON + CAP_BPF。在 Docker 中需添加 --cap-add BPF --cap-add SYS_ADMIN（或 --privileged）。

7. 实操题：使用 bpftrace 一行命令追踪系统中所有执行 execve 系统调用的进程，并打印其命令行。

思路：sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s executed %s\n", comm, str(args->filename)); }'。

8. 综合题：某应用 RSS 内存持续缓慢增长，怀疑内存泄漏。请给出利用 pmap 或 smem 定位泄漏方向的思路。

思路：pmap -x <PID> | sort -k3 -rn | head -20 查看哪些内存段（heap 或匿名映射）占用最大且持续增长。如果是 heap 增长，结合 valgrind 或 heaptrack 分析；如果是 mmap 映射文件未释放，检查代码中的文件流关闭逻辑。
# 第九章：日志管理与定时任务

> **本章定位**：日志是故障排查的"黑匣子"，定时任务是自动化的"心脏"。本章从rsyslog的传统架构讲到journald的现代设计，从crontab的暗坑讲到systemd timer的精准调度，覆盖生产环境的全链路日志与定时任务管理。

## 9.1 日志系统架构演进

**一句话定义**：Linux日志系统经历了 syslogd → rsyslog → systemd-journald 的三代演进，现代生产环境通常是 journald（结构化采集）+ rsyslog（网络转发/持久化） 的混合架构。

### 三代日志系统对比

| 维度 | syslogd (1980s) | rsyslog (2004+) | systemd-journald (2012+) |
|------|-----------------|-----------------|--------------------------|
| 数据格式 | 纯文本 | 纯文本 + 有限结构化 | 二进制结构化（JSON-like） |
| 查询能力 | grep | grep + 简单过滤 | journalctl 多维度过滤 |
| 性能 | 低（同步写） | 中（异步队列） | 高（内存映射 + 索引） |
| 网络转发 | UDP（不可靠） | UDP/TCP/RELP | 需配合 rsyslog |
| 存储效率 | 差（重复文本） | 差 | 优（字段去重 + 压缩） |
| FIPS合规 | 否 | 可选 | 是（签名验证） |
| 现代推荐 | ❌ 淘汰 | ⚠️ 网络转发场景 | ✅ 首选采集端 |

### 为什么生产环境需要"混合架构"？
```bash
# 典型生产架构：
┌─────────────┐     ┌─────────────┐     ┌─────────────────┐
│ 应用进程     │────→│ journald    │────→│ 本地持久存储     │
│ (stdout/stderr)│   │ (结构化采集) │     │ (/var/log/journal)│
└─────────────┘     └──────┬──────┘     └─────────────────┘
                           │
                    ┌──────┴──────┐
                    │  rsyslog      │←── 网络转发到中心日志服务器
                    │ (imjournal)   │    或 Kafka/Elasticsearch
                    └───────────────┘
```
关键理解：
journald 负责"采"：二进制结构化、索引快、查询强
rsyslog 负责"传"：网络协议成熟（TCP/RELP）、生态兼容好
两者通过 imjournal 模块衔接，不是互斥关系
## 9.2 journald：结构化日志的核心

journald 是现代 Linux 日志系统的核心引擎。以下从存储机制、字段系统、查询实战、持久化配置、远程转发五个维度深入拆解。

## 9.2.1 二进制日志的存储机制
**一句话定义**：journald 将日志存储为 二进制 Journal 文件（*.journal），采用 追加写 + 字段索引 + 自动轮转 的设计，查询速度比文本日志快 10-100倍。
```bash
# 1. 查看日志存储位置
$ ls -la /var/log/journal/
drwxr-sr-x 2 root systemd-journal 4096 Jul  3 10:00 .
-rw-r----- 1 root systemd-journal 8.0M Jul  3 10:00 system.journal
-rw-r----- 1 root systemd-journal 8.0M Jul  3 09:00 system@0005f123.journal~
# *.journal = 当前活跃文件
# *.journal~ = 已归档（轮转后）

# 2. 查看单个journal文件信息
$ journalctl --header --file=/var/log/journal/system.journal
File Path: /var/log/journal/system.journal
File ID: f1234567890abcdef...
Machine ID: abc123...
Boot ID: def456...
State: ONLINE
Compatible Flags: COMPRESSED-ZSTD SEALED
Incompatible Flags: (none)
Header size: 240
Arena size: 8388608 (8.0M)
Data Hash Table Size: 2048
Field Hash Table Size: 333
Rotate Suggested: no
# ↑ COMPRESSED-ZSTD = zstd压缩，SEALED = 完整性校验

# 3. 查看当前磁盘占用
$ journalctl --disk-usage
Archived and active journals take up 500.0M in the file system.
# 默认上限是 /var 分区大小的 10% 或 4GB（取小值）
```
## 9.2.2 字段（Field）系统：结构化查询的基石
journald 的每条日志不是纯文本，而是 键值对集合：
```bash
# 查看一条日志的完整字段
$ journalctl -u nginx -n 1 -o verbose
Mon 2026-07-03 10:00:00 CST [s=abc123;i=1234;b=def456;m=5678;t=8901;x=1234567890]
    PRIORITY=6                                    # 日志级别（0-7）
    _UID=33                                       # 进程UID
    _GID=33                                       # 进程GID
    _SYSTEMD_UNIT=nginx.service                   # 所属服务
    _PID=1200                                     # 进程PID
    _COMM=nginx                                   # 进程名
    _EXE=/usr/sbin/nginx                          # 可执行文件路径
    _CMDLINE=nginx: master process /usr/sbin/nginx  # 完整命令行
    _CAP_EFFECTIVE=1ffffffffff                    # Capabilities
    _BOOT_ID=def456...                            # 本次启动ID
    _MACHINE_ID=abc123...                         # 机器ID
    _HOSTNAME=web-server-01                       # 主机名
    _TRANSPORT=stdout                             # 日志来源
    _STREAM_ID=abc...                             # 标准流ID
    MESSAGE=2026/07/03 10:00:00 [notice] 1#1: signal 1 (SIGHUP) received, reconfiguring
# ↑ 这些字段都可以作为 journalctl 的过滤条件
```
## 9.2.3 journalctl 高级查询实战
```bash
# 1. 按服务过滤（最常用）
$ journalctl -u nginx
$ journalctl -u nginx --since "2026-07-01 00:00" --until "2026-07-03 12:00"

# 2. 按优先级过滤
$ journalctl -p err                          # ERROR (3) 及以上
# 优先级：0=emerg 1=alert 2=crit 3=err 4=warn 5=notice 6=info 7=debug

# 3. 按进程/用户/字段过滤
$ journalctl _PID=1200                       # 特定PID
$ journalctl _UID=33                         # 特定UID（如www-data）
$ journalctl _COMM=nginx                     # 进程名
$ journalctl _EXE=/usr/sbin/nginx            # 可执行路径
$ journalctl _SYSTEMD_UNIT=nginx.service _PID=1201  # 组合条件

# 4. 按启动周期过滤（排查崩溃神器）
$ journalctl -b                              # 本次启动
$ journalctl -b -1                           # 上次启动（查崩溃原因）
$ journalctl -b -2                           # 上上次
$ journalctl --list-boots                    # 列出所有启动记录
Idx Boot ID                          First Entry              Last Entry
 -2 abc123...                        Mon 2026-06-30 09:00    Mon 2026-06-30 18:00
 -1 def456...                        Tue 2026-07-01 09:00    Tue 2026-07-01 15:30
  0 ghi789...                        Wed 2026-07-03 09:00    Wed 2026-07-03 10:00
# 系统崩溃后，-1 的日志是排查关键

# 5. 实时跟踪（tail -f 等价）
$ journalctl -f                              # 所有日志
$ journalctl -u nginx -f                     # 仅nginx
$ journalctl -u nginx -f -n 100              # 先显示100行，再跟踪

# 6. 按时间窗口（自然语言）
$ journalctl --since "1 hour ago"
$ journalctl --since "30 minutes ago" --until "5 minutes ago"
$ journalctl --since yesterday --until today

# 7. 输出格式控制
$ journalctl -o short                        # 默认短格式
$ journalctl -o verbose                      # 完整字段（上面示例）
$ journalctl -o json                         # JSON（程序处理）
$ journalctl -o json-pretty                  # 格式化JSON
$ journalctl -o cat                          # 仅MESSAGE字段

# 8. 用grep做不到的"字段级过滤"
$ journalctl MESSAGE_ID=1234567890abcdef...  # 按消息ID（systemd预定义的错误类型）
$ journalctl --grep="failed|error|timeout" --case-insensitive  # journalctl内置grep（比管道快）

# 9. 查看内核日志（替代dmesg）
$ journalctl -k                              # 本次启动内核日志
$ journalctl -k -b -1                        # 上次启动内核日志（排查驱动崩溃）

# 10. 跨主机聚合（配合systemd-remote）
$ journalctl -D /var/log/journal/remote/     # 查看远程主机日志
```
## 9.2.4 持久化配置与存储管理
```bash
# 默认配置：/etc/systemd/journald.conf
$ cat /etc/systemd/journald.conf
[Journal]
Storage=auto                    # auto|persistent|volatile|none
# auto = 有/var/log/journal就持久化，否则存/run（内存）
# persistent = 强制持久化
# volatile = 仅内存（/run/log/journal）
# none = 完全禁用

Compress=yes                    # zstd压缩
Seal=yes                        # FIPS 140-2完整性校验（签名）
SplitMode=uid                   # 按用户拆分日志文件
SystemMaxUse=4G                 # 日志总大小上限
SystemMaxFileSize=100M          # 单个文件上限
SystemMaxFiles=5                # 保留文件数
MaxRetentionSec=1week           # 保留时间
SyncIntervalSec=5min            # 同步到磁盘间隔

# 1. 启用持久化（生产环境必做）
$ sudo mkdir -p /var/log/journal
$ sudo systemd-tmpfiles --create --prefix /var/log/journal
# 创建正确的权限和SELinux标签
$ sudo systemctl restart systemd-journald

# 2. 限制日志大小（防止打满磁盘）
$ sudo mkdir -p /etc/systemd/journald.conf.d/
$ cat <<'EOF' | sudo tee /etc/systemd/journald.conf.d/99-size-limit.conf
[Journal]
SystemMaxUse=2G
SystemMaxFileSize=100M
MaxRetentionSec=2week
EOF
$ sudo systemctl restart systemd-journald

# 3. 手动清理
$ sudo journalctl --vacuum-size=500M       # 压缩到500M
$ sudo journalctl --vacuum-time=7d         # 只保留7天
$ sudo journalctl --vacuum-files=5         # 只保留5个文件

# 4. 查看清理效果
$ journalctl --disk-usage
Vacuuming done, freed 1.5G of archived journals.
Archived and active journals take up 500.0M in the file system.
```
## 9.2.5 远程日志转发（rsyslog + journald 混合架构）
```bash
# 场景：多台服务器日志集中到一台日志服务器

# === 日志服务器（接收端）===
$ sudo apt install rsyslog
$ cat <<'EOF' | sudo tee /etc/rsyslog.d/10-server.conf
# 接收TCP日志（可靠）
module(load="imtcp")
input(type="imtcp" port="514")

# 按主机名分目录存储
$template RemoteLogs,"/var/log/remote/%fromhost-ip%/%programname%.log"
*.* ?RemoteLogs
& stop
EOF
$ sudo systemctl restart rsyslog

# === 客户端（发送端）===
# 方式1：rsyslog直接转发（传统，丢失结构化字段）
$ cat <<'EOF' | sudo tee /etc/rsyslog.d/99-forward.conf
*.* @@192.168.1.10:514          # @@ = TCP, @ = UDP
EOF

# 方式2：journald → rsyslog → 远程（保留结构化）
$ cat <<'EOF' | sudo tee /etc/systemd/journald.conf.d/99-forward.conf
[Journal]
ForwardToSyslog=yes             # 转发到rsyslog
EOF
$ sudo systemctl restart systemd-journald

# rsyslog配置读取journal（imjournal模块）
$ cat <<'EOF' | sudo tee /etc/rsyslog.d/50-default.conf
module(load="imjournal" StateFile="imjournal.state")
*.* @@192.168.1.10:514
EOF
$ sudo systemctl restart rsyslog

# 验证

# 方式3：原生二进制转发（systemd-journal-remote，保留所有结构化字段）
# 服务端：
$ sudo systemctl enable --now systemd-journal-remote.socket
# 客户端：
$ sudo systemctl enable --now systemd-journal-upload --url=http://log-server:19532
# 优势：全程保留 _UID、_SYSTEMD_UNIT 等元数据，无需解析为文本再重传
$ logger -p err "Test error message"
$ ssh 192.168.1.10 "tail /var/log/remote/$(hostname -I | awk '{print $1}')/logger.log"
```
## 9.3 rsyslog：传统但不可替代

rsyslog 在 journald 时代并未淘汰——它承担着网络转发、自定义过滤和兼容传统生态的关键角色。

## 9.3.1 rsyslog 核心概念
```bash
# rsyslog 处理流程：
输入模块 (imuxsock, imjournal, imtcp) 
  → 规则引擎 (Rule Engine) 
  → 过滤条件 (Property-based Filters, RainerScript)
  → 动作/模板 (Action, Template)
  → 输出模块 (omfile, ommysql, omelasticsearch, omkafka)

# 配置文件：/etc/rsyslog.conf + /etc/rsyslog.d/*.conf
```
## 9.3.2 高级过滤与模板
```bash
# 1. 基于属性的过滤（RainerScript，rsyslog v5+）
$ cat <<'EOF' | sudo tee /etc/rsyslog.d/20-custom.conf
# 仅记录ssh登录失败
:programname, isequal, "sshd" {
    :msg, contains, "Failed password" {
        /var/log/ssh-failures.log
        stop                    # 匹配后停止处理，不写入其他文件
    }
}

# 按严重程度分文件
if $syslogseverity <= 3 then {      # 0-3 = emerg/alert/crit/err
    action(type="omfile" file="/var/log/critical.log")
}

# 复杂条件
if $programname == 'nginx' and $msg contains 'error' then {
    action(type="omfile" file="/var/log/nginx-errors.log")
    & action(type="omfwd" target="192.168.1.10" port="514" protocol="tcp")
    # 同时写本地+转发远程
}
EOF

# 2. 模板定制输出格式
$ cat <<'EOF' | sudo tee /etc/rsyslog.d/30-templates.conf
template(name="JsonFormat" type="list") {
    constant(value="{")
    constant(value="\"timestamp\":\"")     property(name="timereported" dateFormat="rfc3339")
    constant(value="\",\"host\":\"")       property(name="hostname")
    constant(value="\",\"severity\":\"")   property(name="syslogseverity-text")
    constant(value="\",\"program\":\"")    property(name="programname")
    constant(value="\",\"message\":\"")    property(name="msg" format="json")
    constant(value="\"}\n")
}
# 使用模板
*.* action(type="omfile" file="/var/log/json.log" template="JsonFormat")
EOF

# 3. 丢弃无用日志（防磁盘打满）
:programname, isequal, "CRON" stop       # 完全丢弃cron日志
:msg, contains, "session opened for user root" stop  # 丢弃root登录记录
```
## 9.3.3 高性能队列与可靠性
```bash
# 磁盘辅助队列（防突发流量压垮）
$ cat <<'EOF' | sudo tee /etc/rsyslog.d/40-reliability.conf
main_queue(
    queue.type="LinkedList"             # 内存队列类型
    queue.filename="main_q"             # 磁盘队列文件名
    queue.maxdiskspace="1g"             # 磁盘队列上限
    queue.saveonshutdown="on"           # 关机时保存队列
    queue.checkpointinterval="100"      # 每100条checkpoint
)

# TCP转发带重连
action(
    type="omfwd"
    target="logs.example.com"
    port="514"
    protocol="tcp"
    queue.type="LinkedList"
    queue.filename="fwd_q"
    queue.maxdiskspace="500m"
    action.resumeRetryCount="-1"        # 无限重试
    action.resumeInterval="30"          # 每30秒重试
)
EOF
```
## 9.4 日志轮转（logrotate）深度实战

日志不轮转 = 磁盘迟早被打满。logrotate 是生产环境日志管理的标配工具。

## 9.4.1 logrotate 工作原理
```bash
# logrotate 不是守护进程！
# 由 cron 每天运行一次：/etc/cron.daily/logrotate
# 或 systemd timer：/lib/systemd/system/logrotate.timer

# 核心机制：
# 1. 重命名当前日志文件（如 access.log → access.log.1）
# 2. 发送信号给进程（如 USR1 让 nginx 重新打开日志）
# 3. 压缩旧日志（.1 → .1.gz）
# 4. 删除超期日志
```
## 9.4.2 生产级配置模板
```bash
# /etc/logrotate.d/nginx（生产优化版）
/var/log/nginx/*.log {
    daily                              # 每天轮转
    missingok                          # 文件不存在不报错
    rotate 30                          # 保留30天（配合压缩约3-5GB）
    compress                           # 压缩旧日志
    delaycompress                      # 延迟1天压缩（给进程时间关闭fd）
    notifempty                         # 空文件不轮转
    create 640 nginx adm               # 新文件权限
    dateext                            # 用日期后缀（access.log-20260703）
    dateformat -%Y%m%d-%s              # 自定义日期格式
    maxsize 100M                       # 超过100M也轮转（即使不到一天）
    minsize 1M                         # 小于1M不轮转
    sharedscripts                      # 所有文件处理完再执行一次脚本
    
    postrotate
        # 优雅重载nginx（不丢连接）
        if [ -f /run/nginx.pid ]; then
            kill -USR1 $(cat /run/nginx.pid) 2>/dev/null || true
        fi
        # 验证日志是否重新打开
        sleep 1
        if lsof -p $(cat /run/nginx.pid) 2>/dev/null | grep -q "deleted"; then
            logger -t logrotate "WARNING: nginx still holding deleted log fd"
        fi
    endscript
    
    prerotate
        # 轮转前检查磁盘空间
        if [ $(df -P /var/log | awk 'NR==2 {print $5}' | tr -d '%') -gt 90 ]; then
            logger -t logrotate "ERROR: Disk space critical, skipping rotation"
            exit 1
        fi
    endscript
    
    lastaction
        # 轮转完成后，异步上传到S3（不阻塞）
        /usr/local/bin/upload-logs-to-s3.sh /var/log/nginx/ &
    endaction
}
```
## 9.4.3 避坑指南：logrotate 常见故障
```bash
# 坑1：进程没收到信号，继续写已删除的文件
$ sudo lsof | grep deleted | grep nginx
nginx  1200  root   5w  REG  8,2  500M  131142 /var/log/nginx/access.log (deleted)
# 解决：确保 postrotate 信号正确，且进程PID文件存在

# 坑2：磁盘满导致轮转失败
$ df -h /var/log
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        50G   50G   0G  100% /
# logrotate 需要磁盘空间创建新文件！
# 解决：设置 maxsize + 提前告警 + prerotate检查

# 坑3：权限错误导致新日志无法创建
$ ls -la /var/log/nginx/
-rw-r--r-- 1 root root 1G access.log    # 被root运行后属主变成root
# nginx（www-data用户）无法写入新文件
# 解决：严格设置 create 640 nginx adm

# 坑4：copytruncate 的丢失风险
# 某些应用不支持信号重载，用 copytruncate（复制+截断）
# 但 copytruncate 有极小概率丢失日志行
# 解决：尽量用信号，copytruncate 作为最后手段

# 调试 logrotate
$ sudo logrotate -d /etc/logrotate.d/nginx   # 调试模式（不执行）
$ sudo logrotate -f /etc/logrotate.d/nginx   # 强制立即执行
$ sudo logrotate -v /etc/logrotate.conf      # 详细输出
```
## 9.5 结构化日志与集中管理

当服务器数量 > 1 时，分散的日志文件就是灾难。在生产环境中，日志必须结构化输出并集中存储。

## 9.5.1 应用层结构化输出
```bash
# Python 结构化日志示例
import logging
import logging.handlers
import json
import sys

class JournalFormatter(logging.Formatter):
    def format(self, record):
        log_entry = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "source": {
                "file": record.filename,
                "line": record.lineno,
                "function": record.funcName
            }
        }
        if hasattr(record, "request_id"):
            log_entry["request_id"] = record.request_id
        if record.exc_info:
            log_entry["exception"] = self.formatException(record.exc_info)
        return json.dumps(log_entry, ensure_ascii=False)

# 输出到 journald（自动带结构化字段）
logger = logging.getLogger("myapp")
handler = logging.handlers.SysLogHandler(address='/dev/log')
handler.setFormatter(JournalFormatter())
logger.addHandler(handler)

# 查询：
# journalctl -t myapp -o json | jq 'select(.LEVEL=="ERROR")'
```
## 9.5.2 轻量级集中方案：Promtail + Loki
```bash
# Promtail 配置：/etc/promtail/promtail.yml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki-server:3100/loki/api/v1/push

scrape_configs:
  - job_name: journal
    journal:
      max_age: 12h
      labels:
        job: systemd-journal
    relabel_configs:
      - source_labels: ['__journal__systemd_unit']
        target_label: 'unit'
      - source_labels: ['__journal__hostname']
        target_label: 'host'

  - job_name: nginx
    static_configs:
      - targets:
          - localhost
        labels:
          job: nginx
          __path__: /var/log/nginx/access.log

# 查询（Grafana）：
# {job="nginx"} |= "500" | json | status_code="500"
```
## 9.6 cron 定时任务：经典但暗坑密布

cron 已经存在了 40 年，但它那些"反人类"的设计至今每年仍在生产环境制造事故。理解它的暗坑比背格式更重要。

## 9.6.1 crontab 格式精解
```bash
# 标准格式：
# .---------------- 分钟 (0-59)
# |  .------------- 小时 (0-23)
# |  |  .---------- 日期 (1-31)
# |  |  |  .------- 月份 (1-12)
# |  |  |  |  .---- 星期 (0-7, 0和7都是周日)
# |  |  |  |  |
# *  *  *  *  *  command

# 特殊字符：
# ,   列表：0,15,30,45
# -   范围：9-17
# /   步进：*/5（每5分钟）
# L   最后（部分实现）
# W   最近工作日（部分实现）

# 常用频率示例：
* * * * *       # 每分钟
*/5 * * * *     # 每5分钟
0 * * * *       # 每小时整点
0 2 * * *       # 每天凌晨2点
0 9 * * 1-5     # 工作日9:00
0 0 1 * *       # 每月1日
0 0 * * 0       # 每周日
@reboot         # 系统启动时（非标准但广泛支持）
@daily          # 每天 = 0 0 * * *
@hourly         # 每小时 = 0 * * * *
```
## 9.6.2 crontab 命令与环境
```bash
# 1. 用户级 crontab
$ crontab -l                          # 列出当前用户
$ crontab -e                          # 编辑（自动语法检查）
$ crontab -r                          # 删除所有（危险！）
$ sudo crontab -l -u alice            # 查看其他用户（需root）

# 2. 系统级定时任务
$ ls /etc/cron.d/                     # 系统任务目录（应用包放置）
$ ls /etc/cron.daily/                 # 每天执行（anacron保证）
$ ls /etc/cron.hourly/                # 每小时
$ ls /etc/cron.weekly/                # 每周
$ ls /etc/cron.monthly/               # 每月

# 3. cron 的极简环境（最大暗坑！）
$ cat /tmp/test-cron.sh
#!/bin/bash
echo "PATH=$PATH" > /tmp/cron-path.log
echo "SHELL=$SHELL" >> /tmp/cron-path.log
echo "HOME=$HOME" >> /tmp/cron-path.log
echo "USER=$USER" >> /tmp/cron-path.log
$ chmod +x /tmp/test-cron.sh
$ echo "* * * * * /tmp/test-cron.sh" | crontab -
$ cat /tmp/cron-path.log
PATH=/usr/bin:/bin                    # ← 极短！没有/usr/local/bin
SHELL=/bin/sh                         # ← 不是bash！
HOME=/home/alice                      # ← 用户家目录
USER=alice
# 结论：cron环境极度干净，必须显式设置PATH和SHELL
```
## 9.6.3 cron 四大天坑与解决方案
```bash
# === 坑1：环境变量缺失 ===
# 错误示范：
0 2 * * * backup.sh                   # backup.sh 找不到 /usr/local/bin 下的命令

# 正确做法：
0 2 * * * . /home/alice/.profile; /usr/local/bin/backup.sh
# 或
0 2 * * * /bin/bash -lc '/usr/local/bin/backup.sh'
# -l = login shell，加载完整环境

# 更安全的做法：在脚本开头自给自足
#!/bin/bash
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
export HOME="/home/alice"
export LANG="en_US.UTF-8"
cd "$HOME" || exit 1

# === 坑2：输出未重定向，邮件塞满/var/mail ===
# 错误示范：
0 2 * * * /usr/local/bin/backup.sh    # stdout/stderr 发给crontab所有者

# 正确做法：
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
# 或完全丢弃（不推荐，丢失错误信息）：
0 2 * * * /usr/local/bin/backup.sh > /dev/null 2>&1

# 更好的做法：按日期分文件
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup-$(date +\%Y\%m\%d).log 2>&1
# 注意：% 在crontab中要转义为 \%

# === 坑3：任务重叠（上一个没跑完，下一个又启动）===
# 错误示范：
* * * * * /usr/local/bin/long-task.sh  # 任务运行2分钟，每分钟启动一个

# 正确做法：文件锁
* * * * * flock -n /tmp/long-task.lock -c /usr/local/bin/long-task.sh
# -n = 非阻塞，如果锁被占用则跳过本次
# 或
* * * * * /usr/local/bin/long-task.sh 2>/dev/null || true
# 在脚本内部用 flock：
#!/bin/bash
exec 200>/var/run/myapp.lock
flock -n 200 || { echo "Another instance running"; exit 1; }

# === 坑4：时区与夏令时 ===
# cron 使用系统时区（/etc/localtime）
# 但可能和应用程序时区不一致！

# 检查：
$ date
Wed Jul  3 10:00:00 CST 2026
$ cat /etc/timezone
Asia/Shanghai

# 在crontab中显式设置：
CRON_TZ=Asia/Shanghai
0 2 * * * /usr/local/bin/backup.sh    # 确保按上海时间2点执行

# 定时任务错乱时，第一时间确认系统时区：
$ timedatectl status               # 看 Time zone 是否一致
$ timedatectl list-timezones        # 查找可用时区
$ sudo timedatectl set-timezone Asia/Shanghai
```
## 9.6.4 cron 调试与监控
```bash
# 1. 查看 cron 日志
$ grep CRON /var/log/syslog           # Debian/Ubuntu
$ grep CRON /var/log/cron             # RHEL/CentOS
Jul  3 10:00:01 server CRON[1234]: (alice) CMD (/usr/local/bin/backup.sh)
Jul  3 10:00:01 server CRON[1234]: (alice) MAIL (mailed 45 bytes of output)

# 2. 测试 crontab 条目
$ echo '* * * * * echo "test $(date)" >> /tmp/cron-test.log 2>&1' | crontab -
# 等1-2分钟，检查 /tmp/cron-test.log

# 3. 检查 cron 守护进程状态
$ sudo systemctl status cron            # Debian
$ sudo systemctl status crond           # RHEL

# 4. 查看用户 cron 权限
$ cat /etc/cron.allow                   # 白名单（存在则仅允许列表内用户）
$ cat /etc/cron.deny                    # 黑名单（allow不存在时生效）
# 安全建议：使用 allow 白名单机制
```
## 9.7 systemd timer：现代化的定时任务

timer 是 cron 的现代化替代品——它解决了 cron 的四个核心缺陷。

## 9.7.1 为什么 timer 优于 cron？

| 维度 | cron | systemd timer |
|------|------|----------------|
| 日志记录 | 需手动重定向 | 自动集成 journald |
| 环境变量 | 极简环境 | 继承 systemd 环境 |
| 依赖管理 | 无 | 可声明 After/Wants |
| 失败处理 | 无 | 自动记录失败状态 |
| 并发控制 | 需手动 flock | 自动防止重叠 |
| 随机延迟 | 无 | RandomizedDelaySec |
| 单调时钟 | 无 | OnBootSec, OnUnitActiveSec |
| 持久化 | 无 | Persistent=true（补偿错过的时间） |
| 时区支持 | 弱 | 完整支持 |

## 9.7.2 timer 文件详解
```bash
# 基本结构：*.timer + *.service（同名）

# /etc/systemd/system/backup.timer
[Unit]
Description=Daily backup timer
Documentation=man:backup(8)

[Timer]
# 实时时钟（wall clock）：类似cron
OnCalendar=*-*-* 02:00:00             # 每天凌晨2点
# 或更复杂的：
# OnCalendar=Mon *-*-01..07 02:00:00  # 每月第一个周一
# OnCalendar=*-*-01 03:00:00          # 每月1日3点

# 单调时钟（monotonic）：基于事件
OnBootSec=15min                       # 启动后15分钟
OnUnitActiveSec=24h                   # 上次执行后24小时
OnUnitInactiveSec=30min               # 上次停止后30分钟

# 随机延迟（防惊群，多台服务器同时执行）
RandomizedDelaySec=30min              # 0-30分钟随机延迟

# 持久化：如果系统关机时错过执行，开机后补偿
Persistent=true

# 精度
AccuracySec=1min                      # 默认1分钟

[Install]
WantedBy=timers.target

# /etc/systemd/system/backup.service
[Unit]
Description=Daily backup
After=network.target postgresql.service

[Service]
Type=oneshot                          # 一次性任务
ExecStart=/usr/local/bin/backup.sh
User=backup
Group=backup
WorkingDirectory=/backup
StandardOutput=journal
StandardError=journal
# 自动防止重叠（如果上次没跑完，不会启动新实例）
```

## 9.7.3 timer 高级实战
```bash
# 1. 查看所有 timer
$ systemctl list-timers --all
NEXT                        LEFT     LAST                        PASSED  UNIT                  ACTIVATES
Wed 2026-07-03 02:00:00 CST 3h left  Tue 2026-07-02 02:15:00 CST 20h ago backup.timer          backup.service
Wed 2026-07-03 03:00:00 CST 4h left  n/a                         n/a     certbot.timer         certbot.service

# 2. 查看 timer 详情
$ systemctl show backup.timer
NextElapseUSecRealtime=Wed 2026-07-03 02:00:00 CST
LastTriggerUSec=Tue 2026-07-02 02:15:00 CST
Result=success
AccuracyUSec=1min

# 3. 手动触发
$ sudo systemctl start backup.service    # 立即执行服务
$ sudo systemctl start backup.timer      # 启动timer（到点就触发）

# 4. 查看上次执行结果
$ systemctl status backup.service
● backup.service - Daily backup
   Loaded: loaded (/etc/systemd/system/backup.service; static)
   Active: inactive (dead) since Tue 2026-07-02 02:20:00 CST; 20h ago
  Process: 1234 ExecStart=/usr/local/bin/backup.sh (code=exited, status=0/SUCCESS)
 Main PID: 1234 (code=exited, status=0/SUCCESS)
# 注意：即使服务已退出，status 仍保留上次执行信息

# 5. 复杂的定时策略：工作日每4小时执行
$ cat /etc/systemd/system/check.timer
[Timer]
OnCalendar=Mon..Fri *-*-* 00,04,08,12,16,20:00:00
Persistent=true
RandomizedDelaySec=10min

# 6. 开机后定期执行 + 失败后重试
$ cat /etc/systemd/system/cleanup.service
[Unit]
Description=Temp file cleanup
StartLimitIntervalSec=300
StartLimitBurst=3

[Service]
Type=oneshot
ExecStart=/usr/local/bin/cleanup.sh
Restart=on-failure
RestartSec=60
# 如果cleanup.sh失败，1分钟后重试，最多5分钟内重试3次
```

## 9.7.4 timer 与 cron 的迁移对照
```bash
# cron: 0 2 * * * /backup.sh
# timer 等价：
[Timer]
OnCalendar=*-*-* 02:00:00

# cron: */5 * * * * /health-check.sh
# timer 等价：
[Timer]
OnUnitActiveSec=5min
AccuracySec=1s

# cron: @reboot /startup.sh
# timer 等价：
[Timer]
OnBootSec=30s
# 更优：用 service 的 After=network-online.target

# cron: 0 9 * * 1-5 /workday-report.sh
# timer 等价：
[Timer]
OnCalendar=Mon..Fri *-*-* 09:00:00
```
## 9.8 实战：生产级日志与定时任务架构
场景：Web 应用全链路监控
```bash
# === 架构设计 ===
# 应用 → journald → Promtail → Loki → Grafana
#       ↘ rsyslog → 远程归档（合规要求）

# 1. 应用日志配置（Python Flask 示例）
# config.py
import logging
import logging.handlers

class RequestIdFilter(logging.Filter):
    def filter(self, record):
        record.request_id = g.get('request_id', 'unknown')
        return True

handler = logging.handlers.SysLogHandler(address='/dev/log')
handler.setFormatter(logging.Formatter(
    'flask-app: %(asctime)s [%(levelname)s] [%(request_id)s] %(message)s'
))
handler.addFilter(RequestIdFilter())
app.logger.addHandler(handler)

# 2. journald 配置
$ cat /etc/systemd/journald.conf.d/99-app.conf
[Journal]
Storage=persistent
MaxRetentionSec=30day
ForwardToSyslog=yes

# 3. Promtail 配置（抓取 journal + 应用日志）
$ cat /etc/promtail/config.yml
scrape_configs:
  - job_name: journal
    journal:
      max_age: 12h
      labels:
        job: systemd-journal
    relabel_configs:
      - source_labels: ['__journal__systemd_unit']
        target_label: 'unit'
      - source_labels: ['__journal_SYSLOG_IDENTIFIER']
        target_label: 'app'

  - job_name: flask-app
    static_configs:
      - targets: [localhost]
        labels:
          job: flask-app
          __path__: /var/log/flask-app/*.log

# 4. 定时任务：每日日志归档 + 清理
$ cat /etc/systemd/system/log-archive.timer
[Timer]
OnCalendar=*-*-* 03:00:00
RandomizedDelaySec=30min
Persistent=true

$ cat /etc/systemd/system/log-archive.service
[Unit]
Description=Archive old logs to S3

[Service]
Type=oneshot
ExecStart=/usr/local/bin/archive-logs.sh
User=logadmin
```

**archive-logs.sh 脚本实现**：

```bash
#!/bin/bash
set -euo pipefail

YESTERDAY=$(date -d "yesterday" +%Y%m%d)
ARCHIVE_DIR="/archive/logs/${YESTERDAY}"

mkdir -p "$ARCHIVE_DIR"

# 归档 journal
journalctl --since "${YESTERDAY} 00:00:00" --until "${YESTERDAY} 23:59:59" -o json > "$ARCHIVE_DIR/journal.json"

# 归档应用日志
cp /var/log/flask-app/app.log-$YESTERDAY* "$ARCHIVE_DIR/" 2>/dev/null || true

# 压缩上传
tar czf - "$ARCHIVE_DIR" | aws s3 cp - s3://mybucket/logs/${YESTERDAY}.tar.gz

# 本地清理（保留7天）
find /archive/logs -maxdepth 1 -mtime +7 -exec rm -rf {} \;

# 清理 journal
journalctl --vacuum-time=7d
本章小结
日志管理决策树
plain
需要结构化查询？
├─ 是 → journald（首选）
│       └─ 需要网络转发？→ rsyslog 混合
└─ 否 → rsyslog（简单场景）

需要集中分析？
├─ 轻量级 → Promtail + Loki
├─ 全功能 → Filebeat + Elasticsearch
└─ 云原生 → Fluent Bit + 云厂商日志服务

需要长期归档？
└─ rsyslog → 对象存储（S3/OSS）
定时任务决策树
plain
需要依赖管理/失败处理/自动日志？
├─ 是 → systemd timer（现代首选）
└─ 否 → cron（简单场景）

需要跨平台（非systemd）？
└─ cron 或 anacron
关键配置检查清单
```bash
# journald 持久化
$ ls -la /var/log/journal/          # 目录存在且权限正确

# logrotate 状态
$ sudo logrotate -d /etc/logrotate.conf  # 无语法错误

# cron 环境
$ crontab -l | grep -E 'PATH|SHELL'     # 显式设置环境

# timer 状态
$ systemctl list-timers --failed        # 无失败timer
> 本章扩充后字数：约 12,000 字
> 涉及命令：journalctl, rsyslogd, logrotate, crontab, systemctl, logger, lsof, flock

---

## 课后练习

1. 概念题：journald 的日志默认存储在 /run/log/journal/（内存）还是 /var/log/journal/（磁盘）？如何永久切换到磁盘持久化？

思路：默认是 auto（存内存）。创建 /var/log/journal/ 目录并设置正确权限（systemd-tmpfiles --create --prefix /var/log/journal），然后重启 systemd-journald 即可。

2. 排障题：logrotate 执行后，Nginx 日志文件 access.log 变成了 0 字节，但磁盘空间依然没释放。为什么？

思路：Nginx 主进程仍然持有旧文件的文件描述符（fd）没释放。logrotate 的 postrotate 脚本需要向 Nginx 发送 USR1 信号（或 nginx -s reopen）使其重新打开日志文件。检查 postrotate 是否执行成功。

3. 实操题：编写一个 logrotate 配置，使 /var/log/myapp/*.log 每天轮转，保留 7 天，压缩，并在轮转后重启 myapp.service。

思路：

```
/var/log/myapp/*.log {
    daily
    rotate 7
    compress
    delaycompress
    sharedscripts
    postrotate
        systemctl reload myapp.service > /dev/null 2>&1 || true
    endscript
}
```

4. 对比题：cron 和 systemd timer 在执行环境（环境变量）上有什么巨大差异？

思路：cron 的环境极度干净（仅有 /usr/bin:/bin，PATH 很短，SHELL 是 /bin/sh），所以脚本中需显式 source ~/.bashrc 或写绝对路径；systemd timer 继承 systemd 环境（可配置 Environment=），更可控。

5. 排障题：Cron 定时任务明明在 crontab -l 里，但就是没执行。列出 3 种可能的原因。

思路：1. Cron 服务没启动（systemctl status cron）；2. 脚本没有执行权限；3. 脚本依赖的 PATH 在 cron 环境下找不到（需写绝对路径）；4. 输出未重定向导致邮件队列堵塞但没提醒。

6. 实操题：使用 systemd timer 实现一个每隔 5 分钟执行一次的健康检查脚本，要求即使系统在预定时间关机，下次开机也要补执行一次。

思路：Timer 配置 OnUnitActiveSec=5min（上次执行后 5 分钟）和 Persistent=true。

7. 安全题：如何通过配置 journald 限制日志总大小不超过 2GB，防止日志撑爆磁盘？

思路：在 /etc/systemd/journald.conf 中设置 SystemMaxUse=2G，然后 systemctl restart systemd-journald。也可以 journalctl --vacuum-size=2G 手动清理。

8. 综合题：如何用 logger 命令向 journald 发送一条 user 设施、warning 级别的消息，标签为 my-deploy？

思路：logger -t my-deploy -p user.warning "Deployment failed: disk full"。查询时用 journalctl -t my-deploy -p warning。
# 第十章：软件包管理

> **本章定位**：软件包管理是系统稳定性的基石。从 apt/dnf 的日常操作到仓库签名验证，从版本锁定到热修补，本章覆盖生产环境软件包管理的全生命周期。
## 10.1 apt / dpkg 深度实战（Debian/Ubuntu）

apt/dpkg 是 Debian 系发行版的包管理核心。以下从缓存机制、仓库管理、版本锁定、dpkg 底层操作四个维度深入。

## 10.1.1 apt 工作原理与缓存机制
```bash
# apt 的完整流程：
# 1. apt update：从仓库下载 Packages.gz / InRelease（签名索引）
# 2. apt upgrade：对比已安装版本与索引，计算依赖树，下载并安装

# 查看缓存
$ ls /var/cache/apt/archives/           # 下载的 .deb 包
$ ls /var/lib/apt/lists/                # 仓库索引文件

# 清理缓存
$ sudo apt clean                        # 清空 /var/cache/apt/archives/
$ sudo apt autoclean                    # 只删除无法下载的旧版本

# 查看仓库索引
$ ls /var/lib/apt/lists/ | grep ubuntu
archive.ubuntu.com_ubuntu_dists_noble_InRelease
archive.ubuntu.com_ubuntu_dists_noble_main_binary-amd64_Packages
# InRelease = 带内签名的索引（现代标准）
# Release + Release.gpg = 旧式分离签名
```
## 10.1.2 仓库管理与 GPG 签名验证
```bash
# 1. 查看已配置的仓库
$ cat /etc/apt/sources.list
deb http://archive.ubuntu.com/ubuntu noble main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu noble-updates main restricted
deb http://security.ubuntu.com/ubuntu noble-security main restricted

# 2. 添加第三方仓库（现代安全方式）
$ sudo mkdir -p /etc/apt/keyrings
$ curl -fsSL https://repo.example.com/key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/example.gpg
$ echo "deb [signed-by=/etc/apt/keyrings/example.gpg] https://repo.example.com/ubuntu noble main" | sudo tee /etc/apt/sources.list.d/example.list
# signed-by = 明确指定密钥，不依赖全局 apt-key（已弃用）

# 3. 验证签名
$ sudo apt update
# 如果签名错误：
# W: GPG error: ... The following signatures couldn't be verified because the public key is not available
$ sudo gpg --keyring /etc/apt/keyrings/example.gpg --verify /var/lib/apt/lists/repo.example.com_ubuntu_dists_noble_InRelease
# 或重新下载密钥

# 4. 查看仓库优先级（Pinning）
$ cat /etc/apt/preferences.d/99-pinning
Package: nginx
Pin: origin repo.example.com
Pin-Priority: 900
# 优先级：>1000 = 强制安装该版本，990 = 默认，100 = 不自动安装，<0 = 从不安装
```
## 10.1.3 版本锁定与回滚
```bash
# 1. 查看可用版本
$ apt-cache policy nginx
nginx:
  Installed: 1.24.0-1ubuntu1
  Candidate: 1.24.0-2ubuntu1
  Version table:
     1.24.0-2ubuntu1 500
        500 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 Packages
 *** 1.24.0-1ubuntu1 100
        100 /var/lib/dpkg/status

# 2. 安装特定版本
$ sudo apt install nginx=1.24.0-1ubuntu1

# 3. 锁定版本（防止自动升级）
$ sudo apt-mark hold nginx
nginx set on hold.
$ apt-mark showhold
nginx

# 4. 查看锁定状态
$ dpkg --get-selections | grep hold
nginx                                         hold

# 5. 解锁
$ sudo apt-mark unhold nginx

# 6. 批量锁定（生产环境基线）
$ cat /usr/local/bin/lock-versions.sh
#!/bin/bash
# 锁定所有关键包，防止意外升级
CRITICAL_PKGS=(nginx postgresql-16 redis-server docker-ce)
for pkg in "${CRITICAL_PKGS[@]}"; do
    if dpkg -l "$pkg" >/dev/null 2>&1; then
        apt-mark hold "$pkg"
        echo "Locked: $pkg"
    fi
done

# 7. 紧急回滚（如果升级后出问题）
$ sudo apt install nginx=1.24.0-1ubuntu1  # 降级到旧版本
$ sudo apt --reinstall install nginx        # 重装当前版本
```

## 10.1.4 dpkg 底层操作与故障恢复
```bash
# 1. 查看包内容
$ dpkg -L nginx                           # 包安装了哪些文件
$ dpkg -l nginx                           # 包状态
$ dpkg -s nginx                           # 包详细信息（状态、依赖、描述）

# 2. 查找文件所属包
$ dpkg -S /etc/nginx/nginx.conf
nginx: /etc/nginx/nginx.conf
$ dpkg -S $(which nginx)
nginx: /usr/sbin/nginx

# 3. 包状态码（第一个字母）
$ dpkg -l | head
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/Half-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name           Version           Architecture Description
+++-==============-=================-============-=================================
ii  nginx          1.24.0-1ubuntu1   amd64        small, powerful, scalable web/proxy server
# ii = 期望安装/已安装/无错误
# rc = 已删除但配置文件残留
# iF = 安装失败

# 快速判断系统包状态是否"干净"：
$ dpkg -l | grep -E '^rc'                  # rc = 已删配置残留，可用 apt purge <pkg> 清理
$ dpkg -l | grep -E '^iU|^iF'              # iU/iF = 安装中断/失败，需 sudo dpkg --configure -a

# 4. 强制重新配置
$ sudo dpkg-reconfigure nginx             # 重新执行postinst脚本

# 5. 修复损坏的包
$ sudo dpkg --configure -a                # 配置所有未配置的包
$ sudo apt --fix-broken install           # 修复依赖关系

# 6. 强制安装（绕过依赖检查，危险！）
$ sudo dpkg -i --force-depends package.deb

# 7. 从 .deb 提取文件（不安装）
$ dpkg-deb -x nginx_1.24.deb /tmp/nginx-extract    # 提取数据文件
$ dpkg-deb -e nginx_1.24.deb /tmp/nginx-control    # 提取控制脚本
```
## 10.2 dnf / rpm 深度实战（RHEL/CentOS/Fedora）

dnf 是 RHEL 8+ 的标准包管理器，最显著的变化是引入了模块流（Modular Stream）概念。

## 10.2.1 dnf 模块流（Modular Stream）
RHEL 8+ 引入的 模块（Module） 机制，允许同一软件多版本共存：
```bash
# 1. 查看可用模块
$ dnf module list
Name                Stream           Profiles           Summary
nodejs              18               default, development, minimal  Javascript runtime
nodejs              20               default, development, minimal  Javascript runtime
nodejs              22               default, development, minimal  Javascript runtime
postgresql          15               client, server     PostgreSQL server
postgresql          16               client, server     PostgreSQL server

# 2. 启用特定版本模块
$ sudo dnf module enable nodejs:20
$ sudo dnf install nodejs             # 安装的是 20 版

# 3. 切换模块版本
$ sudo dnf module reset nodejs        # 重置
$ sudo dnf module enable nodejs:22
$ sudo dnf update nodejs              # 升级到 22 版

# 4. 查看模块详情
$ dnf module info postgresql:16
```
## 10.2.2 版本锁定与热修补
```bash
# 1. 安装版本锁定插件
$ sudo dnf install python3-dnf-plugin-versionlock

# 2. 锁定版本
$ sudo dnf versionlock add nginx-1:1.24.0-1.el9.x86_64
$ sudo dnf versionlock list
nginx-0:1.24.0-1.el9.*

# 3. 排除特定包升级
$ sudo dnf update --exclude=nginx,kernel*

# 4. 内核热修补（Kernel Live Patching）
$ sudo dnf install kpatch-dnf
$ sudo dnf kpatch auto                # 自动应用热补丁
$ sudo kpatch list                    # 查看已应用补丁
kpatch_5_14_0_362_el9_1 [enabled]

# 5. 查看安全更新
$ sudo dnf updateinfo list --security
$ sudo dnf updateinfo list --cves CVE-2024-0001
$ sudo dnf update --security          # 仅安装安全更新
```
## 10.2.3 rpm 数据库与验证
```bash
# 1. rpm 数据库位置
$ ls /var/lib/rpm/
Packages  # Berkeley DB 格式

# 2. 验证包完整性（检测文件被篡改）
$ sudo rpm -V nginx
S.5....T.  c /etc/nginx/nginx.conf
# S = 大小改变, 5 = MD5校验失败, T = 时间戳改变, c = 配置文件（预期内）
# 如果看到 .M... 表示权限被修改，可能是入侵迹象！

# 3. 从 rpm 恢复被删除的配置文件
$ sudo rpm -qf /etc/nginx/nginx.conf   # 确认所属包
$ sudo rpm -iv --force --nodeps nginx.rpm  # 强制重装（危险）
# 或更优雅：
$ sudo dnf reinstall nginx

# 4. 查看包脚本（安装前后执行的脚本）
$ rpm -q --scripts nginx
postinstall scriptlet (using /bin/sh):
if [ $1 -eq 1 ] ; then
    systemctl preset nginx.service >/dev/null 2>&1 || :
fi

# 5. 查看包变更日志
$ rpm -q --changelog nginx | head -20
# 可用于排查"升级后为什么行为变了"
```
## 10.3 生产环境安全更新策略

"装完就不管"是生产环境最大的安全隐患。安全更新策略需要兼顾及时性与稳定性。

## 10.3.1 自动安全更新配置
```bash
# === Debian/Ubuntu ===
$ sudo apt install unattended-upgrades
$ sudo dpkg-reconfigure unattended-upgrades  # 启用

# 配置：/etc/apt/apt.conf.d/50unattended-upgrades
$ cat <<'EOF' | sudo tee /etc/apt/apt.conf.d/50unattended-upgrades
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
    "${distro_id}ESMApps:${distro_codename}-apps-security";
    "${distro_id}ESM:${distro_codename}-infra-security";
};
Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::MinimalSteps "true";
Unattended-Upgrade::InstallOnShutdown "false";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "false";           # 生产环境手动重启
Unattended-Upgrade::Automatic-Reboot-Time "03:00";
Unattended-Upgrade::SyslogEnable "true";
EOF

# 测试运行（不实际安装）
$ sudo unattended-upgrade --dry-run --debug

# === RHEL/CentOS ===
$ sudo dnf install dnf-automatic
$ sudo systemctl enable --now dnf-automatic.timer

# 配置：/etc/dnf/automatic.conf
$ cat <<'EOF' | sudo tee /etc/dnf/automatic.conf
[commands]
upgrade_type = security
random_sleep = 3600
network_online_timeout = 60
download_updates = yes
apply_updates = yes

[emitters]
emit_via = motd

[email]
email_from = root@example.com
email_to = admin@example.com
email_host = localhost
EOF
```
## 10.3.2 更新后检查与重启决策
```bash
# 1. 检查哪些服务需要重启（Debian）
$ sudo apt install debian-goodies
$ sudo checkrestart
Found 3 processes using old versions of upgraded files:
(1) /usr/sbin/nginx (PID 1200) - nginx.service
(2) /usr/lib/postgresql/16/bin/postgres (PID 3000) - postgresql@16-main.service
(3) /usr/bin/python3 (PID 5000) - myapp.service

# 2. 检查哪些服务需要重启（RHEL）
$ sudo dnf install dnf-utils
$ sudo needs-restarting -r
Core libraries or services have been updated:
  systemd -> 252-18.el9
  glibc -> 2.34-60.el9
Reboot is required to ensure the system benefits from these updates.

# 3. 检查内核是否需要重启
$ [ -f /var/run/reboot-required ] && echo "REBOOT REQUIRED" || echo "No reboot needed"
# Debian 特有

# 4. 优雅重启策略（滚动更新场景）
$ cat /usr/local/bin/rolling-restart.sh
#!/bin/bash
set -euo pipefail

SERVICES=(nginx postgresql myapp)
for svc in "${SERVICES[@]}"; do
    echo "Restarting $svc..."
    sudo systemctl reload-or-restart "$svc"  # 优先reload（不丢连接）
    sleep 5
    if ! systemctl is-active --quiet "$svc"; then
        echo "ERROR: $svc failed to start!" >&2
        exit 1
    fi
done
echo "All services restarted successfully"
```
## 10.4 创建自定义软件包

当内部开发的软件需要标准化部署时，打成系统包比"手动拷二进制"可靠得多。

## 10.4.1 创建 deb 包（Debian/Ubuntu）
```bash
# 1. 创建目录结构
$ mkdir -p myapp_1.0.0_amd64/{DEBIAN,usr/local/bin,etc/myapp,lib/systemd/system}

# 2. 控制文件
$ cat > myapp_1.0.0_amd64/DEBIAN/control <<'EOF'
Package: myapp
Version: 1.0.0
Section: utils
Priority: optional
Architecture: amd64
Depends: libc6 (>= 2.35), systemd
Maintainer: Alice <alice@example.com>
Description: My Application
 A production-ready application with systemd integration.
EOF

# 3. 安装脚本
$ cat > myapp_1.0.0_amd64/DEBIAN/postinst <<'EOF'
#!/bin/bash
set -e
useradd -r -s /usr/sbin/nologin myapp || true
systemctl daemon-reload
systemctl enable myapp.service
EOF
chmod 755 myapp_1.0.0_amd64/DEBIAN/postinst

# 4. 文件放入对应目录
$ cp /path/to/myapp myapp_1.0.0_amd64/usr/local/bin/
$ cp /path/to/config.yml myapp_1.0.0_amd64/etc/myapp/
$ cat > myapp_1.0.0_amd64/lib/systemd/system/myapp.service <<'EOF'
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=myapp
ExecStart=/usr/local/bin/myapp
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 5. 构建
$ dpkg-deb --build myapp_1.0.0_amd64
dpkg-deb: building package 'myapp' in 'myapp_1.0.0_amd64.deb'.

# 6. 验证
$ dpkg-deb -I myapp_1.0.0_amd64.deb    # 查看包信息
$ dpkg-deb -c myapp_1.0.0_amd64.deb    # 查看包内容
$ sudo dpkg -i myapp_1.0.0_amd64.deb   # 安装测试
## 10.4.2 创建 rpm 包（RHEL/CentOS）
```bash
# 使用 rpmbuild
$ mkdir -p ~/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS}

# 创建 spec 文件
$ cat > ~/rpmbuild/SPECS/myapp.spec <<'EOF'
Name:           myapp
Version:        1.0.0
Release:        1%{?dist}
Summary:        My Application
License:        MIT
Source0:        myapp-1.0.0.tar.gz

%description
A production-ready application.

%prep
%setup -q

%install
mkdir -p %{buildroot}/usr/local/bin
mkdir -p %{buildroot}/etc/myapp
mkdir -p %{buildroot}/lib/systemd/system
cp myapp %{buildroot}/usr/local/bin/
cp config.yml %{buildroot}/etc/myapp/
cp myapp.service %{buildroot}/lib/systemd/system/

%files
/usr/local/bin/myapp
%config(noreplace) /etc/myapp/config.yml
/lib/systemd/system/myapp.service

%post
%systemd_post myapp.service

%preun
%systemd_preun myapp.service

%postun
%systemd_postun_with_restart myapp.service

%changelog
* Wed Jul 03 2026 Alice <alice@example.com> - 1.0.0-1
- Initial release
EOF

# 构建
$ rpmbuild -ba ~/rpmbuild/SPECS/myapp.spec
$ ls ~/rpmbuild/RPMS/x86_64/
myapp-1.0.0-1.el9.x86_64.rpm

# 构建后质检（RPM 发布前的必经之路）
$ rpmlint ~/rpmbuild/SPECS/myapp.spec
$ rpmlint ~/rpmbuild/RPMS/x86_64/myapp-1.0.0-1.el9.x86_64.rpm
# 0 errors, 0 warnings 才是合格包
```
## 10.5 实战：生产环境包管理检查清单
```bash
#!/bin/bash
# /usr/local/bin/pkg-audit.sh
set -euo pipefail

echo "=== 软件包安全审计 $(date) ==="

echo "1. 可升级包（不含安全更新）："
apt list --upgradable 2>/dev/null | grep -v "Listing..." | wc -l

echo "2. 安全更新待安装："
apt-get -s upgrade | grep -i security | wc -l

echo "3. 版本锁定状态："
apt-mark showhold | while read pkg; do
    echo "  HOLD: $pkg ($(dpkg -l "$pkg" | awk 'NR==6{print $3}'))"
done

echo "4. 第三方仓库："
grep -r "^deb " /etc/apt/sources.list /etc/apt/sources.list.d/ 2>/dev/null | grep -v "ubuntu.com\|debian.org" | while read line; do
    echo "  THIRD-PARTY: $line"
done

echo "5. 过期签名密钥："
apt-key list 2>/dev/null | grep -i expired | wc -l

echo "6. 损坏的包："
dpkg -l | grep -E '^..(F|H)' | wc -l
```

---

## 本章小结

| 场景 | Debian/Ubuntu | RHEL/CentOS |
|------|---------------|-------------|
| 更新索引 | apt update | dnf check-update |
| 升级 | apt upgrade | dnf upgrade |
| 安全更新 | unattended-upgrade | dnf update --security |
| 版本锁定 | apt-mark hold | dnf versionlock |
| 查看文件归属 | dpkg -S | rpm -qf |
| 验证完整性 | debsums / dpkg -V | rpm -V |
| 创建包 | dpkg-deb | rpmbuild |

---

> **本章字数**：约 8,500 字

---

## 课后练习

1. 概念题：Debian 系的 .deb 包和 Red Hat 系的 .rpm 包，在依赖解决方面，包管理器（APT/DNF）分别起到什么作用？

思路：APT 和 DNF 都会从仓库读取元数据并自动计算依赖树。它们不仅能安装，还能处理版本冲突和提供安全更新。

2. 实操题：在 Ubuntu 上，如何锁定 nginx 版本，防止 apt upgrade 将其升级？

思路：sudo apt-mark hold nginx。查看已锁定包 apt-mark showhold，解锁 sudo apt-mark unhold nginx。

3. 排障题：apt update 报错 GPG error: ... NO_PUBKEY ABCDEF123456。如何解决？

思路：缺少公钥。使用 sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys ABCDEF123456 导入，或按现代方式将公钥放到 /etc/apt/keyrings/ 并用 signed-by 指定。

4. 安全题：怀疑系统上的 /etc/ssh/sshd_config 被人篡改，如何用 rpm 验证该文件是否与原包一致？

思路：rpm -V openssh-server。如果输出 S.5....T. c /etc/ssh/sshd_config，说明文件大小(S)、MD5(5)、时间戳(T) 改变了（c 表示配置文件，允许变化，但应审计）。

5. 对比题：apt purge 和 apt remove 的区别是什么？

思路：remove 只删除二进制文件，保留配置文件（/etc 下的）；purge 会连配置文件一起删除。

6. 实操题：在 RHEL 9 中，如何只安装安全相关的更新，而不升级普通功能包？

思路：sudo dnf update --security。或使用 dnf-automatic 配置 upgrade_type = security。

7. 排障题：dpkg -i myapp.deb 报依赖缺失，如何让系统自动从仓库补全依赖？

思路：sudo apt --fix-broken install，它会自动尝试补全缺失的依赖并完成安装。

8. 综合题：公司内部使用自定义源，要求配置仓库时必须验证 GPG 签名且不依赖全局 apt-key。请写出 /etc/apt/sources.list.d/internal.list 的正确内容格式。

思路：deb [signed-by=/etc/apt/keyrings/internal.gpg] https://internal.repo/ubuntu noble main。其中 internal.gpg 是提前 curl 下载并用 gpg --dearmor 转换过的二进制密钥文件。
# 第十一章：安全与审计

> **本章定位**：Linux安全是纵深防御体系，不是单一工具或配置。从SSH加固到内核安全模块，从审计日志到漏洞管理，理解"攻击面分析→防护→检测→响应"的闭环。

## 11.1 安全模型与攻击面分析

**一句话定义**：Linux安全 = 最小权限原则 + 纵深防御 + 可观测性，攻击面分析是识别"敌人可能从哪进来"的系统性方法。

### 攻击面分层模型

```
┌─────────────────────────────────────────┐
│  应用层（Web应用、API、数据库）           │  ← SQL注入、XSS、逻辑漏洞
├─────────────────────────────────────────┤
│  服务层（SSH、HTTP、DNS、邮件）           │  ← 暴力破解、协议漏洞、配置错误
├─────────────────────────────────────────┤
│  容器/虚拟化层（Docker、KVM、Namespace）  │  ← 容器逃逸、资源耗尽、镜像漏洞
├─────────────────────────────────────────┤
│  系统调用层（syscall、Capabilities、Seccomp）│  ← 提权、越权、系统调用滥用
├─────────────────────────────────────────┤
│  内核层（模块、驱动、eBPF、调度器）        │  ← 内核漏洞、驱动漏洞、rootkit
├─────────────────────────────────────────┤
│  硬件/固件层（BIOS/UEFI、TPM、微码）       │  ← 供应链攻击、固件植入
└─────────────────────────────────────────┘
最小权限原则的实践
```bash
# 1. 用户权限：能不用root就不用
$ id www-data
uid=33(www-data) gid=33(www-data) groups=33(www-data)
# www-data只能读写/var/www，不能读/etc/shadow

# 2. 文件权限：默认拒绝，显式允许
$ ls -l /etc/shadow
-rw-r----- 1 root shadow 1234 Jul  3 10:00 /etc/shadow
# 只有root和shadow组成员能读

# 3. 网络权限：默认关闭，按需开放
$ ss -tlnp | grep -v "127.0.0.1\|::1"
# 只监听必要的公网端口

# 4. 能力（Capabilities）：精确授权
$ getcap /usr/bin/ping
/usr/bin/ping cap_net_raw=ep
# ping只需要CAP_NET_RAW，不需要整个root

# 5. 容器权限：默认受限，显式放宽
$ docker run --rm --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx
# 先丢弃所有能力，再按需添加
```
## 11.2 SSH安全加固：远程访问的第一道门

SSH 是大多数 Linux 服务器的唯一入口——加固它就是加固整个防线的最前沿。

## 11.2.1 现代SSH配置模板（OpenSSH 9.x）
```bash
# /etc/ssh/sshd_config
# 基于 Ubuntu 24.04 / RHEL 9 / OpenSSH 9.7+

# === 网络层加固 ===
Port 2222                           # 改端口（减少自动化扫描噪音）
# ⚠ 不是安全机制，只是减少日志量。真正安全靠密钥+防火墙
AddressFamily inet                  # 如果不需要IPv6，禁用减少攻击面
ListenAddress 10.0.0.5              # 仅监听管理网卡（如有跳板机或VPN）

# === 认证层：禁用密码，强制密钥 ===
PermitRootLogin no                  # 禁止root直接登录（强制普通用户+sudo审计）
PasswordAuthentication no           # 完全禁用密码认证
PubkeyAuthentication yes
AuthenticationMethods publickey     # 只允许公钥，拒绝其他方法
MaxAuthTries 3                      # 3次失败后断开（防暴力破解）
MaxSessions 2                       # 每个连接最大会话数
LoginGraceTime 30                   # 30秒内必须完成认证

# === 密钥算法：禁用弱算法，防降级攻击 ===
# OpenSSH 9.x默认已禁用RSA<2048和DSA，但显式声明更安全
HostKeyAlgorithms ssh-ed25519-cert-v01@openssh.com,ssh-ed25519,ecdsa-sha2-nistp521-cert-v01@openssh.com,ecdsa-sha2-nistp384-cert-v01@openssh.com,ecdsa-sha2-nistp256-cert-v01@openssh.com,ecdsa-sha2-nistp521,ecdsa-sha2-nistp384,ecdsa-sha2-nistp256
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-512,hmac-sha2-256,umac-128@openssh.com
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,ecdh-sha2-nistp521,ecdh-sha2-nistp384,ecdh-sha2-nistp256,diffie-hellman-group-exchange-sha256

# === 会话层：防空闲连接占用资源 ===
ClientAliveInterval 300             # 每5分钟发送keepalive探测
ClientAliveCountMax 2               # 2次无响应断开（总超时10分钟）
TCPKeepAlive no                     # 不依赖TCP keepalive（不可靠，中间设备可能静默丢弃）

# === 访问控制：白名单机制 ===
AllowUsers alice bob                # 只允许特定用户（比DenyUsers更安全）
AllowGroups ssh-users               # 只允许特定组
# DenyUsers guest temp              # 黑名单作为补充

# === 转发限制：防止跳板攻击 ===
AllowTcpForwarding no               # 禁止TCP端口转发（防内网跳板）
GatewayPorts no                     # 禁止绑定公网端口（只允许127.0.0.1）
X11Forwarding no                    # 服务器通常不需要图形界面
PermitTunnel no                     # 禁止tun设备隧道

# === 日志与审计 ===
SyslogFacility AUTH
LogLevel VERBOSE                    # 记录公钥指纹（便于审计追踪）
# 查看：journalctl -u sshd | grep "Accepted publickey"

# === 子系统配置：SFTP专用用户（Chroot jail）===
Match User sftp-only
    ForceCommand internal-sftp      # 强制SFTP，禁止shell
    ChrootDirectory /srv/sftp/%u    # 限制在指定目录（%u=用户名）
    AllowTcpForwarding no
    X11Forwarding no
    PasswordAuthentication no
    PubkeyAuthentication yes

# === 验证与重载 ===
$ sudo sshd -t                      # 验证配置语法（不启动）
$ sudo systemctl reload sshd        # 优雅重载：现有连接不中断，新连接用新配置
```
## 11.2.2 密钥管理与轮换策略
```bash
# 1. 生成现代密钥（Ed25519首选）
$ ssh-keygen -t ed25519 -C "alice@web-server-01-2026" -f ~/.ssh/id_ed25519_web01
# -t ed25519：基于Curve25519，256位安全性，密钥仅68字节，签名速度快
# -C：注释包含主机名和日期，便于审计和轮换追踪
# 私钥权限必须600：
$ chmod 600 ~/.ssh/id_ed25519_web01

# 2. 禁止旧算法（客户端配置）
$ cat >> ~/.ssh/config <<'EOF'
Host *
    # 拒绝RSA（<3072位）和DSA（已破解）
    HostKeyAlgorithms +ssh-ed25519,ecdsa-sha2-nistp521,ecdsa-sha2-nistp384
    PubkeyAcceptedAlgorithms +ssh-ed25519,ecdsa-sha2-nistp521,ecdsa-sha2-nistp384
    # 拒绝弱KEX和MAC
    KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,ecdh-sha2-nistp521,ecdh-sha2-nistp384,ecdh-sha2-nistp256
    MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
    Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
EOF

# 3. 密钥轮换流程（90天周期）
$ cat /usr/local/bin/ssh-key-rotate.sh
#!/bin/bash
set -euo pipefail

USER=${1:?Usage: $0 <username>}
DATE=$(date +%Y%m%d)
KEY_DIR="/home/$USER/.ssh"
NEW_KEY="${KEY_DIR}/id_ed25519_${DATE}"

# 生成新密钥对
sudo -u "$USER" ssh-keygen -t ed25519 -f "$NEW_KEY" -N "" -C "$USER@$HOSTNAME-$DATE"
echo "=== 新密钥已生成 ==="
echo "公钥文件: ${NEW_KEY}.pub"
echo "指纹: $(ssh-keygen -lf ${NEW_KEY}.pub)"

echo ""
echo "=== 操作步骤 ==="
echo "1. 将公钥添加到所有目标服务器的 authorized_keys"
echo "2. 测试新密钥登录: ssh -i ${NEW_KEY} user@server"
echo "3. 30天后删除旧密钥: find ${KEY_DIR} -name 'id_ed25519_*' -mtime +30 -delete"
echo "4. 从服务器 authorized_keys 中移除旧公钥"

# 4. 审计已授权密钥（排查幽灵密钥）
$ sudo find /home -name authorized_keys -exec echo "=== {} ===" \; -exec cat {} \; 2>/dev/null | while read key; do
    if [[ "$key" == ssh-* ]]; then
        echo "$key" | ssh-keygen -lf - 2>/dev/null
    fi
done
# 输出示例：
# 256 SHA256:AbCdEf123... alice@web-server-01-2026 (ED25519)
# 256 SHA256:GhIjKl456... bob@db-server-02-2026 (ED25519)
# 发现未知指纹？立即调查！

# 5. 使用SSH证书（大规模环境推荐）
# 相比authorized_keys，证书支持过期时间和 principals（角色）
$ ssh-keygen -s ca_key -I alice@corp -n alice,devteam -V +52w alice_key.pub
# 生成 alice_key-cert.pub，52周后过期，属于alice用户和devteam角色
```
## 11.2.3 失败登录监控与自动封禁
```bash
# === 方案1：fail2ban（轻量级，适合中小规模）===
$ sudo apt install fail2ban

$ cat /etc/fail2ban/jail.local
[DEFAULT]
bantime = 3600                      # 封禁1小时
findtime = 300                      # 5分钟内
maxretry = 3                        # 3次失败即封禁
backend = systemd                   # 使用journald（比文件解析高效）

[sshd]
enabled = true
port = 2222                         # 匹配你的SSH端口
filter = sshd
logpath = /var/log/auth.log         # Debian/Ubuntu
# logpath = /var/log/secure         # RHEL/CentOS
action = iptables-multiport[name=sshd, port="2222", protocol=tcp]
# 或使用nftables：
# action = nftables-multiport[name=sshd, port="2222", protocol=tcp]

# 查看状态
$ sudo fail2ban-client status sshd
$ sudo fail2ban-client status sshd | grep "Banned IP list"
# 手动解封（误封时）
$ sudo fail2ban-client set sshd unbanip 192.168.1.100

# === 方案2：自定义iptables限速（无依赖）===
$ cat <<'EOF' | sudo tee /etc/iptables/rules.v4
# SSH速率限制：每5分钟最多5个新连接
-A INPUT -p tcp --dport 2222 -m state --state NEW -m recent --set --name SSH
-A INPUT -p tcp --dport 2222 -m state --state NEW -m recent --update --seconds 300 --hitcount 5 --name SSH -j DROP
-A INPUT -p tcp --dport 2222 -m state --state NEW -j ACCEPT
EOF

# === 方案3：TOTP双因素认证（高安全环境）===
$ sudo apt install libpam-google-authenticator
$ google-authenticator                  # 用户自行配置，生成二维码
# 然后修改 /etc/pam.d/sshd 添加：
# auth required pam_google_authenticator.so
# 修改 /etc/ssh/sshd_config：
# AuthenticationMethods publickey,keyboard-interactive
# 需要同时提供密钥+手机验证码
```
## 11.3 PAM与认证框架深度配置

PAM（Pluggable Authentication Modules）是 Linux 认证体系的"插线板"——SSH、sudo、login 都通过它完成用户验证。

## 11.3.1 PAM配置结构解析
```bash
# PAM（Pluggable Authentication Modules）是Linux认证的核心框架
# 配置文件：/etc/pam.d/ 目录下的服务同名文件

$ ls /etc/pam.d/
common-auth      # 认证（你是谁？密码对不对？）
common-account   # 账号（账号是否过期？是否被锁定？）
common-password  # 密码变更（新密码是否符合策略？）
common-session   # 会话（登录后设置什么环境？资源限制？）
sshd             # SSH专用规则（继承common-*）
sudo             # sudo专用规则
su               # su专用规则

# 每条规则格式：
# type    control    module-path    [module-arguments]
# 示例：
auth    required     pam_unix.so    nullok_secure
# type=auth, control=required, module=pam_unix, arg=nullok_secure
```
## 11.3.2 控制标志（Control Flags）详解
控制标志  行为  场景
required  必须成功，但失败不立即返回，继续后续模块  密码验证
requisite 必须成功，失败立即返回失败 关键安全检查
sufficient  成功则立即返回成功，失败忽略  备用认证方式（如指纹）
optional  成功与否不影响总体结果 非关键模块
include 包含另一个配置文件 复用common规则
substack  类似include，但失败不传播  子流程
```bash
# 实际例子：/etc/pam.d/sshd
auth       required     pam_sepermit.so          # SELinux上下文检查
auth       substack     common-auth              # 包含标准认证（密码/密钥）
auth       include      postlogin                # 登录后处理

# common-auth 内部：
auth    [success=2 default=ignore]  pam_unix.so nullok_secure  # 本地密码
auth    [success=1 default=ignore]  pam_sss.so use_first_pass   # SSSD/LDAP
auth    requisite                     pam_deny.so                 # 都失败则拒绝
auth    required                      pam_permit.so               # 兜底允许

# [success=2 default=ignore] 语法：
# success=2 → 如果成功，跳过后面2条规则
# default=ignore → 其他情况忽略本条结果，继续
```
## 11.3.3 密码策略与复杂度
```bash
# 安装密码质量检查模块
$ sudo apt install libpam-pwquality    # Debian/Ubuntu
$ sudo dnf install libpwquality        # RHEL

# 配置：/etc/security/pwquality.conf
$ cat <<'EOF' | sudo tee /etc/security/pwquality.conf
minlen = 16                           # 最小16位（长密码优于复杂短密码）
dcredit = -1                          # 至少1位数字
ucredit = -1                          # 至少1位大写
lcredit = -1                          # 至少1位小写
ocredit = -1                          # 至少1位特殊字符
minclass = 3                          # 至少3类字符（降低用户记忆难度）
maxrepeat = 3                         # 最多连续重复3次
maxsequence = 3                       # 最多连续序列3位（如123, abc）
dictcheck = 1                         # 检查字典单词
usercheck = 1                         # 检查是否包含用户名
enforcing = 1                         # 强制执行（0=仅警告）
EOF

# PAM配置应用pwquality
$ cat /etc/pam.d/common-password
password    requisite     pam_pwquality.so retry=3
password    [success=1 default=ignore]  pam_unix.so obscure use_authtok try_first_pass yescrypt
# retry=3：给用户3次尝试输入符合要求的密码

# 测试密码强度
$ pwscore
mypassword123!
# 输出分数：0-100，<50弱，50-80中，>80强
```
## 11.3.4 登录失败锁定（防暴力破解）
```bash
# 使用 pam_faillock（现代标准，替代旧的pam_tally2）
$ cat <<'EOF' | sudo tee /etc/pam.d/common-auth
# 在认证前检查是否已锁定
auth    required    pam_faillock.so preauth silent audit deny=5 unlock_time=600
# 标准认证
auth    [success=1 default=ignore]  pam_unix.so nullok_secure
# 认证后记录失败
auth    [default=die]               pam_faillock.so authfail audit deny=5 unlock_time=600
# 如果成功，重置失败计数
auth    sufficient                  pam_faillock.so authsucc audit deny=5 unlock_time=600
EOF

# 配置详解：
# deny=5：5次失败锁定
# unlock_time=600：锁定10分钟
# audit：记录到audit日志
# silent：preauth阶段不显示错误（防止信息泄露）

# 查看锁定状态
$ sudo faillock --user alice
When                Type  Source                                           Valid
2026-07-03 09:00:01 RHOST 192.168.1.100                                     V
2026-07-03 09:00:15 RHOST 192.168.1.100                                     V
# V = 有效失败记录，达到5次即锁定

# 手动解锁
$ sudo faillock --user alice --reset
```
## 11.3.5 资源限制（ulimit与PAM集成）
```bash
# /etc/security/limits.conf 通过PAM的pam_limits.so生效
$ cat <<'EOF' | sudo tee /etc/security/limits.d/99-production.conf
# 全局限制
*           soft    nofile      65536
*           hard    nofile      65536
*           soft    nproc       65536
*           hard    nproc       65536

# 特定服务用户（防止fork炸弹）
www-data    hard    nproc       4096
www-data    hard    fsize       104857600    # 最大100MB文件

# 数据库用户（大内存需求）
mysql       soft    nofile      65536
mysql       hard    nofile      65536

# 禁止core dump（防信息泄露）
*           hard    core        0

# 最大登录数
*           hard    maxlogins   10
EOF

# 验证
$ ulimit -n                           # 当前shell的soft limit
65536
$ ulimit -Hn                          # hard limit
65536

# 注意：systemd服务不受limits.conf影响！
# 需要在service文件中设置：
# [Service]
# LimitNOFILE=65536
# LimitNPROC=4096
```
## 11.4 SELinux与AppArmor：强制访问控制

> **🧭 延伸阅读**：除了软件层面的 MAC 机制，**Intel 11 代+ / AMD Zen 4+ CPU 提供了硬件级的控制流保护——影子栈（Shadow Stack）**。它通过 CPU 内部的"防篡改返回地址副本"从硅片层面堵死 ROP 攻击。Linux 6.6+ 已支持影子栈。详见**第十三章 13.4 节**。


传统 DAC（自由访问控制）的缺陷是：进程获得用户权限后可以为所欲为。MAC（强制访问控制）补上了这个缺口——即使 root 也要遵守策略。

## 11.4.1 SELinux深度实战（RHEL/CentOS）
**一句话定义**：SELinux是标签-based的强制访问控制（MAC），给每个进程和文件打安全标签，内核根据策略决定"谁可以对什么做什么"，即使root也要遵守。
```bash
# 1. 查看当前模式
$ getenforce
Enforcing                            # Enforcing=强制, Permissive=仅记录, Disabled=关闭
$ sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33

# 2. 理解标签（Label）
$ ls -Z /etc/nginx/nginx.conf
system_u:object_r:nginx_conf_t:s0 /etc/nginx/nginx.conf
# 用户:角色:类型:灵敏度
# ↑ 类型（Type）是最关键的，targeted策略主要用类型强制（TE）

$ ps -eZ | grep nginx
system_u:system_r:nginx_t:s0  1200 ?  00:00:00 nginx
# nginx进程运行在nginx_t域

# 3. 查看策略规则（允许什么）
$ sesearch -A -s nginx_t -t httpd_sys_content_t
# 查看nginx_t域对httpd_sys_content_t类型的允许操作
allow nginx_t httpd_sys_content_t:file { getattr ioctl lock open read };
# nginx可以读httpd_sys_content_t文件

$ sesearch -A -s nginx_t -t httpd_sys_content_t -p write
# 查是否有write权限 → 无输出 = 没有write权限
# 这就是SELinux阻止nginx写入网站目录的原理

# 4. 排查AVC拒绝（Access Vector Cache）
$ sudo ausearch -m avc -ts today
type=AVC msg=audit(1720000000.123:456): avc:  denied  { write } for  pid=1200 comm="nginx" name="uploads" dev="sda2" ino=131142 scontext=system_u:system_r:nginx_t:s0 tcontext=unconfined_u:object_r:var_t:s0 tclass=dir permissive=0
# 解读：
# { write } = 被拒绝的操作
# scontext=nginx_t = 源（进程）
# tcontext=var_t = 目标（文件/目录）
# tclass=dir = 目标类型是目录
# 原因：nginx尝试写入/var/下的目录，但标签是var_t而非httpd_sys_content_t

# 5. 临时解决（测试用）
$ sudo chcon -R -t httpd_sys_content_t /var/www/uploads
# 改标签为nginx可写的类型

# 6. 永久解决（正确方式）
$ sudo semanage fcontext -a -t httpd_sys_content_t "/var/www/uploads(/.*)?"
$ sudo restorecon -Rv /var/www/uploads
# semanage写入策略数据库，restorecon应用标签

# 7. 生成自定义模块（如果标准策略不满足）
$ sudo audit2allow -a -M myapp_policy
# 根据AVC日志生成策略模块
$ sudo semodule -i myapp_policy.pp
# 加载模块
# ⚠ 审计后再生成，不要盲信audit2allow（可能过度宽松）

# 8. 布尔值（Boolean）开关
$ getsebool -a | grep httpd
httpd_can_network_connect --> off
# 允许httpd连接外部网络（如PHP curl请求）
$ sudo setsebool -P httpd_can_network_connect on
# -P = 永久生效
```
## 11.4.2 AppArmor深度实战（Ubuntu/Debian/SUSE）
**一句话定义**：AppArmor是路径-based的强制访问控制，通过配置文件定义"某个程序可以对哪些路径执行什么操作"，比SELinux简单但灵活性稍低。
```bash
# 1. 查看状态
$ sudo aa-status
apparmor module is loaded.
47 profiles are loaded.
45 profiles are in enforce mode.
   /usr/bin/man
   /usr/sbin/named
   /usr/sbin/nginx
```
   ...
2 profiles are in complain mode.
   /usr/sbin/mysqld
# enforce = 强制，complain = 仅记录不阻止

# 2. 查看nginx配置
```bash
$ cat /etc/apparmor.d/usr.sbin.nginx
#include <tunables/global>

/usr/sbin/nginx {
  #include <abstractions/base>
  #include <abstractions/nameservice>
  #include <abstractions/openssl>
  #include <abstractions/web-data>

  capability dac_override,
  capability dac_read_search,
  capability net_bind_service,
  capability setgid,
  capability setuid,

  /etc/nginx/** r,                   # 只读
  /usr/share/nginx/** r,
  /var/log/nginx/** rw,              # 读写
  /var/lib/nginx/** rw,
  /run/nginx.pid rwk,               # 读写+锁定

  /var/www/** r,                    # 默认只读网站目录
  deny /var/www/** w,               # 显式拒绝写入（除非下面放开）

  # 如果网站需要上传目录
  /var/www/uploads/ rw,
}
```

# 语法：r=读, w=写, rw=读写, k=锁定, l=链接, m=内存映射, ix=继承执行

# 3. 修改配置（添加新路径）
$ sudo vim /etc/apparmor.d/local/usr.sbin.nginx
/var/www/custom-app/ rw,
/var/cache/nginx/ rw,

$ sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx
# -r = 重新加载

# 4.  complain模式调试（不阻止，只记录）
$ sudo aa-complain /usr/sbin/nginx
# 运行一段时间，收集日志
$ sudo grep audit /var/log/syslog | grep apparmor
# 根据日志调整规则后
$ sudo aa-enforce /usr/sbin/nginx

# 5. 生成新配置文件
$ sudo aa-genprof /usr/local/bin/myapp
# 交互式工具：运行myapp执行各种操作，aa-genprof自动学习并生成配置
## 11.4.3 SELinux vs AppArmor 选型
维度  SELinux AppArmor
策略模型  标签（Label） 路径（Path）
复杂度 高（需理解TE、RBAC、MLS） 低（路径+权限直观）
灵活性 极高（可定义任意关系） 中等（路径为主）
默认发行版 RHEL/CentOS/Fedora  Ubuntu/Debian/SUSE
学习曲线  陡峭  平缓
容器支持  完善（container-selinux） 一般
审计工具  audit2allow, sesearch aa-genprof, aa-logprof
生产推荐  RHEL生态首选  Ubuntu生态首选
## 11.5 auditd：内核级审计系统

auditd 是 Linux 内核的"监控摄像头"——它可以记录每一次文件访问、每一条命令执行、每一次系统调用。

## 11.5.1 auditd 架构与性能调优
```bash
# 架构：
# 内核 audit 子系统（netlink接口） → auditd 守护进程 → /var/log/audit/audit.log
#                              ↘ audispd/audisp-syslog → 实时告警/外部系统

# 1. 查看内核审计状态
$ sudo auditctl -s
enabled 1                           # 1=启用, 0=禁用, 2=锁定（无法关闭，需重启）
failure 1                           # 1=审计失败时阻塞系统（安全模式）
pid 1234                            # auditd PID
rate_limit 0                        # 每秒事件数限制（0=无限制）
backlog 64                          # 内核缓冲区大小（条数）
lost 0                              # 丢失事件数（>0说明性能瓶颈，需调大backlog）
backlog_wait_time 60000             # 缓冲区满时等待时间（ms，0=丢弃）

# 2. 性能调优（高并发场景）
$ sudo auditctl -b 8192             # 增大内核缓冲区到8192条
$ sudo auditctl -r 1000             # 限制每秒1000条（防日志风暴压垮磁盘）
$ sudo auditctl --backlog_wait_time 0  # 缓冲区满时丢弃而非阻塞（低延迟场景）
$ sudo auditctl -f 1                # 启用失败时的panic模式（最高安全）

# 3. 查看auditd服务状态
$ sudo systemctl status auditd
$ sudo auditctl -l | wc -l          # 当前加载的规则数
$ sudo auditctl -s | grep lost      # 确认无丢失
```
## 11.5.2 规则编写：从基础到高级
```bash
# 规则文件：/etc/audit/rules.d/*.rules（按文件名排序加载，最后执行augenrules --load）

# === 规则1：监控身份认证文件（核心安全数据）===
-w /etc/passwd -p wa -k identity    # w=写, a=属性变更（权限、属主）
-w /etc/shadow -p wa -k identity    # shadow变更 = 密码修改或用户管理
-w /etc/group -p wa -k identity
-w /etc/gshadow -p wa -k identity
-w /etc/security/ -p wa -k identity
-w /etc/sudoers -p wa -k sudoers
-w /etc/sudoers.d/ -p wa -k sudoers

# === 规则2：监控特权程序执行（提权检测）===
-w /usr/bin/sudo -p x -k privilege_escalation
-w /usr/bin/su -p x -k privilege_escalation
-w /usr/bin/sg -p x -k privilege_escalation
-w /usr/bin/passwd -p x -k privilege_escalation
-w /usr/bin/chsh -p x -k privilege_escalation
-w /usr/bin/chfn -p x -k privilege_escalation
-w /usr/bin/newgrp -p x -k privilege_escalation
-w /usr/bin/pkexec -p x -k privilege_escalation  # polkit，历史上多次提权漏洞

# === 规则3：监控用户/组管理命令 ===
-w /usr/sbin/useradd -p x -k user_mgmt
-w /usr/sbin/usermod -p x -k user_mgmt
-w /usr/sbin/userdel -p x -k user_mgmt
-w /usr/sbin/groupadd -p x -k user_mgmt
-w /usr/sbin/groupmod -p x -k user_mgmt
-w /usr/sbin/groupdel -p x -k user_mgmt

# === 规则4：监控SSH密钥（防止未授权添加）===
-w /root/.ssh/ -p wa -k ssh_keys
-w /home/*/.ssh/ -p wa -k ssh_keys
# 检测authorized_keys被修改

# === 规则5：监控内核模块（防rootkit）===
-a always,exit -F arch=b64 -S init_module -S finit_module -k kernel_modules
-a always,exit -F arch=b64 -S delete_module -k kernel_modules
# init_module/finit_module = 加载模块，delete_module = 卸载模块

# === 规则6：监控文件权限变更（篡改检测）===
-a always,exit -F arch=b64 -S chmod -S fchmod -S fchmodat -k perm_change
-a always,exit -F arch=b64 -S chown -S fchown -S fchownat -S lchown -k ownership_change

# === 规则7：监控网络配置变更 ===
-w /etc/hosts -p wa -k network_config
-w /etc/resolv.conf -p wa -k network_config
-w /etc/nsswitch.conf -p wa -k network_config
-w /etc/NetworkManager/ -p wa -k network_config
-w /etc/netplan/ -p wa -k network_config
-w /etc/iptables/ -p wa -k firewall
-w /etc/nftables.conf -p wa -k firewall
-w /etc/ufw/ -p wa -k firewall

# === 规则8：监控PAM配置（认证绕过检测）===
-w /etc/pam.d/ -p wa -k pam_config
-w /etc/security/ -p wa -k pam_config

# === 规则9：监控定时任务（持久化后门检测）===
-w /etc/cron.d/ -p wa -k cron_config
-w /etc/cron.daily/ -p wa -k cron_config
-w /etc/cron.hourly/ -p wa -k cron_config
-w /var/spool/cron/ -p wa -k cron_config
-w /etc/systemd/system/ -p wa -k systemd_config

# === 规则10：监控审计系统本身（防篡改）===
-w /etc/audit/ -p wa -k audit_config
-w /etc/audit/rules.d/ -p wa -k audit_config
-w /var/log/audit/ -p wa -k audit_log
-w /sbin/auditctl -p x -k audit_tools
-w /sbin/auditd -p x -k audit_tools

# 加载规则
$ sudo augenrules --load              # 合并所有/etc/audit/rules.d/*.rules并加载
$ sudo auditctl -l | head -20         # 验证规则加载
```
## 11.5.3 审计日志查询与威胁狩猎
```bash
# 1. 按标记查询
$ sudo ausearch -k identity -ts today
$ sudo ausearch -k privilege_escalation -ts recent  # 最近10分钟

# 2. 按用户查询
$ sudo ausearch -ua alice -ts today   # 用户alice的所有审计记录

# 3. 按文件查询
$ sudo ausearch -f /etc/passwd        # 谁访问了passwd

# 4. 按系统调用查询
$ sudo ausearch -sc execve -ts today  # 今天所有程序执行

# 5. 按成功/失败过滤
$ sudo ausearch -sv no -ts today      # 今天所有失败操作（排查攻击尝试）

# 6. 生成报告
$ sudo aureport --summary             # 总体摘要
$ sudo aureport -au                   # 认证事件报告
$ sudo aureport -f                    # 文件访问报告
$ sudo aureport -x                    # 可执行文件报告
$ sudo aureport -l                    # 登录报告
$ sudo aureport --failed -au          # 失败认证（排查暴力破解）

# 7. 解析具体事件（实战案例）
$ sudo ausearch -k privilege_escalation -ts today | head -20
type=SYSCALL msg=audit(1720000000.123:456): arch=c000003e syscall=59 success=yes exit=0 a0=7f1234 a1=7f5678 a2=7f9abc a3=0 items=2 ppid=2345 pid=5678 auid=1000 uid=1000 gid=1000 euid=0 suid=0 fsuid=0 egid=1000 sgid=1000 fsgid=1000 tty=pts0 ses=1 comm="sudo" exe="/usr/bin/sudo" key="privilege_escalation"
type=PATH msg=audit(1720000000.123:456): item=0 name="/usr/bin/sudo" inode=131072 dev=08:02 mode=0104755 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:sudo_exec_t:s0 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0
type=PATH msg=audit(1720000000.123:456): item=1 name="/lib64/ld-linux-x86-64.so.2" inode=131073 dev=08:02 mode=0100755 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:ld_so_t:s0 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0
type=USER_ACCT msg=audit(1720000000.123:457): pid=5678 uid=1000 auid=1000 ses=1 msg='op=PAM:accounting grantors=pam_unix acct="alice" exe="/usr/bin/sudo" hostname=? addr=? terminal=/dev/pts0 res=success'
```

**解读**：
- syscall=59 = execve（执行程序）
# uid=1000, euid=0 = alice执行sudo，有效UID变成root
# mode=0104755 = setuid位（u+s）
# res=success = PAM认证成功

# 8. 威胁狩猎：查找异常模式
# 查找非工作时间（22:00-06:00）的特权操作
$ sudo ausearch -k privilege_escalation --start 00:00 --end 06:00 | grep -c "type=SYSCALL"

# 查找短时间内大量失败认证
$ sudo aureport --failed -au --summary | awk '$2 > 10 {print}'  # 失败>10次的用户

# 查找新出现的可执行文件（可能的后门）
$ sudo ausearch -x /usr/local/bin -ts today | grep "type=SYSCALL.*syscall=59"
```
## 11.6 容器安全：从镜像到运行时

容器不是虚拟机——共享内核意味着安全边界更模糊。容器安全需要从镜像构建到运行时全链路覆盖。

## 11.6.1 镜像安全：供应链攻击防护
```bash
# 1. 镜像漏洞扫描（构建时）
$ docker build -t myapp:1.0 .
$ trivy image myapp:1.0               # Trivy扫描
# 或使用Snyk、Clair、Grype

# 2. 最小化基础镜像（攻击面最小化）
# ❌ 不推荐：
FROM ubuntu:24.04                     # 180MB+，包含大量无用工具
RUN apt-get install -y python3        # 又增加一堆依赖

# ✅ 推荐：
FROM python:3.12-slim                 # 60MB，仅Python运行时
# 或
FROM gcr.io/distroless/python3        # 20MB，无shell，无包管理器
# 或
FROM scratch                          # 空镜像，仅复制静态二进制

# 3. 多阶段构建：构建工具不进入生产镜像
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server .

FROM scratch                          # 空基础
COPY --from=builder /app/server .
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8080
USER 65534                            # nobody用户
ENTRYPOINT ["/server"]

# 4. 镜像签名与验证（防篡改）
$ docker trust sign myregistry/myapp:1.0
$ docker trust inspect --pretty myregistry/myapp:1.0
# 启用Docker Content Trust：
$ export DOCKER_CONTENT_TRUST=1
$ docker pull myregistry/myapp:1.0   # 如果未签名，拉取失败

# 5. 镜像来源验证
$ cat /etc/docker/daemon.json
{
  "allow-nondistributable-artifacts": [],
  "registry-mirrors": [],
  "insecure-registries": [],           # 禁止非安全仓库
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```
## 11.6.2 运行时安全：Capabilities与Seccomp
```bash
# 1. 默认Docker Capabilities（12个）
$ docker run --rm alpine cat /proc/self/status | grep Cap
CapEff: 00000000a80425fb              # 有效能力
# 解码：
$ capsh --decode=00000000a80425fb
cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap

# 2. 最小化能力：先全部丢弃，再按需添加
$ docker run --rm \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --cap-add=CHOWN \
  --user 65534:65534 \
  nginx:alpine
# 容器内：
$ cat /proc/self/status | grep Cap
CapEff: 0000000000000200              # 只有NET_BIND_SERVICE
# 即使容器被攻破，攻击者也无法mount、修改系统时间等

# 3. Seccomp：系统调用过滤
# Docker默认启用seccomp，过滤约44个危险系统调用
$ docker info | grep seccomp
 Security Options: seccomp
  Profile: default

# 自定义seccomp profile（进一步限制）
$ cat > /etc/docker/seccomp-custom.json <<'EOF'
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_X86", "SCMP_ARCH_AARCH64"],
  "syscalls": [
    {
      "names": [
        "accept", "accept4", "bind", "clone", "close", "connect",
        "epoll_create", "epoll_create1", "epoll_ctl", "epoll_pwait", "epoll_wait",
        "exit", "exit_group", "fcntl", "fstat", "futex", "getpid", "getrandom",
        "ioctl", "listen", "mmap", "mprotect", "munmap", "nanosleep",
        "openat", "poll", "read", "recvfrom", "recvmsg", "rt_sigaction",
        "rt_sigprocmask", "rt_sigreturn", "sendmsg", "sendto", "setitimer",
        "setsockopt", "sigaltstack", "socket", "socketpair", "write", "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
EOF
$ docker run --rm --security-opt seccomp=/etc/docker/seccomp-custom.json myapp

# 4. AppArmor/SELinux在容器中的应用
# Docker默认使用container-selinux（RHEL）或默认AppArmor profile（Ubuntu）
$ docker run --rm --security-opt apparmor=docker-default alpine
# 查看默认profile：
$ cat /etc/apparmor.d/docker
# 自定义：
$ docker run --rm --security-opt apparmor=myapp-profile myapp
```
## 11.6.3 容器运行时监控
```bash
# 1. 运行时行为基线
$ docker run -d --name web --security-opt seccomp=unconfined nginx
$ docker exec web ps aux              # 基线：应该只有nginx进程
$ docker exec web ls /proc            # 基线：标准procfs
# 异常：发现unexpected进程或/proc被篡改 → 可能容器逃逸

# 2. 资源限制（防DoS）
$ docker run -d \
  --memory="512m" \
  --memory-swap="512m" \              # 禁用swap（防swap耗尽）
  --cpus="1.5" \
  --pids-limit=100 \                  # 限制进程数（防fork炸弹）
  --read-only \                       # 根文件系统只读
  --tmpfs /tmp:noexec,nosuid,size=100m \  # tmpfs限制
  --tmpfs /var/cache:size=10m \
  nginx

# 3. 只读根文件系统 + 显式可写卷
$ docker run -d \
  --read-only \
  -v nginx-cache:/var/cache/nginx:rw \
  -v nginx-logs:/var/log/nginx:rw \
  -v nginx-conf:/etc/nginx:ro \
  nginx
# 即使容器被攻破，攻击者也无法修改/usr/sbin/nginx等系统文件

# 4. 无特权模式（防容器逃逸）
$ docker run -d --security-opt no-new-privileges:true myapp
# 禁止容器内进程通过setuid/setcap提升权限
# 即使容器内有setuid二进制，也无法提权
```
## 11.7 漏洞管理与供应链安全

"没打补丁"是安全事件的头号根因。漏洞管理 = 扫描 + 评估 + 修补 + 验证的闭环。

## 11.7.1 漏洞扫描与CVSS评估
```bash
# 1. 系统级漏洞扫描
$ sudo apt install lynis              # 自动化安全审计
$ sudo lynis audit system
# 或使用OpenSCAP：
$ sudo apt install libopenscap8
$ sudo oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_cis_level2_server --results-arf /tmp/scan.xml /usr/share/xml/scap/ssg/content/ssg-ubuntu2404-ds.xml

# 2. 查看已安装软件版本
$ dpkg -l | grep -E 'openssh|openssl|nginx|apache|mysql|postgresql'
$ rpm -qa | grep -E 'openssh|openssl|nginx|httpd|mariadb|postgresql'

# 3. 快速CVE检查脚本
$ cat > /usr/local/bin/cve-check.sh <<'EOF'
#!/bin/bash
set -euo pipefail

echo "=== 关键软件版本与CVE检查 $(date) ==="
echo ""

check_pkg() {
    local pkg=$1
    if command -v dpkg >/dev/null 2>&1; then
        dpkg -l "$pkg" 2>/dev/null | awk 'NR==6{print "  Debian/Ubuntu: " $2 " " $3}' || echo "  Not installed"
    elif command -v rpm >/dev/null 2>&1; then
        rpm -q "$pkg" 2>/dev/null | awk '{print "  RHEL/CentOS: " $1}' || echo "  Not installed"
    fi
}

echo "1. SSH:"
check_pkg openssh-server
check_pkg openssh-clients
ssh -V 2>&1 | head -1

echo ""
echo "2. SSL/TLS:"
check_pkg openssl
openssl version

echo ""
echo "3. Web服务器:"
check_pkg nginx
check_pkg apache2
check_pkg httpd

echo ""
echo "4. 数据库:"
check_pkg mysql-server
check_pkg mariadb-server
check_pkg postgresql

echo ""
echo "5. 容器运行时:"
check_pkg docker-ce
check_pkg containerd
docker --version 2>/dev/null || true

echo ""
echo "建议：访问以下地址交叉验证已知漏洞："
echo "  - https://nvd.nist.gov (NVD数据库)"
echo "  - https://security-tracker.debian.org (Debian安全追踪)"
echo "  - https://access.redhat.com/security/cve (Red Hat CVE数据库)"
EOF
chmod +x /usr/local/bin/cve-check.sh
```
## 11.7.2 自动安全更新策略
```bash
# === Debian/Ubuntu：unattended-upgrades ===
$ sudo apt install unattended-upgrades
$ sudo dpkg-reconfigure unattended-upgrades  # 启用

# 精细配置：/etc/apt/apt.conf.d/50unattended-upgrades
$ cat <<'EOF' | sudo tee /etc/apt/apt.conf.d/50unattended-upgrades
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";                    # 安全更新
    "${distro_id}ESMApps:${distro_codename}-apps-security";        # ESM应用
    "${distro_id}ESM:${distro_codename}-infra-security";           # ESM基础设施
};
Unattended-Upgrade::Package-Blacklist {
    // "nginx";                                                    # 如需排除特定包
    // "postgresql-*";
};
Unattended-Upgrade::AutoFixInterruptedDpkg "true";                 # 修复中断的dpkg
Unattended-Upgrade::MinimalSteps "true";                           # 最小化步骤（降低风险）
Unattended-Upgrade::InstallOnShutdown "false";                     # 不在关机时安装
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "false";                      # 生产环境手动重启
Unattended-Upgrade::Automatic-Reboot-Time "03:00";
Unattended-Upgrade::SyslogEnable "true";                           # 记录到syslog
Unattended-Upgrade::Verbose "0";
EOF

# 测试运行（不实际安装）
$ sudo unattended-upgrade --dry-run --debug

# 查看日志
$ grep unattended-upgrade /var/log/syslog | tail -20

# === RHEL/CentOS：dnf-automatic ===
$ sudo dnf install dnf-automatic
$ sudo systemctl enable --now dnf-automatic.timer

# 配置：/etc/dnf/automatic.conf
$ cat <<'EOF' | sudo tee /etc/dnf/automatic.conf
[commands]
upgrade_type = security
random_sleep = 3600
network_online_timeout = 60
download_updates = yes
apply_updates = yes

[emitters]
emit_via = motd
system_name = myserver

[email]
email_from = root@example.com
email_to = admin@example.com
email_host = localhost
EOF
```
## 11.7.3 内核热修补（无需重启）
```bash
# === Ubuntu Pro：Livepatch ===
$ sudo pro enable livepatch
$ canonical-livepatch status
last check: 2026-07-03 09:00:00 UTC
patch state: ✓ all applicable livepatch modules inserted
# 内核漏洞已自动修补，无需重启

# === RHEL：kpatch ===
$ sudo dnf install kpatch-dnf
$ sudo dnf kpatch auto                # 自动应用热补丁
$ sudo kpatch list
kpatch_5_14_0_362_el9_1 [enabled]
kpatch_5_14_0_362_el9_2 [enabled]

# === 通用：检查重启需求 ===
# Debian
$ [ -f /var/run/reboot-required ] && echo "REBOOT REQUIRED" || echo "No reboot needed"
$ cat /var/run/reboot-required.pkgs   # 哪些包需要重启

# RHEL
$ sudo needs-restarting -r
Core libraries or services have been updated:
  systemd -> 252-18.el9
  glibc -> 2.34-60.el9
Reboot is required to ensure the system benefits from these updates.
```
## 11.8 实战：生产环境安全检查清单
```bash
#!/bin/bash
# /usr/local/bin/sec-audit.sh
# 生产环境安全检查清单（建议每日/每周运行）

set -euo pipefail

REPORT_FILE="/var/log/security-audit-$(date +%Y%m%d).log"
exec > >(tee -a "$REPORT_FILE") 2>&1

echo "=========================================="
echo "  安全审计报告 $(date)"
echo "  主机: $(hostname)"
echo "  内核: $(uname -r)"
echo "=========================================="

echo ""
echo "【1】可登录用户审计"
echo "----------------------------------------"
echo "可登录用户（有shell）："
awk -F: '$7 !~ /nologin|false/ && $3>=1000 {print "  " $1 ": " $7}' /etc/passwd
echo ""
echo "系统用户（UID<1000但有shell，异常！）："
awk -F: '$7 !~ /nologin|false/ && $3<1000 && $3!=0 {print "  ⚠ " $1 ": " $7}' /etc/passwd

echo ""
echo "【2】密码安全审计"
echo "----------------------------------------"
echo "空密码或锁定用户："
awk -F: '($2=="" || $2=="!" || $2=="*") && $3>=1000 {print "  ⚠ " $1 ": " ($2=="" ? "空密码" : "锁定/无密码")}' /etc/shadow
echo ""
echo "密码永不过期（风险）："
chage -l root 2>/dev/null | grep -i "never" && echo "  ⚠ root密码永不过期"
for user in $(awk -F: '$3>=1000{print $1}' /etc/passwd); do
    chage -l "$user" 2>/dev/null | grep -q "password must be changed" && echo "  ⚠ $user 密码过期策略异常"
done

echo ""
echo "【3】sudo权限审计"
echo "----------------------------------------"
echo "免密sudo用户（高风险）："
grep -r "NOPASSWD" /etc/sudoers /etc/sudoers.d/ 2>/dev/null | grep -v "^#" | while read line; do
    echo "  ⚠ $line"
done
echo ""
echo "sudo ALL权限用户："
grep -r "ALL=.*ALL" /etc/sudoers /etc/sudoers.d/ 2>/dev/null | grep -v "^#" | while read line; do
    echo "  ⚠ $line"
done

echo ""
echo "【4】SSH配置审计"
echo "----------------------------------------"
if [ -f /etc/ssh/sshd_config ]; then
    echo "PermitRootLogin: $(grep -E "^PermitRootLogin" /etc/ssh/sshd_config | awk '{print $2}' || echo "未设置（默认yes，风险）")"
    echo "PasswordAuthentication: $(grep -E "^PasswordAuthentication" /etc/ssh/sshd_config | awk '{print $2}' || echo "未设置")"
    echo "Port: $(grep -E "^Port" /etc/ssh/sshd_config | awk '{print $2}' || echo "22")"
    
    # 检查弱算法
    if grep -q "Ciphers.*3des\|Ciphers.*aes128-cbc\|Ciphers.*blowfish" /etc/ssh/sshd_config 2>/dev/null; then
        echo "  ⚠ 发现弱加密算法"
    fi
else
    echo "  ⚠ /etc/ssh/sshd_config 不存在"
fi

echo ""
echo "【5】SUID/SGID审计"
echo "----------------------------------------"
echo "SUID文件（非标准）："
find / -perm -4000 -type f 2>/dev/null | grep -v -E '^/(usr/sbin|usr/bin|bin|sbin)/' | while read f; do
    echo "  ⚠ 非标准路径SUID: $f"
done
echo ""
echo "SGID文件（非标准）："
find / -perm -2000 -type f 2>/dev/null | grep -v -E '^/(usr/sbin|usr/bin|bin|sbin)/' | while read f; do
    echo "  ⚠ 非标准路径SGID: $f"
done

echo ""
echo "【6】文件权限审计"
echo "----------------------------------------"
echo "全局可写文件（非/tmp）："
find / -perm -2 ! -type l 2>/dev/null | grep -v -E '^/(proc|sys|dev|tmp|run|var/tmp)/' | head -20 | while read f; do
    echo "  ⚠ $f"
done

echo ""
echo "【7】网络暴露审计"
echo "----------------------------------------"
echo "监听端口（非本地）："
ss -tlnp | awk 'NR>1 && $4 !~ /127\.0\.0\.1|::1|\*/ {print "  " $4 " (" $6 ")"}' | sort -u

echo ""
echo "【8】失败登录审计"
echo "----------------------------------------"
if command -v lastb >/dev/null 2>&1; then
    echo "最近失败登录："
    lastb | head -10 | while read line; do
        echo "  $line"
    done
else
    echo "  lastb不可用，检查/var/log/auth.log或/var/log/secure"
fi

echo ""
echo "【9】定时任务审计"
echo "----------------------------------------"
echo "系统cron任务："
find /etc/cron* -type f 2>/dev/null | while read f; do
    echo "  $f"
done
echo ""
echo "用户cron任务："
for f in /var/spool/cron/*; do
    [ -f "$f" ] && echo "  $f"
done

echo ""
echo "【10】内核参数安全审计"
echo "----------------------------------------"
echo "内核指针限制: $(sysctl -n kernel.kptr_restrict 2>/dev/null || echo "未设置") (推荐2)"
echo "dmesg限制: $(sysctl -n kernel.dmesg_restrict 2>/dev/null || echo "未设置") (推荐1)"
echo "ptrace范围: $(sysctl -n kernel.yama.ptrace_scope 2>/dev/null || echo "未设置") (推荐1或2)"
echo "kexec限制: $(sysctl -n kernel.kexec_load_disabled 2>/dev/null || echo "未设置") (推荐1)"

echo ""
echo "=========================================="
echo "  审计完成。发现 ⚠ 标记的问题需立即调查。"
echo "  完整报告: $REPORT_FILE"
echo "=========================================="
本章小结
安全纵深防御决策树
plain
```
需要远程访问？
├─ SSH加固：密钥-only + 改端口 + fail2ban + 审计日志
├─ 跳板机/VPN：敏感服务器不直接暴露公网
└─ 多因素认证：高安全环境启用TOTP

需要权限控制？
├─ 普通用户 + sudo精细化授权
├─ SELinux/AppArmor：强制访问控制
├─ Capabilities：替代setuid的最小权限
└─ Seccomp：系统调用过滤

需要审计追踪？
├─ auditd：内核级操作审计
├─ journald：结构化日志
└─ 集中日志：ELK/Loki/SIEM

需要容器安全？
├─ 最小镜像：distroless/scratch
├─ 能力最小化：--cap-drop=ALL
├─ 只读根文件系统：--read-only
├─ 无新特权：--security-opt no-new-privileges:true
└─ 镜像签名：Docker Content Trust

需要漏洞管理？
├─ 自动安全更新：unattended-upgrades / dnf-automatic
├─ 漏洞扫描：Trivy/Lynis/OpenSCAP
├─ 内核热修补：Livepatch / kpatch
└─ 定期安全审计：自动化检查清单
### 关键配置文件速查

| 文件 | 用途 | 关键检查项 |
|------|------|-----------|
| /etc/ssh/sshd_config | SSH服务配置 | PermitRootLogin, PasswordAuthentication, AllowUsers |
| /etc/pam.d/* | 认证策略 | faillock, pwquality, limits |
| /etc/audit/rules.d/* | 审计规则 | 关键文件监控、特权程序监控 |
| /etc/selinux/config | SELinux模式 | enforcing/permissive/disabled |
| /etc/apparmor.d/* | AppArmor配置 | 程序路径权限 |
| /etc/apt/apt.conf.d/50unattended-upgrades | 自动更新 | Allowed-Origins, Package-Blacklist |

### 应急检查命令
```bash
# 发现入侵迹象时立即执行：
$ w                                    # 当前登录用户
$ last                                 # 最近登录记录
$ ps auxf                              # 进程树（找异常）
$ netstat -tulnp 2>/dev/null || ss -tulnp  # 网络连接
$ lsof | grep deleted                  # 已删除但仍打开的文件（木马常用）
$ find /tmp /var/tmp -type f -newer /etc/passwd 2>/dev/null  # 新出现的临时文件
$ dmesg | tail -50                     # 内核异常
$ cat /var/log/auth.log 2>/dev/null | grep -i "failed\|invalid" | tail -20  # 失败认证
> 本章扩充后字数：约 18,500 字
> 涉及命令：sshd, ssh-keygen, fail2ban, pam_tally2/faillock, auditctl, ausearch, aureport, semanage, setsebool, aa-enforce, aa-complain, docker run --security-opt, trivy, lynis, oscap, unattended-upgrade, kpatch, canonical-livepatch
> 基准版本：OpenSSH 9.7, auditd 3.1, SELinux policy 38, AppArmor 3.1, Docker 25.0

---

## 课后练习

1. 概念题：SSH 配置中 PermitRootLogin no 和 PasswordAuthentication no 分别防御什么攻击？

思路：PermitRootLogin no 防止直接暴力破解 root 密码（攻击者不知道用户名，只能盲猜）；PasswordAuthentication no 强制使用密钥，彻底防御密码爆破。

2. 排障题：修改 /etc/ssh/sshd_config 后重启 SSH 失败，如何快速定位语法错误？

思路：sudo sshd -t（测试模式），它会指出错误行号和具体原因。修复后再 systemctl restart sshd。

3. 实操题：使用 auditd 添加一条规则，监控 /etc/sudoers 文件的写入（w）和属性修改（a）操作，并打上标签 sudoers_change。

思路：sudo auditctl -w /etc/sudoers -p wa -k sudoers_change。永久生效需写入 /etc/audit/rules.d/ 下的 .rules 文件。

4. 安全题：SELinux 处于 Enforcing 模式，Nginx 无法写入 /var/www/uploads，ausearch -m avc 显示 denied { write }。请给出两种解决方法。

思路：1. 改标签：chcon -t httpd_sys_rw_content_t /var/www/uploads -R；2. 改布尔值：setsebool -P httpd_unified on（或特定布尔值）。最佳实践是改标签并 restorecon。

5. 对比题：SELinux 和 AppArmor 在策略定义方式上的根本区别是什么？

思路：SELinux 是类型强制（TE）基于 inode 标签，任何文件都有安全上下文；AppArmor 是路径强制，基于程序配置文件允许访问的路径。AppArmor 更简单，SELinux 更精细。

6. 实操题：写一条 Docker 运行命令，要求容器：只读根文件系统、丢弃所有 Capabilities、仅添加 NET_BIND_SERVICE、禁止提权。

思路：docker run --read-only --cap-drop=ALL --cap-add=NET_BIND_SERVICE --security-opt no-new-privileges:true myapp。

7. 排障题：fail2ban 没有封禁任何 IP，但 journalctl -u fail2ban 显示 WARNING 'sshd' not found in 'systemd-journal'。问题出在哪？

思路：fail2ban 的 backend 配置不对。默认 backend=auto 可能选了 pyinotify。应在 /etc/fail2ban/jail.local 中显式设置 backend = systemd 以读取 journald。

8. 综合题：容器中运行 ping 8.8.8.8 报错 Operation not permitted，但宿主机可以。如何在不给容器 --privileged 的情况下修复？

思路：ping 需要 CAP_NET_RAW。启动时添加 --cap-add=NET_RAW 即可。更优雅的做法是不用 ping，用 curl 或 nc 测试连通性。

---
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

## 12.5 存储驱动：overlay2详解

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

## 12.6 镜像优化与多阶段构建

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

## 12.7 网络模型（CNI基础）

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

## 12.8 实战：生产容器化清单

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

---

## 课后练习

1. 概念题：容器的隔离依赖 Linux 内核的哪两大特性？分别负责什么？

思路：命名空间 (Namespace) 负责“隔离”（看不同的 PID、网络、挂载点等）；Cgroups 负责“限制”（控制 CPU、内存、磁盘 IO 配额）。

2. 排障题：Docker 容器内 top 看到 CPU 使用率，但宿主机 top 看到容器进程占满 2 个核。为什么容器内显示的数字可能和宿主机不一致？

思路：Docker 默认容器内的 top 读取的是容器的 Cgroup 限制（如果没限制则看主机全部核心），而宿主机 top 是物理真实消耗。如果容器内存限制小于物理内存，free -m 显示也可能不一致。

3. 实操题：写出 Dockerfile，使用多阶段构建，将一个 Go 程序编译成静态二进制，并最终放入 scratch 空镜像中运行。

思路：

```dockerfile
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server .
FROM scratch
COPY --from=builder /app/server /server
EXPOSE 8080
CMD ["/server"]
```

4. 对比题：Docker 的 bridge 网络模式和 host 网络模式，在性能和端口管理上有什么优缺点？

思路：bridge 有 NAT 和端口映射开销（docker-proxy），但支持端口复用（宿主机 8080 映射容器 80）。host 无 NAT，性能最高，但容器直接占用宿主机端口，易冲突。

5. 存储题：docker volume create 创建的数据卷和 bind mount（-v /host/path:/container/path）在管理方式上有何不同？

思路：Volume 由 Docker 管理（存储在 /var/lib/docker/volumes/），可通过 docker volume 命令生命周期管理，适合生产数据；Bind mount 直接映射宿主机目录，依赖宿主机文件系统布局，适合开发调试。

6. 实战题：启动一个 Podman 容器，要求以非 root 用户（nobody）运行，并挂载宿主机的 /data 为只读。

思路：podman run --user 65534 -v /data:/data:ro --rm alpine ls /data。注意 rootless 模式下挂载宿主机目录可能需要额外配置用户命名空间映射。

7. 网络题：Docker 容器访问宿主机上的 MySQL（宿主机 IP 是 192.168.1.10，端口 3306），容器内应连接什么地址？

思路：对于 Docker，Linux 容器可连接 172.17.0.1（docker0 网桥的网关）或宿主机实际 IP。Mac/Windows 下需连接 host.docker.internal。最佳实践是不直接用 IP，用容器名称或服务发现。

8. 综合题：如何查看一个运行中容器的 overlay2 存储层在宿主机上的具体路径？

思路：docker inspect <container_id> | jq '.[0].GraphDriver.Data.UpperDir'。这将显示该容器可写层的绝对路径。

---
# 第二部分：第十三章 —— Linux 内核 6.x+ 前沿特性

> **本章定位**：如果说前十二章是 Linux 的“躯干与四肢”，那么本章这些特性就是内核的“超强外挂”。从异步 I/O 到可编程调度，从智能拥塞控制到硬件级安全，再到实时能力和内存安全革命——本章覆盖 2024-2026 年 Linux 内核最激动人心的进化，是面向 AI 基础设施、自动驾驶和工业 4.0 的底层基石。

---

## 13.1 io_uring：异步 I/O 的革命

**一句话定义**：io_uring 是 Linux 5.1 引入的高性能异步 I/O 接口，通过在用户态与内核态之间建立共享的环形队列（提交队列 SQ 与完成队列 CQ），实现批量提交与收割 I/O 事件，彻底告别“干等数据”的尴尬。

### 为什么需要 io_uring？

```bash
# 传统同步 I/O：CPU 阻塞等待磁盘
$ read file.txt           # 进程陷入睡眠，直到数据从磁盘返回

# 传统异步 I/O（libaio）：每次提交都需要系统调用
$ sudo strace -e io_submit,io_getevents -p 1234
io_submit(0x7f..., 1, ...) = 1        # 系统调用 1
io_getevents(0x7f..., 1, ...) = 1      # 系统调用 2
# 每次 IO 都要 2 次系统调用，高并发下系统调用开销惊人
```

**io_uring 的解决方案**：
- 用户态和内核态共享两个环形队列（SQ 和 CQ）
- 批量提交：一次系统调用提交 N 个请求
- 批量收割：一次系统调用收割 N 个完成事件
- 支持 `IORING_SETUP_SQPOLL`：内核线程主动轮询，用户态完全零系统调用

### 实战：检查 io_uring 支持

```bash
# 1. 检查内核是否编译了 io_uring
$ grep CONFIG_IO_URING /boot/config-$(uname -r)
CONFIG_IO_URING=y

# 2. 查看当前系统 io_uring 版本
$ cat /proc/self/io_uring
# 如果文件存在，说明当前内核支持

# 3. 查看进程是否使用 io_uring（通过 eBPF 追踪）
$ sudo bpftrace -e 'tracepoint:io_uring:io_uring_submit_sqe { @apps[comm] = count(); }'
Attaching 1 probe...
^C

@apps[nginx]: 12345
@apps[redis-server]: 8765
# 显示哪些应用在使用 io_uring
```

### 实战：使用 fio 测试 io_uring 性能

```bash
# 1. 安装 fio
$ sudo apt install fio

# 2. 使用 libaio 测试随机读（传统方式）
$ sudo fio --name=randread-aio \
    --ioengine=libaio \
    --rw=randread \
    --bs=4k \
    --direct=1 \
    --size=1G \
    --numjobs=16 \
    --runtime=60 \
    --group_reporting \
    --filename=/data/testfile

# 3. 使用 io_uring 测试随机读（现代方式）
$ sudo fio --name=randread-uring \
    --ioengine=io_uring \
    --rw=randread \
    --bs=4k \
    --direct=1 \
    --size=1G \
    --numjobs=16 \
    --runtime=60 \
    --group_reporting \
    --filename=/data/testfile

# 4. 对比结果
# 通常 io_uring 的 IOPS 比 libaio 高 30%-100%
# 延迟（clat）降低 20%-50%
```

### 实战：使用 io_uring 的应用

```bash
# 1. 检查 MySQL 是否使用 io_uring（8.0.26+ 默认启用）
$ mysql -e "SHOW VARIABLES LIKE 'innodb_use_io_uring';"
+---------------------+-------+
| Variable_name       | Value |
+---------------------+-------+
| innodb_use_io_uring | ON    |
+---------------------+-------+

# 2. Redis 7.0+ 使用 io_uring 替代 epoll
$ redis-server --io-threads 4

# 3. QEMU/KVM 使用 io_uring 加速虚拟磁盘
$ qemu-system-x86_64 -drive file=/path/disk.qcow2,if=virtio,aio=io_uring
```

### 避坑指南

```bash
# 坑1：io_uring 需要较新的内核（5.1+，推荐 6.x）
$ uname -r
4.15.0-xxx                 # 太旧，不支持 io_uring

# 坑2：某些发行版默认未启用 io_uring 支持
$ grep CONFIG_IO_URING /boot/config-$(uname -r)
# CONFIG_IO_URING is not set     # 需要重新编译内核

# 坑3：容器环境中可能需要额外权限
$ docker run --rm --cap-add=SYS_ADMIN ubuntu grep CONFIG_IO_URING /boot/config-$(uname -r)
# SYS_ADMIN 能力可能被限制，导致无法访问
```

---

## 13.2 sched_ext：可热替换的 CPU 调度器

**一句话定义**：`sched_ext`（调度器扩展）是 Linux 6.6 引入、6.12 正式稳定的框架，允许开发者使用 eBPF 编写自定义 CPU 调度策略，并支持**无需重启内核**的热加载与热卸载。

### 为什么需要 sched_ext？

```bash
# 传统调度器（CFS / EEVDF）是“写死”在内核里的
# 想改调度策略 → 修改内核 C 代码 → 重新编译内核 → 重启服务器
# 生产环境无法接受

# sched_ext 的解决方式：调度策略变成 eBPF 程序
$ bpftool sched_ext load my_sched.bpf.o
# 一行命令，新调度策略立即生效，无需重启
```

### 实战：检查 sched_ext 支持

```bash
# 1. 内核版本检查（6.12+ 才有稳定支持）
$ uname -r
6.12.0-rc1+                # 6.12 及以上

# 2. 检查内核配置
$ grep CONFIG_SCHED_EXT /boot/config-$(uname -r)
CONFIG_SCHED_EXT=y

# 3. 检查当前调度器
$ cat /sys/kernel/sched_ext/root/ops
# 输出当前活跃的调度器名称（无输出 = 使用默认调度器）
```

### 实战：使用 scx（sched_ext 工具集）

```bash
# 1. 安装 sched_ext 工具
$ git clone https://github.com/sched-ext/scx.git
$ cd scx
$ make

# 2. 查看可用的调度器示例
$ ls *.bpf.o
scx_simple.bpf.o    # 简单调度器
scx_rusty.bpf.o     # Rust 实现的调度器
scx_layered.bpf.o   # 分层调度器
scx_central.bpf.o   # 集中式调度器

# 3. 加载一个调度器（以 scx_simple 为例）
$ sudo bpftool sched_ext load scx_simple.bpf.o
# 调度策略立即生效！

# 4. 验证调度器已加载
$ cat /sys/kernel/sched_ext/root/ops
scx_simple

# 5. 卸载调度器（恢复默认 CFS/EEVDF）
$ sudo bpftool sched_ext unload
# 瞬间恢复默认调度策略
```

### 实战：自定义调度策略（示例）

```bash
# 1. 编写一个简单的 eBPF 调度器（C 语言）
$ cat > my_sched.bpf.c <<'EOF'
// SPDX-License-Identifier: GPL-2.0
#include "scx_common.bpf.h"

// 调度器名称
char _license[] SEC("license") = "GPL";

// 选择 CPU 的逻辑：把任务放到负载最低的 CPU
s32 BPF_STRUCT_OPS(my_select_cpu, struct task_struct *p, s32 prev_cpu, u64 wake_flags)
{
    s32 cpu;
    u64 min_load = -1;
    s32 selected = prev_cpu;
    
    bpf_for(cpu, 0, 8) {  // 假设 8 核
        u64 load = cpu_load(cpu);
        if (load < min_load) {
            min_load = load;
            selected = cpu;
        }
    }
    return selected;
}

// 注册调度器
struct sched_ext_ops my_ops = {
    .select_cpu = my_select_cpu,
    .name       = "my_scheduler",
};
EOF

# 2. 编译为 BPF 对象文件
$ clang -target bpf -g -O2 -c my_sched.bpf.c -o my_sched.bpf.o

# 3. 加载
$ sudo bpftool sched_ext load my_sched.bpf.o

# 4. 查看效果
$ cat /sys/kernel/sched_ext/root/ops
my_scheduler
```

### 实战场景：为 Web 服务定制调度

```bash
# 场景：Nginx 需要低延迟，编译任务可以低优先级
# 使用 scx_layered 实现分层调度

$ cat > layered_config.yaml <<'EOF'
layers:
  - name: nginx
    match:
      comm: nginx
    sched:
      nice: -10
      cpu_range: 0-3        # 绑定到前 4 个核
  
  - name: compile
    match:
      comm: gcc
    sched:
      nice: 19
      cpu_range: 4-7        # 绑定到后 4 个核
EOF

$ sudo ./scx_layered -c layered_config.yaml
# Nginx 获得低延迟 + 专用核心，编译任务跑在后端核心
```

### 避坑指南

```bash
# 坑1：调度器写崩了怎么办？
$ sudo bpftool sched_ext unload        # 立即恢复默认调度器
# 如果无法执行命令，重启系统也会恢复默认调度器

# 坑2：sched_ext 需要 CAP_BPF 和 CAP_SYS_ADMIN
$ sudo bpftool sched_ext load my.bpf.o
Error: operation not permitted
# 解决：用 root 执行，或授予相应能力

# 坑3：部分云环境不允许 eBPF 操作
# 检查：cat /proc/sys/kernel/unprivileged_bpf_disabled
# 如果为 2（永久禁用），无法使用 sched_ext
```

---

## 13.3 BBR v3：智能网络拥塞控制

**一句话定义**：BBR（Bottleneck Bandwidth and RTT）是 Google 开发的 TCP 拥塞控制算法，BBR v3 是其第三代演进，通过主动探测网络瓶颈带宽和最小传播延迟来调整发送速率，在跨数据中心场景下吞吐量提升 15%-20%。

### BBR vs 传统拥塞控制

```bash
# 传统 CUBIC 算法：基于丢包判断拥塞
# 相当于“看到前车刹车灯才踩刹车”——反应滞后

# BBR 算法：主动探测带宽和延迟
# 相当于“AI 雷达实时探测路况”——始终保持最优速度
```

### 实战：启用 BBR

```bash
# 1. 检查当前拥塞控制算法
$ sysctl net.ipv4.tcp_congestion_control
net.ipv4.tcp_congestion_control = cubic

# 2. 检查可用的算法
$ sysctl net.ipv4.tcp_available_congestion_control
net.ipv4.tcp_available_congestion_control = reno cubic bbr

# 3. 启用 BBR（需要 Linux 4.9+）
$ sudo modprobe tcp_bbr
$ sudo sysctl -w net.core.default_qdisc=fq
$ sudo sysctl -w net.ipv4.tcp_congestion_control=bbr

# 4. 验证
$ sysctl net.ipv4.tcp_congestion_control
net.ipv4.tcp_congestion_control = bbr
```

### 实战：检查 BBR 是否生效

```bash
# 1. 查看已建立的连接使用的算法
$ ss -ti | grep -E 'bbr|cubic'
tcp   ESTAB  0  0  192.168.1.100:22  10.0.0.5:54321
     cubic wscale:7,7 rto:204 rtt:0.5/0.2 ...

# 2. BBR 特有的输出字段
$ ss -ti | grep bbr
tcp   ESTAB  0  0  192.168.1.100:443  203.0.113.1:12345
     bbr wscale:7,7 rto:204 rtt:1.2/0.3 pacing_rate 100M bbr:0.5 ...
# bbr:0.5 表示 bw_hi（带宽高估因子）

# 3. 查看 BBR 统计信息
$ nstat -z | grep BBR
TcpExtTCPBBRFlow                   12345         0.0
```

### 实战：BBR v3 性能测试

```bash
# 1. iperf3 测试（服务端）
$ iperf3 -s

# 2. 客户端测试（CUBIC 基线）
$ sysctl -w net.ipv4.tcp_congestion_control=cubic
$ iperf3 -c 10.0.0.1 -t 60 -P 8
[SUM]   0.00-60.00  sec  45.2 GBytes  6.47 Gbits/sec

# 3. 切换到 BBR
$ sysctl -w net.ipv4.tcp_congestion_control=bbr
$ iperf3 -c 10.0.0.1 -t 60 -P 8
[SUM]   0.00-60.00  sec  53.8 GBytes  7.70 Gbits/sec
# 吞吐量提升约 19%
```

### 永久启用 BBR

```bash
# /etc/sysctl.d/99-bbr.conf
$ cat <<'EOF' | sudo tee /etc/sysctl.d/99-bbr.conf
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
EOF

$ sudo sysctl --system
```

### BBR v3 的改进（2024-2026）

| 改进点 | BBR v2 | BBR v3 |
|--------|--------|--------|
| 多流公平性 | 一般 | 显著改善 |
| 延迟探测 | 周期性 | 更精准的 ACK 驱动探测 |
| 带宽高估 | 可能导致队列堆积 | 更精确的带宽估计 |
| 高丢包场景 | 性能下降 | 更好地适应丢包 |

### 避坑指南

```bash
# 坑1：BBR 不适合所有场景
# - 数据中心内部（低延迟、高带宽）→ BBR 效果一般
# - 跨洲/跨数据中心 → BBR 效果显著
# - WiFi/蜂窝网络 → 谨慎启用

# 坑2：BBR 需要 fq（公平队列）qdisc
$ sysctl net.core.default_qdisc
net.core.default_qdisc = fq        # 必须是 fq，否则 BBR 效果打折

# 坑3：云环境可能默认不支持 BBR
$ sysctl net.ipv4.tcp_available_congestion_control
# 如果没有 bbr，说明内核未编译 tcp_bbr
```

---

## 13.4 影子栈（Shadow Stack）：硬件级 ROP 防御

**一句话定义**：影子栈是基于 Intel CET（Control-flow Enforcement Technology）的硬件安全机制，在 CPU 内部维护一个不可篡改的“影子栈”用于存储函数返回地址，与普通栈实时比对，从硅片层面堵死 ROP（面向返回编程）攻击路径。

### 什么是 ROP 攻击？

```
正常程序执行：
函数 A 调用 函数 B → 返回地址存入栈 → 函数 B 执行 → 读取返回地址跳回 A

ROP 攻击（缓冲区溢出）：
攻击者溢出缓冲区 → 篡改栈中的返回地址 → 程序跳转到恶意代码
```

### 影子栈的工作原理

```
普通栈（可被篡改）:    影子栈（硬件保护，不可篡改）:
+------------------+    +------------------+
| 返回地址 A       |    | 返回地址 A（副本）|
| 返回地址 B       |    | 返回地址 B（副本）|
| ...              |    | ...              |
+------------------+    +------------------+
         ↓                        ↓
    被篡改后                 CPU 硬件比对
    返回地址 X      ────→   不一致 → #CP 异常 → 进程终止
```

### 实战：检查影子栈支持

```bash
# 1. 检查 CPU 是否支持 CET
$ grep -E "cet|shadow" /proc/cpuinfo
flags           : ... cet_shadow cet_ibt ...
# 如果有 cet_shadow，说明 CPU 支持影子栈

# 2. 检查内核是否启用 CET 支持
$ grep CONFIG_X86_CET /boot/config-$(uname -r)
CONFIG_X86_CET=y

# 3. 检查当前进程的影子栈状态
$ cat /proc/self/status | grep -i shadow
ShadowStack:     enabled
```

### 实战：编译支持影子栈的程序

```bash
# 1. 编译时启用 CET 保护
$ gcc -fcf-protection=full -o myapp myapp.c

# 2. 检查二进制文件是否启用 CET
$ readelf -n myapp | grep -i cet
   0x00000000 (NT_X86_CET)         Size: 8 bytes
# 如果看到 NT_X86_CET，说明启用了 CET

# 3. 运行时的影子栈状态
$ ./myapp &
$ cat /proc/$(pidof myapp)/status | grep ShadowStack
ShadowStack:     enabled
```

### 实战：检测 ROP 攻击尝试

```bash
# 影子栈检测到攻击时会触发 #CP（Control Protection）异常
# 查看内核日志
$ dmesg | grep -i "control protection"
[ 1234.567] traps: myapp[1234] control protection fault ip:7f... sp:7f...
# 进程被 SIGSEGV 终止

# 查看系统计数器
$ grep -E "cet|shadow" /proc/interrupts
  CP:     0     0     0     0   Control protection interrupts
# 如果 CP 计数器增长，说明有人尝试 ROP 攻击
```

### 生产环境建议

```bash
# 1. 检查所有关键服务是否启用影子栈
$ for svc in nginx mysql redis; do
    pid=$(pgrep -x $svc | head -1)
    [ -n "$pid" ] && echo "$svc: $(cat /proc/$pid/status | grep ShadowStack)"
  done

# 2. 编译参数建议（Makefile）
CFLAGS += -fcf-protection=full -mcet

# 3. 内核参数（禁用 CET 调试）
$ cat /proc/cmdline
... cet=off            # 调试时禁用
... cet=on             # 生产启用
```

### 避坑指南

```bash
# 坑1：影子栈需要硬件支持（Intel 11代+ / AMD Zen 4+）
# 老 CPU 无法使用

# 坑2：某些发行版默认未启用
$ grep CONFIG_X86_CET /boot/config-$(uname -r)
# CONFIG_X86_CET is not set
# 需要重新编译内核

# 坑3：影子栈可能与某些调试器冲突
# gdb attach 时可能报错，调试时可临时禁用
$ echo 0 > /proc/sys/kernel/shadow_stack_enabled
```

---

## 13.5 PREEMPT_RT：硬实时内核正式合入主线

**一句话定义**：`PREEMPT_RT` 是 Linux 内核的实时抢占补丁集，经过近 20 年的开发，于 Linux 6.12 正式合入主线，使 Linux 成为“硬实时（Hard Real-Time）”操作系统，调度延迟从毫秒级降至微秒级（< 50μs）且绝对确定。

### 什么是硬实时？

```
普通 Linux（尽力而为）：
任务执行 → 被中断打断（网卡收包、磁盘 IO）→ 延迟不可控（毫秒级）

实时 Linux（PREEMPT_RT）：
实时任务 → 获得“免死金牌”→ 中断被延迟处理 → 延迟确定（微秒级）
```

### 实战：检查 PREEMPT_RT 支持

```bash
# 1. 查看当前内核配置
$ uname -r
6.12.0-rt1               # -rt 后缀表示实时内核

# 2. 检查内核是否启用 PREEMPT_RT
$ grep PREEMPT_RT /boot/config-$(uname -r)
CONFIG_PREEMPT_RT=y

# 3. 查看调度延迟（cyclictest 工具）
$ sudo apt install rt-tests
$ sudo cyclictest -m -p 95 -t 1 -l 100000
# 输出显示最大延迟（Max Latency）< 50μs 即为硬实时
```

### 实战：启用 PREEMPT_RT

```bash
# 1. 安装实时内核（Ubuntu）
$ sudo apt install linux-image-rt-amd64 linux-headers-rt-amd64

# 2. 安装实时内核（RHEL）
$ sudo dnf install kernel-rt

# 3. 重启后选择实时内核
$ sudo reboot
$ uname -r | grep rt

# 4. 验证
$ cat /sys/kernel/realtime
1                    # 1 = 实时内核启用
```

### 实战：编写实时任务

```bash
# 1. 使用 chrt 设置实时优先级
# SCHED_FIFO：实时调度策略，优先级 1-99
$ sudo chrt -f 99 /usr/local/bin/critical_task

# 2. 查看任务的调度策略
$ chrt -p $(pidof critical_task)
pid 1234's current scheduling policy: SCHED_FIFO
pid 1234's current scheduling priority: 99

# 3. 使用 systemd 配置实时服务
$ cat /etc/systemd/system/critical.service
[Service]
ExecStart=/usr/local/bin/critical_task
CPUSchedulingPolicy=fifo
CPUSchedulingPriority=99
CPUAffinity=0-3           # 绑定到特定核心

# 4. 系统级实时配置
$ cat /etc/security/limits.d/99-realtime.conf
@realtime   soft    rtprio    99
@realtime   hard    rtprio    99
@realtime   soft    memlock   unlimited
@realtime   hard    memlock   unlimited
```

### 实战：测量实时性能

```bash
# 1. 使用 cyclictest 测量延迟
$ sudo cyclictest -m -p 95 -t 4 -l 1000000 -q
# -m: 锁定内存
# -p 95: 实时优先级 95
# -t 4: 4 个线程
# -l 1000000: 循环 100 万次
# -q: 安静模式，只输出统计结果

# 输出示例：
# T: 0 ( 1234) P:95 I:1000 C:1000000 Min:  2 Act:  4 Avg:  3 Max: 45
# Min=2μs, Max=45μs, Avg=3μs ← 硬实时！

# 2. 压力测试下测量
$ sudo stress-ng --cpu 8 --io 4 --vm 2 &
$ sudo cyclictest -m -p 95 -t 4 -l 1000000 -q
# 即使系统满负荷，Max Latency 仍 < 100μs

# 3. 生成延迟分布图
$ sudo cyclictest -m -p 95 -t 4 -l 1000000 -h 100 | tee latency.log
# 输出延迟直方图
```

### 适用场景

| 场景 | 延迟要求 | 是否需 RT |
|------|----------|-----------|
| Web 服务器 | 毫秒级 | ❌ |
| 数据库 | 毫秒级 | ❌ |
| 工业 PLC | < 1ms | ✅ |
| 自动驾驶 | < 100μs | ✅ |
| 手术机器人 | < 50μs | ✅ |
| 专业音频 | < 100μs | ✅ |
| 5G 基站 | < 100μs | ✅ |

### 避坑指南

```bash
# 坑1：实时内核不是“更快”，而是“更可预测”
# 实时内核的吞吐量可能略低于普通内核

# 坑2：实时优先级设置不当可能让系统“冻住”
# 实时任务占用 100% CPU 时，系统无法响应
# 解决：设置 CPUAffinity 或 CPUQuota

# 坑3：某些驱动不支持中断线程化
$ dmesg | grep "threaded IRQ"        # 检查中断线程化状态
# 部分旧驱动可能需要禁用实时模式
```

---

## 13.6 Rust for Linux：内存安全语言的内核革命

**一句话定义**：Rust for Linux 是 Linux 内核引入 Rust 语言支持的项目，始于 Linux 6.1，旨在利用 Rust 的“所有权与生命周期”机制在编译期消灭内存安全问题，是 Linux 内核 30 年来最大的语言级变革。

### 为什么需要 Rust？

```
Linux 内核 70%+ 的严重 CVE 源于内存安全问题：
- Use-After-Free（释放后使用）
- 缓冲区溢出
- 空指针解引用
- 数据竞争

C 语言的解决方案：添加更多工具（KASAN、SLUB 调试）→ 治标不治本
Rust 的解决方案：编译器物理卡死内存漏洞 → 治本
```

### 实战：检查 Rust 内核支持

```bash
# 1. 检查内核是否编译了 Rust 支持
$ grep CONFIG_RUST /boot/config-$(uname -r)
CONFIG_RUST=y
CONFIG_RUST_IS_AVAILABLE=y

# 2. 查看 Rust 版本要求
$ cat /proc/version
Linux version 6.12.0 (rustc 1.78.0) ...
# 内核构建时使用的 Rust 编译器版本

# 3. 检查已加载的 Rust 内核模块
$ lsmod | grep -E 'rust|binder'
binder_linux           65536  0
# binder_linux 是 Android 的 Binder 驱动，用 Rust 重写
```

### 实战：编译 Rust 内核模块

```bash
# 1. 安装 Rust 工具链
$ rustup install stable
$ rustup component add rust-src

# 2. 下载内核源码
$ git clone https://github.com/torvalds/linux.git
$ cd linux
$ git checkout v6.12

# 3. 配置内核以启用 Rust
$ make menuconfig
# General setup → Rust support → [*] Enable Rust
# 或者直接：
$ scripts/config -e CONFIG_RUST -e CONFIG_RUST_IS_AVAILABLE

# 4. 编译 Rust 模块
$ make LLVM=1

# 5. 查看 Rust 模块
$ find . -name "*.rs" | head -5
rust/kernel/lib.rs
rust/kernel/alloc.rs
drivers/android/binder.rs
```

### 实战：Rust 内核模块示例

```rust
// SPDX-License-Identifier: GPL-2.0
//! 一个简单的 Rust 内核模块

use kernel::prelude::*;

module! {
    type: MyModule,
    name: "my_rust_module",
    author: "Example",
    description: "My first Rust kernel module",
    license: "GPL",
}

struct MyModule;

impl kernel::Module for MyModule {
    fn init(_module: &'static ThisModule) -> Result<Self> {
        pr_info!("Hello from Rust kernel module!\n");
        Ok(MyModule)
    }
}

impl Drop for MyModule {
    fn drop(&mut self) {
        pr_info!("Goodbye from Rust kernel module!\n");
    }
}
```

```bash
# 编译和加载
$ make M=samples/rust
$ sudo insmod samples/rust/my_rust_module.ko
$ dmesg | tail -2
[ 1234.567] Hello from Rust kernel module!
$ sudo rmmod my_rust_module
[ 1234.568] Goodbye from Rust kernel module!
```

### 当前 Rust 内核模块覆盖

| 模块 | 状态 | 说明 |
|------|------|------|
| Binder | ✅ 已合并 | Android IPC 驱动 |
| NVMe 驱动 | 🔄 开发中 | 存储驱动 |
| GPU 驱动 | 🔄 开发中 | 部分显卡驱动 |
| PHY 驱动 | ✅ 已合并 | 网络物理层 |
| 网络过滤 | 🔄 开发中 | eBPF 相关 |
| 文件系统 | 🔄 实验性 | 示例 FS |

### 实战：使用 Rust 内核模块的安全性优势

```bash
# 1. 传统的 C 内核模块（存在漏洞风险）
$ sudo modprobe vulnerable_module
# Use-After-Free 可能导致内核崩溃或提权

# 2. Rust 内核模块（编译器保证安全性）
$ sudo insmod rust_module.ko
# 即使代码有缺陷，Rust 编译器也会阻止内存安全问题

# 3. 查看 Rust 模块的 KASAN 报告（如果有）
$ dmesg | grep -i "rust.*kasan"
# Rust 模块的内存错误报告比 C 模块少 90%+
```

### 避坑指南

```bash
# 坑1：Rust 内核支持需要 LLVM 工具链
$ which clang
/usr/bin/clang                    # 必须安装 clang

# 坑2：Rust 版本必须与内核要求匹配
$ cat Documentation/rust/rust_required_version.txt  # 查看版本要求
# 不匹配可能导致编译失败

# 坑3：部分发行版默认不启用 Rust 内核支持
# 需要重新编译内核或使用提供 Rust 支持的发行版
# Fedora 和 openSUSE 已默认启用
```

---

## 本章小结

### 六个内核特性速查表

| 特性 | 引入版本 | 生产可用 | 核心价值 | 适用场景 |
|------|----------|----------|----------|----------|
| **io_uring** | 5.1 | ✅ 6.x | IOPS 翻倍 | 数据库、KV 存储 |
| **sched_ext** | 6.6 | ✅ 6.12 | 调度策略热替换 | 大厂定制调度 |
| **BBR v3** | 6.6 | ✅ | 跨数据中心吞吐 +20% | 广域网传输 |
| **影子栈** | 6.6 | ✅ | 硬件级 ROP 防御 | 高安全环境 |
| **PREEMPT_RT** | 6.12 | ✅ | 微秒级硬实时 | 工业/汽车/音频 |
| **Rust for Linux** | 6.1 | 🔄 发展中 | 编译期内存安全 | 新驱动开发 |

### 决策树：何时使用这些特性？

```
需要极高 I/O 吞吐？
├─ 是 → io_uring（数据库、存储服务）
└─ 否 → 常规 I/O

需要定制 CPU 调度？
├─ 是 → sched_ext（大厂专属业务）
└─ 否 → 默认 CFS/EEVDF

网络跨地域传输？
├─ 是 → BBR v3（跨洲数据中心）
└─ 否 → CUBIC（局域网）

需要防止 ROP 攻击？
├─ 是 → 影子栈（金融/政府/军工）
└─ 否 → 常规 ASLR

需要硬实时响应？
├─ 是 → PREEMPT_RT（工业/汽车/音频）
└─ 否 → 普通内核

正在编写新内核驱动？
├─ 是 → Rust（内存安全）
└─ 否 → C（已有代码库）
```

### 实践建议

```bash
# 生产环境中逐步启用这些特性的推荐顺序：
# 1. BBR v3（无风险，立即收益）
$ sudo sysctl -w net.ipv4.tcp_congestion_control=bbr

# 2. io_uring（应用升级即可）
# 升级 Redis 7.0+、MySQL 8.0.26+、PostgreSQL 16+

# 3. 影子栈（硬件支持则启用）
$ sudo grep -q cet_shadow /proc/cpuinfo && echo "CET supported"

# 4. sched_ext（测试环境先试）
$ sudo bpftool sched_ext load scx_simple.bpf.o

# 5. PREEMPT_RT（需要专用内核）
# 安装 kernel-rt 包

# 6. Rust 内核模块（新模块开发时采用）
# 编写新驱动时优先考虑 Rust
```

---

> **本章字数**：约 10,000 字
> **涉及命令**：fio, bpftool, sysctl, ss, cyclictest, chrt, rustc, insmod
> **基准版本**：Linux 6.6 LTS / 6.12 LTS / 6.19

---

## 13.7 6.11-6.19：安全、调度与可观测性的新边疆

> 本节覆盖 2024 下半年至 2026 年初 Linux 6.11~6.19 的核心新特性，按安全、调度、存储、网络、Rust、可观测性分类。

### 13.7.1 安全增强

**mseal() 系统调用 (6.10+)**

**一句话定义**：`mseal()` 锁定一块内存区域，使其权限在进程生命周期内不可再被修改——即使攻击者获得了代码执行能力，也无法把只读内存改成可写可执行。

```bash
# 检查内核是否支持
$ grep mseal /proc/kallsyms
ffffffff81234567 T __x64_sys_mseal

# 典型场景：浏览器、密码管理器、加密钱包等敏感应用
# 在初始化时调用 mseal() 锁定代码段和密钥存储区
# 即使攻击者利用漏洞获得了任意代码执行，也无法修改这些区域
```

**Intel LASS 支持 (6.19)**：利用 CPU 硬件隔离线性地址空间——用户态代码无法访问内核地址，即使通过侧信道泄露了内核地址也无法使用。与 SMAP/SMEP 互补，构成"纵深防御"的三层防线。

**专用 Slab 分配器 (6.11)**：通过隔离内存池防御"堆喷射"攻击——每个内核对象类型拥有独立的内存池，不同类型之间无法互相污染。这是对 `CONFIG_SLAB_FREELIST_HARDENED` 的进一步强化。

**ARM 影子栈支持 (6.13)**：将第 13.4 节的 Shadow Stack 防护扩展到 ARM 架构。

### 13.7.2 调度器演进

**EEVDF 调度器完整支持 (6.12)**：CFS（完全公平调度器）的继任者，已于 6.6 引入并在 6.12 完整落地。EEVDF（Earliest Eligible Virtual Deadline First）通过**虚拟截止时间**实现更精确的公平性——每个进程有明确的时间预算，用完后必须等待下一轮。

```bash
# 查看当前调度器
$ cat /sys/kernel/debug/sched/features
SCHED_FEAT(GENTLE_FAIR_SLEEPERS, 1)
SCHED_FEAT(EEVDF, 1)                  # ← EEVDF 已启用
```

**惰性抢占模型 PREEMPT_LAZY (6.13)**：填补 `PREEMPT_VOLUNTARY`（自愿抢占）和 `PREEMPT_FULL`（完全抢占）之间的空白。`PREEMPT_LAZY` 允许进程在时间片内持续运行（减少上下文切换开销），但在 tick 边界接受抢占——吞吐量接近 voluntary，延迟接近 full。**适合 Redis/NGINX 等高吞吐+低延迟折中场景**。

```bash
# 查看当前抢占模型
$ cat /proc/config.gz | gunzip | grep PREEMPT
CONFIG_PREEMPT_LAZY=y                  # 惰性抢占
```

### 13.7.3 新系统调用

**listns(2) (6.19)**：直接、高效地列出系统中**所有命名空间**——不再需要遍历 `/proc` 或解析 sysfs。

```bash
# 经典的"查所有 netns"方式（慢+不完整）：
$ ls /var/run/netns/
$ ip netns list

# listns() 的优势：
# - 一次系统调用，内核直接返回完整的命名空间列表
# - 不依赖 procfs 是否挂载、/var/run 是否有对应文件
# - 容器运行时（containerd, cri-o）可以直接用 listns() 扫描所有 netns
```

**uretprobe 系统调用 (6.11)**：将 uprobe（用户态函数追踪）的**返回点探测**从内核实现改为专用 syscall，性能提升 **10-30%**。这对 `bpftrace`、`perf` 等工具的 uprobe 功能有直接加速效果。

### 13.7.4 存储与文件系统

**XFS/Ext4 原子写支持 (6.13)**：数据库（MySQL/PostgreSQL）的 WAL 日志和事务提交可以通过单次系统调用完成，**不再需要"先写数据→fsync→写元数据→fsync"的双重同步**。这是数据库在 Linux 上的性能拐点。

```bash
# 检查 XFS 是否支持原子写
$ grep atomic /proc/self/mountinfo | grep xfs
# 如果看到 "atomic" 即支持
```

**EROFS Zstandard 压缩 (6.10)**：容器镜像文件系统 EROFS 支持 zstd 压缩——比 gzip 高 30% 压缩率，比 lz4 小 50%。Kubernetes 集群的镜像拉取速度直接受益。

**RAID1 读负载均衡 (6.14)**：新增轮询（round-robin）、延迟感知（latency-aware）等多种读策略——传统只从主盘读，现在可以同时从多块镜像盘读取。

### 13.7.5 网络

**TCP 零拷贝接收 (6.12)**：将 TCP payload 直接存入 DMABUF（设备内存缓冲区），绕过内核的 socket buffer 拷贝——**GPU 可以直接从网卡 DMA 读取数据**。AI 训练场景下，GPU 节点收模型参数时不再需要 CPU 中转。

**TCP PSP 加密 (6.18)**：硬件加速的 TCP 连接级加密（类似 IPsec/TLS 但更轻量），可逐连接启用。配合支持 PSP 的网卡（Mellanox CX-7+），加密握手 < 1μs。

**精确 ECN (6.18)**：更精确的网络拥塞反馈机制，与 BBR v3 配合使用可获得最佳跨数据中心吞吐量。

### 13.7.6 Rust 与可观测性

| 版本 | Rust 里程碑 |
|------|------------|
| 6.10 | RISC-V 架构支持 |
| 6.11 | 块设备驱动框架（可写 Rust 块设备驱动） |
| 6.15 | **NOVA DRM 驱动**：首个 Rust 写的 GPU 驱动（NVIDIA 开源内核模块的替代方案） |
| 6.18 | Sheaves：SLUB 分配器高频小内存分配优化（Rust 驱动受益最多） |

**内核内存分析器 (6.10)**：低开销的生产环境内存分析工具——不需要重启、不需要 `perf`、直接读取 `/sys/kernel/debug/page_owner`。

**Panic 时显示 QR 码 (6.12)**：内核 panic 时在屏幕上生成 QR 码——用手机扫描即可获得完整的调用栈和寄存器信息。排查裸金属服务器 "无头" 崩溃的救星。

```bash
# 启用 QR 码 panic 输出
$ cat /boot/config-$(uname -r) | grep CONFIG_DRM_PANIC_QR_CODE
CONFIG_DRM_PANIC_QR_CODE=y
```

### 13.7.7 避坑指南

```bash
# 坑1：EEVDF 对实时应用的影响
#     EEVDF 的虚拟截止时间模型可能导致"优先级反转"——高优先级进程因预算耗尽被降级
#     解决：重要进程加 SCHED_FIFO/SCHED_RR 实时调度类

# 坑2：惰性抢占的性能错觉
#     PREEMPT_LAZY 的"低延迟"仅对 tick 边界有意义
#     tickless（CONFIG_NO_HZ_FULL）场景下，惰性抢占可能退化为 voluntary

# 坑3：mseal() 的内存碎片
#     mseal() 锁定后无法释放，长期运行可能导致碎片堆积
#     适用场景：短生命周期进程（Web请求处理），而非长期进程（数据库）

# 坑4：Rust 驱动的兼容性
#     NOVA DRM 驱动仅支持 NVIDIA Turing (20系)+ 和特定固件版本
#     生产部署前确认 GPU 型号和驱动版本
```

---

> **本节字数**：约 14,500 字（13.7 节）
> **涉及命令**：fio, bpftool, sysctl, ss, cyclictest, chrt, rustc, insmod, bpftrace, perf
> **基准版本**：Linux 6.6 LTS / 6.12 LTS / 6.19

---

## 课后练习

1. 概念题：io_uring 相比传统的 libaio 核心优势是什么？（至少两点）

思路：1. 批量提交/收割：一次系统调用提交 N 个请求，减少 syscall 开销；2. 零拷贝/内核轮询：支持 SQPOLL 模式，用户态无需系统调用也能完成 IO。

2. 排障题：bpftool sched_ext load my_sched.bpf.o 报错 Error: failed to load program: Operation not permitted，除了 root 权限，还缺少什么内核配置？

思路：缺少 CONFIG_SCHED_EXT（6.12+）或内核未开启。检查 uname -r 是否足够新，且 grep CONFIG_SCHED_EXT /boot/config-$(uname -r) 是否为 y。

3. 实操题：启用并永久配置 BBR v3 拥塞控制，写出完整的 sysctl 配置内容。

思路：

```
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```

保存至 /etc/sysctl.d/99-bbr.conf 并 sysctl --system。

4. 安全题：影子栈（Shadow Stack）防御的是哪一类攻击？需要什么硬件支持？

思路：防御 ROP（返回导向编程） 攻击，通过硬件比对普通栈和影子栈的返回地址。需要 Intel CET（11代酷睿+）或 AMD Zen 4+。

5. 实操题：在 RHEL 9 系统上安装实时内核 PREEMPT_RT 的包名是什么？安装后如何确认当前运行的是实时内核？

思路：sudo dnf install kernel-rt。安装后重启，uname -r 应包含 rt 字样，或 cat /sys/kernel/realtime 输出 1。

6. 对比题：EEVDF 调度器取代了 CFS，它引入的核心调度机制是什么？

思路：虚拟截止时间 (Earliest Eligible Virtual Deadline First)。每个进程分配一个时间片预算和截止时间，调度器优先选择截止时间最早的进程运行，实现更精确的公平性和更低的延迟抖动。

7. 排障题：应用使用了 mseal() 系统调用，但运行报错 Function not implemented。最可能的原因是什么？

思路：内核版本低于 6.10 或未编译 CONFIG_MSEAL=y。mseal() 是 6.10 才引入的。

8. 综合题：公司业务是跨大洲的数据库同步，网络延迟高（RTT > 200ms）。你会从本章中选取哪两个内核特性来优化？为什么？

思路：1. BBR v3：主动探测带宽，在高 BDP（带宽时延积）网络中吞吐量远超 CUBIC。2. TCP 零拷贝接收 (6.12+)：如果传输大块数据，零拷贝可大幅降低 CPU 负载。如果遇到丢包，配合精确 ECN 使用。

---

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

## S.7 进阶：将脚本日志接入 journald

**一句话定义**：`logger` 命令将 Shell 脚本的输出发送到 systemd-journald，实现结构化日志、自动轮转、统一查询——告别散落的 .log 文件。

### 为什么不用 echo >> /var/log/script.log？

```bash
# 传统方式（有问题）
$ echo "Backup completed" >> /var/log/backup.log
# 问题：
# 1. 需要自己管理轮转（logrotate 配置）
# 2. 没有优先级/级别区分
# 3. 无法按服务过滤
# 4. 缺少时间戳（需要手动 date）
```

### 使用 logger 接入 journald

```bash
#!/bin/bash
set -euo pipefail
# 带标签和优先级的日志
logger -t my-backup -p user.info "Backup started"
logger -t my-backup -p user.info "Backup completed: $(du -sh /backup)"
# 错误级别
if ! rsync -avz /data/ /backup/; then
    logger -t my-backup -p user.err "Backup failed with code $?"
    exit 1
fi
```

### 实战：全日志闭环的生产脚本模板

```bash
#!/bin/bash
# /usr/local/bin/production-job.sh
# 所有输出都走 journald，不写任何 .log 文件
set -euo pipefail
# 日志函数
log_info() {
    logger -t "$(basename "$0")" -p user.info "$*"
}
log_error() {
    logger -t "$(basename "$0")" -p user.err "$*" >&2
}
log_info "Job started"
# 后台运行的标准输出也重定向到 logger
do_heavy_work() {
    echo "Processing batch 1..."
    echo "Processing batch 2..."
}
do_heavy_work 2>&1 | logger -t my-job -p user.info
# 错误捕获
trap 'log_error "Script failed on line $LINENO"' ERR
log_info "Job completed successfully"
```

### 查询脚本日志

```bash
# 按标签查
$ journalctl -t my-backup -f
# 按优先级过滤
$ journalctl -t my-backup -p err
# 查看最近一次执行
$ journalctl -t my-job --since "5 minutes ago"
# 导出为 JSON 供分析
$ journalctl -t my-backup -o json | jq '.MESSAGE'
```

### 避坑指南

```bash
# 坑1：logger 默认是 user.notice，需显式指定级别
# 坑2：systemd 服务中默认已捕获 stdout，无需额外 logger
# 坑3：在 cron 中使用 logger 时，需确保 PATH 包含 /usr/bin
```


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
# 第三部分：附录 B —— 生产环境故障排查速查表

> **本章定位**：当你凌晨 3 点被报警叫醒，大脑一片空白时，这份速查表就是你最可靠的战友。每个场景都给出了“症状 → 排查命令 → 常见原因 → 解决方案”的完整链路。

---

## B.1 磁盘空间：有空间却无法写入

**症状**：
```bash
$ touch testfile
touch: cannot touch 'testfile': No space left on device
$ df -h /
/dev/sda2       50G   30G   18G  63% /         # 还有 18G！
```

**排查命令**：
```bash
# 1. 检查 inode 使用（根本原因）
$ df -i /
/dev/sda2      100000 99999     1  99% /        # inode 满了！

# 2. 找小文件最多的目录
$ for dir in /var /tmp /home; do
    echo "$dir: $(find $dir -xdev -type f 2>/dev/null | wc -l) files"
  done

# 3. 找具体哪些目录占用了最多 inode
$ find /var -xdev -type f | cut -d/ -f1-4 | sort | uniq -c | sort -rn | head -10
```

**常见原因**：
- `/tmp` 缓存文件堆积
- `/var/spool/postfix` 邮件队列堆积
- `/var/log` 小日志文件过多
- Docker overlay2 层文件过多

**解决方案**：
```bash
# 删除临时文件
$ sudo find /tmp -type f -mtime +7 -delete

# 清理邮件队列
$ sudo postsuper -d ALL

# 清理 Docker
$ docker system prune -f

# 清理 journald
$ sudo journalctl --vacuum-size=500M
```

---

## B.2 进程卡死：CPU 和内存正常但无响应

**症状**：
```bash
# 进程存在，但无响应
$ ps aux | grep myapp
user  1234  0.0  0.1  500M  50M  ?  S  10:00  0:00 myapp
# CPU 0%，内存正常，但状态是 S（可中断睡眠）
```

**排查命令**：
```bash
# 1. 查看进程在等什么
$ cat /proc/1234/wchan
do_futex                      # 在等 futex 锁

# 2. 查看进程调用栈
$ cat /proc/1234/stack
[<0>] futex_wait_queue_me+0xc0/0x120
[<0>] futex_wait+0x120/0x250
[<0>] do_futex+0x180/0x...       # 卡在互斥锁上

# 3. 查看进程打开的文件
$ lsof -p 1234
myapp  1234  user   4u  REG  8,2  0  131142 /tmp/.lockfile   # 等锁文件

# 4. strace 跟踪系统调用
$ sudo strace -p 1234
futex(0x7f..., FUTEX_WAIT, 2, NULL) = ?  # 在等 futex

# 5. 查看进程的所有线程
$ ps -L -p 1234
  PID   LWP  C NLWP
 1234  1234  0    4
 1234  1235  0    4
 1234  1236  0    4
$ sudo strace -p 1235          # 逐个线程跟踪
```

**常见原因**：
- 死锁（多个进程/线程互相等待锁）
- 等待网络响应（超时未设置）
- 等待磁盘 I/O（磁盘故障）
- 等待 NFS 挂载（NFS 服务器无响应）

**解决方案**：
```bash
# 1. 如果是死锁，查看锁状态
$ sudo perf record -p 1234 -g -e futex:* sleep 5
$ sudo perf report

# 2. 如果是网络超时，检查连接
$ ss -tnp | grep 1234

# 3. 如果是磁盘 I/O，检查磁盘状态
$ iostat -x 1 5
$ dmesg | tail -20 | grep -i "error\|fail"

# 4. 终极方案：优雅重启
$ kill -15 1234                  # 先 SIGTERM
$ sleep 5
$ kill -9 1234                   # 如果还在，强制杀
```

---

## B.3 服务无法启动：systemd 失败

**症状**：
```bash
$ sudo systemctl start nginx
Job for nginx.service failed because the control process exited with error code.
```

**排查命令**：
```bash
# 1. 查看服务状态
$ sudo systemctl status nginx
● nginx.service - A high performance web server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled)
   Active: failed (Result: exit-code) since ...
  Process: 1234 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
 Main PID: 1235 (code=exited, status=1/FAILURE)

# 2. 查看完整日志
$ sudo journalctl -u nginx -n 50 --no-pager
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)

# 3. 检查配置文件语法
$ sudo nginx -t
nginx: [emerg] unexpected "}" in /etc/nginx/nginx.conf:23

# 4. 检查端口占用
$ ss -tlnp | grep :80
LISTEN 0 128 *:80 *:* users:(("apache2",pid=5678,fd=4))
# 端口被 Apache 占用

# 5. 手动运行服务（模拟 systemd 环境）
$ sudo -u nginx /usr/sbin/nginx -g 'daemon off;'
```

**常见原因**：
- 配置文件语法错误
- 端口被占用
- 权限不足（无法读日志目录、无法绑定端口）
- 依赖服务未启动（数据库、网络）
- 环境变量缺失

**解决方案**：
```bash
# 1. 修复配置文件
$ sudo vi /etc/nginx/nginx.conf
$ sudo nginx -t                    # 验证语法

# 2. 停止占用端口的服务
$ sudo systemctl stop apache2
$ sudo systemctl disable apache2

# 3. 修复权限
$ sudo chown -R nginx:nginx /var/log/nginx
$ sudo chmod 755 /var/log/nginx

# 4. 检查依赖
$ sudo systemctl status postgresql
$ sudo systemctl start postgresql

# 5. 重载 systemd 配置
$ sudo systemctl daemon-reload
$ sudo systemctl start nginx
```

---

## B.4 网络不通：能 ping 通 IP 但无法解析域名

**症状**：
```bash
$ ping 8.8.8.8
64 bytes from 8.8.8.8: icmp_seq=1 ttl=118 time=15 ms    # 通

$ ping google.com
ping: google.com: Temporary failure in name resolution    # 域名不通
```

**排查命令**：
```bash
# 1. 查看 DNS 配置
$ cat /etc/resolv.conf
nameserver 127.0.0.53           # systemd-resolved

# 2. 检查 systemd-resolved 状态
$ resolvectl status
Global
         DNS Servers: 8.8.8.8 1.1.1.1
          DNSSEC NTA: 10.in-addr.arpa

# 3. 测试 DNS 解析
$ nslookup google.com 8.8.8.8           # 用 Google DNS
Server: 8.8.8.8
Address: 8.8.8.8#53

Non-authoritative answer:
Name:   google.com
Address: 142.250.80.46                  # 成功！

$ nslookup google.com                   # 用默认 DNS
;; connection timed out; no servers could be reached

# 4. 检查系统 DNS 缓存
$ sudo resolvectl flush-caches

# 5. 检查 /etc/nsswitch.conf
$ grep hosts /etc/nsswitch.conf
hosts:          files dns               # 先查 hosts，再查 DNS
```

**常见原因**：
- `/etc/resolv.conf` 被覆盖
- systemd-resolved 未运行
- DNS 服务器地址配置错误
- 防火墙阻挡 DNS（UDP 53）

**解决方案**：
```bash
# 1. 重启 systemd-resolved
$ sudo systemctl restart systemd-resolved

# 2. 临时修改 DNS
$ echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf

# 3. 永久修改（Ubuntu 24.04+）
$ sudo resolvectl dns eth0 8.8.8.8 1.1.1.1
$ sudo resolvectl domain eth0 example.com

# 4. 检查防火墙
$ sudo iptables -L -n -v | grep 53
# 如果阻止了 UDP 53，放开
$ sudo iptables -A OUTPUT -p udp --dport 53 -j ACCEPT
```

---

## B.5 内存不足：应用被 OOM Killer 杀掉

**症状**：
```bash
$ dmesg | tail -20
[12345.678] Out of memory: Kill process 1234 (mysql) score 987 or sacrifice child
[12345.679] Killed process 1234 (mysql) total-vm:12345678kB, anon-rss:9876543kB
$ systemctl status mysql
● mysql.service - MySQL Server
   Active: failed (Result: signal) since ...
```

**排查命令**：
```bash
# 1. 查看内存使用
$ free -h
              total        used        free      shared  buff/cache   available
Mem:           7.7Gi       7.2Gi       0.1Gi       0.2Gi       0.4Gi       0.3Gi
Swap:          2.0Gi       2.0Gi          0B       0.0Gi
# available 接近 0 → 内存紧张

# 2. 查看 OOM 日志
$ sudo grep -i "out of memory" /var/log/kern.log

# 3. 查看 OOM 得分
$ cat /proc/1234/oom_score
987                          # 越高越容易被杀

# 4. 查看进程内存排名
$ ps -eo pid,pmem,rss,cmd --sort=-rss | head -10
  PID %MEM   RSS CMD
 1234 89.0 7.2G /usr/sbin/mysqld
 5678  5.0 400M /usr/bin/redis-server

# 5. 查看内存泄漏
$ sudo pmap -x 1234 | sort -k3 -n | tail -20
```

**常见原因**：
- 应用内存泄漏（RSS 持续增长）
- 配置的 `memory.limit_in_bytes` 太低
- swap 不足或 swapiness 配置不当
- 系统总内存不足

**解决方案**：
```bash
# 1. 保护关键服务不被 OOM
$ sudo systemctl edit mysql
[Service]
OOMScoreAdjust=-1000

# 2. 增加 swap
$ sudo fallocate -l 4G /swapfile
$ sudo chmod 600 /swapfile
$ sudo mkswap /swapfile
$ sudo swapon /swapfile

# 3. 降低 swappiness（更少使用 swap）
$ sudo sysctl vm.swappiness=10

# 4. 限制应用内存（Docker）
$ docker run --memory="4g" ...

# 5. 限制应用内存（systemd）
$ sudo systemctl edit myapp
[Service]
MemoryMax=4G

# 6. 找出内存泄漏的应用并重启
$ sudo systemctl restart mysql
```

---

## B.6 CPU 飙高：负载异常升高

**症状**：
```bash
$ top
top - 10:00:00 up 30 days, load average: 12.50, 8.30, 4.20
Tasks: 234 total,   4 running, 230 sleeping
%Cpu(s): 80.0 us, 15.0 sy,  0.0 ni,  0.0 id,  5.0 wa
# load > 核心数（8核），CPU us 80%
```

**排查命令**：
```bash
# 1. 找 CPU 占用最高的进程
$ ps -eo pid,pcpu,cmd --sort=-pcpu | head -5
  PID %CPU CMD
 1234 150  /usr/bin/python3 heavy_calc.py    # 单进程超 100% = 多线程

# 2. 查看进程的线程
$ top -H -p 1234

# 3. perf 定位热点函数
$ sudo perf top -p 1234
# 看哪个函数消耗最多 CPU

# 4. 如果是内核态 CPU 高（sy > 50%）
$ sudo perf top
# 看是否是 _raw_spin_lock（锁竞争）

# 5. 如果是 IO 等待高（wa > 10%）
$ iostat -x 1 5
# 看 %util 和 await
```

**常见原因**：
- 应用负载增加（正常业务增长）
- 死循环 / 算法效率低
- 锁竞争严重
- 频繁的上下文切换
- 磁盘 I/O 瓶颈（wa 高）

**解决方案**：
```bash
# 1. 调整进程优先级
$ sudo renice -n 10 -p 1234

# 2. CPU 绑定（避免上下文切换）
$ sudo taskset -cp 0-3 1234

# 3. 如果是应用问题，重启
$ sudo systemctl restart myapp

# 4. 垂直扩容（增加 CPU）
# 云环境：升级实例规格

# 5. 水平扩容（增加实例）
# 增加服务副本数
```

---

## B.7 容器无法启动：Docker 常见故障

**症状**：
```bash
$ docker run -d nginx
docker: Error response from daemon: driver failed programming external connectivity ...
```

**排查命令**：
```bash
# 1. 查看容器日志
$ docker logs <container_id>

# 2. 查看 Docker 守护进程日志
$ sudo journalctl -u docker -n 50

# 3. 检查端口冲突
$ ss -tlnp | grep :80

# 4. 检查 cgroups 版本冲突
$ mount | grep cgroup
cgroup2 on /sys/fs/cgroup type cgroup2 ...
$ docker info | grep "Cgroup Driver"
Cgroup Driver: cgroupfs          # 应该为 systemd

# 5. 检查磁盘空间（Docker 数据目录）
$ docker system df
$ df -h /var/lib/docker

# 6. 检查 overlay2 损坏
$ docker info | grep "Storage Driver"
Storage Driver: overlay2
$ sudo dmesg | grep overlay
```

**常见原因**：
- 端口被占用
- cgroup 驱动不匹配
- 磁盘空间不足
- 镜像损坏
- 挂载卷权限错误

**解决方案**：
```bash
# 1. 修复 cgroup 驱动
$ cat /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
$ sudo systemctl restart docker

# 2. 清理 Docker 数据
$ docker system prune -a -f

# 3. 重新拉取镜像
$ docker pull nginx:latest

# 4. 检查挂载权限
$ ls -la /host/path              # 确保容器用户有权限
$ docker run -v /host/path:/container/path:rw ...

# 5. 使用 podman 替代（无守护进程）
$ podman run -d nginx
```

---

## B.8 SSH 无法登录

**症状**：
```bash
$ ssh user@server
ssh: connect to host server port 22: Connection refused
# 或
Permission denied (publickey,password)
```

**排查命令**：
```bash
# 1. 检查 SSH 服务状态（控制台或带外管理）
$ sudo systemctl status sshd
● ssh.service - OpenBSD Secure Shell server
   Active: failed (Result: exit-code) since ...

# 2. 查看 SSH 日志
$ sudo journalctl -u sshd -n 50
sshd[1234]: fatal: /etc/ssh/sshd_config: line 42: Bad configuration option: Portt

# 3. 验证配置文件
$ sudo sshd -t
/etc/ssh/sshd_config: line 42: Bad configuration option: Portt

# 4. 检查端口
$ ss -tlnp | grep :22
# 无输出 = 服务未启动或端口被改

# 5. 检查防火墙
$ sudo iptables -L -n -v | grep 22
```

**常见原因**：
- sshd_config 语法错误
- 端口被修改或占用
- 防火墙阻挡
- `/etc/hosts.allow` / `/etc/hosts.deny`
- PAM 配置错误

**解决方案**：
```bash
# 1. 修复配置文件
$ sudo vi /etc/ssh/sshd_config
$ sudo sshd -t                     # 验证
$ sudo systemctl restart sshd

# 2. 如果端口被改，从控制台检查
$ sudo grep "^Port" /etc/ssh/sshd_config
Port 2222
$ ssh -p 2222 user@server

# 3. 放行防火墙
$ sudo iptables -A INPUT -p tcp --dport 2222 -j ACCEPT

# 4. 检查 PAM
$ sudo grep -v "^#" /etc/pam.d/sshd
```

---

## B.9 日志轮转失败：日志文件过大

**症状**：
```bash
$ ls -lh /var/log/nginx/access.log
-rw-r--r-- 1 root root 50G Jul  3 10:00 /var/log/nginx/access.log
# 日志文件 50GB，未轮转
```

**排查命令**：
```bash
# 1. 检查 logrotate 状态
$ sudo logrotate -d /etc/logrotate.d/nginx
# 看是否有错误

# 2. 检查 logrotate 是否运行
$ sudo systemctl status logrotate
$ ls -la /etc/cron.daily/logrotate

# 3. 检查磁盘空间
$ df -h /var/log

# 4. 检查文件权限
$ ls -la /var/log/nginx/
-rw-r--r-- 1 root root 50G access.log
# 属主是 root，nginx 用户可能无法轮转

# 5. 检查进程是否仍持有旧 fd
$ sudo lsof | grep "access.log (deleted)"
nginx  1200  root   5w  REG  8,2  50G  131142 /var/log/nginx/access.log (deleted)
```

**解决方案**：
```bash
# 1. 修复权限（/etc/logrotate.d/nginx）
create 640 nginx adm              # 确保新日志属主正确

# 2. 强制轮转
$ sudo logrotate -f /etc/logrotate.d/nginx

# 3. 手动清理
$ sudo truncate -s 0 /var/log/nginx/access.log
$ sudo systemctl reload nginx     # 让 nginx 重新打开日志

# 4. 检查 postrotate 脚本
# 确保 postrotate 中的信号正确
kill -USR1 $(cat /run/nginx.pid)
```

---

## B.10 systemd 服务启动超时

**症状**：
```bash
$ sudo systemctl start myapp
Job for myapp.service failed because a timeout was exceeded.
```

**排查命令**：
```bash
# 1. 查看服务状态
$ sudo systemctl status myapp
● myapp.service - My Application
   Active: activating (start) since ...
  Control: 1234 (myapp)
   CGroup: /system.slice/myapp.service
           └─1234 /usr/local/bin/myapp --config /etc/myapp.conf

# 2. 查看超时配置
$ systemctl show myapp -p TimeoutStartSec
TimeoutStartSec=1min 30s

# 3. 查看日志
$ sudo journalctl -u myapp -f
# 看进程卡在哪一步

# 4. 检查是否在等网络/数据库
$ sudo strace -p 1234
connect(3, {sa_family=AF_INET, sin_port=htons(5432), ...}, 16) = -1 EINPROGRESS
# 在等 PostgreSQL 连接

# 5. 检查依赖服务状态
$ sudo systemctl status postgresql
```

**解决方案**：
```bash
# 1. 增加超时时间
$ sudo systemctl edit myapp
[Service]
TimeoutStartSec=5min
TimeoutStopSec=5min

$ sudo systemctl daemon-reload

# 2. 检查依赖链
$ systemctl list-dependencies myapp

# 3. 改为异步启动（不等待）
[Unit]
After=network.target
Wants=postgresql.service          # 软依赖
# 不需要 Requires 或 After 等待

# 4. 使用 Type=notify（应用主动通知就绪）
[Service]
Type=notify
# 应用代码中调用 sd_notify(0, "READY=1")
```

---

## B.11 文件权限错误：Permission denied

**症状**：
```bash
$ cat /var/log/app.log
cat: /var/log/app.log: Permission denied
```

**排查命令**：
```bash
# 1. 查看文件权限
$ ls -l /var/log/app.log
-rw------- 1 root root 1024 Jul  3 10:00 /var/log/app.log

# 2. 查看当前用户
$ id
uid=1000(appuser) gid=1000(appuser)

# 3. 查看 ACL
$ getfacl /var/log/app.log

# 4. 查看 SELinux（RHEL）
$ ls -Z /var/log/app.log
system_u:object_r:var_log_t:s0 /var/log/app.log
$ sudo ausearch -m avc -ts recent

# 5. 查看 AppArmor（Ubuntu）
$ sudo aa-status | grep app
```

**解决方案**：
```bash
# 1. 修改权限
$ sudo chmod 644 /var/log/app.log

# 2. 修改属主
$ sudo chown appuser:appuser /var/log/app.log

# 3. 加入合适的组
$ sudo usermod -aG adm appuser

# 4. SELinux（修改标签）
$ sudo chcon -t var_log_t /var/log/app.log
$ sudo restorecon -v /var/log/app.log

# 5. AppArmor（修改 profile）
$ sudo vi /etc/apparmor.d/usr.sbin.myapp
/var/log/app.log rw,
$ sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.myapp
```

---

## B.12 内核崩溃（Kernel Panic）排查

**症状**：
- 服务器突然无响应
- 控制台显示 Kernel Panic 信息
- 系统自动重启（如果配置了 panic reboot）

**排查命令**：
```bash
# 1. 查看上次启动日志（崩溃前）
$ journalctl -b -1 -k | tail -100

# 2. 查看 crash dump（如果配置了 kdump）
$ ls /var/crash/
vmcore.20260703-100000

# 3. 分析 vmcore
$ crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /var/crash/vmcore

# 4. 检查硬件错误
$ sudo dmesg | grep -i "hardware error\|mce\|machine check"
```

**常见原因**：
- 硬件故障（内存 ECC 错误、CPU 过热）
- 内核 bug（新内核不稳定）
- 驱动 bug（第三方驱动）
- 内存不足（OOM 触发的 panic）

**解决方案**：
```bash
# 1. 启动到旧内核
# GRUB 启动时选择 Advanced options → 选择旧内核版本

# 2. 配置 panic 自动重启
$ sudo sysctl -w kernel.panic=10          # panic 后 10 秒重启
$ echo "kernel.panic = 10" | sudo tee -a /etc/sysctl.conf

# 3. 启用 kdump 保存崩溃现场
$ sudo apt install kdump-tools
$ sudo systemctl enable kdump-tools

# 4. 内存测试
$ sudo memtest86+                        # 从 GRUB 启动
```

---

## B.13 速查表总结

| 场景 | 症状 | 排查命令 | 常见原因 |
|------|------|----------|----------|
| 磁盘空间 | `No space left` | `df -i` | inode 耗尽 |
| 进程卡死 | CPU 0% 无响应 | `cat /proc/PID/wchan` | 死锁/等待 |
| 服务启动失败 | systemd failed | `systemctl status` + `journalctl` | 配置错误/端口占用 |
| 网络不通 | ping IP 通但域名不通 | `resolvectl status` | DNS 配置错误 |
| OOM | 进程被杀 | `dmesg \| grep oom` | 内存不足 |
| CPU 飙高 | load > 核心数 | `ps -eo pcpu` | 应用负载/锁竞争 |
| 容器失败 | Error response | `docker logs` + `journalctl -u docker` | cgroup/端口 |
| SSH 拒绝 | Connection refused | `sshd -t` | 配置语法错误 |
| 日志不轮转 | 单文件 > 10G | `logrotate -d` | 权限/信号错误 |
| 超时 | Timeout exceeded | `systemctl show -p TimeoutStartSec` | 依赖服务未就绪 |
| 权限错误 | Permission denied | `ls -l` + `getfacl` | 权限/ACL/SELinux |
| 内核崩溃 | Kernel Panic | `journalctl -b -1 -k` | 硬件/驱动/内核 bug |

---

> **附录字数**：约 6,000 字
> **涉及命令**：df, find, cat, ps, lsof, strace, systemctl, journalctl, resolvectl, docker, sshd, logrotate, dmesg, crash, sysctl

---

> **涉及命令**：df -i, strace, perf, journalctl -b -1, tcpdump, ss, docker inspect, getenforce, 覆盖 12 类常见故障场景。# 附录C：常用 Linux 命令速查表

> 按功能分类的一页纸命令索引，适合案头速查。

---

## 系统信息

| 命令 | 用途 |
|------|------|
| `uname -a` | 内核版本 |
| `hostnamectl` | 主机信息 |
| `uptime` | 运行时间+负载 |
| `lscpu / lsblk / lsusb` | CPU/块设备/USB设备 |
| `cat /proc/cpuinfo` | CPU详细信息 |
| `cat /proc/meminfo` | 内存详细信息 |
| `df -h / df -i` | 磁盘空间/inode |
| `du -sh <dir>` | 目录占用 |

## 文件管理

| 命令 | 用途 |
|------|------|
| `ls -lah` | 列出（含隐藏、人类可读） |
| `cp -a` / `mv` / `rm -i` | 复制/移动/删除（确认） |
| `find . -name "*.log" -mtime +7 -delete` | 删7天前的日志 |
| `grep -r "pattern" /path/` | 递归搜索 |
| `ln -s target link` | 创建软链接 |
| `stat file` | 文件元信息 |
| `file file` | 文件类型判断 |

## 权限管理

| 命令 | 用途 |
|------|------|
| `chmod 755 script.sh` | 设权限 |
| `chown user:group file` | 改属主 |
| `sudo -i` | 切换到root |
| `getfacl / setfacl` | ACL查看/设置 |
| `chattr +i /etc/passwd` | 防篡改 |
| `lsattr file` | 看属性 |

## 进程管理

| 命令 | 用途 |
|------|------|
| `ps aux --sort=-%cpu \| head` | CPU占用Top进程 |
| `top / htop / btop` | 实时监控 |
| `kill -15 PID` | 优雅终止 |
| `kill -9 PID` | 强制终止 |
| `nohup cmd &` / `disown` | 后台+免疫SIGHUP |
| `strace -p PID` | 追踪系统调用 |
| `lsof -p PID` | 进程打开的文件 |

## 网络诊断

| 命令 | 用途 |
|------|------|
| `ip a` | IP地址 |
| `ip route` | 路由表 |
| `ss -tlnp` | 监听端口 |
| `curl -v https://url` | HTTP调试 |
| `ping -c 4 host` | 连通性 |
| `traceroute host` / `mtr host` | 路由追踪 |
| `sudo tcpdump -i eth0 port 80` | 抓包 |

## 服务管理（systemd）

| 命令 | 用途 |
|------|------|
| `systemctl status nginx` | 服务状态 |
| `systemctl restart nginx` | 重启 |
| `systemctl enable --now nginx` | 开机自启+立即启动 |
| `journalctl -u nginx -f` | 实时日志 |
| `systemctl list-timers` | 定时器列表 |

## 性能调优

| 命令 | 用途 |
|------|------|
| `free -h` | 内存 |
| `vmstat 2 5` | 综合状态 |
| `iostat -x 2 3` | 磁盘IO |
| `sar -n DEV 1` | 网络流量 |
| `perf top` | CPU热点 |
| `sudo bpftrace -e '...'` | eBPF追踪 |
| `sysctl -a \| grep <key>` | 内核参数 |

## 包管理

| apt (Debian) | dnf (RHEL) | 用途 |
|-------------|------------|------|
| `apt update` | `dnf check-update` | 更新索引 |
| `apt install` | `dnf install` | 安装 |
| `apt upgrade` | `dnf upgrade` | 升级 |
| `apt purge` | `dnf remove` | 卸载 |
| `apt-mark hold` | `dnf versionlock` | 锁定版本 |

## 安全审计

| 命令 | 用途 |
|------|------|
| `sudo ausearch -k key -ts today` | 审计日志查询 |
| `getenforce / sestatus` | SELinux状态 |
| `aa-status` | AppArmor状态 |
| `find / -perm -4000 2>/dev/null` | SUID文件扫描 |
| `fail2ban-client status sshd` | 防暴力破解 |

## 容器（Docker/Podman）

| 命令 | 用途 |
|------|------|
| `docker ps / podman ps` | 运行中的容器 |
| `docker logs -f container` | 容器日志 |
| `docker exec -it container sh` | 进入容器 |
| `docker-compose up -d` | 启动服务栈 |
| `docker system prune -a` | 清理未使用资源 |

## 救援与恢复

| 场景 | 命令 |
|------|------|
| 单用户模式 | GRUB 加 `single` 或 `init=/bin/bash` |
| 重置root密码 | `passwd`（单用户模式中） |
| 修复fstab | 单用户模式中 `mount -o remount,rw /` |
| 磁盘修复 | `fsck -y /dev/sda1`（先umount） |
| XFS修复 | `xfs_repair /dev/vg0/data` |

---

> **涉及命令**：约 80 个常用命令，覆盖系统管理全场景。
# 附录D：全书概念索引（133个核心概念）

## 一、系统基础与架构（新增至18个）

**1. 内核（Kernel）**&#x5185;核是操作系统的核心，直接附着在硬件平台之上，控制和管理系统内各种资源（CPU、内存、设备），并向上层提供系统调用接口。它负责进程管理、内存管理、文件系统管理、设备驱动和安全控制等核心功能。Linux内核采用单块架构，但支持模块化加载，允许在运行时动态链接或解除链接功能模块，实现了高效与灵活的统一。

**2. 发行版（Distribution）**&#x53D1;行版是基于Linux内核的完整操作系统套装，包含内核、系统工具、应用软件和包管理器。常见的发行版有Ubuntu（面向桌面用户）、CentOS（面向服务器）、Arch（面向高级用户）等。发行版通过整合GNU工具集、桌面环境（如GNOME、KDE）和软件仓库，降低了用户使用Linux的门槛，同时提供了差异化的生态体验。

**3. Shell**Shell是用户与内核交互的命令解释器，位于操作系统的最外层，接收用户输入的命令并将其传递给内核执行。常见的Shell有Bash（Bourne Again Shell）、Zsh、Fish等。Shell不仅是一个命令行界面，还是一种编程语言，支持变量、循环、条件判断等结构，用户可以通过编写Shell脚本自动化完成复杂任务。

**4. 终端（Terminal）**&#x7EC8;端是运行Shell的窗口程序，提供了文本输入输出的图形化界面。常见的终端模拟器有GNOME Terminal、Konsole、Terminator等。终端本身不执行命令，而是启动一个Shell进程，将用户的键盘输入传递给Shell，并将Shell的输出显示在屏幕上。终端还支持多标签、分屏、配色方案等增强功能。

**5. 控制台（Console）**&#x63A7;制台是系统启动时直接显示的文本界面，通常通过Ctrl+Alt+F1\~F6切换。控制台不依赖图形环境，在系统故障或图形界面无法启动时，仍可通过控制台进行系统维护。每个控制台对应一个独立的虚拟终端，支持多用户同时登录，是Linux系统可靠性的重要保障。

**6. 根用户（Root）**&#x6839;用户是Linux系统中的超级管理员，拥有对系统的完全控制权，其用户标识符（UID）固定为0。根用户可以访问任何文件、执行任何命令、修改系统配置。由于权限过大，日常操作应避免直接使用root账户，而是通过sudo命令临时提权，这样既能完成管理任务，又能记录操作日志，提高安全性。

**7. sudo**sudo（superuser do）允许普通用户以root或其他用户的身份执行命令，同时将操作记录到日志中。通过配置/etc/sudoers文件，管理员可以精细控制哪些用户、在哪些主机上、可以执行哪些命令。sudo相比直接使用root账户更安全，因为它要求用户输入自己的密码（而非root密码），且可以限制命令范围，防止误操作。

**8. 系统调用（System Call）**&#x7CFB;统调用是用户程序请求内核服务的接口，是用户态进入内核态的唯一合法途径。常见的系统调用有open()、read()、write()、fork()、exec()等。当应用程序需要访问硬件设备、创建进程、分配内存时，必须通过系统调用陷入内核，由内核代表应用程序完成操作。系统调用是操作系统提供的最小功能单元，库函数和Shell命令最终都依赖系统调用实现。

**9. POSIX**POSIX（Portable Operating System Interface）是IEEE制定的操作系统接口标准，旨在确保不同Unix系统之间的应用程序可移植性。POSIX定义了系统调用、库函数、Shell、环境变量等规范。Linux遵循POSIX标准，因此许多为其他Unix系统编写的程序可以不经修改地在Linux上编译运行，这也是Linux能够成为Unix生态重要成员的原因。

**10. GNU工具集**GNU（GNU's Not Unix）是自由软件基金会发起的项目，旨在创建一个完全自由的操作系统。GNU工具集包括编译器（GCC）、调试器（GDB）、文本编辑器（Emacs）、Shell（Bash）以及大量命令行工具（ls、grep、sed、awk等）。Linux内核与GNU工具集结合，形成了完整的操作系统，因此严格来说应称为“GNU/Linux”。

**11. 引导流程（Boot Process）**&#x4C;inux系统的引导流程从BIOS/UEFI开始，依次经过引导加载程序（GRUB）、内核加载、init进程启动（现代系统使用systemd）。具体步骤：BIOS/UEFI执行硬件自检并加载GRUB；GRUB读取配置文件，加载内核和**initramfs**（初始内存文件系统）到内存；内核初始化硬件、挂载根文件系统；最后启动init进程（PID=1），由init负责启动其他系统服务。整个流程体现了从硬件到软件的逐层抽象。

**12. init 系统**init是系统启动后的第一个用户空间进程，负责启动、监控和终止其他系统服务。传统SysV init使用串行启动方式，效率较低；现代Linux广泛使用systemd，它采用并行启动、按需启动、依赖管理等机制，大大加快了系统启动速度。systemd还提供了systemctl、journalctl等工具，统一管理服务、日志和系统状态。

**13. cgroups（控制组）**&#x63;groups是Linux内核特性，用于限制、隔离和统计进程组的资源使用（CPU、内存、磁盘I/O、网络带宽）。它通过将进程分组，并为每个组设置资源配额，防止某个进程过度消耗资源影响整个系统。cgroups是容器技术（如Docker、LXC）的底层支撑，每个容器可以看作一个独立的cgroup，实现资源隔离和限制。

**14. 命名空间（Namespace）**&#x547D;名空间是Linux内核的隔离机制，为进程提供独立的系统资源视图。常见的命名空间包括PID（进程隔离）、Network（网络栈隔离）、Mount（文件系统挂载点隔离）、UTS（主机名隔离）、IPC（进程间通信隔离）、User（用户ID隔离）等。命名空间与cgroups结合，构成了容器技术的核心，使得容器内的进程仿佛运行在独立的操作系统中。

**15. 虚拟文件系统（VFS）**&#x56;FS是Linux内核的抽象层，位于用户进程和具体文件系统之间，定义了一组所有文件系统都支持的数据结构和标准接口（如open、read、write）。用户进程只需与VFS交互，无需关心底层是ext4、XFS还是NFS。VFS屏蔽了不同文件系统的实现差异，实现了“一切皆文件”的设计哲学——普通文件、目录、设备、管道、套接字都通过统一的文件接口操作。

**16. sysctl（内核参数动态调优）**&#x73;ysctl是运行时查看和修改内核参数的接口，通过`/proc/sys/`虚拟文件系统实现。使用`sysctl -a`查看所有参数，`sysctl -w net.ipv4.ip_forward=1`临时开启路由转发。持久化配置写入`/etc/sysctl.conf`或`/etc/sysctl.d/*.conf`。生产环境中，`fs.file-max`（最大文件句柄数）、`net.core.somaxconn`（监听队列大小）等参数的调优是保障高并发服务稳定的关键。

**17. initramfs（初始内存文件系统）**&#x69;nitramfs是一个临时的根文件系统，在Linux内核启动初期加载到内存中。它包含加载真实根文件系统所需的驱动（如SATA、NVMe、LVM、RAID）和工具。内核先挂载initramfs，执行其中的`/init`脚本探测硬件、加载模块，然后`pivot_root`切换到真正的根目录。initramfs解决了内核体积限制和驱动加载顺序问题，是系统引导不可或缺的一环。

**18. 运行时目录（/run）**`/run`是`tmpfs`（内存文件系统）挂载的临时目录，用于存储系统启动以来的运行时数据，如进程PID文件（`/run/nginx.pid`）、登录会话信息、systemd运行时单元等。与`/var/run`（通常是指向`/run`的符号链接）不同，`/run`在系统启动早期即可用，且重启后数据自动清空，是系统运行时状态的核心存放地。

## 二、文件系统与目录结构（新增至22个）

**19. 一切皆文件**“一切皆文件”是Unix/Linux的核心设计哲学，意味着所有系统资源（普通文件、目录、设备、管道、套接字、进程信息等）都可以通过文件描述符和统一的read()/write()接口进行访问。这种抽象简化了程序设计，使得用户可以使用相同的系统调用来操作不同类型的资源。例如，向/dev/null写入数据等同于丢弃数据，从/dev/random读取数据则获得随机数。

**20. 根目录（/）**&#x6839;目录是Linux文件系统树状结构的起点，用斜杠“/”表示。所有文件、目录、设备、挂载点都位于根目录之下。根目录是文件系统的顶层，其内容由FHS标准规定，包含/bin、/etc、/home、/usr、/var等子目录。根目录本身也是一个目录文件，其inode编号通常为2，由系统在格式化时创建。

**21. FHS（文件系统层次结构标准）**&#x46;HS定义了Linux系统中目录的布局和用途，确保不同发行版之间的兼容性。例如，/bin存放基本命令（ls、cp），/etc存放配置文件，/var存放可变数据（日志、邮件），/usr存放用户程序和只读数据，/home存放用户家目录。遵循FHS的发行版，用户和管理员可以快速定位文件，便于系统维护和脚本编写。

**22. inode**inode（索引节点）是文件系统中存储文件元数据的数据结构，每个文件或目录都有一个唯一的inode。inode包含文件类型、权限（rwx）、所有者（UID/GID）、文件大小、时间戳（atime/mtime/ctime）、数据块指针、硬链接计数等信息。文件名不存储在inode中，而是存储在目录项中。inode是文件系统的核心，通过它内核可以快速定位文件数据。

**23. 硬链接（Hard Link）**&#x786C;链接是多个目录项指向同一个inode的链接方式。创建硬链接后，文件名和原文件共享相同的inode和数据块，删除其中一个不会影响另一个（只有当链接计数降为0时，inode和数据块才会被释放）。硬链接不能跨文件系统创建，也不能用于目录（防止形成循环）。硬链接常用于备份和文件共享，节省磁盘空间。

**24. 符号链接（Symbolic Link）**&#x7B26;号链接（软链接）是一个独立的文件，其内容存储了目标文件的路径。访问符号链接时，内核会自动重定向到目标文件。与硬链接不同，符号链接可以跨文件系统，也可以指向目录。但删除原文件后，符号链接会变成“悬空链接”，访问时会报错。符号链接常用于创建快捷方式、管理软件版本（如/usr/bin/python -> python3）。

**25. 挂载（Mount）**&#x6302;载是将一个文件系统附加到目录树的过程，使得该文件系统的内容可以通过指定的挂载点目录访问。例如，mount /dev/sda1 /mnt 将分区/dev/sda1挂载到/mnt目录。挂载点必须是已存在的空目录，挂载后该目录的原有内容会被临时隐藏。Linux支持多种文件系统类型，通过mount命令的-t参数指定。持久化挂载需写入/etc/fstab文件。

**26. ext4**ext4是第四代扩展文件系统，是Linux最主流的文件系统之一。它支持最大1EB的分区和16TB的单个文件，具备日志机制（保证数据一致性）、区段（Extents）分配（减少碎片化）、延迟分配、多块分配等特性。ext4向下兼容ext2/ext3，性能稳定，适用于服务器系统盘、个人电脑和普通数据存储。

**27. XFS**XFS是一种高性能64位日志文件系统，由SGI开发，现为RHEL/CentOS的默认文件系统。它支持最大8EB的分区和9EB的单个文件，并发读写性能优异，特别适合大文件和视频存储场景。XFS采用分配组（Allocation Group）设计，支持在线扩容，但缩容较困难。在数据库服务器和高性能计算集群中广泛使用。

**28. Btrfs**Btrfs（B-tree文件系统）是一种写时复制（CoW）文件系统，支持快照、压缩、子卷、RAID管理、数据校验等高级功能。它允许动态调整分区大小，提供自我修复能力（通过校验和检测数据损坏）。**时至今日，Btrfs已进入生产就绪状态，是RHEL 9和Fedora的默认文件系统**，在容器存储、桌面系统和通用服务器中应用广泛；但在极高并发OLTP数据库场景下，仍需结合具体负载进行压测评估。

**29. LVM（逻辑卷管理）**&#x4C;VM是Linux存储管理的抽象层，将物理磁盘（PV）组合成卷组（VG），再从中划分逻辑卷（LV）。逻辑卷支持在线动态扩容、缩容（需文件系统配合）、快照回滚和条带化。LVM极大地提高了存储的灵活性，生产环境中通常将系统盘和数据库目录部署在LVM上，以便在磁盘空间不足时在线扩展，避免了停机扩容的麻烦。

**30. RAID（独立磁盘冗余阵列）**&#x52;AID通过将多块物理磁盘组合成一个逻辑单元，实现性能提升或数据冗余。常用级别：RAID 0（条带化，性能高但无冗余）、RAID 1（镜像，高冗余但容量减半）、RAID 5（分布式奇偶校验，兼顾容量与冗余）、RAID 10（镜像+条带，性能与冗余兼得）。Linux支持软RAID（通过`mdadm`管理）和硬件RAID。在服务器部署中，系统盘常用RAID 1，数据盘常用RAID 10或RAID 5。

**31. Swap（交换空间）**&#x53;wap是磁盘上的交换分区或交换文件，当物理内存不足时，内核将不活跃的内存页换出到Swap，从而释放物理内存给活跃进程。Swap并非越多越好，传统“双倍内存”规则已过时。核心调优参数是`vm.swappiness`（默认60），值越小越倾向使用物理内存，值越大越积极换出。在容器和数据库环境中，常调低`swappiness`以保障性能。

**32. /proc**/proc是一个虚拟文件系统，不占用磁盘空间，由内核在内存中动态生成。它反映了内核和进程的实时状态，例如/proc/cpuinfo显示CPU信息，/proc/meminfo显示内存使用情况，/proc/\/ 目录下包含每个进程的详细信息。用户可以通过读取/proc文件获取系统信息，也可以通过写入某些文件（如/proc/sys/）动态调整内核参数。

**33. /sys**/sys是sysfs虚拟文件系统，提供设备、驱动、电源管理、总线等硬件信息的统一视图。它由内核在启动时创建，将设备模型以文件形式呈现。例如，/sys/class/net/ 下包含网卡信息，/sys/block/ 下包含块设备信息。udev工具利用/sys中的信息动态管理设备节点，实现即插即用功能。

**34. /dev**/dev目录存放设备文件，是用户空间访问硬件设备的接口。设备文件分为字符设备（如/dev/tty、/dev/random）和块设备（如/dev/sda、/dev/nvme0n1）。传统上设备文件是静态创建的，现代Linux使用udev（用户空间设备管理器）动态创建和删除设备节点，当硬件插入或移除时自动更新/dev目录。

**35. /tmp**/tmp目录用于存放临时文件，任何用户都可以在此创建文件，但通常只有文件所有者才能删除（粘滞位保护）。/tmp通常挂载为tmpfs（基于内存的文件系统），重启后内容自动清空。系统服务也会使用/tmp存储临时数据，因此应确保/tmp有足够的空间并定期清理。

**36. /var**/var目录存放可变数据，包括系统日志（/var/log）、邮件队列（/var/mail）、打印队列（/var/spool）、数据库文件（/var/lib）、临时文件（/var/tmp）等。与/tmp不同，/var/tmp中的文件在重启后通常保留。/var是系统运行过程中数据增长的主要区域，需要监控磁盘使用情况，并配置logrotate等工具管理日志轮转。

## 三、文件操作与权限（新增至14个）

**37. 文件权限（rwx）**&#x4C;inux文件权限使用9位二进制表示，分为三组：所有者（User）、所属组（Group）、其他用户（Other），每组包含读（r=4）、写（w=2）、执行（x=1）三种权限。例如，-rwxr-xr--表示所有者可读写执行，组用户可读可执行，其他用户只读。权限通过chmod命令修改，数字模式（如755）或符号模式（如u+x）均可。权限信息存储在inode中。

**38. chmod**chmod用于修改文件或目录的访问权限。数字模式：chmod 755 file 将权限设为rwxr-xr-x（所有者7=4+2+1，组5=4+1，其他5=4+1）。符号模式：chmod u+x file 给所有者添加执行权限，chmod o-w file 移除其他用户的写权限。chmod -R 可递归修改目录及其子文件权限。注意：只有文件所有者和root才能修改权限。

**39. chown**chown用于修改文件或目录的所有者和所属组。语法：chown user:group file，例如chown alice:staff data.txt 将文件所有者改为alice，组改为staff。只修改所有者：chown alice file；只修改组：chown :staff file。chown -R 递归修改。普通用户不能将文件所有权转让给他人，只有root可以任意修改。

**40. chgrp**chgrp专门用于修改文件或目录的所属组，功能与chown :group相同。语法：chgrp group file。普通用户只能将文件组改为自己所属的组（且必须是该组的成员），root可以改为任意组。chgrp -R 递归修改。chgrp是chown的子集，但单独提供便于记忆和使用。

**41. umask**umask是文件创建时的默认权限掩码，用于控制新文件或目录的初始权限。umask值从系统权限中减去，得到实际权限。例如，umask 022 表示新文件权限为666-022=644（rw-r--r--），新目录权限为777-022=755（rwxr-xr-x）。umask通常在Shell配置文件（如.bashrc）中设置，影响当前会话中所有新建文件。

**42. ACL（访问控制列表）**&#x41;CL提供了比传统rwx更细粒度的权限控制，允许为特定用户或组单独设置权限，而不仅限于所有者、组和其他三类。例如，setfacl -m u:alice:rw file 给用户alice赋予读写权限。ACL通过getfacl查看，setfacl设置。ACL在需要复杂权限共享的场景（如多用户协作的项目目录）中非常有用，但会略微增加文件系统开销。

**43. setuid/setgid**setuid和setgid是特殊权限位，当应用于可执行文件时，允许用户以文件所有者（setuid）或文件所属组（setgid）的身份运行该程序，而不是以当前用户身份。例如，/usr/bin/passwd设置了setuid位，普通用户执行它时临时获得root权限，从而可以修改/etc/shadow文件。setgid还可用于目录，使在该目录下创建的新文件继承目录的组。

**44. 粘滞位（Sticky Bit）**&#x7C98;滞位（t位）主要用于/tmp等共享目录，防止非文件所有者删除或重命名其他用户的文件。设置粘滞位后，目录的权限显示为drwxrwxrwt（最后一位t）。只有文件所有者、目录所有者或root才能删除文件。粘滞位通过chmod +t dir设置，对普通文件无意义。它是多用户系统中保护临时文件安全的重要机制。

**45. chattr（文件属性与不可变位）**&#x63;hattr用于设置文件的扩展属性（ext2/ext3/ext4系列文件系统特有）。生产环境最常用`chattr +i /etc/shadow`（设置不可变位），此时任何用户（包括root）都无法修改、删除或重命名该文件，除非执行`chattr -i`解除。`chattr +a`仅允许追加写入，适合日志文件。`lsattr`查看属性。这是系统安全加固中防御勒索病毒或恶意篡改的终极防线之一。

**46. Linux Capabilities（能力机制）**&#x43;apabilities将root的超级权限拆分为独立的“能力位”（如`CAP_NET_ADMIN`管理网络、`CAP_SYS_TIME`修改系统时间、`CAP_NET_BIND_SERVICE`绑定1024以下端口）。通过`setcap cap_net_bind_service+ep /usr/bin/nginx`，普通用户也能启动监听80端口的Nginx，无需`setuid`赋权整个root。这种“最小权限”原则极大降低了安全风险，是容器化和现代服务部署的重要实践。

**47. ls -l**ls -l以长格式列出文件详细信息，输出格式为：-rw-r--r-- 1 alice staff 1024 Jan 1 10:00 file.txt。各部分含义：第一个字符表示文件类型（-普通文件，d目录，l符号链接），后9位为权限，数字为硬链接计数，alice为所有者，staff为所属组，1024为文件大小（字节），Jan 1 10:00为最后修改时间，file.txt为文件名。ls -i可同时显示inode编号。

**48. find**find是强大的文件搜索命令，支持按名称、类型、大小、时间、权限、所有者等条件查找文件。基本语法：find \[路径] \[条件] \[动作]。例如，find /home -name "\*.txt" 查找所有.txt文件；find /var -size +100M 查找大于100MB的文件；find . -mtime -7 查找7天内修改过的文件。find还支持-exec参数对找到的文件执行命令，是系统管理和脚本编写的重要工具。

**49. xargs**xargs用于将标准输入数据转换成命令行参数，常与管道和find配合使用，解决“参数列表过长”问题。例如，`find /tmp -name "*.log" | xargs rm -f`批量删除日志文件。xargs支持`-n`（分组数量）、`-P`（并行进程数）等高级选项，是Shell批处理任务中不可或缺的性能利器。

**50. sort / uniq**sort用于对文本行进行排序（`-n`数字排序，`-r`降序，`-k`指定字段）；uniq用于报告或忽略重复行（通常与sort联用，如`sort file | uniq -c`统计重复次数）。两者在日志分析（如统计IP访问频率`awk '{print $1}' access.log | sort | uniq -c | sort -nr`）中频繁出场，是文本统计的黄金组合。

## 四、进程与作业管理（新增至14个）

**51. 进程（Process）**&#x8FDB;程是程序在计算机上的一次执行活动，是系统进行资源分配和调度的基本单位。每个进程有独立的地址空间、文件描述符、环境变量等资源。进程由内核创建，通过fork()系统调用复制父进程，再通过exec()加载新程序。进程拥有唯一PID（进程ID），所有进程形成树状结构，根为init（PID=1）。进程是动态的，有生命周期：创建、运行、等待、终止。

**52. 守护进程（Daemon）**&#x5B88;护进程是后台持续运行的服务进程，通常以d结尾（如sshd、httpd、crond）。它们脱离终端控制，在系统启动时自动启动，并在后台等待处理请求。守护进程通过fork()创建子进程后，父进程退出，子进程调用setsid()创建新会话，从而与终端完全脱离。守护进程是Linux服务架构的基础，常见于网络服务、系统监控、定时任务等。

**53. PID**PID（进程标识符）是系统为每个进程分配的唯一正整数，用于区分不同进程。PID从1开始（init/systemd），最大值为/proc/sys/kernel/pid\_max（通常32768）。当进程终止后，其PID可被新进程重用，但系统会避免立即重用。PID是进程管理的基础，通过PID可以使用kill、renice、wait等命令操作特定进程。

**54. PPID**PPID（父进程ID）是创建当前进程的父进程的PID。每个进程都有父进程（除了init），通过ps -ef或pstree命令可以查看进程树。PPID在进程创建时由内核设置，用于追踪进程层级关系。当父进程先于子进程终止时，子进程成为孤儿进程，被init收养，其PPID变为1。

**55. 僵尸进程（Zombie）**&#x50F5;尸进程是已经终止但未被父进程回收的进程，其进程描述符仍保留在进程表中，占用一个PID。僵尸进程不占用CPU和内存，但会消耗进程表项（系统资源有限）。如果父进程没有调用wait()或waitpid()来获取子进程的退出状态，子进程就会变成僵尸。父进程终止后，僵尸进程会被init收养并自动清理。

**56. 孤儿进程**孤儿进程是指父进程先于子进程终止的进程。此时子进程成为孤儿，被init（PID=1）收养，init会定期调用wait()回收它们。孤儿进程不会变成僵尸，因为init会自动处理。孤儿进程在后台运行时不会产生问题，但如果父进程意外崩溃，子进程可能失去控制，需要系统管理员注意。

**57. ps**ps（process status）用于查看当前系统的进程快照。常用选项：ps aux 显示所有进程的详细信息（包括用户、CPU/内存使用率、状态、启动时间等）；ps -ef 显示标准格式；ps -eo pid,ppid,cmd 自定义输出列。ps从/proc文件系统读取进程信息，是系统监控和故障排查的基础工具。

**58. top/htop**top是动态实时监控进程资源占用的工具，默认按CPU使用率排序，显示进程ID、用户、CPU%、内存%、运行时间、命令等。按P键按CPU排序，按M键按内存排序，按k键可终止进程。htop是top的增强版，支持彩色显示、鼠标操作、树状视图、垂直/水平滚动，更直观易用。两者都依赖/proc文件系统，是性能分析的首选工具。

**59. kill**kill用于向进程发送信号，默认发送SIGTERM（15），请求进程优雅终止。常用信号：kill -9 PID 发送SIGKILL强制终止；kill -15 PID 发送SIGTERM；kill -1 PID 发送SIGHUP（让守护进程重新加载配置）。kill通过PID指定目标进程，如果进程不存在会报错。killall命令可通过进程名发送信号。

**60. nice/renice**nice用于以指定优先级启动程序，renice用于调整已运行进程的优先级。优先级范围从-20（最高）到19（最低），默认值为0。只有root可以设置负优先级（提高优先级）。语法：nice -n 10 command 以较低优先级运行；renice -n 5 -p PID 将进程优先级调整为5。调整优先级可以控制CPU资源分配，确保重要任务获得更多CPU时间。

**61. 作业控制（Job Control）**&#x4F5C;业控制是Shell对前台和后台进程组的管理机制。`command &` 将命令放入后台运行；`Ctrl+Z` 挂起当前前台作业；`jobs` 查看作业列表；`fg %1` 将作业1切回前台；`bg %1` 让作业1在后台继续执行。作业控制允许用户在一个终端内灵活切换多个任务，是交互式Shell高效使用的必备技能。

**62. nohup 与终端复用（screen/tmux）**`nohup command &` 使进程忽略SIGHUP信号，即使退出终端，进程仍继续运行（输出重定向到nohup.out）。更强大的方案是**终端复用器**——`tmux`和`screen`，它们创建持久会话，网络断开后重连即可恢复会话，且支持分屏多窗口。在远程服务器开发、长期运行数据迁移任务中，tmux是事实上的标准工具。

**63. iostat / iotop**`iostat`（来自sysstat包）报告CPU利用率和磁盘I/O统计信息（如`iostat -x 1`查看每秒的读写速率、`await`等待时间、`util`磁盘繁忙度）。`iotop`类似于`top`，但专门按I/O使用率排序进程。两者是定位“磁盘慢、应用卡顿”问题的核心工具，能快速识别是哪个进程在疯狂读写磁盘导致系统负载过高。

**64. 平均负载（Load Average）**&#x7CFB;统平均负载是单位时间内处于可运行状态（R状态）和不可中断睡眠状态（D状态，通常为等待I/O）的进程平均数。`top`或`uptime`显示1分钟、5分钟、15分钟三个值。理想情况下，负载值应小于CPU核心数。若负载长期高于核心数，通常意味着CPU资源不足或磁盘I/O瓶颈，是性能告警的首要关注指标。

## 五、信号与进程间通信（8个，保持原样，新增信号trap概念移植至Shell章节）

**65. 信号（Signal）**&#x4FE1;号是软件中断，用于通知进程发生了异步事件。信号由内核或另一个进程发送，进程可以捕获、忽略或执行默认操作。Linux支持多种信号，如SIGINT（Ctrl+C）、SIGTERM（终止请求）、SIGKILL（强制终止）、SIGCHLD（子进程状态变化）等。信号是进程间通信的一种简单方式，常用于进程控制、错误通知和定时器。

**66. SIGKILL (9)**&#x53;IGKILL是强制终止信号，进程无法捕获、忽略或阻塞它。收到SIGKILL后，内核立即终止进程并释放其资源，不给进程任何清理机会。因此，SIGKILL应作为最后手段使用，仅当进程无响应且SIGTERM无效时使用。使用kill -9 PID发送。注意：SIGKILL不能杀死僵尸进程（僵尸已死，只是未回收）。

**67. SIGTERM (15)**&#x53;IGTERM是优雅终止信号，请求进程自行退出。进程可以捕获SIGTERM并执行清理操作（如保存数据、关闭文件、释放资源），然后退出。如果进程忽略SIGTERM，则不会终止。大多数守护进程和应用程序会注册SIGTERM处理函数，实现平滑关闭。使用kill PID（默认发送SIGTERM）或kill -15 PID。

**68. SIGINT (2)**&#x53;IGINT是中断信号，通常由用户按下Ctrl+C触发。前台进程收到SIGINT后，默认行为是终止。进程可以捕获SIGINT并执行自定义操作（如提示确认退出）。SIGINT与SIGTERM类似，但通常用于用户主动中断，而SIGTERM用于系统或管理员请求终止。

**69. SIGCHLD**SIGCHLD是子进程状态变化时（终止、停止、继续）发送给父进程的信号。父进程可以通过捕获SIGCHLD来异步回收子进程，避免子进程变成僵尸。如果父进程不处理SIGCHLD，子进程终止后父进程需要显式调用wait()来回收。现代程序常使用SIGCHLD结合waitpid()实现高效的子进程管理。

**70. 管道（Pipe）**&#x7BA1;道是Unix/Linux中最基本的进程间通信方式，将一个命令的标准输出连接到另一个命令的标准输入，用竖线“|”表示。例如，ls -l | grep "txt" 将ls的输出作为grep的输入。管道是单向的、无名的，只能在有亲缘关系的进程（如父子进程）之间使用。管道本质上是内核中的一个缓冲区，默认大小通常为64KB。

**71. 命名管道（FIFO）**&#x547D;名管道（FIFO）是一种特殊的文件类型，通过mkfifo命令创建。与普通管道不同，FIFO有文件名，允许无亲缘关系的进程通过它通信。FIFO遵循先进先出原则，数据以流的形式传输。常用于客户端-服务器模型，例如一个进程写入FIFO，另一个进程读取。FIFO在文件系统中可见，使用ls -l查看时类型为p。

**72. 信号量（Semaphore）**&#x4FE1;号量是用于同步多个进程对共享资源访问的计数器。它支持两种原子操作：P（等待，减少计数）和V（释放，增加计数）。当计数为0时，P操作会阻塞进程直到其他进程执行V操作。信号量可以解决互斥和同步问题，例如控制对共享内存的访问。Linux提供System V信号量和POSIX信号量两种实现。

## 六、用户与组管理（8个，保持原样）

**73. 用户（User）**&#x7528;户是系统资源的访问者，每个用户有唯一的用户名和UID（用户标识符）。用户信息存储在/etc/passwd文件中，包括用户名、加密密码占位符（x）、UID、GID、用户描述、家目录、登录Shell。用户通过登录认证后，获得相应的权限。Linux是多用户系统，支持同时多个用户登录，每个用户拥有独立的家目录和环境。

**74. 组（Group）**&#x7EC4;是用户的集合，用于简化权限管理。每个组有唯一的组名和GID（组标识符）。组信息存储在/etc/group文件中。一个用户可以属于多个组，其中一个是主组（在/etc/passwd中指定），其他为附加组。通过将用户加入特定组，可以批量授予文件访问权限，例如将开发人员加入dev组，然后设置项目目录的组权限为读写。

**75. /etc/passwd**/etc/passwd是系统用户账户信息文件，每行代表一个用户，格式为：用户名:密码占位符:UID:GID:描述:家目录:Shell。例如，alice:x:1000:1000:Alice:/home/alice:/bin/bash。密码字段通常为x，表示密码存储在/etc/shadow中。该文件对所有用户可读，因此不能存储明文密码。修改用户信息应使用usermod、useradd等专用命令。

**76. /etc/shadow**/etc/shadow存储加密后的用户密码和密码策略信息，仅root可读。每行格式为：用户名:加密密码:最后修改日:最小天数:最大天数:警告天数:禁用天数:过期日:保留字段。加密密码使用SHA-512（$6$）等算法。密码策略包括密码有效期、过期警告、账户锁定等，通过chage命令管理。shadow文件的存在提高了密码安全性。

**77. /etc/group**/etc/group存储组信息，每行格式为：组名:密码占位符:GID:成员列表。例如，staff:x:100:alice,bob。密码字段通常为空（x），组密码很少使用。成员列表是用逗号分隔的用户名，表示该组的附加成员。主组成员不在/etc/group中列出，而是通过/etc/passwd中的GID确定。

**78. useradd**useradd用于创建新用户，常用选项：-m 创建家目录，-s 指定Shell，-G 指定附加组，-u 指定UID。例如，useradd -m -s /bin/bash -G sudo alice 创建用户alice，家目录为/home/alice，Shell为bash，加入sudo组。useradd还会在/etc/shadow中创建密码条目（需用passwd设密码）。不同发行版默认行为略有差异。

**79. passwd**passwd用于设置或修改用户密码。普通用户只能修改自己的密码（需输入旧密码），root可以修改任何用户的密码。passwd会加密新密码并更新/etc/shadow。密码强度由系统策略（如pam\_cracklib）控制，要求包含大小写字母、数字和特殊字符，长度至少8位。使用passwd -l可锁定账户，-u解锁。

**80. usermod**usermod用于修改已有用户的属性，如用户名（-l）、家目录（-d）、Shell（-s）、UID（-u）、附加组（-aG）等。例如，usermod -aG docker alice 将用户添加到docker组（-a表示追加，防止覆盖现有组）。usermod -L 锁定账户（在密码前加!），-U解锁。修改用户信息后，用户需重新登录才能生效。

## 七、网络与远程访问（新增至14个）

**81. IP地址**IP地址是网络设备的唯一标识，分为IPv4（32位，如192.168.1.1）和IPv6（128位，如fe80::1）。Linux中通过ifconfig或ip addr查看IP地址。IP地址与子网掩码配合使用，确定网络地址和主机地址。IP地址是网络通信的基础，每个网络接口（如eth0、wlan0）可以配置一个或多个IP地址。

**82. SSH**SSH（Secure Shell）是加密的远程登录协议，默认端口22。它使用公钥加密技术，确保数据传输的机密性和完整性。SSH客户端（ssh命令）连接到SSH服务器（sshd守护进程），支持密码认证和密钥认证。SSH还支持端口转发、X11转发、文件传输（通过SCP、SFTP）等功能，是Linux系统管理的必备工具。

**83. SCP**SCP（Secure Copy）基于SSH协议，用于在本地和远程主机之间安全复制文件。语法：scp source destination，例如scp file.txt user\@host:/path/。SCP支持递归复制目录（-r）、保留文件属性（-p）等选项。与FTP不同，SCP全程加密，适合传输敏感数据。

**84. rsync（远程增量同步）**&#x72;sync是比SCP更强大的文件传输和同步工具，核心优势是**增量传输**（仅传输差异部分）和**断点续传**。语法：`rsync -avz --progress /local/dir user@host:/remote/dir`。它支持本地复制、远程Shell（SSH）传输、守护进程模式，并能保留权限、属主、时间戳等所有属性。在生产环境中，rsync是日志备份、代码分发、数据迁移的绝对主力工具。

**85. ping**ping用于测试网络连通性，通过发送ICMP回显请求并等待响应。ping hostname 会持续发送数据包，显示响应时间和丢包率。ping -c 4 hostname 发送4个包后停止。ping是网络故障排查的第一步，可以判断目标主机是否可达、网络延迟是否正常。注意：某些防火墙会屏蔽ICMP，导致ping不通但其他服务正常。

**86. netstat/ss**netstat和ss用于查看网络连接、路由表、接口统计等。ss是netstat的现代替代，性能更好。常用选项：ss -tuln 显示所有监听端口（t TCP，u UDP，l监听，n数字格式）；ss -an 显示所有连接。通过ss可以查看哪些服务在监听哪些端口，以及当前建立的连接状态（ESTABLISHED、TIME\_WAIT等），是网络调试的重要工具。

**87. tcpdump**tcpdump是命令行网络抓包工具，用于捕获和分析网络流量。语法：tcpdump -i eth0 host 192.168.1.1 抓取eth0接口上与192.168.1.1通信的包。支持过滤表达式（如port 80、tcp、udp），可保存到文件（-w）并用Wireshark分析。tcpdump是网络故障排查、协议分析和安全审计的利器，但需要root权限。

**88. iptables/nftables**iptables是Linux内核的包过滤防火墙工具，通过规则表（filter、nat、mangle）控制网络流量。nftables是iptables的现代替代，语法更简洁，性能更好。防火墙规则可以允许、拒绝或修改数据包，实现访问控制、端口转发、NAT等功能。例如，iptables -A INPUT -p tcp --dport 22 -j ACCEPT 允许SSH连接。防火墙配置通常保存为脚本或使用firewalld管理。

**89. DNS**DNS（域名系统）将域名（如www\.example.com）解析为IP地址。Linux系统通过/etc/resolv.conf配置DNS服务器（如8.8.8.8）。常用DNS工具：nslookup、dig、host。DNS解析是网络通信的前提，如果DNS配置错误，域名将无法访问。

**90. systemd-resolved**现代Linux发行版（如Ubuntu、Fedora）使用systemd-resolved管理DNS解析，它通过D-Bus接口提供域名解析，并缓存结果以加速访问。`resolvectl status`查看当前DNS配置，`resolvectl query example.com`测试解析。systemd-resolved监听127.0.0.53，需注意它可能覆盖`/etc/resolv.conf`，新手配置DNS时常在此处踩坑。

**91. DHCP**DHCP（动态主机配置协议）自动为网络设备分配IP地址、子网掩码、网关、DNS等参数。Linux客户端通过dhclient或NetworkManager获取IP。DHCP服务器（如isc-dhcp-server）管理地址池，避免IP地址冲突。DHCP简化了网络配置，特别适合移动设备和大型网络。静态IP则需手动配置/etc/network/interfaces或使用nmcli。

**92. NetworkManager 与 nmcli**NetworkManager是Linux系统的动态网络控制与管理守护进程，不仅管理有线/Wi-Fi网络，还支持VPN、蓝牙共享等。`nmcli`是其命令行工具：`nmcli dev status`查看设备状态，`nmcli con add type ethernet ifname eth0 con-name static ip4 192.168.1.10/24 gw4 192.168.1.1`创建静态连接。无论是在桌面环境还是服务器（RHEL 8+默认启用），nmcli都是现代网络配置的首选工具。

**93. curl/wget**curl和wget是命令行HTTP/HTTPS请求工具。curl功能更丰富，支持多种协议（HTTP、FTP、SMTP等）、自定义请求头、Cookie、认证等。例如，curl -O [https://example.com/file.zip](https://example.com/file.zip) 下载文件。wget更专注于下载，支持递归下载、断点续传。两者都是自动化脚本和API测试的常用工具。

## 八、Shell脚本与文本处理（新增至14个）

**94. Bash**Bash（Bourne Again Shell）是Linux默认的Shell，兼容sh，并扩展了命令行编辑、作业控制、数组、算术运算等功能。Bash脚本以#!/bin/bash开头，支持变量、条件判断（if）、循环（for、while）、函数、输入输出重定向等。Bash是系统管理自动化的核心语言，通过编写脚本可以批量处理文件、监控系统、部署应用。

**95. 环境变量**环境变量是影响进程行为的动态命名值，通过export命令设置。常见环境变量：PATH（可执行文件搜索路径）、HOME（用户家目录）、USER（当前用户名）、LANG（语言环境）、SHELL（当前Shell）。环境变量在进程创建时从父进程继承，子进程可以修改自己的环境变量而不影响父进程。配置文件如.bashrc、.profile用于设置用户环境。

**96. PATH**PATH环境变量定义了Shell查找可执行文件的目录列表，各目录用冒号分隔。例如，PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin。当用户输入命令时，Shell按顺序在PATH目录中查找同名可执行文件。如果命令不在PATH中，需使用绝对路径或相对路径。修改PATH（如export PATH=\$PATH:/opt/bin）可以扩展命令搜索范围。

**97. grep**grep是强大的文本搜索工具，根据正则表达式匹配文件内容。基本语法：grep pattern file。常用选项：-i 忽略大小写，-r 递归搜索目录，-n 显示行号，-v 反向匹配，-c 统计匹配行数，-E 支持扩展正则。例如，grep -rn "error" /var/log/ 在日志中递归搜索error。grep常与管道结合，用于过滤命令输出。

**98. sed**sed（流编辑器）用于对文本进行非交互式编辑，支持替换、删除、插入、打印等操作。基本语法：sed 's/old/new/g' file。常用命令：s（替换）、d（删除行）、a（追加）、i（插入）、p（打印）。sed逐行处理文本，默认输出到标准输出，使用-i选项直接修改文件。sed是文本处理的重要工具，常用于批量替换配置文件中的参数。

**99. awk**awk是一种模式扫描与处理语言，擅长格式化文本报告。基本语法：awk 'pattern \{action}' file。awk自动将每行分割为字段，$1、$2等表示第1、2个字段。例如，awk '\{print $1,$3}' file 打印每行的第1和第3列。awk支持变量、数组、算术运算、控制流，可以编写复杂的数据处理脚本。awk与grep、sed并称文本处理三剑客。

**100. 正则表达式**正则表达式是用于模式匹配的字符串规则，被grep、sed、awk、vim等工具支持。基本元字符：. 匹配任意字符，\* 匹配前一个字符零次或多次，^ 匹配行首，\$ 匹配行尾，\[] 匹配字符集，\ 转义。扩展正则（ERE）支持+、?、|、()等。正则表达式是文本处理的基石，掌握正则可以高效地搜索、替换和提取文本信息。

**101. 脚本调试（set -x / -e / -u）**&#x53;hell脚本调试通过`bash -x script.sh`或脚本内`set -x`实现，显示每条命令及其参数（前面加+号）。`set -e`使脚本在遇到非零返回码时立即退出（防止错误蔓延），`set -u`将未定义变量视为错误并终止。`trap`命令可以捕获信号（如`EXIT`、`ERR`）执行清理操作。良好的调试习惯能快速定位脚本中的逻辑错误。

**102. printf**printf是比echo更强大的格式化输出命令，继承自C语言。它支持格式说明符（`%s`字符串、`%d`整数、`%f`浮点数）和转义序列（`\n`换行、`\t`制表符）。例如`printf "%-10s %5d\n" "Alice" 100`输出左对齐字符串和右对齐数字。在生成报表、填充表格时，printf是awk之外最精准的输出利器。

**103. 重定向与文件描述符**Linux中每个进程默认有三个文件描述符：0（标准输入）、1（标准输出）、2（标准错误）。重定向操作：`>`覆盖标准输出，`>>`追加，`2>`重定向标准错误，`&>`同时重定向两者。`2>&1`将标准错误合并到标准输出。高级技巧：`exec 3>file`创建自定义描述符，`read -u 3`读取。重定向是Shell脚本与文件交互最基础的机制。

**104. 进程替换（\<() / >()）**&#x8FDB;程替换是Bash的高级特性，允许将命令的输出作为文件参数传递给其他命令。`diff <(ls dir1) <(ls dir2)`比较两个目录的列表，而无需创建临时文件。`grep error <(tail -f app.log)`实时过滤日志。进程替换比管道更灵活，常用于对比、组合多个命令的输出流。

**105. jq（JSON处理利器）**&#x968F;着API和微服务普及，JSON成为主流数据格式。`jq`是命令行下的JSON处理器：`curl -s api.example.com | jq '.data[0].id'`提取特定字段，`jq '. | map(select(.age > 18))'`过滤数组。jq支持复杂的切片、映射、拼接操作，是云原生架构中排查接口返回、处理Kubernetes资源清单的核心辅助工具。

## 九、系统服务与日志（新增至10个）

**106. systemd**systemd是Linux系统的服务管理器，取代了传统的SysV init。它使用单元（unit）文件管理服务、挂载点、套接字、定时器等。systemd支持并行启动、按需启动、依赖管理、自动重启等特性，显著加快系统启动速度。systemd还集成了日志系统（journald）、时间同步（timesyncd）、网络管理（networkd）等功能，成为现代Linux发行版的标准组件。

**107. systemctl**systemctl是systemd的控制命令，用于管理服务单元。常用操作：systemctl start nginx 启动服务；systemctl stop nginx 停止；systemctl restart nginx 重启；systemctl enable nginx 设置开机自启；systemctl status nginx 查看服务状态。systemctl还支持mask（屏蔽服务）、daemon-reload（重载配置）等高级功能。

**108. systemd Targets（运行目标）**&#x54;arget是systemd中一组单元的集合，用于定义系统的运行状态，类似于SysV的“运行级别”（Runlevel）。常见Target：`multi-user.target`（多用户命令行模式）、`graphical.target`（图形界面）、`rescue.target`（单用户维护模式）。通过`systemctl get-default`查看默认启动目标，`systemctl set-default multi-user.target`切换。Target之间存在依赖链，实现服务的分组启动控制。

**109. journalctl**journalctl用于查询systemd日志（journald），支持按时间、服务、优先级等过滤。例如，journalctl -u nginx 查看nginx服务的日志；journalctl --since "1 hour ago" 查看最近一小时的日志；journalctl -f 实时跟踪新日志。journalctl将日志存储在二进制文件中，支持结构化查询，比传统文本日志更高效。

**110. cron**cron是Linux的定时任务调度器，通过crontab文件配置。crontab格式：分 时 日 月 周 命令。例如，0 2 \* \* \* /usr/bin/backup.sh 每天凌晨2点执行备份。cron服务（crond）每分钟检查一次crontab，执行到期的任务。用户可以使用crontab -e编辑自己的任务，root可以管理所有用户的任务。cron是系统自动化的基础，常用于日志轮转、数据备份、系统更新。

**111. systemd Timers（定时器）**&#x73;ystemd定时器是cron的现代替代方案，通过`.timer`单元和对应的`.service`单元配合工作。优势：精确到微秒、支持日历事件和单调计时、可设置依赖链、日志集成到journald。示例：`systemd-analyze calendar "*-*-* 02:00:00"`测试触发时间，`systemctl enable backup.timer`启用。在RHEL/CentOS 7+中，systemd定时器正逐步成为官方推荐的任务调度方式。

**112. logrotate**logrotate用于管理日志文件，防止日志无限增长占用磁盘空间。它根据配置自动轮转、压缩、删除旧日志。配置文件在/etc/logrotate.conf和/etc/logrotate.d/中。例如，/var/log/syslog \{ weekly rotate 4 compress } 表示每周轮转一次，保留4个备份，并压缩。logrotate通常由cron定时执行，是系统日志管理的重要工具。

**113. syslog/rsyslog**syslog是系统日志记录框架，rsyslog是其增强版，支持多源日志收集、过滤、转发。日志消息根据设施（facility）和优先级（priority）分类，写入/var/log/下的文件（如messages、auth.log）。rsyslog支持TCP/UDP传输、数据库存储、模板格式化等高级功能。日志是系统故障排查和安全审计的重要依据。

**114. 服务单元文件（Service Unit）**&#x53;ystemd的服务单元文件（`.service`）位于`/etc/systemd/system/`或`/usr/lib/systemd/system/`，采用INI格式。核心段：`[Unit]`（描述、依赖）、`[Service]`（ExecStart、ExecStop、Restart、User）、`[Install]`（WantedBy）。编写自定义服务单元是将任何应用程序（如Python Web应用、Java Jar包）托管为系统守护进程的标准做法，便于统一管理和监控。

**115. 系统启动时间分析（systemd-analyze）**`systemd-analyze`是分析系统启动性能的工具：`systemd-analyze time`显示内核和用户空间启动耗时；`systemd-analyze blame`按耗时的降序列出每个服务单元的启动时间；`systemd-analyze critical-chain`显示关键路径上的依赖链。此工具是排查服务器重启缓慢、优化启动速度的利器，能精准定位哪些服务拖慢了系统。

## 十、软件包管理（新增至8个）

**116. 包管理器**包管理器是自动化安装、更新、卸载软件的工具，解决软件依赖关系。Linux发行版通常使用两种包格式：Debian系（.deb）使用APT，Red Hat系（.rpm）使用YUM/DNF。包管理器从软件仓库（repository）下载预编译的二进制包，自动处理依赖，确保系统一致性。包管理器的出现大大简化了软件安装过程，是Linux生态繁荣的重要基础。

**117. APT**APT（Advanced Package Tool）是Debian/Ubuntu的包管理工具。常用命令：apt update 更新软件源列表；apt install package 安装包；apt remove package 卸载包；apt upgrade 升级所有可升级的包；apt search keyword 搜索包。APT自动处理依赖关系，并将包信息缓存到本地。现代APT还支持apt list、apt show等查询命令。

**118. YUM/DNF**YUM（Yellowdog Updater Modified）和DNF（Dandified YUM）是Red Hat/CentOS/Fedora的包管理器。DNF是YUM的下一代，性能更好，依赖解析更准确。常用命令：dnf install package；dnf remove package；dnf update；dnf search keyword。DNF使用RPM作为底层包格式，支持插件扩展（如自动镜像选择、增量更新）。

**119. RPM/DEB**RPM（Red Hat Package Manager）和DEB（Debian Package）是两种主流的软件包格式。RPM用于Red Hat系（CentOS、Fedora），DEB用于Debian系（Ubuntu、Debian）。包文件包含二进制程序、配置文件、文档和元数据（版本、依赖、描述）。直接使用rpm -ivh package.rpm或dpkg -i package.deb可以安装，但不处理依赖，因此通常通过包管理器使用。

**120. 仓库（Repository）**&#x8F6F;件仓库是存储预编译包的服务器，通过配置文件（如/etc/apt/sources.list）指定。仓库分为官方仓库（如Ubuntu的main、universe）和第三方仓库（如PPA、EPEL）。包管理器从仓库下载包并验证签名，确保软件来源可靠。添加第三方仓库可以获取官方未提供的软件，但需注意安全风险。仓库机制是Linux软件分发的主要方式。

**121. 模块流（AppStream）**&#x52;HEL/CentOS 8+引入了AppStream模块流机制，允许用户在同一发行版上选择不同版本的软件包（如PHP 7.4或PHP 8.0）。`dnf module list`查看可用模块，`dnf module enable php:8.0`启用特定流，`dnf install php`安装。模块流解决了传统RPM发行版“版本固定”的痛点，使企业级系统既能保持稳定内核，又能灵活使用新版应用软件。

## 十一、安全与审计（新增至6个）

**122. 防火墙（Firewall）**&#x9632;火墙是监控和控制网络流量的安全系统，基于规则允许或拒绝数据包。Linux防火墙通过iptables/nftables实现，也可使用firewalld（动态防火墙管理工具）简化配置。防火墙规则可以基于IP地址、端口、协议、状态等条件。例如，firewall-cmd --add-service=http --permanent 开放HTTP服务。防火墙是系统安全的第一道防线。

**123. fail2ban**fail2ban是入侵防御工具，通过监控日志文件（如/var/log/auth.log）检测多次认证失败，然后使用iptables临时封禁攻击IP。它支持多种服务（SSH、Apache、Postfix等），可自定义封禁时间和重试次数。fail2ban能有效防止暴力破解攻击，是服务器安全加固的常用工具。配置在/etc/fail2ban/jail.conf中。

**124. SELinux**SELinux（安全增强型Linux）是强制访问控制（MAC）机制，由美国国家安全局开发。它通过安全策略定义进程可以访问哪些文件、端口、资源，即使进程被攻破，也无法越权操作。SELinux有三种模式：Enforcing（强制）、Permissive（仅记录）、Disabled（禁用）。配置复杂，但能提供极高的安全性，常用于政府、军事等高安全环境。

**125. AppArmor**AppArmor是基于路径的强制访问控制，与SELinux互补。它通过配置文件（profile）限制程序可以访问的文件、网络、能力。例如，/usr/bin/evince的profile允许它读取PDF文件但禁止访问网络。AppArmor比SELinux更易配置，适合桌面和通用服务器。Ubuntu默认使用AppArmor，而CentOS使用SELinux。

**126. 审计（Audit）**&#x4C;inux审计系统（auditd）用于记录系统安全相关事件，如文件访问、系统调用、用户登录等。审计规则通过auditctl命令配置，日志写入/var/log/audit/audit.log。审计可以检测入侵行为、追踪用户操作、满足合规要求。例如，auditctl -w /etc/passwd -p wa -k passwd\_changes 监控passwd文件的写和属性修改。审计日志需定期审查，结合ausearch工具分析。

**127. 加密与哈希**Linux中密码存储使用SHA-512（影子文件），文件完整性校验常用`md5sum`、`sha256sum`。GPG（GNU Privacy Guard）用于非对称加密和签名验证，常用于软件包签名（如APT源验证）。OpenSSL是SSL/TLS协议的实现，提供对称/非对称加密、证书管理功能。在生产环境中，配置HTTPS、加密数据传输、校验软件包完整性均依赖这些工具。

## 十二、容器化与云原生基石（全新章节，共6个）

**128. OverlayFS（联合文件系统）**&#x4F;verlayFS是Linux内核原生的联合挂载文件系统，将多个目录（lowerdir、upperdir）合并成一个统一的视图。它是**Docker/Podman容器镜像分层存储**的底层技术：只读层作为lowerdir，容器可写层作为upperdir，写时复制（CoW）机制保证容器修改不影响镜像层。`docker inspect`查看GraphDriver即OverlayFS。理解OverlayFS是排查容器磁盘占用和文件丢失问题的基础。

**129. 容器网络模型（CNI与veth）**&#x5BB9;器使用**veth pair**（虚拟以太网对）连接宿主机网络桥接（如docker0、cni0），一端在容器网络命名空间内，另一端在宿主机上。**CNI（容器网络接口）**&#x662F;容器运行时与网络插件交互的标准，实现IP分配、路由配置。常用插件包括bridge、macvlan、flannel、Calico。容器的网络隔离与通信本质上依赖Linux网桥、iptables/nftables规则和路由表转发。

**130. Podman（无守护进程容器引擎）**&#x50;odman是Red Hat主导的容器引擎，与Docker CLI完全兼容（可别名`alias docker=podman`）。最大区别是**无守护进程**（无dockerd），采用fork/exec模型，容器直接作为子进程运行，支持Rootless模式（非root用户运行容器），安全性更高。在RHEL/CentOS 8+中，Podman是默认容器工具，与systemd集成良好（通过`podman generate systemd`生成服务单元）。

**131. 容器运行时（runC与CRI-O）**`runC`是OCI（开放容器标准）的参考实现，负责创建和运行容器（底层调用namespaces和cgroups）。Docker和Podman都依赖runC。**CRI-O**是Kubernetes CRI（容器运行时接口）的轻量级实现，直接兼容OCI镜像，专为K8s设计，替代docker-shim。了解容器运行时的层级（CLI -> 守护进程/管理器 -> runC -> 内核），有助于诊断“容器无法启动”或“资源限制不生效”等深层次问题。

**132. 容器镜像构建（Dockerfile与多阶段构建）**&#x44;ockerfile是定义容器镜像构建步骤的文本文件，核心指令包括`FROM`（基础镜像）、`RUN`（执行命令）、`COPY`/`ADD`（添加文件）、`EXPOSE`（声明端口）、`CMD`/`ENTRYPOINT`（启动命令）。**多阶段构建**允许在一个Dockerfile中使用多个`FROM`，将构建环境与运行环境分离，大幅缩减最终镜像体积（如将Golang编译环境与Alpine运行环境分离），是云原生微服务镜像瘦身的核心技巧。

**133. Kubernetes 基本概念（Pod、Service、Ingress）**&#x867D;然K8s是编排层，但底层依赖Linux内核特性。**Pod**是最小调度单元，包含一个或多个共享网络命名空间（同IP、同IPC）的容器；**Service**通过标签选择器提供稳定的ClusterIP和负载均衡；**Ingress**管理外部HTTP/HTTPS流量路由。理解Pod的网络共享（依赖容器间命名空间共享）和服务发现（依赖DNS和iptables/IPVS），能帮助运维精准定位服务连接超时、网络策略冲突等云原生故障。

### 总结与进阶建议

以上 **133 个概念** 涵盖了从底层**内核机制（Cgroup/Namespace/OverlayFS）**、**企业存储（LVM/RAID/XFS/Btrfs）**、**系统调优（sysctl/swap）** 到**自动化管理（systemd/rsync/jq）** 以及**云原生基石（容器/K8s基础）** 的完整知识链路。

随着 2026 年技术演进，Linux 已不仅仅是一个操作系统，更是云原生、边缘计算和 AI 基础设施的底座。建议读者：

1. **实践为王**：在虚拟机或云服务器上亲手执行 `lvm` 扩容、编写 `systemd` 服务单元、配置 `rsync` 定时备份。
2. **追踪内核**：关注 Linux 内核 6.x 在 BPF（eBPF）、新调度器和安全模块上的创新。
3. **融入生态**：将学到的 `cgroups`、`namespace` 知识与 Kubernetes 调度、资源配额（ResourceQuota）相互印证，达到融会贯通。

这份万字长文不仅是一份备考手册，更是一张清晰的企业级 Linux 技术地图。
# 后记：给即将开始学习 Linux 的你——前方是星辰大海

现在，你翻过了这本书的最后一页。可能你正热血沸腾，恨不得立刻打开终端大干一场；也可能你看着那些命令和报错，心里打起了退堂鼓——**"我真的能学会吗？"**

我想告诉你一个秘密：2026 年的今天，你学习 Linux，赶上的不是一个"系统管理员"的时代，而是一个**"AI 智能体正在重塑世界"**的时代。

## 你学的不是命令，是 AI 时代的"底层语法"

你可能没意识到，你正在打开一扇通往未来的大门。想象一下：

- 你写的 Python 脚本，正在某个 Kubernetes Pod 里——**跑在 Linux 上**；
- 你调教的 AI Agent 正在自主规划任务、调用工具、执行代码——**跑在 Linux 上**；
- 你搭建的 RAG 知识库，在向量数据库里检索、推理、生成答案——**跑在 Linux 上**；
- 你开发的智能体协作平台，几百个 AI 进程在并行决策——**跑在 Linux 上**。

你没有在学一堆"过时的命令"。你正在学习的，是**下一代 AI 基础设施的底层语法**。那些 `systemd`、`cgroups`、`namespace`、`eBPF`——它们正是支撑 AI 智能体大规模部署、隔离、调度和观测的**核心引擎**。

## 当 AI 开始"自主行动"，Linux 是它脚下的土地

2026 年是 AI Agents 从"对话玩具"变成"数字员工"的关键转折点。它们不再只是回答问题，而是开始：

- **执行任务**：自主调用 API、操作数据库、发送邮件、部署服务；
- **协作分工**：多个智能体像一支团队，分工协作完成复杂项目；
- **持续运行**：24/7 不间断地监控、决策、优化、迭代。

而所有这些能力，都建立在同一个地基上——**Linux 内核**。

当你学会了用 `systemd` 管理服务，你就是在学习如何让 AI 智能体"开机自启、永不掉线"；当你理解了 `cgroups` 的资源限制，你就是在学习如何让 100 个 AI 智能体"和平共处、互不抢食"；当你掌握了 `journalctl` 日志查询，你就是在学习如何让 AI 的每一次决策"有迹可循、可审计、可回溯"。

**你不是在学一个操作系统，你是在学如何让智能体在数字世界里站稳脚跟。**

## 为什么现在是最好的时候？

十年前学习 Linux，你面对的是物理服务器、机柜、网线和 UPS。你学会了，能维护一台机器。

五年前学习 Linux，你面对的是云主机、容器编排和微服务。你学会了，能管理一个集群。

2026 年学习 Linux，你面对的是 AI 智能体、大模型推理和自主决策系统。你学会了，能**指挥一支数字军团**。

**门槛从来没有像今天这么低，天花板从来没有像今天这么高。**

你不需要是内核开发者，也能用 `bpftrace` 洞察智能体的每一次系统调用；你不需要是分布式系统专家，也能用 `systemd` 和 `podman` 让 AI 服务在生产环境稳定运行。工具越来越强大，但它们的根系——始终扎在 Linux 这片土壤里。

## 给初学者的三个"定心丸"

**第一，你真的可以学会。**

我见过太多人，从"连 `cd` 和 `ls` 都分不清"到"三天写一个生产级部署脚本"。Linux 学习曲线陡峭，但它很诚实——你付出多少，它回报多少，从不欺骗你。

**第二，你现在遇到的每一个报错，都是未来的"肌肉记忆"。**

当你在凌晨三点被报警叫醒，大脑一片空白，却能凭借肌肉记忆敲出 `journalctl -b -1 -p err` 精准定位根因——你会感谢今天那个对着报错发呆却不肯放弃的自己。

**第三，你的技能在 2026 年比以往任何时候都更值钱。**

AI 时代不缺懂 Prompt 的人，不缺会调 API 的人。缺的是**知道 AI 的代码在什么操作系统上跑、怎么让它跑得稳、跑得快、跑得安全**的人。你正在成为那种人。

## 星辰大海，从敲下第一个命令开始

技术的浪潮一波接一波，但 Linux 的底层逻辑，在过去的三十年里坚如磐石，在未来的三十年依然如此。你现在投入的每一个小时，学的每一行命令，踩的每一个坑，都会在 AI 智能体大爆发的时代，为你提供源源不断的回报。

当你第一次用自己写的脚本自动化了某个繁琐的部署任务，你会明白我在说什么。

当你第一次用 `kubectl` 和 `systemd` 让一个 AI 服务稳定运行了三个月没有重启，你会明白我在说什么。

当你某天站在台上，向团队展示你如何用 Linux 工具链，把一个 100 个智能体协作的平台从"能用"优化到"好用"——你会发现，你已经走到了当年仰望的那片星空之下。

那是这一路走来，最好的回报。

现在，你面前有一个终端，一个光标，一行闪烁的提示符。

那就是你的起点。AI 的时代正在敲门，而你——正在学习如何打开那扇门。

**去吧，把世界跑在 Linux 上。**

<br>
<p align="right"><b>作者</b></p>
<p align="right">2026 年 夏</p>
