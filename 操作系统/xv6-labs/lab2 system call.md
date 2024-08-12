## 写在最前
这篇笔记主要记录一下lab2 system call遇到的困难和一些思考，以及在做这个实验要求要看的xv6源码的整理。



## Sysinfo (moderate)

- 实验目的

在这个小实验中，需要实现一个系统调用 `sysinfo`, 返回系统运行时的一些信息，例如进程的数目，物理内存的多少等。我们需要定义一个`sysinfo`的结构体，这个结构体中有一些系统运行时的信息。



- 实验流程

1. 把 $U/_sysinfotest 添加到Makefile中的 UPROGS。
2. 为用户进程调用该系统调用提供一个接口： user/user.h 中。
3. 在 usys.pl文件中添加一行 `entry("sysinfo")`
```c
// user/user.h
int sysinfo(struct sysinfo *);
// usys.pl
entry("sysinfo")
```
> 在 usys.pl文件中有对应的一行 entry("sysinfo")，会直接生成相应的汇编代码。生成的汇编代码中指明了系统调用号，还有一条 ecall 指令陷入系统内核。
> 
```
sub entry {
    my $name = shift;
    print ".global $name\n";
    print "${name}:\n";
    print " li a7, SYS_${name}\n";
    print " ecall\n";
    print " ret\n";
}
```

4. 在kernel/syscall.h中新定义相应的系统调用号。
5. 在kernel/sysproc.c中实现 `sys_sysinfo` 函数
```c
// 由于已经进入内核，相应的参数存在寄存器中。
uint64 sys_sysinfo(void);
```
 >   实际上，用户程序执行完 ecall 指令后，会执行 syscall 函数(关于 ecall 指令的具体细节会在下一次trapoline章节中详细讲到)。

```c
void syscall(void){
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if (num > 0 && num < NELEM(syscalls) && syscalls[num]){
    p->trapframe->a0 = syscalls[num]();

    if (((1 << num) & (p->mask)) > 0){
      printf("%d: syscall %s -> %d \n", p->pid, table[num], p->trapframe->a0);
    }
  }
  else
  {
    printf("%d %s: unknown sys call %d\n",
           p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```
syscall函数又会调用 `syscalls[num]`。
```c
static uint64 (*syscalls[])(void) = {
    [SYS_fork] sys_fork,       // SYS_fork是宏，展开后是数字
    [SYS_exit] sys_exit,
    .....
    [SYS_sysinfo] sys_sysinfo,
};

```
> 这几行代码定义了一个名为 `syscalls`的静态数组，数组的每个元素都是一个指向函数的指针。

6. 实现 `sys_sysinfo`函数
```c
uint64 sys_sysinfo(void){
  // 有一个参数，表示 sysinfo 结构体的地址
  uint64 addr;
  if (argaddr(0, &addr) < 0){
    return -1;
  }
  // freemem    还剩下多少个字节(物理内存)
  // nproc      number of processes whose state is not UNUSED
  uint64 np;
  uint64 fm;
  np = cal_proc_unused();
  fm = cal_freemem();
  struct proc *p = myproc();
  struct sysinfo tmp;
  tmp.freemem = fm;
  tmp.nproc = np;
  if (copyout(p->pagetable, addr, (char *)(&tmp), sizeof(tmp)) < 0){
    return -1;
  }
  return 0;
}

```

- 内核中物理内存的组织 (kalloc.c文件)
> xv6 将物理内存组织成一个链表。
> kmem是指向空闲物理页的链表。
```c
struct run{
  struct run *next;
};

struct
{
  struct spinlock lock;
  struct run *freelist;
} kmem;
```

分配物理页和回收物理页
```c
void *
kalloc(void){
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if (r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if (r)
    memset((char *)r, 5, PGSIZE); // fill with junk
  return (void *)r;
}


void kfree(void *pa){
  struct run *r;

  if (((uint64)pa % PGSIZE) != 0 || (char *)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run *)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}
```

根据上面的提示，就可以写出计算剩余物理内存的函数了。
```c
uint64 cal_freemem(){
  struct run *r;
  uint64 mem = 0;
  // acquire(&kmem.lock);
  r = kmem.freelist;
  while (r)
  {
    r = r->next;
    mem++;
  }
  // release(&kmem.lock);
  return mem * PGSIZE;
}

```





