

# SEL4 kernel code notes

- [ ] 中断源码
- [ ] ntfn源码
- [ ] 两篇基于faas论文的概念
- [ ] sel4+faas论文的实现结构

[TOC]



# CAPABILITIES

对用户来说，capabilities对内核对象直接在物理内存层面进行管理。

![image-20231215155250577](./assets/image-20231215155250577.png)

一个cnode包含了一个胖指针数组，每个胖指针都指向系统中的一个内核对象。

capabilities本质上是一个access token，如下图所示

![image-20231215161707913](./assets/image-20231215161707913.png)

## 解构CNODE

> A CNode is a table of slots, each of which may contain a capability. This may include capabilities to further CNodes, forming a directed graph.
>

其中，一个slot也被称为一个CTE,包含cap_t和mdb_node_t,cap_t的类型有多种，下面以cnode_cap为例解构了cap_t的一种组成方式。当一个cte中存的cap类型为cnode时，这个cnode又指向一个cte数组，可以继续寻址；存的是其他类型cap时，代表这是某一thread拥有的某个kernel_object的cap。



```c
cte:
struct cte {
    cap_t cap;              /*Z u64[2] */
    mdb_node_t cteMDBNode;  /*Z u64[2] */
};
typedef struct cte cte_t; 

cap_t:
struct cap {
    uint64_t words[2];
};
typedef struct cap cap_t;

mdb_node_t:
struct mdb_node {
    uint64_t words[2];
};
typedef struct mdb_node mdb_node_t;

structure：
block mdb_node {
    padding 16
    field_high mdbNext 46
    field mdbRevocable 1
    field mdbFirstBadged 1
    field mdbPrev 64
}
```

一个cte图示，以untyped_cap为例

![image-20231211145617913](./assets\image-20231211145617913.png)





以cnode_cap为例展示其结构，其余结构(其他内核对象的cap)均类似

cnode_cap结构：`在代码实现中word[0]和word[1]是倒置的`

![image-20231211142331643](./assets\image-20231211142331643.png)

```
block cnode_cap(capCNodeRadix, capCNodeGuardSize, capCNodeGuard,
                capCNodePtr, capType) {
    field capCNodeGuard 64 具体的guard数值；
    field capType 5 
    field capCNodeGuardSize 6 guard占的位数；
    field capCNodeRadix 6 CNode的最大slot个数；
    field_high capCNodePtr 47 指向CNode的指针；
}
```

一个cnode_cap可以理解为封装了指向cte数组指针(capCNodePtr)的数据结构，也即fat pointer，提供一些管理信息。

## CNODE创建

> CNode must be created by calling seL4_Untyped_Retype() 
>
> The caller must therefore have a capability to enough untyped memory as well as enough free capability slots available in existing CNodes for the seL4_Untyped_Retype() invocation to succeed

关于untyped_retype的讲解见后文。

### 第一个CNODE

在init_thread被创建的时候，其tcb_t内的cnode已经被初始化，部分初始化的cslot如下所示

![image-20231212142421496](./assets\image-20231212142421496.png)

**最开始Thread中CSPACE的布局，蓝色线是tutorial中第一个cnodecopy的操作**，包括cnode复制和mdb_node的赋值

![image-20231212225704002](./assets\image-20231212225704002.png)

> 对于一个thread来说，在初始化时tcb_t内自带一个cslot数组，部分slot被初始化为不同的cap。在第一个cslot中存放的是`tcbCTable`(ctable)，代表这个thread的Cspace root，是一个cnode_cap。这个cap中的cnode指向的cslot数组才是上述图中的initial thread's CNode content.

## CNODE相关method

- **seL4_CNode_Mint()**:从现有cap创建一个新的cap
- **seL4_CNode_Copy()** :
- **seL4_CNode_Mutate()** 
- **seL4_CNode_Rotate()**
- **seL4_CNode_Delete()** 
- **seL4_CNode_Revoke()** 
- **seL4_CNode_SaveCaller()**
- **seL4_CNode_CancelBadgedSends()** 

### seL4_CNode_Mint调用流程

**用户：**

