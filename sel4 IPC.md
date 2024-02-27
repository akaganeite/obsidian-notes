# Sel4 notes

- 内核只提供mechanism。不提供服务
- OS服务由受保护的用户级服务进程提供
- 这种服务通过IPC机制唤起

# IPC通过`endpoints`进行握手

<img src="E:\note\typora\assets\image-20231116103411112.png" alt="image-20231116103411112" style="zoom:80%;" />

- 信息通过`message registers`传递
- seL4允许在同一个endpoint使用不同的badge(标记)，用Mint（）方法进行标记，

# VSPACE

- 页目录（pd）映射页表，页表项映射页表
- 一个VSPACE由一个PD pbject代表，同生同灭

# Capabilities

- a pointer with access rights
- 内核控制的caps和所有的资源都在初始化时给了root

```c
seL4_TCB_ReadRegisters(seL4_CapInitThreadTCB, 0, 0, num_registers, &registers);
//seL4_CapInitThreadTCB代表一个capability，具体是，在inittask的cnode中寻址槽(SLOT)seL4_CapInitThreadTCB
//implicitly looks up the seL4_CapInitThreadTCB CSlot in the CNode pointed to by the //CSpace root capability of the calling thread, which here is the root task.
```

## category

- control access to kernel objects , TCB
- control access to abstract resources,IRQcontrol
- untyped caps, for memory management, allocation

## CNodes CSlots CSpaces

- CNodes 可以被理解为一个capabilities数组，数组中的每一项是一个CSlot，`seL4_CapInitThreadTCB`是root task CNode中拥有TCB能力的slot。
- 0号槽永远是空的
- A *CSpace* (capability-space) is the full range of capabilities accessible to a thread, which may be formed of one or more CNodes.
- Each thread has a special CNode capability installed in its TCB as its *CSpace root*.每个进程TCB都有能力空间根节点，指向一个cnode

# SYCALL的调用流程

- 以seL4_CNodeCopy为例

seL4_CNode_Copy->seL4CallwithMRs-><plat>_sys_send_recv->handleSyscall->handleInvocation->decodeInvocation(cnode_cap)->decodeCNodeInvocation(inv_label=19==CNodeCopy)->invokeCNodeInsert->cteInsert

- 在syscall路径中的inv_label一直是第一个参数，从seL4_MessageInfo_t中获取，在CNode_Copy函数中被设置为19，CNodeCopy

  ```c++
  seL4_MessageInfo_t tag = seL4_MessageInfo_new(CNodeCopy, 0, 1, 5);
  ```

  tag是send_recv的第四个参数，`msgInfo.words[0]`,在syscall传递寄存器时被存入，在handleInvocation函数中被取回

  ```c
  info = messageInfoFromWord(getRegister(thread, msgInfoRegister));
  ```

- decodeCNodeInvocation中进入`case CNodeCopy`，执行到cteinsert函数对cnode进行操作

  - cte(capability table entry),就是cnode中的一个slot，包含cap_t和mdb_node_t.`CNode[cptr] = CTE`,`*slot = CTE`.

<img src="E:\note\typora\assets\image-20231117215001268.png" alt="image-20231117215001268" style="zoom:67%;" />

- cteInsert是这个系统调用最核心的函数，对cnode的一个slot进行复制

  - `mdb_node_ptr_set_mdbNext`,`mdb_node_ptr_set_mdbPrev`是用于构建CDT tree，内核用这棵CDT来管理cap的继承关系

    <img src="E:\note\typora\assets\image-20231117225723549.png" alt="image-20231117225723549" style="zoom:80%;" />

