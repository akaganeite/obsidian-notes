# 问题

- domain是什么？由什么组成？maybe一个crate，一段代码，有一些接口，被proxy包装。
- domain和thread的关系。thread运行在domain上
- domain unwind的时候如何进行资源回收->依赖Drop的实现
- domain崩溃，怎么终止所有thread



入口：trusted_entry

# unwind相关代码

```assembly
    .text 
    .align  16              
__unwind:
    movq 16(%rdi), %rcx;rdi存continuation的地址，这里是恢复现场
    movq 24(%rdi), %rdx
    movq 32(%rdi), %rsi

    movq 48(%rdi), %r8
    movq 56(%rdi), %r9
    movq 64(%rdi), %r10


    movq 136(%rdi), %rsp
    movq 128(%rdi), %rbp
    movq 120(%rdi), %rbx
    movq 112(%rdi), %r11
    movq 104(%rdi), %r12
    movq 96(%rdi), %r13
    movq 88(%rdi), %r14
    movq 80(%rdi), %r15
    pushq 72(%rdi)
    popfq

    movq (%rdi), %rax
    movq %rax, -8(%rsp)
    movq 8(%rdi), %rax

    movq 40(%rdi), %rdi

    jmpq *-8(%rsp) ",
    options(att_syntax)
```



# IDL代码

## domain_entry

这个自动生成的 crate 是用来创建特定域（domain）的入口点（entry point）。在 Rust 项目中，crate 是一个包或库，包含了相关的代码、依赖和构建信息。这个入口点 crate 主要有以下几个作用：

### 主要用途

1. **初始化域组件**：
   - 在 `main.rs` 文件中定义的 `trusted_entry` 函数会初始化特定域所需的组件。这些组件包括 `syscalls`、`heap`、`mmap` 等，具体取决于域的需求。

2. **提供域的主要功能入口**：
   - `trusted_entry` 函数作为域的主要入口点，调用域的主函数 `main`。这个主函数包含域的核心逻辑和操作。

3. **管理和组织域的依赖**：
   - 通过生成 `Cargo.toml` 文件，自动配置并管理该域的依赖项。包括与其他模块或库的依赖关系，如 `interface`、`libsyscalls`、`syscalls`、`console` 等。

### 工作流程

1. **定义域组件**：
   - `DomainCreateComponent` 枚举定义了域的不同组件，如 `Domain`、`Heap`、`MMap`。这些组件在 `trusted_entry` 函数中被初始化，并作为参数传递给域的主函数。

2. **生成和写入文件**：
   - `generate_entrypoint_cargo` 函数生成 `Cargo.toml` 文件内容，描述 crate 的元数据和依赖。
   - `generate_entrypoint_main_rs` 函数生成 `main.rs` 文件内容，定义入口点函数 `trusted_entry`。
   - `write_entrypoint_crate` 函数将生成的内容写入到指定路径，创建相应的文件夹和文件。

3. **调用域主函数**：
   - 在 `main.rs` 文件中，`trusted_entry` 函数会初始化所有需要的组件，然后调用域的主函数 `main`。这个主函数实现了域的核心逻辑。

### 具体示例

假设你有一个域 `example`，那么生成的 crate 会：

1. 创建一个新的目录 `example_entry_point`，包含 `Cargo.toml` 和 `src/main.rs` 文件。
2. `Cargo.toml` 文件会配置该 crate 的元数据和依赖项。
3. `main.rs` 文件会定义 `trusted_entry` 函数，初始化所有需要的域组件，并调用 `example::main` 函数。

### 总结

这个自动生成的 crate 用于为特定域创建一个入口点。它初始化域所需的组件，管理域的依赖，并调用域的主函数，实现域的核心逻辑。这个过程简化了域的创建和管理，使得开发者可以专注于实现域的具体功能，而无需手动配置和初始化相关组件。

## proxy 生成

### 调用图

