---
layout: post
title: "线程池原理&实现"
date: 2023-04-20
description: "C语言实现线程池"
tag: C/C++
---

### 前言

线程池（thread pool）：一种线程使用模式。线程过多会带来调度开销，进而影响缓存局部性和整体性能。而线程池维护着多个线程，等待着监督管理者分配可并发执行的任务。这避免了在处理短时间任务时创建与销毁线程的代价。线程池不仅能够保证内核的充分利用，还能防止过分调度。可用线程数量应该取决于可用的并发处理器、处理器内核、内存、网络sockets等的数量。 例如，对于计算密集型任务，线程数一般取cpu数量+2比较合适，线程数过多会导致额外的线程切换开销。

任务调度以执行线程的常见方法是使用同步队列，称作任务队列。池中的线程等待队列中的任务，并把执行完的任务放入完成队列中。

线程池模式一般分为两种：HS/HA半同步/半异步模式、L/F领导者与跟随者模式。

（1）半同步/半异步模式又称为生产者消费者模式，是比较常见的实现方式，比较简单。分为同步层、队列层、异步层三层。同步层的主线程处理工作任务并存入工作队列，工作线程从工作队列取出任务进行处理，如果工作队列为空，则取不到任务的工作线程进入挂起状态。由于线程间有数据通信，因此不适于大数据量交换的场合。

（2）领导者跟随者模式，在线程池中的线程可处在3种状态之一：领导者leader、追随者follower或工作者processor。任何时刻线程池只有一个领导者线程。事件到达时，领导者线程负责消息分离，并从处于追随者线程中选出一个来当继任领导者，然后将自身设置为工作者状态去处置该事件。处理完毕后工作者线程将自身的状态置为追随者。这一模式实现复杂，但避免了线程间交换任务数据，提高了CPU cache相似性。在ACE(Adaptive Communication Environment)中，提供了领导者跟随者模式实现。
线程池的伸缩性对性能有较大的影响。

创建太多线程，将会浪费一定的资源，有些线程未被充分使用。
销毁太多线程，将导致之后浪费时间再次创建它们。
创建线程太慢，将会导致长时间的等待，性能变差。
销毁线程太慢，导致其它线程资源饥饿。

——《维基百科》

### 目录

