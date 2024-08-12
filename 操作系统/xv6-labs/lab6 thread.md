## Uthread: switching between threads
题目概述：补充代码展现线程调换的机制。代码在文件user/uthread.c 和 user/uthread_switch.S中。主要内容见课程主页。

1. 首先需要在thread结构体中添加一个user_context保存上下文。
```c
struct user_context{
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};

struct thread
{
  char stack[STACK_SIZE]; /* the thread's stack */
  int state;              /* FREE, RUNNING, RUNNABLE */
  struct user_context ctx;
};
```
2. 在thread_create函数中添加初始化，把返回地址指向线程第一条指令的地址，同时还需要把栈指针指向线程自己的栈空间。这样当执行thread_schedule函数进行thread_switch时，会把被调度的新线程保存在thread结构体中的user_context.ra正确地填入到ra寄存器重中，**就好像是被调度的新线程调用了这个thread_switch函数而返回**
```c
void thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++)
  {
    if (t->state == FREE)
      break;
  }

  t->state = RUNNABLE;
  // YOUR CODE HERE
  t->ctx.ra = (uint64)func;
  t->ctx.sp = (uint64)(t->stack + STACK_SIZE - 1);
}
```
3. 随后在thread_schedule函数中添加thread_switch(?,?)函数。
```c
void thread_schedule(void)
{
  struct thread *t, *next_thread;

  /* Find another runnable thread. */
  next_thread = 0;
  t = current_thread + 1;
  for (int i = 0; i < MAX_THREAD; i++)
  {
    if (t >= all_thread + MAX_THREAD)
      t = all_thread;
    if (t->state == RUNNABLE)
    {
      next_thread = t;
      break;
    }
    t = t + 1;
  }

  if (next_thread == 0)
  {
    printf("thread_schedule: no runnable threads\n");
    exit(-1);
  }

  if (current_thread != next_thread)
  { /* switch threads?  */
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
    thread_switch((uint64)&t->ctx, (uint64)&next_thread->ctx);
  }
  else
    next_thread = 0;
}
```



## Using threads

比较容易，为每一个bucket加一把锁，在put的时候上锁,修改完后释放锁即可。

```c
pthread_mutex_t lock_bucket[NBUCKET];

void initial_locks()
{
  for (int i = 0; i < NBUCKET; i++)
  {
    pthread_mutex_init(&lock_bucket[i], NULL);
  }
}

static void put(int key, int value)
{
  int i = key % NBUCKET;

  // is the key already present?
  struct entry *e = 0;

  pthread_mutex_lock(&lock_bucket[i]);

  for (e = table[i]; e != 0; e = e->next)
  {
    if (e->key == key)
      break;
  }
  if (e)
  {
    // update the existing key.
    e->value = value;
  }
  else
  {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }

  pthread_mutex_unlock(&lock_bucket[i]);
}
```



## Barrier

只需要在barrier函数中添加代码即可。
首先需要知道pthread_cond_wait和pthread_cond_broadcast函数的语义。
> pthread_cond_wait: 等待条件变量的信号。当调用线程执行到 pthread_cond_wait 时，它会原子地**释放传入的互斥锁 mutex**，并进入**阻塞状态**，**直到接收到条件变量 cond 的信号**
> pthread_cond_broadcast:发送广播信号给**等待在条件变量 cond 上的所有线程**。这会唤醒所有等待的线程，使它们从 **pthread_cond_wait 函数返回**，并重新**获取 mutex**。
> 

```c
static void
barrier(){
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&bstate.barrier_mutex);
  bstate.nthread++;
  if (bstate.nthread == nthread)
  {
    pthread_cond_broadcast(&bstate.barrier_cond);
    bstate.nthread = 0;
    bstate.round++;
  }
  else
  {
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```
