### Linux 的两面

#### Kernel

* 加载第一个进程 （init 应用程序）
> 相当于在操作系统中放置一个位于初始状态的状态机

* 包含一些进程可以操纵的操作系统对象
* 然后 Linux 变成一个中断(系统调用)处理程序

systemd为什么是进程树的根 (init 进程并不是 systemd)

#### Linux Kernel 系统调用上的发行版和应用生态
* 系统工具 coreutils, binutils, systemd, ....
* 桌面系统 
* 应用程序

### 构建最小的Linux
目标：把Linux内核启动起来，把minimal.S的二进制文件加载，打印HelloWorld，然后就退出。

我们真正的壁垒
1. 怎么样提出问题
2. 怎样回答问出的问题


问题：  我希望用 QEMU 在给定的 Linux 内核完成初始化后，直接执行我自己编写的、静态链接的 init 二进制文件。我应该怎么做？
1. 需要编译一个静态链接的 init 二进制文件。
```
$ gcc -static -o init init.c
```
2. 创建一个 initramfs 文件系统，其中包含您的 init 二进制文件和任何其他必需的文件和目录。
```
initramfs:
# Copy kernel and busybox from the host system
	@mkdir -p build/initramfs/bin
	sudo bash -c "cp /boot/vmlinuz build/ && chmod 666 build/vmlinuz"
	cp init build/initramfs/
	cp $(shell which busybox) build/initramfs/bin/
```
- 什么是 cpio 
> cpio 是一个类似于 tar 的工具，用于创建和提取归档文件。最终，归档文件将输出到名为 initramfs.cpio 的文件中。这个命令通常用于创建一个自定义的 initramfs 文件系统，以便在启动时加载自定义的软件和配置文件。
```
   cd build/initramfs && \
	  find . -print0 \
	  | cpio --null -ov --format=newc \
	  | gzip -9 > ../initramfs.cpio.gz
```

- 什么是 initramfs
> initramfs 是一个临时文件系统，它被加载到内存中，以便在 Linux 内核初始化后提供一个初始根文件系统。这个临时的根文件系统在根文件系统挂载之前执行必要的初始化任务。
```
-initrd build/initramfs.cpio.gz
```

3. 然后，您需要在 QEMU 中将 initramfs 文件系统加载到内存中，并将 init 二进制文件设置为 init 进程。您可以使用以下命令启动 QEMU：
```
run:
# Run QEMU with the installed kernel and generated initramfs
	qemu-system-x86_64 \
	  -serial mon:stdio \
	  -kernel build/vmlinuz \
	  -initrd build/initramfs.cpio.gz \
	  -machine accel=kvm:tcg \
	  -append "console=ttyS0 quiet rdinit=$(INIT)"
```
> rdinit=init 是一个内核命令行参数，用于指定内核启动后应该运行哪个程序作为根文件系统的初始化进程。在 Linux 系统中，init 进程是所有进程的祖先进程，负责启动系统中的各种服务和进程。rdinit=init 参数告诉内核在启动时运行名为 init 的程序作为 init 进程，这通常是指在 initramfs 文件系统中的程序。
> 如果这个init进程中途结束退出了的话，例如把这个init进程换成我们的一个minimal.S(简单的打印后退出)，那么由于init进程结束，系统会panic。init进程有着特殊的作用。

- 什么是busybox
> Busybox是一个开源工具集，集成了许多常用的Unix工具，如ls、cat、grep、find等，可以在嵌入式系统中提供命令行界面的支持。Busybox的目标是提供一个小巧、高效的Unix工具集。
1. busybox sh 就变成一个 shell
2. busybox ls 就执行 ls 命令
* busybox 可以看成Linux所有程序的一个打包




- init 程序
```
#!/bin/busybox sh

# initrd, only busybox and /init
BB=/bin/busybox

# (1) Print something and exit
$BB echo -e "\033[31mHello, OS World\033[0m"

# (2) Run a shell on the init console
$BB sh
```
把 init 程序换成这个脚本后，系统启动没有 kernel panic了。
make run启动后，得到了一个Linux的终端，但输入 ls 命令是没有反应的。
> 回想一下：系统在启动以后，只有 init 和 busybox，系统并不认识 ls 命令。可以 /bin/busybox ls。
- 那应该怎么办？
```
#!/bin/busybox sh

# initrd, only busybox and /init
BB=/bin/busybox

# (1) Print something and exit
$BB echo -e "\033[31mHello, OS World\033[0m"

# (3) Rock'n Roll!
for cmd in $($BB --list); do
  $BB ln -s $BB /bin/$cmd
done

$BB sh

mkdir -p /tmp
mkdir -p /proc && mount -t proc  none /proc
mkdir -p /sys  && mount -t sysfs none /sys
mknod /dev/tty c 4 1
setsid /bin/sh </dev/tty >/dev/tty 2>&1
```
- 在 /bin 目录下创建多个命令的快捷方式
```
for cmd in $($BB --list); do
  $BB ln -s $BB /bin/$cmd
done
```


- 最后回答什么是 systemd, 它为什么是进程树的根。
> 在之后，会执行 /usr/sbin/init 
> 可以看到，这是个快捷方式，指向 /lib/systemd/systemd