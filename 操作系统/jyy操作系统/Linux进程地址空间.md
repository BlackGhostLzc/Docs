## Linux进程的地址空间

有什么工具可以来查看进程的地址空间
1. pmap
2. cat /proc/ [pid] / maps
3. gdb
4. readelf
5. objdump

有的程序刚开始执行就结束了(比如打印一个东西就退出)，如果要查看这个进程的地址空间。那怎么办？
> 使用gdb。
> 使用gdb命令 info inferiors得到进程的pid

* 该命令打印gdb当前管理的inferiors列表，每个inferior都有自己的不同地址空间，inferior与进程对应。

1. 得到进程的pid后,使用命令 !pmap [pid], 在gdb中使用shell命令需要在前面加上 ！。

2. 同样，在gdb中，还可以使用 !cat /proc/ [pid] /maps来查看进程的地址空间。

* 其实pmap就是使用系统中的 /proc/[pid]/这个文件实现的。
怎么证明呢？
> 使用 strace   strace pmap [pid]
> 

```c
#include <stdio.h>

int main()
{
    printf("Hello World\n");
}
```

####  动态链接
gdb调试starti之后，查看进程的地址空间：
```
0000555555554000      4K r---- a.out
0000555555555000      4K r-x-- a.out
0000555555556000      4K r---- a.out
0000555555557000      8K rw--- a.out
00007ffff7fbd000     16K r----   [ anon ]
00007ffff7fc1000      8K r-x--   [ anon ]
00007ffff7fc3000      8K r---- ld-linux-x86-64.so.2
00007ffff7fc5000    168K r-x-- ld-linux-x86-64.so.2
00007ffff7fef000     44K r---- ld-linux-x86-64.so.2
00007ffff7ffb000     16K rw--- ld-linux-x86-64.so.2
00007ffffffdd000    136K rw---   [ stack ]
 total              416K
```
我们还发现，在按下starti后，有一条信息：
```
0x00007ffff7fe32b0 in _start () from /lib64/ld-linux-x86-64.so.2
```
说明动态链接的第一条指令在/lib64/ld-linux-x86-64.so.2中，甚至在地址空间中此时也没有libc这个库。
在状态机在刚刚被初始化的一瞬间，在进程里面还没有printf。
> 动态链接的ELF文件中，有一个INTERP, 就是这里的ld-linux-x86-64.so.2, 需要另外一个程序，才能执行现在这个程序，对于动态链接来说，这就是加载器。

再在main函数上打个断点，continue后再打印一次进程的地址空间
```
0000555555554000      4K r---- a.out
0000555555555000      4K r-x-- a.out
0000555555556000      4K r---- a.out
0000555555557000      4K r---- a.out
0000555555558000      4K rw--- a.out
00007ffff7d7f000     12K rw---   [ anon ]
00007ffff7d82000    160K r---- libc.so.6
00007ffff7daa000   1620K r-x-- libc.so.6
00007ffff7f3f000    352K r---- libc.so.6
00007ffff7f97000     16K r---- libc.so.6
00007ffff7f9b000      8K rw--- libc.so.6
00007ffff7f9d000     52K rw---   [ anon ]
00007ffff7fbb000      8K rw---   [ anon ]
00007ffff7fbd000     16K r----   [ anon ]
00007ffff7fc1000      8K r-x--   [ anon ]
00007ffff7fc3000      8K r---- ld-linux-x86-64.so.2
00007ffff7fc5000    168K r-x-- ld-linux-x86-64.so.2
00007ffff7fef000     44K r---- ld-linux-x86-64.so.2
00007ffff7ffb000      8K r---- ld-linux-x86-64.so.2
00007ffff7ffd000      8K rw--- ld-linux-x86-64.so.2
00007ffffffdd000    136K rw---   [ stack ]
 total             2644K
```
如上，我们发现libc已经有了。加载器也还在，未来可能还需要这个加载器加载其他动态链接库。


#### 其他小细节

```
#include<stdlib.h>

int main{
	time(0);
}
```
在库都加载完成后，用 !cat /proc/14776//maps 查看该进程的地址空间

