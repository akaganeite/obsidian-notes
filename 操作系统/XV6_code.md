# push_off()
disable interrupts
# pop_off()
enable interrupts

# acquire()/release()
acquire a lock loop until lock released

sd s0,8(sp):
store value in s0 to sp+8
ld s0,8(sp)
load value from sp+8 to s0
# regs
ra:return address
s0:callee saved reg

# QA
1. a0 a2-a7 a2 holds 13
2. 没有进行函数调用，内联展开了
3. 64a
4. 38 函数调用后ra为返回地址