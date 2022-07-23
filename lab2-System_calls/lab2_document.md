# System Call

## 预备知识

将硬件资源抽象为服务，将各个进程隔离；RISC-V的CPU可运行在`machine`模式，`supervisor`模式，`user`模式，`ecall`指令可以切换模式

内核的组织存在两种，为`monolithic kernel`和`microkernel`,前者设计更为方便，操作系统内核的各部分协作更方便，但对于使用者来说更易于出错，后者最小化执行在`supervisor`模式的代码，将大体量的操作系统执行在用户模式，xv6采用monolithic kernel.

进程抽象使程序拥有独立的地址空间，并给程序提供拥有整个硬件资源的假象，xv6使用页表给进程提供独占的地址空间，RISC-V的页表将虚拟地址`virtual address`转换为物理地址`physical address`

对于64位的RISC-V指令集，xv6使用38位作为寻址范围，最大地址为0x3fffffffff,在xv6的地址空间的顶端有在用户空间和内核空间之间跳转的页表*trampoline*和*trapframe*,前者包含进出内核的代码

xv6内核将进程的有关信息保存在结构体`struct proc`中，每个进程包含一个执行线程，线程的信息存储在栈中，每个进程包含两个栈，用户栈和内核栈，进程执行ecall指令，提升硬件特权级，进入内核态执行系统调用，`sret`指令降低硬件特权级并返回到用户指令继续执行。

RISC-V计算机启动后会首先加载*boot loader*,引导装载程序