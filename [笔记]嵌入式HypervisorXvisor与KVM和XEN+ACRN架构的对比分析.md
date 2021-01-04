# 目录

[TOC]

# 摘要

开源的hypervisor：Xen，linux KVM，OKL4 Microvisor

本文：介绍Xvisor（eXtensible Versatile hypervisor），从对整个系统性能的影响上，与两个常用虚拟机管理器KVM和Xen进行对比

环境：ARM、linux

结论：在ARM处理器架构上，Xvisor具有**更低的CPU开销**，更高的**内存带宽**，更低的锁同步延迟和**虚拟定时器中断开销**，并且能够全面提升嵌入式系统性能。

# 1.介绍

*（这一部分是原论文的介绍，在原文基础上，将ACRN也做相应的比较）*

可行性分析：

​	1. 本次研究基于ARM架构。这是由于最新的ARM处理器架构已经提供了**硬件虚拟化扩展**，并且ARM处理器在嵌入式系统中已经得到了广泛应用。另外，**这3个Hypervisor共有的的大部分板级支持包（BSP）都使用了ARM架构处理器。**

​	2.KVM和Xen被选中作为对比的虚拟机管理器，是基于如下原因：

​		o  **ARM架构支持;**

​		o  允许我们没有任何限制的收集性能数据的**开源特性**；

​		o  与Xvisor支持同样的单板；

​		o  在**嵌入式系统**中得到了应用。

时间约束测试：

​	o  Hypervisor的**低内存**和CPU开销；

​	o  Hypervisor**最小负荷情况下**的客户机调度效率。

# 2.虚拟化技术分类

