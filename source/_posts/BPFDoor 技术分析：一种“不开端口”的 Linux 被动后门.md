---
abbrlink: ''
categories:
- - 手法技巧
date: '2026-05-22T14:56:53.377985+08:00'
excerpt: 写在前面 提到 Linux 后门，很多人第一反应是反弹 shell、WebShell、异常监听端口或者可疑计划任务。但 BPFDoor 比这些常见形态更隐蔽：它平时不一定主动外联，也不一定打开一个固定端口等待连接，而是通过 PF_PACKET packet socket 或 raw socket 在主机网络栈中捕获链路层帧或原始 IP 包。只有收到特定格式的数据包后，它才会触发后续动作。 这也是 ...
tags:
- BPFDoor
title: BPFDoor 技术分析：一种“不开端口”的 Linux 被动后门
updated: '2026-05-22T14:57:33.544+08:00'
---
## 写在前面

提到 Linux 后门，很多人第一反应是反弹 shell、WebShell、异常监听端口或者可疑计划任务。但 BPFDoor 比这些常见形态更隐蔽：它平时不一定主动外联，也不一定打开一个固定端口等待连接，而是通过 PF\_PACKET packet socket 或 raw socket 在主机网络栈中捕获链路层帧或原始 IP 包。只有收到特定格式的数据包后，它才会触发后续动作。

这也是 BPFDoor 最值得关注的地方：它不是简单的反弹 shell，而是一种被动触发型 Linux 后门。

BPFDoor 早期公开报告将其关联到 Red Menshen，后续也被 Trend Micro 等厂商跟踪为 Earth Bluecrow。公开资料显示，该家族长期出现在电信、政府、金融、零售等高价值环境中，尤其适合在高可用 Linux 服务器、边界服务、核心网主机、虚拟化平台或容器节点中长期潜伏。

BPFDoor 的核心特征不是“打开一个端口等待连接”，而是创建 PF\_PACKET packet socket 或 raw socket，并通过 BPF/cBPF 过滤器筛选进入主机网络栈的数据包。只有当数据包中包含预设的 magic bytes、密码或控制字段时，程序才会触发 bind shell、reverse shell、临时端口重定向或存活探测。

Rapid7 在 2026 年发布的研究和检测脚本中强调，较新变体不仅保留 classic BPFDoor 的 raw socket 与 BPF 触发机制，还可能将控制字段嵌入看似正常的 HTTPS 流量，并结合 ICMP 控制信号、代理友好的偏移规则和更隐蔽的控制器逻辑提升隐蔽性。

这篇文章会围绕三个问题展开：

1. BPFDoor 到底是怎么工作的？
2. 它和普通反弹 shell 有什么本质区别？
3. 在 Linux 主机上应该如何发现、验证和处置它？

重点关注的行为特征包括：

* 异常 packet/raw socket；
* BPF filter 附加行为；
* 伪装系统服务的进程；
* deleted executable；
* /dev/shm、/tmp、/var/tmp 等临时目录执行；
* HISTFILE=/dev/null、MYSQL\_HISTFILE=/dev/null 等可疑环境变量；
* packet\_recvmsg、wait\_for\_more\_packets 等内核栈痕迹；
* 临时 iptables/nftables 重定向规则；
* 历史高端口 shell 与异常交互流量。

## 参考资料

* Hackers-Arise: Compromising Telecom Systems: Deploying and Detecting the BPFDoor Backdoor
  https://hackers-arise.com/compromising-telecom-systems-deploying-and-detecting-the-bpfdoor-backdoor/
* Rapid7 Labs BPFDoor README
  https://github.com/rapid7/Rapid7-Labs/blob/main/BPFDoor/README.md
* Rapid7 BPFDoor detection script
  https://raw.githubusercontent.com/rapid7/Rapid7-Labs/main/BPFDoor/rapid7\_detect\_bpfdoor.sh
* Rapid7 Threat Research: BPFDoor Telecom Networks Sleeper Cells
  https://www.rapid7.com/blog/post/tr-bpfdoor-telecom-networks-sleeper-cells-threat-research-report/
* pjt3591oo/bpfdoor PoC
  https://github.com/pjt3591oo/bpfdoor
* Sandfly Security: BPFDoor technical analysis
  https://sandflysecurity.com/blog/bpfdoor-an-evasive-linux-backdoor-technical-analysis/
* Elastic Security Labs: A peek behind the BPFDoor
  https://www.elastic.co/security-labs/a-peek-behind-the-bpfdoor
* Trend Micro: BPFDoor Hidden Controller
  https://www.trendmicro.com/en\_us/research/25/d/bpfdoor-hidden-controller.html
* MITRE ATT&CK S1161 BPFDoor
  https://attack.mitre.org/software/S1161/

## 一句话理解 BPFDoor

普通反弹 shell 是“程序执行后主动连出去”；BPFDoor 更像是“程序先通过 packet/raw socket 被动接收数据包，等特定敲门包出现后再行动”。

这个“敲门包”就是 magic packet。它可以藏在 TCP、UDP、ICMP，甚至看起来像正常 HTTPS 流量的包里。程序收到后，会根据包里的控制字段决定执行哪种动作：回连、开放本地 shell、临时重定向端口，或者只返回一个存活状态。

这种设计让 BPFDoor 具备几个明显优势：

* 平时几乎没有网络噪声；
* 不依赖固定监听端口；
* 可以把控制入口伪装在正常业务端口上；
* 通过 BPF filter 在内核侧先过滤数据包，降低用户态处理压力；
* 可以绕开只关注监听端口和固定 C2 的检测逻辑。

## 工作原理拆解

### 典型运行链路

flowchart TD     A[目标 Linux 主机已存在执行权限] --> B[投放 BPFDoor 二进制]     B --> C[复制到 /dev/shm、/tmp 或伪装路径]     C --> D[删除原始文件或隐藏磁盘痕迹]     D --> E[伪装进程名: dbus-daemon / auditd / systemd-journald 等]     E --> F[创建 PF\_PACKET 或 raw socket]     F --> G[通过 SO\_ATTACH\_FILTER 加载 BPF/cBPF 过滤器]     G --> H[长期被动接收并筛选入站数据包]     H --> I{数据包是否匹配 magic / 密码 / 控制字段}     I -- 否 --> H     I -- 是 --> J{控制模式}     J -- reverse shell --> K[主动连接控制端]     J -- bind shell/direct mode --> L[短暂开放本地 shell 端口]     J -- redirect mode --> M[临时 iptables/nftables 重定向]     J -- pingback --> N[返回存活状态]     K --> O[建立交互通道]     L --> O     M --> O     N --> H

BPFDoor 通过 BPF filter 在 socket 层先过滤数据包，将绝大多数正常流量丢弃，只把符合特定条件的包交给用户态进程处理。这种设计带来几个重要结果：

* 常态下没有明显监听端口，端口扫描和 ss -lntp 不一定能发现；
* 当使用 PF\_PACKET socket 时，程序可以在主机协议栈较早的位置接收链路层帧，因此部分触发包即使后续被本机防火墙策略阻断，也可能已经被程序看到；
* 真实交互连接可能伪装为访问 SSH、HTTPS 或其他正常业务端口；
* bind shell 端口可能只在触发后短暂出现，并通过 conntrack 维持连接；
* 进程名、路径、环境变量和运行痕迹往往经过伪装或削弱。

### BPF 触发机制

经典变体中，程序会创建 raw socket 或 packet socket，然后通过 setsockopt(..., SO\_ATTACH\_FILTER, ...) 附加 BPF 过滤器。

