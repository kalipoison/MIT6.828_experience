

实验分为三个部分：

- 熟悉汇编语言、QEMU x86模拟器、PC上电启动过程
- 检查我们的6.828内核的boot loader程序，它位于`lab`的`boot`目录下。
- 深入研究6.828内核本身的初始模板，位于`kernel`目录下。

# PART1: PC Bootstrap

```shell
gohb@gohb-virtual-machine:~$ mkdir ~/6.828
gohb@gohb-virtual-machine:~$ cd 6.828/
gohb@gohb-virtual-machine:~/6.828$ git clone https://pdos.csail.mit.edu/6.828/2018/jos.git lab
```

编译代码

```
gohb@gohb-virtual-machine:~/6.828/lab$ make
```

如果看到

![image-20220212005317686](../pic/image-20220212005317686.png)

就说明编译成功了。

```
gohb@gohb-virtual-machine:~/6.828/lab$ make qemu
```

![image-20220212005709722](../pic/image-20220212005709722.png)

#### PC的物理地址空间

PC 的物理地址空间是硬连线的，具有以下一般布局：

```

+------------------+  <- 0xFFFFFFFF (4GB)
|      32-bit      |
|  memory mapped   |
|     devices      |
|                  |
/\/\/\/\/\/\/\/\/\/\

/\/\/\/\/\/\/\/\/\/\
|                  |
|      Unused      |
|                  |
+------------------+  <- depends on amount of RAM
|                  |
|                  |
| Extended Memory  |
|                  |
|                  |
+------------------+  <- 0x00100000 (1MB)
|     BIOS ROM     |
+------------------+  <- 0x000F0000 (960KB)
|  16-bit devices, |
|  expansion ROMs  |
+------------------+  <- 0x000C0000 (768KB)
|   VGA Display    |
+------------------+  <- 0x000A0000 (640KB)
|                  |
|    Low Memory    |
|                  |
+------------------+  <- 0x00000000
```

基于 16 位 Intel 8088 处理器的第一台 PC 只能处理 1MB的物理内存。因此，早期 PC的物理地址空间将从 0x00000000 开发，但以 0x000FFFFF 而不是 0xFFFFFFFF 结束。 标有 “Low Memory” 的 640KB区域是早期 PC 可以使用的唯一随机存取存储器（RAM）；事实上，最早的 PC 只能配置 16KB, 32KB 或者 64KB 的 RAM。

从 0x000A0000 到 0x000FFFFF 的 384KB 区域 由硬件保留用于特殊用途，例如视频显示缓冲区和保存在非易失性存储器中的固件。这个保留区域中最重要的部分是 基本输入/输出系统（BIOS），它占据了从 0x000F0000 到 0x000FFFFF 的 64KB 区域。在早期的PC中，BIOS 保存在真正的只读存储器（ROM）中，但当前的 PC 将 BIOS 存储在可更新的闪存中。BIOS 负责执行基本的系统初始化，例如激活显卡和检查安装的内存量。执行完初始化后，BIOS 从软盘、硬盘、CD-ROM 或网络等适当位置加载操作系统，并将机器的控制权交给操作系统。

当 Intel 最终用 80286 和 80386 处理器“打破 1MB 障碍”，分别支持 16MB 和 4GB 物理地址空间时，PC 架构师仍然保留了低 1MB 物理地址空间的原始布局，以确保向后兼容现有软件。因此，现代 PC 在物理内存中从 0x000A0000 到 0x00100000 有一个“洞”，将 RAM 分为“低”或“常规内存”（前 640KB）和“扩展内存”（其他所有内存）。此外，PC 的 32 位物理地址空间最顶端的一些空间，尤其是物理 RAM，现在通常由 BIOS 保留，供 32 位 PCI 设备使用。

最近的 x86 处理器可以支持 *超过*4GB 的物理 RAM，因此 RAM 可以进一步扩展到 0xFFFFFFFF 以上。在这种情况下，BIOS 必须安排在系统 RAM 中的 32 位可寻址区域顶部留出*第二个孔，以便为这些 32 位设备的映射留出空间。*由于设计限制，JOS 无论如何都只会使用 PC 的前 256MB 物理内存，所以现在我们假设所有 PC “只有”一个 32 位物理地址空间。但是处理复杂的物理地址空间和多年来演变的硬件组织的其他方面是操作系统开发的重要实际挑战之一。

#### ROM BIOS

打开两个终端窗口和两个 shell。在一个中，输入make qemu-gdb（或make qemu-nox-gdb）。这会启动 QEMU，但 QEMU 在处理器执行第一条指令之前停止并等待来自 GDB 的调试连接。在第二个终端中，从您运行`make`的同一目录中运行make gdb。

![image-20220212174932398](../pic/image-20220212174932398.png)

![image-20220212175056646](../pic/image-20220212175056646.png)
