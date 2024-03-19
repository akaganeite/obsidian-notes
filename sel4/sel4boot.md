# 系统启动流程

## RISCV common_riscv.lds







## head.S

`_start`:内核入口函数->`_start64`->`_entry_64`->boot_sys->c_boot_sys

`_start`->`common_init`:fsgs段寄存器、分页、64位、GDT、SYSCALL等初始化

Boot_sys->try_boot_sys->try_boot_sys_node->init_sys_state

## boot_sys.c

```c
BOOT_CODE VISIBLE void boot_sys(
    unsigned long multiboot_magic,
    void *mbi)
{
    bool_t result = false;
    /*Z 内存图、initrd模块地址、图形显示配置、ACPI表 */
    if (multiboot_magic == MULTIBOOT_MAGIC) {           /*Z 是由兼容Multiboot规范的引导程序加载的 */
        result = try_boot_sys_mbi1(mbi);//第一个函数调用
    } 
    /*Z 内核映像、CPU、SMP、IOMMU、APIC(中断控制器)、I/OAPIC、页表、定时器、FPU，可配置地启动VMX虚拟机；
    创建initrd线程、idle线程；初始化中断、系统运行状态，启动其它核；初始化并获取大内核锁； */
    if (result) {
        result = try_boot_sys();//第二个
    }

    if (!result) {
        fail("boot_sys failed for some reason :(\n");
    }

    ARCH_NODE_STATE(x86KScurInterrupt) = int_invalid;
    ARCH_NODE_STATE(x86KSPendingInterrupt) = int_invalid;
    schedule();//第三个
    activateThread();
}/*Z 这个函数返回后，ret指令弹出的是restore_user_context()地址，相应的会释放内核锁 */

/*------------------------------------------------------------------------------------------------------------*/
static BOOT_CODE bool_t try_boot_sys(void)
{
    /*Z 建立内核页表，映射物理地址、APIC、IOAPIC、IOMMU，清空TLB、paging缓存、内核滑动窗口等；
    初始化cpu关键寄存器、数据结构，设置APIC、定时器、FPU，可配置地启动VMX虚拟机；
    创建initrd线程，包括地址空间、IPC buffer、bootinfo、TCB、CNode、页表、有关能力等；
    创建idle线程；初始化中断、IOMMU、系统运行状态，记录空闲内存等 */
    printf("Starting node #0 with APIC ID %lu\n", boot_state.cpus[0]);
    if (!try_boot_sys_node(boot_state.cpus[0])) {
        return false;
    }  
  	... ...
  	printf("Booting all finished, dropped to user space\n");
    //结束boot，转到用户态运行
}

static BOOT_CODE bool_t try_boot_sys_node(cpu_id_t cpu_id)
{
  /*Z 建立静态内核页表，映射所有可能的物理地址(实际不一定都存在)，映射APIC、IOAPIC、IOMMU设备内存，
	清空TLB、paging缓存 */
  map_kernel_window();
  /*Z 将一级页表地址写入CR3，从而开始一个虚拟地址空间 */
  setCurrentVSpaceRoot(kpptr_to_paddr(X86_KERNEL_VSPACE_ROOT), 0);
  /*Z 初始化cpu关键寄存器、数据结构，设置APIC、定时器、FPU，可配置地启动VMX虚拟机 */
  init_cpu(config_set(CONFIG_IRQ_IOAPIC) ? 1 : 0));
  /*Z 创建initrd线程，包括地址空间、IPC buffer、bootinfo、TCB、CNode、页表、有关能力等；
	创建idle线程；初始化中断、IOMMU、系统运行状态，记录空闲内存等 */
  init_sys_state(
    cpu_id,
    &boot_state.mem_p_regs,
    boot_state.ui_info,
    boot_mem_reuse_p_reg,
    /* parameters below not modeled in abstract specification */
    boot_state.num_drhu,
    boot_state.drhu_list,
    &boot_state.rmrr_list,
    &boot_state.acpi_rsdp,
    &boot_state.vbe_info,
    &boot_state.mb_mmap_info,
    &boot_state.fb_info
    )
}
```