`seL4_CNode_Mint`(seL4_CNode _service, seL4_Word dest_index, seL4_Uint8 dest_depth, seL4_CNode src_root, seL4_Word src_index, seL4_Uint8 src_depth, seL4_CapRights_t rights, seL4_Word badge)

- `_service`,`dest_index`,`dest_depth`:destination cspace root cnode，index,depth
- src_root,index,depth: source cspace root cnode，index,depth
- seL4_CapRights_t `rights`:新cnode的权利
- seL4_Word `badge`:给endpoint用的badge

调用实例：

```c
seL4_CNode_Copy(seL4_CapInitThreadCNode, first_free_slot, seL4_WordBits,
                seL4_CapInitThreadCNode, seL4_CapInitThreadTCB, seL4_WordBits,
                seL4_AllRights,0);
```

- depth统一为64
- source和dest的都为本thread的root_cnode_cap`seL4_CapInitThreadCNode=2`
- `seL4_CapInitThreadTCB`为sourceslot，是tcb_cap
- `first_free_slot`是第一个空槽，是dest

------

`seL4_CallWithMRs`(_service, tag,&mr0, &mr1, &mr2, &mr3);

- tag是`seL4_MessageInfo_new`(CNodeMint, 0, 1, 6);用于存放要唤醒的函数类型(label)以及消息寄存器个数等
- mr0：dest_index,mr1:dest_depth
- mr2:src_index,mr3:src_depth
- mr4:rights,mr5:badge

------

`x64_sys_send_recv`(seL4_SysCall, dest, &dest, msgInfo.words[0], &info.words[0], &msg0, &msg1, &msg2, &msg3, 0);

转到内核的系统调用处理函数

------

**内核：**

SLOWPATH:

```c
void NORETURN slowpath(syscall_t syscall)//省略了一些判断
{
    ksKernelEntry.is_fastpath = 0;
    handleSyscall(syscall);//syscall_handler
    restore_user_context();//释放锁并恢复用户上下文：或处理等待的中断、或恢复虚拟机、或系统调用和中断返回
    UNREACHABLE();
}
```

------

根据syscall类型选择不同的处理路径，这里是`SysCall`，调用handleinvocation，再调用decodeinvocation

```c
exception_t handleSyscall(syscall_t syscall){
    case SysCall:
    ret = handleInvocation(true, true, true, false, getRegister(NODE_STATE(ksCurThread), capRegister));
    //省略出错处理
    break;
}
/*Z 处理主动类系统调用
Send:   false, true
NBSend: false, false
Call:   true,  true */
static exception_t handleInvocation(bool_t isCall, bool_t isBlocking){//true,true
	thread = NODE_STATE(ksCurThread);//拿到当前cpu的信息？我感觉是啥也没改，原路返回，thread=ksCurThread
    //get register是拿到在x86指令集下syscall这个指令传递的一些参数，不是mr们，这里拿到第三个reg的内容，就是info.word[0]
    info = messageInfoFromWord(getRegister(thread, msgInfoRegister));
    cptr_t cptr = getRegister(thread, capRegister);/*Z 使用的CSlot句柄,拿到dest_capptr(2) */
    /*Z 查找CSlot句柄指示的能力和CSlot */
    /* faulting section */
    lu_ret = lookupCapAndSlot(thread, cptr);
    /*Z 查找当前线程的IPC buffer */
    buffer = lookupIPCBuffer(false, thread);
    /*Z 在当前线程CSpace中按机器字深度查找额外能力CSlot指针，赋予全局变量 */
    status = lookupExtraCaps(thread, buffer, info);
    /* Syscall error/Preemptible section */
    length = seL4_MessageInfo_get_length(info);
    if (unlikely(length > n_msgRegisters && !buffer)) {
        length = n_msgRegisters;
    }
    status = decodeInvocation(seL4_MessageInfo_get_label(info), length,
                              cptr, lu_ret.slot, lu_ret.cap,
                              current_extra_caps, isBlocking, isCall,
                              buffer);
}
```

`lookupCapAndSlot`使用tcb和一个cnode_cap(实际是在tcb的cslot中的偏移，本例中为2)进行寻址，调用路径为

