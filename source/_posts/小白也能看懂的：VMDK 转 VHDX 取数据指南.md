---
abbrlink: ''
categories:
- - 数据还原
date: '2026-06-02T16:20:17.537373+08:00'
tags:
- vmdk 转 vhdx
title: 小白也能看懂的：VMDK 转 VHDX 取数据指南
updated: '2026-06-02T16:20:17.917+08:00'
---
## 什么时候需要看这篇教程？

当你用 VMware Workstation 打开虚拟机，磁盘报错、无法启动，但磁盘文件（.vmdk）还在硬盘上，你需要把里面的数据取出来。

举个真实例子——我们这次遇到的 VM 同时出了两个问题：

1. 磁盘描述文件里文件名乱码 → VMware 找不到文件，连系统盘都打不开
2. VM 启用了加密 → 想挂载第二块数据盘，VMware 直接拒绝，报 unencrypted and cannot be used

最终靠这篇教程的方法成功取出了所有数据。

---

## 这个方法的思路是什么？

```
你的 VMDK                中间格式（VHDX）             结果
┌──────────┐    转换     ┌──────────┐   双击挂载    ┌──────────┐
│ 被 VMware │ ────────→  │ Windows   │ ───────────→ │ 盘符出现  │
│ 锁死的盘  │            │ 原生格式  │              │ 随便拷文件│
└──────────┘            └──────────┘              └──────────┘
```

简单说就是：把 VMware 的磁盘格式，转成 Windows 系统自己能打开的格式。整个过程不修改原始文件，数据安全。

---

## 准备工具

### 工具一：qemu-img（磁盘格式转换工具）

这是一个免费开源的小工具，能把各种虚拟磁盘格式互相转换。

下载地址：👉 https://qemu.weilnetz.de/w64/

进入后找最新的 qemu-w64-xxx.zip（xxx 是版本号，选数字最大的下载即可）。

下载后解压，找到里面的 qemu-img.exe，记住它的完整路径，例如：

```
C:\Users\你的用户名\Downloads\qemu\qemu-img.exe
```

> 💡 小技巧：把压缩包解压到 C:\\Tools\\qemu\\ 这样固定的位置，以后还能用。

验证安装成功（在 PowerShell 里运行，路径换成你自己的）：

```powershell
C:\Tools\qemu\qemu-img.exe --version
```

如果看到类似 qemu-img version X.X 的输出，就说明可以用了。

### 工具二：一个能运行命令的终端

Windows 自带，右键开始菜单 → Windows PowerShell（管理员） 或 终端（管理员）。

> ⚠️ 建议用管理员身份打开，后面挂载 VHDX 需要管理员权限。

---

## 动手操作（分四步）

### 第一步：查看 VMDK 磁盘信息

了解你的磁盘多大、实际数据量多少、是什么格式。

```powershell
# 把下面两个路径换成你自己的
#   路径1：qemu-img.exe 的位置
#   路径2：你的 .vmdk 文件的位置

C:\Tools\qemu\qemu-img.exe info "C:\你的VM路径\你的磁盘.vmdk"
```

你会看到类似这样的信息：

```
virtual size: 1 TiB          ← 虚拟磁盘总大小
disk size: 31.5 GiB          ← 实际占用的空间（转换后的 VHDX 大概这么大）
create type: twoGbMaxExtentSparse   ← VMDK 格式
```

重点关注 disk size（实际数据量），确保你的硬盘有 1.5 倍以上的空闲空间。

---

### 第二步：转换成 VHDX

```powershell
C:\Tools\qemu\qemu-img.exe convert -f vmdk -O vhdx -o subformat=dynamic `
    "C:\你的VM路径\你的磁盘.vmdk" `
    "C:\你的VM路径\你的磁盘-数据.vhdx"
```

参数解释（不用记，用的时候看一眼就行）：


| 参数                 | 含义                                     |
| -------------------- | ---------------------------------------- |
| convert              | 转换模式                                 |
| -f vmdk              | 源格式是 VMDK                            |
| -O vhdx              | 目标格式是 VHDX                          |
| -o subformat=dynamic | 动态磁盘（实际占多少就多大，不浪费空间） |
| 最后一个路径         | 输出文件位置，建议和原始 VMDK 放同一目录 |

⏱️ 等待时间：通常 5-15 分钟，取决于磁盘实际数据量。30GB 左右大约 3-5 分钟。

---

### 第三步：修复 VHDX 文件属性

这一步非常关键！如果你直接跳到第四步挂载，大概率会报这个错误：

