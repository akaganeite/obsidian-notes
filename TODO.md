- [ ] ntfn mail list

- [ ] 自己写一个systemxml，跑

- [ ] TREESLS

- [ ] sel4中断以及线程ntfn绑定的工作流程

- [x] 查看ntfn以及endpoint可以承载的最大消息量

- [x] server的启动流程，重点关注数据存储，通信机制和并发

- [x] client工作流程

- [ ] 了解mailbox的具体实现

- [ ] cantrip一个component怎么启动的，相互之间如何通信

- [x] camkes是否支持在运行时创建线程-应该是不支持

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

- ntfn无法携带除badge外的额外信息，可以使用endpoint传递信息(RPC)

  - ntfn的badge如何更好的使用？
- server和client的通信，使用一个多对一的通信？
  - server在run函数中轮询检查

