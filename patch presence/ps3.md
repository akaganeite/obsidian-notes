# work flow

## preprocess

- 解析diff文件，拿到更改的函数名与源文件名

  ```
  parse_result [{
  	'path': 'fs/proc/proc_sysctl.c', 
  	'functions': {'check_conditions': [<Hunk: @@ 80,0 81,3 @@ bool check_conditions(struct custom_header*header) {>]}}]
  ```

  

- 使用gdb deassemble出函数symbol的全部指令地址，使用addr2line得到line-number:addr的映射

  ```
  debug_parser.parse_result:
  {'check_conditions': {79: [18446744071582136608], 81: [18446744071582136612, 18446744071582136616], 86: [18446744071582136619]}}
  ```

- 对比vuln和patch版本的parse_result（从diff文件中提取的信息，包含增加，删除与未更改的行号）。拿到有改变的行号后通过先前建立的映射得到一个地址列表，在本例中diff文件仅有增加行，因此地址列表代表增加行在二进制中对应的地址.add_pattern与remove_pattern为diff文件中是否有新增或删除的if语句

  ```bash
  [
  DiffResult(
  	funcname='check_conditions', 
  	hunks=[HunkDiff(
  		add=[18446744071582136612, 18446744071582136619, 18446744071582136621, 18446744071582136623, 18446744071582136626, 18446744071582136628, 18446744071582136635, 18446744071582136638, 		18446744071582136614], 
  		remove=[], 
  		type='add', 
  		hunk=<Hunk: @@ 80,0 81,3 @@ bool check_conditions(struct custom_header *header) {>, 
  	add_pattern=Patterns(patterns=[Pattern(pattern='Call', name='is_flag_invalid', number=1, wildcard=[False]), Pattern(pattern='If', name=None, number=0, wildcard=None)]), 
  	remove_pattern=None)])
  ]
  ```

- 后续的符号执行将只关注上述add=[]中的地址