模拟源码如下：

```c
// 模拟代码：展示 BPFDoor 类后门的核心结构，不是完整可运行样本
int fd = socket(PF_PACKET, SOCK_RAW, htons(ETH_P_ALL));

struct sock_filter filter[] = {
    // BPF 字节码：只放行包含特定 magic bytes 的 TCP/UDP/ICMP 包
};

struct sock_fprog prog = {
    .len = sizeof(filter) / sizeof(filter[0]),
    .filter = filter,
};

setsockopt(fd, SOL_SOCKET, SO_ATTACH_FILTER, &prog, sizeof(prog));

while (1) {
    ssize_t n = recvfrom(fd, buffer, sizeof(buffer), 0, NULL, NULL);
    if (n <= 0) {
        continue;
    }

    if (!valid_magic_packet(buffer, n)) {
        continue;
    }

    struct command cmd = parse_command(buffer, n);
    if (!valid_password(&cmd)) {
        continue;
    }

    dispatch_command(&cmd);
}
```

PoC 项目 pjt3591oo/bpfdoor 展示了一个最小化思路：程序创建 PF\_PACKET/SOCK\_RAW socket，加载 BPF filter，被动接收以特定字节开头的 UDP payload，然后解析目标 IP 并建立 reverse shell。该 PoC 不等同于完整真实家族样本，但适合理解“packet socket 被动接收 + magic byte + reverse shell”的核心模型。

简化后的 PoC 逻辑可表示为：

```c
sd = socket(PF_PACKET, SOCK_RAW, htons(ETH_P_ALL));
apply_bpf_filter(sd);

while (1) {
    pkt = recvfrom(sd, buf, sizeof(buf), 0, NULL, NULL);
    payload = ethernet_header + ip_header + udp_header;

    if (payload[0] == 'X') {
        host = parse_host(payload + 1);
        if (fork() == 0) {
            reverse_shell(host, 4444);
        }
    }
}
```

公开分析中提到过多种历史 magic 模式，包括 TCP magic 0x5293、UDP/ICMP magic 0x7255、0x39393939、0xdead 等。检测时不应只依赖固定 magic 值，因为重新编译或轻微修改样本即可改变这些特征。

### Direct Mode 与临时端口重定向

BPFDoor 的一个重要隐蔽点是 direct mode。该模式下，程序可能只对指定源 IP 添加临时 NAT 重定向规则，把访问合法业务端口的连接转发到本地高端口 shell，例如 42391-43390 范围内的端口。规则随后被删除，但连接可通过 Linux conntrack 继续存在。

sequenceDiagram     participant C as Control Host     participant N as Network / Firewall     participant H as Infected Host     participant B as BPFDoor Process     participant S as Hidden Shell Port      C->>H: 发送 magic packet 到正常业务端口     H->>B: packet/raw socket 捕获触发包     B->>H: 添加临时 NAT REDIRECT 规则     C->>N: 连接 SSH/HTTPS 等正常端口     N->>H: 转发到受影响主机     H->>S: 重定向到本地 hidden shell 端口     B->>H: 删除 NAT REDIRECT 规则     C<->>S: conntrack 维持交互通道

如果只在某个时间点查看防火墙规则，可能错过临时规则。因此建议将 iptables/nftables 规则变更纳入审计，并关注短时间内新增后删除的 REDIRECT --to-ports 行为。

## 它和普通反弹 Shell 有什么区别

普通反弹 shell 的典型模式是：受影响主机上的进程主动连接远端监听端口，例如通过 bash、sh、python、perl、nc 等程序建立交互通道。

BPFDoor 也可能最终建立 reverse shell，但它不是普通反弹 shell。它更接近“被动触发型后门”：平时不一定主动外联，也不一定监听 TCP 端口，而是通过 packet/raw socket 接收进入主机的链路层帧或原始 IP 包。只有收到特定 magic packet 后，才决定是否回连、开放 shell 或临时重定向。


| 对比项           | 普通反弹 shell               | BPFDoor                                                               |
| ---------------- | ---------------------------- | --------------------------------------------------------------------- |
| 通信方式         | 主机主动连接远端监听端口     | 平时通过 packet/raw socket 被动接收数据包，收到 magic packet 后才动作 |
| 是否暴露监听端口 | 通常不监听，但会立即外联     | 常态下通常不暴露监听端口                                              |
| 触发方式         | 命令执行后立即回连           | 特定 TCP/UDP/ICMP magic packet 触发                                   |
| 检测重点         | 出站连接、shell 进程、命令行 | raw/packet socket、BPF filter、伪装进程、deleted exe                  |
| 隐蔽性           | 较低到中等                   | 高                                                                    |
| 防火墙影响       | 依赖出站策略宽松             | 可在本机防火墙前看到触发包                                            |
| 持久化           | 需要额外机制                 | 程序本身可长期驻留                                                    |
| 控制方式         | C2 端口通常相对固定          | 可用正常业务端口触发，甚至临时重定向                                  |

普通反弹 shell：

```text
远端监听 4444
    ↑
受影响主机主动连接 1.2.3.4:4444
    ↑
执行 /bin/sh
```

BPFDoor：

```text
控制端发送 magic packet 到受影响主机
    ↓
BPFDoor 用 packet/raw socket 捕获
    ↓
校验 magic / 密码 / 模式
    ↓
按需回连、开放 shell 或临时重定向
```

因此，检测 BPFDoor 不能只找“谁连接了外部 4444 端口”，还需要关注：

```bash
ss -0pb
cat /proc/net/packet
grep -H "packet_recvmsg\|wait_for_more_packets" /proc/*/stack 2>/dev/null
ls -al /proc/*/exe 2>/dev/null | grep deleted
```

## 为什么传统排查容易漏掉

### 端口扫描不可靠

BPFDoor 常态不需要公开监听端口。即使存在 bind shell，它也可能只在触发后短暂开放。因此外部扫描和 netstat -lntp、ss -lntp 只能发现一部分场景。

### 进程名伪装

样本会伪装成合法系统服务，例如：

* dbus-daemon
* auditd
* systemd-journald
* hpasmlited
* dockerd
* pickup
* atd

只看 ps 输出中的进程名不足以判断，需要结合 /proc/<pid>/exe、/proc/<pid>/cmdline、/proc/<pid>/cwd、包管理器文件归属和 systemd unit 配置。

### Deleted Executable

BPFDoor 可能启动后删除磁盘上的二进制，使 /proc/<pid>/exe 指向 deleted 文件。该行为不是 BPFDoor 独有，但在服务器进程中非常值得关注。

### 环境变量与历史记录规避

多个公开分析提到样本或交互 shell 会设置：

```text
HISTFILE=/dev/null
MYSQL_HISTFILE=/dev/null
HOME=/tmp
```

这些环境变量单独出现不一定代表异常，但与异常 shell、可疑父进程、外联连接或 deleted executable 同时出现时，应按高危事件处理。

## Rapid7 的检测思路值得借鉴

Rapid7 发布的 rapid7\_detect\_bpfdoor.sh 不是简单 hash IOC 扫描，而是行为导向的 live triage 脚本。它面向 classic 和 modern BPFDoor 变体，覆盖进程伪装、raw/packet socket、BPF magic、内核栈、可疑环境变量、内存驻留、互斥文件、C2 连接、历史端口和持久化位置。

### 检测逻辑图

