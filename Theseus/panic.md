## 测试堆资源释放

### 使用panic！

```rust
let boxed_data = Box::new(CustomData::new());
panic!("heap dealloc test");
```

- 由于编译器优化，Box::new不会在堆上申请/释放内存
  - 看一下hir这里是不是被优化了

```rust
let boxed_data = Box::new(CustomData::new());
match args.get(0).map(|s| &**s) {
    // cause a page fault to test unwinding through a machine exception
    Some("-p") => panic!("panic"),
    // test catch_unwind and then resume_unwind
    Some("-np") => test_np(),
    _ => test_np(),
};
```

- 通过用户命令行输入来分支，编译器无法判断是否会触发panic，测试时`boxed_data`被正确释放

```rust
kernel/unwind/src/lib.rs:874: 
Jumping to landing pad (cleanup function) at 0x3F2F896

applications/test_unwind/src/lib.rs:69: 
Dropping CustomData of size: 101 bytes

kernel/slabmalloc/src/sc.rs:324: 
SCAllocator(128) is trying to deallocate ptr = 0xfffffe8004d7e000 layout=Layout { size: 101, align: 1 (1 << 0) } P.size= 8192

```



### 使用unsafe

```rust
let boxed_data = Box::new(CustomData::new());
unsafe { *(0x5050DEADBEEF as *mut usize) = 0x5555_5555_5555; }
```

- 对一个随机的内存解引用

- 堆上allocate了CustomData，但是panic时没有deallocate操作

- 再次spawn这个task，失败：`AppCrateRef`没有被drop,

- 出现exeception时的unwinding并不一定成功，这一点在论文中有提及,文中提到仅有excepted stack可能没有正确释放资源.

- 为什么task没有正确退出

  > As hardware faults can occur at any instruction, Theseus’s unwinder may only find an inexact match for a faulted instruction pointer in the unwinding table, with a cleanup routine that may not completely release all resources acquired at the point of failure. Note that only local variables in the excepted stack frame may be missed, all other stack frames are properly handled.