> lookupCapAndSlot(tcb_t *thread, cptr_t cPtr)`->`lookupSlot(thread, cPtr);`->`resolveAddressBits(threadRoot, capptr, wordBits);

这一段路径是内核代码对某一个tcb的cnode寻址的路径

## CNODE寻址(lookupcapandslot)

`resolveAddressBits` in `cspace.c`

```C
resolveAddressBits_ret_t resolveAddressBits(cap_t nodeCap, cptr_t capptr, word_t n_bits){
    while (1) {
        //radixbits是这个cnode对应的cslot的索引位位数
        offset = (capptr >> (n_bits - levelBits)) & MASK(radixBits);/*Z 获取capptr中的索引值 */
        //cap_cnode_cap_get_capCNodePtr找到capNodeptr并返回，也即找到所指向的数组首地址
        slot = CTE_PTR(cap_cnode_cap_get_capCNodePtr(nodeCap)) + offset;/*Z 获取该索引值对应的CSlot */
        if (likely(n_bits <= levelBits)) {/*Z 解析位恰好用完(小于的情况上面已排除)，结束 */
            ret.status = EXCEPTION_NONE;
            ret.slot = slot;
            ret.bitsRemaining = 0;
            return ret;
        }
        /*Z 递进至下一级 */
        n_bits -= levelBits;
        nodeCap = slot->cap;
        /*Z 下一级的能力不是CNode能力，则此“下一级”是叶子CSlot，结束 */
        if (unlikely(cap_get_capType(nodeCap) != cap_cnode_cap)) {
            ret.status = EXCEPTION_NONE;
            ret.slot = slot;
            ret.bitsRemaining = n_bits;
            return ret;
        }
    }
}
```

根据传入的nodecap，index(capptr)和depth(n_bits)寻找对应的cnode。其过程就相当于指针数组寻址，根据一个给定的偏移和搜索深度进行寻址，可能有多层数组需要寻找。

------

`decodeInvocation`,根据传入的cap来决定执行什么操作，这里是`cap_cnode_cap`

```
exception_t decodeInvocation(word_t invLabel, word_t length,/*Z 系统调用传入的当前线程参数： 消息标签(错误类型)、长度 */
                             cptr_t capIndex, cte_t *slot, cap_t cap,/*Z CSlot句柄、CSlot(包含cnode和mdbnode)、能力(槽中包含的那个cnode) */
                             extra_caps_t excaps, bool_t block, bool_t call,/*Z 额外能力、是否阻塞、是否Call调用 */
                             word_t *buffer)/*Z IPC buffer */
{
 	switch (cap_get_capType(cap)) {
 	...
    case cap_cnode_cap:
    	return decodeCNodeInvocation(invLabel, length, cap, excaps, buffer);
    ...
    	}
}

```



------

```c
/*Z 引用cap_cnode_cap能力的系统调用 */
exception_t decodeCNodeInvocation(word_t invLabel, word_t length, cap_t cap,/*Z 消息标签、长度、能力 */
                                  extra_caps_t excaps, word_t *buffer)/*Z 额外能力、IPC buffer */
{
	index = getSyscallArg(0, buffer);/*Z消息传参(公共部分)：0-要操作的目标能力句柄 */
    w_bits = getSyscallArg(1, buffer);/*Z 1-句柄深度。不能为0 */
    /*Z 查找要操作能力句柄指代的能力 */
    lu_ret = lookupTargetSlot(cap, index, w_bits);//dest
    destSlot = lu_ret.slot;
    srcIndex = getSyscallArg(2, buffer);/*Z 消息传参(个例部分)：2-源能力句柄 */
    srcDepth = getSyscallArg(3, buffer);/*Z 3-句柄深度 */
	srcRoot = excaps.excaprefs[0]->cap;/*Z extraCaps0-源CNode */
	lu_ret = lookupSourceSlot(srcRoot, srcIndex, srcDepth);
    srcSlot = lu_ret.slot;
	//然后根据invlabel选择要做的操作
	switch (invLabel) {
	 case CNodeMint:/*Z子功能：制作(拷贝并修改)能力 */
            cap_rights = rightsFromWord(getSyscallArg(4, buffer));/*Z消息传参：4-新能力权限 */
            capData = getSyscallArg(5, buffer);/*Z 5-新能力的可更新参数 */
            srcCap = maskCapRights(cap_rights, srcSlot->cap);/*Z 根据新权限设置能力最后的权限，只减不增 */
            /*Z 返回拷贝(导出)的能力。基本是原能力，要作一些排错、复位等处理 */
            dc_ret = deriveCap(srcSlot,updateCapData(false, capData, srcCap));
            newCap = dc_ret.cap;
            isMove = false;
            break;
	}
	setThreadState(NODE_STATE(ksCurThread), ThreadState_Restart);
	//ismove=false
	if (isMove) {/*Z 移动CSlot(目标使用新能力) */
        return invokeCNodeMove(newCap, srcSlot, destSlot);
    } else {/*Z 源、目的CSlot建立关联，并将新能力拷贝到目的CSlot。目的CSlot必须为空能力 */
        return invokeCNodeInsert(newCap, srcSlot, destSlot);
    }
}
```

