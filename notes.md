# COMPONENT LOG

## Client:cantrip-os-common->logger

```rust
//logger/src/lib.rs
pub struct CantripLogger {
    endpoint: sel4_sys::seL4_CPtr,
    buffer: Mutex<&'static mut [u8]>,
}
//processmanager->component->run.rs展开后:
static CANTRIP_LOGGER: CantripLogger = CantripLogger::new(
    LOGGER_INTERFACE_ENDPOINT,
    &mut LOGGER_INTERFACE_DATA.data,
);
//设置全局logger
log::set_logger(&CANTRIP_LOGGER).unwrap();
log::set_max_level(log::LevelFilter::Trace);
```

`CantripLogger`实现了`log`crate 的`Log`trait，使其能作为日志记录器使用。这个结构体包含两个字段：

1. **`endpoint`**：类型为 `sel4_sys::seL4_CPtr`，表示与 seL4 内核通信的端点。这通常用于 IPC 通信，允许日志消息被发送到另一个组件或线程进行处理。
2. **`buffer`**：一个互斥锁包装的静态可变字节切片`Mutex<&'static mut [u8]>`，用于存储日志消息。这个缓冲区用于构建要发送的日志消息。

`CantripLogger`提供了一个`const fn new`函数用于创建实例，以及实现`log::Log` trait 的方法（`enabled`, `log`, 和 `flush`）来处理日志记录逻辑。

在component的run.rs中，使用macro实例化一个静态CANTRIP_LOGGER,每次要打log就用sel4_Call给debug_console发消息，要发送的消息存在buffer中，我推测是这个东西的ipc_buffer

> 实现log trait的log方法，最终是调用sel4_Call打log
>
> ```rust
> let _ = postcard::to_slice(
>                 &LoggerRequest::Log {
>                     level: record.level() as u8,
>                     msg: unsafe { from_utf8_unchecked(&buf[..pos]) },
>                     // XXX Ok a hack
>                 },
>                 *self.buffer.lock(),
>             )
>             .map(|_| unsafe {
>                 sel4_sys::seL4_Call(
>                     self.endpoint,
>                     sel4_sys::seL4_MessageInfo::new(
>                         /*label=*/ 0, /*capsUnwrapped=*/ 0, /*extraCaps=*/ 0,
>                         /*length=*/ 0,
>                     ),
>                 )
>                 .get_label()
>             });
> ```
>
> self.endpoint在每个component生成的camkes.rs中有定义，
>
> pub **static** LOGGER_INTERFACE_ENDPOINT: sel4_sys::seL4_CPtr = 23;

## Server:component->cantrip-debug-console

cantrip=debug-console->run.rs

```rust
//sel4_Call的接收端
struct LoggerInterfaceThread;
impl CamkesThreadInterface for LoggerInterfaceThread {
    fn run() {
        rpc_shared_recv!(logger, logger::MAX_MSG_LEN, LoggerError::Success);
    }
}
//设置logger
log::set_logger(&LoggerInterfaceThread).unwrap();
```

## Connection:system.camkes

```rust
// Connect the LoggerInterface to each component that needs to log
// to the console. Note this allocates a 4KB shared memory region to
// each component and copies data between components.
connection cantripRPCOverMultiSharedData multi_logger(
    from process_manager.logger,
    from memory_manager.logger,
    from security_coordinator.logger,
    from sdk_runtime.logger,
    to debug_console.logger);
```





# 随手记

- cantrip/projects/camkes-tool/camkes/parser/tests/bad-at-s6/std_connector.camkes

```rust
connector cantripRPCCall {
    from Procedures with 0 threads;
    to Procedure;
    attribute string isabelle_connector_spec = "seL4RPC"; /* builtin connector */
}
```

高并发 安全事件 linux安全事件->安全芯片处理->高并发机制->基于sel4

- sel4社区 mail list-高并发实践讨论，高效处理多个notification
- serverless->把服务当成函数，服务-事件机制-funciton，`事件驱动`相关论文
- 看透notification，和signal





