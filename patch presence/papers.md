# paper list

- signature based
  - Hang Zhang and Zhiyun Qian. 2018. Precise and Accurate Patch Presence Test for Binaries.. In USENIX Security Symposium. 887–902.
  -  Peiyuan Sun, Qiben Yan, Haoyi Zhou, and Jianxin Li. 2021. Osprey: A fast and accurate patch presence test framework for binaries. Computer Communications 173 (2021), 95–106
- Similarity based
  - Pdiff: Semantic-based patch presence testing for downstream kernels
  - Precise and Efficient Patch Presence Test for Android Applications against Code Obfuscation
  - Patch based vulnerability matching for binary programs
- back ground
  - 编译器优化对二进制文件的影响
    - The influences of compiler optimization on binary files similarity detection
    -  Identifying Compiler and Optimiza tion Level in Binary Code From Multiple Architectures



# review of patch presence

以下是基于符号执行生成签名进行补丁存在性检测的代表性工作，按技术特点和贡献分类整理：

---

### **1. FIBER**  
- **核心方法**：  
  通过分析源码级补丁，生成**细粒度二进制签名**。签名结合语法（如控制流图拓扑）和语义（符号执行提取的路径约束）特征，用于在目标二进制中匹配补丁引入的代码变更。  
- **技术亮点**：  
  - **签名生成**：选择补丁中最具代表性的代码变更点（如函数调用、条件分支），通过符号执行提取上下文相关指令的语义信息。  
  - **局部化匹配**：仅关注补丁影响的函数区域，避免全函数匹配的开销。  
  - **跨架构支持**：支持不同编译器和架构生成的二进制文件。  
- **效果**：  
  在107个真实漏洞的Android内核镜像测试中，准确率显著优于传统方法。

---

### **2. PS³（Precise Patch Presence Test based on Semantic Symbolic Signature）**  
- **核心方法**：  
  利用**语义级符号仿真**提取签名，通过符号执行模拟补丁前后的代码行为，生成与编译器选项无关的稳定签名，并在语义层面比较参考和目标签名的差异。  
- **技术亮点**：  
  - **稳定性设计**：签名对编译器优化选项（如O0/O3）不敏感，解决了传统方法因编译差异导致的误报问题。  
  - **多级验证**：结合语法（CFG结构）和语义（符号执行约束）特征，提升匹配精度。  
- **效果**：  
  在3,631个CVE-二进制对的测试中，F1得分达0.89，比基线方法提高33%。

---

### **3. Robin**  
- **核心方法**：  
  通过**轻量级符号执行**生成**恶意函数输入（MFI）**，驱动目标函数执行到补丁或漏洞代码路径，捕获执行轨迹的语义特征（如内存访问模式），生成补丁签名。  
- **技术亮点**：  
  - **输入导向分析**：MFI专注于触发补丁相关的代码路径，减少无关路径探索。  
  - **动态行为摘要**：记录执行过程中的内存状态和路径约束，生成高区分度的签名。  
- **效果**：  
  在287个漏洞的测试中，误报率降低94.3%，检测速度达0.47秒/函数。

---

### **4. PDiff**  
- **核心方法**：  
  针对下游内核的代码定制化问题，从补丁影响的**锚点基本块（Anchor Block）**中提取语义摘要（如路径约束、内存访问对），通过符号执行比较目标内核与补丁前后的相似性。  
- **技术亮点**：  
  - **锚点选择策略**：选择补丁影响的核心基本块，减少无关代码干扰。  
  - **抗定制化设计**：通过语义摘要容忍下游厂商的代码修改。  
- **效果**：  
  在715个Linux内核镜像的测试中，漏报率仅4.13%，远低于FIBER的26.35%。

---

### **5. PHunter**  
- **核心方法**：  
  专注于对抗代码混淆的补丁检测，通过符号执行提取**混淆无关的语义特征**（如关键API调用序列、漏洞触发条件），生成鲁棒性签名。  
- **技术亮点**：  
  - **抗混淆能力**：忽略代码结构变化，关注漏洞的核心语义逻辑。  
  - **动态特征提取**：结合符号执行与动态插桩，捕获运行时行为。  
- **应用场景**：  
  主要用于安卓应用的补丁检测，消除第三方库（TPL）的误报。

---

### **技术对比与趋势**  
| **方法** | **签名生成技术**         | **主要优势**               | **适用场景**          |
| -------- | ------------------------ | -------------------------- | --------------------- |
| FIBER    | 源码级变更点映射到二进制 | 高精度局部匹配             | 开源软件补丁检测      |
| PS³      | 符号仿真+语义摘要        | 跨编译器选项稳定性         | 大规模异构二进制分析  |
| Robin    | 轻量级符号执行+动态轨迹  | 低误报率、高效性           | 漏洞验证与误报过滤    |
| PDiff    | 锚点语义摘要             | 抗下游代码定制化           | Linux内核下游版本检测 |
| PHunter  | 抗混淆语义特征           | 对抗代码混淆与第三方库干扰 | 安卓应用安全检测      |

---

### **未来研究方向**  
1. **跨语言支持**：现有工作主要针对C/C++，需扩展至Rust、Go等新兴语言。  
2. **自动化签名生成**：结合AI优化签名选择策略（如PS³的稳定性增强）。  
3. **实时检测**：集成到CI/CD流程，实现补丁状态的动态监控。  
4. **抗优化增强**：应对高级编译器优化（如LTO、PGO）对签名稳定性的影响。  

如需具体实现细节或实验数据，可参考上述文献的公开代码与数据集。

### references

- [论文阅读 -- "Precise and Accurate Patch Presence Test for Binaries" | HotSpurzzZ](https://hotspurzzz.github.io/2022/03/09/Precise and Accurate Patch Presence Test for Binar/)
- [Towards Practical Binary Code Similarity Detection: Vulnerability Verification via Patch Semantic Analysis - A1gerN0n - 博客园](https://www.cnblogs.com/ICE-9/p/17936793)





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
