### 本讲内容
* Shell
* xv6 shell 代码讲解


#### 什么是 shell 

- os = API + 对象
> 人不可能直接使用系统调用来使用操作系统，所以人和操作系统之间隔了一个应用程序，这个应用程序就叫 shell。shell 把内核的 API 和对象做一层封装，来帮助用户管理操作系统对象的一个**应用程序** 。
> 有 graphic shell 和 command line shell。
>


#### shell 编程语言
- 基于文本替换的快速工作流搭建
> 再把 shell 命令构建成一棵树，解释为一组系统调用。
1. 重定向: cmd > file < file 2> /dev/null
2. 顺序结构: cmd1; cmd2, cmd1 && cmd2, cmd1 || cmd2
3. 管道: cmd1 | cmd2
4. 预处理: $(), <()
5. 变量/环境变量、控制流……

- Job control
类比窗口管理器里的 “叉”、“最小化”
jobs, fg, bg, wait
(今天的 GUI 并没有比 CLI 多做太多事)
> 例如用 jobs 查看所有的进程，用 fg 命令使该进程变成前台进程等等。

#### 复刻经典
一个简单地 shell 的实现
推荐阅读网站源代码。


#### 管道的一些细节
- 在gdb中如何调试会产生子进程(多进程)的程序？
1. set follow-fork-mode child
> 这可以在 fork 后直接切到子进程执行流。
> 

```
set follow-fork-mode child
set detach-on-fork off
set follow-exec-mode same
set confirm off
set pagination off
source visualize.py
break _start
run
n 2
define hook-stop
    pdump
end
```
2. info inferiors 命令查看 gdb中的进程
```
Num  Description       Executable
1    process 1234      /path/to/parent
2    process 5678      /path/to/child
```

3. inferior 命令切换进程
> 例如 inferior 1

- 利用好工具，定制化的gdb
```python
import gdb
import subprocess
import re

class ProcDump(gdb.Command):
    def __init__(self):
        super(ProcDump, self).__init__(
            "pdump", gdb.COMMAND_DATA, gdb.COMPLETE_SYMBOL
        )

    def invoke(self, *_):
        print()
    
        for proc in gdb.inferiors():
            pid = proc.pid
            if int(pid) == 0:
                continue
    
            print(f'Process {proc.num} ({pid})', end='')
            if proc is gdb.selected_inferior():
                print('*')
            else:
                print()
    
            for fd_desc in subprocess.check_output(
                ['ls', '-l', f'/proc/{pid}/fd'], encoding='utf-8'
            ).splitlines()[1:]:
                perm, *_, fd, _, fname = fd_desc.split()
    
                if 'rw' in perm: rw = '<->'
                elif 'r' in perm: rw = '<--'
                elif 'w' in perm: rw = '-->'
                if 'pipe:' in fname:
                    pipe_id = re.search(f'[0-9]+', fname).group()
                    print(f'  {fd} {rw} [=== {pipe_id} ===]')
                else:
                    print(f'  {fd} {rw} {fname}')

ProcDump()


```
> 这个类继承自gdb.Command类，这个类是一个GDB命令的基类，用于创建新的GDB命令。在这个类的构造函数中，调用了父类的构造函数，并传入了三个参数，分别是"pdump"，gdb.COMMAND_DATA和gdb.COMPLETE_SYMBOL。其中，"pdump"是命令名称，gdb.COMMAND_DATA表示这是一个处理数据的命令，gdb.COMPLETE_SYMBOL表示这个命令需要在符号表中进行自动补全。
> 在你的代码中，你需要在ProcDump类中实现一个名为invoke的方法，这个方法将会被调用来执行pdump命令的实际功能。
> 


- init.gdb脚本
init.gdb脚本是一个GDB初始化脚本，它会在GDB启动时自动执行。你可以在这个脚本中设置GDB的一些默认行为，例如设置别名、定义宏、加载符号表等。这个脚本可以包含任意数量的GDB命令，它们会按照在脚本中出现的顺序依次执行。
```
gdb +x init.gdb a.out
```



- 什么是 /dev/pts ?
> /dev/pts是一个特殊的文件系统，它提供了一个伪终端（pseudo-terminal）接口，允许用户与计算机进行交互。当用户登录到计算机时，系统会为该用户分配一个伪终端，这个伪终端就会在/dev/pts目录下创建一个对应的设备文件。

echo hello > /dev/pts/8 会发生什么
> 会在相应终端产生输出。使用 tty 命令可以查看当前 shell 终端对应的终端文件是哪一个。

- 说明
由于这里文件描述符的应用我在 MIT 6.S081 中的实验1中实现过类似的，所以在这里不再详细地描述调代码的细节。



- 关于 va_list
  1. va_start sets arg_ptr to the first optional argument in the list of arguments that's passed to the function.
  2. va_arg retrieves a value of type from the location that's given by arg_ptr, and increments arg_ptr to point to the next argument in the list by using the size of type to determine where the next argument starts. 
  3. va_end resets the pointer to NULL

是不是似乎好像会实现 printf 函数了。