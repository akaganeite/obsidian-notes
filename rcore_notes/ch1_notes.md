# 库操作系统
LibOS 以函数库的形式存在，为应用程序提供操作系统的基本功能。
# 建立LibOS，RUST需要实现什么
- 通过sbi_call和硬件交互(关机，打印)
- panic_handler
- println
- 内嵌内核启动代码

# QEMU启动
1. 固化在 Qemu 内的一小段汇编程序负责
	Qemu CPU 的程序计数器（PC, Program Counter）会被初始化为 `0x1000` 
	```c
0x1000: auipc   t0,0x0
0x1004: addi    a2,t0,40
0x1008:      csrr    a0,mhartid
0x100c:      ld      a1,32(t0)
0x1010:      ld      t0,24(t0)
0x1014:      jr      t0
```
	执行几条指令后跳转至`0x80000000`，物理地址的起始位置
2. 执行bootloader，加载内核镜像，对计算机进行一些初始化工作，并跳转到下一阶段软件的入口。即`0x80200000`.上述工作由RUST-SBI完成
3. 开始执行内核第一条指令

# ch_1各个文件作用
entry.asm:内核第一个指令，衔接。分配启动栈空间，sp设置为栈顶
linker.ld:调整内核内存布局，定义内核起始地址与布局
	`BASE_ADDRESS = 0x80200000;`
lang_items.rs :实现panic！处理
sbi.rs:实现sbi服务接口函数，通过sbi_call可以和硬件进行交互
console.rs:实现格式化输出print，println宏
logger.rs:调用log crate实现分级输出

# question
1. 为什么没有操作系统，没有syscall，可以使用crate log