>  内核虚拟地址到物理地址的转换--减去KERNEL_BASE_OFFSET。kernel image在虚拟和物理内存上都是连续的。
>
>  第一个thread，initrd在内存中的位置紧挨着kernel_elf
>
>  kernel_elf:start=0x100000 end=0xa13000 
>
>  initrd:ui_info.p_reg.start:0xa13000;ui_info.p_reg.end:0xb3a000,image:images/capabilities-image-x86_64-pc99

`image加载过程：`

1. Detected 1 boot module(s):

   module #0: start=0xa14000 end=0xa65bb8 size=0x51bb8 name='images/capabilities-image-x86_64-pc99'

2. `load_boot_module`:把module_start:0xa14000，加载到load_paddr:0xa66000;

3. 把image挪一下`Moving loaded userland images to final location: from=0xa66000 to=0xa13000 size=0x127000`

4. image对应的虚拟地址：`ui_v_reg.start:0x400000;ui_v_reg.end:0x527000`

5. initrd在`0x527000`后添加一个IPC_buffer和bootinfo页以及额外信息

6. 最终initrd在线性地址中占的空间为`0x400000-0x52a000`

## `initrd的vspace`(以capabilities为例)

```c
//boot.c::init_sys_state->

/*Z 设定initrd线程的区域 */
/* The region of the initial thread is the user image + ipcbuf and boot info */
it_v_reg.start = ui_v_reg.start;//0x400000
it_v_reg.end = ROUND_UP(extra_bi_frame_vptr + BIT(extra_bi_size_bits), PAGE_BITS);//0x52a000

/*Z 创建完整的页表及其访问能力 */
/* Construct an initial address space with enough virtual addresses
 * to cover the user image + ipc buffer and bootinfo frames */
it_vspace_cap = create_it_address_space(root_cnode_cap, it_v_reg);


/*Z 为initrd映像创建页表项和访问能力 */
/* create all userland image frames */
create_frames_ret =create_frames_of_region(
        root_cnode_cap,     /*Z CNode */
        it_vspace_cap,      /*Z VSpace */
        ui_reg,             /*Z initrd映像线性区域 */
        true,
        ui_info.pv_offset);

/*Z 为initrd创建ASID(PCID) pool能力，并关联其VSpace */
it_ap_cap = create_it_asid_pool(root_cnode_cap);
write_it_asid_pool(it_ap_cap, it_vspace_cap);


/*Z 创建idle线程 */
/* create the idle thread */
if (!create_idle_thread()) {
    return false;
}

/*Z 填充TCB CNode和tcb_t有关数据结构，创建initrd线程 */
tcb_t *initial = create_initial_thread(
  					 root_cnode_cap,  /*Z CNode */
             it_vspace_cap,   /*Z VSpace */
             ui_info.v_entry, /*Z ELF入口点虚拟地址 */
             bi_frame_vptr,   /*Z bootinfo虚拟地址 */
             ipcbuf_vptr,     /*Z IPC buffer虚拟地址 */
  
init_core_state(initial);/*Z 初始化系统运行状态，设置调度器预选对象 */
  
/*Z 设备RAM、ROM，空闲内存和内核启动内存，作为untyped，记录在ndks_boot中，并在根CNode中创建相应的能力 */
if (!create_untypeds(root_cnode_cap, boot_mem_reuse_reg)) {
    return false;
}
```

```c
//vspace.c::create_it_address_space->
/*全局一级页表拷贝到rootserver.vaspace*/
copyGlobalMappings(PML4_PTR(rootserver.vspace));

/*创建vspace的cap*/
vspace_cap = cap_pml4_cap_new(IT_ASID,/* capPML4MappedASID */
   														rootserver.vspace,/* capPML4BasePtr   *///一级页表基地址
   														1/* capPML4IsMapped   */);

/*Z 赋予rootserver的VSapce(一级页表)访问能力 */
write_slot(SLOT_PTR(pptr_of_cap(root_cnode_cap), seL4_CapInitThreadVSpace), vspace_cap);
/*用循环赋予2，3，4级页表访问能力*/

```

在`copyGlobalMappings`之前，kernel页表状态如下图所示

![mapping](../assets\mapping.png)

level2paging->16

level3paging->17

level4paging->18

extra bootinfo->19

userland_image_frames->20-314

![image-20240229232436276](../assets\image-20240229232436276.png)