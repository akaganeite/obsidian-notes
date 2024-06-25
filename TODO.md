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

 

## panic堆资源调研

- [ ] rust-os-gdb
- [ ] 了解rust编译，看看Box底层实现
- [ ] 看一下theseus的exception handler
- [ ] 实验：把theseus是否正确释放资源搞清楚，通过查看堆内容/
  - 编译器优化问题，继续调研
  - 手动panic切换成unsafe触发一个页错误

实验设计：

- 手动触发一个page fault，参考unwinding test
- catch_unwind?
- 在sc deallocate的时候注入一个panic看一下资源释放情况
  - Gdb查看特定地址的值

## safe os

- [x] 跑起来
  - [ ] 看现有的代码

- [ ] ntfn和ep：做一个初步的构建？再讨论。
- [ ] 细致了解theseus的channel/rust的channel，
- [ ] 重温sel4的ep结构
- [ ] 看theseus的loadcrate调研报告

- [x] 看theseus和safeos的heap这块
  - [x] slab allocator
  - [ ] safeos具体实现有什么不同


## others

- capability的steal



一个使用场景：

- mm初始化之后,在某个ep上block_on_receive
- 其他组件需要map_page，向mm的ep发送请求，
- mm接到请求根据参数(地址，页数)做出相应操作，并返回
- mm继续block_on_receive





simple channel

https://github.com/NamiLiy/Theseus/commit/4f370d1ee29552f5d4e121f67e0b2f07200fdb43

fault injection

https://github.com/NamiLiy/Theseus/blob/fault_injection_artifacts/osdi20ae/fault_injection/README.md

unwinding现存的一个问题

https://github.com/theseus-os/Theseus/issues/199
