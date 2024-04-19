# 内核涉及endpoint发送的函数

```c
/*Z 引用cap_endpoint_cap能力的系统调用：发送IPC及extraCaps给EP队列第1个接收者 */
exception_t performInvocation_Endpoint(endpoint_t *ep, word_t badge,/*Z 能力的EP指针、标记 */
                                       bool_t canGrant, bool_t canGrantReply,/*Z 能力的2个字段 */
                                       bool_t block, bool_t call)/*Z 是否阻塞、是否Call调用 */
{
    sendIPC(block, call, badge, canGrant, canGrantReply, NODE_STATE(ksCurThread), ep);

    return EXCEPTION_NONE;
}
```

在sendIPC中查询ep的状态

- idle or send 且 非阻塞
  - 无事发生
- idle or send 且 阻塞
  - 进行一系列阻塞操作，将本线程tcb插入ep的等待队列
- recv
  - 从等待队列中取出一个线程，调用`doIPCTransfer` 将消息发送给接收线程
  - 消息包括，所有的MRs，extracaps，badge

