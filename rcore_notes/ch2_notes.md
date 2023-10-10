# 代码中的问题
1. buf: *const u8 rust指针问题
2. asm!, 嵌入汇编
3. 汇编中，.align 3 , csrr
4. RefCell
5. lazy_static 
## refcell与内部可变性
内部可变性，让程序拥有多个共享引用的时候还可以修改引用的数据。比如对外看起来是不变的接口，内部仍然有改变。

# code-tree
- src/内核相关code
	- batch.rs:设置用户栈，内核栈：保存寄存器（trapcontext）
		- appmanager：被封装在upsafecell里面，是lazy_static的.
			方法：loadapp，做一个memcpy，程序加载到0x80400,若所有程序已运行结束，直接调用sbi的shutdown，关闭os
		- 函数：==run_next_app==，__由main.rs直接调用__。调用loadapp装载下一个app，init Trapcontext设置sstatus，sp(user stack)，sepc(0x8040)。调用`__restore`去执行app
	- sync/：实现upsafecell，包装appmanager
	- trap/： `__alltraps` 将 Trap 上下文保存在内核栈上，跳转到trap_handler 完成 Trap 分发及处理。trap_handler返回后，使用`__restore`从保存在内核栈上的 Trap 上下文恢复寄存器。最后sret 回到应用程序
		- context.rs:实现Trapcontext
		- Trap.s:`__alltraps`和`__restore`
		- mod.s:trap_init:初始化stvec(trap入口地址)；trap_handler:根据scause执行中断，出错-跑下一个app或者执行app的系统调用
		- `__alltraps`：交换sp和sscratch，换后sp-kernel；ssc-user。按照trapcontext的布局保存寄存器至内核栈，其中x2存ssc（用户态sp），sp交给a0，即traphandler的参数
		- `__restore`：恢复寄存器，交换sp和ssc，sret
	- syscall/:mod实现syscall函数，处理调用；process实现sys_exit,打印信息后调用run_next_app;fs实现sys_write
	- lang_items.rs:实现panic||console.rs:实现putchar||sbi.rs:实现sbi调用
	- main.rs:启动器，trap_init;batch_init.最后调用==run_next_app==
	- linker-qemu.ld:内核内存布局，从0x80200开始
	- build.rs:生成link_app.s,负责将应用作为数据段链接到内核，user/bin有几个app就连接几个
- user/src/用户相关code
	- bin/:batch系统要跑的程序
	- lib.rs:用户程序的依赖库
		- 用户程序入口点设置为`__start`，即0x80400，start相当于用户的entry.asm
		- `__start`:clear-bss，调用main（指向某个用户程序），执行exit(sys_call)-内核执行==run_next_app==
		- 提供系统调用的用户接口:write(sys_write),exit(sys_exit)
	- syscall.rs:实现系统调用，将ecall包装在syscall函数中，由sys_write,sys_exit调用
	- linker.ld:用户程序内存布局，从0x80400开始


# 问答题
## 1. 
 bad-register,错误是Illegal instruction，非法指令。试图在用户态获取==sstatus==的状态
 bad-instruction,错误是Illegal instruction。在用户态使用跨特权级的指令==sret==
 bad-address，错误是Segmentation fault。这里没加合法检查，试图写入0x0，报段错误
 ## 2.
 1. a0代表内核栈栈顶；两种方法分别是由中断返回调用，正常结束中断，或者是在运行新app时用于返回用户态
 2. 特殊处理三个CSR；sepc表示程序入口；sscratch存的是用户栈指针；sstatus包含当前运行状态等信息
 3. x2是栈寄存器，稍后保存；x4为tp一般应用程序不会需要这个东西
 4. sp->user_stack;sscratch->kernel_stack
 5. sret,从S模式返回U模式
 6. sp->kernel_stack;sscratch->user_stack
 7. ecall