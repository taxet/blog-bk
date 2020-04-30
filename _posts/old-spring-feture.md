---
title: JAVA Future 与异步化
date: 2018-01-11 18:16:34
tags: 
- JAVA
- SPRING
category: CODING
---

你在超市结帐排队的时候会玩手机吗？

写程序也是一样。当一个进程等待一个需要很长时间处理的结果的时候也可以去做其他事。就像排队时玩手机一样。这种方法叫做异步调用。

异步调用最经典的语言就是javascript了。因为这是一个针对浏览器的语言，会经常有发送网络连接这样耗费很多时间的任务，而且javascript是单线程的，所以异步化对于它特别重要。

这里讨论一下java的异步化。和javascript不同，java的异步化一定是多线程的。

# Future
和字面上的意思一样，Future的意思就是给你一个未来。

（我的手中抓住了未来.jpg）

那么Future怎么用呢？先介绍一下一些基本元件

#### Callable
Runnable大家应该很熟悉了，就是一个进程，这里不多做介绍了，不知道的去看一下[其他资料](https://docs.oracle.com/javase/7/docs/api/java/lang/Runnable.html)。
Callable和Runnable类似，也是一个进程接口，不过和Runnable有几点区别：
1. Callable 的call()函数会返回一个值，而Runnable 的run()返回的是个void
2. Callable里可以抛出错误，Runnable不行

显然，Future里需要拿到一个进程的结果，Callable便当仁不让了。

#### Executor
executor这个词的意思是执行人，不过一般特指遗嘱的执行人。Java里的Executor也是这个意思，要去执行一个遗嘱（进程）

不过要注意的是Executor这个接口只有一个void execute(Runnable r)的函数，也就意味着它无法去调用callable来返回一个值，所以我么用它的子类接口ExecutorService。这个接口提供了Future<T> submit(Callable<T> c)这个函数可以用于返回一个future。

准备都介绍完了，接下来看代码
```java
ExecutorService executor = Executors.newFixedThreadPool(1);
Future<String> f = executor.submit(
        new Callable<String>(){
            @Override
            public String call() throws Exception {
                System.out.println("task started!");
                Thread.sleep(3000);
                System.out.println("task finished!");
                return "hello";
            }
        }
);
System.out.println(f.get());
System.out.println("main thread is blocked");
executor.shutdown();
```
之后运行可以看到结果
```
task started!
#这里会等三秒
task finished!
hello
main thread is blocked
```

这就是Future的基本操作
另外，Java8里可以吧Callable写成lambda的形式：
```ExecutorService executor = Executors.newFixedThreadPool(1);
Future<String> f = executor.submit(
        () -> {
            System.out.println("task started!");
            Thread.sleep(3000);
            System.out.println("task finished!");
            return "hello";
        }
);
System.out.println(f.get());
System.out.println("main thread is blocked");
executor.shutdown();
```
一个效果。

不过这并没有异步化啊。

所以我么要一个新东西来作异步化

# CompletableFuture
Completable Future是Java8新提供的一种特殊的Future。看字面上很难理解对吧？它在Javascript里面叫Promise...

CompletableFuture里提供了各种thenxxx的函数，其中还提供了很多thenxxx的函数，和Promise的then用法类似。

代码：
```java
ExecutorService executor = Executors.newFixedThreadPool(2);
CompletableFuture<String> future = CompletableFuture.supplyAsync(
        () -> {
            System.out.println("task started!");
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "task finished!";
            }, executor);

future.thenAccept(e -> System.out.println(e + " ok"));

System.out.println("main thread is running");
executor.shutdown();
```

结果：
```
task started!
main thread is running
#这里会等三秒
task finished! ok
```

完美~