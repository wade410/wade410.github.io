---
layout: post
title: 理解Java线程池：ThreadPoolExecutor
date: 2020-09-02 16:15:06 
tag: 线程池 
---

# 1.简单介绍线程池

## 1.1线程池的作用

线程池采用“池化”的思想，方便管理和重用“池”中的线程。具有降低资源的消耗（重复利用已经创建的线程）、提升响应速度（某些任务可以不需要等待线程创建）、提高线程的管理性（HashSet存储线程池中的线程，方便进行管理）。
# 2.线程池的原理
## 2.1当一个任务提交到线程池中经过那些流程？
当任务提交到线程池，会进行**任务分配**。任务的处理方式有四种，**第一种情况**，当线程池的数量小于核心线程数，直接创建一个线程去执行任务。**第二种情况**线程池中的数量大于核心线程数，任务会被放入阻塞队列进行任务缓冲。**第三种情况**，当阻塞队列已满并且线程数量小于最大线程数，此时线程池仍会创建一个线程去执行任务。**第四种情况**，当阻塞队列已满，并且线程数量已经大于最大线程数，此时就会任务拒绝。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200425104301174.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1MjUzNTM1NzU4NQ==,size_16,color_FFFFFF,t_70)
上图可以看出，线程池的内部实际上构建了一个消费者生产者模型，将任务和线程两者进行解耦。所以线程池的运行分为两部分：任务管理、线程管理。**任务管理**充当生产者的角色，当任务提交到线程池中，线程池会判断该任务的流向：（1）直接执行该任务、（2）进入阻塞队列等待线程去执行、（3）拒绝该任务。**线程管理**充当消费者，线程放在线程池统一管理，执行完任务后会从阻塞队列里面自动获取任务。
## 2.2线程池如何维护自身的状态？
线程池内部使用一个变量维护两个值：运行状态（runState）、线程数量（workerCount）。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020042511084541.png)
ct1一个AtomicIntege类型的值，包含了两个信息：线程池的运行状态 (runState) 和线程池内有效线程的数量 (workerCount)。其中高3位保存runState，低29位保存workerCount。
以下便是线程池ct1高3位保存的运行状态（runState）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200425111236652.png)
### 2.2.1线程池五种运行状态的介绍：
1. RUNNING ：能接受新提交的任务，并且也能处理阻塞队列中的任务；
2. SHUTDOWN：关闭状态，不再接受新提交的任务，但却可以继续处理阻塞队列中已保存的任务。
3. STOP：不能接受新任务，也不处理队列中的任务，会中断正在处理任务的线程。
4. TIDYING：如果所有的任务都已终止了，workerCount (有效线程数) 为0，线程池进入该状态后会调用 terminated() 方法进入TERMINATED 状态。
5. TERMINATED：在terminated() 方法执行完后进入该状态。
下图为线程池的状态转换过程：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200425111603309.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1MjUzNTM1NzU4NQ==,size_16,color_FFFFFF,t_70)
### 2.2.2状态管理的一些执行方法
tryTerminate方法：tryTerminate方法根据线程池状态进行判断是否结束线程池。
shutdown方法：要将线程池切换到SHUTDOWN状态，并调用interruptIdleWorkers方法请求中断所有空闲的worker，最后调用tryTerminate尝试结束线程池。
interruptIdleWorkers方法：interruptIdleWorkers遍历workers中所有的工作线程，若线程没有被中断tryLock成功，就中断该线程。
shutdownNow方法与shutdown方法类似，不同的地方在于：设置状态为STOP；中断所有工作线程，无论是否是空闲的；取出阻塞队列中没有被执行的任务并返回。

## 2.3线程池的构造方法
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200425114905456.png)

### 2.3.1了解一下几个重要的参数：
**corePoolSize**：核心线程数。     
**maximumPoolSize**：线程池最大线程数。      
**keepAliveTime**：线程池除核心线程外的其他线程的最长空闲时间，超过该时间的空闲线程会被销毁  。    
**unit**：keepAliveTime的单位，TimeUnit中的几个静态属性 。     
**workQueue**：线程池所使用的任务缓冲队列（阻塞队列）。      
**threadFactory**：线程池中创建线程的工厂类，一般使用默认的就可以了。      
**handler**：当线程池中的线程数量超过maximumPoolSize时，提交新任务时会被线程池拒绝，handler为拒绝任务的处理策略。
线程池提供如下策略：

