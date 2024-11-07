## things need to be done during system bootstrap

- designate a continuous block of RAM as disk
- initialize filesystem,
  - Metadata in super block

## source code of redoxfs

- discard
  - Mount/
  - disk/
  - Bin/
  - key.rs
  - archive.rs
  - Unmount.rs
- Modify
  - Filesystem.rs->boot logic
  - Transaction.rs(discard encrtyption)
- best not to modify
  - Allocator,block,dir,header,node,record,tree
- reimplement
  - disk trait
  - scheme trait

## dependency and unsafe code

- unsafe
  - From_raw_parts(_mut)

- dependency

  ```rust
  # excluded
  range-tree = { version = "0.1", optional = true }
  termion = { version = "4", optional = true }
  log = { version = "0.4.14", default-features = false, optional = true}
  libc = "0.2"
  getrandom = { version = "0.2.5", optional = true }
  aes = { version = "=0.7.5", default-features = false }
  argon2 = { version = "0.4", default-features = false, features = ["alloc"] }
  base64ct = { version = "1", default-features = false }
  env_logger = { version = "0.11", optional = true }
  # used
  uuid = { version = "1.4", default-features = false }
  seahash = { version = "4.1.0", default-features = false }
  endian-num = "0.1"
  
  # redox related
  redox_syscall = { version = "0.5" }
  redox-path = "0.3.0"
  libredox = { version = "0.1.3", optional = true }
  redox-scheme = {  version = "0.2.1", optional = true }
  ```


# where are unsafe functions used in redoxfs

## BlockTrait

- 所有impl这个trait的struct应该都实现了deref这个trait
- 在transaction的function中如下四个需要参数是实现了blocktrait和deref的
  - read
  - read_block_or_empty
  - read_record
  - sync_block