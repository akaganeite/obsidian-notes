# [Understanding Async Await in Rust: From State Machines to Assembly Code (eventhelix.com)](https://www.eventhelix.com/rust/rust-to-assembly-async-await/)

### 以下提到的实现不包括executor

```rust
goto(unit.clone(), 10).await;
//分解为goto函数调用，await调用
```

- goto函数：创建一个闭包，实现了poll函数，await相当于执行这个闭包,返回Future结构体(不是很理解)。结构体包含闭包的一些信息`closure environment`

  ```rust
  fn goto(unit: Unit, target_pos: i32) -> impl Future<Output = ()> {
      poll_fn(goto_closure)
  }
  ```

  > The `goto` function just returns the `Future` object. Calling the `goto` function does not execute the async function. The async function is executed when the `Future` object is `await`ed.

- await：执行先前创建的闭包`goto::{{closure}}`,在闭包中内联了自己实现的poll函数。

  - 额外添加了一个状态机管理Future,每次executor调度到本闭包都会根据当前状态选择分支。
  - wake()函数属于executor范畴，它让executor第二次选择本Future时状态会被推进(依据Rust book的简单channel实现)


### 总结

```rust
// Await this function until the unit has reached the target position.
async fn goto(unit: UnitRef, pos: i32) {
    UnitGotoFuture {
        unit,
        target_pos: pos,
    }
    .await;
}
goto(unit.clone(), 10).await;
```



- 在async内部进行await
  - An `await` in an async function typically results in a `poll` call on the future
- 仅调用async函数返回future，包装了一个闭包，由future的poll方法调用



# blog os keyboard int

```rust
static SCANCODE_QUEUE: OnceCell<ArrayQueue<u8>> = OnceCell::uninit();//scancode的队列
static WAKER: AtomicWaker = AtomicWaker::new();//来中断时唤醒的对象
//由int handler 调用
pub(crate) fn add_scancode(scancode: u8) {
    if let Ok(queue) = SCANCODE_QUEUE.try_get() {//如果有这个queue
        if let Err(_) = queue.push(scancode) {//放进queue中
            println!("WARNING: scancode queue full; dropping keyboard input");
        } else {
            WAKER.wake();//wake executor
        }
 

//最外层的fut
pub async fn print_keypresses() {
    while let Some(scancode) = scancodes.next().await
    //next返回scancodes的下一个fut，await这个fut，执行poll_next函数
    {}
}

fn poll_next(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Option<u8>> {
        let queue = SCANCODE_QUEUE.try_get();
        // fast path
        if let Some(scancode) = queue.pop() {
            return Poll::Ready(Some(scancode));
        }
        WAKER.register(&cx.waker());
        match queue.pop() {
            Some(scancode) => {
                WAKER.take();
                Poll::Ready(Some(scancode))
            }
            None => Poll::Pending,
        }
    }
```

- executor第一次poll`print_keypresses`时，会执行`poll_next`，第一次pop这个queue失败了，注册自己的waker，返回pending
- 来一个keyboard_int,转到`add_scancode`，把scancode加进queue里面，wake()
- executor再次poll`print_keypresses`,可以走fastpath,将scancode返回print_keypresses处理

# riscv int

## 中断时硬件完成U->S

1. 如果该 trap 是一个设备中断并且 `sstatus` 的 SIE bit 为 0，那么不再执行下述过程
2. 通过置零 SIE 禁用中断
3. 将 pc 拷贝到 `sepc`
4. 保存当前的特权级到 `sstatus` 的 SPP 字段
5. 将 `scause` 设置成 trap 的原因
6. 设置当前特权级为 supervisor
7. 拷贝 `stvec`（中断服务程序的首地址）到 pc
8. 开始执行中断服务程序

## safeos 做了什么

- `w_stvec((uint64)s_trap_vector);`,将中断处理程序首地址写在stvec寄存器中u->s
- `m_stvec((uint64)m_trap_vector);`,将中断处理程序首地址写在mtvec寄存器中s->m

### 来一个定时器中断

**时钟中断和软中断，它们默认被硬连线到M-Mode中断.首先陷入M-Mode下的中断处理程序，然后触发一个S-Mode下的中断再mret回S-Mode下处理**

- 转到s_trap_vector->s_trap
- s_trap分发中断，定时器--s_timer_trap
- s_timer_trap设置下一个定时器中断，再次ecall

- S->M

- do_ecall

- m_trap_vector,判断来源是ecall，处理ecall，处理过后mret，返回s_timer_trap

  - 处理ecall：现在只有针对定时器的一些操作：清除s模式中断挂起位，使能m模式timer中断，mepc自增

- `w_sstatus(r_sstatus() | SSTATUS_SIE);`

- 返回s_trap_vector，恢复现场，执行sret

  