![虚拟化分类](https://github.com/qijiuliushisan/images/image-20210104125148828.png)

## 2.1 Hypervisor设计

### 	1.完全宏内核设计

完全宏内核Hypervisor使用一个**单一的软件层**来负责主机硬件访问，CPU虚拟化和客户机IO模拟。例如**Xvisor**和**VMware ESXi Server**

### 	2.部分宏内核设计

通用目的宏内核操作系统的**扩展**。他们的操作系统内核支持主机硬件访问和CPU虚拟化，并通过用户空间软件支持客户机IO模拟。例如**Linux KVM**和**VMware Workstation**。

### 	3.微内核设计

提供基本的主机硬件访问和CPU虚拟化功能，依赖于一个**管理VM**来支持整个主机的硬件访问、客户机IO模拟和其他服务。例如**Xen**，**微软Hyper-V**，**OKL4 Microvisor**和**INTEGRITY Multivisor**。

## 2.2 虚拟化模式

### 	1.全虚拟化

允许未经修改的客户机操作系统作为客户机运行。（**译者注**：该模式需要借助硬件的虚拟化支持，例如X86架构AMD-V/Intel VT，ARMv8和Power架构的虚拟化profile等。）

### 	2.半虚拟化

通过提供Hypercall（虚拟机调用接口）来进行各种**IO操作**（例如，网络收发，块读写，终端读写等等）。

# 3. **嵌入式系统的开源Hypervisor**

## 3.1 XEN

![XEN架构](https://github.com/qijiuliushisan/images/image-20210104131455362.png)

特性：**支持全虚拟化、半虚拟化，微内核，轻量级内核**

功能：**CPU虚拟化、MMU虚拟化、虚拟中断处理、客户机间通讯**

**Domain**（域）是Xen内核相应于虚拟机或客户机的概念。

**Dom0**运行着一个Linux内核的修改版本，利用Linux内核提供**IO虚拟化**和**客户机管理**服务。优先级最高，对主机硬件有着**完全访问权限**。使用运行在Dom0**用户空间**的**Xen工具栈**来实现管理客户机的用户接口。

**DomU**运行Guest。

**I/O处理**：

- **半虚拟化客户机**：**客户机的IO事件模拟和半虚拟化**通过DomU和Dom0之间的通讯来实现。使用**Xen事件通道**来完成。
- **全虚拟化客户机**：使用运行在Dom0用户空间中的QEMU来模拟客户机IO事件。

**优势**：重用Linux内核现有的设备驱动和其他部分

**劣势**：Dom0只是Xen的另一个域，有它自己的嵌套页表，并且可能被Xen调度器调度出去。

## 3.2 KVM

![KVM架构](https://github.com/qijiuliushisan/images/image-20210104133149204.png)

**特性**：支持**全虚拟化技术**和**半虚拟化技术**的**部分宏内核**Hypervisor，以可选的VirtIO设备形式提供半虚拟化支持。

​			**客户机模式**：允许客户机操作系统与主机操作系统运行在相同的执行模式下

**I/O处理**：HOST将虚拟机视作一个**QEMU进程**，I/O访问将陷入到主机Linux内核。KVM在HOST上虚拟化CPU，依赖于运行在用户空间的QEMU来处理客户机IO事件。

**KVM包含两个组件**：

1. 内核空间字符设备驱动，通过一个字符设备文件/dev/kvm提供CPU虚拟化服务和内存虚拟化服务
2. 提供客户机硬件模拟的用户空间模拟器（如qemu）

​    **IOCTRL**：实现与这两个组件之间的服务请求通讯，例如虚拟机和vCPU的创建

**优势**：重用Linux内核现有的设备驱动和其他部分

**劣势**：KVM的 客户机模式到主机模式 的 虚拟机切换 依赖于 嵌套页表故障机制，特殊指令陷阱（Trap），主机中断，客户机IO事件，和另一个从主机模式唤醒客户机执行的虚拟机切换，导致了KVM整体性能本质上的降低。

## 3.3 **Xvisor**

![image-20210104140952732](https://github.com/qijiuliushisan/images/image-20210104140952732.png)

**特性**：支持全虚拟化和版虚拟化技术的**完全宏内核**Hypervisor，以**VirtIO设备**的形式**提供半虚拟化支持**。**轻量级**

​			Xvisor的所有核心组件，例如**CPU虚拟化**，**客户机IO模拟**，**后端线程**，**半虚拟化服务**，**管理服务**和**设备驱动**，都作为一个独立的软件层运行，不需要任何必备的工具或者二进制文件。

**Orphan vCPUs**：没有分配给某个客户机的vCPU。运行所有设备驱动和管理功能的后端处理。

**客户机配置信息**：以设备树(Device Tree)的形式维护。

**优势**：

​		最高特权等级的**单一软件层**提供所有虚拟化相关服务。上下文切换非常**轻量级**。嵌套页表、特殊指令陷阱、主机中断和客户机IO事件等的处理也非常快速。

​		所有设备驱动都作为Xvisor的一部分直接运行，具有完全的特权并且没有嵌套页表。

​		Xvisor的vCPU调度器是基于单CPU的，不处理多核系统的负载均衡。

**劣势**：缺少Linux那样丰富的单板和设备驱动

## 3.4 ACRN

![ACRN架构](https://github.com/qijiuliushisan/images/image-20210104163121167.png)

**特性**：VMXVirtual Machine Extension）是Intel 64和IA-32架构处理器级别的功能，用于支持虚拟化。理论上支持全虚拟化，通过vitio实现半虚拟化。可以看出是**微内核**的Hypervisor。对设备的虚拟化由Hypervisor实现，设备驱动模块在SVM上。

​		实时、安全。



## 4.客户机IO事件模拟

## 4.1 Xen ARM

![Xen ARM客户机IO事件流](https://github.com/qijiuliushisan/images/image-20210104144715382.png)

**DomU发起IO：**

1. Xen Hypervisor监测到IO事件
2. Xen事件通道转发
3. Dom0内核接收到Xen事件
4. Dom0由内核态转换成用户态，QEMU模拟IO事件
5. HYP唤醒DomU

**开销来源**：红框所圈的**xen事件通道**和**由内核态向用户态的上下文切换**增加了IO的开销。

## 4.2 KVM ARM

![KVM ARM的IO事件流](https://github.com/qijiuliushisan/images/image-20210104144852480.png)



**客户机发起IO：**

1. 客户机IO事件引起一个VM-Exit事件，引起KVM从客户机模式切换到主机模式。
2. 主机内核态
3. 主机由内核态转换为用户态，交给QEMU处理客户机IO事件。
4. VM-enter发生，引起KVM从主机模式切换到客户机模式。

**开销来源**：VM-exit和Vm-enter上下文切换

## 4.3 Xvisor ARM

![Xvisor ARM的IO事件流](https://github.com/qijiuliushisan/images/image-20210104145449219.png)

**客户机发起IO：**

1. Xvisor ARM捕获客户机IO事件
2. 事件在标志2处的不可睡眠的通用上下文中被处理以确保时间被处理（**是架构上的孤儿vcpu吗？**）

## 4.4 ACRN

![image-20210104194232822](https://github.com/qijiuliushisan/images/image-20210104194232822.png)

**SVM的IO流：**ACRN只捕获其对中断设备的访问（红框），其他I / O访问直接进入物理设备（绿框）**。中断设备包括可编程中断控制器（PIC），I / O高级可编程中断控制器（IOAPIC）和本地高级可编程中断控制器（LAPIC）。

*虚拟化中断设备的好处：每个VM可以任意配置虚拟中断设备，而不必担心中断向量冲突。此外，所有虚拟中断设备都在hypervisor中进行仿真，以实现快速响应。*

**UVM的IO流**：非直通设备：①通过1~3步将I / O访问转发到hypervisor内虚拟中断设备 或 ②通过1~5步转发到SVM中，创建一个I/O请求，并将其发送到“ Virtio和虚拟机管理程序服务模块（VHM）”中的I / O调度程序，该模块是提供远程服务的SOS内核模块。

**RTVM的IO流**：一般都是直通设备。

# 5.主机中断

## 5.1 Xen ARM

![Xen ARM的中断处理](https://github.com/qijiuliushisan/images/image-20210104150400563.png)

**中断处理**：Dom0

**流程**：

1. Xen内核捕捉到主机IRQ（中断请求）的触发
2. 由HYP路由到Dom0
3. 切换到Dom0内核态
4. Dom0处理中断

*注意：如果一个主机中断在DomU运行时被触发，那么它将在Dom0被调度进来后才能得到处理*

## 5.2 KVM ARM

![KVM ARM的主机中断处理流程](https://github.com/qijiuliushisan/images/image-20210104150929036.png)

**流程：**

1. 主机IRQ触发VM-exit
2. 主机进入内核态
3. 主机IRQ处理
4. VM-entry恢复客户机

**开销：**VM-exit和VM-entry增加了相当大的主机中断处理开销。如果主机中断被转发到KVM客户机，那么调度开销也会存在。

## 5.3 Xvisor ARM

![Xvisor ARM主机IRQ的处理](https://github.com/qijiuliushisan/images/image-20210104155243465.png)

Xvisor的主机设备驱动通常作为Xvisor的一部分以最高权限运行。处理主机中断时是不需要引发调度和上下文切换开销的。只有当主机中断被转发到一个当前没有运行的客户机时，才会引发调度开销。

## 5.4 ACRN

![image-20210104194825067](https://github.com/qijiuliushisan/images/image-20210104194825067.png)

**SVM的IRQ流程**：物理中断由Hypervisor的中断分配器转发给虚拟中断控制器。

**UVM的IRQ流程**：一种是由中断分配器进行分配，一种是经由SVM处理后，交由中断分配器转发。

**RTVM**：只处理MSI

# 6. 锁同步延迟

**产生原因**：**Hypervisor调度器**和**客户机OS调度器**互相意识不到对方，导致客户机vCPU被Hypervisor随意抢占。

**vCPU抢占问题**：当一个运行在持有锁的某主机CPU（pCPU0）上的vCPU（vCPU0）被抢占，而同时另一个运行在其他主机CPU（pCPU1）上的vCPU（vCPU1）正在等待这个锁，那么vCPU抢占问题就会发生。

![vCPU抢占问题](https://github.com/qijiuliushisan/images/image-20210104160327760.png)

**vCPU堆积问题**：发生在一个运行着多个vCPU的单主机CPU上的锁调度冲突问题也会导致vCPU堆积问题发生。也就是说，希望获取某个锁的vCPU（vCPU1）抢占了运行在同一个主机CPU上的vCPU（vCPU0），但是vCPU0正在持有这个锁。

![vCPU堆积问题](https://github.com/qijiuliushisan/images/image-20210104160359108.png)

*在ARM机构上，操作系统典型的使用WFE（等待事件）指令来等待请求一个锁，并使用SEV（发送事件）指令来释放一个锁。ARM架构允许WFE指令被Hypervisor捕获，但是SEV指令不能被捕获。*

**堆积问题的解决方案**：所有3种Hypervisor（Xen ARM，KVM ARM和Xvisor ARM）都使用捕获WFE指令的方法使得vCPU让出时间片。

**vCPU抢占问题的解决方案**：通过使用半虚拟化锁的方式来解决，但是需要对客户机操作系统进行源码级的修改。

# 7.内存管理

ARM架构提供**2级翻译表**（或者说**嵌套页表**），用于客户机内存虚拟化,即图13所示的2阶段MMU。客户机操作系统负责编程第1阶段页表，将客户机虚拟地址(GVA)翻译到间接物理地址（IPA）。ARM Hypervisor负责编程第2阶段页表来从将间接物理地址（IPA）翻译成实际物理地址（PA）。

![image-20210104161038934](https://github.com/qijiuliushisan/images/image-20210104161038934.png)

比如最糟糕的情况下，N级第1阶段翻译表和M级第2阶段翻译表需要NxM次内存访问。对任何虚拟化系统上的客户机来说，TLB-miss损失都是非常昂贵的。为了减少2阶段MMU中的TLB-miss损失，**ARM Hypervisor在第2阶段创建更大的页。**

## 7.1 Xen ARM

Xen ARM为每个客户机或域（Dom0或DomU）创建一个独立的**3级第2阶段翻译表**。Xen ARM能创建4K字节，2M字节或1G字节的第2阶段翻译表项。Xen ARM也按需分配客户机内存，并试图基于IPA和PA对齐构造尽可能最大的第2阶段翻译表项。

## 7.2  KVM ARM

KVM用户空间工具（**QEMU**）预先分配作为客户机RAM使用的用户空间内存，并向KVM内核模块通知其位置。KVM内核模块为**每个客户机vCPU创建一个独立的3级第2阶段翻译表**。典型的，KVM ARM将创建4K字节大小的第2阶段翻译表项，但是也能够使用巨大化TLB优化模式创建2M字节大小的第2阶段翻译表项。

## 7.3 Xvisor ARM

Xvisor ARM在客户机创建时，预先分配连续的主机内存以做为客户机RAM。它为每个客户机创建一个独立的3级第2阶段翻译表。Xvisor ARM能创建4K字节，2M字节或1G字节的第2阶段翻译表项。另外，Xvisor ARM总是基于IPA和PA对齐创建尽可能最大的第2阶段翻译表项。最后，客户机RAM是**扁平化和连续**的（不像其它Hypervisor）。这有助于缓存预取访问，从而进一步提升客户机内存访问性能。

# *参考*

*ACRN a big little hypervisor for IoT development*

嵌入式HypervisorXvisor与KVM和XEN的对比分析(中文翻译)





















































