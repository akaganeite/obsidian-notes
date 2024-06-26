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

 

- [ ] std:async/await
- [ ] redis
- [ ] tock async executor

## safe os

- [x] 跑起来
  - [ ] 看现有的代码
- [ ] ntfn和ep：做一个初步的构建？再讨论。
- [ ] 细致了解theseus的channel/rust的channel，
- [x] 重温sel4的ep结构
- [ ] 看theseus的loadcrate调研报告


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