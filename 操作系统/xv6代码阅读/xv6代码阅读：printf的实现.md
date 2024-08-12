## **printf函数的实现**

在介绍`printf` 函数之前，我们先要来了解一下C语言里面是如何实现边长参数的。`printf` 正是通过变长参数来实现格式化的打印的。



### **C 可变参数**

声明方式为：

```c
int func (int arg1, ...);
```

其中`...` 表示可变参数列表。

请注意，函数 **func()** 最后一个参数写成省略号，即三个点号（**...**），省略号之前的那个参数是 **int**，代表了要传递的可变参数的总数（这里只是在我们的例子中这样规定，实际上没有任何关系）。为了使用这个功能，您需要使用 **stdarg.h** 头文件，该文件提供了实现可变参数功能的函数和宏。具体步骤如下：

* 定义好函数，最后一个参数为省略号，省略号前面设置自定义参数
* 在函数定义中创建一个`va_list` 类型变量。
* 使用int参数和`va_start()` 宏来初始化`va_list` 变量为一个参数列表。
* 使用`va_arg()` 宏和`va_list` 变量访问参数列表每一项。
* 使用宏`va_end()` 来青绿赋予`va_list` 变量的内存。

常用的宏有：

- `va_start(ap, last_arg)`：初始化可变参数列表。`ap` 是一个 `va_list` 类型的变量，`last_arg` 是最后一个固定参数的名称（也就是可变参数列表之前的参数）。该宏将 `ap` 指向可变参数列表中的第一个参数。
- `va_arg(ap, type)`：获取可变参数列表中的下一个参数。`ap` 是一个 `va_list` 类型的变量，`type` 是下一个参数的类型。该宏返回类型为 `type` 的值，并将 `ap` 指向下一个参数。
- `va_end(ap)`：结束可变参数列表的访问。`ap` 是一个 `va_list` 类型的变量。该宏将 `ap` 置为 `NULL`。

我们看一个小例子：

```c
#include <stdio.h>
#include <stdarg.h>
 
double average(int num,...){
    va_list valist;
    double sum = 0.0;
    int i;
 
    /* 初始化 valist, num 为最后一个固定参数，valist指向了可变参数列表的第一个参数 */
    va_start(valist, num);
 
    /* 访问所有赋给 valist 的参数 */
    for (i = 0; i < num; i++){
       sum += va_arg(valist, int);
    }
    /* 清理为 valist 保留的内存 */
    va_end(valist);
 
    return sum/num;
}
 
int main(){
   printf("Average of 2, 3, 4, 5 = %f\n", average(4, 2,3,4,5));
   printf("Average of 5, 10, 15 = %f\n", average(3, 5,10,15));
}
```





### **printf格式化打印**

有了可边长参数的知识，我们就可以实现一个简单的`printf` 函数了。

发现这个`printf` 函数实现也确实比较容易。

```c
void
printf(char *fmt, ...)
{
  va_list ap;
  int i, c, locking;
  char *s;

  locking = pr.locking;
  if(locking)
    acquire(&pr.lock);

  if (fmt == 0)
    panic("null fmt");

  va_start(ap, fmt);
  for(i = 0; (c = fmt[i] & 0xff) != 0; i++){
    if(c != '%'){
      consputc(c);
      continue;
    }
    c = fmt[++i] & 0xff;
    if(c == 0)
      break;
    switch(c){
    case 'd':
      printint(va_arg(ap, int), 10, 1);
      break;
    case 'x':
      printint(va_arg(ap, int), 16, 1);
      break;
    case 'p':
      printptr(va_arg(ap, uint64));
      break;
    case 's':
      if((s = va_arg(ap, char*)) == 0)
        s = "(null)";
      for(; *s; s++)
        consputc(*s);
      break;
    case '%':
      consputc('%');
      break;
    default:
      // Print unknown % sequence to draw attention.
      consputc('%');
      consputc(c);
      break;
    }
  }

  if(locking)
    release(&pr.lock);
}
```

`char* fmt` 就是最后一个固定参数，也是一个字符串，比如这样`a is %d, b is %d \n` ，这样，我们的目的就是遍历这个`fmt` 的字符数组，找到所有以`%` 开头的，这都是一个潜在的格式化打印的地方。

我们来看`printint` 函数，因为只是一个数字，我们的`consputc` 只能打印字符，所以我们不仅需要把整数转换成字符串，还需要考虑正负号(`sign` 为1表示要打印符号)

```c
static void
printint(int xx, int base, int sign)
{
  char buf[16];
  int i;
  uint x;

  if(sign && (sign = xx < 0))
    x = -xx;
  else
    x = xx;

  i = 0;
  do {
    buf[i++] = digits[x % base];
  } while((x /= base) != 0);

  if(sign)
    buf[i++] = '-';

  while(--i >= 0)
    consputc(buf[i]);
}
```



其实这个`printf` 实现还是很简单，例如没有考虑到`%ld` 这种`%` 后面跟两个字符的格式化标志。