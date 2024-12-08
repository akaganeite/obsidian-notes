```shell
lmbench_musl lat_syscall -P 1 null 
lmbench_musl lat_syscall -P 1 read
lmbench_musl lat_syscall -P 1 write
lmbench_musl lat_syscall -P 1 stat hello 
lmbench_musl lat_syscall -P 1 fstat hello 
lmbench_musl lat_syscall -P 1 open test 
lmbench_musl lat_sig -P 1 install 
lmbench_musl lat_sig -P 1 catch  
lmbench_musl lat_pipe -P 1 
lmbench_musl lat_proc -P 1 fork 
lmbench_musl lat_proc -P 1 exec 
lmbench_musl lat_proc -P 1 shell 
lmbench_musl lmdd of=/dev/zero move=10m fsync=1 print=3 
lmbench_musl lat_pagefault -P 1 /test 
lmbench_musl lat_mmap -P 1 512k /test 
lmbench_musl lat_fs  
lmbench_musl bw_pipe -P 1 -m 1024 -M 1024 
lmbench_musl bw_file_rd -P 1 512k io_only /test 
lmbench_musl bw_file_rd -P 1 512k open2close /test 
lmbench_musl bw_mmap_rd -P 1 512k mmap_only /test 
lmbench_musl bw_mmap_rd -P 1 512k open2close /test 
lmbench_musl lat_ctx -P 1 -s 32 2 4 8 16 24 32 64 96
```

```
./lmbench_all lat_syscall -P 1 null
```

```
mkdir mnt
sudo mount rootfs.img mnt
su
sudo cp -r ./* ../mnt
cd ..
sudo umount mnt


sudo qemu-system-riscv64 -nographic -machine virt \
    -kernel linux/arch/riscv/boot/Image \
    -append "root=/dev/vda rw console=ttyS0 init=/bin/sh" \
    -drive file=rootfs.img,format=raw,id=hd0 \
    -device virtio-blk-device,drive=hd0

sudo qemu-system-riscv64 -nographic -machine virt \
     -kernel linux/arch/riscv/boot/Image -append "root=/dev/vda ro console=ttyS0" \
     -drive file=rootfs.img,format=raw,id=hd0 \
     -device virtio-blk-device,drive=hd0
     
qemu-system-riscv64 -M virt -m 256M -nographic \
  -kernel linux/arch/riscv/boot/Image \
  -drive file=rootfs.img,format=raw,id=hd0 \
  -device virtio-blk-device,drive=hd0 \
  -append "root=/dev/vda rw console=ttyS0"
  
  
  qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -kernel vmlinux \
    -initrd rootfs.cpio.gz \
    -append "root=/dev/ram rw console=ttyS0"

```

