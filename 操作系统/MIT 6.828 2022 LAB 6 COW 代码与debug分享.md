# 实验目标
在XV系统实现copy on write，具体要求见[Lab: Copy-on-Write Fork for xv6 (mit.edu)](https://pdos.csail.mit.edu/6.828/2022/labs/cow.html)
# 官方教程
按照官方给出的plan of attack，实现COW需要：
1. 更改uvmcopy()，目的是在fork调用该函数时不分配物理页，仅进行页表拷贝
2. 更改usertrap()，fork后的父or子进程写入页时会触发页错误，更改usertrap()以识别页错误并修复
3. 更改kalloc.c，添加引用计数(reference count)功能
>   Ensure that each physical page is freed when the last PTE reference to it goes away
4. 更改copyout(),在遇到COW页时做出和usertrap中同样的操作

# 实现
根据plan of attack，实现流程如下：
1. 搞定物理页分配的ref count功能
2. 修改uvmcopy，这个较为简单
3. 完成专门应对cowpage的函数，copyout，usertrap两个函数使用同样处理策略，因此抽象出一个函数来应对COWpage问题
4. 在usertrap和copyout中调用cowpage函数
## 准备工作
- PTE中的COW位
```c
#define PTE_C (1L << 8)
```
- 其余函数在defs.h中声明
- 共享计数锁在kinit中初始化
## kalloc.c
并不难，根据官方教程，添加一个计数数组即可。实测是否加锁对cowtest和usertest没有影响，但以防万一还是应该加上锁。
- 添加结构体
```c
struct {
  struct spinlock lock;
  int ref[PHYSTOP/PGSIZE];
} pageref;
```
- 更改kalloc，在分配页时将共享计数数组ref对应位设置为1
```c
if(r)
  {
    kmem.freelist = r->next;
    pageref.ref[((uint64)r)/PGSIZE]=1;
  }
```
- 更改kfree，这个函数是减少共享计数的唯一方式。在计数>1时减少计数，不释放物理页;计数=1时才释放物理页。
```c
// Fill with junk to catch dangling refs.
  acquire(&pageref.lock);
  if(pageref.ref[(uint64)pa/PGSIZE]>1)
  {
    pageref.ref[(uint64)pa/PGSIZE]-=1;
    release(&pageref.lock);
    return;
  }
  release(&pageref.lock);
```
- 添加计数函数和获取计数的函数
```c
int inc(uint64 pa)
{
  if((pa % PGSIZE) != 0 || (char*)pa < end || pa >= PHYSTOP)
    return -1;
  acquire(&pageref.lock);
  pageref.ref[pa/PGSIZE]+=1;
  release(&pageref.lock);
  return 1;
}

int get_ref(uint64 pa)
{
  return pageref.ref[pa/PGSIZE];
}
```
获取共享计数这里没有上锁，因为懒得上锁了。
## uvmcopy
```c
for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    if(*pte&PTE_W)
    {
      *pte |= PTE_C;
      *pte &= ~PTE_W;
    }
    flags = PTE_FLAGS(*pte);
    if(mappages(new, i, PGSIZE, pa, flags) != 0){
      //kfree(mem);
      goto err;
    }
    inc(pa);
  }
  return 0;
```
注意事项：
1. 有写权限的页PTE_C才能被置1，成为COWpage
2. mappages后要增加共享计数
3. 需要同时更改父进程(old),子进程(new)页表项中的flags
## cowpage 处理函数
这个函数检测并处理cowpage情况
1. 检测，检查pte的flag的PTE_C
2. 处理，依据不同共享计数选择仅更改页表项or请求新物理页
```c
int cow(pagetable_t pagetable,uint64 va)
{
  if(va >= MAXVA)//合法检查
    return -1;
  pte_t* pte=walk(pagetable,va,0);
  if(pte == 0 || (*pte & (PTE_V)) == 0 || (*pte & PTE_U) == 0)//合法检查
    return -1;
  va=PGROUNDDOWN(va);//pagealign否则会在mappages多分配一页
					 //另一个解决方案是把mappages中的PGISZE参数改为1
  uint64 pa = PTE2PA(*pte);
  uint flags = PTE_FLAGS(*pte);
  //COW位为0，且没有写权限，非法写入
  if(!(*pte&PTE_C)&&!(*pte&PTE_W))return -1;
  //有写权限orCOW位是0，该页不是COWpage
  if(*pte&PTE_W||(*pte&PTE_C)==0)return 0;
  //COWpage处理，均是COW位是1且没有写权限的情况
  if(get_ref(pa)>1)//分配新页
  {
    char* mem;
    if((mem = kalloc()) == 0)
        panic("no free mem to uncow");
    memmove(mem, (char*)pa, PGSIZE);//复制
    //unmap掉现在的页表项，方便后面重新map。也可以直接更改*pte，
    //那么再mappages后还需调用kfree减少refcount
    uvmunmap(pagetable,PGROUNDDOWN(va),1,1);
    flags &= ~PTE_C;
    flags |=PTE_W;
    if(mappages(pagetable, va, PGSIZE, (uint64)mem, flags) != 0)
    {
      kfree(mem);
      return -1;
    }
    return 0;
  }
  else if(get_ref(pa)==1)//只需进行uncow操作
  {
      *pte &= ~PTE_C;
      *pte |=PTE_W;
    return 0;
  }
  else{return -1;}
}
```
在uvmcopy后，父子进程共享物理页，先进入cow函数的进程所访问的物理页共享计数必定大于1，分配新物理页，更改页表项。后进入cow函数的进程无需再分配物理页，只需要更改PTE_C与PTE_W解除COW状态即可。
## usertrap与copyout
较为简单，只需调用cow即可
```c
copyout:
...
  uint64 n, va0, pa0;
  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    if(cow(pagetable,va0)<0)
      return -1;
    pa0 = walkaddr(pagetable, va0);
...
    

usertrap:
...
else if((which_dev = devintr()) != 0){
    // ok
  }else if(r_scause()==15||r_scause()==13)
  {
    //scause和stval解析见riscv-privileged
    uint64 va = r_stval();
    if(cow(p->pagetable,va)<0)
    {
      //panic("page fault cow failed");
      setkilled(p);
    }
  }
  else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    setkilled(p);
  }
...
```
## 总览
- defs.h
![[Pasted image 20230720145845.png]]
- kalloc.c
![[Pasted image 20230720145955.png]]
- trap.c
![[Pasted image 20230720150027.png]]
- vm.c
cow
![[Pasted image 20230720150135.png]]
uvmcopy
![[Pasted image 20230720150203.png]]
copyout
![[Pasted image 20230720150259.png]]
# DEBUG-COWTEST
## scause=0xc,0xf
usertrap中没有处理页故障导致的，添加正确的页处理程序可以解决。
## kalloc.c中的问题
debug时出现
- usertrap(): unexpected scause 0x0000000000000002 pid=2
            sepc=0x0000000000001000 stval=0x0000000000000000
- panic: init exiting
- simple: sbrk(89478484) failed
大概率是kfree释放，共享计数这方面的错误。我就因为一直出现0x2的报错，全部推翻重写过一遍。其中，如果随便使用exit()退出进程也有概率导致panic: init exiting，建议多用panic，比较安全。
## copyout
```text
file: ereerrorrr:r or:r oeard efaadr: ri failedled
```
测试cowtest出现这样的错误，检查copyout是否有正确处理cowpage
# DEBUG-USERTESTS
通过cowtest只是第一步，usertests会对系统中的各项功能进行全面检查
## 合法性检查
```
usertests starting
test copyin: OK
test copyout: scause 0x000000000000000d
sepc=0x0000000080000b44 stval=0x0000000000000000
panic: kerneltrap
QEMU: Terminated

usertests starting
test copyin: OK
test copyout: panic: walk
```
在cow中缺少对传入参数的合法性检查
```c
if(va >= MAXVA)
    return -1;

if(pte == 0 || (*pte & (PTE_V)) == 0 || (*pte & PTE_U) == 0)
    return -1;
```
## uvmcopy给程序的text段写使能
```
test textwrite: FAILED
SOME TESTS FAILED
```
在uvmcopy中没有判断本页表项是否本就不可写
```c
if(*pte&PTE_W)
{
  *pte |= PTE_C;
  *pte &= ~PTE_W;
}
```
不添加这个判断在老版的6.S081中疑似是可行的，版本更新后会在usertests中报错
# 总结
写到现在为止难度最大的一个lab，用gdb调试收效甚微。第一次写先实现了cow功能，后面再写refcount功能，导致全部乱套，推翻重写。第二次写，只用了一个小时左右就成功过cowtest，又花了一个多小时调试usertests，主要是加上了各种安全性检查等等。
完成这个lab需要对
1. fork-exec的运行逻辑
2. vm.c中的页表匹配，解除匹配等函数
3. 中断流程
4. 物理页分配，回收流程
5. 简单同步互斥
有比较全面的了解，如果对上述部分仍有疑问可以先阅读对应源码再来完成这个lab。
# 参考资料
https://segmentfault.com/a/1190000043577388
[MIT 6.s081 xv6-lab6-cow - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/429821940)
[MIT6.S081 lab6 Copy On Write Fork - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/510110822#%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95)
[Using the GNU Debugger (mit.edu)](https://pdos.csail.mit.edu/6.828/2019/lec/gdb_slides.pdf)
# 本人代码
[akaganeite/MIT-6.1810 at cow (github.com)](https://github.com/akaganeite/MIT-6.1810/tree/cow)