> Make sure the file is in an NTFS volume and isn't in a compressed folder or volume.

这是因为 qemu-img 生成的 VHDX 文件带有一个叫 SparseFile（稀疏文件） 的属性，Windows 不允许挂载带这个属性的 VHDX。

一行命令去掉它：

```powershell
fsutil sparse setflag "C:\你的VM路径\你的磁盘-数据.vhdx" 0
```

没有任何报错信息就说明成功了。

---

### 第四步：挂载 VHDX，取数据

三种方式任选一个：

方式 A（最简单）：文件管理器双击

1. 打开文件管理器，找到 .vhdx 文件
2. 右键 → 装载
3. 左侧导航栏会出现一个新盘符

方式 B：磁盘管理（更稳定）

1. 按 Win + X → 选择 磁盘管理
2. 顶部菜单 操作 → 附加 VHD
3. 浏览，选择你的 .vhdx 文件 → 确定
4. 稍等几秒，资源管理器里就会出现新盘符

方式 C：命令行（适合记命令的人）

```powershell
Mount-DiskImage -ImagePath "C:\你的VM路径\你的磁盘-数据.vhdx"
```

---

挂载成功后，VHDX 里的内容就和你插了一个 U 盘一样 —— 直接打开、复制、粘贴，把需要的数据拷出来就行。

---

## 用完后怎么处理？

### 卸载 VHDX

右键新出现的盘符 → 弹出，和弹出 U 盘一样的操作。

或者用命令：

```powershell
Dismount-DiskImage -ImagePath "C:\你的VM路径\你的磁盘-数据.vhdx"
```

### 清理 VHDX 文件

确认数据已经全部取出后，可以删掉 .vhdx 文件来释放空间（原始 VMDK 还在，不会丢）：

```powershell
Remove-Item "C:\你的VM路径\你的磁盘-数据.vhdx"
```

> ⚠️ 注意：只删 .vhdx，不要动.vmdk 和 .vmdk 旁边的一堆 -s001.vmdk、-s002.vmdk 文件，那些是原始数据。

---

## 常见报错解决

### 报错 ①：挂载时提示 NTFS / compressed

```
Make sure the file is in an NTFS volume and isn't in a compressed folder or volume.
```

解决：

```powershell
# 先看文件有没有问题属性
(Get-Item "你的文件.vhdx").Attributes

# 如果输出包含 "SparseFile"，运行：
fsutil sparse setflag "你的文件.vhdx" 0

# 如果输出包含 "Compressed"，运行：
compact /u "你的文件.vhdx"
```

然后再试挂载。

### 报错 ②：qemu-img 找不到 extent 文件

```
Could not open 'xxx-s001.vmdk': No such file or directory
```

原因：VMDK 的描述文件（那个很小的 .vmdk）和实际数据文件（一堆 -s001.vmdk）不在同一目录，或者描述文件里写的文件名和实际文件名不一致。

解决：

* 确保所有 -sxxx.vmdk 文件和描述文件在同一个文件夹
* 如果文件名确实不匹配，需要编辑描述文件（用记事本打开那个很小的 .vmdk，检查 RW XXXX SPARSE "文件名" 那一行引号里的名字和实际文件是否一致）

### 报错 ③：磁盘管理里 VHDX 显示为 RAW 或未初始化

这可能是因为 VMDK 里的文件系统本身有问题，或者 VMDK 是 Linux 的格式（ext4/xfs）。

* Linux 格式：安装 [Linux Reader](https://www.diskinternals.com/linux-reader/) 来读取
* 文件系统损坏：尝试用 Windows 的 chkdsk 修复（不过成功率不高，此时 VMDK 的数据可能本身就有问题）

---

## 和我们这次遇到的问题对照


| 我们的情况                                 | 怎么解决的                                                     |
| ------------------------------------------ | -------------------------------------------------------------- |
| VMDK 描述文件里 GBK 编码的中文名变成乱码   | 先用perl -pe 命令修复了描述文件里的文件名                      |
| VM 启用了加密，VMware GUI 拒绝挂载未加密盘 | 转成 VHDX 后完全绕过了 VMware，因为 Windows 不管 VM 的加密策略 |
| qemu-img 生成的 VHDX 带 SparseFile 属性    | fsutil sparse setflag xx.vhdx 0 去掉                           |

核心经验：只要 VMDK 的数据文件还在，即使 VMware 完全不让你打开，也可以通过 qemu-img → VHDX → Windows 直接挂载这条路径把数据拿出来。
