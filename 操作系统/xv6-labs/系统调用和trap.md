## 简单介绍
这篇笔记主要帮助回顾xv6是如何经过trap陷入到内核去的，介绍相关的代码的重要的细节。

- 重要的寄存器
1. `SATP`（Supervisor Address Translation and Protection）：它包含了指向page table的物理内存地址
2. `STVEC`（Supervisor Trap Vector Base Address Register）：它指向了内核中处理trap的指令的起始地址。
3. `SEPC`（Supervisor Exception Program Counter）：在trap的过程中保存程序计数器的值。
4. `SSRATCH`（Supervisor Scratch Register）寄存器：在ecall指令前指向的是进程trapframe的地址
> trapframe在虚拟地址空间中位于trampoline的上一页(地址低于trampoline)。

- trap 调用的函数
> uservec -> usertrap -> usertrapret -> userret.
> 



## ecall指令做了什么？

1. `ecall`将代码从user mode改到supervisor mode。
2. `ecall`将程序计数器的值保存在了SEPC寄存器。
3. `ecall`会跳转到STVEC寄存器指向的指令
> ecall并没有切换到内核的页表。也没有找到一个内核栈(C代码的执行需要栈)
> 



## uservec

执行完`ecall`指令，会跳转到`STVEC`所指向的地址处，这是位于`trampoline`页面的第一条地址。
> 内核已经事先设置好了STVEC寄存器的内容为0x3ffffff000。
> 

1. 首先交换`a0`和`SSRATCH`寄存器的值
> 腾出一个通用寄存器, a0指向trapframe,注意此时还没有切换页表。
> 
```asm
csrrw a0, sscratch, a0
```

2. 将各种寄存器的值保存在trapframe中。
```asm
sd ra, 40(a0)
sd sp, 48(a0)
.....
```
3. 在trapframe中获取一些相关的信息，比如说`usertrap`的地址， kernel stack， current hartid
4. 切换到内核的页表
```asm
ld t1, 0(a0)
csrw satp, t1
```
5. 然后跳转到`usertrap`函数中。



##  usertrap
这时候已经在内核的页表中了。
1. 更改`stvec`寄存器的值,获取当前进程
> trap如果从内核空间发起，将会是一个非常不同的处理流程
```c
w_stvec((uint64)kernelvec);

struct proc *p = myproc();
```
> myproc函数实际上会查找一个根据当前CPU核的编号索引的数组，CPU核的编号是hartid，如果你还记得，我们之前在uservec函数中将它存在了tp寄存器。这是myproc函数找出当前运行进程的方法。
> 

2. 保存pc
> 中途可能切换到另一个进程
```c
// save user program counter.
  p->trapframe->epc = r_sepc();
```

3. 检查trap的原因，并执行相应的操作。例如如果是系统调用就调用syscall函数
```c
if(r_scause() == 8){
    // system call

    if(p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
```
> epc还需要+4，是因为在RISC-V中，存储在SEPC寄存器中的程序计数器，是用户程序中触发trap的指令的地址。但是当我们恢复用户程序时，我们希望在下一条指令恢复，也就是ecall之后的一条指令。
> 

4. 调用`usertrapret`函数


## usertrapret
返回到用户空间之前内核要做的工作.

1. 关闭了中断,stvec指向用户空间的trap处理代码
2. 设置trapframe中的数据，这样下一次从用户空间转换到内核空间时可以用到这些数据。
3. 跳转到函数userret。
> 这个跳转有一点细节
```c
uint64 fn = TRAMPOLINE + (userret - trampoline);
((void (*)(uint64, uint64))fn)(TRAPFRAME, satp);
```
> 首先计算出 userret 在trampoline页面的地址，然后再进行一次函数跳转
> 此时**a0寄存器的值为 TRAPFRAME**,**a1寄存器的值为进程的satp**。


## userret
> 这段代码也在trampoline页中,所以切换页表后pc并不会出现错误。
> 

1. 切换page table。
2. 恢复寄存器现场
```asm
# put the saved user a0 in sscratch, so we
# can swap it with our a0 (TRAPFRAME) in the last step.
ld t0, 112(a0)
csrw sscratch, t0

# restore all but a0 from TRAPFRAME
ld ra, 40(a0)
ld sp, 48(a0)
ld gp, 56(a0)
.....
```
此时`sscratch`寄存器的值是系统调用的返回值。
回顾一下，在`syscall`函数中有下面一行,系统调用的返回值覆盖了我们保存在`trapframe`中的`a0`寄存器的值：
```c
// kernel/syscall.c
p->trapframe->a0 = syscalls[num]();
```

3. 交换a0和sscratch的值
```asm
# restore user a0, and save TRAPFRAME in sscratch
csrrw a0, sscratch, a0
```
这样`a0`就是系统调用的返回值了。`sscratch`就指向`trapframe`的地址了。

4. 最后调用`sret`指令
- 程序会切换回user mode
- SEPC寄存器的数值会被拷贝到PC寄存器（程序计数器）
- 重新打开中断

