# endpoint_prototype

```rust
pub struct ChannelWaker(Arc<CapsuleNode>);//按照rust要求实现的waker

pub(crate) struct CapsuleNode{//相当于future中的task，拥有fut和调度队列
    pub fut: Mutex<BoxedFut>,
    pub tx: Sender<Arc<Self>>
}
type BoxedFut = Pin<Box<dyn Future<Output = ()> + 'static>>;

pub struct CapsuleExecutor{//相当于async executor
    tx: Sender<Arc<CapsuleNode>>,
    ready_queue: Receiver<Arc<CapsuleNode>>
}

lazy_static! {//executor全局静态一个
    static ref KERNEL_EXECUTOR: CapsuleExecutor = CapsuleExecutor::new();
}

pub trait IntoCapsule{}//ep需要实现的trait，在send时调用add_to_job_queue，将创建的fut包装到node中加入调度队列。resolve用来call回调函数

pub struct CapsuleHandle<R: Send>{
    return_data: Receiver<R> 
}//支持block send与返回值传递
```

流程：

- A创建一个endpoint，创建时给出回调函数callback
- B对该ep调用send函数，传递ipcbuffer
- send:
  - 创建fut对象，包含async的代码块执行resolve函数。resolve函数则转去执行callback(ipcbuffer)
  - 调用executor.atomic_push->创建capsulenode，包裹fut，并在channel的输入端加入新node
- executor recv到新的node，为其创建waker，执行poll函数，实际执行callback(ipcbuffer)
- 效果：B在拥有一个A创建ep的情况下调用了A的callback函数，传递了自己的参数。

block send:

- B对ep调用send函数

  - `IntoCapsule::add_to_job_queue(self)->atomic_push`：创建一个传递返回值的内部channel，创建一个新的`async block` 在block内部await `resolve async`,拿到结果后送入channel的sender端。将新的async block装进capsulenode中，发送到`executor_queue`中。返回一个`CapsuleHandle`,包装channel的recv端。

  - 上面的函数的返回值被用于`block_on`的参数。

    ```rust
    pub fn block_on<R: Send>(handle: CapsuleHandle<R>) -> R{
        handle.return_data.recv()//等待，直到接收端有值，也就是之前的async task完成了
    }
    ```



# modifications

- 假设只有一个线程，如果第一次exec无法完成task，调度器将重复执行exec函数，直到该线程就绪为止

## executor

- exec：只对当前queue的每个task进行一次poll
- nb_recv:不阻塞线程，直接返回task or None

## condvar

- `block_wait`:thread设置为block,返回task_cx_ptr,返回后立刻schedule
- `signal`:将condvar绑定的thread设置为ready,不直接schedule

## 流程

sys_gettid->ep.send(block)->schedule->exec->callback->signal(wake)->continue the thread->get ret value

## todo

- safeos中实现condvar
- channel--crossbeamqueue
- callback async
- nb_send-->.await

## 问题

- exec函数何时执行
- 没有使用waker
- send时不能在resolve里面await



# 简介

**endpoint**：为内核组件提供crate间通信。**endpoint**：负责不同crate间的通信。

endpoint的主要功能是提供一个高效、安全的通信机制，用于不同crate间的数据传输。系统架构设计充分利用Rust的异步编程功能，

所有权和生命周期管理特性，确保通信过程中的内存安全和数据一致性。

# 架构设计

## 子系统

- executor：负责管理的和调度任务的执行
- endpoint：负责定义和管理不同的通信端点，通过回调函数处理传入的请求。
- channel：一种基于通道（Channel）机制的线程安全通信方法，用于在不同线程或内核组件之间传递消息。该模块通过无锁的并发队列（SegQueue）和条件变量（Condvar）实现了发送和接收操作。

# 模块详细设计

## Executor模块

Executor模块的主要功能是管理任务队列，并调度任务的执行。其核心结构包括：

- `CapsuleExecutor`：执行器的核心结构，包含任务发送器和就绪队列。
- `CapsuleNode`：表示一个任务节点，包含任务的未来对象和任务的发送器。
- `CapsuleHandle`：用于管理任务的返回数据。

接口设计

- CapsuleExecutor接口

  - `new()`：创建一个新的CapsuleExecutor实例。

  - `atomic_push()`：将任务推送到执行队列中。

- CapsuleNode接口
  - `new(fut: BoxedFut, tx: &Sender<Arc<Self>>)`：创建一个新的任务节点。

-  ChannelWaker接口
  - `new(node: Arc<CapsuleNode>)`：创建一个新的ChannelWaker实例。



**waker的设计**

## endpoint模块