> __`lookupTargetSlot(cap, index, w_bits);`和`lookupSourceSlot(srcRoot, srcIndex, srcDepth);`传入的第一个参数均是cap_t类型的，是实打实的cap，而不是需要寻址的东西。这第一个参数就是root，后两个参数代表在root内寻址，本例中，sourceroot(srcRoot)和destroot(cap)均是tcb cspace中的第三项seL4_CapInitThreadCNode,`指向tcb_cspace首地址`。sourceindex是seL4_CapInitThreadTCB，destindex(1)是first_free_slot(409).上述两个函数就是找到src和destindex索引到的cslot并返回__

------

invokecnodeinsert->cteinsert

```c
/*Z 源、目的CSlot建立关联，并将新能力拷贝到目的CSlot。目的CSlot必须为空能力 */
exception_t invokeCNodeInsert(cap_t cap, cte_t *srcSlot, cte_t *destSlot)
{
    cteInsert(cap, srcSlot, destSlot);

    return EXCEPTION_NONE;
}

void cteInsert(cap_t newCap, cte_t *srcSlot, cte_t *destSlot)
{
    mdb_node_t srcMDB, newMDB;
    cap_t srcCap;
    bool_t newCapIsRevocable;

    srcMDB = srcSlot->cteMDBNode;
    srcCap = srcSlot->cap;
    /*Z 新能力相对于源能力是否可撤销 */
    newCapIsRevocable = isCapRevocable(newCap, srcCap);//不可撤销，false
    /*Z 建立CSlot间关联 */
    newMDB = mdb_node_set_mdbPrev(srcMDB, CTE_REF(srcSlot));//直接从srcmdb修改，prev指向source_cnode的mdb
    newMDB = mdb_node_set_mdbRevocable(newMDB, newCapIsRevocable);//不可撤销，设置1bit
    newMDB = mdb_node_set_mdbFirstBadged(newMDB, newCapIsRevocable);//1bit，false
    /* Prevent parent untyped cap from being used again if creating a child
     * untyped from it. */
    setUntypedCapAsFull(srcCap, newCap, srcSlot);

    destSlot->cap = newCap;//赋值cap
    destSlot->cteMDBNode = newMDB;//赋值node
    mdb_node_ptr_set_mdbNext(&srcSlot->cteMDBNode, CTE_REF(destSlot));//source_mdb.next->dest_slot
    if (mdb_node_get_mdbNext(newMDB)) {/*Z 如果源CSlot有下一个关联 */
        mdb_node_ptr_set_mdbPrev(/*Z 则下一个关联的前向指向目的CSlot，即目的CSlot插入关联链条 */
            &CTE_PTR(mdb_node_get_mdbNext(newMDB))->cteMDBNode,
            CTE_REF(destSlot));
    }
}
```

完成后的src_mdb,dest_mdb以及mdb格式:

```c
mdb_node_t格式：同一对象（资源）的能力通过此域链接起来，但反之不然
        u64[0]     63                                          0
                             关联的上一个CSlot(cte_t*类型)
        u64[1]     63       47             2        1           0
                   padding   关联的下一个CSlot     是否可撤销  是否首个标记的
srcmdb.word_0:0x0000000000000000,srcmdb.word_1:0x0000ff801fc03323
newmdb.word_0:0xffffff801fc00020,newmdb.word_1:0x0000000000000000
```

