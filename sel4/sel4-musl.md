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

## seL4_Untyped_Retype

_service=270, type=10, size_bits=12, root=2, node_index=0, node_depth=0, node_offset=271, num_objects=1

用刚分配的left_node retype一个pagetable，放在slot271中,`retype类型为：seL4_X86_PageTable`



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

- vka_alloc_frame_maybe_device type转换，从size_bits(12)->type(8)

- vka_alloc_object_at_maybe_dev, alloc的主函数，先初始化cptr再初始化ut
- am_vka_cspace_alloc，将data转换为allocman_t格式，下面进入allocman阶段
- _cspace_single_level_alloc，通过位图找到一个空的cslot，调用下级函数
- _cspace_single_level_make_path，赋值cspacepath_t，offset是上级函数找到的空闲slot
- vka_cspace_make_path把之前赋值的cspacepath_t再赋值一遍，不做alloc，不遍历bitmap



- sel4utils_map_page_pd，创建vka_object_t数组：一次page_mapping需要alloc五个对象，待接下来的函数调用依次初始化；
- 第一次调用map_page系统调用失败了，错误码是：`SEL4_MAPPING_LOOKUP_NO_PT`

# 重要的数据结构

## allocman_t

总的alloc,存在于vka_t中,在allocman这个库内部才被当作结构体指针，在其他结构体中以`void* data`的形式存在，看到带`allocman`的函数命名基本都是以allocman作为管理结构



## cspacepath_t

描述一个cap存放的位置

```c
typedef struct _cspacepath_t {
    seL4_CPtr   capPtr;
    seL4_Word   capDepth;
    seL4_CNode  root;
    seL4_Word   dest;
    seL4_Word   destDepth;
    seL4_Word   offset;
    seL4_Word   window;
} cspacepath_t;
```

## vka_object_t

一个wrapper，cptr指向本cap；`ut描述分配本cap的untyped_cap`,类型也是vka_object

```c
typedef struct vka_object {
    seL4_CPtr cptr;
    seL4_Word ut;
    seL4_Word type;
    seL4_Word size_bits;
} vka_object_t;
```

## vka_t

```c
//抽象接口类，涉及cspace和utspace的alloc，data就是allocman的地址
typedef struct vka {
    void *data;
    vka_cspace_alloc_fn cspace_alloc;
    vka_cspace_make_path_fn cspace_make_path;
    vka_utspace_alloc_fn utspace_alloc;
    vka_utspace_alloc_maybe_device_fn utspace_alloc_maybe_device;
    vka_utspace_alloc_at_fn utspace_alloc_at;
    vka_cspace_free_fn cspace_free;
    vka_utspace_free_fn utspace_free;
    vka_utspace_paddr_fn utspace_paddr;
} vka_t;
```



## UNTYPED_CAPS管理

utspace_split_node&&utspace_split_t,`存在于utspace_interface中的utspace中，interface又存在allocman中`

`void* utspace=&utspace_split`(in utspace_interface)

```c
typedef struct utspace_split {
    /* untypeds from the kernel window. Used for anything */
    struct utspace_split_node *heads[CONFIG_WORD_SIZE];
    /* untypeds that are unknown device regions */
    struct utspace_split_node *dev_heads[CONFIG_WORD_SIZE];
    /* untypeds that are known to be RAM from the device region */
    struct utspace_split_node *dev_mem_heads[CONFIG_WORD_SIZE];
} utspace_split_t;

/* This is an untyped manager that works by splitting each untyped in half to
 * create smaller untypeds. */
struct utspace_split_node {
    cspacepath_t ut;
    /* if this is a child node, represents our parent. Our parent must by
     * definition be considered allocated */
    struct utspace_split_node *parent;
    /* if we have a parent, then this is a pointer to our other sibling */
    struct utspace_split_node *sibling;
    /* which (if any) free list this is in */
    struct utspace_split_node **head;
    /* which free list this should go back into */
    struct utspace_split_node **origin_head;
    /* physical address of the node */
    uintptr_t paddr;
    /* if this node is not allocated then these are the next/previous pointers in the free list */
    struct utspace_split_node *next, *prev;
};
```

## CSPACE管理

### cspace_single_level_t

代码`void* cspace`标明的就是这个结构，bitmap存哪些slot空闲

```c
typedef struct cspace_single_level {
    struct cspace_single_level_config config;
    size_t *bitmap;
    size_t bitmap_length;
    size_t last_entry;
} cspace_single_level_t;
```

## VSPACE管理

`muslc_this_vspace`,全局变量，在用户app被初始化，sel4_libs可以直接调用

### vspace_t

data是`sel4utils_alloc_data_t`类型的；vspace主要变量都是函数指针

```c
/* Portable virtual memory allocation interface */
struct vspace {
    void *data;//vka_t类型
    ... ...
    void *allocated_object_cookie;
};
```

### sel4utils_alloc_data_t

`vspace_mid_level_t`，页表的数组实现，vspace的镜像

```c
typedef struct sel4utils_alloc_data {
    seL4_CPtr vspace_root;
    vka_t *vka;
    vspace_mid_level_t *top_level;
    uintptr_t next_bootstrap_vaddr;
    uintptr_t last_allocated;
    vspace_t *bootstrap;
    sel4utils_map_page_fn map_page;
    sel4utils_res_t *reservation_head;
    bool is_empty;
} sel4utils_alloc_data_t;
```



## 内存管理

内存管理使用`k_r_malloc`算法管理一个`fixed_pool(内存池)`

### mspace_interface

存在allocman中的数据结构

```c
struct mspace_interface {
    void *(*alloc)(struct allocman *alloc, void *cookie, size_t bytes, int *error);
    void (*free)(struct allocman *alloc, void *cookie, void *ptr, size_t bytes);
    struct allocman_properties properties;
    void *mspace;
};
```

### mspace

存放内存池的结构，有固定池和虚拟池

```c
typedef struct mspace_dual_pool {
    size_t fixed_pool_start;
    size_t fixed_pool_end;
    mspace_fixed_pool_t fixed_pool;
    int have_virtual_pool;
    mspace_virtual_pool_t virtual_pool;
} mspace_dual_pool_t;
```

### fixed_pool

```c
typedef struct mspace_fixed_pool {//内存池顺序分配，krmalloc进行动态管理，尽可能防止碎片
    uintptr_t pool_ptr;
    size_t remaining;
    mspace_k_r_malloc_t k_r_malloc;//krmalloc里面的cookie又指向这个fixed_pool
} mspace_fixed_pool_t;
```

### k_r_malloc

空闲链表的管理结构

```c
typedef struct mspace_k_r_malloc {
    k_r_malloc_header_t base;
    k_r_malloc_header_t *freep;
    size_t cookie;//一个指向fixed_pool的指针，指示内存池的情况，k_r_malloc只负责空闲区管理分配，fixed_pool管理真正的内存
    k_r_malloc_header_t *(*morecore)(size_t cookie, struct mspace_k_r_malloc *k_r_malloc, size_t new_units);
} mspace_k_r_malloc_t;
```

### k_r_malloc_header

空闲链表的节点

```c
typedef union k_r_malloc_header {
    struct {
        union k_r_malloc_header *ptr;//使用联合体所以每个header大小是固定的
        size_t size;
    } s;
    /* Force alignment */
    long long x;
} k_r_malloc_header_t;
```