flowchart TD     A[开始主机巡检] --> B[枚举进程与 /proc 信息]     B --> C{进程名是否伪装系统服务}     C -- 是 --> C1[校验 exe 路径、cmdline、包归属]     C -- 否 --> D[继续 socket 检查]     C1 --> D      D[检查 packet/raw socket] --> E{是否存在异常 PF\_PACKET/raw socket}     E -- 是 --> E1[关联 PID、进程名、用户、fd]     E -- 否 --> F[检查内核栈]     E1 --> F      F[扫描 /proc/\*/stack] --> G{是否命中 packet\_recvmsg / wait\_for\_more\_packets}     G -- 是 --> G1[标记为强可疑 sniffer/backdoor]     G -- 否 --> H[检查环境变量]     G1 --> H      H[扫描 /proc/\*/environ] --> I{是否存在 HISTFILE=/dev/null 等组合}     I -- 是 --> I1[关联交互 shell 与父进程]     I -- 否 --> J[检查 deleted exe 与 /dev/shm 执行]     I1 --> J      J --> K{是否存在 deleted executable 或临时目录执行}     K -- 是 --> K1[复制内存中二进制并保全证据]     K -- 否 --> L[检查 iptables/nftables 与历史端口]     K1 --> L      L --> M{是否命中多项 BPFDoor 行为}     M -- 是 --> N[隔离主机、取证、扩大排查范围]     M -- 否 --> O[记录低风险发现并持续监控]

### 重点检查项

#### 进程伪装

Rapid7 脚本会关注进程命令行是否冒充常见系统服务，再对比真实可执行路径。建议重点关注：

* 命令行像系统服务，但真实路径位于 /dev/shm、/tmp、/var/tmp 或异常目录；
* /proc/<pid>/exe 指向 (deleted)；
* 进程名与 systemd unit、包管理器归属不一致；
* 低频系统服务突然创建 raw socket 或对外连接。

#### Packet/Raw Socket 与 BPF Filter

Rapid7 脚本会检查 ss -0pb、/proc/net/packet、/proc/net/raw、/proc/net/raw6 等信息，用于发现可疑 packet/raw socket。

可使用：

```bash
ss -0pb
cat /proc/net/packet 2>/dev/null
cat /proc/net/raw 2>/dev/null
cat /proc/net/raw6 2>/dev/null
```

正常情况下，合法使用 packet/raw socket 的进程通常包括抓包工具、安全探针、网络监控 agent、容器网络组件或 IDS/IPS。任何伪装系统服务、临时目录执行文件或 deleted executable 使用 packet/raw socket，都应优先调查。

#### 内核栈

后门长期等待 packet socket 数据时，内核栈可能出现：

```text
packet_recvmsg
wait_for_more_packets
```

排查命令：

```bash
grep -H "packet_recvmsg\|wait_for_more_packets" /proc/*/stack 2>/dev/null
```

该命中不是 BPFDoor 独有，合法抓包程序也可能出现类似栈。但如果命中进程同时具备伪装、deleted exe、可疑环境变量或临时目录执行等特征，风险显著升高。

#### 环境变量

检查禁用历史记录的环境变量：

```bash
for p in /proc/[0-9]*; do
  tr '\0' '\n' < "$p/environ" 2>/dev/null | \
  grep -E "HISTFILE=/dev/null|MYSQL_HISTFILE=/dev/null|HOME=/tmp" && echo "$p"
done
```

建议把这些环境变量与父进程、登录来源、TTY、网络连接一起关联。

#### Mutex 与运行痕迹

早期公开样本曾使用 /var/run/haldrund.pid 等互斥文件。Rapid7 检测脚本也会检查多组历史 mutex 文件名和零字节 pid/lock 文件。

建议重点关注：

* /var/run/\*.pid 或 /run/\*.pid 中无法对应 systemd 服务的文件；
* 零字节 lock 文件；
* 文件创建时间与异常登录、异常进程启动时间接近；
* pid 文件名伪装成系统服务但没有包管理器归属。

#### 历史端口与规则变更

公开分析中多次提到 42391-43390、8000 等历史端口，但端口不是稳定特征。更应关注行为组合：

```bash
iptables-save
nft list ruleset
ss -antupw
```

重点看：

* REDIRECT --to-ports 42391:43390；
* 短时间出现又消失的 NAT PREROUTING 规则；
* 对单个源 IP 特殊放行或重定向；
* 非网络管理进程调用 iptables、nft、ip、tc。

## 主机上怎么查

下面这些命令适合在可疑主机上做快速确认。需要注意的是，单条命令命中并不等于一定存在 BPFDoor；关键是看多个异常是否同时出现。

### 快速排查命令

```bash
# 查找 deleted executable
ls -al /proc/*/exe 2>/dev/null | grep deleted

# 查找 packet socket 相关栈
grep -H "packet_recvmsg\|wait_for_more_packets" /proc/*/stack 2>/dev/null

# 查看 packet/raw sockets
ss -0pb
cat /proc/net/packet 2>/dev/null
cat /proc/net/raw 2>/dev/null
cat /proc/net/raw6 2>/dev/null

# 检查可疑环境变量
for p in /proc/[0-9]*; do
  tr '\0' '\n' < "$p/environ" 2>/dev/null | \
  grep -E "HISTFILE=/dev/null|MYSQL_HISTFILE=/dev/null|HOME=/tmp" && echo "$p"
done

# 检查临时目录可执行文件
find /dev/shm /tmp /var/tmp -maxdepth 2 -type f -perm -111 -ls 2>/dev/null

# 检查 NAT/防火墙规则
iptables-save 2>/dev/null
nft list ruleset 2>/dev/null
```

### 可疑程度评分

flowchart TD     A[发现异常进程] --> B{是否 packet/raw socket}     B -- 否 --> C[继续常规调查]     B -- 是 --> D{是否合法网络工具或安全 agent}     D -- 是 --> E[核对版本、路径、配置、变更记录]     D -- 否 --> F{是否存在 deleted exe 或临时目录执行}     F -- 是 --> H[高危: 立即取证隔离]     F -- 否 --> G{是否伪装系统服务}     G -- 是 --> H     G -- 否 --> I{是否有可疑 env / iptables / 高端口 shell}     I -- 是 --> H     I -- 否 --> J[中危: 保留证据并扩大上下文排查]

## 日志与检测规则怎么设计

建议将以下规则作为组合检测，而不是孤立告警：

* root 用户从 /dev/shm、/tmp、/var/tmp 执行 ELF；
* 非白名单进程创建 PF\_PACKET、SOCK\_RAW 或调用 SO\_ATTACH\_FILTER；
* 进程名伪装系统服务，但真实可执行路径不在标准位置；
* /proc/<pid>/exe 指向 deleted 文件；
* 进程环境变量包含 HISTFILE=/dev/null 或 MYSQL\_HISTFILE=/dev/null；
* 非网络管理工具执行 iptables -t nat、nft、tc 或修改转发表；
* 新增并快速删除 NAT REDIRECT 规则；
* 服务器正常业务端口上出现非协议交互流量；
* 主机主动连接罕见外部 IP 或历史 C2 端口。

网络侧可关注：

* TCP payload 中出现历史 magic 模式，如 0x5293、0x39393939；
* UDP/ICMP payload 中出现历史 magic 模式，如 0x7255；
* ICMP Echo request 携带固定长度异常控制字段；
* 单字节 UDP 状态回包，例如 "1"；
* 对 SSH/HTTPS/其他业务端口的低速人工交互型流量；
* 访问业务端口后，主机出现异常外联或交互通道。

需要注意，网络 IOC 容易被重新编译绕过。实际检测应把网络触发包、主机 packet socket、进程上下文和防火墙规则变更关联起来。

## 发现可疑主机后怎么处置

