## 应用程序在执行过程中，会占用哪些计算机资源？
1. CPU
2. 内存
## 给出应用程序A的代码段/数据段/堆/栈的地址空间范围
### 段数据
格式：
	Name              Type             Address           Offset
	Size              EntSize          Flags  Link  Info  Align
代码段：
[14] .text             PROGBITS         0000000000018090  00018090
       0000000000123873  0000000000000000  AX       0     0     16
数据段：
[16] .rodata           PROGBITS         000000000013c000  0013c000
       000000000000de98  0000000000000000   A       0     0     16
[28] .data             PROGBITS         0000000000187000  00186000
       0000000000000038  0000000000000000  WA       0     0     8
[29] .bss              NOBITS           0000000000187038  00186038
       0000000000000150  0000000000000000  WA       0     0     8

### 堆栈
需要在程序运行时查看，cat /proc/[pid]/maps
堆栈信息：<start_address>-<end_address>  perms          offset         dev      inode         pathname
堆栈数据： 55f9c37b9000-55f9c51b2000        rw-p        00000000    00:00        0            [heap]
				   7ffe4ab84000-7ffe4aba6000         rw-p        00000000    00:00        0            [stack]
- perms 表示内存分段的访问权限，如 r（可读）、w（可写）、x（可执行）等。
- offset 表示该内存分段在文件中的偏移量，如果不是文件映射则为 0。
- dev 表示设备号，通常是一个主设备号和次设备号的组合。
- inode 表示节点号，对于文件映射，表示文件在文件系统中的索引节点号。
- pathname 表示映射到内存中的文件路径，如果没有映射文件则为 [anon]

## 请简要说明应用程序与操作系统的异同之处
### 相同
都是程序，运行在计算机上，存储在磁盘中
### 不同
OS要负责和硬件交互，应用程序只要和用户以及OS交互即可

## 说明RISC-V硬件加电后的几条指令在哪里？完成了哪些功能？
在0x1000
```c
 0x1000:      auipc   t0,0x0
 0x1004:      addi    a2,t0,40
 0x1008:      csrr    a0,mhartid
 0x100c:      ld      a1,32(t0)
 0x1010:      ld      t0,24(t0)
 0x1014:      jr      t0
```
jr跳转至RUSTSBI的第一条指令，在0x80000000

## RISC-V中的SBI的含义和功能是啥？
RISC-V SBI is short for RISC-V Supervisor Binary Interface. SBI acts as an interface to environment for your operating system kernel. An SBI implementation will allow furtherly bootstrap your kernel, and provide an environment while the kernel is running.

More generally, The SBI allows supervisor-mode (S-mode or VS-mode) software to be portable across all RISC-V implementations by defining an abstraction for platform (or hypervisor) specific functionality.

## 为了让应用程序能在计算机上执行，操作系统与编译器之间需要达成哪些协议
- 程序库、依赖库等文件信息：需要约定好 OS 中各种库存放的位置，以方便应用程序调用。
- 二进制文件格式协议：需要约定可执行文件以什么格式存储，如 ELF（Executable and Linkable Format）就是一种常见的格式。
- 系统调用接口协议：要约定应用如何使用各种系统调用，在编译时写入对应操作系统的相关代码。
- 内存管理协议：需要约定内存管理的规范，编译器需要获知应用程序运行时分配得到的内存地址范围等信息。
- 进程管理协议：需要约定进程管理的模型，编译器需要据此来生成汇编代码控制应用的运行、发起子进程等等。
- 文件系统协议：需要约定应用程序如何访问文件系统，编译器通过约定的方式生成汇编代码对文件系统进行访问。
- 异常处理协议：需要约定如何处理应用程序运行时出现的各种异常。
## 说明从QEMU模拟的RISC-V计算机加电开始运行到执行应用程序的第一条指令这个阶段的执行过程
1. 设备加电，QEMU CPU的PC寄存器被初始化为0x1000.
2. CPU执行0x100的程序，跳转至物理地址0x80000000
3. bootloader/rustsbi-qemu.bin位于该地址，执行初始化操作并把os镜像从硬盘加载至内存
4. 跳转至0x80200000，将控制权交给内核，开始执行内核的第一条指令
5. 内核设置栈空间，跳转至OS的main函数

## 为何应用程序员编写应用时不需要建立栈空间和指定地址空间？
程序员面向虚拟地址编程，每个进程都是从逻辑地址0开始，实际使用时会从虚拟地址转至物理地址。

## 调试器可以如何复原出最顶层的几个调用栈信息？
1. 当前pc为flip函数开头，正在执行flip，sp尚未做出改变，1300-1310将会是flip的栈
2. 当前ra为flap(1)函数体，flap(1)调用flip；1310-1320是flap(1)的栈，栈中存有0x10750 <flap+32>
3. 0x10750 <flap+32>是flap(1)的返回地址，指向flap函数，所以flap(1)的调用者是自己，即flap(2)
4. 以此类推 栈空间为1320-1330的flap(2)被flip调用
5. ... ...