## 本讲内容：一节真正的 “编程 ”课：如何正确地 (并发) 编程：

- Lock ordering
- 防御性编程
- 运行时检查


## Lock ordering
- 任意时刻系统中的锁都是有限的
- 严格按照固定的顺序获得所有锁 (Lock Ordering)，就可以消灭循环等待
- “在任意时刻获得 “最靠后” 锁的线程总是可以继续执行”
也就是为所有的锁固定一个顺序，某个线程想要获取某种资源就要按顺序去获得这些锁，这样就消除了**循环等待**。

* 哲学家就餐问题的解决
>先获取序号小的那把锁，再获取序号大的那把锁。
```c
sem_t avail[N];

void Tphilosopher(int id) {
  int lhs = (id + N - 1) % N;
  int rhs = id % N;

  // Enforce lock ordering
  if (lhs > rhs) {
    int tmp = lhs;
    lhs = rhs;
    rhs = tmp;
  }

  while (1) {
    P(&avail[lhs]);
    printf("+ %d by T%d\n", lhs, id);
    P(&avail[rhs]);
    printf("+ %d by T%d\n", rhs, id);

    // Eat

    printf("- %d by T%d\n", lhs, id);
    printf("- %d by T%d\n", rhs, id);
    V(&avail[lhs]);
    V(&avail[rhs]);
  }
}

int main() {
  for (int i = 0; i < N; i++) {
    SEM_INIT(&avail[i], 1);
  }
  for (int i = 0; i < N; i++) {
    create(Tphilosopher);
  }
}
```


## Bug 的本质和防御性编程
始终假设自己的代码是错的。
及早检查、及早报告、及早修复。
例如，多用断言`assert`函数
在xv6代码中也有防御性编程的智慧。
1. 避免连续两次`acquire`同一把锁的检查。
```c
// acquire函数
if(holding(lk))
   panic("acquire");
```
2. 避免连续两次`release`同一把锁的检查。
```c
//release函数
if(!holding(lk))
   panic("release");
```


## 自动运行时检查