flowchart TD     A[疑似发现 BPFDoor] --> B[不要立即 kill 进程]     B --> C[保全进程、网络、规则、文件证据]     C --> D[隔离主机网络或迁移业务]     D --> E[复制 deleted exe 与内存映射]     E --> F[提取 /proc/pid/environ / fd / maps / stack / cmdline]     F --> G[导出 iptables-save / nft ruleset / ss 连接]     G --> H[分析初始访问与横向移动]     H --> I[按行为特征扩大排查范围]     I --> J[凭据轮换与持久化清理]     J --> K[重装或可信恢复高价值主机]     K --> L[回补检测规则与基线]

### 现场保全

建议先收集：

```bash
date -u
hostname -f
who -a
w
last -ai
ps -efww
pstree -ap
ss -antupw
ss -0pb
iptables-save
nft list ruleset
```

对可疑 PID 收集：

```bash
PID=<suspicious_pid>

readlink -f /proc/$PID/exe
ls -al /proc/$PID/exe
tr '\0' '\n' < /proc/$PID/environ
tr '\0' ' ' < /proc/$PID/cmdline
cat /proc/$PID/status
cat /proc/$PID/maps
cat /proc/$PID/stack
ls -al /proc/$PID/fd
```

如果 /proc/\$PID/exe 显示 deleted，可以尝试在隔离后复制：

```bash
cp /proc/$PID/exe /root/forensics/pid_${PID}_exe_deleted
sha256sum /root/forensics/pid_${PID}_exe_deleted
```

### 清除与恢复

如果确认主机存在 BPFDoor，不建议只删除单个进程或文件后继续使用。由于该类程序通常意味着主机已具备高权限执行历史，必须假设存在其他持久化和凭据泄露。

建议：

* 隔离主机，阻断交互通道；
* 保全证据后下线受影响服务；
* 从可信镜像重装高价值服务器；
* 轮换 SSH key、运维账号、数据库账号、API token、CI/CD secret；
* 排查同网段服务器、跳板机、堡垒机、VPN、ESXi、Kubernetes 节点和数据库服务器；
* 审计 cron、systemd、rc.local、profile、authorized\_keys、容器镜像、DaemonSet、initContainer；
* 检查日志缺口、异常清理行为和时间线。

## 加固建议

### Linux 主机基线

* 限制 root 直接登录，强制使用可审计的提权路径；
* 对 /dev/shm、/tmp、/var/tmp 尽量启用 noexec，同时评估业务兼容性；
* 对关键目录启用文件完整性监控，例如 /usr/sbin、/usr/bin、/etc/systemd、/etc/cron\*、/etc/sysconfig；
* 监控 /proc/\*/exe deleted 状态；
* 对 raw socket、packet socket、BPF filter 相关行为做采集；
* 记录 iptables/nftables 规则变更审计；
* 对服务器出站连接做最小化控制，降低异常交互通道建立概率。

### 检测工程化

推荐将检测拆成三层：

flowchart LR     A[主机 telemetry] --> D[关联分析]     B[网络 telemetry] --> D     C[配置与变更审计] --> D     D --> E{是否命中多维行为}     E -- 是 --> F[高危告警]     E -- 否 --> G[低优先级观察]      A1[进程、socket、/proc、环境变量] --> A     B1[magic 包、异常协议、C2 连接] --> B     C1[iptables/nftables、systemd、cron] --> C

单条 IOC 容易误报或漏报。更实用的高置信组合包括：

* PF\_PACKET/raw socket + deleted exe；
* PF\_PACKET/raw socket + 进程名伪装；
* SO\_ATTACH\_FILTER + 临时目录执行；
* iptables REDIRECT + 高端口 shell；
* HISTFILE=/dev/null + 异常网络连接；
* packet\_recvmsg 栈 + 非白名单进程。

## 安全复现实验：验证检测面，不复现真实后门

这一节的目标不是复现真实攻击链，而是在隔离环境里构造“可观测行为”，用来验证检测脚本、日志采集和规则效果。实验不建议在生产环境执行，也不应复现持久化、真实控制通道、凭据窃取或可用交互 shell。推荐使用一次性 Linux 虚拟机或容器宿主测试机，并确保测试网络与办公、生产网络隔离。

### 实验目标

复现并观察以下行为：

* 进程创建 PF\_PACKET / raw socket；
* 进程通过 SO\_ATTACH\_FILTER 加载 BPF/cBPF filter；
* /proc/net/packet、ss -0pb 中出现 packet socket；
* /proc/<pid>/stack 中可能出现 packet\_recvmsg 或相关等待收包函数；
* 临时目录执行文件的检测效果；
* deleted executable 的检测效果；
* Rapid7 检测脚本或自定义排查命令的命中结果。

不复现以下行为：

* 不建立可用 reverse shell；
* 不开放 bind shell；
* 不添加持久化；
* 不修改系统服务；
* 不连接公网控制端；
* 不投放真实恶意样本。

### 实验拓扑

flowchart LR     A[Test Sender VM] -->|发送 UDP/ICMP/TCP 测试包| B[Isolated Linux VM]     B --> C[Benign BPF Listener]     B --> D[Detection Script]     B --> E[System Logs / procfs / ss]

建议准备两台隔离虚拟机：

* Test Sender VM：用于发送普通测试包；
* Isolated Linux VM：运行无害 BPF listener 和检测脚本。

如果只有一台虚拟机，也可以在本机使用 loopback 或本机网卡发送测试包，但 packet socket、网卡和权限行为可能与真实网络环境略有差异。

### 环境准备

在隔离 Linux VM 上安装基础工具：

```bash
sudo apt-get update
sudo apt-get install -y build-essential iproute2 tcpdump netcat-openbsd
```

CentOS/RHEL 系可使用：

```bash
sudo yum groupinstall -y "Development Tools"
sudo yum install -y iproute tcpdump nmap-ncat
```

运行 packet/raw socket 通常需要 root 权限或 CAP\_NET\_RAW 能力。因此实验程序建议只在隔离环境中使用 sudo 临时运行。

### 无害 BPF Listener 示例

下面示例只演示 packet socket 与 BPF filter 的检测面，不执行 shell、不连接远端、不修改防火墙规则。它会捕获 UDP 包中首字节为 X 的 payload，并打印一行日志。

```c
// benign_bpf_listener.c
// 仅用于隔离实验环境中的检测验证，不包含 shell、持久化或网络控制功能。
#include <arpa/inet.h>
#include <linux/filter.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/udp.h>
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>

int main(void) {
    int fd = socket(PF_PACKET, SOCK_RAW, htons(ETH_P_ALL));
    if (fd < 0) {
        perror("socket");
        return 1;
    }

    /*
     * 简化 BPF：放行所有包。
     * 检测验证重点是 SO_ATTACH_FILTER 与 packet socket 行为。
     * 生产级检测不应只依赖 BPF 字节码内容。
     */
    struct sock_filter code[] = {
        BPF_STMT(BPF_RET | BPF_K, 0xFFFFFFFF),
    };
    struct sock_fprog prog = {
        .len = (unsigned short)(sizeof(code) / sizeof(code[0])),
        .filter = code,
    };

    if (setsockopt(fd, SOL_SOCKET, SO_ATTACH_FILTER, &prog, sizeof(prog)) < 0) {
        perror("setsockopt");
        close(fd);
        return 1;
    }

    printf("benign listener started, pid=%d\n", getpid());

    unsigned char buf[2048];
    while (1) {
        ssize_t n = recvfrom(fd, buf, sizeof(buf), 0, NULL, NULL);
        if (n < 0) {
            perror("recvfrom");
            break;
        }

        if ((size_t)n < sizeof(struct ethhdr) + sizeof(struct iphdr)) {
            continue;
        }

        struct iphdr *iph = (struct iphdr *)(buf + sizeof(struct ethhdr));
        if (iph->protocol != IPPROTO_UDP) {
            continue;
        }

        size_t ip_len = iph->ihl * 4;
        if ((size_t)n < sizeof(struct ethhdr) + ip_len + sizeof(struct udphdr) + 1) {
            continue;
        }

        unsigned char *payload = buf + sizeof(struct ethhdr) + ip_len + sizeof(struct udphdr);
        if (payload[0] == 'X') {
            printf("matched harmless test packet, len=%zd\n", n);
            fflush(stdout);
        }
    }

    close(fd);
    return 0;
}
```

