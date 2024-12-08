- Lmbech_musl:每次开机重复一个特定测试10次，算出mean和std_dev.重启，进行下一个测试
- lmbench_homemade:每次开机测试一个lmbench，重启测试下一个。

# homemade final-和redox比

## bw bigger better

|     Test/mean+std_dev     |                         Description                          |          Redox           |        Safeos        |
| :-----------------------: | :----------------------------------------------------------: | :----------------------: | :------------------: |
|       null syscall        |                       get_pid syscall                        |       0.660 0.000        |     0.709,0.916      |
|   bw_file_rd open2close   |   Open a 4KB-sized file, read its contents, then close it.   |  3932.256 118.646 KB/s   |     10762  65.92     |
|    bw_file_rd io_only     |           Bandwidth for reading a 4KB-sized file.            |  12632.959 510.939 KB/s  |     22855.6 2.9      |
| bw_mmap_rd 512 mmap_only  | Bandwidth for map, read every mapped byte, then unmap a 512KB file |      936.048 30.071      |  60469.683 314.060   |
| bw_mmap_rd 512 open2close | Add file opening and closing operations compared to the previous test. |    268.395 0.571 MB/s    |    389.331 0.012     |
|     bw_pipe 1024 1024     | create a pipe between two processes , and moves 1024KB through the pipe in 1024MB chunks |    54.560 0.023 MB/s     |      576.2 1.94      |
|          lat_fs           | times how many files can be created or deleted in one second |      iteration=1000      |                      |
|          Create           |                                                              |   33.922 32.784 31.449   | 43.410 31.583 31.352 |
|          delete           |                    删除文件比redox慢。。                     | 55.893     64.932 62.954 | 58.123 44.975 45.441 |

## lat smaller better

|                Test                |                                                              |     Redox      |    Safeos    |
| :--------------------------------: | :----------------------------------------------------------: | :------------: | :----------: |
|  ./lmbench lat_pipe (mean,stddev)  | uses two processes communicating through a pipe to measure IPC latencies. |  28.070 0.283  | 12.166 0.113 |
| ./lmbench lat_mmap 512 mean stddev |    times how fast a 512KB mapping can be made and unmade.    | 104.470 0.148  | 14.069 0.013 |
|           Lat_pagefault            | Latency for handling page faults for each page of a 512KB file. | 1928.923 2.184 | 3.742 0.006  |

# lmbench musl final-和linux比

## Lat smaller better

|          Mean + std_dev           |                         Description                          |                   Safeos                   |    Linux     |             备注             |
| :-------------------------------: | :----------------------------------------------------------: | :----------------------------------------: | :----------: | :--------------------------: |
|            lat_syscall            |                           Get_ppid                           |              0.87931  0.36129              |    2.6150    |  Linux的所有测试没有std_dev  |
| measures how long to do a syscall |                 read one byte from /dev/zero                 |              **1.1444** 0.003              |    5.3967    |                              |
|                                   |                 Write one byte to /dev/null                  |             **1.06247** 0.003              |    2.9577    |                              |
|                                   |                      fstat an open file                      |             **1.59731** 0.005              |    5.7010    |                              |
|                                   |                         stat a file                          |           **368.140 ** **0.150**           |   15.6598    |                              |
|                                   |                  open and then close a file                  |           **541.527** **0.222**            |   29.7511    |                              |
|         lat_sig  install          |             measures the time to install signals             |             **1.74458** 0.004              |    8.4321    |                              |
|             lat_pipe              | uses two processes communicating through a pipe to measure IPC latencies. |             **32.6425** 8.437              |   71.0699    |                              |
|             lat_proc              | fork:The time it takes to split a process into two (nearly) identical copie and have one exit |             **438.746** 1.603              |   769.6250   |                              |
|                                   | `fork+exec`:The time it takes to create a new process and have that new process run a new program |           **997.683** **3.242**            |   997.8333   |                              |
|                                   | `fork+/bin/sh -c`:The time it takes to create a new process and have that new process run a new program by asking the system shell to find that program and run it |          **1071.903**  **3.329**           |  8282.0000   |     Linux还需要再测一遍      |
|         lat_ctx  -s 32 2          | lat_ctx -s 32 n: measures context switching time for n processes 32 means the process does some work before switch to another |              **5.673** 0.176               |    24.37     | lat_ctx safeos实现可能有问题 |
|         lat_ctx  -s 32 4          |                                                              |              **4.068**  0.239              |    33.32     |                              |
|         lat_ctx  -s 32 8          |                                                              |            **2.995** **0.372**             |    35.36     |                              |
|         lat_ctx  -s 32 16         |                                                              |            **5.466** **0.916**             |    52.97     |                              |
|         lat_ctx  -s 32 24         |                                                              |            **8.871** **1.612**             |    55.47     |                              |
|         lat_ctx  -s 32 32         |                                                              |            **9.815** **2.391**             |    59.54     |                              |
|             lat_mmap              |    times how fast a 512KB mapping can be made and unmade.    |           **746.556** **1.257**            |      59      |                              |
|           lat_pagefault           |      times how fast a page of a file can be faulted in.      |           **193.661** **0.691**            |    2.4193    |                              |
|      lat_fs `bigger better`       | creates a number of small files in the current working directory and then removes the files.  Both the creation and removal of  the files is timed. |  files created per sec \| deleted per sec  |              |                              |
|         0k create\|delete         |                                                              | **57.1** **10.222** \| **68.8** **11.746** | 10696  13463 |                              |
|         1k create\|delete         |                                                              |    **39.1** 4.253 \|  **64** **11.361**    | 6809   10664 |                              |
|         4k create\|delete         |                                                              |   35 **3.493** \|  **61.778** **6.729**    | 7052   10466 |                              |
|        10k create\|delete         |                                                              |    **34.6** 4.079 \|   **60** **6.229**    | 4633   8701  |                              |