- AbortPolicy：直接抛出异常，这是默认策略；
 - CallerRunsPolicy：用调用者所在的线程来执行任务； 
- DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，并执行当前任务； 
-  DiscardPolicy：直接丢弃任务

## 2.4我们是怎么提交任务的（execute方法）？ 
execute(Runnable cmmand）方法是任务进入线程池中的入口。也就是我们生产者开始“生产”任务。从源码中看出，任务的处理方式有四种。

```java
 public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))//第一种情况，当前线程数小于核心线程数，创建一个线程。
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {//第二种情况，大于核心线程数，放入阻塞队列（workQueue）
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);//这个是一种特殊的情况考虑，如何初始化设置的核心线程数为0，那么进来一个任务就没办法执行。所以就创建一个没有任务的线程。
    }
    else if (!addWorker(command, false))////第三种情况，大于核心线程数且小于最大线程数，并且阻塞队列已经满了，创建一个线程。
        reject(command);//第四种情况，拒绝任务
}
/*
addWorker(command false)表示会跟最大线程数maximumPoolSize进行比较
addWorker(command ture)表示会跟最大线程数corePoolSize进行比较。
*/
```
execute方法的执行流程如下：![在这里插入图片描述](https://img-blog.csdnimg.cn/20200425115013342.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1MjUzNTM1NzU4NQ==,size_16,color_FFFFFF,t_70)
## 2.5线程池是怎么创建线程的（addWorker）？ 

execute方法中，使用addWorker方法来创建一个线程去执行任务。
以下是addWorker方法的截取。	
### 2.5.1什么时候创建任务失败呢？
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200425120013215.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1MjUzNTM1NzU4NQ==,size_16,color_FFFFFF,t_70)
1、当线程池的状态不是Running。
2、当线程数大于最大线程数maximumPoolSize时。
### 2.5.2如何创建线程呢？
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200425120549911.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1MjUzNTM1NzU4NQ==,size_16,color_FFFFFF,t_70)
上面就是创建一个新的线程去执行。workers是一个HashSet类型的对象，workers里面存放的就是线程池里的所有线程，根据workers，可以管理线程池里的线程。
## 2.6线程池里线程是一个worker对象 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200425151434600.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1MjUzNTM1NzU4NQ==,size_16,color_FFFFFF,t_70)
1是Worker实现了Runnable方法，也就相当于创建了一个线程。
2是Worker类中的两个属性:thread、firstTask。
3构造方法传入firstTask，thread由线程工程创建。
4是Worker的执行方法（run方法）。
线程池里的线程其实是Worker类对象。只是Worker类对象实现了Runnable方法，重写run方法。也就是一个线程。所以Worker对象执行的时候会调用Worker类中的run方法。
## 2.7线程也创建了，那么如何执行呢（runWorker）？ 
线程的执行当然是Worker类中的run方法了。run方法中执行了runWorker方法。runWorker方法是线程池中线程的最终执行方法。 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200425153112736.png)
下面是runWorker方法的截取部分
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020042515334622.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1MjUzNTM1NzU4NQ==,size_16,color_FFFFFF,t_70)
1是一个while循环获取任务，其中task指的是当前任务（也就是任务进线程池时，初次创建线程去执行的任务），getTask（）指的是从阻塞队列里面获取的任务。这里的任务是Runnable对象。
2是代表任务的执行，Runnable对象执行run方法。
3是销毁线程，表示while循环退出，也就是getTask方法返回null。
runWorker是线程的执行，while循环和getTask方法解释了线程如何从阻塞队列里面获取任务了。
## 2.8那么如何从阻塞队列里面获取任务的呢（getTask方法）？
getTask从阻塞队列里面获取任务。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200425154812413.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1MjUzNTM1NzU4NQ==,size_16,color_FFFFFF,t_70)
1. timed变量用于判断是否需要进行超时控制。allowCoreThreadTimeOut默认是false，也就  是 核心线程不允许进行超时；
2. 根据timed来决定阻塞队列（workQueue）使用poll方法还是take方法。