编译并从临时目录运行：

```bash
gcc -O2 -Wall benign_bpf_listener.c -o benign_bpf_listener
cp benign_bpf_listener /dev/shm/benign_bpf_listener
sudo /dev/shm/benign_bpf_listener
```

另开一个终端，观察 packet socket：

```bash
sudo ss -0pb
sudo cat /proc/net/packet
pidof benign_bpf_listener
```

对 PID 做进一步检查：

```bash
PID=$(pidof benign_bpf_listener)
sudo readlink -f /proc/$PID/exe
sudo tr '\0' ' ' < /proc/$PID/cmdline
sudo cat /proc/$PID/stack 2>/dev/null
sudo ls -al /proc/$PID/fd
```

从发送端发送测试 UDP 包：

```bash
printf 'Xtest' | nc -u -w1 <isolated_linux_vm_ip> 9999
```

预期结果：

* listener 终端打印 matched harmless test packet；
* ss -0pb 能看到 packet socket 与进程关联；
* /proc/net/packet 出现对应 socket；
* 检测命令能够识别临时目录执行与 packet socket 行为。

### deleted executable 复现

该实验用于验证 deleted executable 检测，不涉及真实恶意能力。

启动 listener 后，在另一个终端删除运行中的文件：

```bash
sudo rm -f /dev/shm/benign_bpf_listener
PID=$(pidof benign_bpf_listener)
sudo ls -al /proc/$PID/exe
```

预期可看到 /proc/<pid>/exe 指向的路径包含 (deleted)。

使用排查命令验证：

```bash
sudo ls -al /proc/*/exe 2>/dev/null | grep deleted
```

检测意义：

* deleted executable 本身不是恶意行为的充分条件；
* 但与 packet/raw socket、临时目录执行、伪装进程、异常网络连接组合出现时，风险显著升高。

### Rapid7 脚本验证

在隔离环境中下载或复制 Rapid7 检测脚本后运行：

```bash
chmod +x rapid7_detect_bpfdoor.sh
sudo ./rapid7_detect_bpfdoor.sh
```

观察脚本是否对以下行为产生提示：

* packet/raw socket；
* 临时目录执行；
* deleted executable；
* 可疑进程上下文；
* /proc 相关异常。

由于本实验使用的是无害 listener，不包含真实 BPFDoor 的 magic bytes、mutex、C2、进程伪装和历史端口，脚本不一定会给出完整高危结论。这里的验证重点是确认基础 telemetry 是否可见，以及检测链路是否能覆盖关键行为。

### 清理实验环境

实验结束后清理进程和文件：

```bash
PID=$(pidof benign_bpf_listener)
sudo kill $PID
sudo rm -f /dev/shm/benign_bpf_listener
rm -f benign_bpf_listener benign_bpf_listener.c
```

确认没有残留 packet socket：

```bash
sudo ss -0pb
sudo cat /proc/net/packet
```

### 复现结果记录模板

建议记录以下信息，便于后续调优检测规则：

```text
测试时间:
测试主机:
内核版本:
发行版版本:
listener PID:
执行路径:
是否命中 /proc/net/packet:
是否命中 ss -0pb:
是否出现 packet_recvmsg:
是否出现 deleted executable:
Rapid7 脚本输出摘要:
自定义规则命中情况:
误报/漏报说明:
后续规则调整:
```

## 总结

BPFDoor 的威胁模型不是“某个恶意端口”，而是“通过 packet/raw socket 被动接收数据包，并由特定 magic packet 触发的后门”。它利用 Linux 合法能力形成低噪声持久访问：BPF 在 socket 层过滤数据包，packet/raw socket 负责接收链路层帧或原始 IP 包，进程名伪装降低人工排查概率，iptables 临时重定向混淆真实连接，deleted executable 和禁用历史记录减少取证痕迹。

Rapid7 的检测思路值得借鉴：围绕行为组合做 live triage，而不是只扫 hash 或固定 magic byte。生产环境中应将这些检查固化为日志规则、终端检测策略、Linux 基线巡检、高价值服务器专项排查和事件响应手册。

优先级最高的落地项是：

1. 建立合法 packet/raw socket 进程白名单。
2. 监控 deleted executable、临时目录执行和伪装系统服务。
3. 审计 iptables/nftables 规则变更。
4. 关联 packet\_recvmsg 内核栈、可疑环境变量和网络连接。
5. 对电信、边界服务、VPN、堡垒机、数据库、虚拟化和容器节点开展专项排查。

BPFDoor 是一种面向 Linux/Unix 服务器的被动式后门程序，早期公开报告将其关联到 Red Menshen，后续也被 Trend Micro 等厂商跟踪为 Earth Bluecrow。公开资料显示，该家族长期出现在电信、政府、金融、零售等高价值环境中，尤其适合在高可用 Linux 服务器、边界服务、核心网主机、虚拟化平台或容器节点中长期潜伏。

BPFDoor 的核心特征不是“打开一个端口等待连接”，而是创建 raw socket 或 packet socket，并通过 BPF/cBPF 过滤器在较低网络层观察入站流量。只有当数据包中包含预设的 magic bytes、密码或控制字段时，程序才会触发 bind shell、reverse shell、临时端口重定向或存活探测。

Rapid7 在 2026 年发布的研究和检测脚本中强调，较新变体不仅保留 classic BPFDoor 的 raw socket 与 BPF 触发机制，还可能将控制字段嵌入看似正常的 HTTPS 流量，并结合 ICMP 控制信号、代理友好的偏移规则和更隐蔽的控制器逻辑提升隐蔽性。

本文从技术分析、检测排查、日志关联和事件处置角度进行整理，重点关注以下行为特征：

* 异常 packet/raw socket；
* BPF filter 附加行为；
* 伪装系统服务的进程；
* deleted executable；
* /dev/shm、/tmp、/var/tmp 等临时目录执行；
* HISTFILE=/dev/null、MYSQL\_HISTFILE=/dev/null 等可疑环境变量；
* packet\_recvmsg、wait\_for\_more\_packets 等内核栈痕迹；
* 临时 iptables/nftables 重定向规则；
* 历史高端口 shell 与异常交互流量。

## 2. 参考资料

* Hackers-Arise: Compromising Telecom Systems: Deploying and Detecting the BPFDoor Backdoor
  https://hackers-arise.com/compromising-telecom-systems-deploying-and-detecting-the-bpfdoor-backdoor/
* Rapid7 Labs BPFDoor README
  https://github.com/rapid7/Rapid7-Labs/blob/main/BPFDoor/README.md
* Rapid7 BPFDoor detection script
  https://raw.githubusercontent.com/rapid7/Rapid7-Labs/main/BPFDoor/rapid7\_detect\_bpfdoor.sh