------

# UNTYPED

## untyped cap 布局

![image-20231215110613207](./assets\image-20231215110613207.png)

## untyped retype

这个函数在内核通过正常系统调用途径，`handlesyscall->decodeinvocation`,而后调用`untyped.c`中的`decodeUntypedInvocation`和`invokeUntyped_Retype`

```c
/*Z 引用cap_untyped_cap能力的系统调用：分配内存创建seL4内核对象和能力，并与untyped CSlot建立关联 */
exception_t decodeUntypedInvocation(word_t invLabel, word_t length, cte_t *slot,/*Z 消息标签、长度、引用的CSlot */
                                    cap_t cap, extra_caps_t excaps,/*Z 引用的能力、extraCaps */
                                    bool_t call, word_t *buffer)/*Z 是否Call调用、IPC buffer */
{
    //第一部分 接收参数 inde=0，depth=0，offset是目标cap的起始slot(必须为空)，window是数量
    newType     = getSyscallArg(0, buffer);/*Z消息传参：0-新分配对象的类型 */
    userObjSize = getSyscallArg(1, buffer);//Z 1-要求的对象大小(位数。对CNode是索引位，对untyped、SC是内存大小，其它忽略) 
    nodeIndex   = getSyscallArg(2, buffer);//2-存放访问能力的目标CNode句柄。实际调用中右对齐拼接：保护位1 索引位1 保护位2 索引位2；这些位加起来就是深度
    nodeDepth   = getSyscallArg(3, buffer);/*Z 3-深度。为0时extraCaps0即为目标CNode，nodeIndex参数忽略 */
    nodeOffset  = getSyscallArg(4, buffer);/*Z 4-存放访问能力的CSlot开始索引 */
    nodeWindow  = getSyscallArg(5, buffer);/*Z 5-存放访问能力的连续CSlot数量(新对象数量) */
    
    //第二部分 找到目标的slot数组的cnode_cap(initcnode)
    if (nodeDepth == 0) {
         nodeCap = excaps.excaprefs[0]->cap;//initthreadcnodecap aka. root_thread
    }
    
    //第三部分 根据前面的cnode_cap找到要被retype的序列，存入slot_range_t slots，
    /* Ensure that the destination slots are all empty. */
    slots.cnode = CTE_PTR(cap_cnode_cap_get_capCNodePtr(nodeCap));
    slots.offset = nodeOffset;
    slots.length = nodeWindow;
    
    
    //第四部分
    /*Z 获取未分配内存起始空闲块(16字节大小)索引 ，存入freeRef*/
    /*Z seL4没有将撤销的资源交回untyped的机制，因此这里是检查可以回收资源的时机，仅当无任何子对象存在时（所有分配出去的均已
    全部删除），重新开始分配。好处是简单有效，坏处是利用率 */
    status = ensureNoChildren(slot);
    if (status != EXCEPTION_NONE) {
        freeIndex = cap_untyped_cap_get_capFreeIndex(cap);
        reset = false;
    } else {/*Z untyped CSlot无子CSlot，说明未分配过 */
        freeIndex = 0;
        reset = true;/*Z reset用于指示清零内存，重置untyped CSlot中的空闲索引域 */
    }   /*Z 首个空闲块地址 */
    freeRef = GET_FREE_REF(cap_untyped_cap_get_capPtr(cap), freeIndex);
  	alignedFreeRef = alignUp(freeRef, objectSize);//对齐
    return invokeUntyped_Retype(slot, reset,/*Z 引用的CSlot、是否尚未分配过 */
           (void *)alignedFreeRef, newType, userObjSize,/*Z 对齐后的首个空闲块地址、对象类型、要求的大小 */
           slots, deviceMemory);/*Z CSlot存放范围、是否设备内存 */
}

/*------------------------------------------------------------------------------------------------------------*/
exception_t invokeUntyped_Retype(cte_t *srcSlot,/*Z 引用的CSlot */
                                 bool_t reset, void *retypeBase,/*Z 是否尚未分配过、对齐后的首个空闲块地址 */
                                 object_t newType, word_t userSize,/*Z 对象类型、要求的大小 */
                                 slot_range_t destSlots, bool_t deviceMemory)/*Z CSlot存放范围、是否设备内存 */
{
    word_t freeRef;
    word_t totalObjectSize;
    //cap_untyped_cap_get_capFreeIndex拿到cap中存的内存地址，
    void *regionBase = WORD_PTR(cap_untyped_cap_get_capPtr(srcSlot->cap));/*Z untyped内存首地址(线性地址)*/
    exception_t status;
	/*Z untyped内存首个空闲内存块地址 */  
    //cap_untyped_cap_get_capFreeIndex拿到cap中存的首个空闲块地址
    freeRef = GET_FREE_REF(regionBase, cap_untyped_cap_get_capFreeIndex(srcSlot->cap));
    
    /*Z 更新untyped能力中的首个空闲块索引 */
    totalObjectSize = destSlots.length << getObjectSize(newType, userSize);
    freeRef = (word_t)retypeBase + totalObjectSize;//原来的起始地址＋新分配后的内存大小=现在的空闲索引
    srcSlot->cap = cap_untyped_cap_set_capFreeIndex(srcSlot->cap,
                                                    GET_FREE_INDEX(regionBase, freeRef));
    /*Z 创建seL4内核对象和能力，并与untyped CSlot建立关联 */
    createNewObjects(newType, srcSlot, destSlots, retypeBase, userSize,
                     deviceMemory);

    return EXCEPTION_NONE;
}

```

