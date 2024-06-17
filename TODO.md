rust语言的安全特性进行隔离，性能提升了。



关注sel4的panic->fault handler的tut

劣势：

- 隔离强度变弱了，和地址空间隔离相比，现在在一个地址空间内
- 关注panic，panic还是会影响其他组件。对panic进行分类
  - 不会因为没有定义的行为发生panic(越界)
  - 对定义行为上发生panic，整个系统范围panic
  - 控制上述定义行为的panic，就算发生了
  - 解决：把上层所有组件的defined panic截获到底层来做。

## safe os

- 同一地址空间
- capabilities机制
- sel4内核对象迁移（ntfn ep）
  - ep：是否需要---theseus如何实现的
- fault isolution：
  - sel4 进程隔离
  - panic导致整个kernel崩溃---kernel的tcb承载所有的故障
  - defined panic：传递到微内核，决定下一步的操作，不影响其他的sub system
- 移植内核服务，去掉所有unsafe，接口移植

 

- [x] mmstruct和函数整理
- [ ] panic堆资源调研
  - [ ] 了解rust编译，看看Box底层实现
  - [ ] 实验：把theseus是否正确释放资源搞清楚，通过查看堆内容
- [ ] safe os
  - [ ] 跑起来，看现有的代码
  - [ ] ntfn和ep：做一个初步的构建？再讨论。
  - [ ] 细致了解theseus的channel/rust的channel，
  - [ ] 看theseus的loadcrate调研报告

0xfffffe80047fe000