* Rapid7 Threat Research: BPFDoor Telecom Networks Sleeper Cells
  https://www.rapid7.com/blog/post/tr-bpfdoor-telecom-networks-sleeper-cells-threat-research-report/
* pjt3591oo/bpfdoor PoC
  https://github.com/pjt3591oo/bpfdoor
* Sandfly Security: BPFDoor technical analysis
  https://sandflysecurity.com/blog/bpfdoor-an-evasive-linux-backdoor-technical-analysis/
* Elastic Security Labs: A peek behind the BPFDoor
  https://www.elastic.co/security-labs/a-peek-behind-the-bpfdoor
* Trend Micro: BPFDoor Hidden Controller
  https://www.trendmicro.com/en\_us/research/25/d/bpfdoor-hidden-controller.html
* MITRE ATT&CK S1161 BPFDoor
  https://attack.mitre.org/software/S1161/

## 3. BPFDoor 技术原理

### 3.1 典型运行链路

flowchart TD     A[目标 Linux 主机已存在执行权限] --> B[投放 BPFDoor 二进制]     B --> C[复制到 /dev/shm、/tmp 或伪装路径]     C --> D[删除原始文件或隐藏磁盘痕迹]     D --> E[伪装进程名: dbus-daemon / auditd / systemd-journald 等]     E --> F[创建 PF\_PACKET 或 raw socket]     F --> G[通过 SO\_ATTACH\_FILTER 加载 BPF/cBPF 过滤器]     G --> H[长期被动观察入站流量]     H --> I{数据包是否匹配 magic / 密码 / 控制字段}     I -- 否 --> H     I -- 是 --> J{控制模式}     J -- reverse shell --> K[主动连接控制端]     J -- bind shell/direct mode --> L[短暂开放本地 shell 端口]     J -- redirect mode --> M[临时 iptables/nftables 重定向]     J -- pingback --> N[返回存活状态]     K --> O[建立交互通道]     L --> O     M --> O     N --> H

BPFDoor 通过 BPF filter 将绝大多数正常流量丢弃，只把符合特定条件的包交给用户态进程处理。这种设计带来几个重要结果：

* 常态下没有明显监听端口，端口扫描和 ss -lntp 不一定能发现；
* 触发包可能被本机防火墙阻断，但程序仍可能在 packet/raw socket 层看到；
* 真实交互连接可能伪装为访问 SSH、HTTPS 或其他正常业务端口；
* bind shell 端口可能只在触发后短暂出现，并通过 conntrack 维持连接；
* 进程名、路径、环境变量和运行痕迹往往经过伪装或削弱。

### 3.2 BPF 触发机制

经典变体中，程序会创建 raw socket 或 packet socket，然后通过 setsockopt(..., SO\_ATTACH\_FILTER, ...) 附加 BPF 过滤器。

模拟源码如下：

```c
// 模拟代码：展示 BPFDoor 类后门的核心结构，不是完整可运行样本
int fd = socket(PF_PACKET, SOCK_RAW, htons(ETH_P_ALL));

struct sock_filter filter[] = {
    // BPF 字节码：只放行包含特定 magic bytes 的 TCP/UDP/ICMP 包
};

struct sock_fprog prog = {
    .len = sizeof(filter) / sizeof(filter[0]),
    .filter = filter,
};

setsockopt(fd, SOL_SOCKET, SO_ATTACH_FILTER, &prog, sizeof(prog));

while (1) {
    ssize_t n = recvfrom(fd, buffer, sizeof(buffer), 0, NULL, NULL);
    if (n <= 0) {
        continue;
    }

    if (!valid_magic_packet(buffer, n)) {
        continue;
    }

    struct command cmd = parse_command(buffer, n);
    if (!valid_password(&cmd)) {
        continue;
    }

    dispatch_command(&cmd);
}
```

PoC 项目 pjt3591oo/bpfdoor 展示了一个最小化思路：程序创建 PF\_PACKET/SOCK\_RAW socket，加载 BPF filter，被动接收以特定字节开头的 UDP payload，然后解析目标 IP 并建立 reverse shell。该 PoC 不等同于完整真实家族样本，但适合理解“被动监听 + magic byte + reverse shell”的核心模型。

简化后的 PoC 逻辑可表示为：

```c
sd = socket(PF_PACKET, SOCK_RAW, htons(ETH_P_ALL));
apply_bpf_filter(sd);

while (1) {
    pkt = recvfrom(sd, buf, sizeof(buf), 0, NULL, NULL);
    payload = ethernet_header + ip_header + udp_header;

    if (payload[0] == 'X') {
        host = parse_host(payload + 1);
        if (fork() == 0) {
            reverse_shell(host, 4444);
        }
    }
}
```

公开分析中提到过多种历史 magic 模式，包括 TCP magic 0x5293、UDP/ICMP magic 0x7255、0x39393939、0xdead 等。检测时不应只依赖固定 magic 值，因为重新编译或轻微修改样本即可改变这些特征。

### 3.3 Direct Mode 与临时端口重定向

BPFDoor 的一个重要隐蔽点是 direct mode。该模式下，程序可能只对指定源 IP 添加临时 NAT 重定向规则，把访问合法业务端口的连接转发到本地高端口 shell，例如 42391-43390 范围内的端口。规则随后被删除，但连接可通过 Linux conntrack 继续存在。

sequenceDiagram     participant C as Control Host     participant N as Network / Firewall     participant H as Infected Host     participant B as BPFDoor Process     participant S as Hidden Shell Port      C->>H: 发送 magic packet 到正常业务端口     H->>B: packet/raw socket 捕获触发包     B->>H: 添加临时 NAT REDIRECT 规则     C->>N: 连接 SSH/HTTPS 等正常端口     N->>H: 转发到受影响主机     H->>S: 重定向到本地 hidden shell 端口     B->>H: 删除 NAT REDIRECT 规则     C<->>S: conntrack 维持交互通道

如果只在某个时间点查看防火墙规则，可能错过临时规则。因此建议将 iptables/nftables 规则变更纳入审计，并关注短时间内新增后删除的 REDIRECT --to-ports 行为。

## 4. 与普通反弹 Shell 的区别

普通反弹 shell 的典型模式是：受影响主机上的进程主动连接远端监听端口，例如通过 bash、sh、python、perl、nc 等程序建立交互通道。

BPFDoor 也可能最终建立 reverse shell，但它不是普通反弹 shell。它更接近“被动触发型后门”：平时不一定主动外联，也不一定监听 TCP 端口，而是通过 raw/packet socket 观察经过主机的数据包。只有收到特定 magic packet 后，才决定是否回连、开放 shell 或临时重定向。


| 对比项           | 普通反弹 shell               | BPFDoor                                              |
| ---------------- | ---------------------------- | ---------------------------------------------------- |
| 通信方式         | 主机主动连接远端监听端口     | 平时被动监听，收到 magic packet 后才动作             |
| 是否暴露监听端口 | 通常不监听，但会立即外联     | 常态下通常不暴露监听端口                             |
| 触发方式         | 命令执行后立即回连           | 特定 TCP/UDP/ICMP magic packet 触发                  |
| 检测重点         | 出站连接、shell 进程、命令行 | raw/packet socket、BPF filter、伪装进程、deleted exe |
| 隐蔽性           | 较低到中等                   | 高                                                   |
| 防火墙影响       | 依赖出站策略宽松             | 可在本机防火墙前看到触发包                           |
| 持久化           | 需要额外机制                 | 程序本身可长期驻留                                   |
| 控制方式         | C2 端口通常相对固定          | 可用正常业务端口触发，甚至临时重定向                 |

