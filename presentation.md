Questionss. If this paper uses Rust,.
1. What is the research question? 
  - 设备驱动隔离。设备驱动暴露的漏洞多，驱动如何更小的影响系统其他组件
2. What is the method/solution?
  - 堆隔离，fault isolation
  - 用safeRust，将unsafe封装在lib中
  - interface proxy，重要
  - 状态解藕
  - temporal safety with Rust
3. How do the authors implement them?
   - previous work
4. What can you take away/learn from this paper?
   - auto code generation
5. what feature/property of Rust is used in this task?
   - linear type
   - ownership
6. if this paper uses rust
  1. Do you think Rust is suitable for this task?
    - 是的
  2. What trade off do you observe?
    - 不知道