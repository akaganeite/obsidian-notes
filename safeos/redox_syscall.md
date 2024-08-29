# syscall path

以mmap为例说明syscall的路径

- driver/common/dma..rs:

  ```rust
  fn alloc_and_map(length: usize) -> Result<(usize, *mut ())> {
      ... ...
          let virt = libredox::call::mmap(MmapArgs {
              fd: fd.raw(),
              offset: 0,                   // ignored
              addr: core::ptr::null_mut(), // ignored
              length,
              flags: flag::MAP_PRIVATE,
              prot: flag::PROT_READ | flag::PROT_WRITE,
          })?;
          ... ...
  }
  ```

- 使用libredox(repo libredox)的mmap，在libredox中被转换为对`redox_mmap_v1`的调用

- 该函数在relibc/src/platform/redox/libredox.rs中实现,被翻译为redox支持的syscall:fmap

  ```rust
  pub unsafe extern "C" fn redox_mmap_v1(
      addr: *mut (),
      unaligned_len: usize,
      prot: u32,
      flags: u32,
      fd: usize,
      offset: u64,
  ) -> RawResult {
      Error::mux(syscall::fmap(
          fd,
          &syscall::Map {
              address: addr as usize,
              offset: offset as usize,
              size: unaligned_len,
              flags: syscall::MapFlags::from_bits_truncate(
                  ((prot << 16) | (flags & 0xffff)) as usize,
              ),
          },
      ))
  }
  ```

- 在syscall这个repo中被翻译为syscall3

  ```rust
  pub unsafe fn fmap(fd: usize, map: &Map) -> Result<usize> {
      syscall3(SYS_FMAP, fd, map as *const Map as usize, mem::size_of::<Map>())
  }
  ```

- 之后进入kernel，详见syscall format

# syscall format

## redox kernel

上层posix兼容，将posix兼容接口转换为syscall1,syscall2,syscall3...的形式

```rust
//宏展开后的样子,以syscall3为例
pub unsafe fn syscall3(mut a: usize, b: usize, c: usize, d: usize) -> Result<usize> {
    asm!(
        "syscall",
        inout("rax") a,  // 系统调用号，通过寄存器 rax 传入
        in("rdi") b,     // 第一个参数，通过寄存器 rdi 传入
        in("rsi") c,     // 第二个参数，通过寄存器 rsi 传入
        in("rdx") d,     // 第三个参数，通过寄存器 rdx 传入
        out("rcx") _,    // rcx 用作临时寄存器，由内核修改
        out("r11") _,    // r11 用作临时寄存器，由内核修改
        options(nostack), // 不允许访问栈
    );
    Error::demux(a) // 使用返回的值检查是否有错误，并返回结果
}
syscall! {
    syscall0(a,);
    syscall1(a, b,);
    syscall2(a, b, c,);
    syscall3(a, b, c, d,);
    syscall4(a, b, c, d, e,);
    syscall5(a, b, c, d, e, f,);
}
```

执行syscall指令，

```rust
msr::wrmsr(msr::IA32_LSTAR, syscall_instruction as u64);//注册syscallhandle函数
```

保存现场等省略，最后会调用内核的syscall函数进行syscall dispatch

syscall类型是类linux的,dispatch时会参考当前的`scheme`,不同的scheme对同一个syscall有不同的处理方式

以fmap为例，

```rust
//dispatch
SYS_FMAP => {
    ... ...
    file_op_generic(fd, |scheme, number| {
        scheme.kfmap(number, &addrspace, &map, false)
    })
  }
}
```

