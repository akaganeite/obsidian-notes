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

## seL4_Untyped_Retype

 _service=273, type=8, size_bits=12, root=2, node_index=0, node_depth=0, node_offset=272, num_objects=1

Untype_cap在第273个slot，type=8:一个x86page类型，存在第272个slot

## seL4_X86_Page_Map

_service=272, vspace=3, vaddr=0x10000000, rights={1111}b,attr=seL4_X86_Default_VMAttributes

272是一个page_cap,把它map到vspace中，rights=all_rights

## seL4_Untyped_Retype*2

 _service=275, type=0, size_bits=12, root=2, node_index=0, node_depth=0, node_offset=269, num_objects=1

type=0:Untyped_object;275是一个untyped，retype到269，仍是untyped

`这是对新alloc的两个node的right进行的系统调用`，对left的系统调用也类似，不过slot是270



第二次进入sys_brk_dynamic时，`newbrk=0x10001000;muslc_brk_reservation_start=0x10000000`

root_ut，head[27]

```c
p /x *head[27]
 = {ut = {capPtr = 0xb6, capDepth = 0x40, root = 0x2, dest = 0x0, destDepth = 0x0, offset = 0xb6,window = 0x1}
parent = 0x0, sibling = 0x0, head = 0x0, origin_head = 0x442e28,paddr = 0x10000000, next = 0x4479a0, prev = 0x0}
```

- paddr=0x10000000

root_ut->next

```c
p /x *head[27]->next
$20 = {ut = {capPtr = 0xb5, capDepth = 0x40, root = 0x2, dest = 0x0, destDepth = 0x0, offset = 0xb5,window = 0x1}
parent = 0x0, sibling = 0x0, head = 0x0, origin_head = 0x442e28, paddr = 0x8000000,next = 0x0, prev = 0x447a20
```

- paddr = 0x8000000

Head[26]

```c
(gdb) p /x *head[26]
$21 = {ut = {capPtr = 0xb7, capDepth = 0x40, root = 0x2, dest = 0x0, destDepth = 0x0, offset = 0xb7,window = 0x1}
parent = 0x0, sibling = 0x0, head = 0x0, origin_head = 0x442e20,paddr = 0x18000000, next = 0x447920, prev = 0x0}

(gdb) p /x *head[26]->next
$23 = {ut = {capPtr = 0xb4, capDepth = 0x40, root = 0x2, dest = 0x0, destDepth = 0x0, offset = 0xb4,window = 0x1}
parent = 0x0, sibling = 0x0, head = 0x0, origin_head = 0x442e20, paddr = 0x4000000,
  next = 0x0, prev = 0x447aa0}
```

