# TODO

- [x] 从debug console运行一个app的路径 
- [x] camkesthread是什么
- [x] camkes声明的进程间通讯具体作用
- [x] component怎么打log
- [x] rust中的项目架构和模块化，mod.rs lib.rs
- [ ] sel4 thread_control
- [x] sel4 IPC
- [ ] cantrip-root-server 源码
- [ ] cantrip-memory-manager 源码

# component organization

- `system.camkes` is the toplevel CAmkES Assembly, that defines which components exist in the entire system, and which components speak to which other components.
- 在代码中system.camkes为每一种架构一个，在system/platforms下
- /interfaces定义每个组件对外开放的接口

## sel4_sys

- syscall包装

arch/syscall_commmon:定义系统调用接口，具体实现有不同架构的.rs文件实现

> std_connector.camkes is a built-in file that defines the standard CAmkES connector types

# syscall invoke 路径

- cantrip_proc_ctrl_start(bundle_id: &str)
- cantrip_proc_ctrl_request<T: DeserializeOwned>(request: &ProcessControlRequest,)
- rpc_basic_send
- crate::camkes::rpc_basic::send($inf_endpoint, \$request_len)
- pub unsafe fn send(endpoint: seL4_CPtr, request_len: usize) -> (usize, usize)
- pub unsafe fn seL4_Call(dest: seL4_CPtr, msgInfo: seL4_MessageInfo) -> seL4_MessageInfo
- 

# Process Manager代码组织

├── Cargo.toml
├── ProcessManager.camkes
├── cantrip-proc-component
│   ├── Cargo.toml
│   ├── build.rs
│   └── src
│       └── run.rs
├── cantrip-proc-interface
│   ├── Cargo.toml
│   └── src
│       ├── bundle_image.rs
│       └── lib.rs
├── cantrip-proc-manager
│   ├── Cargo.toml
│   ├── build.rs
│   └── src
│       ├── lib.rs
│       ├── proc_manager
│       │   └── mod.rs
│       └── sel4bundle
│           ├── arch
│           │   ├── aarch64.rs
│           │   ├── arm.rs
│           │   ├── riscv.rs
│           │   └── riscv32.rs
│           ├── feature
│           │   ├── mcs.rs
│           │   ├── no_mcs.rs
│           │   ├── no_smp.rs
│           │   ├── no_spill_tcb_args.rs
│           │   ├── smp.rs
│           │   └── spill_tcb_args.rs
│           └── mod.rs

### proc-manager-component/run.rs

一个control，两个interface，均实现了`CamkesThreadInterface`，也都实现了`CamkesThreadStart`用于启动线程

- `ProcessManagerControlThread` CamkesThread->control
- `PkgMgmtInterfaceThread` CamkesThread->interface
- `ProcCtrlInterfaceThread` CamkesThread->interface

cantrip_proc(),返回一个CANTRIP_PROC_MANAGER

### cantrip-proc-interface/lib.rs

#### 结构体定义

1. **`Bundle`**
   - 代表一个进程包，包含进程包 ID (`app_id`) 和进程包占用的内存大小 (`app_memory_size`)。

2. **`GetRunningBundlesResponse`**
   - 用于返回正在运行的进程包列表的响应结构体，包含一个 `bundle_ids` 字段，这是一个 `Vec<String>` 类型，存储进程包 ID 的集合。

3. **`GetBundleStateResponse`**
   - 用于返回特定进程包状态的响应结构体，包含一个 `bundle_state` 字段，类型为 `BundleState` 枚举，描述进程包的当前状态。

4. **`InstallResponse`**
   - 用于返回安装操作的结果，包含一个 `bundle_id` 字段，这是一个 `String` 类型，存储被安装进程包的 ID。

### 枚举定义

1. **`BundleState`**
   - 描述进程包可能的状态，包括 `Stopped`、`Running`、`Exited` 和 `Faulted`。

2. **`ProcessManagerError`**
   - 定义进程管理操作可能遇到的错误类型，包括 `Success`、`BundleIdInvalid`、`PackageBufferLenInvalid` 等。

