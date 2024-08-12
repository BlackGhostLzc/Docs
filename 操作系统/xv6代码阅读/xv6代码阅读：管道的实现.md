## **管道的实现**

```c
int pipe(int*);
```

在Linux中，一切皆文件，管道也是一种文件！

我们来看在xv6中，管道究竟是如何被实现的。我们直接来看`sys_pipe`函数。

我们首先获取到传进来的数组`fdarray`，这是数组在进程地址空间中虚拟地址，然后就调用`pipealloc`向向操作系统内核申请两个文件。

> `struct file`是操作系统管理的，而文件描述符则是进程所有的。操作系统有一个`ftable`的`file`数组，管理所有打开的文件。

```c
uint64 sys_pipe(void){
  uint64 fdarray; // user pointer to array of two integers
  struct file *rf, *wf;
  int fd0, fd1;
  struct proc *p = myproc();

  if (argaddr(0, &fdarray) < 0)
    return -1;
  if (pipealloc(&rf, &wf) < 0)
    return -1;
  fd0 = -1;
```

我们接着来看`pipealloc`函数的代码：

这个函数获取两个`file`，并把其中一个文件设置为只读，一个文件设置为只写，将文件类型设置为管道类型。

```c
int pipealloc(struct file **f0, struct file **f1)
{
  struct pipe *pi;

  pi = 0;
  *f0 = *f1 = 0;
  if((*f0 = filealloc()) == 0 || (*f1 = filealloc()) == 0)
    goto bad;
  if((pi = (struct pipe*)kalloc()) == 0)
    goto bad;
  pi->readopen = 1;
  pi->writeopen = 1;
  pi->nwrite = 0;
  pi->nread = 0;
  initlock(&pi->lock, "pipe");
  (*f0)->type = FD_PIPE;
  (*f0)->readable = 1;
  (*f0)->writable = 0;
  (*f0)->pipe = pi;
  (*f1)->type = FD_PIPE;
  (*f1)->readable = 0;
  (*f1)->writable = 1;
  (*f1)->pipe = pi;
  return 0;

 bad:
  if(pi)
    kfree((char*)pi);
  if(*f0)
    fileclose(*f0);
  if(*f1)
    fileclose(*f1);
  return -1;
}

```

然后还做了一件事，申请了一个`pipe`结构体,这个`pipe`结构体中就有一个缓冲区，这样就实现了一个管道了。

```c
struct pipe {
  struct spinlock lock;
  char data[PIPESIZE];
  uint nread;     // number of bytes read
  uint nwrite;    // number of bytes written
  int readopen;   // read fd is still open
  int writeopen;  // write fd is still open
};
```



获取到两个文件后，再调用`fdalloc`函数，进程分配两个文件描述符指向这个`file`结构体。

```c
if ((fd0 = fdalloc(rf)) < 0 || (fd1 = fdalloc(wf)) < 0){
    if (fd0 >= 0)
      p->ofile[fd0] = 0;
    fileclose(rf);
    fileclose(wf);
    return -1;
}
```



获得了两个文件描述符后，就要用`copyout`函数将这两个文件描述符写该进程地址空间地址的`fdarray` 和`fdarray+sizeof(fd0)` 处。

```c
if (copyout(p->pagetable, fdarray, (char *)&fd0, sizeof(fd0)) < 0 ||
      copyout(p->pagetable, fdarray + sizeof(fd0), (char *)&fd1, sizeof(fd1)) < 0)
  {
    p->ofile[fd0] = 0;
    p->ofile[fd1] = 0;
    fileclose(rf);
    fileclose(wf);
    return -1;
  }
  return 0;
```



之后，在对进行管道读取操作时，`sys_read` 系统调用会调用函数`fileread` ，`fileread` 函数会判断文件的类型，如果该文件是管道，那么就会调用`piperead`函数，这个函数就会从`pipe`结构体的缓冲区中读取，前提是`nwrite` 要大于`nread`，否则就会阻塞。

