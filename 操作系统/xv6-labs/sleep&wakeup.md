## Sleep & wakeup

`sleep`是当一个进程在等待某一个事件时陷入休眠状态，当这个事件发生时另外一个进程唤醒它。陷入休眠状态可以让这个进程不在等待的时候占用CPU资源

`sleep(chan)`让这个进程睡眠在`chan`这个wait channel上，`wakeup(chan)`将所有睡眠在`chan`上的进程全部唤醒。

lost wake-up problem：当一个进程A即将睡眠时，另外一个进程B发现已经满足了唤醒它的条件进行了唤醒，但是这时还没有进程睡眠在`chan`上，当进程A开始进入睡眠后，进程B可能不会再对进程A进行唤醒，进程A永远进入睡眠状态。

```c
void sleep(void *chan, struct spinlock *lk)
{
  struct proc *p = myproc();
  acquire(&p->lock);  //DOC: sleeplock1
  release(lk);

  // Go to sleep.
  p->chan = chan;
  p->state = SLEEPING;

  sched();

  // Tidy up.
  p->chan = 0;

  // Reacquire original lock.
  release(&p->lock);
  acquire(lk);
}

// Wake up all processes sleeping on chan.
// Must be called without any p->lock.
void wakeup(void *chan)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    if(p != myproc()){
      acquire(&p->lock);
      if(p->state == SLEEPING && p->chan == chan) {
        p->state = RUNNABLE;
       }
      release(&p->lock);
    }
  }
}
```

在`sleep`函数调用前，首先要获取`lk`这把锁，这把锁是用来保护访问的共享资源的。`sleep`最后调用`acquire(lk)`也是为了在进程需要被唤醒时，能够安全地访问之前释放的共享资源。

下面直接看`sleep`和`wakeup`运用的场景。



## Code: Pipes

每一个`pipe`都有一个`struct pipe`，包括了一个`lock`和一个`data`缓冲数组。此外，还有一个读和一个写的信号量。
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

`pipewrite()`往管道中写入n个字节。

首先需要获取`pipe`的锁，这是为了保护`pi`结构体里面的共享资源。
通过`pi->nwrite == pi->nread+PIPESIZE`判断缓冲区是否已经满了，如果已经满了就唤醒睡在`&pi->nread`上的`piperead`进程对缓冲区进行读取，自己睡在`&pi->nwrite`等待唤醒，否则就从user space的`addr`中`copyin`到内核态中的`pi`缓冲区内，完成n字节的读取之后将`piperead`进程唤醒，释放`&pi->lock`。
```c
int
pipewrite(struct pipe *pi, uint64 addr, int n)
{
  int i = 0;
  struct proc *pr = myproc();

  acquire(&pi->lock);
  while(i < n){
    if(pi->readopen == 0 || pr->killed){
      release(&pi->lock);
      return -1;
    }
    if(pi->nwrite == pi->nread + PIPESIZE){ //DOC: pipewrite-full
      wakeup(&pi->nread);
      sleep(&pi->nwrite, &pi->lock);
    } else {
      char ch;
      if(copyin(pr->pagetable, &ch, addr + i, 1) == -1)
        break;
      pi->data[pi->nwrite++ % PIPESIZE] = ch;
      i++;
    }
  }
  wakeup(&pi->nread);
  release(&pi->lock);

  return i;
}
```

`piperead`从管道中读取n个字节。

也要先获取`pi->lock`，判断当前缓冲区内是不是空的，如果是空的就进入睡眠，等待`pipewrite`进行写入并唤醒，否则循环读取n字节缓冲区数据，将缓冲区的数据`copyout`到用户空间的`addr`地址中，待n字节数据全部读取完成之后将`pipewrite`唤醒。
```c
int
piperead(struct pipe *pi, uint64 addr, int n)
{
  int i;
  struct proc *pr = myproc();
  char ch;

  acquire(&pi->lock);
  while(pi->nread == pi->nwrite && pi->writeopen){  //DOC: pipe-empty
    if(pr->killed){
      release(&pi->lock);
      return -1;
    }
    sleep(&pi->nread, &pi->lock); //DOC: piperead-sleep
  }
  for(i = 0; i < n; i++){  //DOC: piperead-copy
    if(pi->nread == pi->nwrite)
      break;
    ch = pi->data[pi->nread++ % PIPESIZE];
    if(copyout(pr->pagetable, addr + i, &ch, 1) == -1)
      break;
  }
  wakeup(&pi->nwrite);  //DOC: piperead-wakeup
  release(&pi->lock);
  return i;
}
```