* [第一章](#线程池原理)
* [第二章](#线程池实现)

### <a name="chapter1"></a>线程池原理

池的概念是一种非常常见的空间换时间的概念，除了有线程池之外还有进程池、内存池等等。其实他们的思想都是一样的就是我先申请一批资源出来，然后就随用随拿，不用再放回来。线程池的好处有：

（1）降低资源消耗：通过重复利用现有的线程来执行任务，避免多次创建和销毁线程。

（2）提高相应速度：因为省去了创建线程这个步骤，所以在拿到任务时，可以立刻开始执行。

（3）提供附加功能：线程池的可拓展性使得我们可以自己加入新的功能，比如说定时、延时来执行某些线程。

线程池的组成主要分为`3个部分`，这三部分配合工作就可以得到一个完整的线程池：

1、 `任务队列`，存储需要处理的任务，由工作的线程来处理这些任务

* 通过线程池提供的 API 函数，将一个待处理的任务添加到任务队列，或者从任务队列中删除

* 已处理的任务会被从任务队列中删除

* 线程池的使用者，也就是调用线程池函数往任务队列中添加任务的线程就是生产者线程

2、 `工作的线程`（任务队列任务的消费者） ，N个

* 线程池中维护了一定数量的工作线程，他们的作用是是不停的读任务队列，从里边取出任务并处理

* 工作的线程相当于是任务队列的消费者角色，

* 如果任务队列为空，工作的线程将会被阻塞 (使用条件变量 / 信号量阻塞)

* 如果阻塞之后有了新的任务，由生产者将阻塞解除，工作线程开始工作

3、 `管理者线程`（不处理任务队列中的任务），1个

* 它的任务是周期性的对任务队列中的任务数量以及处于忙状态的工作线程个数进行检测

* 当任务过多的时候，可以适当的创建一些新的工作线程

* 当任务过少的时候，可以适当的销毁一些工作的线程

![](https://s3.uuu.ovh/imgs/2023/04/21/12905c53d685331d.png)

### <a name="chapter2"></a>线程池实现实例（C语言）

* 任务队列

		// 任务结构体
		typedef struct Task
		{
		    void (*function)(void* arg);
		    void* arg;
		}Task;

* 线程池定义

		// 线程池结构体
		struct ThreadPool
		{
		    // 任务队列
		    Task* taskQ;
		    int queueCapacity;  // 容量
		    int queueSize;      // 当前任务个数
		    int queueFront;     // 队头 -> 取数据
		    int queueRear;      // 队尾 -> 放数据
		
		    pthread_t managerID;    // 管理者线程ID
		    pthread_t *threadIDs;   // 工作的线程ID
		    int minNum;             // 最小线程数量
		    int maxNum;             // 最大线程数量
		    int busyNum;            // 忙的线程的个数
		    int liveNum;            // 存活的线程的个数
		    int exitNum;            // 要销毁的线程个数
		    pthread_mutex_t mutexPool;  // 锁整个的线程池
		    pthread_mutex_t mutexBusy;  // 锁busyNum变量
		    pthread_cond_t notFull;     // 任务队列是不是满了
		    pthread_cond_t notEmpty;    // 任务队列是不是空了
		
		    int shutdown;           // 是不是要销毁线程池, 销毁为1, 不销毁为0
		};

* 头文件声明

		#ifndef _THREADPOOL_H
		#define _THREADPOOL_H
		
		typedef struct ThreadPool ThreadPool;
		// 创建线程池并初始化
		ThreadPool *threadPoolCreate(int min, int max, int queueSize);
		
		// 销毁线程池
		int threadPoolDestroy(ThreadPool* pool);
		
		// 给线程池添加任务
		void threadPoolAdd(ThreadPool* pool, void(*func)(void*), void* arg);
		
		// 获取线程池中工作的线程的个数
		int threadPoolBusyNum(ThreadPool* pool);
		
		// 获取线程池中活着的线程的个数
		int threadPoolAliveNum(ThreadPool* pool);
		
		//////////////////////
		// 工作的线程(消费者线程)任务函数
		void* worker(void* arg);
		// 管理者线程任务函数
		void* manager(void* arg);
		// 单个线程退出
		void threadExit(ThreadPool* pool);
		#endif  // _THREADPOOL_H

* 源文件定义

		ThreadPool* threadPoolCreate(int min, int max, int queueSize)
		{
		    ThreadPool* pool = (ThreadPool*)malloc(sizeof(ThreadPool));
		    do 
		    {
		        if (pool == NULL)
		        {
		            printf("malloc threadpool fail...\n");
		            break;
		        }
		
		        pool->threadIDs = (pthread_t*)malloc(sizeof(pthread_t) * max);
		        if (pool->threadIDs == NULL)
		        {
		            printf("malloc threadIDs fail...\n");
		            break;
		        }
		        memset(pool->threadIDs, 0, sizeof(pthread_t) * max);
		        pool->minNum = min;
		        pool->maxNum = max;
		        pool->busyNum = 0;
		        pool->liveNum = min;    // 和最小个数相等
		        pool->exitNum = 0;
		
		        if (pthread_mutex_init(&pool->mutexPool, NULL) != 0 ||
		            pthread_mutex_init(&pool->mutexBusy, NULL) != 0 ||
		            pthread_cond_init(&pool->notEmpty, NULL) != 0 ||
		            pthread_cond_init(&pool->notFull, NULL) != 0)
		        {
		            printf("mutex or condition init fail...\n");
		            break;
		        }
		
		        // 任务队列
		        pool->taskQ = (Task*)malloc(sizeof(Task) * queueSize);
		        pool->queueCapacity = queueSize;
		        pool->queueSize = 0;
		        pool->queueFront = 0;
		        pool->queueRear = 0;
		
		        pool->shutdown = 0;
		
		        // 创建线程
		        pthread_create(&pool->managerID, NULL, manager, pool);
		        for (int i = 0; i < min; ++i)
		        {
		            pthread_create(&pool->threadIDs[i], NULL, worker, pool);
		        }
		        return pool;
		    } while (0);
		
		    // 释放资源
		    if (pool && pool->threadIDs) free(pool->threadIDs);
		    if (pool && pool->taskQ) free(pool->taskQ);
		    if (pool) free(pool);
		
		    return NULL;
		}
		
		int threadPoolDestroy(ThreadPool* pool)
		{
		    if (pool == NULL)
		    {
		        return -1;
		    }
		
		    // 关闭线程池
		    pool->shutdown = 1;
		    // 阻塞回收管理者线程
		    pthread_join(pool->managerID, NULL);
		    // 唤醒阻塞的消费者线程
		    for (int i = 0; i < pool->liveNum; ++i)
		    {
		        pthread_cond_signal(&pool->notEmpty);
		    }
		    // 释放堆内存
		    if (pool->taskQ)
		    {
		        free(pool->taskQ);
		    }
		    if (pool->threadIDs)
		    {
		        free(pool->threadIDs);
		    }
		
		    pthread_mutex_destroy(&pool->mutexPool);
		    pthread_mutex_destroy(&pool->mutexBusy);
		    pthread_cond_destroy(&pool->notEmpty);
		    pthread_cond_destroy(&pool->notFull);
		
		    free(pool);
		    pool = NULL;
		
		    return 0;
		}
		
		
		void threadPoolAdd(ThreadPool* pool, void(*func)(void*), void* arg)
		{
		    pthread_mutex_lock(&pool->mutexPool);
		    while (pool->queueSize == pool->queueCapacity && !pool->shutdown)
		    {
		        // 阻塞生产者线程
		        pthread_cond_wait(&pool->notFull, &pool->mutexPool);
		    }
		    if (pool->shutdown)
		    {
		        pthread_mutex_unlock(&pool->mutexPool);
		        return;
		    }
		    // 添加任务
		    pool->taskQ[pool->queueRear].function = func;
		    pool->taskQ[pool->queueRear].arg = arg;
		    pool->queueRear = (pool->queueRear + 1) % pool->queueCapacity;
		    pool->queueSize++;
		
		    pthread_cond_signal(&pool->notEmpty);
		    pthread_mutex_unlock(&pool->mutexPool);
		}
		
		int threadPoolBusyNum(ThreadPool* pool)
		{
		    pthread_mutex_lock(&pool->mutexBusy);
		    int busyNum = pool->busyNum;
		    pthread_mutex_unlock(&pool->mutexBusy);
		    return busyNum;
		}
		
		int threadPoolAliveNum(ThreadPool* pool)
		{
		    pthread_mutex_lock(&pool->mutexPool);
		    int aliveNum = pool->liveNum;
		    pthread_mutex_unlock(&pool->mutexPool);
		    return aliveNum;
		}
		
		void* worker(void* arg)
		{
		    ThreadPool* pool = (ThreadPool*)arg;
		
		    while (1)
		    {
		        pthread_mutex_lock(&pool->mutexPool);
		        // 当前任务队列是否为空
		        while (pool->queueSize == 0 && !pool->shutdown)
		        {
		            // 阻塞工作线程
		            pthread_cond_wait(&pool->notEmpty, &pool->mutexPool);
		
		            // 判断是不是要销毁线程
		            if (pool->exitNum > 0)
		            {
		                pool->exitNum--;
		                if (pool->liveNum > pool->minNum)
		                {
		                    pool->liveNum--;
		                    pthread_mutex_unlock(&pool->mutexPool);
		                    threadExit(pool);
		                }
		            }
		        }
		
		        // 判断线程池是否被关闭了
		        if (pool->shutdown)
		        {
		            pthread_mutex_unlock(&pool->mutexPool);
		            threadExit(pool);
		        }
		
		        // 从任务队列中取出一个任务
		        Task task;
		        task.function = pool->taskQ[pool->queueFront].function;
		        task.arg = pool->taskQ[pool->queueFront].arg;
		        // 移动头结点
		        pool->queueFront = (pool->queueFront + 1) % pool->queueCapacity;
		        pool->queueSize--;
		        // 解锁
		        pthread_cond_signal(&pool->notFull);
		        pthread_mutex_unlock(&pool->mutexPool);
		
		        printf("thread %ld start working...\n", pthread_self());
		        pthread_mutex_lock(&pool->mutexBusy);
		        pool->busyNum++;
		        pthread_mutex_unlock(&pool->mutexBusy);
		        task.function(task.arg);
		        free(task.arg);
		        task.arg = NULL;
		
		        printf("thread %ld end working...\n", pthread_self());
		        pthread_mutex_lock(&pool->mutexBusy);
		        pool->busyNum--;
		        pthread_mutex_unlock(&pool->mutexBusy);
		    }
		    return NULL;
		}
		
		void* manager(void* arg)
		{
		    ThreadPool* pool = (ThreadPool*)arg;
		    while (!pool->shutdown)
		    {
		        // 每隔3s检测一次
		        sleep(3);
		
		        // 取出线程池中任务的数量和当前线程的数量
		        pthread_mutex_lock(&pool->mutexPool);
		        int queueSize = pool->queueSize;
		        int liveNum = pool->liveNum;
		        pthread_mutex_unlock(&pool->mutexPool);
		
		        // 取出忙的线程的数量
		        pthread_mutex_lock(&pool->mutexBusy);
		        int busyNum = pool->busyNum;
		        pthread_mutex_unlock(&pool->mutexBusy);
		
		        // 添加线程
		        // 任务的个数>存活的线程个数 && 存活的线程数<最大线程数
		        if (queueSize > liveNum && liveNum < pool->maxNum)
		        {
		            pthread_mutex_lock(&pool->mutexPool);
		            int counter = 0;
		            for (int i = 0; i < pool->maxNum && counter < NUMBER
		                && pool->liveNum < pool->maxNum; ++i)
		            {
		                if (pool->threadIDs[i] == 0)
		                {
		                    pthread_create(&pool->threadIDs[i], NULL, worker, pool);
		                    counter++;
		                    pool->liveNum++;
		                }
		            }
		            pthread_mutex_unlock(&pool->mutexPool);
		        }
		        // 销毁线程
		        // 忙的线程*2 < 存活的线程数 && 存活的线程>最小线程数
		        if (busyNum * 2 < liveNum && liveNum > pool->minNum)
		        {
		            pthread_mutex_lock(&pool->mutexPool);
		            pool->exitNum = NUMBER;
		            pthread_mutex_unlock(&pool->mutexPool);
		            // 让工作的线程自杀
		            for (int i = 0; i < NUMBER; ++i)
		            {
		                pthread_cond_signal(&pool->notEmpty);
		            }
		        }
		    }
		    return NULL;
		}
		
		void threadExit(ThreadPool* pool)
		{
		    pthread_t tid = pthread_self();
		    for (int i = 0; i < pool->maxNum; ++i)
		    {
		        if (pool->threadIDs[i] == tid)
		        {
		            pool->threadIDs[i] = 0;
		            printf("threadExit() called, %ld exiting...\n", tid);
		            break;
		        }
		    }
		    pthread_exit(NULL);
		}

* 测试代码


		void taskFunc(void* arg)
		{
		    int num = *(int*)arg;
		    printf("thread %ld is working, number = %d\n",
		        pthread_self(), num);
		    sleep(1);
		}
		
		int main()
		{
		    // 创建线程池
		    ThreadPool* pool = threadPoolCreate(3, 10, 100);
		    for (int i = 0; i < 100; ++i)
		    {
		        int* num = (int*)malloc(sizeof(int));
		        *num = i + 100;
		        threadPoolAdd(pool, taskFunc, num);
		    }
		
		    sleep(30);
		
		    threadPoolDestroy(pool);
		    return 0;
		}

### 参考资源

* [维基百科：线程池](https://zh.wikipedia.org/wiki/线程池)

* [subingwen](https://subingwen.cn/linux/threadpool/)

转载请注明：[sizheluo的博客](https://sizheluo.github.io) » [ThreadPool](https://sizheluo.github.io/2023/04/ThreadPool/)