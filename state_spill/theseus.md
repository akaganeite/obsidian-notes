# How theseus deal with state spill



## crates

- root:`a special concrete implementation of the Directory trait`

- Fsnode:`defines the traits for File and Directory, defines common methods for both Files and Directories`



## FS node

```rust
pub trait File : FsNode + ByteReader + ByteWriter + KnownLength 

pub trait Directory : FsNode

/// A trait that covers any filesystem node, both files and directories.
pub trait FsNode

#[derive(Clone)]
pub enum FileOrDir {
    File(FileRef),
    Dir(DirRef),
}

/// A reference to any type that implements the [`File`] trait,
/// which can only represent a File (not a Directory).
pub type FileRef = Arc<Mutex<dyn File + Send>>;
/// A weak reference to any type that implements the [`File`] trait,
/// which can only represent a File (not a Directory).
pub type WeakFileRef = Weak<Mutex<dyn File + Send>>;
/// A reference to any type that implements the [`Directory`] trait,
/// which can only represent a Directory (not a File).
pub type DirRef =  Arc<Mutex<dyn Directory + Send>>;
/// A weak reference to any type that implements the [`Directory`] trait,
/// which can only represent a Directory (not a File).
pub type WeakDirRef = Weak<Mutex<dyn Directory + Send>>;

```

## Root

- 文件系统的根目录，没有父节点

```rust
lazy_static! {
    /// The root directory
    /// Returns a tuple for easy access to the name of the root so we don't have to lock it
    pub static ref ROOT: (String, DirRef) = {
        let root_dir = RootDirectory {
            children: BTreeMap::new() 
        };
        let strong_root = Arc::new(Mutex::new(root_dir)) as DirRef;
        (ROOT_DIRECTORY_NAME.to_string(), strong_root)
    };
}
```

- RootDirectory.children的btreemap中的Key为fileordir enum，这个enum会拿ROOT目录下的文件或目录的arc指针。
  - ？？？/目录下的所有文件的ownership都由ROOT持有？？？
- ROOT的get_parent实现为返回`/`,即当前目录

## HeapFile

- 内存文件，结构体拥有一个堆上数组代表文件存储空间，类似磁盘，但是数据是分布式存储的不是集中存储
- 创建文件时需要提供父目录的arc指针，在父目录(ROOT)中存有当前新文件的arc指针，file内部存储parent的weak指针