|                    test                    |                            result                            |
| :----------------------------------------: | :----------------------------------------------------------: |
|           lat_syscall -P 1 null            |            Simple syscall: `3.7935` microseconds             |
|           lat_syscall -P 1 read            |              Simple read:`79.8806` microseconds              |
|           lat_syscall -P 1 write           |            Simple write: `212.0769` microseconds             |
|        lat_syscall -P 1 stat hello         |             Simple stat: `188.5862` microseconds             |
|        lat_syscall -P 1 fstat hello        |             Simple fstat: `9.4718` microseconds              |
|        lat_syscall -P 1 open hello         |          Simple open/close: `194.0000` microseconds          |
|            lat_sig -P 1 install            |      Signal handler installation: `7.7637` microseconds      |
|               lat_pipe -P 1                |             Pipe latency: `99.8388` microseconds             |
|             lat_proc -P 1 fork             |          Process fork+exit: `397.5833` microseconds          |
|             lat_proc -P 1 exec             |         Process fork+execve: `416.5000` microseconds         |
|            lat_proc -P 1 shell             |       Process fork+/bin/sh -c: `396.3571` microseconds       |
| lmdd of=/dev/zero move=10m fsync=1 print=3 |                        `29914` KB/sec                        |
|          lat_pagefault -P 1 test           |          Pagefaults on test: `28.0700` microseconds          |
|          lat_mmap -P 1 512k test           |                        `0.524288 141`                        |
|            lmbench_musl lat_fs             | 0k      11      986     28281k      9       752     2701  4k      8       730     253410k     8       697     2396 |
|        bw_pipe -P 1 -m 1024 -M 1024        |                Pipe bandwidth: `19.09` MB/sec                |
|    bw_mmap_rd -P 1 512k mmap_only test     |                    `0.524288` `15551.29`                     |
|    bw_mmap_rd -P 1 512k open2close test    |                     `0.524288` `301.57`                      |
|     bw_file_rd -P 1 512k io_only test      |                     `0.524288` `343.63`                      |
|                                            |                                                              |
|          VERSION:2024/11/25:14:09          |                                                              |
|                                            |                                                              |
|                    test                    |                            result                            |
|           lat_syscall -P 1 null            |         Simple syscall: `2.7625` microseconds 0.5598         |
|           lat_syscall -P 1 read            |          Simple read:` 64.3000` microseconds 1.1123          |
|           lat_syscall -P 1 write           |         Simple write: `229.4348` microseconds 0.8240         |
|        lat_syscall -P 1 stat hello         |         Simple stat: `275.0056` microseconds 6.4845          |
|        lat_syscall -P 1 fstat hello        |         Simple fstat: `14.2895` microseconds 1.5641          |
|        lat_syscall -P 1 open hello         |      Simple open/close: `339.2267` microseconds 10.2750      |
|            lat_sig -P 1 install            |  Signal handler installation: `26.1521` microseconds 1.5308  |
|               lat_pipe -P 1                |         Pipe latency: `99.8388` microseconds 61.3979         |
|             lat_proc -P 1 fork             |     Process fork+exit: `417.5385` microseconds 970.4259      |
|             lat_proc -P 1 exec             |    Process fork+execve: `356.3750` microseconds 1012.5660    |
|            lat_proc -P 1 shell             |  Process fork+/bin/sh -c: `349.4286` microseconds 6769.1083  |
| lmdd of=/dev/zero move=10m fsync=1 print=3 |                    `29532` KB/sec 478514                     |
|          lat_pagefault -P 1 test           |     Pagefaults on test: `119.2568` microseconds.  9.1166     |
|          lat_mmap -P 1 512k test           |                    `0.524288` `125`. 163                     |
|            lmbench_musl lat_fs             |                                                              |
|        bw_pipe -P 1 -m 1024 -M 1024        |                Pipe bandwidth: `19.09` MB/sec                |
|    bw_mmap_rd -P 1 512k mmap_only test     |                  `0.524288 14301.42`  9371                   |
|    bw_mmap_rd -P 1 512k open2close test    |                   `0.524288` `137.14` 337                    |
|     bw_file_rd -P 1 512k io_only /test     |                    `0.524288 302.91` 2352                    |
|   bw_file_rd -P 1 512k open2close /test    |                   `0.524288` `273.85` 2220                   |



./lmbench_musl lat_syscall -P 1 null
Simple syscall: 0.5598 microseconds
./lmbench_musl lat_syscall -P 1 read
Simple read: 1.1123 microseconds
./lmbench_musl lat_syscall -P 1 write
Simple write: 0.8240 microseconds
./lmbench_musl lat_syscall -P 1 stat /var/tmp/lmbench
Simple stat: 6.4845 microseconds
./lmbench_musl lat_syscall -P 1 fstat /var/tmp/lmbench
Simple fstat: 1.5641 microseconds
./lmbench_musl lat_syscall -P 1 open /var/tmp/lmbench
Simple open/close: 10.2750 microseconds
./lmbench_musl lat_sig -P 1 install
Signal handler installation: 1.5308 microseconds
./lmbench_musl lat_pipe -P 1
Pipe latency: 61.3979 microseconds
./lmbench_musl lat_proc -P 1 fork
Process fork+exit: 970.4259 microseconds
./lmbench_musl lat_proc -P 1 exec
Process fork+execve: 1012.5660 microseconds
./lmbench_musl lat_proc -P 1 shell
Process fork+/bin/sh -c: 6769.1083 microseconds
./lmbench_musl lmdd label="File /var/tmp/XXX write bandwidth:" of=/var/tmp/XXX move=645m fsync=1 print=3
write: wanted=8192 got=4096
File /var/tmp/XXX write bandwidth:478514 KB/sec
./lmbench_musl lat_pagefault -P 1 /var/tmp/XXX
Pagefaults on /var/tmp/XXX: 9.1166 microseconds
./lmbench_musl lat_mmap -P 1 512k /var/tmp/XXX
0.524288 163
./lmbench_musl lat_fs /var/tmp
0k      2444    45520   60848
1k      1258    24003   61041
4k      1203    23845   61246
10k     1206    23939   61127
./lmbench_musl bw_pipe -P 1
Pipe bandwidth: 528.60 MB/sec
./lmbench_musl bw_file_rd -P 1 512k io_only /var/tmp/XXX
0.524288 2352.83
./lmbench_musl bw_file_rd -P 1 512k open2close /var/tmp/XXX
0.524288 2220.75
./lmbench_musl bw_mmap_rd -P 1 512k mmap_only /var/tmp/XXX
0.524288 9371.14
./lmbench_musl bw_mmap_rd -P 1 512k open2close /var/tmp/XXX
0.524288 377.36
./lmbench_musl lat_ctx -P 1 -s 32 2 4 8 16 24 32 64 96
size=32k ovr=6.72
2 28.68
4 34.00
8 35.03
16 36.15
24 35.96
32 37.67
64 38.99
96 38.58