普通反弹 shell：

```text
远端监听 4444
    ↑
受影响主机主动连接 1.2.3.4:4444
    ↑
执行 /bin/sh
```

BPFDoor：

```text
控制端发送 magic packet 到受影响主机
    ↓
BPFDoor 用 raw socket 捕获
    ↓
校验 magic / 密码 / 模式
    ↓
按需回连、开放 shell 或临时重定向
```

因此，检测 BPFDoor 不能只找“谁连接了外部 4444 端口”，还需要关注：

```bash
ss -0pb
cat /proc/net/packet
grep -H "packet_recvmsg\|wait_for_more_packets" /proc/*/stack 2>/dev/null
ls -al /proc/*/exe 2>/dev/null | grep deleted
```

## 5. 为什么传统排查容易漏掉

### 5.1 端口扫描不可靠

BPFDoor 常态不需要公开监听端口。即使存在 bind shell，它也可能只在触发后短暂开放。因此外部扫描和 netstat -lntp、ss -lntp 只能发现一部分场景。

### 5.2 进程名伪装

样本会伪装成合法系统服务，例如：

* dbus-daemon
* auditd
* systemd-journald
* hpasmlited
* dockerd
* pickup
* atd

只看 ps 输出中的进程名不足以判断，需要结合 /proc/<pid>/exe、/proc/<pid>/cmdline、/proc/<pid>/cwd、包管理器文件归属和 systemd unit 配置。

### 5.3 Deleted Executable

BPFDoor 可能启动后删除磁盘上的二进制，使 /proc/<pid>/exe 指向 deleted 文件。该行为不是 BPFDoor 独有，但在服务器进程中非常值得关注。

### 5.4 环境变量与历史记录规避

多个公开分析提到样本或交互 shell 会设置：

```text
HISTFILE=/dev/null
MYSQL_HISTFILE=/dev/null
HOME=/tmp
```

这些环境变量单独出现不一定代表异常，但与异常 shell、可疑父进程、外联连接或 deleted executable 同时出现时，应按高危事件处理。

## 6. Rapid7 的排查与防护思路

Rapid7 发布的 rapid7\_detect\_bpfdoor.sh 不是简单 hash IOC 扫描，而是行为导向的 live triage 脚本。它面向 classic 和 modern BPFDoor 变体，覆盖进程伪装、raw/packet socket、BPF magic、内核栈、可疑环境变量、内存驻留、互斥文件、C2 连接、历史端口和持久化位置。

### 6.1 Rapid7 检测逻辑图

flowchart TD     A[开始主机巡检] --> B[枚举进程与 /proc 信息]     B --> C{进程名是否伪装系统服务}     C -- 是 --> C1[校验 exe 路径、cmdline、包归属]     C -- 否 --> D[继续 socket 检查]     C1 --> D      D[检查 packet/raw socket] --> E{是否存在异常 PF\_PACKET/raw socket}     E -- 是 --> E1[关联 PID、进程名、用户、fd]     E -- 否 --> F[检查内核栈]     E1 --> F      F[扫描 /proc/\*/stack] --> G{是否命中 packet\_recvmsg / wait\_for\_more\_packets}     G -- 是 --> G1[标记为强可疑 sniffer/backdoor]     G -- 否 --> H[检查环境变量]     G1 --> H      H[扫描 /proc/\*/environ] --> I{是否存在 HISTFILE=/dev/null 等组合}     I -- 是 --> I1[关联交互 shell 与父进程]     I -- 否 --> J[检查 deleted exe 与 /dev/shm 执行]     I1 --> J      J --> K{是否存在 deleted executable 或临时目录执行}     K -- 是 --> K1[复制内存中二进制并保全证据]     K -- 否 --> L[检查 iptables/nftables 与历史端口]     K1 --> L      L --> M{是否命中多项 BPFDoor 行为}     M -- 是 --> N[隔离主机、取证、扩大排查范围]     M -- 否 --> O[记录低风险发现并持续监控]

### 6.2 Rapid7 重点检查项

#### 6.2.1 进程伪装

Rapid7 脚本会关注进程命令行是否冒充常见系统服务，再对比真实可执行路径。建议重点关注：

* 命令行像系统服务，但真实路径位于 /dev/shm、/tmp、/var/tmp 或异常目录；
* /proc/<pid>/exe 指向 (deleted)；
* 进程名与 systemd unit、包管理器归属不一致；
* 低频系统服务突然创建 raw socket 或对外连接。

#### 6.2.2 Packet/Raw Socket 与 BPF Filter

Rapid7 脚本会检查 ss -0pb、/proc/net/packet、/proc/net/raw、/proc/net/raw6 等信息，用于发现可疑 packet/raw socket。

可使用：

```bash
ss -0pb
cat /proc/net/packet 2>/dev/null
cat /proc/net/raw 2>/dev/null
cat /proc/net/raw6 2>/dev/null
```

正常情况下，合法使用 packet/raw socket 的进程通常包括抓包工具、安全探针、网络监控 agent、容器网络组件或 IDS/IPS。任何伪装系统服务、临时目录执行文件或 deleted executable 使用 packet/raw socket，都应优先调查。

#### 6.2.3 内核栈

后门长期等待 packet socket 数据时，内核栈可能出现：

```text
packet_recvmsg
wait_for_more_packets
```

排查命令：

```bash
grep -H "packet_recvmsg\|wait_for_more_packets" /proc/*/stack 2>/dev/null
```

该命中不是 BPFDoor 独有，合法抓包程序也可能出现类似栈。但如果命中进程同时具备伪装、deleted exe、可疑环境变量或临时目录执行等特征，风险显著升高。

#### 6.2.4 环境变量

检查禁用历史记录的环境变量：

```bash
for p in /proc/[0-9]*; do
  tr '\0' '\n' < "$p/environ" 2>/dev/null | \
  grep -E "HISTFILE=/dev/null|MYSQL_HISTFILE=/dev/null|HOME=/tmp" && echo "$p"
done
```

建议把这些环境变量与父进程、登录来源、TTY、网络连接一起关联。

#### 6.2.5 Mutex 与运行痕迹

早期公开样本曾使用 /var/run/haldrund.pid 等互斥文件。Rapid7 检测脚本也会检查多组历史 mutex 文件名和零字节 pid/lock 文件。

建议重点关注：

* /var/run/\*.pid 或 /run/\*.pid 中无法对应 systemd 服务的文件；
* 零字节 lock 文件；
* 文件创建时间与异常登录、异常进程启动时间接近；
* pid 文件名伪装成系统服务但没有包管理器归属。

#### 6.2.6 历史端口与规则变更

公开分析中多次提到 42391-43390、8000 等历史端口，但端口不是稳定特征。更应关注行为组合：

```bash
iptables-save
nft list ruleset
ss -antupw
```

重点看：

* REDIRECT --to-ports 42391:43390；
* 短时间出现又消失的 NAT PREROUTING 规则；
* 对单个源 IP 特殊放行或重定向；
* 非网络管理进程调用 iptables、nft、ip、tc。

## 7. 主机排查手册

### 7.1 快速排查命令