3. **`ProcessControlRequest`**
   - 定义了可以发送给进程控制接口的请求类型，如 `Start`、`Stop`、`GetRunningBundles` 等。

4. **`PackageManagementRequest`**
   - 定义了可以发送给包管理接口的请求类型，如 `Install`、`InstallApp`、`Uninstall`。

### Trait定义

1. **`BundleImplInterface`**
   - 定义了进程包实现必须提供的接口，包括 `start`、`stop`、`suspend`、`resume` 和 `capscan` 方法。

2. **`ProcessManagerInterface`**
   - 定义了进程管理功能必须提供的接口，包括 `install`、`uninstall`、`start`、`stop` 和 `capscan` 方法。

3. **`PackageManagementInterface`**
   - 定义了包管理功能必须提供的接口，与 `ProcessManagerInterface` 类似，但专注于包的安装和卸载。

4. **`ProcessControlInterface`**
   - 定义了进程控制功能必须提供的接口，包括 `start`、`stop`、`get_running_bundles`、`get_bundle_state` 和 `capscan` 方法。

### cantrip-proc-manager/mod.rs

这段 Rust 代码定义了 Cantrip OS 的进程管理系统的核心功能，其中涉及多个结构体、枚举和特质（trait），用于处理包（应用/服务）的安装、启动、停止和状态查询等操作。以下是对代码中出现的主要组件的分析：

### 结构体

1. **`BundleData<T>`**:
   - 封装了单个进程包的数据，包括状态、包本身的信息以及可能的实现特定接口的实例。
   - 字段包括状态（`state`）、包信息（`bundle`），以及实现了 `BundleImplInterface` 的实例（`bundle_impl`）。

2. **`ProcessManager<P>`**:
   - 核心管理结构体，负责进程包的整体管理。使用泛型参数 `P` 来抽象处理进程管理的接口。
   - 包含一个实现了 `ProcessManagerInterface` 的管理器（`manager`）和一个包含所有进程包数据的哈希表（`bundles`）。
   - 该结构体本身实现了`ProcessControlInterface`和`PackageManagementInterface`
   - 这个结构体在lib.rs中有实例化，`CantripProcManager`
     - `Mutex<Option<ProcessManager<CantripManagerInterface>>>`

### cantrip-proc-manager/lib.rs

### 结构体

1. **`CantripProcManager`**:
   - 一个封装了 `ProcessManager` 的结构体，使用 `Mutex` 来同步对 `ProcessManager` 实例的访问。这个结构体提供了延迟初始化的能力，允许先创建一个空的 `CantripProcManager` 实例，然后在运行时完成初始化。

2. **`Guard`**:
   - 一个辅助结构体，用于安全地访问被 `Mutex` 保护的 `ProcessManager` 实例。提供了操作进程管理功能的方法，如安装、启动、停止包，以及查询包状态等。

3. **`CantripManagerInterface`**:被`ProcessManager`封装
   - cantrip_proc()函数调用返回的引用对象，
   - 实现了 `ProcessManagerInterface` 特质，为 `ProcessManager` 提供具体的进程管理操作实现。这包括与安全协调器（SecurityCoordinator）交互来执行包的安装、卸载、启动和停止。

### Trait

1. **`PackageManagementInterface` 和 `ProcessControlInterface` 实现**:
   - 通过 `Guard` 结构体实现了包管理和进程控制接口，使得 `CantripProcManager` 可以代理包管理和进程控制的请求。
   - guard对最内层manager进行一次封装
   
2. **`ProcessManagerInterface`**:
   - 由`CantripManagerInterface`实现，定义了进程管理操作的底层逻辑，包括与安全协调器的交互。
   - 其中的start函数加载初始化并运行bundle，是由rpc_recv触发的

### 功能和流程

- **包管理（Package Management）**:
  - 提供了安装（`install`）、安装应用（`install_app`）、卸载（`uninstall`）等功能。这些操作主要涉及与安全协调器交互，以确保包的安全性。

- **进程控制（Process Control）**:
  - 提供了启动（`start`）、停止（`stop`）、获取正在运行的包（`get_running_bundles`）以及查询包状态（`get_bundle_state`）的功能。这些操作需要通过 `ProcessManager` 来执行，`Guard` 结构体提供了安全的访问方式。

