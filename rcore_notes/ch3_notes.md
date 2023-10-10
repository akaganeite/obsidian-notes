# code-tree
- os/src/task,实现switch，由trap_handler调用
	- mod.rs 总管任务切换
		- `TaskManager` 总结构体，内含`TaskManagerInner`被包在UPSafeCell中。`TaskManagerInner`中有当前运行task_id和==task_list==。
			- `TASK_MANAGER`为结构体的实例化，使用lazy_static。在初始化时把每个task用其起始位置和用户栈指针初始化出一个`TrapContext`,而后调用goto_restore把`TaskManager`的ra设置为restore.s,sp设置为当前内核栈指针。这样后续开始运行app时执行switch，ret后跳转到restore.s恢复最开始压栈的`TrapContext`，得到ra和sp，顺利转入用户程序入口。
		- `TaskManager`方法：
			- run_first_task：跑第一个task，用switch把task换入寄存器。
			- run_next_task:找到下一个ready的task并换入
	- switch.S 当前task的`TaskContext`保存到==current_task_cx_ptr==，取出放在==next_task_cx_ptr==的`TaskContext`并赋值给寄存器们.switch.rs将switch嵌入为可调用函数
	- task 实现`TaskControlBlock`(struct)->==TaskStatus==,==TaskConetxt==，任务状态以及context;`TaskStatus`(enum),任务状态
	- context.rs 实现`TaskContext`，存有ra，sp，s0-s11。
		- zero_init:结构体初始化为全0
		- goto_restore：__restore__ 为`TaskContext`的返回地址(ra)，sp为内核栈顶指针
- loader.rs:负责把程序放在合适的位置上


switch.s->restore.s->用户态start.s
每个app独享一个user_stack 一个kernel_stack