### 2.8.1这里的timed表示超时控制是什么意思呢？ 
阻塞队列（workQueue）的poll方法的意思是：在队列为空的情况下，在keepAliveTime的时间内没有获取到任务就会返回null，队列不为空就会直接返回队列中的任务。
阻塞队列（workQueue）的take方法的意思是：在队列为空的情况下，如果获取不到任务就会一直阻塞等待，队列不为空就会直接返回队列中的任务。
当前线程数大于核心线程数，timed会为true。当阻塞队列为空的时候，如果在keepAliveTime的时间内没有获取到任务就会返回null。getTask返回null意味着没有获取任务就会销毁此线程。这也是为什么线程池要使用阻塞队列的原因之一，同时也解释了在没有任务的时候，线程池是如何销毁大于核心线程数的线程，如何保持线程池中的线程数量小于等于核心线程数的。
## 2.9那么线程是如何销毁的呢（processWorkerExit）？ 
从runWorker方法中知道当getTask方法返回空或者异常，就会进行退出while循环进入processWorkerExit方法。也就是说当阻塞队列里面的线程为空时会撤销一部分线程，使得线程池中的线程数等够小于或者等于核心线程数。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200425163127845.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1MjUzNTM1NzU4NQ==,size_16,color_FFFFFF,t_70)
从线程池中移除，也就是从workers（存储所有worker对象的HashSet）移除。如果是异常而进行线程撤销，那么会重新创建一个线程。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200425163234770.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1MjUzNTM1NzU4NQ==,size_16,color_FFFFFF,t_70)




# 3.总结
以下就是任务提交到线程撤销的流程。
从execute方法开始，Worker使用ThreadFactory创建新的工作线程，runWorker通过getTask获取任务，然后执行任务，如果getTask返回null，进入processWorkerExit方法，整个线程结束
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200425163419989.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1MjUzNTM1NzU4NQ==,size_16,color_FFFFFF,t_70)
根据上面的流程去剖析源码，能够了解源码的想要表达什么。
# 4. 线程池的使用方法
在理解了线程池的工作原理和关键参数后，业务人员可以根据自身的业务情况创建具有不同参数的线程池，就是设置corePoolSize、maximumPoolSize、keepAliveTime、unit、workQueue、threadFactory、handler。如果没有特别要求，threadFactory、handler可以直接使用默认值。

**corePoolSize**：对于CPU密集型的业务，如Netty框架，那么它的线程模型中corePoolSize的值默认就是CPU核数。对于IO密集型任务，因为线程大多数时间处于阻塞状态，为了更好的处理任务，一般会把corePoolSize的值设置为cpu核数的2~3倍，当然这并不是一定的，最终还是可以根据业务和CPU的负载来调优。

**maximumPoolSize**：思路与corePoolSize的分析方法类似，CPU密集型的一般设置为CPU核数的2倍，IO密集型的一般设置为CPU的4~6倍，并且要根据业务的实际情况和CPU负载来具体分析，比如业务量比较小，那么可以将值设置的小一点，业务量大且CPU的负载低的情况下，可以将值设置的更大一点。

**keepAliveTime和unit**：这两个值没有固定的思路，如果不想频繁创建和销毁线程，则可以设置大一点，比如10分钟，如果希望保持尽量少的线程处理任务，则可以设置小一点，比如1分钟。

**workQueue**：阻塞队列，可以使用BlockingQueue接口的任意一个实现类，有ArrayBlockingQueue底层基于数组，所以在必须指定容量。LinkedBlockingQueue底层基于链表，所以可以是无界队列（因此也要非常注意，容易耗尽内存）。PriorityBlockingQueue是优先级队列，在初始化的时候可以传入一个Comparator来决定哪个对象的优先级更高，优先级更高的对象将会被更早取出，因为涉及到排序，所以必定会降低性能，因此非特别情况下不要使用。SynchronousQueue队列中并不存储元素，它只是提供两个线程进行信息交换的场所。DelayQueue支持延迟获取元素，只能存入Delayed接口实现的对象。实际上没有特殊要求，最好使用LinkedBlockingQueue，并且设置好队列的容量。

**threadFactory**：线程创建工厂，基本上使用默认的就可以了。

**handler**：拒绝策略类，基本上使用默认的就可以了。
# 5.资料参考 
[深入理解Java线程池：ThreadPoolExecutor](http://www.ideabuffer.cn/2017/04/04/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Java%E7%BA%BF%E7%A8%8B%E6%B1%A0%EF%BC%9AThreadPoolExecutor/)
[Java线程池实现原理及其在美团业务中的实践](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)
[Java基础篇-线程与线程池](https://zhuanlan.zhihu.com/p/106803927)





    