- **安全和隔离**:
  - 通过与安全协调器的交互，`CantripManagerInterface` 实现了包内容的加载、应用的安装和卸载操作，同时确保了操作的安全性和隔离性。

- **延迟初始化**:
  - `CantripProcManager` 的设计允许在不可变的上下文中创建实例（使用 `const fn`），然后在运行时完成实际的初始化。这对于在系统启动时设置全局管理器非常有用。



# sel4_recv

- macro rules 实现rpc_basic_recv调用recv_loop，死循环接收sel4_call发送的消息

# console命令调用路径

## console的thread

- struct DebugConsoleControlThread

这个结构体实现了`CamkesThreadInterface`特征，在run函数中有

- DebugConsole/cantrip-debug-console/src/run.rs

 ```rust
 // Entry point for DebugConsole. Optionally runs an autostart script
 // after which it runs an interactive shell with UART IO.
     fn run() {
         #[cfg(feature = "autostart_support")]
         run_autostart_shell();
 
         #[cfg(feature = "interactive_shell")]
         run_sparrow_shell();
    }
 // Run any "autostart.repl" file in the eFLASH through the shell with output
 // sent either to the console or /dev/null depending on the feature selection.
 #[cfg(feature = "autostart_support")]
 fn run_autostart_shell() {
     // Rx data comes from the embedded script
     // Tx data goes to either the uart or /dev/null
     // XXX test if autostart.repl is present
     let mut rx = cantrip_io::BufReader::new("source -q autostart.repl\n".as_bytes());
     cantrip_shell::repl_eof(&mut get_tx(), &mut rx);//
 }
 ```

## thread启动过程

### camkes.rs

out/cantrip/rpis/debug/process_manager/camkes.rs中使用

```rust
static_interface_thread!(
    /*name=*/ proc_ctrl,
    /*tcb=*/ SELF_TCB_PROC_CTRL,
    /*ipc_buffer=*/ core::ptr::addr_of!(_camkes_ipc_buffer_process_manager_proc_ctrl_0000.data[4096]),
    &CAMKES
);
```

创建一个thread，macro的声明在startup.rs中，对如何使用也有具体解释

创建thread后使用::start()启动：

```rust
//cantrip-os-common/src/camkes/src/arch/riscv32.rs
core::arch::global_asm!(
    "
    .section .text._camkes_start
    .align 2
    .globl _camkes_start
    .type _camkes_start, @function
_camkes_start:
    .option push
    .option norelax
    la gp, __global_pointer$
    .option pop

    addi sp,sp,-4
    sw a0, 0(sp)
    jal _camkes_start_rust//在汇编中直接找到函数地址并跳转
"
);

//camkes.rs
pub fn _camkes_start_rust(thread_id: sel4_sys::seL4_CPtr) -> ! {
    let thread = CAMKES.thread(thread_id).unwrap();
    match thread_id {
        SELF_TCB_CONTROL => crate::ProcessManagerControlThread::start(thread),
        SELF_TCB_PROC_CTRL => crate::ProcCtrlInterfaceThread::start(thread),
        SELF_TCB_PKG_MGMT => crate::PkgMgmtInterfaceThread::start(thread)
        _ => unreachable!(),
    }
    // ?unreachable!();
}
//startup.rs 为所有实现了CamkesThreadInterface特征的结构体加一个start
pub trait CamkesThreadStart {//
    fn start(thread: &CamkesThread) -> !;//发散函数
}
impl<T: CamkesThreadInterface> CamkesThreadStart for T {
    fn start(thread: &CamkesThread) -> ! {
        match thread {
            ... ...
            CamkesThread::Interface(_, thread_name, .., &ref camkes) => {
                camkes.pre_init.wait(); // Wait for Control::pre_init to complete.
                unsafe {thread.init_tls();thread.init_ipc_buffer();}
                T::init();//初始化
                camkes.interface_init.post(); camkes.post_init.wait();
                T::run();//调用run函数开始运行
                log::trace!(target: camkes.name, "{}::run returned", thread_name);
                camkes.pre_init.wait(); // blocking
                unreachable!();
            }
           ... ...
        }
    }
}
```





