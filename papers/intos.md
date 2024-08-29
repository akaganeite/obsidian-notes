# TODO

- 确定文章的真正亮点是什么，我要讲的重点是什么
  - 首个支持preemptive调度的
  - 重点讲replay技术，结合undo-logging
  - replay是在控制流中加一些额外的分支
  - undo-logging是通过rust类型系统，在对nvm obj deref时候自动log一些东西
- 调研其他crash consistency是如何实现的
  - 什么是manual task decomposition
  - 为什么其他工作不支持multithread/timer/semaphore
- 本工作这样的transaction有什么好处，实质性的好处
  - 这个volatile的state是如何恢复的
- 这个工作focus embedded system，他对我们有什么启示吗(非embedded system)
- 用到了rust的什么特性
- 和PMDK的区别

# 讲解思路

- 以transaction的数据结构为核心，先展示数据结构
- 单线程，多线程的replay/recover
  - 对图稍微讲解一下
  - 讲解一下代码的recover流程
  - 在这里面要顺带将log的逻辑
    - syscall的macro
    - user/kernel tx
    - replay tables
- 利用rust deref实现的undo-logging
- 上述三者如何结合
  - 三个数据结构如何支持replay/recover
  - undo-logging，log到哪里，log什么变量

## 看什么代码

- recover
- replay
- undo-logging

# replay

- 整个task struct都被放在nvm中，作为crash recover的一个锚点
- 用图讲一下大体思路
- 结合简单代码和数据结构讲解具体的实现
- 从三个数据结构入手

## syscall_replay_cache

- 存放syscall的返回值，若没有返回值存空值占位，16字节对齐
- 开始syscall时首先检查cache中是否有值，若有则直接返回
  - 若没有值，代表上次没有执行syscall or syscall结果没有写入cache
  - 陷入内核，准备运行syscall代码
- syscall结束后将结果写入cache，并清空对应的journal，journal只会记录一个transaction中的pm value
  - 在写入结果后ptr=tail，标记syscall结束

## syscall_tx_cache

- 每次syscall开始时将其清除
- syscall结束时写入，表示syscall的结果写入了replay_cache

## user_tx

- 任意process使用transaction::run_sys()发起一个事务时，该事物会被记录在user_tx中
- 在开始事物时检查user_tx,的cache。若有结果，则直接返回。
  - 否则，清空replay_cache
  - 运行run_sys传入的闭包，将返回值写入user_tx的cache中，清空tx的journal

## tx

- 原理同user_tx,只不过是syscall陷入内核后，内核记录transaction的数据结构

# recover

- multithread时在每个线程被switch到的时候做recover

- 在正式recover之前：将系统恢复运行

  ```rust
  pub fn recover_and_boot() {
      use crate::{recover::current_generation, task::reset_scheduler_started};
      debug_print!("Booting..., gen={}", current_generation());
      #[cfg(feature = "power_failure")]
      {
          if current_generation() == 0 {
              arch_config_cycle_count();
          } else {
              #[cfg(board = "msp430fr5994")]
              crate::board::msp430fr5994::peripherals::setup_timer_interrupt();
          }
      }
      #[cfg(not(feature = "power_failure"))]
      arch_config_cycle_count();
      unsafe { reset_scheduler_started() };
      increase_generation();
      init_boot_tx();//初始化一个transaction
      // before doing anything, run a recovery protocal
      recover();
      run_boot_sequence();
  }
  
  ```

  

- 由recover函数实现

```rust
if in_ctx_switch_tx() {
  // no list tx here 切换现场也是一个transaction
  debug_print!("Recovering from unfinished context switch transaction...");
  let tx = get_boot_tx();
  tx.roll_back_if_uncommitted();
  exit_all_ctx_switch_tx();
} else {
  // outstanding system call transaction ?
  // If a system-call  is just finished...
  debug_print!("Recovering from unfinished syscall transaction...");
  let tx = current().get_mut_tx();
  if is_in_critical() {
      tx.roll_back_if_uncommitted();
      exit_all_critical();
  }
}

if !current().is_schedulable() {
    crate::os_dbg_print!("Reschedule before start....");
    unsafe {
        critical::with_no_interrupt(|_| {
            task_switch();
        });
    }
} else {
    current().jit_recovery();
}
```

- `in_ctx_switch_tx`获取nvm变量`IN_CONTEXT_SWITCH_TX`是否为0,是否正在上下文切换
- jit_recovery将没有commit的记录恢复，具体操作是将nvm中log的对象状态加载至内存中

# undo-logging

## journal

- journal记录所有的nvm object的值和地址，记录是自动进行的，通过在deref中加入append_log_of

### 日志内容示例

| 偏移量 | 内容          | 描述            |
| ------ | ------------- | --------------- |
| 0      | `42`          | 对象 `A` 的数据 |
| 8      | 地址 `0x1000` | 对象 `A` 的地址 |
| 16     | 大小 `4`      | 对象 `A` 的大小 |
| 24     | `3.14`        | 对象 `B` 的数据 |
| 32     | 地址 `0x2000` | 对象 `B` 的地址 |
| 40     | 大小 `8`      | 对象 `B` 的大小 |

```
| Object Data | Aligned Padding | Object Address | Object Size |
      ↓
| Object Data | Aligned Padding | Object Address | Object Size |
      ↓
| Object Data | Aligned Padding | Object Address | Object Size |
      ↓
   (Tail Pointer)

```

# chapter 1 introduction of nvm

以文中所用的texas instrucment的嵌入式开发板为例。

`MSP430`

- FRAM is a nonvolatile memory that reads and writes like standard SRAM.
- FRAM (rwx) : ORIGIN = 0x4000, LENGTH = 0xBF80 /* END=0xFF7F, size 49024 */，在链接脚本中定义的nvm范围

# chapter 2 intro of the paper

- focus on crash consistency for embedded systems with the need of intermittent computing
  - Intermittent: unexpected power failure can happen at any time
- 
