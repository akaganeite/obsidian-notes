# Rust Panic

## links

### panic

[原来Rust的panic也能被捕捉？浅谈Rust的panic机制 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/687092686)

[rust 如何处理【程序崩溃 panic】 - Rust语言中文社区 (rustcc.cn)](https://rustcc.cn/article?id=04fbf832-0395-49dc-ab60-ef4496a34060)

[Rust竟然没有异常处理？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/183193931)

### 错误处理

[Rust错误处理最佳实践(1) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/137813420)

[细说Rust错误处理 - Rust语言中文社区 (rustcc.cn)](https://rustcc.cn/article?id=75dbd87c-df1c-4000-a243-46afc8513074)

[Rust竟然没有异常处理？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/183193931)

# Panic路径

## core_lib/panicking.rs

- panic函数，作为调用`panic!`宏后直接调用的第一级函数
- panic_fmt，构造panic信息；调用panic_impl函数，转到std_lib中的panic handler或是用户自定义的handler

## std_lib/panicking.rs

- pub fn begin_panic_handler(info: &PanicInfo<'_>) -> !：std_lib中的panic入口点
- rust_panic_with_hook：处理panic的核心函数，进行检查后将panic分发给运行时
  - 检查是否多层panic，是否直接abort
  - 调用hook函数
  - 根据can_unwind判断是否unwind
- fn rust_panic(msg: &**mut** dyn PanicPayload) -> !
  - let code = unsafe { __rust_start_panic(msg) }; 

## lib:panic_unwind

### lib.rs

- 分发panic

  ```rust
  pub unsafe fn __rust_start_panic(payload: &mut dyn PanicPayload) -> u32 {
      let payload = Box::from_raw(payload.take_box());
      imp::panic(payload)
  }
  ```

### gcc.rs

- Theseus的panic实现，参考了libgcc unwind
- panic函数调用`return uw::_Unwind_RaiseException(exception_param) as u32;`
- 转到c代码执行

# gcclibunwind:C++异常处理和stack unwinding

[c++ 异常处理（上）-睿初科技软件开发技术博客 (brionas.github.io)](https://brionas.github.io/2014/05/13/C++-Exception-Handle-1/)

[c++ 异常处理（下）-睿初科技软件开发技术博客 (brionas.github.io)](https://brionas.github.io/2014/05/13/C++-Exception-Handle-2/)

- unwind的过程可以简单看成是函数调用的逆过程，这个过程在实现上由一个专门的stack unwind库来进行，在intel平台上，它属于Itanium ABI接口中的一部分，且与具体的语言无关，由系统提供实现





```rust
let ret_code = panic::catch_unwind(move || panic::catch_unwind(main).unwrap_or(101) as isize)
    .map_err(move |e| {
        mem::forget(e);
        rtabort!("drop of the panic payload panicked");
    });
```





# 关于Rust panic drop的链接

[How panic! calls drop functions - The Rust Programming Language Forum (rust-lang.org)](https://users.rust-lang.org/t/how-panic-calls-drop-functions/53663/13)

[How Drop clear memory - help - The Rust Programming Language Forum (rust-lang.org)](https://users.rust-lang.org/t/how-drop-clear-memory/26388/5)

[How does Rust know whether to run the destructor during stack unwind? - Stack Overflow](https://stackoverflow.com/questions/39750841/how-does-rust-know-whether-to-run-the-destructor-during-stack-unwind)
