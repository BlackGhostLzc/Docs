## **实验背景**

大概就是说我们需要实现 UNIX 中的写时复制技术 （copy on write）。在没有写时复制的系统中，调用 `fork()` 时，我们会把父进程的所有的内存都拷贝到子进程的空间，自然，这个耗时是巨大且不可接受的。

并且在实际应用中，`fork()` 时拷贝的大部分内存都时不会被用到的，比如，在 UNIX 中新建一个进程的通常会先调用 `fork()`，然后调用 `exec()`。那么原先复制过来的数据就全部没用了。

在 `fork()` 时，只有一种情况是需要复制内存的。就是写入数据时，如果父进程或子进程尝试往某个地址写入值，那么为了确保写入的这个值不会影响别的进程，我们需要复制这个页帧。

而写时复制就是这样的一个技术，我们会把父进程和子进程共享页帧的 PTE 标为不可写的。那么有任何一个进程尝试往这个页帧写入时，就会产生缺页错误。在 `usertrap()` 函数中，我们可以处理这样的情况，也就是把共享页帧复制一份给尝试写入的进程，这个被复制的页帧会被标记为可写的。

实现写时复制后，可能会有多个进程同时共享一个页帧，那么只有所有的进程都不需要这个共享页帧时，我们才能真正的释放这个页帧。



## **思路**

1. 修改`uvmcopy`函数，使得子进程的所有虚拟页都指向父进程相应虚拟页所对应的物理地址。在子进程和父进程的PTE中，都要把W属性给抹掉。这里可以使用pte中预留的两位。

2. 修改`usertrap`函数，在里面增加页错误的处理逻辑。

3. 物理页面的引用计数。

4. 最后还需要注意`copyout`函数。



  

### uvmcopy函数
`fork`函数中会调用`uvmcopy`函数来复制一份完整的进程空间，修改后使父进程和子进程的pte都指向同一物理页。

> **如果页面是可写的，才把PTE_W属性给抹去，同时还添加PTE_COW属性**，表示这个页面将来被某个进程写时会另外分配物理内存给这个进程。
> 这个函数会调用increase_ref(pa)函数来增加物理页的引用计数;
```c
int uvmcopy(pagetable_t old, pagetable_t new, uint64 sz){
  pte_t *pte;
  uint64 pa, i;
  uint flags;

  for (i = 0; i < sz; i += PGSIZE){
    if ((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if ((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    if (*pte & PTE_W){
      *pte = *pte & ~(PTE_W);
      *pte = *pte | PTE_COW;
    }
    flags = PTE_FLAGS(*pte);
    if (mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){
      goto err;
    }
    increase_ref(pa);
  }

  return 0;

err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```



### usertrap函数

在`usertrap`函数中增加COW处理的逻辑。首先得判断是否是COW的页异常is_cow_fault，然后调用`cow_alloc`函数分配一个物理页面。
```c
else if (r_scause() == 15 || r_scause() == 13){
    uint64 va = r_stval();
    if (is_cow_fault(p->pagetable, va)){
      if (cow_alloc(p->pagetable, va) < 0){
        printf("usertrap: cow_alloc failed\n");
        p->killed = 1;
      }
    }
    else{
      printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
      printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
      p->killed = 1;
    }
  }
```



### is_cow_fault函数

```c
int is_cow_fault(pagetable_t pgtbl, uint64 va){
  va = PGROUNDDOWN(va);
  pte_t *pte = walk(pgtbl, va, 0);
  if (pte == 0)
    return 0;
  if ((*pte & PTE_V) == 0)
    return 0;
  if ((*pte & PTE_U) == 0)
    return 0;
  if (*pte & PTE_COW)
  {
    return 1;
  }
  return 0;
}
```



### cow_alloc函数

在这个函数中，会申请物理内存，再把原来的物理页的内容复制到新的物理页上。
在这里`memmove`函数和`uvmunmap`函数的顺序不能够颠倒。新的物理页的映射就不需要COW标志位了，但需要添加W位，引用计数也要置为1。这里有一定的技巧，在后面会讲到。

```c
int cow_alloc(pagetable_t pgtbl, uint64 va){
  va = PGROUNDDOWN(va);
  pte_t *pte = walk(pgtbl, va, 0);
  int flag = PTE_FLAGS(*pte);
  uint64 pa = PTE2PA(*pte);
  char *mem = kalloc();
  if (mem == 0)
  {
    return -1;
  }
  memmove(mem, (void *)pa, PGSIZE);
  uvmunmap(pgtbl, va, 1, 1);

  flag &= ~(PTE_COW);
  flag |= PTE_W;
  if (mappages(pgtbl, va, PGSIZE, (uint64)mem, flag) < 0)
  {
    kfree(mem);
    return -1;
  }
  return 0;
}

```



### kfree函数

如果该物理页的引用计数为1，那么该页面要被释放掉，否则，该物理页面不应该被释放。在这里要注意锁的使用。
```c
void kfree(void *pa){
  struct run *r;

  if (((uint64)pa % PGSIZE) != 0 || (char *)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  acquire(&ref_lock);
  if (ref_cnt[(uint64)pa / PGSIZE] <= 0)
  {
    panic("Impossible for page references less than 1\n");
  }
  release(&ref_lock);

  decrease_ref((uint64)pa);

  acquire(&ref_lock);
  if (ref_cnt[(uint64)pa / PGSIZE] >= 1)
  {
    release(&ref_lock);
    return;
  }
  // Fill with junk to catch dangling refs.
  release(&ref_lock);
  memset(pa, 1, PGSIZE);

  r = (struct run *)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}
```



### freerange函数

在`freerange`遍历所有的物理页面中，首先把物理页面的引用计数记为1，然后执行`kfree`函数，这样每个物理页面的引用计数会为0，然后所有物理页面也会清除。
记得在`kinit`函数中初始化ref_lock锁。

```c
int ref_cnt[PHYSTOP / PGSIZE];
struct spinlock ref_lock;

void freerange(void *pa_start, void *pa_end){
  char *p;
  p = (char *)PGROUNDUP((uint64)pa_start);
  for (; p + PGSIZE <= (char *)pa_end; p += PGSIZE)
  {
    ref_cnt[(uint64)p / PGSIZE] = 1;
    kfree(p);
  }
}
```

### increase_ref函数和decrease_ref函数
```c
void increase_ref(uint64 pa){
  if (pa >= PHYSTOP)
  {
    panic("increase ref_cnt panic\n");
  }
  acquire(&ref_lock);
  int pn = pa / PGSIZE;
  ref_cnt[pn] += 1;
  release(&ref_lock);
}

void decrease_ref(uint64 pa){
  if (pa >= PHYSTOP)
  {
    panic("increase ref_cnt panic\n");
  }
  acquire(&ref_lock);
  int pn = pa / PGSIZE;
  ref_cnt[pn] -= 1;
  release(&ref_lock);
}
```
想一下，在哪里会调用这两个函数
> `kfree`函数会调用`decrease_ref`函数
> `uvmcopy`函数应该调用`increase_ref`函数
> `kalloc`分配物理页面时把引用计数记为1
> 
