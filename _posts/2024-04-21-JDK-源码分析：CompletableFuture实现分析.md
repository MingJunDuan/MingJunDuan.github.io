---
layout: post
title: "JDK-源码分析：CompletableFuture实现分析"
author: "Inela"
---

​	CompletableFuture是Doug Lea在JDK1.8引入的，解决了FutureTask阻塞调用、多个Task依赖处理的痛点。

​	CompletableFuture源码中下面两个核心的关键属性result和stack，关于这两个属性也有核心的注释，result可能是返回的结果集，也可能是包装的AltResult，stack这个属性暴露出了CompletableFuture的整体的结构是一个栈。

```java
volatile Object result;       // Either the result or boxed AltResult
volatile Completion stack;    // Top of Treiber stack of dependent actions
```

​	测试例子如下，结果是"1 2 3"

```java
@Test
public void test1() throws ExecutionException, InterruptedException {
  CompletableFuture<String> base = new CompletableFuture<>();
  CompletableFuture<String> future1 = base.thenApply(s -> s + " 2");
  CompletableFuture<String> future2 = future1.thenApply(s -> s + " 3");
  base.complete("1");
  System.out.println(future2.get());
}
```



​	Completion是个抽象类，其实现子类很多，它有个重要的属性next是Completion，可以看出是个链表，next指向下一个引用，而CompletableFuture中的stack永远指向栈顶

```java
abstract static class Completion extends ForkJoinTask<Void>
    implements Runnable, AsynchronousCompletionTask {
    volatile Completion next;      // Treiber stack link

    /**
     * Performs completion action if triggered, returning a
     * dependent that may need propagation, if one exists.
     *
     * @param mode SYNC, ASYNC, or NESTED
     */
    abstract CompletableFuture<?> tryFire(int mode);

    /** Returns true if possibly still triggerable. Used by cleanStack. */
    abstract boolean isLive();

    public final void run()                { tryFire(ASYNC); }
    public final boolean exec()            { tryFire(ASYNC); return true; }
    public final Void getRawResult()       { return null; }
    public final void setRawResult(Void v) {}
```

​	thenApply有三个方法，以Async结尾的表示异步执行，如果传入Executor则以指定线程池执行，否则默认使用的线程池是ForkJoinPool

```java
public <U> CompletableFuture<U> thenApply(
    Function<? super T,? extends U> fn) {
    return uniApplyStage(null, fn);
}

public <U> CompletableFuture<U> thenApplyAsync(
    Function<? super T,? extends U> fn) {
    return uniApplyStage(asyncPool, fn);
}

public <U> CompletableFuture<U> thenApplyAsync(
    Function<? super T,? extends U> fn, Executor executor) {
    return uniApplyStage(screenExecutor(executor), fn);
}
```

​	thenApply的实现如下

```java
public <U> CompletableFuture<U> thenApply(
    Function<? super T,? extends U> fn) {
    return uniApplyStage(null, fn);
}

private <V> CompletableFuture<V> uniApplyStage(Executor e, Function<? super T,? extends V> f) {
    if (f == null) throw new NullPointerException();
    1.创建一个新的CompletableFuture对象
    CompletableFuture<V> d =  new CompletableFuture<V>();
  	由于d是新创建的，属性result值是null，d.uniApply方法会返回false
    if (e != null || !d.uniApply(this, f, null)) {
        2. 构建UniApply e代表线程池 d 代表新的CompletableFuture this 代表当前,f 代表方法 这个时候 UniApply 内部的所有的引用都处于为null的状态
        UniApply<T,V> c = new UniApply<T,V>(e, d, this, f);
        3. c其实就是Completion对象，被push到栈中
        push(c);
        4. 尝试执行c
        c.tryFire(SYNC);
    }
    5. 这个d会一直返回到调用thenApply的地方，后续的链式调用会作用在这个d上面
    return d;
}

final void push(UniCompletion<?,?> c) {
  if (c != null) {
    由于base中的result是null，所以将c入栈
    while (result == null && !tryPushStack(c))
      lazySetNext(c, null); // clear on failure
  }
}

final boolean tryPushStack(Completion c) {
  Completion h = stack;
  将c的next指向h，而此时h是null（base是最先创建的，它的stack为null）
  lazySetNext(c, h);
  将base中的stack指向c
  return UNSAFE.compareAndSwapObject(this, STACK, h, c);
}

static void lazySetNext(Completion c, Completion next) {
  UNSAFE.putOrderedObject(c, NEXT, next);
}

@SuppressWarnings("serial")
static final class UniApply<T,V> extends UniCompletion<T,V> {
    Function<? super T,? extends V> fn;
    UniApply(Executor executor, CompletableFuture<V> dep,CompletableFuture<T> src,Function<? super T,? extends V> fn) {
        2.1 向上执行
        super(executor, dep, src); this.fn = fn;
    }
}

abstract static class UniCompletion<T,V> extends Completion {
    Executor executor;                 // executor to use (null if none)
  	//依赖哪个先执行完成
    CompletableFuture<V> dep;          // the dependent to complete
    CompletableFuture<T> src;          // source for action

    UniCompletion(Executor executor, CompletableFuture<V> dep,CompletableFuture<T> src) {
        2.2 dep就是新创建的d  src就是当前的this，即base
        this.executor = executor; this.dep = dep; this.src = src;
    }
}
```