```
generate_interface_proxy
│
├── generate_proxy_impl
│   ├── generate_proxy_impl_one (多次调用)
│   └── generate_proxy_impl_one
│
├── generate_trampolines
│   └── generate_trampolines (内部处理方法)
│
├── generate_proxy_impl_one
├── generate_trampolines
│
generate_proxy
├── generate_proxy_impl
│   ├── generate_proxy_impl_one (多次调用)
│   └── generate_proxy_impl_one
└── generate_proxy_impl_one (多次调用)
```

其中`generate_trampolines`这个函数为每个方法生成 trampoline 实现。trampoline 的作用是为每个代理方法提供一种安全的回退机制，当调用过程中发生 panic 时，可以通过 trampoline 机制进行处理。

### 如何对一个函数进行代理

`generate_proxy_impl_one` 函数是用于为每个方法生成代理实现的关键函数。它处理每个 trait 方法，生成相应的代理方法实现。以下是对该函数的详细解释：

```rust
fn generate_proxy_impl_one(
    trait_ident: &Ident,
    method: &TraitItemMethod,
    cleaned_method: &TraitItemMethod,
```

### 参数解析

- `trait_ident`：引用传递的 trait 的标识符。
- `method`：当前处理的 trait 方法。
- `cleaned_method`：已经去除 `&self` 和 `&mut self` 参数的 trait 方法。

### 生成的代理方法示例

假设有一个 trait 方法 `foo`：

```rust
trait MyTrait {
    fn foo(&self, x: i32) -> i32;
}
```

生成的代理方法实现如下：

```rust
fn foo(&self, x: i32) -> i32 {
    #[cfg(not(feature = "trampoline"))]
    let r = self.domain.foo(x);
    #[cfg(feature = "trampoline")]
    let r = unsafe { MyTrait_foo_tramp(&self.domain, x) };

    #[cfg(feature = "trampoline")]
    unsafe {
        ::libsyscalls::syscalls::sys_discard_cont();
    }

    r
}
```

## PROXY生成后的实际调用流程

one_arg详见idl repo中的test1.rs

### `one_arg` 函数的详细调用流程

#### 正常调用流程

1. **调用 `one_arg_tramp`**
   - 由代理方法调用 `one_arg_tramp`，实际执行的是 trampoline 函数。
   - 保存寄存器的值在continuation stack上，
   
2. **调用 `one_arg`**
   - 由 `one_arg_tramp` 调用 `one_arg`，执行实际的逻辑。

3. **返回结果**
   - 如果 `one_arg` 正常执行，返回结果给 `one_arg_tramp`，然后返回给调用者。

#### 发生 panic 时的处理流程

1. **panic 发生**
   - 调用core::panic,调用panic_impl,也就是自定义的panic_handler函数
   - 如果 `one_arg` 发生 panic，panic_handler调用unwind函数，恢复continuation stack的信息
2. 调用 `one_arg_err`，执行错误处理逻辑。
   - 打印错误信息，返回`Err(unsafe{crate::rpc::RpcError::panic()})`

# thread

- 在不同domain运行，使用migrating thread。这好像就是用continuation实现的吧，保存在domain入口的全部寄存器，函数正常或异常返回都会用到continuation。thread自带存储单位可以装下很多个cont

- 在进入另一个domain的时候proxy会调用func_tramp函数，保存寄存器值，调用push_continuation,压栈，然后`jmp func`调用函数。函数运行时panic后转到core panic，再转到panic_handler

- handler调用unwind函数，使用pop_continuation出栈，现在thread恢复到进入domain之前的状态了，之后调用`func_err`函数, 返回调用者一个错误

  > 这里很有意思，我thread运行到一半panic，之后返回进入domain之前的状态。论文中说销毁thread那其实就是把所有在当前domain的thread的状态回退到进入前那一刻的状态，并且返回一个错误。thread回退到调用domain，携带一个错误。

