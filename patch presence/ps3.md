# work flow

## original diff backup

```c
    if (header->size == 0 || is_flag_invalid(header->flag)) {
        return true;  
    }
```



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



## About mem addr in sigs generated

### mem addr

- 当patch修改的代码包含全局变量时，会出现内存地址

- 例子：CVE-2023-38409

  - diff:

  ```
  @@ -846,10 +846,11 @@ static int set_con2fb_map(int unit, int newidx, int user)
   		if (err)
   			return err;
   
  -		con2fb_map[unit] = newidx;
   		fbcon_add_cursor_work(info);
   	}
   
  +	con2fb_map[unit] = newidx;
  +
  ```

  - sigs:

  ```
      patch_effect:{
                      Store: 18446744071617626944 + SR(72) = SR(64)
                  }
  ```

- 这里，SR(72) 和 SR(64) 分别代表函数入参unit和newidx。18446744071617626944为内核数据段地址，对应con2fb_map，为全局变量，在set_con2fb_map函数所在文件中声明：`static signed char con2fb_map[MAX_NR_CONSOLES];`

### no mem addr

- 以ko的漏洞为例

  - diff:

  ```
  @@ -19,0 +20 @@ bool check_conditions(struct custom_header *header) {
  +    header->data=0x1000;
  ```

  - 增加了这一行，对入参结构体指针的某一个成员赋值。
  - 汇编： `movl   $0x1000,0x8(%rdi)`，将0x1000赋值给rdi寄存器+8偏移，也就是data字段。
  - sig：Store: 8 + SR(72) = 4096，由于赋值操作只涉及寄存器(符号化为SR (72)),因此该签名是鲁棒的。

### 如何解决？

- 试图解析内存地址的更多语义信息，如通过内存地址锁定对应的symbol