## 以start为例

> 在syscall之前的调用链虽然在proc-manager文件中，但其实是属于debug-console相关thread在运行的代码，在syscall后recv端是`ProcCtrlInterfaceThread`的代码，负责加载运行一个新的app。console和皮肉从manager间的通讯以一个endpoint连接，在camkes中被声明为一个connection。

- DebugConsole/cantrip-shell/src/lib.rs

```rust
/// Stripped down repl for running automation scripts. Like repl but prints
/// each cmd line and stops at EOF/error.
pub fn repl_eof<T: io::BufRead>(output: &mut dyn io::Write, input: &mut T) {
    let cmds = get_cmds();//get_cmds得到所有支持的命令和对应的调用函数
    let mut line_reader = LineReader::new();
    loop {
        // NB: LineReader echo's input
        let _ = write!(output, "CANTRIP> ");
        if let Ok(cmdline) = line_reader.read_line(output, input) {
            eval(cmdline, &cmds, output, input);//调用eval
        } else {
            let _ = writeln!(output, "EOF");
            break;
        }
    }
}

pub fn eval<T: io::BufRead>(
    cmdline: &str,
    cmds: &HashMap<&str, CmdFn>,
    output: &mut dyn io::Write,
    input: &mut T,
) {
    let mut args = cmdline.split_ascii_whitespace();
    match args.next() {
        ... ...
        Some(cmd) => {
            let result = cmds.get(cmd).map_or_else(
                || Err(CommandError::UnknownCommand),
                |func| func(&mut args, input, output),//使用闭包调用对应命令的执行函数
            );
        ... ...
        }
    ... ...
    }
}

fn start_command(... ...) -> Result<(), CommandError> {
    let bundle_id = args.next().ok_or(CommandError::BadArgs)?;
    match cantrip_proc_ctrl_start(bundle_id) {//调用启动函数，给一个bundle_id(APP名)
        Ok(_) => {
            writeln!(output, "Bundle \"{}\" started.", bundle_id)?;
        }
   ... ...
}
```

- ProcessManager/cantrip-proc-interface/src/lib.rs

```rust
pub fn cantrip_proc_ctrl_start(bundle_id: &str) -> Result<(), ProcessManagerError> {
    cantrip_proc_ctrl_request(&ProcessControlRequest::Start(bundle_id))
}

fn cantrip_proc_ctrl_request<T: DeserializeOwned>(//所有和bundle相关的,查询，运行，停止都要经过这里
    request: &ProcessControlRequest,
) -> Result<T, ProcessManagerError> {
    trace!("cantrip_proc_ctrl_request {:?}", &request);
 	... ...
    Camkes::clear_request_cap();
    match rpc_basic_send!(proc_ctrl, request_slice.len()).0.into() {//proc_ctrl是camkes声明的
        ... ...
}
```

- cantrip-os-common/src/camkes/src/rpc_basic.rs

```rust
macro_rules! rpc_basic_send {
    ... ...
    (@end $inf_endpoint:ident, $request_len:expr) => {
        unsafe {
            extern "C" {
                static $inf_endpoint: sel4_sys::seL4_CPtr;
            }
            crate::camkes::rpc_basic::send($inf_endpoint, 			   $request_len)//inf_tag_INTERFACE_ENDPOINT,
        }
    };
}

/// Sends an RPC message and blocks waiting for a reply. The message contents are assumed to be marshalled in the storage returned by get_buffer_mut(). Data in the msg buffer are assumed partitioned into <request><reply> slices with the <request> slice starting at offset 0 into the slice/buffer.Note the msg buffer is per-thread (TLS) but global and no synchronization is done to guard against re-use.
pub unsafe fn send(endpoint: seL4_CPtr, request_len: usize) -> (usize, usize) {
    const WORD_SIZE: usize = size_of::<seL4_Word>();
    let info = seL4_Call(
        endpoint,
        seL4_MessageInfo::new(
            /*label=*/ 0,
            /*capsUnwrapped=*/ 0,
            /*extraCaps=*/ 0,
            /*length=*/ (request_len + WORD_SIZE - 1) / WORD_SIZE,
        ),
    );
    (info.get_label(), info.get_length())
}
```