```c
void cteInsert(cap_t newCap, cte_t *srcSlot, cte_t *destSlot)
{
    mdb_node_t srcMDB, newMDB;
    cap_t srcCap;
    bool_t newCapIsRevocable;

    srcMDB = srcSlot->cteMDBNode;//mdb_node
    srcCap = srcSlot->cap;//cap_t

    newCapIsRevocable = isCapRevocable(newCap, srcCap);

    newMDB = mdb_node_set_mdbPrev(srcMDB, CTE_REF(srcSlot));//set new according to old  
    newMDB = mdb_node_set_mdbRevocable(newMDB, newCapIsRevocable);
    newMDB = mdb_node_set_mdbFirstBadged(newMDB, newCapIsRevocable);

    /* Haskell error: "cteInsert to non-empty destination" */
    assert(cap_get_capType(destSlot->cap) == cap_null_cap);
    /* Haskell error: "cteInsert: mdb entry must be empty" */
    assert((cte_t *)mdb_node_get_mdbNext(destSlot->cteMDBNode) == NULL &&
           (cte_t *)mdb_node_get_mdbPrev(destSlot->cteMDBNode) == NULL);

    /* Prevent parent untyped cap from being used again if creating a child
     * untyped from it. */
    setUntypedCapAsFull(srcCap, newCap, srcSlot);

    destSlot->cap = newCap;
    destSlot->cteMDBNode = newMDB;
    mdb_node_ptr_set_mdbNext(&srcSlot->cteMDBNode, CTE_REF(destSlot));
    if (mdb_node_get_mdbNext(newMDB)) {
        mdb_node_ptr_set_mdbPrev(
            &CTE_PTR(mdb_node_get_mdbNext(newMDB))->cteMDBNode,
            CTE_REF(destSlot));
    }
}
```

# IPC

## Message Info

- `length`: message中的MR个数
- `extraCaps`:message中的能力们
- `capsUnwrapped`:标记unwrapped能力
- `label`:data that is transferred unmodified by the kernel from sender to receiver

## seL4_Recv

调用后转至内核函数：`handleRecv(bool_t isBlocking)`,在syscall.c中，由系统调用统一处理函数`handleSyscall`调用，

```c
switch (cap_get_capType(lu_ret.cap)) {
    case cap_endpoint_cap:
        deleteCallerCap(NODE_STATE(ksCurThread));
        receiveIPC(NODE_STATE(ksCurThread), lu_ret.cap, isBlocking);
        break;
```

删除暂时的callercap，调用`void receiveIPC(tcb_t *thread, cap_t cap, bool_t isBlocking)`函数参数为(current_thread,endpoint_cap,true)，根据endpoint中的state域来进行不同的操作，如果ep现在是有消息的，那么

```c
queue = ep_ptr_get_queue(epptr);//拿到等待队列
sender = queue.head;//队列头是sender
/* Dequeue the first TCB */
queue = tcbEPDequeue(sender, queue);//从队列上摘除
ep_ptr_set_queue(epptr, queue);//把队列再赋值给ep

if (!queue.head) {//如果没有排队的进程那就把ep设置成idle状态
    endpoint_ptr_set_state(epptr, EPState_Idle);
}

/* Get sender IPC details */
badge = thread_state_ptr_get_blockingIPCBadge(&sender->tcbState);
canGrant =thread_state_ptr_get_blockingIPCCanGrant(&sender->tcbState);
canGrantReply =thread_state_ptr_get_blockingIPCCanGrantReply(&sender->tcbState);
/* Do the transfer */
doIPCTransfer(sender, epptr, badge,canGrant, thread);
do_call = thread_state_ptr_get_blockingIPCIsCall(&sender->tcbState);
```



调用doIPCTransfer对两个thread的IPCbuffer进行复制。其余和ep相关的系统调用也大致相同。

## endpoint

- ep就是一个拥有队列首尾指针的内存区域，指针指向等待在这个ep上的tcb们，ep拥有一个state域，根据state不同在syscall的时候做出不同的响应
- ep并不是能力！能力是访问ep的权限，存储在tcb的cslot中

## ipc流程

为了简单起见，举一个简单但非常实用的例子：有两个进程，一个进程发送信息，另外一个进程接收信息。具体流程如下：

1. 双方进程申请endpoint，得到endpoint的cap。两个进程的cap是同一个cap，该方法由sel4提供的capdl（capability description language）实现；
2. 进程1执行seL4_Send，该函数从ipc buffer里面取出数据放到寄存器x2-x5里，并发起系统调用；
3. 然后执行sendIPC函数，该函数判断endpoint的状态，此时为idle状态（还没有进程在endpoint处等待），且blocking为True。故将当前进程1挂载到endpoint的队列上，置endpoint的状态为send状态；
4. 进程2执行seL4_Recv，发起系统调用；
5. 然后执行receiveIPC函数，该函数判断endpoint的状态，此时为send状态（说明进程需要的信息已经发出了）。故将endpoint的队列取出进程1，读取发送的message和badge给进程2；
6. 完成。

