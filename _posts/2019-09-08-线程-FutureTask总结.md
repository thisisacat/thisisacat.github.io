---
layout: post
title: 线程-FutureTask总结
date: 2019-09-08
tags: 线程  
---


### futuretask只会执行一次,可用于创建缓存等
run方法

```
if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
```
### get()方法会一直阻塞，当使用线程池 DiscardPolicy和DiscardOldestPolicy的时候需要注意，否则会一直阻塞

### FutureTask内部有一个state用来展示任务的状态,并且是volatile修饰的

```
     * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
```

### FutureTask内部的阻塞、超时机器是通过LockSupport来实现

### FutureTask内部WaitNode阻塞队列，类似无界链表



转载请注明：[这不是一只猫的博客](http://1024.notacat.cn) » [点击阅读原文](http://1024.notacat.cn/2019/09/%e7%ba%bf%e7%a8%8b-FutureTask%e6%80%bb%e7%bb%93%2f/)


