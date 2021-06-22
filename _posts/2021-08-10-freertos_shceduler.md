---
layout: post
title: "Freertos任务调度原理"
date: 2021-08-10 
description: "Freertos任务切换的底层原理"
#tag: 
---  

## 任务创建
freerots中使用xTaskCreate来进行任务的创建，该函数的调用关系如下图所示，首先是通过pvPortMalloc函数创建任务的堆栈stack以及任务控制块TCB。值得一提的是，有些时候为了检测堆栈溢出，分配的stack内会被填充满0xA5(10100101)，当检测到栈的末尾的值不是0xA5了，说明栈溢出了。而TCB的作用就是记录所创建的任务的所有信息，包括名字，优先级，栈的开始位置，以及的栈指针sp寄存器的值。然后会调用prvInitialiseNewTask为这个任务的栈填充一些必要的值使其看起来像是已经被调度过之后的状态了。之后会通过prvAddNewTaskToReadyList函数来将这个新创建的任务加到readylist的任务链表里，readylist这个链表里都是已经就绪，准备被调度的task。

### 初始化堆栈stack
prvInitialiseNewTask这个函数主要是为stack填充了下面几个值，为什么是如下这些值可以参考`<<arm cortex-ms 权威指南>>`的p146。一个任务被切换的时候必然会进入中断，而cortex-m3进入中断是硬件自动将xPSR PC等等，也就是表格最后一行之前的那些寄存器都入栈，然后等切回该任务又会将其出栈，所里这里的栈中的值可以理解为提前准备好的一些值，等这个任务被调度了，这些值就会被pop回寄存器中。关于最后一行的那些寄存器，它们是由软件来保存的，后面任务切换taskyield那边会提到。
<center>

 | stack的值| 
 |--- |
 |xPSR状态寄存器|
 |task函数的指针(PC指针) |
 | LR寄存器|
 |R12 R3 R2 R1 值是内存初始化值，其实就是0|
 | R0 (task函数传入的参数)|
 | R11 R10 R9 R8 R7 R6 R5 R4 值是内存初始化值，其实就是0|

</center>

### 各种任务链表
freerots中task存在几种状态，就绪态、阻塞态等，其实在代码中具体实现就是靠各种链表维护的，不同状态的task的TCB被插入到自己所处状态的链表中，当状态改变之后，从原来状态的链表中摘除，然后插入到新状态的链表中。例如vTaskDelay()延时的任务就会处于delayedTasklist链表，阻塞状态的任务就会处于supendedTasklist链表。而运行态的任务比较特殊，它不是一个链表，因为一个时刻运行的只有一个task，它是一个指针`pxCurrentTCB`。而就绪任务链表readylist是指以及满足条件可以被调度的task，其实这是一个链表组，每个链表中都是同一优先级的task，而不同优先级的链表头组成了一个数组，数组下标就是优先级。prvAddNewTaskToReadyList函数的调用关系如下，其中prvInitialiseTaskLists是用来初始化这些任务链表的，但这个函数只有在第一次创建task时才执行。这里存在一种特殊的情况，就是新创建的task比当前所有task的优先级都高，那么就应该直接把当前运行的task顶掉，所以就调用了taskYIELD_IF_USING_PREEMPTION。这个函数最终调用的是portYIELD。



