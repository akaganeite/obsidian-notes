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

## change site analyzer

- 



