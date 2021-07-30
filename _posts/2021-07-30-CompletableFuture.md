---
layout: post
title: CompletableFuture
date: 2021-07-30
tags: CompletableFuture    
---
*Doug Lea 果然是大神中的大神。CompletableFuture强撸了一个又一个美丽的夜晚*
### **结论** 
**CompletableFuture 更多的是体现一种依赖关系src,dep,snd**

### **源码分析**

*mode<br/>*
mode的不同,影响在d.uniRun(a = src, fn, mode > 0 ? null : this)),postFire()等<br/>

```
 static final int SYNC   =  0; //当前线程触发
 static final int ASYNC  =  1; //线程池触发
    {
    	public final void run()                { tryFire(ASYNC); }
        public final boolean exec()            { tryFire(ASYNC); return true; }
    }
    static final int NESTED = -1;//postComplete触发
   
    
```

*postFire<br/>*
根据mode的不同，返回this还是null,从而影响postComplete方法里面f的返回值,f的不同影响h的取值，是src的CompletableFuture还是dep的CompletableFuture

```
 final CompletableFuture<T> postFire(CompletableFuture<?> a, int mode) {
        if (a != null && a.stack != null) {
            if (mode < 0 || a.result == null)
                a.cleanStack();
            else
                a.postComplete();
        }
        if (result != null && stack != null) {
            if (mode < 0)
                return this;
            else
                postComplete();
        }
        return null;
     }
```

*push<br/>*
注意这里 是采用头插法的形式

```
 final void push(UniCompletion<?,?> c) {
        if (c != null) {
            while (result == null && !tryPushStack(c))
                lazySetNext(c, null); // clear on failure
        }
    }
```

*tryFire*

```
 final CompletableFuture<Void> tryFire(int mode) {
            CompletableFuture<Void> d; CompletableFuture<T> a;
            if ((d = dep) == null ||
            	// 根据mode的不同，传入null或则this，mode=ASYNC,uniRun方法里面直接f.run() （此时已经处于另外一个线程里面）
                !d.uniRun(a = src, fn, mode > 0 ? null : this))
                return null;
            dep = null; src = null; fn = null;
            return d.postFire(a, mode);
        }
```

*uniRun*

```
 final boolean uniRun(CompletableFuture<?> a, Runnable f, UniRun<?> c) {
        Object r; Throwable x;
        //通过src.result判断是否要调用自己
        if (a == null || (r = a.result) == null || f == null)
            return false;
        if (result == null) {
        	//异常和result 都是AltResult
            if (r instanceof AltResult && (x = ((AltResult)r).ex) != null)
                completeThrowable(x, r);
            else
                try {
                	//mode=ASYNC的时候c=null,直接调用f.run(),此时在另外一个线程里面
                    if (c != null && !c.claim())
                        return false;
                    f.run();
                    completeNull();
                } catch (Throwable ex) {
                    completeThrowable(ex);
                }
        }
        return true;
    }
```

*claim*

```
 final boolean claim() {
            Executor e = executor;
            if (compareAndSetForkJoinTaskTag((short)0, (short)1)) {
                if (e == null)
                    return true;
                executor = null; // disable
                //这里会触发tryFile(mode=ASYNC) 模式
                e.execute(this);
            }
            return false;
        }
```

*postFire<br/>*
tryFire 和 postFire里面的两个return 关系到postComplete<br/>
根据mode的不同，返回this还是null,从而影响postComplete方法里面f的返回值,f的不同影响h的取值，是src的CompletableFuture还是dep的CompletableFuture<br/>

```
 final CompletableFuture<T> postFire(CompletableFuture<?> a, int mode) {
        if (a != null && a.stack != null) {
            if (mode < 0 || a.result == null)
                a.cleanStack();
            else
                a.postComplete();
        }
        if (result != null && stack != null) {
            if (mode < 0)
                return this;
            else
                postComplete();
        }
        return null;
    }
```

*postComplete<br/>*

