# code-tree
- os/src/task,实现switch，由trap_handler调用
	- mod.rs 总管任务切换
		- `TaskManager` 总结构体，内含`TaskManagerInner`被包在UPSafeCell中。`TaskManagerInner`中有当前运行task_id和==task_list==。`TASK_MANAGER`为结构体的实例化，使用lazy_static。
		- 
	- switch.S 当前task的`TaskContext`保存到==current_task_cx_ptr==，取出放在==next_task_cx_ptr==的`TaskContext`并赋值给寄存器们.switch.rs将switch嵌入为可调用函数
	- task 实现`TaskControlBlock`(struct)->==TaskStatus==,==TaskConetxt==，任务状态以及context;`TaskStatus`(enum),任务状态
	- context.rs 实现`TaskContext`，存有ra，sp，s0-s11。
		- zero_init:结构体初始化为全0
		- goto_restore：
- timer.rs:系统调用获取时间
- loader.rs:承接ch2batch的部分工作，从link_app.S中找到各个app并把app们加载到内存的指定位置
- user/build.py:用linker.ld构建每个app的内存布局，放在不同起始地址的内存上
	- `os.system('cargo build --bin %s --release' % app)`构建好后用这一条指令build
switch.s->restore.s->用户态start.s
