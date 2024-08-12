### 我们该如何学习 C 标准库?

* ~~直接调试 glibc (像我们上课那样)~~
* 寻找更好的替代品，一定有为嵌入式设备实现的简化 libc。
  > 选择 musl
  > musl是一个轻量级的C标准库，它专注于提供高效、可靠和安全的C语言运行时环境。musl库的设计目标是遵循POSIX标准，同时保持代码的简洁和易于维护。它主要用于嵌入式系统、轻量级容器和其他类似的应用程序中，因为它比其他标准库更小、更快、更安全，并且不包含任何专有代码或许可证限制。在某些情况下，它也可以用作替代glibc的标准库。

- 那怎么编译一个 C 程序用 musl 作为 libc 而不是用 glibc呢 ？
  1.  sudo apt-get install musl-tools
  2.  编写C程序，包含 stdio.h 头文件
  3.  musl-gcc -o hello hello.c

### libc 的基本功能

- 基础数据的体系结构无关抽象
* inttypes.h

- 字符串和数组操作



### 操作系统对象与环境

- 环境变量
```c
#include <stdio.h>

int main() {
  extern char **environ;
  for (char **env = environ; *env; env++) {
    printf("%s\n", *env);
  }
}
```
environ是一个字符指针的指针。

- environ是谁赋值的？

gdb调试
第一条指令的时候 environ 还没有被赋值。
```
0x00007ffff7fc7bfe in _dlstart () from /lib/ld-musl-x86_64.so.1
(gdb) p(char*) environ
$1 = 0x0
```
而在main函数的时候，environ是有正确的值的。
```
//__libc_start_main函数
char **envp = argv + argc + 1;
.....
.....
//__init_libc函数
environ = envp
```




### 动态内存管理