- 这里将实际处理逻辑交给scheme.kfmap。实现了kfmap的scheme有：

  | **Name** | **sourcecode**                                               | **Description**                 |
  | -------- | ------------------------------------------------------------ | ------------------------------- |
  | `user`   | [user.rs](https://gitlab.redox-os.org/redox-os/kernel/-/blob/master/src/scheme/user.rs) | Dispatch for user-space schemes |
  | `proc`   | [proc.rs](https://gitlab.redox-os.org/redox-os/kernel/-/blob/master/src/scheme/proc.rs) | Process context manager         |
  | `memory` | [memory.rs](https://gitlab.redox-os.org/redox-os/kernel/-/blob/master/src/scheme/memory.rs) | Physical memory mapping manager |

### 以memory的fmap为例，解析一下内存映射syscall是如何进行的

`kfmap` 和 `physmap` 函数都是 `MemoryScheme` 结构体中处理内存映射的重要方法。它们主要负责将物理内存或匿名内存映射到用户进程的虚拟地址空间中。下面将详细解析这两个函数，并说明它们的作用和执行流程。

### `kfmap` 函数详解

`kfmap` 函数是 `MemoryScheme` 实现 `KernelScheme` 特性中的方法之一，它主要用于处理内存映射请求。

```rust
fn kfmap(
    &self,
    id: usize,
    addr_space: &Arc<AddrSpaceWrapper>,
    map: &Map,
    _consume: bool,
) -> Result<usize> {
    let (handle_ty, mem_ty, flags) = u32::try_from(id)
        .ok()
        .and_then(from_raw)
        .ok_or(Error::new(EBADF))?;

    match handle_ty {
        HandleTy::Allocated => Self::fmap_anonymous(
            addr_space,
            map,
            flags.contains(HandleFlags::PHYS_CONTIGUOUS),
        ),
        HandleTy::PhysBorrow => Self::physmap(map.offset, map.size, map.flags, mem_ty),
    }
}
```

#### 参数解析

- `id: usize`：表示内存句柄的标识符，包含内存类型和相关标志的编码信息。
- `addr_space: &Arc<AddrSpaceWrapper>`：表示当前进程的地址空间。
- `map: &Map`：包含映射请求的详细信息，如目标地址、大小和标志。
- `_consume: bool`：这个参数在该实现中没有被使用，可能与资源管理策略有关。

#### 处理流程

1. **解析内存句柄 (`id`)：**
   - 使用 `u32::try_from(id)` 将 `usize` 类型的 `id` 转换为 `u32` 类型。这个 `id` 是在 `kopen` 函数中生成的，包含了内存的类型信息。
   - 调用 `from_raw` 函数解析 `id`，将其转换为 `(HandleTy, MemoryType, HandleFlags)` 元组。如果解析失败，返回错误 `EBADF` (坏文件描述符)。

2. **根据内存句柄的类型执行对应的映射操作：**
   - 如果句柄类型为 `HandleTy::Allocated`（即匿名分配的内存），调用 `fmap_anonymous` 函数进行匿名内存映射。如果 `flags` 包含 `HandleFlags::PHYS_CONTIGUOUS`，则执行物理连续性映射。
   - 如果句柄类型为 `HandleTy::PhysBorrow`（即借用的物理内存），调用 `physmap` 函数将物理内存映射到虚拟地址空间中。

### `physmap` 函数详解

`physmap` 函数用于将指定的物理内存区域映射到当前进程的虚拟地址空间中。

```rust
pub fn physmap(
    physical_address: usize,
    size: usize,
    flags: MapFlags,
    memory_type: MemoryType,
) -> Result<usize> {
    // TODO: Check physical_address against the real MAXPHYADDR.
    let end = 1 << 52;
    if (physical_address.saturating_add(size) as u64) > end || physical_address % PAGE_SIZE != 0 {
        return Err(Error::new(EINVAL));
    }

    if size % PAGE_SIZE != 0 {
        log::warn!(
            "physmap size {} is not multiple of PAGE_SIZE {}",
            size,
            PAGE_SIZE
        );
        return Err(Error::new(EINVAL));
    }
    let page_count = NonZeroUsize::new(size.div_ceil(PAGE_SIZE)).ok_or(Error::new(EINVAL))?;

    let current_addrsp = AddrSpace::current()?;

    let base_page = current_addrsp.acquire_write().mmap_anywhere(
        &current_addrsp,
        page_count,
        flags,
        |dst_page, mut page_flags, dst_mapper, dst_flusher| {
            match memory_type {
                // Default
                MemoryType::Writeback => (),

                #[cfg(any(target_arch = "x86", target_arch = "x86_64"))] // TODO: AARCH64
                MemoryType::WriteCombining => {
                    page_flags = page_flags.custom_flag(EntryFlags::HUGE_PAGE.bits(), true)
                }

                MemoryType::Uncacheable => {
                    page_flags = page_flags.custom_flag(EntryFlags::NO_CACHE.bits(), true)
                }

                #[cfg(target_arch = "aarch64")]
                MemoryType::DeviceMemory => {
                    page_flags = page_flags.custom_flag(EntryFlags::DEV_MEM.bits(), true)
                }

                _ => (),
            }

            Grant::physmap(
                Frame::containing_address(PhysicalAddress::new(physical_address)),
                PageSpan::new(dst_page, page_count.get()),
                page_flags,
                dst_mapper,
                dst_flusher,
            )
        },
    )?;
    Ok(base_page.start_address().data())
}
```

#### 参数解析

- `physical_address: usize`：要映射的物理地址的起始位置。
- `size: usize`：要映射的内存区域大小，以字节为单位。
- `flags: MapFlags`：内存映射标志，控制内存映射的行为（例如是否共享、是否私有等）。
- `memory_type: MemoryType`：内存类型，决定映射的内存的特性，如缓存策略。

#### 处理流程

1. **输入参数检查：**
   - 确保物理地址与大小在合法范围内，且物理地址对齐到页面大小 (`PAGE_SIZE`) 的边界上。
   - 确保映射大小是页面大小的整数倍。如果不符合条件，则返回 `EINVAL` 错误。

2. **获取页面数量：**
   - 计算需要映射的页面数量 (`page_count`)，将内存大小除以页面大小，如果不能整除，向上取整。

3. **获取当前进程的地址空间：**
   - 调用 `AddrSpace::current()` 获取当前进程的地址空间，确保内存映射是在正确的地址空间中进行的。

4. **执行映射操作：**
   - 调用 `mmap_anywhere` 函数，在地址空间中找到合适的虚拟地址范围，将物理内存映射到这个范围。
   - 在映射过程中，根据 `memory_type` 设置页面标志。例如，对于 `WriteCombining` 内存类型，设置 `EntryFlags::HUGE_PAGE` 标志。

5. **返回虚拟地址：**
   - 成功映射后，返回映射区域的起始虚拟地址。

## redox支持的syscall

```rust
syscall/src/number.rs
pub const SYS_LINK: usize =     SYS_CLASS_PATH | SYS_ARG_PATH | 9;
pub const SYS_OPEN: usize =     SYS_CLASS_PATH | SYS_RET_FILE | 5;
pub const SYS_RMDIR: usize =    SYS_CLASS_PATH | 84;
pub const SYS_UNLINK: usize =   SYS_CLASS_PATH | 10;
pub const SYS_CLOSE: usize =      SYS_CLASS_FILE | 6;
pub const SYS_DUP: usize =        SYS_CLASS_FILE | SYS_RET_FILE | 41;
pub const SYS_DUP2: usize =       SYS_CLASS_FILE | SYS_RET_FILE | 63;
pub const SYS_READ: usize =       SYS_CLASS_FILE | SYS_ARG_MSLICE | 3;
pub const SYS_READ2: usize =      SYS_CLASS_FILE | SYS_ARG_MSLICE | 35;
pub const SYS_WRITE: usize =      SYS_CLASS_FILE | SYS_ARG_SLICE | 4;
pub const SYS_WRITE2: usize =     SYS_CLASS_FILE | SYS_ARG_SLICE | 45;
pub const SYS_LSEEK: usize =      SYS_CLASS_FILE | 19;
pub const SYS_FCHMOD: usize =     SYS_CLASS_FILE | 94;
pub const SYS_FCHOWN: usize =     SYS_CLASS_FILE | 207;
pub const SYS_FCNTL: usize =      SYS_CLASS_FILE | 55;
pub const SYS_FEVENT: usize =     SYS_CLASS_FILE | 927;
pub const SYS_SENDFD: usize =     SYS_CLASS_FILE | 34;
// TODO: Rename FMAP/FUNMAP to MMAP/MUNMAP
pub const SYS_FMAP_OLD: usize =   SYS_CLASS_FILE | SYS_ARG_SLICE | 90;
pub const SYS_FMAP: usize =       SYS_CLASS_FILE | SYS_ARG_SLICE | 900;
// TODO: SYS_FUNMAP should be SYS_CLASS_FILE
// TODO: Remove FMAP/FMAP_OLD
pub const SYS_FUNMAP_OLD: usize = SYS_CLASS_FILE | 91;
pub const SYS_FUNMAP: usize =     SYS_CLASS_FILE | 92;
pub const SYS_MREMAP: usize = 155;
pub const SYS_FPATH: usize =      SYS_CLASS_FILE | SYS_ARG_MSLICE | 928;
pub const SYS_FRENAME: usize =    SYS_CLASS_FILE | SYS_ARG_PATH | 38;
pub const SYS_FSTAT: usize =      SYS_CLASS_FILE | SYS_ARG_MSLICE | 28;
pub const SYS_FSTATVFS: usize =   SYS_CLASS_FILE | SYS_ARG_MSLICE | 100;
pub const SYS_FSYNC: usize =      SYS_CLASS_FILE | 118;
pub const SYS_FTRUNCATE: usize =  SYS_CLASS_FILE | 93;
pub const SYS_FUTIMENS: usize =   SYS_CLASS_FILE | SYS_ARG_SLICE | 320;
// b = file, c = flags, d = required_page_count, uid:gid = offset
pub const KSMSG_MMAP: usize = SYS_CLASS_FILE | 72;
// b = file, c = flags, d = page_count, uid:gid = offset
pub const KSMSG_MSYNC: usize = SYS_CLASS_FILE | 73;
// b = file, c = page_count, uid:gid = offset
pub const KSMSG_MUNMAP: usize = SYS_CLASS_FILE | 74;
// b = file, c = flags, d = page_count, uid:gid = offset
pub const KSMSG_MMAP_PREP: usize = SYS_CLASS_FILE | 75;
// b = target_packetid_lo32, c = target_packetid_hi32
pub const KSMSG_CANCEL: usize = SYS_CLASS_FILE | 76;
pub const SYS_CLOCK_GETTIME: usize = 265;
pub const SYS_EXIT: usize =     1;
pub const SYS_FUTEX: usize =    240;
pub const SYS_GETEGID: usize =  202;
pub const SYS_GETENS: usize =   951;
pub const SYS_GETEUID: usize =  201;
pub const SYS_GETGID: usize =   200;
pub const SYS_GETNS: usize =    950;
pub const SYS_GETPID: usize =   20;
pub const SYS_GETPGID: usize =  132;
pub const SYS_GETPPID: usize =  64;
pub const SYS_GETUID: usize =   199;
pub const SYS_IOPL: usize =     110;
pub const SYS_KILL: usize =     37;
pub const SYS_MPROTECT: usize = 125;
pub const SYS_MKNS: usize =     984;
pub const SYS_NANOSLEEP: usize =162;
pub const SYS_VIRTTOPHYS: usize=949;
pub const SYS_SETPGID: usize =  57;
pub const SYS_SETREGID: usize = 204;
pub const SYS_SETRENS: usize =  952;
pub const SYS_SETREUID: usize = 203;
pub const SYS_UMASK: usize =    60;
pub const SYS_WAITPID: usize =  7;
pub const SYS_YIELD: usize =    158;
```



# 什么是scheme