- cantrip-os-common/src/sel4-sys/arch/syscall_common.rs

```rust
#[inline(always)]//	和sel4的syscall stub基本一致
pub unsafe fn seL4_Call(dest: seL4_CPtr, msgInfo: seL4_MessageInfo) -> seL4_MessageInfo {
    let mut info: seL4_Word;
    let mut msg0 = seL4_GetMR(0);
    let mut msg1 = seL4_GetMR(1);
    let mut msg2 = seL4_GetMR(2);
    let mut msg3 = seL4_GetMR(3);

    asm_send_recv!(SyscallId::Call, dest => _, msgInfo.words[0] => info, msg0, msg1, msg2, msg3);//自定义宏，具体如何调用取决于架构

    seL4_SetMR(0, msg0);
    seL4_SetMR(1, msg1);
    seL4_SetMR(2, msg2);
    seL4_SetMR(3, msg3);
    seL4_MessageInfo { words: [info] }
}
```

- cantrip-os-common/src/sel4-sys/arch/aarch64_no_mcs.rs

```rust
// Does a send operation (with message registers) followed by a receive that returns the sender's badge plus all message registers. Used for directed send+receive where data flows in both directions, like seL4_Call.
#[macro_export]
macro_rules! asm_send_recv {
... ...
    // NB: for seL4_Call*，忙等
    ($syscall:expr, $src:expr => _, $info:expr => $info_recv:expr, $mr0:expr, $mr1:expr, $mr2:expr, $mr3:expr) => {
        asm!("svc 0",
            in("x7") swinum!($syscall),
            inout("x0") $src => _,
            inout("x1") $info => $info_recv,
            inout("x2") $mr0,
            inout("x3") $mr1,
            inout("x4") $mr2,
            inout("x5") $mr3,
        )
    };
}
```

下面跳转到sel4的内核态进行处理，这里进行了一个seL4_Call的syscall，请求运行一个新的app

------

下面是recv方的调用路径，

- ProcessManager/cantrip-proc-component/src/run.rs
- `ProcCtrlInterfaceThread`同样也有camkesthreadinterface特征，在run被执行后忙式等待，得到请求后进行相应处理，处理结束继续等待，所以这个thread只干处理console命令这一件事

```rust
struct ProcCtrlInterfaceThread;
impl CamkesThreadInterface for ProcCtrlInterfaceThread {
    fn run() {
        rpc_basic_recv!(proc_ctrl, PROC_CTRL_REQUEST_DATA_SIZE, ProcessManagerError::Success);
    }
}
```

- cantrip-os-common/src/camkes/src/rpc_basic.rs

```rust
/// Server-side implementation for basic RPC without any capability
/// passing. This macro is normally invoked in a CamkesInterfaceThread's
/// run method to implement the server side of a cantripRPCCall or
/// cantripRPCCallSignal connection. The thread must implement the
/// Dispatch callback to handle deserialization of marshalled parameters
/// and processing of the request(s).
#[macro_export]
macro_rules! rpc_basic_recv {
... ...
    (@end $inf_dispatch:expr, $inf_endpoint:ident, $inf_reply:ident, $inf_request_size:expr, $inf_success:expr) => {
        unsafe {
            crate::camkes::rpc_basic::recv_loop(
                $inf_dispatch,
                $inf_endpoint,
                $inf_reply,
                $inf_request_size,
                $inf_success,
            )
        }
    };
}

/// Server-side implementation for basic RPC without any capability
/// passing. This is normally invoked by the |rpc_basic_recv| macro.
pub unsafe fn recv_loop<E>(... ...) -> !//无尽的循环
where
    usize: From<E>,
{
    let success_word: seL4_Word = success.into();

    let mut client_badge: seL4_Word = 0;
    seL4_Recv(//忙等
        /*src=*/ endpoint,
        /*sender=*/ &mut client_badge as _,
        /*reply=*/ reply,
    );
    loop {
        ... ...
        let response = dispatch(client_badge, request_slice, reply_slice);//调用处理函数
        //处理命令后给出答复
        seL4_ReplyRecv(//reply后忙等
            /*src=*/ endpoint,
            /*msgInfo=*/
            seL4_MessageInfo::new(
                /*label=*/ label, /*capsUnwrapped=*/ 0, /*extraCaps=*/ 0,
                /*length=*/ length,
            ),
            /*sender=*/ &mut client_badge as _,
            /*reply=*/ reply,
        );
    }
}
```

