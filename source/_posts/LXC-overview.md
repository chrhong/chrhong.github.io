---
title: LXC overview
date: 2018-03-21 14:17:06
tags:
    - Cgroups
    - Namespace
    - SR-IOV
categories:
    - LXC
---
## 一、什么是 LXC

LXC(Linux Container, Linux 容器) 是一套为内核容器功能提供的用户空间的接口。通过这些功能强大的 API 和简易的脚本工具，Linux 用户可以轻易的创建和管理系统以及应用的容器。容器技术其实是一种资源隔离技术，主要使用 chroot, cgroups, namespace 等技术实现轻量级的系统级的虚拟化。不同于传统的虚拟化技术，需要虚拟软件构建虚拟层，上层虚拟机的所有动作都要转化为底层宿主机的指令。容器的开销小，创建快，性能好，正在越来越广泛的被应用，近几年火爆起来的 Docker 项目就是基于 LXC 发展起来的。

![LXC](lxc.png)<!-- more -->

Cgroups(Control groups，控制组) 是由 Linux 内核提供的一种可以限制、记录和隔离进程组使用的物理资源（如CPU、内存、IO等）的工具。其最早由 Google 工程师提出，后来集成到 Linux 内核中。Cgroups 是 LXC 实现虚拟化的资源管理工具，可以说没有 Cgroups 就没有 LXC。

![Cgroups](cgroups.png)

Namespace 是一种全局系统资源的封装隔离工具。不同的命名空间的进程可以具有独立的全局系统资源。在一个命名空间中更改系统资源只会影响当前命名空间的进程，其他命名空间中的进程将不受影响。以下图中的进程 PID 为例，我们都知道 PID 在系统中是唯一的，不存在两个相同 PID 的进程，而在不同的容器则可以拥有同样的 PID 的进程，因为对于每个容器内部来说，自己就是一个独立的系统。

![Namespace](namespace.png)

目前 Linux 一共实现有六种不同类型的 Namespace:

|Name   |Macro              |Isolated resources|
|:-:    |:-:                |:-|
|cgroup |CLONE_NEWCGROUP    |cgroup root directory (since Linux 4.6)|
|IPC    |CLONE_NEWIPC       |System V IPC, POSIX message queues (since Linux 2.6.19)|
|Network|CLONE_NEWNET	    |Network devices, stacks, ports, etc. (since Linux 2.6.24)|
|Mount 	|CLONE_NEWNS	    |Mount points (since Linux 2.4.19)|
|PID	|CLONE_NEWPID	    |Process IDs (since Linux 2.6.24)|
|User	|CLONE_NEWUSER	    |User and group IDs (started in Linux 2.6.23 and completed in Linux 3.8)|
|UTS	|CLONE_NEWUTS	    |Hostname and NIS domain name (since Linux 2.6.19)|

## 二、如何使用 LXC

### 1. 安装
#### 检查安装条件：
1. Linux kernel(Linux kernel >= 2.6.32, ….)
2. Lib dependency

#### 安装:
以 ubuntu 为例, 你可以可以使用 `apt-get` 工具安装： `sudo apt-get install lxc`

