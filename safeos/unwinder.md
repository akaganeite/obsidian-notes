

# lib_unwinder

## a rough routine

- Panic_handler:panic
- do_panic
- panic.rs:begin_panic
- Panicking.rs:begin_panic
- _Unwind_RaiseException
- with_context
  - raise_exception_phase2


## _Unwind_RaiseException

这个 Rust 代码实现了 `_Unwind_RaiseException` 函数，这是一个异常处理机制的一部分。该函数分为两个阶段（phase），搜索异常处理程序和实际展开栈。以下是详细解析：

### 函数签名

```rust
pub unsafe extern "C-unwind" fn _Unwind_RaiseException(
    exception: *mut UnwindException,
) -> UnwindReasonCode
```

- `pub unsafe extern "C-unwind"`：该函数使用 C 调用约定，并允许栈展开（异常传播）。
- `exception: *mut UnwindException`：指向异常信息的指针。
- 返回类型为 `UnwindReasonCode`，表示异常处理的结果。

### with_context

`with_context` 函数用于获取并保存当前的上下文状态。传递一个闭包，在闭包中执行异常处理逻辑。

- 这个函数将传入的闭包转成function pointer(`f`)的形式，调用`save_context`
- `save_context`保存当前上下文，call函数`f`->使用asm！完成
- 这个`f`就是下面说的程序

### Phase 1: 搜索处理程序

```rust
let mut ctx = saved_ctx.clone();
let mut signal = false;
loop {
    if let Some(frame) = try1!(Frame::from_context(&ctx, signal)) {
        if let Some(personality) = frame.personality() {
            let result = unsafe {
                personality(
                    1,
                    UnwindAction::SEARCH_PHASE,
                    (*exception).exception_class,
                    exception,
                    &mut UnwindContext {
                        frame: Some(&frame),
                        ctx: &mut ctx,
                        signal,
                    },
                )
            };

            match result {
                UnwindReasonCode::CONTINUE_UNWIND => (),
                UnwindReasonCode::HANDLER_FOUND => {
                    break;
                }
                _ => return UnwindReasonCode::FATAL_PHASE1_ERROR,
            }
        }

        ctx = try1!(frame.unwind(&ctx));
        signal = frame.is_signal_trampoline();
    } else {
        return UnwindReasonCode::END_OF_STACK;
    }
}
```

1. **初始化上下文和信号标志**：
   - `let mut ctx = saved_ctx.clone();`
   - `let mut signal = false;`

2. **循环处理每个栈帧**：
   - 使用 `Frame::from_context` 获取当前上下文中的栈帧。
   - 如果栈帧存在，尝试获取其个性化处理函数（personality function）。

3. **调用 personality 函数**：
   - `personality` 函数用于判断当前栈帧是否包含异常处理程序。
   - 如果找到异常处理程序（`HANDLER_FOUND`），则跳出循环。
   - 否则继续搜索下一个栈帧，直到栈的末尾（`END_OF_STACK`）。

### Disambiguate normal frame and signal frame

```rust
let handler_cfa = ctx[Arch::SP] - signal as usize;
unsafe {
    (*exception).private_1 = None;
    (*exception).private_2 = handler_cfa;
}
```

- 计算处理程序的帧地址（CFA）。
- 更新异常对象中的私有字段。

### Phase 2: 展开栈并调用处理程序

```rust
let code = raise_exception_phase2(exception, saved_ctx, handler_cfa);
match code {
    UnwindReasonCode::INSTALL_CONTEXT => unsafe { restore_context(saved_ctx) },
    _ => code,
}
```

- 调用 `raise_exception_phase2` 进行第二阶段的异常处理，展开栈并调用处理程序。
- 根据返回的 `UnwindReasonCode` 执行相应的操作。

### 具体解析

```rust
with_context(|saved_ctx| {
    // 阶段 1：搜索异常处理程序
    let mut ctx = saved_ctx.clone();
    let mut signal = false;
    loop {
        // 尝试从上下文中获取当前栈帧
        if let Some(frame) = try1!(Frame::from_context(&ctx, signal)) {
            // 获取栈帧的个性化处理函数
            if let Some(personality) = frame.personality() {
                let result = unsafe {
                    personality(
                        1,
                        UnwindAction::SEARCH_PHASE,
                        (*exception).exception_class,
                        exception,
                        &mut UnwindContext {
                            frame: Some(&frame),
                            ctx: &mut ctx,
                            signal,
                        },
                    )
                };

                // 根据 personality 函数的结果决定下一步操作
                match result {
                    UnwindReasonCode::CONTINUE_UNWIND => (),
                    UnwindReasonCode::HANDLER_FOUND => {
                        break;
                    }
                    _ => return UnwindReasonCode::FATAL_PHASE1_ERROR,
                }
            }

            // 更新上下文，处理下一个栈帧
            ctx = try1!(frame.unwind(&ctx));
            signal = frame.is_signal_trampoline();
        } else {
            return UnwindReasonCode::END_OF_STACK;
        }
    }

    // 区分正常帧和信号帧
    let handler_cfa = ctx[Arch::SP] - signal as usize;
    unsafe {
        (*exception).private_1 = None;
        (*exception).private_2 = handler_cfa;
    }

    // 阶段 2：展开栈并调用处理程序
    let code = raise_exception_phase2(exception, saved_ctx, handler_cfa);
    match code {
        UnwindReasonCode::INSTALL_CONTEXT => unsafe { restore_context(saved_ctx) },
        _ => code,
    }
})
```

### 总结

- **阶段 1**：搜索异常处理程序，通过遍历栈帧并调用 personality 函数来找到异常处理程序。
- **阶段 2**：展开栈并调用找到的异常处理程序。
- `extern "C-unwind"` 允许函数在发生异常时进行栈展开，确保异常能够正确传播和处理。



# Theseus

## backtrace

```Rust
//核心函数
pub fn stack_trace(
    on_each_stack_frame: &mut dyn FnMut(StackFrame, &StackFrameIter) -> bool,
    max_recursion: Option<usize>,
)
```

- `on_each_stack_frame`:在每个stack上要invoke的闭包，闭包通过stack_frame和stack_frame_iter读取获取当前栈帧调用地址的符号偏移量
  - 如果有符号偏移量，打印包含符号名和偏移量的地址信息
  - 如果没有符号偏移量，打印调用地址和未知符号的地址信息

- 函数内部调用`invoke_with_current_registers`,在传入的闭包里迭代call stack 定位函数位置

# How to Unwind

- c++ abi
- eh_frame/eh_frame_hdr/gcc_except_table
- context restore
- _Uwind_Resume&&core::intrinsics::catch_unwind