```
7ffff7fbd000-7ffff7fc1000 r--p 00000000 00:00 0                          [vvar]

7ffff7fc1000-7ffff7fc3000 r-xp 00000000 00:00 0                          [vdso]

```
我们发现这两行， vvar 和 vdso 是什么？

> 不进入内核的系统调用。
> vvar is a memory region that contains kernel variables that are frequently accessed by user-space programs. These variables are read-only and can be accessed directly by the user-space programs without making a system call.
> 例如  
> 当前的时间， 系统页面大小
> vdso is a memory region that contains a small shared library provided by the kernel. This library contains a set of functions that are commonly used by user-space programs and can be excuted directly in user mode, without the need for a system call.
> 


### 进程地址空间的管理
操作系统应该提供一个修改进程地址空间的系统调用
```c
// 映射
void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);
int munmap(void *addr, size_t length);

// 修改映射权限
int mprotect(void *addr, size_t length, int prot);
```

本质：在状态机状态上增加/删除/修改一段可访问的内存

* mmap: 可以用来申请内存 (MAP_ANONYMOUS)，也可以把文件 “搬到” 进程地址空间中

一小段示例代码
```c
#include <unistd.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>

#define GiB * (1024LL * 1024 * 1024)

int main() {
  volatile uint8_t *p = mmap(NULL, 8 GiB, PROT_READ | PROT_WRITE, MAP_ANONYMOUS | MAP_PRIVATE, -1, 0);
  printf("mmap: %lx\n", (uintptr_t)p);
  if ((intptr_t)p == -1) {
    perror("cannot map");
    exit(1);
  }
  *(p + 2 GiB) = 1;
  *(p + 4 GiB) = 2;
  *(p + 7 GiB) = 3;
  printf("Read get: %d\n", *(p + 4 GiB));
  printf("Read get: %d\n", *(p + 6 GiB));
  printf("Read get: %d\n", *(p + 7 GiB));
}
```
疑问：这个程序运行会不会需要很长的时间，因为它分配了那么多的内存？

> 其实一瞬间就完成了。也就是说，在使用mmap的时候，只是在操作系统中标记了这个进程这么多的内存，这个进程中这些内存还并没有开始分配，只是在后面用到了才会产生缺页中断。
> 


### 入侵地址空间

进程 (M,  R 状态机) 在 “无情执行指令机器” 上执行

- 状态机是一个封闭世界
- 但如果允许一个进程对其他进程的地址空间有访问权？

一些入侵地址空间的例子
1. 调试器(gdb)
* gdb 可以任意观测和修改程序的状态
2. Profiler (perf)



