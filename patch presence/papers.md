# FIBER

- 人类进行补丁检测的逻辑
  1. 挑选哪些补丁更改(change site)更具代表性
     - CFG上的改变，函数参数变化。。。
  2. 模糊匹配
     - 匹配reference和target的函数的CFG结构，关注涉及change site的基本块
     - 在target中选出符合reference概括出的模式的基本块
  3. 精确匹配，继续筛选模糊匹配选出的candidates
     - 建立汇编指令到源码的映射->人工去看
- signature是什么？`一段指令`
- FIBER基本follow上述逻辑，只是由代码自动分析与检测

# workflow of FIBER

## parse patch

目的是解析补丁文件（通常是 `diff` 格式的文件），并提取相关的改动信息。`parse_patch` 返回的字典格式是：

- **键**：三元组 `(文件路径, 函数名, 函数位置)`。

- 值

  ：包含以下字段的字典：

  - `add`：新增的代码行（按行号和代码内容存储）。
  - `del`：删除的代码行（按行号和代码内容存储）。
  - `func_range`：函数的范围（起始行号和结束行号）。
  - `arg_cnt`：函数的参数数量。（体系结构相关的，函数参数压栈约定不同）

这个字典格式会用于表示补丁文件中各个修改的详细信息。

 ## do pick sig

### generate_line_candidates

- 源码中哪些代码行被更改了
- 代码行行号列表

确保选中的代码行在符号表中存在

### 对选中函数进行语法分析，拿到func_inf

- `func_inf`

  ：一个字典，包含每个修改的函数的解析信息。它的结构为：

  ```
  {
      (文件路径, 函数名, 函数位置): {
          'if': [(起始行号, 结束行号, 条件字符串, 代码块结束行号), ...],
          'for': [(起始行号, 结束行号, 条件字符串, 代码块结束行号), ...],
          'while': [(起始行号, 结束行号, 条件字符串, 代码块结束行号), ...],
          'func': [(起始行号, 结束行号, 函数名, 参数列表), ...],
          'ret': [(起始行号, 结束行号), ...],
          'else': [(起始行号, 结束行号), ...],
          'goto': [(起始行号, 结束行号), ...],
          'decl': [(起始行号, 结束行号), ...],
          'comm': [(起始行号, 结束行号), ...]
      }
  }
  ```

### 筛选合格代码行

- _is_decl_cand删除入变量声明这样的patch添加行

### refine_line_candidates

- 对所有的候选行，change sites进行调整，若不是unique，需要加context。如果包含多个代码行，需要修剪

### rank candidates

- 为候选行打分，选出分最高的

## ext_sig

- 输入：带有补丁的内核二进制，符号表，vmlinux。先前筛选出的change sites。
- 输出：二进制的补丁签名

1. 通过符号表查询函数起始地址，与大小--(`symbol_table.lookup_func_name(func_name)`)

2. 使用addr2line,`addrs = get_addrs_from_lines_aarch64(sys.argv[3], func_name, func_addr, func_addr + func_size, lnos)`,拿到函数中选定的change sites对应的addr2line信息:

   ``` {
   {   
   		line_number1(change sites 1): {address1, address2, ...},
       line_number2: {address3, address4, ...},
       ...
   }
   ```

3. 生成签名

   -  sig = networkx.DiGraph()





1.	
