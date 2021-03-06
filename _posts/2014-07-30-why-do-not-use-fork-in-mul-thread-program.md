---
layout: post
category: question
title: 为什么不能在多线程程序中使用fork函数？
tagline: by 浅奕
tags: [network, Linux]
---

## 问题描述

为什么不能在多线程程序中使用fork函数？有例外吗？（回答出linux线程模型变迁为好）

----------------------------------------

By: 高源

###历史背景：

在Linux创建的初期，内核一直就没有实现“线程”这个东西。后来因为实际的需求，便逐步产生了LinuxThreads这个项目，其主要的贡献者是Xavier Leroy。LinuxThreads项目使用了clone()这个系统调用对线程进行了模拟，按照《Linux内核设计与实现》的说法，调用clone()函数参数是

	clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND,  0)

即创建一个新的进程，同时让父子进程共享地址空间、文件系统资源、文件描述符、信号处理程序以及被阻断的信号等内容。也就是说，此时的所谓“线程”模型符合以上两本经典巨著的描述，即在内核看来，没有所谓的“线程”，我们所谓的“线程”其实在内核看来不过是和其他进程共享了一些资源的进程罢了。

[参考刘欢学长《Linux线程的前世今生》](http://www.0xffffff.org/?p=482)
                                                  
当线程调用fork()时，就为子进程创建了整个进程地址空间的副本。（子进程中该线程的ID与父进程中发起fork()调用的线程ID是一样的，因此，线程ID相同的情况有时我们需要做特殊的处理。）在子进程内部，只存在一个线程，它是由父进程调用fork()的线程的副本构成的。也就是说不能同时创建出像父进程一样多线程的子进程。其他线程均在子进程中消失，『1』并且不会为这些线程调用清理函数以及针对线程局部存储变量的析构函数。如果父进程中的线程占有锁，子进程将同样占有这些锁。但子进程并不包含占有锁的线程的副本，故子进程没有办法知道它占有了哪些锁、需要释放哪些锁。可能导致下列情况：

   1、虽然只将发起调用的线程复制到子进程中，但全局变量的状态以及所有的Pthreads对象（如互斥量、条件变量等）都会在子进程中得以保留。（因为在父进程中为这些Pthreads对象分配了内存，而子进程则获得了该内存的一份拷贝）一个线程在fork()被调用前锁定了某个互斥量，且对某个全局变量的更新也做到了一半，此时fork()被调用，所有数据及状态被拷贝到子进程中，那么子进程中对该互斥量就无法解锁，如果再试图锁定该互斥量就会导致死锁。所以，在fork函数被调用后，此时不能调用线程安全的函数，除非函数是可重入的。

  2、因为并未执行清理函数和针对线程局部存储数据的析构函数，所以多线程情况下可能会导致子进程的内存泄露。另外，子进程中的线程可能无法访问父进程中由其他线程所创建的线程局部存储变量，因为子进程没有任何相应的引用指针。

为了避免不一致状态的问题，在fork(0返回和子进程调用其中一个exec函数之间，子进程只能调用异步信号安全的函数。可以通过调用pthread_atfork函数建立fork处理程序。但其仍然存在不足，只能在有限情况下使用。

###  特殊情况:
如果子进程从fork()返回以后马上调用其中一个exec函数，这样，旧的地址空间就被丢弃,锁的状态就无关紧要。
                  
                  以上内容参考自 《Unix环境高级编程第三版》


>#####『1』《Linux/UNIX系统编程手册》33.3  *Threads and Process Control*
When a multithreaded process calls fork(), only the calling thread is replicated in the child process. (The ID of the thread in the child is the same as the ID of the thread that called fork() in the parent.) All of the other threads vanish in the child;no thread-specific data destructors or cleanup handlers are executed for those threads.
        
  
