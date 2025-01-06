## create操作更改了redoxfs的什么状态

- 参数：`create(parent_id: u64, name: &str, mode: u32, _umask: u32, _flags: i32) -> usize`
- `FILE_SYSTEM`

- `DiskCache`

```rust
pub struct FileSystem<D: Disk> {
    pub disk: D,
    pub block: u64,
    pub header: Header,
    pub(crate) allocator: Allocator,
    pub(crate) aes_opt: Option<crate::key::Aes128>,
    // _phantom: core::marker::PhantomData<U>,
}

pub struct DiskCache<T> {
    inner: T,
    cache: HashMap<u64, [u8; BLOCK_SIZE as usize]>,
    order: VecDeque<u64>,
    size: usize,
}
```