//Redox supported lmbench. These tests are reimplemented in rust 
./lmbench null
./lmbench ctx
./lmbench bw_file_rd
./lmbench lat_fs
./lmbench lat_pipe
./lmbench lat_mmap 512
./lmbench bw_mmap_rd 512 mmap_only
./lmbench bw_mmap_rd 512 open2close
./lmbench bw_pipe 1024 1024

./lmbench page_fault



# strace on riscv-linux

pipe2([3, 4], 0)                        = 0
pipe2([5, 6], 0)                        = 0
pipe2([7, 8], 0)                        = 0
pipe2([9, 10], 0)                       = 0
rt_sigprocmask(SIG_UNBLOCK, [RT_1 RT_2], NULL, 8) = 0
rt_sigaction(SIGTERM, {sa_handler=0x56942, sa_mask=[ABRT BUS FPE USR2 ALRM TERM STKFLT CONT STOP], sa_flags=SA_RESTART|0x4000000}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGCHLD, {sa_handler=0x56968, sa_mask=[ABRT BUS FPE USR2 ALRM TERM STKFLT CONT STOP], sa_flags=SA_RESTART|0x4000000}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
brk(NULL)                               = 0x7c000
brk(0x7e000)                            = 0x7e000
mmap(0x7c000, 4096, PROT_NONE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7c000
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fff9f8ae000
rt_sigprocmask(SIG_BLOCK, ~[RTMIN RT_1 RT_2], [], 8) = 0
rt_sigprocmask(SIG_BLOCK, ~[], ~[KILL STOP RTMIN RT_1 RT_2], 8) = 0
clone(child_stack=NULL, flags=SIGCHLD)  = 98
rt_sigprocmask(SIG_SETMASK, ~[KILL STOP RTMIN RT_1 RT_2], NULL, 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
close(4)                                = 0
close(5)                                = 0
close(7)                                = 0
close(9)                                = 0
pselect6(4, [3], NULL, [3], {tv_sec=1, tv_nsec=0}, {sigmask=NULL, sigsetsize=8}) = 1 (in [3], left {tv_sec=0, tv_nsec=999876800})
read(3, "\0", 1)                        = 1
write(6, "\0", 1)                       = 1
pselect6(4, [3], NULL, [3], {tv_sec=1, tv_nsec=0}, {sigmask=NULL, sigsetsize=8}) = 0 (Timeout)
pselect6(4, [3], NULL, [3], {tv_sec=1, tv_nsec=0}, {sigmask=NULL, sigsetsize=8}) = 0 (Timeout)
pselect6(4, [3], NULL, [3], {tv_sec=1, tv_nsec=0}, {sigmask=NULL, sigsetsize=8}) = 0 (Timeout)
pselect6(4, [3], NULL, [3], {tv_sec=1, tv_nsec=0}, {sigmask=NULL, sigsetsize=8}) = 0 (Timeout)
pselect6(4, [3], NULL, [3], {tv_sec=1, tv_nsec=0}, {sigmask=NULL, sigsetsize=8}) = 0 (Timeout)
pselect6(4, [3], NULL, [3], {tv_sec=1, tv_nsec=0}, {sigmask=NULL, sigsetsize=8}) = 0 (Timeout)
pselect6(4, [3], NULL, [3], {tv_sec=1, tv_nsec=0}, {sigmask=NULL, sigsetsize=8}) = 0 (Timeout)
pselect6(4, [3], NULL, [3], {tv_sec=1, tv_nsec=0}, {sigmask=NULL, sigsetsize=8}) = 0 (Timeout)
pselect6(4, [3], NULL, [3], {tv_sec=1, tv_nsec=0}, {sigmask=NULL, sigsetsize=8}) = 0 (Timeout)
pselect6(4, [3], NULL, [3], {tv_sec=1, tv_nsec=0}, {sigmask=NULL, sigsetsize=8}) = 0 (Timeout)
pselect6(4, [3], NULL, [3], {tv_sec=1, tv_nsec=0}, {sigmask=NULL, sigsetsize=8}) = 0 (Timeout)
pselect6(4, [3], NULL, [3], {tv_sec=1, tv_nsec=0}, {sigmask=NULL, sigsetsize=8}) = 0 (Timeout)
pselect6(4, [3], NULL, [3], {tv_sec=1, tv_nsec=0}, {sigmask=NULL, sigsetsize=8}) = 1 (in [3], left {tv_sec=0, tv_nsec=92568400})
read(3, "\0", 1)                        = 1
write(8, "\0", 1)                       = 1
pselect6(4, [3], NULL, [3], {tv_sec=1, tv_nsec=0}, {sigmask=NULL, sigsetsize=8}) = 1 (in [3], left {tv_sec=0, tv_nsec=999869500})
read(3, "\v\0\0\0\0\0\0\0\374\356\20\0\0\0\0\0\340\345\36\0\0\0\0\08\342\20\0\0\0\0\0"..., 184) = 184
rt_sigaction(SIGCHLD, {sa_handler=SIG_DFL, sa_mask=[ABRT BUS FPE USR2 ALRM TERM STKFLT CONT STOP], sa_flags=SA_RESTART|0x4000000}, {sa_handler=0x56968, sa_mask=[ABRT BUS FPE USR2 ALRM TERM STKFLT CONT], sa_flags=SA_RESTART}, 8) = 0
write(10, "\v", 1)                      = 1
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=98, si_uid=0, si_status=0, si_utime=2 /* 0.02 s */, si_stime=1289 /* 12.89 s */} ---
close(3)                                = 0
close(6)                                = 0
close(8)                                = 0
close(10)                               = 0
rt_sigaction(SIGCHLD, {sa_handler=SIG_DFL, sa_mask=[ABRT BUS FPE USR2 ALRM TERM STKFLT CONT STOP], sa_flags=SA_RESTART|0x4000000}, {sa_handler=SIG_DFL, sa_mask=[ABRT BUS FPE USR2 ALRM TERM STKFLT CONT], sa_flags=SA_RESTART}, 8) = 0
rt_sigaction(SIGALRM, {sa_handler=0x569e6, sa_mask=[ABRT BUS FPE USR2 ALRM TERM STKFLT CONT STOP], sa_flags=SA_RESTART|0x4000000}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
setitimer(ITIMER_REAL, {it_interval={tv_sec=0, tv_usec=0}, it_value={tv_sec=5, tv_usec=0}}, {it_interval={tv_sec=0, tv_usec=0}, it_value={tv_sec=0, tv_usec=0}}) = 0
wait4(98, NULL, 0, NULL)                = 98
setitimer(ITIMER_REAL, {it_interval={tv_sec=0, tv_usec=0}, it_value={tv_sec=0, tv_usec=0}}, {it_interval={tv_sec=0, tv_usec=0}, it_value={tv_sec=4, tv_usec=998513}}) = 0
rt_sigaction(SIGALRM, {sa_handler=SIG_DFL, sa_mask=[ABRT BUS FPE USR2 ALRM TERM STKFLT CONT STOP], sa_flags=SA_RESTART|0x4000000}, {sa_handler=0x569e6, sa_mask=[ABRT BUS FPE USR2 ALRM TERM STKFLT CONT], sa_flags=SA_RESTART}, 8) = 0
writev(2, [{iov_base="Simple syscall: 0.5427 microseco"..., iov_len=36}, {iov_base=NULL, iov_len=0}], 2Simple syscall: 0.5427 microseconds
) = 36
exit_group(0)                           = ?
+++ exited with 0 +++





![截屏2024-11-05 09.39.19](./assets/截屏2024-11-05 09.39.19.png)

子线程第一次getppid后的第一个pselect返回后的 backtrace





lmbench_musl lmdd of=/dev/zero move=10m fsync=1 print=3 ✅
lmbench_musl lat_pagefault -P 1 /test ✅
lmbench_musl lat_mmap -P 1 512k /test ✅
busybox echo file system latency
lmbench_musl lat_fs /var/tmp  ✅ 
busybox echo Bandwidth measurements
lmbench_musl bw_pipe -P 1 -m 1 -M 1024 ✅
lmbench_musl bw_file_rd -P 1 512k io_only /test ✅
lmbench_musl bw_file_rd -P 1 512k open2close /test ✅
lmbench_musl bw_mmap_rd -P 1 512k mmap_only /test ✅
lmbench_musl bw_mmap_rd -P 1 512k open2close /test ✅
busybox echo context switch overhead
lmbench_musl lat_ctx -P 1 -s 32 2 4 8 16 24 32 64 96 //partial success





# Lmbench musl result on arm

|                          test                           |    Linux(raspi board)    |     Safeos(raspi qemu)      |                                  |
| :-----------------------------------------------------: | :----------------------: | :-------------------------: | :------------------------------: |
|           lmbench_musl lat_syscall -P 1 null            |          2.6150          |           4.1722            |                                  |
|           lmbench_musl lat_syscall -P 1 read            |          5.3967          |           4.9149            |                                  |
|           lmbench_musl lat_syscall -P 1 write           |          2.9577          |           5.4120            |                                  |
|        lmbench_musl lat_syscall -P 1 stat hello         |         15.6598          |          273.6500           |             Redoxfs              |
|        lmbench_musl lat_syscall -P 1 fstat hello        |          5.7010          |           5.3044            |                                  |
|        lmbench_musl lat_syscall -P 1 open hello         |         29.7511          |          212.2692           |                                  |
|            lmbench_musl lat_sig -P 1 install            |          8.4321          |           6.1499            |                                  |
|             lmbench_musl lat_sig -P 1 catch             |         221.0400         |           0.9610            |    not implemented in safeos?    |
|               lmbench_musl lat_pipe -P 1                |         71.0699          |          151.4508           |                                  |
|        lmbench_musl bw_pipe -P 1 -m 1024 -M 1024        |        179.71MB/s        |          16.73MB/s          |                                  |
|     lmbench_musl bw_file_rd -P 1 512k io_only /test     |          741.44          |           286.91            |                                  |
|   lmbench_musl bw_file_rd -P 1 512k open2close /test    |          655.24          |           254.98            |                                  |
|    lmbench_musl bw_mmap_rd -P 1 512k mmap_only /test    |         4009.85          |           4208.52           |                                  |
|   lmbench_musl bw_mmap_rd -P 1 512k open2close /test    |         1040.25          |            73.63            | file backed mmap not implemented |
|             lmbench_musl lat_proc -P 1 fork             |         769.6250         |          688.6875           |                                  |
|             lmbench_musl lat_proc -P 1 exec             |         997.8333         |          640.7778           |                                  |
|            lmbench_musl lat_proc -P 1 shell             |        8282.0000         |          685.5000           |                                  |
| lmbench_musl lmdd of=/dev/zero move=10m fsync=1 print=3 |       17744 KB/sec       |        160800 KB/sec        |         We have no sync          |
|          lmbench_musl lat_pagefault -P 1 /test          |          2.4193          |           50.7938           |                                  |
|          lmbench_musl lat_mmap -P 1 512k /test          |            59            |             251             |                                  |
|                   lmbench_musl lat_fs                   | 0k    61    10696  13463 | 0k      45      763     809 |                                  |
|                                                         | 1k    40    6809   10664 | 1k      36      617     846 |                                  |
|                                                         | 4k    39    7052   10466 | 4k      34      598     826 |                                  |
|                                                         | 10k   26    4633   8701  | 10k     32      545    853  |                                  |
|  lmbench_musl lat_ctx -P 1 -s 32 2 4 8 16 24 32 64 96   |         2 24.37          |           2 8.96            |                                  |
|                                                         |         4 33.32          |           4 6.50            |                                  |
|                                                         |         8 35.36          |           8 5.46            |                                  |
|                                                         |         16 52.97         |           16 9.68           |                                  |
|                                                         |         24 55.47         |          24 15.19           |                                  |
|                                                         |         32 59.54         |          32 19.43           |                                  |
|                                                         |         64 68.39         |          64 22.34           |                                  |
|                                                         |         96 61.64         |          96 22.46           |                                  |





# OVERHEAD

overhead包含atomicusize的set时间

## SYS_READ 

- 4k buffer 
- 去掉trap->1000-1200us
  - 内核堆区创建buffer：6us
  - invoke至pm：2us
    - pm::sys_read->file handle.read->filehandle.readraw->inode.read->invoke2redoxfs->30us
    - redox_fs_file_read->1104us
      - Read_tree->1114us read 5 blocks
        - Read 4k->190us
  - Copy2user:42us
- 



- Kernel:io.rs
- pm::sys_read->file handle.read->filehandle.readraw





- 

# pofiling

## null syscall

- `Redox` on board, multi core disabled:,260 nano sec in kernel (syscall/mod.rs)
- `safeos` on board, 320 nano sec in kernel, 950 nano sec of entire rust code









## redox profile 

```
let mtime = SystemTime::now().duration_since(UNIX_EPOCH).unwrap();
```







## 