```
 final void postComplete() {
        /*
         * On each step, variable f holds current dependents to pop
         * and run.  It is extended along only one path at a time,
         * pushing others to avoid unbounded recursion.
         */
        CompletableFuture<?> f = this; Completion h;
        while ((h = f.stack) != null ||
               (f != this && (h = (f = this).stack) != null)) {
            CompletableFuture<?> d; Completion t;
            if (f.casStack(h, t = h.next)) {
            	//注意这里t!=null,f!=this的情况下，最后一个t==null,下面的逻辑不触发
                if (t != null) {
                	//注意这里,f!=this的情况下，会发生push操作
                    if (f != this) {
                        pushStack(h);
                        continue;
                    }
                    h.next = null;    // detach
                }
                //类似thenRunAsync这些,基本都是返回null，然后另外一个线程去处理thenRunAsync的逻辑，然后在另外一个线程里面会调用public final void run() { tryFire(ASYNC); }，开启ASYNC模式
                f = (d = h.tryFire(NESTED)) == null ? this : d;
            }
        }
    }
```

### **执行顺序**

大致的执行逻辑是 stack(dep) 为一个单元执行完，然后转下一个stack(dep)<br/>
![](/images/posts/CompletableFuture/process.png)<br/>

*r1->r4->r5->r6->r8->r7->r3->r2<br/>*

```
ThreadPoolExecutor executorService1 = new ThreadPoolExecutor(10, 10,
                0L, TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<>(100));

        Runnable r1 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
//                int a =1;
//                int b =0;
//                int c = a/b;
                try {
                    System.out.println("begin r1 begin r1 begin r1 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r1 finish r1 finish r1 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r2 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r2 begin r2 begin r2 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r2 finish r2 finish r2 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };


        Runnable r3 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r3 begin r3 begin r3 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r3 finish r3 finish r3 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };


        Runnable r4 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r4 begin r4 begin r4 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r4 finish r4 finish r4 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r5 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r5 begin r5 begin r5 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r5 finish r5 finish r5 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r6 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r6 begin r6 begin r6 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r6 finish r6 finish r6 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r7 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r7 begin r7 begin r7 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r7 finish r7 finish r7 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r8 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r8 begin r8 begin r8 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r8 finish r8 finish r8 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };


        CompletableFuture cf1 = CompletableFuture.runAsync(r1,executorService1);

        CompletableFuture cf2 = cf1.thenRun(r2);
        CompletableFuture cf3 = cf1.thenRun(r3);
        CompletableFuture cf4 = cf1.thenRun(r4);


        CompletableFuture cf5 = cf4.thenRun(r5);
        CompletableFuture cf6 = cf4.thenRun(r6);
        CompletableFuture cf7 = cf4.thenRun(r7);

        CompletableFuture cf8 = cf6.thenRun(r8);


//     r1->r4->r5->r6->r8->r7->r3->r2
```
*r1->r4->r7->r10->r3->r6->r9->r2->r5->r8<br/>*

```
ThreadPoolExecutor executorService1 = new ThreadPoolExecutor(10, 10,
                0L, TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<>(100));

        Runnable r1 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
//                int a =1;
//                int b =0;
//                int c = a/b;
                try {
                    System.out.println("begin r1 begin r1 begin r1 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r1 finish r1 finish r1 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r2 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r2 begin r2 begin r2 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r2 finish r2 finish r2 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };


        Runnable r3 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r3 begin r3 begin r3 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r3 finish r3 finish r3 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };


        Runnable r4 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r4 begin r4 begin r4 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r4 finish r4 finish r4 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r5 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r5 begin r5 begin r5 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r5 finish r5 finish r5 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r6 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r6 begin r6 begin r6 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r6 finish r6 finish r6 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r7 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r7 begin r7 begin r7 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r7 finish r7 finish r7 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r8 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r8 begin r8 begin r8 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r8 finish r8 finish r8 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r9 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r9 begin r9 begin r9 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r9 finish r9 finish r9 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r10 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r10 begin r10 begin r10 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r10 finish r10 finish r10 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };



        CompletableFuture cf1 = CompletableFuture.runAsync(r1,executorService1);

        CompletableFuture cf2 = cf1.thenRun(r2);
        CompletableFuture cf3 = cf1.thenRun(r3);
        CompletableFuture cf4 = cf1.thenRun(r4);


        CompletableFuture cf5 = cf2.thenRun(r5);
        CompletableFuture cf6 = cf3.thenRun(r6);
        CompletableFuture cf7 = cf4.thenRun(r7);

        CompletableFuture cf8 = cf5.thenRun(r8);
        CompletableFuture cf9 = cf6.thenRun(r9);
        CompletableFuture cf10 = cf7.thenRun(r10);

//        r1->r4->r7->r10->r3->r6->r9->r2->r5->r8
```
*r1->r4->(r5异步,r6异步,r7同步)  r6结束->r8,r7结束->r3->r2<br/>*