------

```c
/*Z 创建seL4内核对象和能力，并与untyped CSlot建立关联 */
void createNewObjects(object_t t, cte_t *parent, slot_range_t slots,/*Z 对象类型、引用的untyped CSlot、新CSlot存放范围 */
                      void *regionBase, word_t userSize, bool_t deviceMemory)/*Z 对齐后的首个空闲块地址、要求的对象大小、是否设备内存 */
{
    word_t objectSize;
    void *nextFreeArea;
    word_t i;
    word_t totalObjectSize UNUSED;

    /* ghost check that we're visiting less bytes than the max object size */
    objectSize = getObjectSize(t, userSize);
    totalObjectSize = slots.length << objectSize;
    /** GHOSTUPD: "(gs_get_assn cap_get_capSizeBits_'proc \<acute>ghost'state = 0
        \<or> \<acute>totalObjectSize <= gs_get_assn cap_get_capSizeBits_'proc \<acute>ghost'state, id)" */

    /* Create the objects. */
    nextFreeArea = regionBase;
    for (i = 0; i < slots.length; i++) {
        /* Create the object. *//*Z 用指定的空闲内存创建指定的seL4内核对象，返回新对象的操控能力 */
        /** AUXUPD: "(True, typ_region_bytes (ptr_val \<acute> nextFreeArea + ((\<acute> i) << unat (\<acute> objectSize))) (unat (\<acute> objectSize)))" */
        cap_t cap = createObject(t, (void *)((word_t)nextFreeArea + (i << objectSize)), userSize, deviceMemory);
        /*Z 赋值能力并插入到父untyped CSlot关联链的头，置首个、可撤销标记 */
        /* Insert the cap into the user's cspace. */
        insertNewCap(parent, &slots.cnode[slots.offset + i], cap);

        /* Move along to the next region of memory. been merged into a formula of i */
    }
}

//根据不同的能力类型创建能力，如果能力需要引用额外内存，则把regionbase转为对应类型的指针传给cap，下次调用cap进行操作时可以找到指针
cap_t createObject(object_t t, void *regionBase, word_t userSize, bool_t deviceMemory)
{                   /*Z 对象类型、空闲块地址、要求的对象大小、是否设备内存 */
    /* Handle architecture-specific objects. */
    if (t >= (object_t) seL4_NonArchObjectTypeCount) {
        return Arch_createObject(t, regionBase, userSize, deviceMemory);
    
    /* Create objects. */
    switch ((api_object_t)t) {
    case seL4_TCBObject: {/*Z 初始化部分TCB寄存器、时间片、调度域、名字、亲和cpu数据，返回线程能力 */
        tcb_t *tcb;
        /*Z 初始化上下文中cpu、FPU、DEBUG寄存器值 */
        /*此处省略*/ 
        tcbDebugAppend(tcb);/*Z 将线程插入到亲和cpu的DEBUG线程双向链表头 */
        return cap_thread_cap_new(TCB_REF(tcb));
    }

    case seL4_EndpointObject:/*Z 返回无标记、有全权的端点能力 */
        return cap_endpoint_cap_new(0, true, true, true, true,EP_REF(regionBase));

    case seL4_NotificationObject:/*Z 返回无标记、有全权的通知能力 */
        return cap_notification_cap_new(0, true, true,NTFN_REF(regionBase));
    case seL4_CapTableObject:/*Z 返回指定索引位数、无保护的CNode能力 */
        return cap_cnode_cap_new(userSize, 0, 0, CTE_REF(regionBase));

    case seL4_UntypedObject:/*Z 返回指定内存大小、全空闲的untyped能力 */
        return cap_untyped_cap_new(0, !!deviceMemory, userSize, WORD_REF(regionBase));

    default:
        fail("Invalid object type");
    }
}
```