## bw bigger better

|    Mean + std_dev     |                         Description                          |         Safeos          |  Linux  |            备注             |
| :-------------------: | :----------------------------------------------------------: | :---------------------: | :-----: | :-------------------------: |
|        bw_pipe        | create a pipe between two processes , and moves 1024KB through the pipe in 1024MB chunks |   **133.785** 47.120    | 179.71  |                             |
|  bw_file_rd  io_only  |               times the read of a 512KB file.                |  **276.285** **0.182**  | 741.44  |                             |
| bw_file_rd open2close | Add file opening and closing operations compared to the previous test. |  **92.105** **0.069**   | 655.24  |                             |
| bw_mmap_rd mmap_only  | creates a memory mapping to a 512KB file and then reads the mapping | **5149.764** **71.672** | 4009.85 | fs相关测试，但性能优于linux |
| bw_mmap_rd open2close | Add file opening and closing operations compared to the previous test. |  **20.997** **0.076**   | 1040.25 |                             |

# Cap 开销

## tcb cap

- enable disk cache and path cache，use RAM disk

|        Baseline:null=0.6x         |          Cap          |         No Cap         | Overhead |
| :-------------------------------: | :-------------------: | :--------------------: | :------: |
| `lmbench_musl lat_proc -P 1 fork` | **209.672** **2.069** | **209.589**  **1.664** |  0.04%   |

## frame cap

|                      Homemade                      |      Cap       |     No Cap      |         Overhead         |
| :------------------------------------------------: | :------------: | :-------------: | :----------------------: |
|                    `Page fault`                    |  3.665 0.006   |   3.275 0.007   |          11.90%          |
|                    mmap latency                    |  14.069 0.013  |   3.191 0.013   |           440%           |
|             bw_mmap_rd 512 open2close              | 915.489 0.032  |  918.164 0.023  | 0.29% 有文件打开关闭操作 |
|             `bw_mmap_rd 512 mmap only`             | 132087 453.188 | 152390 548.239  |          15.4%           |
|                                                    |                |                 |                          |
|                        Musl                        |                |                 |                          |
|       lmbench_musl lat_pagefault -P 1 /test        |      190+      | **0.173** 0.001 |            ??            |
|       lmbench_musl lat_mmap -P 1 512k /test        |      746       | **3.508** 0.000 |            ??            |
| lmbench_musl bw_mmap_rd -P 1 512k open2close /test |       92       |       760       |            ??            |

## Device memory

|                                      |             Cap             |           No Cap           |      |
| :----------------------------------: | :-------------------------: | :------------------------: | :--: |
|               homemade               |                             |                            |      |
|    micro_benchmark(write/read us)    |                             |                            |      |
|               `Write `               | **17127.9978**  **638.674** | **17333.2335** **440.142** |      |
|               `read `                |  **3797.1578** **153.216**  |  **3720.7604** **98.722**  |      |
| `homemade bw_file_rd open2close(KB/s |         561.3 2.10          |         582.1 3.3          |      |
| `homemade bw_file_rd io_only(KB/s)`  |        1244.4 40.998        |       1293.9 35.260        |      |