```
ThreadPoolExecutor executorService1 = new ThreadPoolExecutor(10, 10,
                0L, TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<>(100));

        ThreadPoolExecutor executorService2 = new ThreadPoolExecutor(10, 10,
                0L, TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<>(100));

        Runnable r1 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r1 begin r1 begin r1 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r1 finish r1 finish r1 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r2 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r2 begin r2 begin r2 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r2 finish r2 finish r2 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };


        Runnable r3 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r3 begin r3 begin r3 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r3 finish r3 finish r3 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };


        Runnable r4 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r4 begin r4 begin r4 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r4 finish r4 finish r4 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r5 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r5 begin r5 begin r5 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r5 finish r5 finish r5 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r6 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r6 begin r6 begin r6 " + System.currentTimeMillis());
                    Thread.sleep(3000);
                    System.out.println("finish r6 finish r6 finish r6 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r7 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r7 begin r7 begin r7 " + System.currentTimeMillis());
                    Thread.sleep(10000);
                    System.out.println("finish r7 finish r7 finish r7 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("r7 cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r8 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r8 begin r8 begin r8 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r8 finish r8 finish r8 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };


        CompletableFuture cf1 = CompletableFuture.runAsync(r1,executorService1);

        CompletableFuture cf2 = cf1.thenRun(r2);
        CompletableFuture cf3 = cf1.thenRun(r3);
        CompletableFuture cf4 = cf1.thenRun(r4);

        CompletableFuture cf5 = cf4.thenRunAsync(r5);
        CompletableFuture cf6 = cf4.thenRunAsync(r6);

        CompletableFuture cf7 = cf4.thenRun(r7);

        CompletableFuture cf8 = cf6.thenRun(r8);


        System.out.println(cf2);

        // r1->r4->(r5异步,r6异步,r7同步)  r6结束->r8,r7结束->r3->r2
```
*此方法可以调用到CompletableFuture<T> postFire{a.postComplete()};<br/>*

```
ThreadPoolExecutor executorService1 = new ThreadPoolExecutor(10, 10,
                0L, TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<>(100));

        ThreadPoolExecutor executorService2 = new ThreadPoolExecutor(10, 10,
                0L, TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<>(100));

        Runnable r1 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r1 begin r1 begin r1 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r1 finish r1 finish r1 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r2 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r2 begin r2 begin r2 " + System.currentTimeMillis());
                    Thread.sleep(5000);
                    System.out.println("finish r2 finish r2 finish r2 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };


        Runnable r3 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r3 begin r3 begin r3 " + System.currentTimeMillis());
                    Thread.sleep(10000);
                    System.out.println("finish r3 finish r3 finish r3 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };


        Runnable r4 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r4 begin r4 begin r4 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r4 finish r4 finish r4 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r5 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r5 begin r5 begin r5 " + System.currentTimeMillis());
                    Thread.sleep(1);
                    System.out.println("finish r5 finish r5 finish r5 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        CompletableFuture cf1 = CompletableFuture.runAsync(r1,executorService1);

        CompletableFuture cf2 = cf1.thenRun(r2);
        CompletableFuture cf3 = cf1.thenRun(r3);
        CompletableFuture cf4 = cf1.thenRunAsync(r4);
        CompletableFuture cf5 = cf1.thenRunAsync(r5);


        System.out.println(cf2);

//        此方法可以调用到CompletableFuture<T> postFire{a.postComplete()};
```
*此方法可以调用到CompletableFuture<T> postFire{postComplete()};<br/>*

