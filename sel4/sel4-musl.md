# libsel4muslsys

- 提供musl->sel4的接口
- vsyscall:在musllib中的syscall调用跳转至此





# NOTES

(gdb) p /x morecore_base 0x423000
(gdb) p /x morecore_top 0x523000

`CONFIG_LIB_SEL4_MUSLC_SYS_MORECORE_BYTES`决定是否invoke sel4内核

若设置则为static分配，不invoke,在sysmorecore中直接声明一个区域

```c
char __attribute__((aligned(PAGE_SIZE_4K))) morecore_area[CONFIG_LIB_SEL4_MUSLC_SYS_MORECORE_BYTES];

```

`CONFIG_LIB_SEL4_MUSLC_SYS_MORECORE_BYTES`由gen_config指定，在elf文件中位于

> 0000000000423000 g   O .bss  0000000000100000 morecore_area

若没设置则需要在用户态app中绑定vspace

# 一次malloc(动态)触发的系统调用

第二次进入sys_brk_dynamic时，`newbrk=0x10001000;muslc_brk_reservation_start=0x10000000`