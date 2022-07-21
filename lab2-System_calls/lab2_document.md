# System Call

## 预备知识

将硬件资源抽象为服务，将各个进程隔离；RISC-V的CPU可运行在`machine`模式，`supervisor`模式，`user`模式，`ecall`指令可以切换模式