---
layout:  post
title:   "CentOS生成core dump文件调试"
date:   2024-12-04 8:51:00
author:  'Xiangxiao'
header-img: 'img/post-bg-2015.jpg'
catalog:   false
tags:
- c
- core dump文件
- CentOS
- postgres内核
---

# 1 如何生成coredump

核心转储基本上是程序崩溃时内存的快照。

在很多时候会遇到程序异常关闭的情况，当操作系统由于程序中的故障而终止进程时，进程会转储核心。发生这种情况的最典型原因是程序访问了无效的指针值。例如，当我们试图取消引用NULL指针时，在退出之前，我们会收到一个SEGV信号。作为这个过程的一部分，操作系统试图将我们的信息写入一个文件，以便稍后进行事后分析。

我们可以使用核心转储来诊断和调试我们的计算机程序，方法是将核心文件与可执行文件一起加载到调试器中（用于符号和其他调试信息）。由于核心转储可以占用大量的磁盘空间，因此它们的大小有一个可配置的限制

所以可以使用如下命令查看对于core文件的大小限制

```shell
ulimit -c
```

然后可以使用如下命令设置为无限制

```shell
ulimit -S -c unlimited
```

然后修改/etc/sysctl.conf文件，增加如下：

```
kernel.core_pattern = ./core_%e_%p
kernel.core_uses_pid = 0
```

这两个参数分别对应：
- core文件相关的配置文件存放路径。
- core_uses_pid可以控制core文件的文件名中是否添加pid作为扩展名。文件内容为1，表示添加pid作为扩展名，为0则表示不添加。

修改完之后可以使用如下命令设置
```shell
sysctl -p /etc/sysctl.conf
sysctl 0
```

最后就是可以查看当前的设置：
```shell
cat /proc/sys/kernel/core_pattern
cat /proc/sys/kernel/core_uses_pid
```

在设置中，%e表示可执行文件名，%p表示pid。
当然也可以增加其他设置，具体如下：

| VALUE  |  Function  |
| ------ | ---------- |
|  %	 |‘%’ is dropped|
|  %%    |	output one ‘%’|
|  %p    |	Includes PID|
|  %P    |	Includes Global PID|
|  %i    |	Shows Thread ID|
|  %I    |	Global Thread ID|
|  %u    |	User ID|
|  %g    |	Group ID|
|  %d    |	Dump mode|
|  %s    |	Signal number|
|  %t    |	UNIX time of dump|
|  %h    |	Hostname|
|  %e    |	Executable file|
|  %E    |	Executable file path|

# 2 测试例子

编写一个c文件，如下：
```c
#include <stdio.h>

int myfunc(int a,int b)
        {int *p= NULL;
        *p = 1234;
        return 0;
        }

int main()
{
        int a = 10;
        int c = 100;
        myfunc(c,a);
        return 0;
}
```
然后使用gcc进行编译
```shell
gcc -g test.c -o test
```
完成之后执行可执行文件test

得到core文件，存储的目录在core_pattern设置的目录下

然后基于这个core文件使用gdb开始调试，调试命令如下：
```shell
gdb test可执行文件 core文件
```
然后在gdb中可以使用`bt`查看函数调用栈


# 3 如何在pg中使用core调试

首先是将生成的coredump文件放到安装目录下的bin下面
```shell
cp coredump文件  数据库安装目录/bin
```

然后将postgres可执行文件作为调试

```shell
gdb postgres coredump文件
```

# 4 参考文件链接
centos7开启coredump——————>
https://www.cnblogs.com/chacebai/p/17538786.html
CentOS开启coredump转储并生成core文件的配置——————>
https://blog.csdn.net/ProgramVAE/article/details/105921381