# Badge

- 服务器线程可以通过`mint()`操作将客户端的capabilities和一个badge绑定，`mint()`将一个未badge的epcap加上badge后赋值给新的empty capslot
- 允许对同一个ep标记不同的badge

从cnode_mint看badge，

```c
//syscall_stub
seL4_CNode_Mint(seL4_CNode _service, seL4_Word dest_index, seL4_Uint8 dest_depth, seL4_CNode src_root, seL4_Word src_index, seL4_Uint8 src_depth, seL4_CapRights_t rights, seL4_Word badge)
//buffer中的6号MR是badge
seL4_SetMR(5, badge);
//剩余调用路线和cnodecopy相同
//cnode.c:decodeCNodeInvocation
case CNodeMint:
    cap_rights = rightsFromWord(getSyscallArg(4, buffer));
    capData = getSyscallArg(5, buffer);//获得badge
    srcCap = maskCapRights(cap_rights, srcSlot->cap);
    dc_ret = deriveCap(srcSlot,updateCapData(false, capData, srcCap));
    newCap = dc_ret.cap;
    isMove = false;
    break;
//最后调用该函数插入slot
invokeCNodeInsert(newCap, srcSlot, destSlot);
```

ref manual中的描述：

>Endpoint capabilities may be *minted* to create a new endpoint capability with a *badge*attached to it, a data word chosen by the invoker of the *mint* operation. When amessage is sent to an endpoint using a badged capability, the badge is transferred tothe receiving thread’s badge register.
>
>An endpoint capability with a zero badge is said to be *unbadged*. Such a capabilitycan be badged with the seL4_CNode_Mutate() or seL4_CNode_Mint() invocationson the CNode containing the capability. 
>
>Endpoint capabilities with badges cannot beunbadged, rebadged or used to create child capabilities with different badges.

# UNTYPED

- 除去用于创建root task的对象，所有可用物理内存的capabilities都以untyped memory的形式传给root task。
- untyped memory是一块有特定大小的连续物理内存，管理权限是untyped caps

## SYSCALL

- seL4_Untyped_Retype函数，具体描述如下

  ```c
  seL4_Untyped_Retype(parent_untyped, // the untyped capability to retype,要retype的cap
                                  seL4_UntypedObject, // type，retype后的类型
                                  untyped_size_bits,  //size，类型的大小
                                  seL4_CapInitThreadCNode, // root，cnode
                                  0, // node_index
                                  0, // node_depth index的depth用于指定cnode
                                  child_untyped, // node_offset，在cnode中选定cslot
                                  1 // num_caps
                                  );
  ```

- MR和IPCBuffer设置：

  ```c++
  seL4_SetCap(0, root);//ipcbuffer->ipc_or_badge
  /* Marshal and initialise parameters. */
  mr0 = type;
  mr1 = size_bits;
  mr2 = node_index;//0
  mr3 = node_depth;//0
  seL4_SetMR(4, node_offset);//slot_id
  seL4_SetMR(5, num_objects);//1
  ```

  

- 调用路径

  `seL4_Untyped_Retype->seL4_CallWithMRs->x64_sys_send_recv->handleSyscall->handleInvocation->decodeInvocation->decodeInvocation->decodeUntypedInvocation->invokeUntyped_Retype->createNewObjects`

  ```c
  //syscall.c:handleInvocation
  decodeInvocation(seL4_MessageInfo_get_label(info), //seL4_MessageInfo_new(UntypedRetype,...);UntypedRetype==1
                   length,//6
                   cptr, //thread->tcbArch.tcbContext.registers[0];
                   lu_ret.slot, //lu_ret,通过解析地址获得的一些东西
                   lu_ret.cap,
                   isBlocking,//true
                   isCall, //true
                   buffer//当前进程的ipcbuffer
  );
  //objecttype.c:decodeInvocation
  case cap_untyped_cap:
          return decodeUntypedInvocation(invLabel, length, slot, cap, call, buffer);
  //untyped.c:decodeUntypedInvocation
  if (userObjSize >= wordBits || objectSize > seL4_MaxUntypedBits)//太大了，装不下
  if (newType == seL4_CapTableObject && userObjSize == 0)//大小不对
  if (newType == seL4_UntypedObject && userObjSize < 4)//untyped最小为2^4
  invokeUntyped_Retype(slot, reset,
                       (void *)alignedFreeRef, newType, userObjSize,
                       destCNode, nodeOffset, nodeWindow, deviceMemory);
  //objecttype.c:invokeUntyped_Retype
  createNewObjects(newType, srcSlot, destCNode, destOffset, destLength,
                       retypeBase, userSize, deviceMemory);
  
  ```
  

