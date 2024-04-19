## POSIX

musllibc提供383个系统调用接口

## SEL4

- set_thread_area

- set_tid_address 部分实现

- writev/write

- readv/read

- sched_yield->seL4_Yield

- exit

- tkill/tgkill->仅支持杀死当前进程

- open/openat 仅支持只读模式打开

- close

- prlimit64

- lseek/llseek

- access

- brk

- mmap2/mmap

- mremap

  共21个

## Linux

- riscv：441
- arm-64：292
- x86-64：333

## Cantrip

```c
enum syscall {
    SysCall = -1,
    SysReplyRecv = -2,
    SysNBSendRecv = -3,
    SysNBSendWait = -4,
    SysSend = -5,
    SysNBSend = -6,
    SysRecv = -7,
    SysNBRecv = -8,
    SysWait = -9,
    SysNBWait = -10,
    SysYield = -11,
#if defined CONFIG_PRINTING
    SysDebugPutChar = -12,
    SysDebugDumpScheduler = -13,
    SysDebugDumpCNode = -14,
#endif /* defined CONFIG_PRINTING */
#if defined CONFIG_DEBUG_BUILD
    SysDebugHalt = -15,
    SysDebugCapIdentify = -16,
    SysDebugSnapshot = -17,
    SysDebugNameThread = -18,
#endif /* defined CONFIG_DEBUG_BUILD */
#if defined CONFIG_DEBUG_BUILD && CONFIG_MAX_NUM_NODES > 1
    SysDebugSendIPI = -19,
#endif /* defined CONFIG_DEBUG_BUILD && CONFIG_MAX_NUM_NODES > 1 */
#if defined CONFIG_DANGEROUS_CODE_INJECTION
    SysDebugRun = -20,
#endif /* defined CONFIG_DANGEROUS_CODE_INJECTION */
#if defined CONFIG_ENABLE_BENCHMARKS
    SysBenchmarkFlushCaches = -21,
    SysBenchmarkResetLog = -22,
    SysBenchmarkFinalizeLog = -23,
    SysBenchmarkSetLogBuffer = -24,
    SysBenchmarkNullSyscall = -25,
#endif /* defined CONFIG_ENABLE_BENCHMARKS */
#if defined CONFIG_BENCHMARK_TRACK_UTILISATION
    SysBenchmarkGetThreadUtilisation = -26,
    SysBenchmarkResetThreadUtilisation = -27,
#endif /* defined CONFIG_BENCHMARK_TRACK_UTILISATION */
#if defined CONFIG_DEBUG_BUILD && defined CONFIG_BENCHMARK_TRACK_UTILISATION
    SysBenchmarkDumpAllThreadsUtilisation = -28,
    SysBenchmarkResetAllThreadsUtilisation = -29,
#endif /* defined CONFIG_DEBUG_BUILD && defined CONFIG_BENCHMARK_TRACK_UTILISATION */
#if defined CONFIG_KERNEL_X86_DANGEROUS_MSR
    SysX86DangerousWRMSR = -30,
    SysX86DangerousRDMSR = -31,
#endif /* defined CONFIG_KERNEL_X86_DANGEROUS_MSR */
#if defined CONFIG_VTX
    SysVMEnter = -32,
#endif /* defined CONFIG_VTX */
#if defined CONFIG_SET_TLS_BASE_SELF
    SysSetTLSBase = -33,
#endif /* defined CONFIG_SET_TLS_BASE_SELF */
};
typedef word_t syscall_t;
```

共33个

