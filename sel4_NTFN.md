# NOTIFICATIONS

# NTFN的主要用途

1. 进程间通信
2. 中断控制

# 相关函数接口

## SYSCALL

### SIGNAL

LIBSEL4_INLINE_FUNC void seL4_Signal

Signal a notification.

|   Type    | Name |          Description          |
| :-------: | :--: | :---------------------------: |
| seL4_CPtr | dest | The capability to be invoked. |

Sel4_Send()的wrapper，syscall类型为syssend，实际调用的函数为

```c
x64_sys_send_null(seL4_SysSend, dest, seL4_MessageInfo_new(0, 0, 0, 1).words[0]);

static inline void x64_sys_send_null(seL4_Word sys, seL4_Word dest, seL4_Word info)
{
    asm volatile(
        "movq   %%rsp, %%rbx        \n"
        "syscall                    \n"
        "movq   %%rbx, %%rsp        \n"
        :
        : "d"(sys),
        "D"(dest),
        "S"(info)
        : "%rcx", "%rbx", "%r11"
    );
}
```

### WAIT

LIBSEL4_INLINE_FUNC void seL4_Wait

Perform a receive on a notification object.

|    Type     |  Name  |                         Description                          |
| :---------: | :----: | :----------------------------------------------------------: |
|  seL4_CPtr  |  dest  |                The capability to be invoked.                 |
| seL4_Word * | sender | The address to write sender information to.接收sender信息，不在syscall中传递，作为返回值，实际是发送方的badge |

Sel4-Recv的wrapper

#### 调用路径

`handleRecv`(syscall.c)->`receiveSignal`(notification.c)

### POLL

Perform a non-blocking recv on a notification object.

seL4_NBRecv()的wrapper

其他的和WAIT一样

## IRQ_HANDLER

static inline int `seL4_IRQHandler_SetNotification`

Set the notification which the kernel will signal on interrupts controlled by the supplied

IRQ handler capability

|      Type       |     Name     |                 Description                 |
| :-------------: | :----------: | :-----------------------------------------: |
| seL4_IRQHandler |   _service   |         The IRQ handler capability.         |
|    seL4_CPtr    | notification | The notification which the IRQs will signal |

![irq_set_ntfn](E:\note\typora\irq_set_ntfn.png)

## TCB

### **Bind Notification**

static inline int `seL4_TCB_BindNotification`

Binds a notification object to a TCB

|   Type    |     Name     |                    Description                    |
| :-------: | :----------: | :-----------------------------------------------: |
| seL4_TCB  |   _service   | Capability to the TCB which is being operated on. |
| seL4_CPtr | notification |               Notification to bind                |

### **Unbind Notification**

static inline int `seL4_TCB_UnbindNotification`

Unbinds any notification object from a TCB

|   Type   |   Name   |                    Description                    |
| :------: | :------: | :-----------------------------------------------: |
| seL4_TCB | _service | Capability to the TCB which is being operated on. |

### 调用路径

![ntfn](E:\note\typora\ntfn.png)

在bind操作中，TCB的cptr通过`_service`传递，是syscall第一个参数，NTFN的cptr存在ipcbuffer中，在`handleinvocation`中调用`lookupExtraCaps`函数从ipcbuffer中解析出ntfn的cap并对全局变量`extra_caps_t current_extra_caps;`进行赋值