#### Endpoint结构
Endpoint结构表示一个通信端点，每个端点在初始化时都会静态注册一个回调函数。当有请求到来时，请求会被封装成一个Capsule，并由内核调度器进行调度执行。

- **成员变量**：
  - `callback`：回调函数，用于处理请求。
  - `ipc_buf`：可选的IPC缓冲区，用于存储请求数据。

- **方法**：
  - `new(callback: fn(Box<IPCBuffer>) -> R)`：创建一个新的Endpoint实例，并注册回调函数。
  - `nb_send(buf_ptr: Box<IPCBuffer>) -> ReturnDataHook<R>`：非阻塞发送请求，并返回一个ReturnDataHook实例，用于管理返回数据。(描述如何使用)
  - `send(buf_ptr: Box<IPCBuffer>) -> R`：阻塞发送请求，等待返回结果。

#### ReturnDataHook结构

用于在nb_send时提供阻塞获取返回数据的功能。

ReturnDataHook结构用于管理任务的返回数据。它包装了一个CapsuleHandle实例，并提供了阻塞获取返回数据的功能。

- **成员变量**：
  - `hooked_data`：包含返回数据的CapsuleHandle实例。

- **方法**：
  - `new(hooked_data: CapsuleHandle<R>) -> Self`：创建一个新的ReturnDataHook实例。
  - `block(self) -> R`：阻塞等待返回数据，并返回结果。

- **特性实现**：
  - `Drop`：在Drop特性实现中，检查是否在任务完成之前被销毁，如果是，则抛出异常。

#### CapsuleHandle结构
CapsuleHandle结构用于管理任务的返回数据的句柄。它提供了获取返回数据的方法。

- **成员变量**：
  - `return_data`：包含返回数据的接收器。

- **方法**：
  - `new(rx: Receiver<R>) -> Self`：创建一个新的CapsuleHandle实例。
  - `try_take_data(self) -> Option<R>`：尝试获取返回数据，如果数据尚未准备好，返回None。

#### IntoCapsule特性
IntoCapsule特性用于将Endpoint实例转换为一个future类型的任务，并将其添加到任务队列中。

- **关联类型**：
  - `Output`：任务的返回类型。

- **方法**：
  - `resolve(self) -> impl Future<Output = Self::Output>`：将Endpoint实例转换为一个未来任务，执行回调函数并返回结果。
  - `add_to_job_queue(self) -> CapsuleHandle<Self::Output>`：将任务添加到执行队列中，并返回一个CapsuleHandle实例。

以下是 `sync/mod.rs` 模块的架构和模块设计的详细描述。

## channel模块

`sync/mod.rs` 模块提供了一种基于通道（Channel）机制的线程安全通信方法，用于在不同线程或内核组件之间传递消息。该模块通过无界的并发队列（SegQueue）和条件变量（Condvar）实现了发送和接收操作，使其适用于各种同步场景。该模块主要包含以下部分：

- **Channel结构**：负责消息的存储和通知机制。
- **Sender结构**：负责向Channel发送消息。
- **Receiver结构**：负责从Channel接收消息。

#### Channel结构

Channel结构是该模块的核心，负责管理消息队列和通知机制。它使用Arc<Mutex<SegQueue<T>>>来实现线程安全的无界队列，通过Arc<Condvar>来实现通知机制，确保在消息到达时能够及时通知等待的接收者。

- **成员变量**：
  - `inner`：一个线程安全的无界队列，用于存储消息。
  - `notifier`：条件变量，用于在消息到达时通知等待的接收者。

- **方法**：
  - `new()`：创建一个新的Channel实例，初始化消息队列和条件变量。

- **特性实现**：
  - `Clone`：实现了Clone特性，使Channel可以被安全地复制，以便在多个Sender和Receiver之间共享。

接口：

- `new()`：创建一个新的Channel实例。

#### Sender/Receiver结构
Sender和Receiver结构封装了Channel结构，负责将消息发送到Channel中or将。每个Sender/Receiver实例都持有一个Channel的引用，并通过它来执行发送操作。

- **成员变量**：包含Channel结构的实例。
  
- **方法**：
  - `Sender:send(&self, data: T)`：将消息发送到Channel中，并通过条件变量通知等待的接收者。
  - `Receiver:recv(&self) -> Option<T>`：阻塞接收消息，若没有消息可接收，则阻塞等待。
  - `Receiver:nb_recv(&self) -> Option<T>`：非阻塞接收消息，若没有消息可接收，则返回None。
  
- **特性实现**：
  - `Clone`：实现了Clone特性，使Sender可以被安全地复制，以便在多个线程中使用。
  - `Send`和`Sync`：通过unsafe代码块实现，使Sender能够跨线程安全地传递和共享。