```
ThreadPoolExecutor executorService1 = new ThreadPoolExecutor(10, 10,
                0L, TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<>(100));

        ThreadPoolExecutor executorService2 = new ThreadPoolExecutor(10, 10,
                0L, TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<>(100));

        Runnable r1 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r1 begin r1 begin r1 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r1 finish r1 finish r1 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r2 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r2 begin r2 begin r2 " + System.currentTimeMillis());
                    Thread.sleep(5000);
                    System.out.println("finish r2 finish r2 finish r2 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };


        Runnable r3 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r3 begin r3 begin r3 " + System.currentTimeMillis());
                    Thread.sleep(10000);
                    System.out.println("finish r3 finish r3 finish r3 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };


        Runnable r4 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r4 begin r4 begin r4 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r4 finish r4 finish r4 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r5 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r5 begin r5 begin r5 " + System.currentTimeMillis());
                    Thread.sleep(1);
                    System.out.println("finish r5 finish r5 finish r5 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r6 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r6 begin r6 begin r6 " + System.currentTimeMillis());
                    Thread.sleep(3000);
                    System.out.println("finish r6 finish r6 finish r6 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r7 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r7 begin r7 begin r7 " + System.currentTimeMillis());
                    Thread.sleep(5000);
                    System.out.println("finish r7 finish r7 finish r7 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r8 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r8 begin r8 begin r8 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r8 finish r8 finish r8 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        CompletableFuture cf1 = CompletableFuture.runAsync(r1,executorService1);

        CompletableFuture cf2 = cf1.thenRun(r2);
        CompletableFuture cf3 = cf1.thenRun(r3);
        CompletableFuture cf4 = cf1.thenRun(r4);


        CompletableFuture cf5 = cf4.thenRunAsync(r5);
        CompletableFuture cf6 = cf4.thenRunAsync(r6);
        CompletableFuture cf7 = cf4.thenRunAsync(r7);

        CompletableFuture cf8 = cf6.thenRunAsync(r8);


        System.out.println(cf2);

//        此方法可以调用到CompletableFuture<T> postFire{postComplete()};

```
*r1->r3->r4->r5->r6->r7->r8->r9->r2<br/>*

```
ThreadPoolExecutor executorService1 = new ThreadPoolExecutor(10, 10,
                0L, TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<>(100));

        Runnable r1 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r1 begin r1 begin r1 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r1 finish r1 finish r1 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r2 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r2 begin r2 begin r2 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r2 finish r2 finish r2 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };


        Runnable r3 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r3 begin r3 begin r3 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r3 finish r3 finish r3 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };


        Runnable r4 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r4 begin r4 begin r4 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r4 finish r4 finish r4 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r5 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r5 begin r5 begin r5 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r5 finish r5 finish r5 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r6 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r6 begin r6 begin r6 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r6 finish r6 finish r6 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r7 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r7 begin r7 begin r7 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r7 finish r7 finish r7 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r8 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r8 begin r8 begin r8 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r8 finish r8 finish r8 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };

        Runnable r9 = new Runnable() {
            @Override
            public void run() {
                long start = System.currentTimeMillis();
                try {
                    System.out.println("begin r9 begin r9 begin r9 " + System.currentTimeMillis());
                    Thread.sleep(1000);
                    System.out.println("finish r9 finish r9 finish r9 " + System.currentTimeMillis());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("cost:" + (System.currentTimeMillis() -start));
            }
        };
        
        CompletableFuture cf1 = CompletableFuture.runAsync(r1,executorService1);

        CompletableFuture cf2 = cf1.thenRun(r2);
        CompletableFuture cf3 = cf1.thenRun(r3);


        CompletableFuture cf4 = cf3.thenRun(r4);
        CompletableFuture cf5 = cf3.thenRun(r5);


        CompletableFuture cf6 = cf5.thenRun(r6);
        CompletableFuture cf7 = cf5.thenRun(r7);

        CompletableFuture cf8 = cf7.thenRun(r8);
        CompletableFuture cf9 = cf7.thenRun(r9);


        System.out.println(cf2);

        // r1->r3->r4->r5->r6->r7->r8->r9->r2
        
```


### **其他**
CompletableFuture.allOf(...),CompletableFuture.anyOf(...),参数里面的cf管自己的顺序执行,没有先后顺序.<br/>
allof、anyof只是做结果判断,如果所有cf都有result了(allOf),或只要有一个cf有result(anyOf).
这里重要的是一个协调判断,最后都调用到BiCompletion进行判断(allOf(BiRelay),anyOf(OrRelay)).
其实也就是依赖关系，最后都依赖到同一个Completion





- **使用方法参考链接<br/>**
  [CompletableFuture 使用详解](https://www.jianshu.com/p/6bac52527ca4)  
  

  
转载请注明：[这不是一只猫的博客](http://1024.notacat.cn) » [点击阅读原文](https://1024.notacat.cn/2021/07/CompletableFuture/)


