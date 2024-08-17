## C语言的运行库
`main`函数并不是程序的第一个执行的函数，在这之前，全局变量的初始化已经结束，`argc`和`argv`参数已经被正确传了进来，堆和栈也初始化完成，系统IO也初始化了，因此可以放心使用`printf`和`malloc`。
>此外有一个函数叫做`atexit`，它接受一个函数指针作为参数，保证程序正常退出时，这个函数指针指向的函数会被调用。
>操作系统在创建进程后，把控制权交给入口函数，由入口函数进行堆栈、全局变量等的初始化，完成初始化后，调用`main`函数。`mian`函数执行完毕后，返回到入口函数，进行清理工作，比如全局变量的析构，堆销毁，关闭IO等。

## 入口函数的glibc实现
这里简单介绍一下静态glibc用于可执行文件时候的例子。
glibc的入口函数为`_start`(ld链接器默认的链接脚本所指定，也可以设置另外的入口)，在`_start`函数之前，装载器会把用户参数和环境变量压入到栈中，栈顶元素是`argc`，接下来是`argv`和环境变量的数组。
这个函数的伪代码如下：
```C
void _start()
{
	%ebp = 0;
	int argc = pop from stack;
	char **argv = top of stack;
	__libc_start_main(main, argc, argv, __libc_csu_init, __libc_csu_fini, edx, top of stack);
}
```

`_start`函数调用一个叫`__lib_start_main`的函数。
```c
int __libc_start_main(
	int (*main) (int, char **, char **),
	int argc,
	char *__unbounded *__unbounded ubp_av,
	__typeof (main) int,
	void (*fini) (void),
	void (*rtld_fini) (void),
	void *__unbounded stack_end
)
```
一共7个参数，main,argc和argv(这里是ubp_av，因为还包括环境变量表)，还有三个函数指针。
* init: main调用前的初始化函数
* fini: main结束后的收尾工作
* rtld_fini: 和动态加载有关的收尾工作
* stack_end: 栈底的地址，即最高的栈地址

接下来代码：
```C
char **ubp_ev = &ubp_av[argc + 1];
__environ = ubp_ev;
__libc_stack_end = stack_end;
```
这段代码设置了全局环境变量指针__environ，还将栈底地址stack_end存储在了全局变量__libc_stack_end中。

接下来代码较繁琐，过滤掉大量信息保留关键函数调用：
```C
__pthread_initialize_minimal();
__cxa_atexit(rtld_fini, NULL, NULL);
__libc_init_first(argc, argv, __environ);
__cxa_atexit(fini, NULL, NULL);
(*init)(argc, argv, __environ);
```
`__cxa_init_first`是glibc内部函数，等同于`atexit`，用于将参数指定的函数在`main`结束后调用（i.e. 注册退出清理函数），让以参数传入`fini`和`rtld_fini`均是用于`main`结束之后调用的。
在`__libc_start_main`末尾：
```C
result = main(argc, argv, __environ);
exit(result);
// !end __libc_start_main
```
接下来看看`exit`实现：
_start -> __libc_start_main -> exit：
```C
void exit(int status)
{
	while (__exit_funcs != NULL)
	{
		...
		__exit_funcs = __exit_funcs->next;
	}
	...
	_exit(status);
}
```
其中`__exit_funcs`是存储由`__cxa_atexit`和`atexit`注册的函数的链表。而`_exit`实现由汇编实现，与平台相关。
```C
_exit:
	movl 4(%esp), %ebx
	movl $__NR_exit, %eax
	int $0x80
	hlt
```
`_exit`作用仅仅是调用了exit这个系统调用。`_exit`调用后，进程就会直接结束。
程序正常结束两种情况：
1. `main`函数返回；
2. 程序中，调用`exit`退出；
2种情况都会调用`exit`，`atexit`注册的退出清理函数交给`exit`来做是没有问题的。


## C语言的运行库
任何一个语言的程序，背后都有一套庞大代码来支撑，以使得程序正常运行，这套代码包括入口函数，及其所依赖的函数所构成的函数集合，各种标准库函数的实现。这样的代码集合称为运行时库（Runtime Library）。而C语言的运行库，称为CRT。

CRT大致包含如下功能：
1. 启动与退出：包括入口函数，及入口函数所依赖的其他函数等；
2. 标准函数：由C语言标准规定的C语言标准库所拥有的函数实现；
3. I/O：I/O功能的封装和实现，参见上一节I/O初始化部分；
4. 堆：堆的封装和实现，参见第十章堆初始化部分；
5. 语言实现：语言中一些特殊功能的实现；
6. 调试：实现调试功能的代码；

## 变长参数
```C
int printf(const char *format, ...);
```
假设`lastarg`是变长参数列表最后一个有名字的参数（如`printf`的`format`），那么在函数内部定义类型`va_list`的变量：
```C
va_list ap;
```
该变量以后会依次指向各个可变参数。`ap`必须用宏`va_start`初始化一次，其中`lastarg`必须是函数的最后一个具有名称的参数。
```C
va_start(ap, lastarg);
// va_start(ap, format);
```
`va_start`之后，就可以用`va_arg`宏来获得下一个不定参数了（假设已经知道其类型为type，type可以是int, char, char *, struct 等等）
```C
type next = va_arg(ap, type);
```
在具有可变长参数的函数结束前，还必须用宏`va_end`来清理现场。

**变长参数是如何实现的**
首先我们得了解一下C函数调用时的参数压栈机制,也就是**cdecl调用惯例**，比如我们有如下这个函数：
```C
int sum(unsigned num, ...);
```
当调用`int n = sum(3, 16, 38, 53)`时，参数在栈上的布局如下，也就是左边的参数先压栈(变长参数的函数应该不会用寄存器传参吧?)
```
top    ...    53    38    16    3     ...      
栈顶                                  栈底
```
根据这个规则，我们就有了va系列宏实现的一个大致思路了，甚至`printf`的实现也不在话下(根据`format`中的%d,%s,%c这些模式，遇到一次调用一次`va_arg`，再根据要求完成类型的转换就行)。

va系列宏的一个简单实现如下：
```C
#define va_list char *
#define va_start(ap, arg) (ap=(va_list)&arg + sizeof(arg))
#define va_arg(ap, t) (*(t*)(ap += sizeof(t)) - sizeof(t))
#define va_end(ap) (ap = (va_list)0)
```

* 变长参数宏
很多时候希望定义宏的时候，也能像`printf`一样可以使用变长参数，即宏的参数可以是任意个，该功能可由编译器的变长参数宏实现。
GCC编译器下，变长参数宏可以使用“##”宏字符串连接操作实现，如：
```C
#define printf(args...) fprintf(stdout, ##args)
```
那么`printf("%d %s", 123, "hello")`会被展开为：
```C
fprintf(stdout, "%d %s", 123, "hello")
```