## 任务调度
任务调度的核心就是切换task，依赖于portYIELD函数，其实这个就只是挂起了pendsv的中断位，其他的事情就交给pendsv这个中断的handler来处理了。freertos的cortex-m3代码中，pendsv中断服务函数是xPortPendSVHandler，其具体的内容如下
```armasm
mrs r0, psp 
isb                         ;步骤1

ldr r3, pxCurrentTCBConst 
ldr r2, [r3]                ;步骤2

stmdb r0!, {r4-r11} 
str r0, [r2]                ;步骤3

stmdb sp!, {r3, r14} 
mov r0, %0 
msr basepri, r0 
bl vTaskSwitchContext 
mov r0, #0 
msr basepri, r0 
ldmia sp!, {r3, r14}        ;步骤4

ldr r1, [r3] 
ldr r0, [r1] 
ldmia r0!, {r4-r11} 
msr psp, r0 
isb 
bx r14                      ;步骤5
```
下面详细解释一下每一步的过程，每个步骤跟下面的小标对应
1. cortex-m3有是由两个堆栈指针寄存器的，一个msp，一个psp。当进入中断状态之后，msp起作用，步骤1就是将psp值赋给r0，psp指向的是中断之前的task的堆栈顶部。
2. 将pxCurrentTCBConst(其实就是pxCurrentTCB)的地址赋给r3，将r3指向的值进行赋给r2，此时r2其实存放的就是当前task(也就是中断之前的task)的TCB地址。
3. 将r4到r11的寄存器依次存入r0往下的空间(r0放的就是之前task的是sp指针)，其实就是入栈了，r0还更新成了最新的栈顶指针。将r0的值赋值给[r2]，其实就是将最新的栈指针赋值给了TCB的第一项。结合前两个步骤看，这些动作其实就是中断之后，软件完成r4到r11寄存器的入栈，并且将最新的栈顶指针存到TCB相应的成员中。
4. 此时sp其实是msp，将r3 r14存入msp所指向的主堆栈。意味着主堆栈中存放至pxCurrentTCBConst变量地址和LR的值，这个LR的值是入中断时自动更新的，值是EXC_RETURN。这里可以理解为保护这两个寄存器的值，因为等会儿还要跳转到vTaskSwithContext，说不定就会把这连个寄存器的值冲掉。而后面还要用到这连个寄存器，所以先保护起来。后面先屏蔽中断，调用vTaskSwithContext，这个函数其实就是把readylist中符合运行条件的task赋值给pxCurrentTCB。之后开启中断，将r3 r4再从主堆栈pop出来。
5. 从r3中取出新TCB的指针，然后将这个TCB存储的sp指针值赋值给r0，然后从r0往上的栈空间的值赋值给r4-r11，其实也就是出栈，然后将最新的sp指针再psp寄存器，最终bx r14跳出中断。
此外，这里还隐藏了两个重要的步骤，前面提到中断时候，会有自动入栈，中断结束会自动出栈。所以步骤1之前，原先的task的stack中已经被压入了xPSR, PC等寄存器，而步骤5跳出中断之后，新的task的这组寄存器也会被硬件自动弹出栈。总之整个过程就是旧task的寄存器值入栈，并更新TCB内保存的sp指针值，切换currentTask，然后新task的寄存器出栈，更新psp。这是个对称的操作过程。

## 任务延时阻塞及唤醒
vTaskDelay函数的调用关系如下，主要是prvAddCurrentTaskToDelayedList这个函数，在加入了delaylist之后，剩下的就交给systick的中断处理函数，systick系统时钟，在一定时间例如1ms就会产生一个systick中断，也就是系统节拍。在systick中断中调用了xTaskIncrementTick()函数，这个函数每次为ticks加1，并且和delaylist中的task的delay time的值比较，如果发现有task的延迟时间到期了，就会将这个task摘下来挂到readylist中。之后的操作和portYield类似，挂起了pendsv中断，等待pendsv中断handler来进行实际的任务切换。


<br>
其实分析下来，freertos任务调度的底层核心就是pendsv的handler，它负责实际task的上下文的切换，内核通过systick中断或者主动portYield来触发pendsv中断。逻辑上则依赖于各类状态的链表来维护各个状态的任务，并通过pxCurrentTCB来指向当前运行的任务。抛开pendsv中断这个底层实现来看，freertos的任务调度从逻辑上就变成了怎样将TCB从一个状态链表上摘下来放到另一个状态链表上的过程了。
<br>

分析了一大堆之后，发现[别人](https://my.oschina.net/u/3699634/blog/1547467)比我分析的还要到位，所以我这篇总结自己将就看看吧，是真的懒得把所有示意图都画出来了。