#### 入侵进程地址空间 (1): 金山游侠
```c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include <stdint.h>
#include <unistd.h>
#include <stdbool.h>
#include <stdarg.h>
#include <fcntl.h>

#define MAX_WATCH 65536

struct game {
  const char *name;  // Name of the binary
  int pid;           // Process ID

  int memfd;         // Address space of the process
  int bits;          // Search bit-width (16, 32, or 64)
  bool has_watch;    // Watched values
  uintptr_t watch[MAX_WATCH];
};

FILE* popens(const char* fmt, ...);
uint64_t mem_load(char *mem, int off, int bits);

void scan(struct game *g, uint32_t val) {
  uintptr_t start, kb;
  char perm[16];
  FILE *fp = popens("pmap -x $(pidof %s) | tail -n +3", g->name);
  int nmatch = 0;

  while (fscanf(fp, "%lx", &start) == 1) {
    fscanf(fp, "%ld%*ld%*ld%s%*[^\n]s", &kb, perm);
    if (perm[1] != 'w') continue;  // Non-writable areas

    uintptr_t size = kb * 1024;
    char *mem = calloc(size + 16, 1);  // Ignores error handling for brevity
    lseek(g->memfd, start, SEEK_SET);  // Don't do this in production!
    size = read(g->memfd, mem, size);

    printf("Scanning %lx--%lx\n", start, start + size);

    if (!g->has_watch) {
      // First-time search; scan all memory
      for (int off = 0; off < size; off += 2) {
        uint64_t v = mem_load(mem, off, g->bits);
        if (v == val && nmatch < MAX_WATCH) {
          g->watch[nmatch++] = start + off;
        }
      }
    } else {
      // Search in the watched values
      for (int i = 0; i < MAX_WATCH; i++) {
        intptr_t off = g->watch[i] - start;
        if (g->watch[i] && 0 <= off && off < size) {
          uint64_t v = mem_load(mem, off, g->bits);
          if (v == val) nmatch++;
          else g->watch[i] = 0;
        }
      }
    }
    free(mem);
  }
  pclose(fp);

  if (nmatch > 0) {
    g->has_watch = true;
  }
  printf("There are %d match(es).\n", nmatch);
}

void overwrite(struct game *g, uint64_t val) {
  int nwrite = 0;
  for (int i = 0; i < MAX_WATCH; i++)
    if (g->watch[i]) {
      lseek(g->memfd, g->watch[i], SEEK_SET);
      write(g->memfd, &val, g->bits / 8);
      nwrite++;
    }
  printf("%d value(s) written.\n", nwrite);
}

void reset(struct game *g) {
  for (int i = 0; i < MAX_WATCH; i++) {
    g->watch[i] = 0;
  }
  g->has_watch = false;
  printf("Search for %d-bit values in %s.\n", g->bits, g->name);
}

int load_game(struct game *g, const char *name) {
  FILE *pid_fp;
  int ret = 0;

  g->name = name;
  g->bits = 32;
  reset(g);

  pid_fp = popens("pidof %s", g->name);
  if (fscanf(pid_fp, "%d", &g->pid) != 1) {
    fprintf(stderr, "Panic: fail to get pid of \"%s\".\n", g->name);
    ret = -1;
    goto release;
  }

  char buf[64];
  snprintf(buf, sizeof(buf), "/proc/%d/mem", g->pid);
  g->memfd = open(buf, O_RDWR);
  if (g->memfd < 0) {
    perror("/proc/[pid]/mem");
    ret = -1;
    goto release;
  }

release:
  if (pid_fp) pclose(pid_fp);
  return ret;
}

void close_game(struct game *g) {
  if (g->memfd >= 0) {
    close(g->memfd);
  }
}

int main(int argc, char *argv[]) {
  long val;
  char buf[64];
  struct game game;

  if (load_game(&game, argv[1]) < 0) {
    goto release;
  }

  while (!feof(stdin)) {
    printf("(%s %d) ", game.name, game.pid);
    if (scanf("%s", buf) <= 0) goto release;

    switch (buf[0]) {
      case 'q': goto release; break;
      case 'b': scanf("%ld", &val); game.bits = val; reset(&game); break;
      case 's': scanf("%ld", &val); scan(&game, val); break;
      case 'w': scanf("%ld", &val); overwrite(&game, val); break;
      case 'r': reset(&game); break;
    }
  }

release:
  close_game(&game);
  return 0;
}

FILE* popens(const char* fmt, ...) {
  char cmd[128];
  va_list args;
  va_start(args, fmt);
  vsnprintf(cmd, sizeof(cmd), fmt, args);
  va_end(args);
  FILE *ret = popen(cmd, "r");
  assert(ret);
  return ret;
}

uint64_t mem_load(char *mem, int off, int bits) {
  uint64_t val = *(uint64_t *)(&mem[off]);
  switch (bits) {
    case 16: val &= 0xffff; break;
    case 32: val &= 0xffffffff; break;
    case 64: break;
    default: assert(0);
  }
  return val;
}
```

代码导读

