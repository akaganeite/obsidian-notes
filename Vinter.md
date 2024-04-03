在 `PersistentMemoryTracer` 类中，通过物理内存读写操作和特定CPU指令执行的回调函数，追踪信息被写入到指定的输出文件中。这些输出信息的格式主要由 `print` 语句中指定的格式字符串决定。根据提供的脚本内容，我们可以总结出以下几种写入文件的格式：

### 物理内存写操作 (`pmem_write`)
对于物理内存写操作，输出格式如下：
```
write,<地址>,<大小>,<内容的十六进制表示>,<是否为非暂态写入>,<元数据字符串>
```
- `<地址>`：写操作的目标物理地址。
- `<大小>`：写操作的字节数。
- `<内容的十六进制表示>`：写入的数据的十六进制字符串。
- `<是否为非暂态写入>`：一个布尔值，表示这次写入操作是否为非暂态（non-temporal）写入。
- `<元数据字符串>`：包含额外信息的元数据字符串，如是否在内核代码中执行、调用的内核符号等。

### 物理内存读操作 (`pmem_read`)
对于物理内存读操作，输出格式如下：
```
read,<地址>,<大小>,<读取的内容的十六进制表示>
```
- `<地址>`：读操作的源物理地址。
- `<大小>`：读操作的字节数。
- `<读取的内容的十六进制表示>`：读取的数据的十六进制字符串。

### 特定CPU指令执行后 (`pmem_after_insn_exec`)
对于特定CPU指令执行后，如果启用该回调，输出格式可能如下（虽然在提供的脚本中该回调被禁用了）：
```
insn,<助记符>,<物理地址>,<元数据字符串>
```
- `<助记符>`：执行的CPU指令的助记符。
- `<物理地址>`：指令操作的物理地址（如果适用）。
- `<元数据字符串>`：与 `pmem_write` 中的 `<元数据字符串>` 类似，包含额外的执行上下文信息。

### 超级调用 (`guest_hypercall`)
对于模拟的超级调用，输出格式如下：
```
hypercall,<动作>,<值>
```
- `<动作>`：超级调用指定的动作，如 "checkpoint"。
- `<值>`：与动作相关的值，如检查点的标识符。

这些格式定义了输出文件中的数据结构，便于后续的分析和处理。输出文件为文本格式，每行记录一个事件，不同字段通过逗号分隔，这种格式使得文件易于被人读取和解析。







```python
# 初始化两个集合，用于存储故障后读取操作的地址和大小，以及被读取的内存行号
unpersisted_reads: set[Tuple[int, int]] = set()
unpersisted_reads_lines: set[int] = set()

# 获取故障前和故障后的内存状态对象引用
pre_failure_mem = self.pre_failure_replayer.mem
post_failure_mem = post_failure_replayer.mem

# 获取内存行的粒度大小，通常为64字节
line_granularity = post_failure_mem.line_granularity

# 遍历故障后的读取操作列表
for address, size in post_failure_reads:
    # 计算读取操作涉及的最小和最大内存行号
    min_line_number = address // line_granularity
    max_line_number = (address + size - 1) // line_granularity

    # 遍历涉及的内存行号
    for line_number in range(min_line_number, max_line_number + 1):
        # 检查该行在故障前是否存在未持久化的写入
        if (line := pre_failure_mem.unpersisted_content.get(line_number)) \
                and any(pmem.range_overlap(write.address_range, range(address, address + size)) for write in line.writes):
            # 如果存在，并且故障后的读取操作与该行中任何写入操作的地址范围重叠
            # 则将该读取操作的地址和大小添加到unpersisted_reads集合中
            unpersisted_reads.add((address, size))
            # 将涉及的内存行号添加到unpersisted_reads_lines集合中
            unpersisted_reads_lines.add(line_number)

```