## Code: wait & exit

### wait代码实现
`wait`中是一个无限循环，每个循环中先对所有的进程循环查找自己的子进程，当发现有子进程并且子进程的状态为`ZOMBIE`时，将子进程的退出状态`np->xstate` `copyout`到`wait`传入的用户空间的`addr`中，然后释放掉子进程占用的所有的内存空间，返回子进程的pid。如果没有发现任何`ZOMBIE`子进程，睡眠在`p`上以等待子进程`exit`时唤醒`p`。
>exit函数会调用wakeup(p->parent)唤醒父进程。
>

**注意**：
`wait()`先要获取调用进程的`p->lock`作为`sleep`的condition lock，然后在发现`ZOMBIE`子进程后获取子进程的`np->lock`，因此xv6中必须遵守先获取父进程的锁才能获取子进程的锁这一个规则。因此在循环查找`np->parent == p`时，不能先获取`np->lock`，因为`np`很有可能是自己的父进程，这样就违背了先获取父进程锁再获取子进程锁这个规则，可能造成死锁。

```c
int
wait(uint64 addr)
{
  struct proc *np;
  int havekids, pid;
  struct proc *p = myproc();

  acquire(&wait_lock);

  for(;;){
    // Scan through table looking for exited children.
    havekids = 0;
    for(np = proc; np < &proc[NPROC]; np++){
      if(np->parent == p){
        // make sure the child isn't still in exit() or swtch().
        acquire(&np->lock);

        havekids = 1;
        if(np->state == ZOMBIE){
          // Found one.
          pid = np->pid;
          if(addr != 0 && copyout(p->pagetable, addr, (char *)&np->xstate,
                                  sizeof(np->xstate)) < 0) {
            release(&np->lock);
            release(&wait_lock);
            return -1;
          }
          freeproc(np);
          release(&np->lock);
          release(&wait_lock);
          return pid;
        }
        release(&np->lock);
      }
    }

    // No point waiting if we don't have any children.
    if(!havekids || p->killed){
      release(&wait_lock);
      return -1;
    }
    
    // Wait for a child to exit.
    sleep(p, &wait_lock);  //DOC: wait-sleep
  }
}
```

### exit代码实现
`exit`关闭所有打开的文件，将自己的子进程reparent给`init`进程，因为`init`进程永远在调用`wait`，这样就可以让自己的子进程在`exit`后由`init`进行`freeproc`等后续的操作。然后获取进程锁，设置退出状态和当前状态为`ZOMBIE`，进入`scheduler`中并且不再返回。

注意：在将`p->state`设置为`ZOMBIE`之后才能释放掉`wait_lock`，否则`wait()`的进程被唤醒之后发现了`ZOMBIE`进程之后直接将其释放，此时`ZOMBIE`进程还没运行完毕。

`exit`是让自己的程序进行退出，`kill`是让一个程序强制要求另一个程序退出。`kill`不能立刻终结另一个进程，因为另一个进程可能在执行敏感命令，因此kill仅仅设置了`p->killed`为1，且如果该进程在睡眠状态则将其唤醒。当被`kill`的进程进入`usertrap`之后，将会查看`p->killed`是否为1，如果为1则将调用`exit`。

```c
void
exit(int status)
{
  struct proc *p = myproc();

  if(p == initproc)
    panic("init exiting");

  // Close all open files.
  for(int fd = 0; fd < NOFILE; fd++){
    if(p->ofile[fd]){
      struct file *f = p->ofile[fd];
      fileclose(f);
      p->ofile[fd] = 0;
    }
  }

  begin_op();
  iput(p->cwd);
  end_op();
  p->cwd = 0;

  acquire(&wait_lock);

  // Give any children to init.
  reparent(p);

  // Parent might be sleeping in wait().
  wakeup(p->parent);
  
  acquire(&p->lock);

  p->xstate = status;
  p->state = ZOMBIE;

  release(&wait_lock);

  // Jump into the scheduler, never to return.
  sched();
  panic("zombie exit");
}
```





