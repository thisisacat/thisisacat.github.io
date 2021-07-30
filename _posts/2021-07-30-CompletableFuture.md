---
layout: post
title: CompletableFuture
date: 2021-07-30
tags: CompletableFuture    
---
*Doug Lea 果然是大神中的大神。*
### **结论** 
**CompletableFuture 更多的是体现一种依赖关系src,dep,snd**

### **源码分析**

mode

```
 static final int SYNC   =  0; //当前线程触发
 static final int ASYNC  =  1; //线程池触发
    {
    	public final void run()                { tryFire(ASYNC); }
        public final boolean exec()            { tryFire(ASYNC); return true; }
    }
    static final int NESTED = -1;//postComplete触发
   
    mode的不同,影响在d.uniRun(a = src, fn, mode > 0 ? null : this)),postFire()等
```

postFire</br>
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

push</br>
注意这里 是采用头插法的形式

```
 final void push(UniCompletion<?,?> c) {
        if (c != null) {
            while (result == null && !tryPushStack(c))
                lazySetNext(c, null); // clear on failure
        }
    }
```

tryFire

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

uniRun

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

claim

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

postFire</br>
tryFire 和 postFire里面的两个return 关系到postComplete</br>
根据mode的不同，返回this还是null,从而影响postComplete方法里面f的返回值,f的不同影响h的取值，是src的CompletableFuture还是dep的CompletableFuture</br>

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

postComplete</br>

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

大致的执行逻辑是 stack(dep) 为一个单元执行完，然后转下一个stack(dep)</br>
![](/images/posts/CompletableFuture/process.png)<br/>


### **其他**
CompletableFuture.allOf(...),CompletableFuture.anyOf(...),参数里面的cf管自己的顺序执行,没有先后顺序,allof、anyof只是做结果判断，如果所有cf都有result了(allOf),只要有一个cf有result(anyOf）,这里重要的是一个协调判断最后都调用到BiCompletion进行判断(allOf(BiRelay),anyOf(OrRelay))





- **使用方法参考链接<br/>**
  [CompletableFuture 使用详解](https://www.jianshu.com/p/6bac52527ca4)  
  

  
转载请注明：[这不是一只猫的博客](http://1024.notacat.cn) » [点击阅读原文](https://1024.notacat.cn/2021/07/CompletableFuture/)


