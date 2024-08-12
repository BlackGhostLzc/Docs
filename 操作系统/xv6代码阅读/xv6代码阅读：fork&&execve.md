## **Fork**

我们来看一下`fork`函数是如何实现的，`fork`是完全把父进程的状态机复制了一份。

首先，先获取当前进行系统调用的线程，然后再`allocproc`申请一个新的进程。在xv6操作系统中，进程的管理是一个`proc`数组。

```c
int fork(void)
{
  int i, pid;
  struct proc *np;
  struct proc *p = myproc();

  // Allocate process.
  if((np = allocproc()) == 0){
    return -1;
  }
```



然后就把页表复制一份给子进程。再设置一下子进程的`trapframe`。

> 回忆一下，trapframe是什么，trapframe就是进行系统调用陷入内核的时候需要保存的一些寄存器现场等相关的Context。

接下来设置`np->trapframe->a0 = 0;`，也就是子进程返回值是0。

```c
  // Copy user memory from parent to child.
  if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
  }
  np->sz = p->sz;
  *(np->trapframe) = *(p->trapframe);

  np->trapframe->a0 = 0;
```



接下来，由于子进程会完整地继承父进程打开的文件，所以这里需要`filedup`。设置子进程当前的工作目录`cwd`，然后就是把子进程的状态设置为`RUNNABLE`，再下次线程切换的时候，子进程就可以执行了。

```c
// increment reference counts on open file descriptors.
  for(i = 0; i < NOFILE; i++)
    if(p->ofile[i])
      np->ofile[i] = filedup(p->ofile[i]);
  np->cwd = idup(p->cwd);

  safestrcpy(np->name, p->name, sizeof(p->name));

  pid = np->pid;

  release(&np->lock);

  acquire(&wait_lock);
  np->parent = p;
  release(&wait_lock);

  acquire(&np->lock);
  np->state = RUNNABLE;
  release(&np->lock);

  return pid;
```

这里的`return pid`和`np->trapframe->a0 = 0;`很tricky，调用`sys_fork`系统调用的只是父进程，所以父进程返回`pid`；而系统调用的返回值通常保存在寄存器`a0`中，所以这样设置实现了`fork`两次返回的效果。



## **Exec**

`exec`函数相比之下还是要复杂得多了，这个函数需要解析elf文件格式，将二进制文件加载到进程的地址空间中。

首先我们打开这个文件，`namei`函数就是通过文件路径找到文件的`inode`。

```c
int exec(char *path, char **argv)
{
  char *s, *last;
  int i, off;
  uint64 argc, sz = 0, sp, ustack[MAXARG], stackbase;
  struct elfhdr elf;
  struct inode *ip;
  struct proghdr ph;
  pagetable_t pagetable = 0, oldpagetable;
  struct proc *p = myproc();

  begin_op();

  if((ip = namei(path)) == 0){
    end_op();
    return -1;
  }
  ilock(ip);
```



然后，我们通过`readi`函数读取elf文件的elf header。

```c
if(readi(ip, 0, (uint64)&elf, 0, sizeof(elf)) != sizeof(elf))
    goto bad;
  if(elf.magic != ELF_MAGIC)
    goto bad;
```



有了elf header，就知道elf文件中程序头表（Program Header Table）的偏移量，程序头表就是一个个程序头(Program Header)的数组的起始位置，然后我们就可以获取它的每一个程序头，这个程序头`proghdr`有一个虚拟地址`ph.vaddr`表示它应该被映射到地址空间的位置，`ph.filesz`表示要映射的大小，`ph.off`表示要映射的文件内容在文件中的偏移量。

```c
  // Load program into memory.
  for(i=0, off=elf.phoff; i<elf.phnum; i++, off+=sizeof(ph)){
    if(readi(ip, 0, (uint64)&ph, off, sizeof(ph)) != sizeof(ph))
      goto bad;
    if(ph.type != ELF_PROG_LOAD)
      continue;
    if(ph.memsz < ph.filesz)
      goto bad;
    if(ph.vaddr + ph.memsz < ph.vaddr)
      goto bad;
    uint64 sz1;
    if((sz1 = uvmalloc(pagetable, sz, ph.vaddr + ph.memsz)) == 0)
      goto bad;
    sz = sz1;
    if((ph.vaddr % PGSIZE) != 0)
      goto bad;
    if(loadseg(pagetable, ph.vaddr, ip, ph.off, ph.filesz) < 0)
      goto bad;
  }
```

再来展示一下`loadseg`的代码。

```c
static int
loadseg(pagetable_t pagetable, uint64 va, struct inode *ip, uint offset, uint sz)
{
  uint i, n;
  uint64 pa;

  for(i = 0; i < sz; i += PGSIZE){
    pa = walkaddr(pagetable, va + i);
    if(pa == 0)
      panic("loadseg: address should exist");
    if(sz - i < PGSIZE)
      n = sz - i;
    else
      n = PGSIZE;
    if(readi(ip, 0, (uint64)pa, offset+i, n) != n)
      return -1;
  }
  
  return 0;
}
```

