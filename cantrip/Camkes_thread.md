# Relevent structures

## Camkes

```rust
//cantrip-os-common:camkes/src/lib.rs
pub struct Camkes {//代表一个component?
    name: &'static str, // Component name
    pre_init: &'static baresema::seL4_BareSema,
    post_init: &'static baresema::seL4_BareSema,
    interface_init: &'static baresema::seL4_BareSema,
    threads: &'static [&'static CamkesThread],
    recv_path: Mutex<seL4_CPath>, // IPCBuffer receive path
}
//实现
impl Camkes {
    pub const fn new()
    pub fn thread(&self, thread_id: usize) -> Option<&CamkesThread>
    pub fn init_allocator(&self, heap: &'static mut [u8])
    pub fn init_slot_allocator(&self, first_slot: seL4_CPtr, last_slot: seL4_CPtr)
    pub fn pre_init(&self, heap: &'static mut [u8])
    #[inline]
    pub fn top_level_path(slot: seL4_CPtr) -> seL4_CPath
    // Initializes the IPCBuffer receive path with |path|.
    pub fn init_recv_path(&self, path: &seL4_CPath)

    // Returns the path specified with init_recv_path.
    pub fn get_recv_path(&self) -> seL4_CPath { *self.recv_path.lock() }

    // Returns the component name.
    #[inline]
    pub fn get_name(&self) -> &'static str { self.name }

    // Returns the current receive path from the IPCBuffer.
    pub fn get_current_recv_path(&self) -> seL4_CPath

    // Returns the current receive path from the IPCBuffer; clears any
    // capability the cpath points to when dropped.
    #[must_use]
    pub fn get_owned_current_recv_path(&self) -> OwnedCPath

    // Check the current receive path in the IPCBuffer against what was
    // setup with init_recv_path.
    pub fn check_recv_path(&self)

    // Like check_recv_path but asserts if there is an inconsistency.
    pub fn assert_recv_path(&self)
    // Deletes the capability at |path|,
    pub fn delete_path(path: &seL4_CPath) -> seL4_Result
    // Attaches a capability to a CAmkES RPC request MessageInfo and
    // returns a helper to reset/cleanup on block exit. seL4 will copy
    // the capabilty if the MessageInfo indicates there are capabilities
    // attached (beware this is currently buried in the CAmkES template).
    #[must_use]
    pub fn set_request_cap(cptr: seL4_CPtr) -> RequestCapCleanup
    // Arranges for the CAmkES RPC request capability be clear'd on
    // block exit. This is to guard against accidentally attaching a
    // capability to a reply.
    // TODO(sleffler): remove after the C templates are replaced
    pub fn cleanup_request_cap() -> RequestCapCleanup
    // Immediately clears any capability attached to a CAmkES RPC request
    // msg. NB: cleanup_request_cap may be more useful.
    pub fn clear_request_cap() { set_cap(0); }
    // Returns the capability attached to an seL4 IPC.
    pub fn get_request_cap() -> seL4_CPtr { get_cap() }
    // Attaches a capability to a CAmkES RPC reply msg and arranges for
    // resources to be released after the reply completes.
    #[must_use]
    pub fn set_reply_cap_release(cptr: seL4_CPtr) -> ReplyCapRelease
    // Clears any capability attached to a CAmkES RPC reply msg.
    // XXX dangerous
    pub fn clear_reply_cap() { set_cap(0); }
    // Returns the capability attached to an seL4 IPC.
    pub fn get_reply_cap() -> seL4_CPtr { get_cap() }
    // Wrappers for sel4_sys::debug_assert macros.
    pub fn debug_assert_slot_empty(tag: &str, path: &seL4_CPath)
    pub fn debug_assert_slot_cnode(tag: &str, path: &seL4_CPath)
    pub fn debug_assert_slot_frame(tag: &str, path: &seL4_CPath)
    // debug_assert wrappers for the current recv_path.
    pub fn debug_assert_recv_path_empty(&self, tag: &str)
    pub fn debug_assert_recv_path_cnode(&self, tag: &str)
    pub fn debug_assert_recv_path_frame(&self, tag: &str)
    // Dumps the contents of the toplevel CNode to the serial console.
    pub fn capscan()
}
//startup.rs
/// Synchronization helpers for CamkesThreadStart::start. These are run on
/// the component's control thread (see above).
impl Camkes {
    /// Handles synchronization of interface threads after the control
    /// thread's pre_init method runs.
    pub fn pre_init_sync(&self)
    /// Handles synchronization of interface threads after the control
    /// thread's post_init method runs.
    pub fn post_init_sync(&self)
```

## Camkes thread

```rust
pub enum CamkesThread {
    Control(
        seL4_CPtr,
        &'static str,
        &'static seL4_IPCBuffer,
        &'static StaticTLS,
        &'static Camkes,
    ),
    Interface(
        seL4_CPtr,
        &'static str,
        &'static seL4_IPCBuffer,
        &'static StaticTLS,
        &'static Camkes,
    ),
    PassiveInterface(
        seL4_CPtr,
        &'static str,
        &'static seL4_IPCBuffer,
        &'static StaticTLS,
        &'static Camkes,
    ),
}
```