------

insertnewcap：

```c
/*Z slot赋值能力并插入到父CSlot关联链的头，置首个、可撤销标记 */
void insertNewCap(cte_t *parent, cte_t *slot, cap_t cap)
{
    cte_t *next;

    next = CTE_PTR(mdb_node_get_mdbNext(parent->cteMDBNode));
    slot->cap = cap;            /*Z 这意味着一条CSlot关联链中可有多个节点有首标记 */
    slot->cteMDBNode = mdb_node_new(CTE_REF(next), true, true, CTE_REF(parent));//新cnode的mdbnext是next，prev是parent
    if (next) {
        mdb_node_ptr_set_mdbPrev(&next->cteMDBNode, CTE_REF(slot));
    }
    mdb_node_ptr_set_mdbNext(&parent->cteMDBNode, CTE_REF(slot));
}

/*Z 源、目的CSlot建立关联，并将新能力拷贝到目的CSlot。目的CSlot必须为空能力 */
void cteInsert(cap_t newCap, cte_t *srcSlot, cte_t *destSlot)
{
    srcMDB = srcSlot->cteMDBNode;
    srcCap = srcSlot->cap;
    /*Z 新能力相对于源能力是否可撤销 */
    newCapIsRevocable = isCapRevocable(newCap, srcCap);
    /*Z 建立CSlot间关联 */
    newMDB = mdb_node_set_mdbPrev(srcMDB, CTE_REF(srcSlot));
    newMDB = mdb_node_set_mdbRevocable(newMDB, newCapIsRevocable);
    newMDB = mdb_node_set_mdbFirstBadged(newMDB, newCapIsRevocable);

    destSlot->cap = newCap;
    destSlot->cteMDBNode = newMDB;
    mdb_node_ptr_set_mdbNext(&srcSlot->cteMDBNode, CTE_REF(destSlot));
    if (mdb_node_get_mdbNext(newMDB)) {/*Z 如果源CSlot有下一个关联 */
        mdb_node_ptr_set_mdbPrev(/*Z 则下一个关联的前向指向目的CSlot，即目的CSlot插入关联链条 */
            &CTE_PTR(mdb_node_get_mdbNext(newMDB))->cteMDBNode,
            CTE_REF(destSlot));
    }
}
```

上面两个函数对mdb_node的操作实现了一个CDT(capability derivation tree)，CDT实现可视化：

![image-20231215210136571](./assets\image-20231215210136571.png)



untyped_retype流程图示：

![image-20231215210258358](./assets\image-20231215210258358.png)

三个棕色CNODE代表同一个进程的同一个cspace，







# 系统启动流程

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

![mapping](./assets\mapping.png)

level2paging->16

level3paging->17

level4paging->18

extra bootinfo->19

userland_image_frames->20-314

![image-20240229232436276](./assets\image-20240229232436276.png)