​	接下来分析4处的c.tryFire(SYNC)， SYNC   =  0， ASYNC  =  1， NESTED = -1；

```java
static final class UniApply<T,V> extends UniCompletion<T,V> {
    Function<? super T,? extends V> fn;
    UniApply(Executor executor, CompletableFuture<V> dep,
             CompletableFuture<T> src,
             Function<? super T,? extends V> fn) {
        super(executor, dep, src); this.fn = fn;
    }
    final CompletableFuture<V> tryFire(int mode) {
        CompletableFuture<V> d; CompletableFuture<T> a;
        此时dep是null
        if ((d = dep) == null ||
            当前依赖的src没有结果，即src对应的CompletableFuture的result是null，会返回null
            !d.uniApply(a = src, fn, mode > 0 ? null : this))
            直接返回null
            return null;
        dep = null; src = null; fn = null;
        return d.postFire(a, mode);
    }
}
```

​	uniApply方法实现如下

```java
final <S> boolean uniApply(CompletableFuture<S> a,
                           Function<? super S,? extends T> f,
                           UniApply<S,T> c) {
    Object r; Throwable x;
    如果a(也是c中的src)没有准备完成，那result是空，这里就会直接返回false
    if (a == null || (r = a.result) == null || f == null)
        return false;
    tryComplete: if (result == null) {
        if (r instanceof AltResult) {
            if ((x = ((AltResult)r).ex) != null) {
                completeThrowable(x, r);
                break tryComplete;
            }
            r = null;
        }
        try {
            if (c != null && !c.claim())
                return false;
            @SuppressWarnings("unchecked") S s = (S) r;
            completeValue(f.apply(s));
        } catch (Throwable ex) {
            completeThrowable(ex);
        }
    }
    return true;
}
```

​	最终形成的有依赖关系的图如下：

![CompletableFuture1](https://github.com/MingJunDuan/mingjunduan.github.io/raw/main/images/mjduan/2024-04-21/CompletableFuture1.png)

​	Debug的图

![CompletableFuture1单元测试结果](https://github.com/MingJunDuan/mingjunduan.github.io/raw/main/images/mjduan/2024-04-21/CompletableFuture1单元测试结果.png)

​	当执行到单元测试的base.complete("1")时候，也就是complete方法，实现如下：

1. completeValue方法将result设置为value
2. postComplete方法，会触发一些列的计算，将先前计算好的结果作为参数传递给后面的

```java
public boolean complete(T value) {
    boolean triggered = completeValue(value);
    postComplete();
    return triggered;
}

final boolean completeValue(T t) {
  return UNSAFE.compareAndSwapObject(this, RESULT, null,(t == null) ? NIL : t);
}

final void postComplete() {
    1. this表示当前的CompletableFuture, 也就是我们base
    CompletableFuture<?> f = this; Completion h;
    2. 判断stack是否为空  或者如果f的栈为空且不是this则重置
    while ((h = f.stack) != null ||
            (f != this && (h = (f = this).stack) != null)) {
        CompletableFuture<?> d; Completion t;
        3. CAS出栈
        if (f.casStack(h, t = h.next)) {
            由于t=h.next是null，不会进到if中
            if (t != null) {
                if (f != this) {
                    pushStack(h);
                    continue;
                }
                h.next = null;    6. detach
            }
            h.tryFire方法，回到单元测试，base中的result是1，feture1会从src中获取result，且传递给feture1中的Funtion做参数计算出result2，接着调用调用feture2的postFire(a, mode)
            f = (d = h.tryFire(NESTED)) == null ? this : d;  7.调用tryFire
        }
    }
}
```





