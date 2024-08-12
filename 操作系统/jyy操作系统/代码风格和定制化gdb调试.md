## 本讲内容

1. 编程中的一些细节
2. 调试工具的正确使用方法

### 软件的热更新DSU

#### 代码实现
```c
#include <stdio.h>
#include <string.h>
#include <sys/mman.h>
#include <stdint.h>
#include <assert.h>

#define STRINGIFY(s) #s
#define TOSTRING(s)  STRINGIFY(s)

void padding() {
  asm volatile(
    ".fill " TOSTRING(PADDING) ", 1, 0x90"
  );
}

__attribute__((noinline)) void foo() {
  printf("In old function %s\n", __func__);
}

__attribute__((noinline)) void foo_new() {
  printf("In new function %s\n", __func__);
}

// 48 b8 (64-bit imm)   movabs $imm,%rax
// ff e0                jmpq   *%rax
const char PATCH[] = "\x48\xb8--------\xff\xe0";

void DSU(void *func, void *func_new) {
  int flag = PROT_WRITE | PROT_READ | PROT_EXEC, rc, np;

  // Grant write permission to the memory
  // We must handle boundary cases
  uintptr_t fn = (uintptr_t)func;
  uintptr_t base = fn & ~0xfff;
  if (fn + sizeof(PATCH) > base + 4096) {
    np = 2;
  } else {
    np = 1;
  }
  printf("np = %d\n", np);

  rc = mprotect((void *)base, np * 4096, flag);
  assert(rc == 0);  // Not expecting a failure
  
  // Patch the first instruction (this is UB in C spec)
  memcpy(func, PATCH, sizeof(PATCH));
  memcpy((char *)func + 2, &func_new, sizeof(func_new));

  // Revoke the write permission
  rc = mprotect((void *)base, np * 4096, PROT_READ | PROT_EXEC);
  assert(rc == 0);  // Not expecting a failure
}

int main() {
  setbuf(stdout, NULL);
  foo();
  DSU(foo, foo_new);  // Dynamic software update
  foo();
}
```
#### 一些编程小技巧
- 什么是 __func__？
> __func__ 是C语言中的一个内置宏，它返回当前函数的名称作为一个字符串常量。它可以用于调试和错误报告，以便在程序出错时能够更容易地确定错误发生在哪个函数中。
相当于：
```c
void my_function() {
#define __func__ "my_func"
    printf("Current function: %s\n", __func__);
#undef __func__ 
}
```
> 使用 __func__ 宏不需要包含任何头文件，因为它是C语言的内置宏，可以直接在代码中使用。
> 

- 使用 assert 断言
> 有利于 bug 的定位

#### 代码讲解
- 为什么要把函数设置成 inline?
内联函数（inline function）是一种编译器提供的优化手段，它的本质是将函数在调用处展开，从而避免了函数调用的开销。也就是说，内联函数不是真正的函数调用，而是将函数的代码嵌入到调用处，类似于宏替换。

- 打一个小补丁
> 我们知道，在调用一个函数的时候，首先 call foo, 把返回地址压栈，并跳转到foo函数处，然后再在foo函数那里给上一个补丁。
```
movabs $imm , %rax
jump *(rax)
```
%rax是 foo_new函数的地址，因为foo_new函数最后也会调用 ret 指令，所以结束后返回到原来的地方。



### 用好工具

- 如何让gdb以更友好的方式帮我们打印相关的信息？
计算机公理3：让你感到不适的 tedious 工作，一定有办法提高效率。

> 用python写一个脚本，增加一个自定义的gdb命令

```python
import gdb
from pathlib import Path

REGS = [
    'rax', 'rbx', 'rcx', 'rdx',
    'rbp', 'rsp', 'rsi', 'rdi',
    'r8', 'r9', 'r10', 'r11',
]

class RegDump(gdb.Command):
    def __init__(self):
        super(RegDump, self).__init__(
            "rdump", gdb.COMMAND_DATA, gdb.COMPLETE_SYMBOL
        )

    def invoke(self, arg, _):
        # 得到变量 ctx 的值
        # 每次输入 rdump 命令会执行 invoke 函数
        ctx = gdb.parse_and_eval(f'ctx')
        for i, r in enumerate(REGS):
            print(
                f'{r.upper():3} = {int(ctx[r]):016x}',
                end=['  ', '\n'][i % 2]
            )
        print('-' * 40)

RegDump()

def get_source_line(address):
    # by GPT-4

    # Find the source code line corresponding to the given address
    symtab_and_line = gdb.find_pc_line(address)

    # Check if the source code line was found
    if symtab_and_line.symtab is not None:
        # Get the source file name and line number
        filename = symtab_and_line.symtab.filename
        line_number = symtab_and_line.line

        return f'{Path(filename).name}:{line_number}'
    else:
        return "Source code line not found"

class ProcDump(gdb.Command):
    def __init__(self):
        super(ProcDump, self).__init__(
            "pdump", gdb.COMMAND_DATA, gdb.COMPLETE_SYMBOL
        )

    def invoke(self, *_):
        n = gdb.parse_and_eval(f'NTASK')
        for i in range(n):
            tsk = gdb.parse_and_eval(f'tasks[{i}]')
            pc = int(tsk['context']['rip'])
            is_current = int(
                gdb.parse_and_eval(f'&tasks[{i}] == current')
            )
            print(
                f'Proc-{i}{" *"[is_current]} ',
                get_source_line(pc)
            )
        
        print('-' * 40)

ProcDump()

```









