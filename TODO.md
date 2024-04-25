- [ ] ntfn mail list

- [ ] 自己写一个systemxml，跑

- [ ] TREESLS

- [ ] sel4中断以及线程ntfn绑定的工作流程

- [x] 查看ntfn以及endpoint可以承载的最大消息量

- [x] server的启动流程，重点关注数据存储，通信机制和并发

- [x] client工作流程

- [ ] 了解mailbox的具体实现

- [ ] component怎么启动流程·，ctrl->interface

- [x] NATS的client和server分别如何工作，三种工作模式的实现路径

- [ ] memory manager源码

  





# 问题

## server

- subscription在cantrip中由什么代替
- server.client扮演什么角色

## client

- 完成任务的client是component吗？是component 
- mailbox作为一个client，在初始化时订阅全部任务相关的sublist？
- 完成特定任务的app在初始化时等待



- [ ] 是否需要PUB/SUB类型的通信方式
- [ ] 一个任务类型(sublist)是否只有一个线程订阅，相应的，队列组是否需要
- [ ] mailbox的事件处理逻辑
  - 收到一个完整的任务请求
  - 以PUB/REQUEST的消息格式通过ep唤醒server
  - server创建一个线程处理这个问题
- [ ] server如何处理请求
  - [ ] 创建新的线程？是否可行？创建时需要消耗部分内核资源，进行共享内存映射(传递sublist)，是否必要？

  - [ ] 若server自己执行请求则可能导致忙等？，并发度降低，需要进一步调研

- [ ] request/reply模式如何更改

  - NATS：client在监听时就需要传入reply内容
  - NATS：如果有多个client注册了reply消息，那么只有一个client会被server选中，发送回复








# 实现细节

- nats的server端会为每一个client的连接创建一个goroutine(轻量级线程)；cantrip中一个connection对应一个interfacethread.
- mailbox发送消息，经由server解析并分发给完成具体任务的线程，任务完成后线程发送消息，经由server传递给mailbox
- client向server通信，使用一个多对一的通信(RPCCall),由server提供
  - from-clients to-server，多生产者，单消费者
  - server使用状态机处理请求
    - SUB：更新sublist
    - PUB：查找订阅sublist的线程，并转发消息
  - server收到消息，解析出目的component，直接转发
    - 以ima为例，server收到mailbox发来的`PUB ima MSG`消息，直接使用`rpc_basic_send!(ima,...)`即可传递消息，ima为camkes中声明的conection
- server向client通信，一对一通信，由client提供，camkes中声明的
  - client负责通信的interfacethread调用rpc_basic_recv
  - server在解析到PUB消息时向对应的client调用rpc_basic_send，以预先约定的格式传递信息
  - client.interfacethread收到消息后执行任务，任务完成后发送消息给server

- PUB或SUB的消息由rpc_basic_buffer传递



# 问题

- sublist
  - NATS使用sublist可以存储并快速定位一个订阅队列，向队列中的每一个client发送消息
  - cantrip的component间通信使用一endpoint为基础的RPCCall，server无需sublist，可以直接根据mailbox发来的任务信息定位相关endpoint并与目标component通信
- 消息类型
  - SUB类型消息：cantrip中完成特定任务的component是确定的。以完整性度量为例，该component在camkes中声明与server通信的RPCCall类型connection，建立用于通信的endpoint。那么就无需向server发送如SUB ima这样的消息，server收到mailbox发来的ima请求时直接通过camkes约定好的ep进行消息传递即可。
  - 同样，一个endpoint支持的RPCCall是单向的，不同endpoint上的通信可以区分消息类型，无需PUB类型的消息


# 实现

- nats的server端会为每一个client的连接创建一个goroutine(轻量级线程)；cantrip中一个connection对应一个interfacethread.
- mailbox发送消息，经由server解析并分发给完成具体任务的线程，任务完成后线程发送消息，经由server传递给mailbox
- server向client分发事件，一对一通信，由client提供，每个client都与server通过一个RPCCall通信
  - client负责通信的interfacethread调用rpc_basic_recv
  - server在解析到PUB消息时向对应的client调用rpc_basic_send，以预先约定的格式传递信息
  - client.interfacethread收到消息后执行任务，任务完成后发送消息给server
- server接收client发来的事件，使用一个多对一的通信(RPCCall),由server提供
  - from-clients to-server，多生产者，单消费者
  - server使用状态机处理请求
    - SUB：更新sublist
    - PUB：查找订阅sublist的线程，并转发消息
  - server收到消息，解析出目的component，直接转发
    - 以ima为例，server收到mailbox发来的`ima MSG`消息，直接使用`rpc_basic_send!(ima,...)`即可传递消息，ima为camkes中声明的conection
- rpc_basic_buffer传递消息