更多的的细节可以参照 [LXC - Get started](https://linuxcontainers.org/lxc/getting-started/)。

### 2. 使用
LXC 提供多种方式来使用它，包括 liblxc 库，支持 Python, Go 等多种语言的 API，还有一套标准的命令行工具来创建和管理 LXC。
此处我们仅介绍最简单的 `lxc-*` 工具的使用。

#### 容器的创建

* 创建一个容器
```
lxc-create {-n name} [-f config_file] {-t template} [-B backingstore] [-- template-options]
# 创建一个存储配置信息和用户信息的系统对象。name 用来标识容器，以使用不同的 lxc-* 命令。
# 这个对象以文件夹的形式存放在 /var/lib/lxc 目录下，并且以 name 字段来区分
```

#### 容器的启动

* 启动一个处于停止状态的容器
```
lxc-start {-n name} [-f config_file] [-c console_device] [-L console_logfile] [-d] [-F] [-p pid_file] [-s KEY=VAL] [-C] [--share-[net|ipc|uts] name|pid] [command]

# 如果 command 没有指定，那么就执行默认的 /sbin/init
# 如果 command 指定，则启动容器之后就运行指定的程序
# TODO: 容器一定要在 stopped 状态吗？命令结束容器会自动停止 ？
```
![](lxc-create.png)

#### 容器的运行

* 在一个运行状态的容器里面启动一个程序。
```
lxc-attach {-n name} [-- command] 
#如果 command 没有指定，那么就会启动默认的 `shell`
```

* 快速启动一个容器。
``` 
lxc-execute {-n name} [-f config_file] [-s KEY=VAL] [-- command]
# 如果 command 结束退出，则容器就会结束销毁
```

* 检查存在的容器
```
lxc-ls [-1] [--active] [--frozen] [--running] [--stopped] [--defined] [-f] [-F format] [-g groups] [--nesting=NUM] [--filter=regex]
```

* 检查一个容器的信息
```
lxc-info {-n name} [-c KEY] [-s] [-p] [-i] [-S] [-H]
```

#### 容器的销毁

* 停止一个容器
```
lxc-stop {-n name} [-W] [-r] [-t timeout] [-k] [--nokill] [--nolock]
```

* 等待容器到达指定的状态之后退出
```
lxc-wait {-n name} {-s states}
```

* 销毁一个容器并删除它的文件系统（rootfs 文件夹）
```
lxc-destroy {-n name} [-f] [-s]
```

* 直接删除容器所在的文件夹

#### 容器的配置

* 指定容器的 Hostname
```
lxc.uts.name = 0x1234
```

* 指定一个具有挂载信息的文件位置
```
lxc.mount.fstab = /var/lib/lxc/fstab
```

* 指定一个挂载点
```
lxc.mount.entry  = /ram/node_0xe00a/opt /opt none bind 0 0
```

* 是否使用自动挂载的 `/dev`
```
lxc.autodev = 0
# 如果不想 LXC 启动时只挂载和填充最小的 /dev, 需要将此项设置为 0
# 如果没有设置，默认为 1，很多 Host 上的设备将在容器中不可见
```

* 指定一个文件夹作为根文件系统
```
lxc.rootfs.path = /var/lib/lxc/rootfs
# 可以是一个镜像, 文件夹或者块设备。 如果没有指定，容器将共享 Host 上的根文件系统
```

* 限定容器可用的 CPU
```
lxc.cgroup.cpuset.cpus = 0,1
```

* 指定一种网络虚拟化方式
```
lxc.net.[i].type = none/empty/veth/vlan/macvlan/phys
```

* 指定真实要使用的网卡接口
```
lxc.net.[i].link
```

* 其他网络配置选项
```
lxc.net.[i].flags = up
lxc.net.[i].mtu
lxc.net.[i].name
lxc.net.[i].hwaddr
lxc.net.[i].ipv4.address
lxc.net.[i].ipv6.address
```

## 三、容器中的网络虚拟化

####  Linux Bridge
A bridge is a technology that implements relay at the link layer and forwards frames. According to MAC partition blocks, it can isolate collisions and network devices that connect multiple network segments of the network at the data link layer.

![Linux Bridge](linuxbridge.png)

#### Single Root I/O Virtualization(SR-IOV)
SR-IOV introduced two functional type:
• PF (Physical Function) : own all PCIe devices resources, mainly used for config and manage SR-IOV.
• VF (Virtual Function) : “simplified” PCIe function, including resource of data transferring, could be created and destroyed by PF.
SR-IOV allows a PCIe device to appear as multiple separate physical PCIe devices, which solve the problem of exclusive ownership of physical resources.

![SR-IOV](SR-IOV.png)
