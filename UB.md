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

## Panic路径

- pub fn begin_panic_handler(info: &PanicInfo<'_>) -> !
- rust_panic_with_hook
  - 检查是否多层panic，是否直接abort
  - 调用hook函数
  - 根据can_unwind判断是否unwind
- fn rust_panic(msg: &**mut** dyn PanicPayload) -> !
  - let code = unsafe { __rust_start_panic(msg) }; 



