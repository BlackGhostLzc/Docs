## **Lock Lab**

## **Memory allocator**

操作系统把物理内存组织成一个链表，每个CPU获取物理内存时都需要在这个链表中进行获取，这样当多个CPU获取时就要通过锁保护好这个链表，而在这个实验中，我们需要为每一个CPU都分配一个物理内存的链表，这样就可以降低`kalloc`实现中锁的竞争。

但存在这样一种情况，某一个CPU中`kalloc`过多导致它自己的链表中空闲的物理页面已经不足了，这就需要它从其他CPU的链表中去“偷”物理页面。

我们来看代码实现，首先是设置好链表并初始化锁，随后调用`freerange`函数。

```c
struct
{
  struct spinlock lock;
  struct run *freelist;
} kmem[NCPU];

void kinit()
{
  /*
  initlock(&kmem.lock, "kmem");
  freerange(end, (void *)PHYSTOP);
  */
  initlock(&kmem[0].lock, "kmem0");
  initlock(&kmem[1].lock, "kmem1");
  initlock(&kmem[2].lock, "kmem2");
  initlock(&kmem[3].lock, "kmem3");
  initlock(&kmem[4].lock, "kmem4");
  initlock(&kmem[5].lock, "kmem5");
  initlock(&kmem[6].lock, "kmem6");
  initlock(&kmem[7].lock, "kmem7");

  freerange(end, (void *)PHYSTOP);
}
```

```c
void freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char *)PGROUNDUP((uint64)pa_start);
  for (; p + PGSIZE <= (char *)pa_end; p += PGSIZE)
  {
    kfree(p);
  }
}
```

我们来看`kfree`函数的实现，这个实现非常简单，就是得到当前的CPU号，再把`pa`插入到free list链表中

```c
void kfree(void *pa)
{
  struct run *r;

  if (((uint64)pa % PGSIZE) != 0 || (char *)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  int cpu_id = cpuid();

  memset(pa, 1, PGSIZE);

  r = (struct run *)pa;

  acquire(&kmem[cpu_id].lock);
  r->next = kmem[cpu_id].freelist;
  kmem[cpu_id].freelist = r;
  release(&kmem[cpu_id].lock);
}

```

`kalloc`函数也比较简单，大致流程就是判断当前CPU的链表中有没有空闲的页，如果没有，就要尝试获取其他CPU的free list链表。

```c
void * kalloc(void){
  struct run *r;

  int cpu_id = cpuid();

  acquire(&kmem[cpu_id].lock);
  r = kmem[cpu_id].freelist;
  if (r)
    kmem[cpu_id].freelist = r->next;
  else{
    for (int i = 0; i < NCPU; i++){
      if (i == cpu_id)
        continue;
      if (kmem[i].freelist == 0){
        continue;
      }
      acquire(&kmem[i].lock);

      r = kmem[i].freelist;
      if (r){
        // 可分配
        kmem[i].freelist = r->next;
        release(&kmem[i].lock);
        break;
      }
      else{
        release(&kmem[i].lock);
        continue;
      }
    }
  }
  release(&kmem[cpu_id].lock);

  if (r)
    memset((char *)r, 5, PGSIZE); // fill with junk
  return (void *)r;
}

```



## **Buffer Cache**

- 这个lab要解决的问题是什么

我们先来看`struct buf* bread(uint dev, uint blockno)`函数，这个函数的任务就是从磁盘中获取`blockno`这个块，并把内容复制到`buf`缓冲区中，并返回这个`buf`指针。

```c
struct buf* bread(uint dev, uint blockno){
  struct buf *b;
  b = bget(dev, blockno);
  if(!b->valid) {
    virtio_disk_rw(b, 0);
    b->valid = 1;
  }
  return b;
}
```

这个函数首先调用`bget`函数从`bcache`缓存中获取一个`buf`，然后再调用`virtio_disk_rw`函数把磁盘上的内容搬运到内存上。

```c
struct {
  struct spinlock lock;
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  struct buf head;
} bcache;

```

那么这就有一个性能瓶颈了，当多进程同时需要或许`bcache`缓存时，都会先对`bcache`上锁`bcache.lock`，锁住整个`bcache`，这样锁的粒度就太大，多个进程不能同时(申请、释放)磁盘缓存。



- 解决思路

我们可以采用哈希表的思想，建立一个从`blockno`到`buf`的哈希表，并为每个桶单独加锁。并且如果一个桶的`buf`已经用完了的时候，它还能从别的桶中偷。原来的`bcache`结构体中的`head`头就需要桶的个数个。

```c
#define NBUCKET 13
#define HASH_FUNCTION(X) (X % NBUCKET)

struct{
  // struct spinlock lock;
  struct buf buf[NBUF];
  struct buf hash_buf[NBUCKET];   // head
  struct spinlock hash_lock[NBUCKET];
} bcache;
```

我们初始化的时候，把所有的`buf`先都挂在第一个桶上。

```c
void binit(void){
  struct buf *b;
  // initlock(&bcache.lock, "bcache");

  for (int i = 0; i < NBUCKET; i++){
    initlock(&bcache.hash_lock[i], "hash_cache");
    bcache.hash_buf[i].next = 0;
  }

  for (int i = 0; i < NBUF; i++){
    b = &bcache.buf[i];
    initsleeplock(&b->lock, "buffer");
    b->last_used = 0;
    b->refcnt = 0;
    b->next = bcache.hash_buf[0].next;
    bcache.hash_buf[0].next = b;
  }
}
```