- unwind策略：在panichandler中手动设定我要调用unwind函数，然后会转到错误处理函数，其实就是向调用者返回一个rpcerror。内存释放工作还是由rust自己完成，不像theseus。这里只是手动设定了一个panichandler。在裸机环境中

# shadow device driver

```rust
struct ShadowInternal{
    create: Arc<dyn Create_device>,
    dom: Option<Box<dyn syscalls::Domain>>,
    device: Box<dyn device>，
}

struct Shadow{
    shadow: Mutex<ShadowInternal>,
}
```







# BDEV

```Rust
impl BDev for BDevProxy {
        fn read(
            &self,
            block: u32,
            data: crate::rref::rref::RRef<[u8; 4096]>,
        ) -> crate::rpc::RpcResult<crate::rref::rref::RRef<[u8; 4096]>> {
            #[cfg(not(feature = "trampoline"))]
            let r = self
                .domain
                .read(block: u32, data: crate::rref::rref::RRef<[u8; 4096]>);
            #[cfg(feature = "trampoline")]
            let r = unsafe {
                BDev_read_tramp(
                    &self.domain,
                    block: u32,
                    data: crate::rref::rref::RRef<[u8; 4096]>,
                )
            };
            #[cfg(feature = "trampoline")]
            unsafe {	`
                ::libsyscalls::syscalls::sys_discard_cont();
            }
            r
        }
    
 extern "C" fn BDev_read(
        redidl_generated_domain_bdev: &alloc::boxed::Box<dyn BDev>,
        block: u32,
        data: crate::rref::rref::RRef<[u8; 4096]>,
    ) -> crate::rpc::RpcResult<crate::rref::rref::RRef<[u8; 4096]>> {
        (&**redidl_generated_domain_bdev)
            .read(block: u32, data: crate::rref::rref::RRef<[u8; 4096]>)
    }
    #[cfg(feature = "trampoline")]
    #[cfg(feature = "proxy")]
    #[no_mangle]
    extern "C" fn BDev_read_err(
        redidl_generated_domain_bdev: &alloc::boxed::Box<dyn BDev>,
        block: u32,
        data: crate::rref::rref::RRef<[u8; 4096]>,
    ) -> crate::rpc::RpcResult<crate::rref::rref::RRef<[u8; 4096]>> {
        #[cfg(feature = "proxy-log-error")]
        ::console::println!("proxy: {} aborted", stringify!(read));
        Err(unsafe { crate::rpc::::panic() })
    }
    #[cfg(feature = "trampoline")]
    #[cfg(feature = "proxy")]
    #[no_mangle]
    extern "C" fn BDev_read_addr() -> u64 {
        BDev_read_err as u64
    }
    #[cfg(feature = "proxy")]
    #[cfg(feature = "trampoline")]
    extern "C" {
        fn BDev_read_tramp(
            redidl_generated_domain_bdev: &alloc::boxed::Box<dyn BDev>,
            block: u32,
            data: crate::rref::rref::RRef<[u8; 4096]>,
        ) -> crate::rpc::RpcResult<crate::rref::rref::RRef<[u8; 4096]>>;
    }
    
    
    
    
fn write(
            &self,
            block: u32,
            data: &crate::rref::rref::RRef<[u8; 4096]>,
        ) -> crate::rpc::RpcResult<()> {
            #[cfg(not(feature = "trampoline"))]
            let r = self
                .domain
                .write(block: u32, data: &crate::rref::rref::RRef<[u8; 4096]>);
            #[cfg(feature = "trampoline")]
            let r = unsafe {
                BDev_write_tramp(
                    &self.domain,
                    block: u32,
                    data: &crate::rref::rref::RRef<[u8; 4096]>,
                )
            };
            #[cfg(feature = "trampoline")]
            unsafe {
                ::libsyscalls::syscalls:: ();
            }
            r
        }
    }
```



