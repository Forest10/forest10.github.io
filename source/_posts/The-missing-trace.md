---
title: 消失的异常堆栈
date: 2019-12-10 14:24:35
tags: [JVM]
---

原文引用自:[消失的异常栈][1]
**最近在分析日志的时候发现有个日志发现了NPE，但是没有异常堆栈信息,只有java.lang.NullPointerException这一条信息,无法知道是从哪里抛出来的, 经过查找资料知道是JIT编译器对异常进行了优化，当代码中的某个位置抛出同一个异常很多次后,JIT服务端编译器(C2)会将其优化成抛出一个事先编译好的、类型匹配的异常,异常堆栈信息就看不到了。**

引用R大的一段话:

> HotSpot VM有个许多人觉得“匪夷所思”的优化，叫做fast
> throw：有些特定的隐式异常类型（NullPointerException、ArithmeticException（ /
> 0）之类）如果在代码里某个特定位置被抛出过多次的话，HotSpot Server Compiler（C2）会透明的决定用fast throw来优化这个抛出异常的地方——直接抛出一个事先分配好的、类型匹配的异常对象。这个对象的message和stack trace都被清空。抛出这个异常的速度是非常快，不但不用额外分配内存，而且也不用爬栈；但反面就是可能正好是需要知道哪里出问题的时候看不到stack trace了。从Sun JDK5开始要避免C2做这个优化还得额外传个VM参数：-XX:-OmitStackTraceInFastThrow。

**问题重现
测试代码如下,使用的是jdk1.8.0_144:**

    public static void main(String[] args) {
        for (int i = 0; i < 200_000; i++) {
            try {
                String s = null;
                s.toString();
            } catch (Exception e) {
                System.out.println(i + ":" + e.getStackTrace().length);
                if (e.getStackTrace().length == 0) {
                    System.out.println("stackTrace omit after " + i + " times");
                    e.printStackTrace();
                    break;
                }
            }
        }

    }

**测试结果是在调用115715次后JIT做了编译优化,在第115716次时异常堆栈看不到了,stackTrace长度为0:**

        
    115709:1
    115710:1
    115711:1
    115712:1
    115713:1
    115714:1
    115715:0
    stackTrace omit after 115715 times
    java.lang.NullPointerException

**解决办法**
**JVM提供了-XX:-OmitStackTraceInFastThrow这个虚拟机参数来告诉JIT编译器禁用这种异常fastThrow的优化,当然如果你使用-Xint参数后虚拟机运行在解释器模式也不会出现这个问题，但是禁用JIT会对整体的性能有影响,因此不建议使用-Xint参数,如果想看到具体的异常堆栈,推荐使用-XX:-OmitStackTraceInFastThrow参数。
那JVM为什么要对异常进行优化呢，这里就牵扯到另一个问题了,如果你在系统响应慢的时候分析过线程堆栈,可能遇到过线程耗在调用fillInStackTrace()这个native方法的时间非常长,fillInStackTrace()方法用来爬取线程的调用堆栈,我之前就遇到过Log4j打印日志非常慢的问题,结果抓取线程堆栈后发现线程都是卡在fillInStackTrace()这个native方法，如果有些使用场景不需要完整的调用堆栈时,建议重写fillInStackTrace()，让它直接return this，可以一定程度的提高系统的吞吐量。**

**fillInStackTrace优化**

> 我们知道所有的Exception和Error都是Throwable的子类，构造子类实例前都先调用父类的实例构造方法，我们看下Throwable类的源码就会发现在构造方法里调用了fillInStackTrace()方法:

   

     /**
         * Constructs a new throwable with {@code null} as its detail message.
         * The cause is not initialized, and may subsequently be initialized by a
         * call to {@link #initCause}.
         *
         * <p>The {@link #fillInStackTrace()} method is called to initialize
         * the stack trace data in the newly created throwable.
         */
        public Throwable() {
            fillInStackTrace();
        }
        
        public synchronized Throwable fillInStackTrace() {
            if (stackTrace != null || backtrace != null ) {
                //这里调用native的fillInStackTrace方法
                fillInStackTrace(0);
                stackTrace = UNASSIGNED_STACK;
            }
            return this;
        }

**可以看到当stackTrace不为null时需要调用native的fillInStackTrace()方法，那什么时候stackTrace变量为null呢，通过追踪源码可以发现Throwable类有多个重载的构造方法,其中有个方法可以传递一个writableStackTrace参数,当这个参数为false的时候stackTrace就为null，这时候就不会调用native的fillInStackTrace()方法去爬取线程堆栈，当然你也可以重写fillInStackTrace()方法，让他直接返回this,这样也可以避免爬栈,但是还是建议大家谨慎使用，毕竟需求时刻在变，说不定什么时候就需要这个堆栈来定位问题了。**


    protected Throwable(String message, Throwable cause,
                        boolean enableSuppression,
                        boolean writableStackTrace) {
        if (writableStackTrace) {
            fillInStackTrace();
        } else {
            stackTrace = null;
        }
        detailMessage = message;
        this.cause = cause;
        if (!enableSuppression)
            suppressedExceptions = null;
    }

**前面提到了Log4j打印日志慢的问题,那Log4j打印日志为什么也涉及到这个fillInStackTrace方法呢，对Log4j有研究过的同学应该知道如果Log4j配置文件里配置了%C(类全限定包名)、%F(文件名)、%M(打印日志的方法名称)和%L(行号)这几个用于定位调用者信息的pattern时,Log4J会先抛出一个异常出来，然后从异常堆栈中来获取调用者的信息,既然是抛异常出来必然涉及到调用native的fillInStackTrace方法来爬取线程堆栈,因此开启这些参数对系统的性能是有影响的。**


  [1]: https://www.ezlippi.com/blog/2018/02/the-missing-stacktrace.html