- va_list，va_start()、va_arg() 和 va_end() 是什么
> 这可以使C语言实现变长参数。
> > va_list 是一个类型，用于表示可变参数列表。
> > va_start() 宏用于初始化可变参数列表
> > va_arg() 宏用于访问可变参数列表中的下一个参数
> > va_end() 宏用于结束可变参数列表的访问
> > 
- 一段小例子
```c
#include <stdio.h>
#include <stdarg.h>

double average(int count, ...) {
    va_list ap;
    int i;
    double total = 0.0;

    va_start(ap, count); // 初始化可变参数列表

    for (i = 0; i < count; i++) {
        total += va_arg(ap, double); // 获取下一个参数
    }

    va_end(ap); // 结束可变参数列表

    return total / count;
}

int main() {
    double avg = average(5, 1.0, 2.0, 3.0, 4.0, 5.0);
    printf("平均值为：%f\n", avg);
    return 0;
}
```
> popen函数
>
> > popen()会调用fork()产生子进程，然后从子进程中调用/bin/sh -c来执行参数command的指令。参数type可使用“r”代表读取，“w”代表写入。依照此type值，popen()会建立管道连到子进程的标准输出设备或标准输入设备，然后返回一个文件指针。随后进程便可利用此文件指针来读取子进程的输出设备或是写入到子进程的标准输入设备中。
- 也就是说，首先获取游戏进程的名字后，先创建一个子进程执行 pidof [name]的命令，可以获取游戏进程pid。然后fscanf(pid_fp, "%d", &g->pid) != 1读取该进程pid。
接着打开/proc/[pid]/mem这个文件。g->memfd指向这个文件。
```c
snprintf(buf, sizeof(buf), "/proc/%d/mem", g->pid);
  g->memfd = open(buf, O_RDWR);
  if (g->memfd < 0) {
    perror("/proc/[pid]/mem");
    ret = -1;
    goto release;
  }

```
- scan函数
在用pmap得到虚拟地址区域后，就可以把这个区域映射到入侵程序的地址空间中，并得到起始地址。
```c
char *mem = calloc(size + 16, 1);  // Ignores error handling for 
```
- 之后把 /proc/pid/mem的文件偏移设为虚拟地址区域起始处。
* 如果想要在/proc/pid/mem 文件访问进程的虚拟地址 0x12345678，您需要将文件偏移量设置为 0x12345678。
并把这个区域的内存全部写入入侵进程的地址空间中。
```c
lseek(g->memfd, start, SEEK_SET);  // Don't do this in production!
size = read(g->memfd, mem, size);
```
- 然后就可以根据偏移，可以把相应的地址对应起来了。大致意思就是在入侵地址里暴力寻找符合条件的地址，然后根据找的的符合条件的地址，由于偏移是一样的，也就可以把这个地址对应到被入侵进程的相应虚拟地址区域中了。



#### 入侵进程地址空间 (2): 变速齿轮

用修改程序系统调用的手段来欺骗程序对时间的认识，就可以实现游戏的加速和减速。

- 简单的一段C程序
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

int hp = 10000;

__attribute__((noinline)) int hit(int damage) {
  return hp - damage;
}

int main() {
  while (1) {
    hp = hit(rand() % 10);
    printf("hp = %d\n", hp);
    usleep(10000);
    if (hp < 0) {
      printf("Game Over\n");
      break;
    }
  }
}
```

- python 脚本

```python
#!/usr/bin/env python3

from sys import argv
import subprocess

if len(argv) < 2:
    print(f'Usage {argv[0]} [--hp] [--fast] [--slow]')
    exit(1)

def patch(addr, patch):
    pid = int(subprocess.check_output(['pidof', 'game']))
    with open(f'/proc/{pid}/mem', 'wb') as fp:
        fp.seek(addr)
        fp.write(patch)

def name(symbol):
    for line in subprocess.check_output(['nm', 'game']).splitlines():
        tokens = line.decode('utf-8').split()
        if tokens[-2:] == ['T', symbol]:
            return int(tokens[0], base=16)

if '--hp' in argv:
    # hit -> mov $9999999, %eax; ret
    patch(name('hit'), b'\xb8\x7f\x96\x98\x00\xc3')

if '--slow' in argv:
    # usleep (endbr64) -> shl $0x4, %rdi
    patch(name('usleep'), b'\x48\xc1\xe7\x04')

if '--fast' in argv:
    # usleep (endbr64) -> shr $0x4, %rdi
    patch(name('usleep'), b'\x48\xc1\xef\x04')
```













