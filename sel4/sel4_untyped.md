# UNTYPED

## untyped cap 布局

![image-20231215110613207](../assets\image-20231215110613207.png)

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

![image-20231215210136571](../assets\image-20231215210136571.png)



untyped_retype流程图示：

![image-20231215210258358](../assets\image-20231215210258358.png)

三个棕色CNODE代表同一个进程的同一个cspace，