123123

# fault handle

- 出现fault的时候kernel停止thread并试图通过和thread绑定的特殊ep发送消息。监听这个ep的thread叫做fault handler，用于处理错误并在处理后告诉kernel重新调度已经被停止的进程

## fault ep

# CAMKES

## Component 

component是代码和资源的逻辑组，component之间可以通过静态定义的接口进行通信

`在component中做出control声明的comp会优先被运行，运行入口为该component的run函数`

## Interface

- RPC:同步交互
- Event:notification
- Dataport：共享数据

### RPC

通过SeL_Call实现，属于synchronous IPC，也被称为procedural interface。是一个函数调用的接口集合使用关键字`procedure`，`Procedures consist of a series of methods that can be invoked independently`。

### Connector

```c
connection seL4RPCCall hello_RPC(from client.hello, to echo.hello);
connection seL4RPC hello_RPC(from client.hello, to echo.hello); 
```

## Events

对应tutorial-camkes2，events是interface的一种，由实现不同component之间的异步交流

### 发送

`emit`，用在component中的关键字，代表在这个comp中提供这一事件。

```camkes
emits MyEvent e;
 e_emit();
```

代表emit类型为MyEvent的事件e。在c代码中使用<instance>_emit()的方式来送出一个event。

类型名并不重要，发送接收方通过instance的名称通信

### 接收

接收使用关键字`consumes`,

```camkes
 consumes MyEvent s;
```

接收方式有三种

```c
s_reg_callback(&handler);//handler是一个函数指针
s_poll()
s_wait();
```

`reg_callback`在入参中给定event被emit时要唤醒的handler。这只是一个登记功能，在下一次有s接收到event时直接唤醒handler。在handler被唤醒时先前在入参指定的handler失效，想要在下一个event来时继续使用当前handler则必须再用reg_callback显式声明一次。__callback登记的方式在event来时拥有最高优先级__。

`poll`只进行查询，并返回是否有待consume的event

`wait`进行阻塞等待，有event后返回

### Connector

```camkes
connection seL4Notification channel(from source.e, to sink.s);
```

类型是notification，代表着异步通信，通道实例名称为channel，发送端是emits声明的e，接收端是consumes声明的s

两个线程可以通过notification机制实现同步

## Dataports

对应tutorial-camkes2.Dataports是camkes对共享内存的抽象

dataport的默认类型是Buf，在底层是C语言byte数组，大小为PAGESIZE。dataport的类型也可以是用户自定义的。

```c
example.h:
typedef struct MyData {
  char data[10];
  bool ready;
} MyData_t;//用户自定

example.camkes:
component a{
    include "example.h";
    dataport Buf d1;//buf类型
    dataport MyData_t d2;//自定义类型
}
```

camkes可以引用头文件定义的结构体，只需在component中显式include就可以

### 同步

`*_acquire()`在对dataport的多个读操作之间使用

> An acquire memory fence. Any read from the dataport preceding this fence in program order will take place before any read or write following this fence in program order.

`*_release()`在多个写操作之间使用

> A release memory fence. Any write to the dataport following this fence in program order will take place after any read or write preceding this fence in program order.

### 使用

在component中定义的名称可以直接使用，类型是指针。

`dataport_ptr_t`指向dataport的指针结构，可以在comp之间进行传递

**`dataport_wrap_ptr(void ptr)`** **`dataport_unwrap_ptr(dataport_ptr_t ptr)`**

对指针进行封装与解封将普通的指向dataport数据的指针封装为dataport_ptr,解封进行逆操作，封装后的指针可以被放在dataport中进行传输，也可通过procedure接口传给接收component

## Connector

实例化的Connector被称为connection

定义方式：

`connection <Connector> <conn_name>(from <comp>.<inf>,to <comp>.<inf>);`	





# TODO

1. lu_ret