实验文档里面，还提到了using `ticks` in kernel/trap.c，并使用LRU算法。我没看到每一次发生时钟中断的时候都会把时间戳加1。

```c
void clockintr(){
  acquire(&tickslock);
  ticks++;
  wakeup(&ticks);
  release(&tickslock);
}
```

实验中还要求`begt`函数要select the least-recently used block based on the time-stamps。也就是说每一次`bget`某一个`blockno`，先要在当前的`Hash(blockno)`的桶中寻找是否存在该缓存目标，如果不存在，就要在全局的`buf`中利用LRU算法找到相应的`buf`。这里容易产生死锁问题。

> 这里需要为`struct buf`添加一个字段`last_use`，什么时候设置这个字段呢，当然是释放某个`buf`的时候，当这个`buf`的引用计数为0，就可以把`buf`的`last_use`设置为当前的`ticks`了。



我们先来看`bget`函数。首先就是判断桶中是否已有了缓存，有了的话直接返回就行。

```c
static struct buf *bget(uint dev, uint blockno){
  struct buf *b;

  int key = HASH_FUNCTION(blockno);
  acquire(&bcache.hash_lock[key]);

  // Is the block already cached?
  for (b = bcache.hash_buf[key].next; b; b = b->next)
  {
    if (b->dev == dev && b->blockno == blockno)
    {
      b->refcnt++;
      release(&bcache.hash_lock[key]);
      acquiresleep(&b->lock);
      return b;
    }
  }
  release(&bcache.hash_lock[key]);
```



接下来就要调用函数`find_lru`了，这个函数会根据`last_use`寻找使用哪一个`buf`，以及找到后，它会设置`id_ret`表示这个`buf`当前在哪一个桶的管理下。当这个函数返回时，是必须要获取`bcache.hash_lock[id_ret]`这把锁的，原因也很明显，就是为了避免发生数据不一致。同时`bret`指针就是指向`buf`数组里的某个`struct buf`了。

```c
  int id_ret = -1;
  struct buf *b_ret = find_lru(&id_ret);

  if (b_ret == 0 || id_ret == -1){
    panic("bget: no buffers");
  }

  // 设置 b_ret,如果不存在bcache.hash_buf[id]数组中,要新加一个引用
  struct buf *lru = b_ret->next;
  b_ret->next = lru->next;

  // 可以释放 bcache.hash_buf[id_ret]的锁了，因为已经把这个 buf 给删除了
  release(&bcache.hash_lock[id_ret]);
```



随后，就是把“偷”来的`buf`添加当前的桶里面去。**这里再次判断了是当前`blockno`是否存在于桶的链表中**，就是为了避免这样一种可能，在执行`find_lru`函数的时候，其他别的进程获取了当前桶的锁，进而把这个相同的`blockno`插入进了当前桶的链表中。等到当前进程从`find_lru`返回时，它重新获取了当前桶的锁，它并不知道在它释放锁到获取锁之间发生了什么事情。

```c
 acquire(&bcache.hash_lock[key]);

  for (b = bcache.hash_buf[key].next; b; b = b->next)
  {
    if (b->blockno == blockno && b->dev == dev)
    {
      b->refcnt++;
      release(&bcache.hash_lock[key]);
      acquiresleep(&b->lock);
      return b;
    }
  }

  lru->next = bcache.hash_buf[key].next;
  bcache.hash_buf[key].next = lru;

  lru->refcnt = 1;
  lru->dev = dev;
  lru->blockno = blockno;
  lru->valid = 0;

  // 释放这把锁
  release(&bcache.hash_lock[key]);
  acquiresleep(&lru->lock);
  return lru;
```



来看一下`find_lru`函数的实现。

```c
struct buf *find_lru(int *ret_id){
  struct buf *b;
  struct buf *tmp_buf_ans = 0;

  for (int i = 0; i < NBUCKET; i++){
    acquire(&bcache.hash_lock[i]);
    int find_new = 0;
    for (b = &bcache.hash_buf[i]; b->next; b = b->next)
    {
      if (b->next->refcnt == 0 && (!tmp_buf_ans || tmp_buf_ans->next->last_used > b->next->last_used))
      {
        find_new = 1;
        tmp_buf_ans = b;
      }
    }

    // 需要释放掉之前那把锁，而不是当前这把
    if (find_new){
      if ((*ret_id) != -1)
        release(&bcache.hash_lock[*ret_id]);

      *ret_id = i;
    }
    // 需要释放掉当前这把锁
    else{
      release(&bcache.hash_lock[i]);
    }
  }

  return tmp_buf_ans;
}
```





最后来看一下`brelse`函数。这个函数就比较容易理解，需要注意的一点就是，当`buf->refcnt==0`时，需要把`last_used`设置成`ticks`。

```c
void brelse(struct buf *b){
  if (!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  int key = HASH_FUNCTION(b->blockno);
  acquire(&bcache.hash_lock[key]);

  b->refcnt--;
  if (b->refcnt == 0){
    // no one is waiting for it.
    b->last_used = ticks;
  }

  release(&bcache.hash_lock[key]);
}

```



