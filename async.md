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

  