```bash
# 查找 deleted executable
ls -al /proc/*/exe 2>/dev/null | grep deleted

# 查找 packet socket 相关栈
grep -H "packet_recvmsg\|wait_for_more_packets" /proc/*/stack 2>/dev/null

# 查看 packet/raw sockets
ss -0pb
cat /proc/net/packet 2>/dev/null
cat /proc/net/raw 2>/dev/null
cat /proc/net/raw6 2>/dev/null

# 检查可疑环境变量
for p in /proc/[0-9]*; do
  tr '\0' '\n' < "$p/environ" 2>/dev/null | \
  grep -E "HISTFILE=/dev/null|MYSQL_HISTFILE=/dev/null|HOME=/tmp" && echo "$p"
done

# 检查临时目录可执行文件
find /dev/shm /tmp /var/tmp -maxdepth 2 -type f -perm -111 -ls 2>/dev/null

# 检查 NAT/防火墙规则
iptables-save 2>/dev/null
nft list ruleset 2>/dev/null
```

### 7.2 可疑程度评分建议

flowchart TD     A[发现异常进程] --> B{是否 packet/raw socket}     B -- 否 --> C[继续常规调查]     B -- 是 --> D{是否合法网络工具或安全 agent}     D -- 是 --> E[核对版本、路径、配置、变更记录]     D -- 否 --> F{是否存在 deleted exe 或临时目录执行}     F -- 是 --> H[高危: 立即取证隔离]     F -- 否 --> G{是否伪装系统服务}     G -- 是 --> H     G -- 否 --> I{是否有可疑 env / iptables / 高端口 shell}     I -- 是 --> H     I -- 否 --> J[中危: 保留证据并扩大上下文排查]

## 8. 日志与检测规则方向

建议将以下规则作为组合检测，而不是孤立告警：

* root 用户从 /dev/shm、/tmp、/var/tmp 执行 ELF；
* 非白名单进程创建 PF\_PACKET、SOCK\_RAW 或调用 SO\_ATTACH\_FILTER；
* 进程名伪装系统服务，但真实可执行路径不在标准位置；
* /proc/<pid>/exe 指向 deleted 文件；
* 进程环境变量包含 HISTFILE=/dev/null 或 MYSQL\_HISTFILE=/dev/null；
* 非网络管理工具执行 iptables -t nat、nft、tc 或修改转发表；
* 新增并快速删除 NAT REDIRECT 规则；
* 服务器正常业务端口上出现非协议交互流量；
* 主机主动连接罕见外部 IP 或历史 C2 端口。

网络侧可关注：

* TCP payload 中出现历史 magic 模式，如 0x5293、0x39393939；
* UDP/ICMP payload 中出现历史 magic 模式，如 0x7255；
* ICMP Echo request 携带固定长度异常控制字段；
* 单字节 UDP 状态回包，例如 "1"；
* 对 SSH/HTTPS/其他业务端口的低速人工交互型流量；
* 访问业务端口后，主机出现异常外联或交互通道。

需要注意，网络 IOC 容易被重新编译绕过。实际检测应把网络触发包、主机 packet socket、进程上下文和防火墙规则变更关联起来。

## 9. 事件处置流程

flowchart TD     A[疑似发现 BPFDoor] --> B[不要立即 kill 进程]     B --> C[保全进程、网络、规则、文件证据]     C --> D[隔离主机网络或迁移业务]     D --> E[复制 deleted exe 与内存映射]     E --> F[提取 /proc/pid/environ / fd / maps / stack / cmdline]     F --> G[导出 iptables-save / nft ruleset / ss 连接]     G --> H[分析初始访问与横向移动]     H --> I[按行为特征扩大排查范围]     I --> J[凭据轮换与持久化清理]     J --> K[重装或可信恢复高价值主机]     K --> L[回补检测规则与基线]

### 9.1 现场保全

建议先收集：

```bash
date -u
hostname -f
who -a
w
last -ai
ps -efww
pstree -ap
ss -antupw
ss -0pb
iptables-save
nft list ruleset
```

对可疑 PID 收集：

```bash
PID=<suspicious_pid>

readlink -f /proc/$PID/exe
ls -al /proc/$PID/exe
tr '\0' '\n' < /proc/$PID/environ
tr '\0' ' ' < /proc/$PID/cmdline
cat /proc/$PID/status
cat /proc/$PID/maps
cat /proc/$PID/stack
ls -al /proc/$PID/fd
```

如果 /proc/\$PID/exe 显示 deleted，可以尝试在隔离后复制：

```bash
cp /proc/$PID/exe /root/forensics/pid_${PID}_exe_deleted
sha256sum /root/forensics/pid_${PID}_exe_deleted
```

### 9.2 清除与恢复

如果确认主机存在 BPFDoor，不建议只删除单个进程或文件后继续使用。由于该类程序通常意味着主机已具备高权限执行历史，必须假设存在其他持久化和凭据泄露。

建议：

* 隔离主机，阻断交互通道；
* 保全证据后下线受影响服务；
* 从可信镜像重装高价值服务器；
* 轮换 SSH key、运维账号、数据库账号、API token、CI/CD secret；
* 排查同网段服务器、跳板机、堡垒机、VPN、ESXi、Kubernetes 节点和数据库服务器；
* 审计 cron、systemd、rc.local、profile、authorized\_keys、容器镜像、DaemonSet、initContainer；
* 检查日志缺口、异常清理行为和时间线。

## 10. 加固建议

### 10.1 Linux 主机基线

* 限制 root 直接登录，强制使用可审计的提权路径；
* 对 /dev/shm、/tmp、/var/tmp 尽量启用 noexec，同时评估业务兼容性；
* 对关键目录启用文件完整性监控，例如 /usr/sbin、/usr/bin、/etc/systemd、/etc/cron\*、/etc/sysconfig；
* 监控 /proc/\*/exe deleted 状态；
* 对 raw socket、packet socket、BPF filter 相关行为做采集；
* 记录 iptables/nftables 规则变更审计；
* 对服务器出站连接做最小化控制，降低异常交互通道建立概率。

### 10.2 检测工程化

推荐将检测拆成三层：

flowchart LR     A[主机 telemetry] --> D[关联分析]     B[网络 telemetry] --> D     C[配置与变更审计] --> D     D --> E{是否命中多维行为}     E -- 是 --> F[高危告警]     E -- 否 --> G[低优先级观察]      A1[进程、socket、/proc、环境变量] --> A     B1[magic 包、异常协议、C2 连接] --> B     C1[iptables/nftables、systemd、cron] --> C

单条 IOC 容易误报或漏报。更实用的高置信组合包括：

* PF\_PACKET/raw socket + deleted exe；
* PF\_PACKET/raw socket + 进程名伪装；
* SO\_ATTACH\_FILTER + 临时目录执行；
* iptables REDIRECT + 高端口 shell；
* HISTFILE=/dev/null + 异常网络连接；
* packet\_recvmsg 栈 + 非白名单进程。

## 11. 关键结论

BPFDoor 的威胁模型不是“某个恶意端口”，而是“被动式内核网络层触发后门”。它利用 Linux 合法能力形成低噪声持久访问：BPF 做过滤，raw/packet socket 做被动监听，进程名伪装降低人工排查概率，iptables 临时重定向混淆真实连接，deleted executable 和禁用历史记录减少取证痕迹。

Rapid7 的检测思路值得借鉴：围绕行为组合做 live triage，而不是只扫 hash 或固定 magic byte。生产环境中应将这些检查固化为日志规则、终端检测策略、Linux 基线巡检、高价值服务器专项排查和事件响应手册。

优先级最高的落地项是：

1. 建立合法 packet/raw socket 进程白名单。
2. 监控 deleted executable、临时目录执行和伪装系统服务。
3. 审计 iptables/nftables 规则变更。
4. 关联 packet\_recvmsg 内核栈、可疑环境变量和网络连接。
5. 对电信、边界服务、VPN、堡垒机、数据库、虚拟化和容器节点开展专项排查。