- ProcessManager/cantrip-proc-component/src/run.rs

- call方把要运行的bundle_id封装进`ProcessControlRequest`枚举中，这里再把它取出来，解构得到bundle_id,也就是app名

- cantrip_proc()可以拿到一个CantripProcManager类

  > CantripProcManager bundles an instance of the ProcessManager that operates on CantripOS interfaces and synchronizes public use with a Mutex. There is a two-step dance to setup an instance because we want CANTRIP_PROC static and ProcessManager is incapable of supplying a const fn due it's use of hashbrown::HashMap.

- start函数是`ProcessControlInterface`这一特征的

```rust
fn dispatch(
        _client_badge: usize,
        request_buffer: &[u8],
        reply_buffer: &mut [u8],
    ) -> ProcCtrlResult {
        let request = match postcard::from_bytes::<ProcessControlRequest>(request_buffer) {
            Ok(request) => request,
            Err(_) => return Err(ProcessManagerError::DeserializeError),
        };
        match request {//根据不同的request类型进行操作
            ProcessControlRequest::Start(bundle_id) => Self::start_request(bundle_id),
            ... ...
        }
    }

fn start_request(bundle_id: &str) -> ProcCtrlResult {
        // TODO(283265795): copy bundle_id from the IPCBuffer
        cantrip_proc().start(&String::from(bundle_id)).map(|_| 0)
    }
```

- ProcessManager/cantrip-proc-manager/src/proc_manager/mod.rs

```rust
fn start(&mut self, bundle_id: &str) -> Result<(), ProcessManagerError> {
        trace!("start bundle_id {}", bundle_id);
        match self.bundles.get_mut(&bid) {
			... ...
            None => {
                trace!("builtin auto-install");
                let mut bundle = BundleData::new(&Bundle::new(&bid));
                bundle.bundle_impl = Some(self.manager.start(&bundle.bundle)?);//调用启动函数
                bundle.state = BundleState::Running;
			... ...
    }
```

- ProcessManager/cantrip-proc-manager/src/lib.rs

```rust
fn start(&mut self, bundle: &Bundle) -> Result<Self::BundleImpl, ProcessManagerError> {
        trace!("ProcessManagerInterface::start {:?}", bundle);

        // Design doc says:
        // 1. Ask security core for application footprint with SizeBuffer
        // 2. Ask security core for manifest (maybe piggyback on SizeBuffer)
        //    and parse for necessary info (e.g. whether kv Storage is
        //    required, other privileges/capabilities)
        // 3. Ask MemoryManager for shared memory pages for the application
        //    (model handled separately by MlCoordinator since we do not know
        //    which model will be used)
        // 4. Allocate other seL4 resources:
        //     - VSpace, TCB & necessary capabiltiies
        // 5. Ask security core to VerifyAndLoad app into shared memory pages
        // 6. Complete seL4 setup:
        //     - Setup application system context and start thread
        //     - Badge seL4 recv cap w/ bundle_id for (optional) StorageManager
        //       access
        // What we do atm is:
        // 1. Ask SecurityCoordinator to return the application contents to load.
        //    Data are delivered as a read-only ObjDescBundle ready to copy into
        //    the VSpace.
        // 2. Do 4+6 with BundleImplInterface::start.

        // TODO(sleffler): awkward container_slot ownership
        let mut container_slot = CSpaceSlot::new();
        let bundle_frames = cantrip_security_load_application(&bundle.app_id, &container_slot)?;
        let mut sel4_bundle = seL4BundleImpl::new(bundle, &bundle_frames)?;
        // sel4_bundle owns container_slot now; release our ref so it's not
        // reclaimed when container_slot goes out of scope.
        container_slot.release();

        sel4_bundle.start()?;

        Ok(sel4_bundle)
    }
```

