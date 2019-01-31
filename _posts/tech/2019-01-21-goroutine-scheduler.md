---
layout: post
title: Goroutine的调度
category: 技术
tags: golang
keywords: golang, goroutine
---
## G-P-M模型
- G：Goroutine，代表协程
- P：Processor，对于G来说，P就是运行它的那个Thread，其实是一个处理器
- M：Work Thread，真正的底层Thread

![G-P-M模型](https://raw.githubusercontent.com/Carey6918/Carey6918.github.io/master/assets/images/goscheduler1.png)


1. 启动时生成固定（​GOMAXPROCS​，最大256）个P
2. 在调用go时，生成一个G，放入P的Local队列或者Global队列，
3. 如果还有空闲的P，则生成并绑定一个M
4. M优先找绑定的P的Local队列中的G来执行，如果找不到，则找Global队列，如果还找不到，则从其他P中窃取  
    a. 切换到G的执行栈上并执行G的函数     
    b. 做goexit清理并回到M    
    c. M不保留G的状态，因此G可以跨M调度   
  
## 抢占调度
如果一个G执行太久，会一直占用这个线程，如何让它停止并执行下一个G？

### sysmon（监控线程）
> Go程序启动时，runtime会去启动一个名为sysmon的m(一般称为监控线程)，该m无需绑定p即可运行，该m在整个Go程序的运行过程中至关重要。


sysmon每20us~10ms启动一次，完成以下工作：
> - 释放闲置超过5分钟的span物理内存;
> - 如果超过2分钟没有垃圾回收，强制执行；
> - 将长时间未处理的netpoll结果添加到任务队列；
> - 向长时间运行的G任务发出抢占调度；
> - 收回因syscall长时间阻塞的P；

如何做的？

### retake超时抢占
1. 记录所有P的G任务计数schedtick，每次执行不同的G任务，则计数+1
2. 如果10ms内，schedtick没有增加，说明该P一直在执行同一个G，在该G中加一个tag抢占标记
3. G任务在执行的时候，如果遇到非内联函数调用，就会检查tag，如果有tag，runtime便可以将G抢占，并重新放入队列等待下次被调度
4. 如果没有遇到非内敛函数调用，就无法被抢占，该任务会一直被执行。
5. 中断时，将寄存器里的栈信息保存到自己的G对象中；恢复时，将G对象中的栈信息复制到寄存器中

### channel或network I/O阻塞
1. G被放置到某个wait队列中，M会执行下一个G
2. 如果没有G供M执行，M会解绑P，进入sleep状态
3. 当I/O available或channel操作完成，wait队列中的G被唤醒，放入P，绑定一个M执行

### syscall阻塞
1. 不仅G被阻塞，M也会解绑P，与G一起进入sleep状态
2. 如果此时有idle的M，则P与其绑定执行下一个G
3. 如果没有idle的M，则创建一个新M，与P绑定执行下一个G

## Work Stealing
> - Work-sharing: When a processor generates new threads, it attempts to migrate some of them to the other processors with the hopes of them being utilized by the idle/underutilized processors.
> - Work-stealing: An underutilized processor actively looks for other processor’s threads and “steal” some.

与Work-sharing相比，Work-stealing发生的频率更低，当所有P都有工作要运行时，不会迁移任何线程。只有有空闲P时，才会考虑迁移。

### How?
1. 当P执行完G，并且在自己的队列&全局队列中都无法获取到G时
2. 会把其他P的队列队尾的G抢一半可运行的分配给自己

## 一张有趣的图片

![G-P-M模型](https://raw.githubusercontent.com/Carey6918/Carey6918.github.io/master/assets/images/goscheduler2.png)

土拨鼠：M；小车：P；砖块：G。
https://zhuanlan.zhihu.com/p/22352969

## Reference
[golang的goroutine调度机制](https://blog.csdn.net/liangzhiyang/article/details/52669851)

[也谈goroutine调度器](https://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/)

[Go's work-stealing scheduler](https://rakyll.org/scheduler/)


