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

![image-20240708111718867](C:\Users\jetta\AppData\Roaming\Typora\typora-user-images\image-20240708111718867.png)

## 问题

- exec函数何时执行
- 没有使用waker
- send时不能在resolve里面await

