---
layout: post
title: "LLDB Command"
date: 2016-06-03 16:36:26 +0800
comments: true
categories: 
---

###设置断点：

* 对所有C函数 functionA 设置断点

    * breakpoint set --name functionnameA --name functionNameB

    * br s -n functionA

    * b functionA
<!--MORE-->
* 对FileA文件的LineNum行设置断点

    * breakpoint set --file FileA --line LineNumber

    * b fileA:LineNumber

* C++ 函数， 对所有C++ 的 foo 函数设置断点

    * breakpoint set --method foo  

    * br s -M foo 

* OC 对所有 aSelector 设置断点

    * breakpoint set --selector aSelector

    * br s -S aSelectro

* 设置 [NSString stringWithFormat] 断点

    * breakpoint set --name "-[NSString stringWithFormat:]" (也是通过name的方式)

    * b -[NSString stringWithFormat:]

* 可以添加 --shlib xxx.lib 来限制只设置这个模块的断点

    * breakpoint set --name functionA --shlib foo.dylib

    * breakpoint set -n functionA -s foo.dylib

* 设置条件断点

    * breakpoint modify -c 'i == 10'

* 命中n次后停止

    * breakpoint modify -i n

* 当开始执行某个queue的时候停止

    * breakpoint modify -q 'queue.name'

* 显不所有断点

    * breakpoint list

    * br l

* 删除断点

    * breakpoint delete 1.1

    * br del 1.1



* 添加命令

    *  (lldb) breakpoint command add 1.1

    * Enter your debugger command(s). Type 'DONE' to end.

    * bt

    * DONE



###WatchPoint

* 当变量值改变时触发断点

    * (lldb) watch set var global

* 添加条件,(当值改为5时才触发)

    * (lldb) watch modify -c 'global == 5'

* 当指针指向的地址内存数据发生变化时触发

    * watchpoint set expression -- myptr

    * wa s e -- myptr

    * 如果没有指定 -x byte_size ，默认观察指针大小的区域

###查看内存

* ( lldb ) memory read --format x --size 4 0xbffff3c0

    * 以16进制格式显示0xbffff3c0开始的内存数据，数据是4个字节为一个单位的

* (lldb)memory read --outfile /tmp/mem.bin --binary 0x1000 0x1200

    * 把0x100 到 0x1200的内存数据输出到文件  

###修改内存

* 在bytes+1的地方写入一个字节的数据：10

    * memory write 'bytes + 1' 10

* 在bytes+1的地方写入大小为4字节的数据

    * memory write -s 4 'byte + 1' 15

* 在bytes开始的8个字节写入1，2，3，4。每个数据2字节

    * memory write -s 2 'byte' 1 2 3 4



###步进

* 代码级别单步执行

    * step

* 代码级别执行洗一条语句

    * next

* 指令级别单步

    * stepi

* 指定级别执行下一条指令

    * nexti

* 跳出当前帧函数

    * finish

 

###其他

* 初始化时会读配置文件

    * ~/.lldbinit

* 可指定别名

  1.  (lldb) command alias bfl breakpoint set -f %1 -l %2 
  1. (lldb) bfl foo.c 12

* 查看当前堆栈

    ﻿* thread backtrace

    ﻿* bt

* 切换当前堆栈帧

    ﻿* frame select 3

* 显示当前栈帧的参数和局部变量信息

    ﻿* frame var -A