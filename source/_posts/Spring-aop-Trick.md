---
title: Spring-aop的小把戏
date: 2019-12-11 10:56:15
tags: [Spring,AOP,trick]
---
**背景：**
   多核CPU的普及对于多线程是契机.但是异步使用不当就会造成"困扰".

**原因：**
BeanPostProcessor是一个工厂钩子，允许Spring框架在新创建Bean实例时对其进行定制化修改。例如：通过检查其标注的接口或者使用代理对其进行包裹。应用上下文会从Bean定义中自动检测出BeanPostProcessor并将它们应用到随后创建的任何Bean上。
普通Bean对象的工厂允许在程序中注册post-processors，应用到随后在本工厂中创建的所有Bean上。典型的场景如：post-processors使用postProcessBeforeInitialization方法通过特征接口或其他类似的方式来填充Bean；而为创建好的Bean创建代理则一般使用postProcessAfterInitialization方法。
BeanPostProcessor本身也是一个Bean，一般而言其实例化时机要早过普通的Bean，但是BeanPostProcessor也会依赖一些Bean，这就导致了一些Bean的实例化早于BeanPostProcessor，由此会导致一些问题

 **1. 问题场景**


----------


邮件异步发送可参考[程序猿DD的文章][1]
异步配置
```
@Slf4j
@EnableAsync
@Configuration
public class AsyncTaskPoolConfig extends AsyncConfigurerSupport {

    @Lazy
    @Resource
    private MailService mailService;

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("Hi-AsyncExecutor-");
        executor.setRejectedExecutionHandler(getRejectedExecutionHandler());
        //防止任务未执行完毕就关闭线程池
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);
        return executor;
    }


    private RejectedExecutionHandler getRejectedExecutionHandler() {
        return new SpringRejectedExecutionHandler();
    }

    class SpringRejectedExecutionHandler implements RejectedExecutionHandler {

        @Override
        public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
            if (!executor.isShutdown()) {
                r.run();
            }
            log.error("rejectedTask in async method");
            sendMail("发生线程池堵塞", "发生线程池堵塞.由主线程自己执行");
        }
    }
    private void sendMail(String setSubject, String content) {
        mailService.send();
    }

}
```

按理说这样的配置在直接或者间接发送邮件时候应该都是异步的才对.可是这样配置过后邮件发送就莫名的成了同步阻塞了.

**2、解决问题**


----------


2.1 spring async失效的表象原因
首先找到spring async失效的表象/直接原因.我们知道spring async使用Spring AOP和拦截器的方式拦截定义了特定标注的方法，然后执行特定逻辑。因此其实现依赖于动态代理机制auto-proxy，而经过初步调试发现，当被AsyncTaskPoolConfig依赖以后，mailService就不会被代理了，因此无从进入AOP的pointcut，也就是说AOP切面失效了！

2.2 从spring async的集成机制分析

为何没有被代理呢，我们先来确认一下正常情况下什么时候进行代理封装，根据Spring加载机制 BeanPostProcessor允许在Bean实例化的前后对其做一些特定的处理，比如代理。我们在BeanPostProcessor的实现类中发现了InstantiationAwareBeanPostProcessor、SmartInstantiationAwareBeanPostProcessor、AbstractAutoProxyCreator、InfrastructureAdvisorAutoProxyCreator等等。而反观@EnableAsync标注在启动的时候会@import AsyncConfigurationSelector，其selectImports方法会返回ProxyAsyncConfiguration和rg.springframework.scheduling.aspectj.AspectJAsyncConfiguration的全类名（我们定义了mode=AdviceMode.PROXY），也就是加载ProxyAsyncConfiguration。第一个的作用就是实例化AsyncAnnotationBeanPostProcessor便于进行addBeanPostProcessor操作。第二个的作用就是注册了AsyncAnnotationAdvisor和AnnotationAsyncExecutionInterceptor。
因此，当正常情况下，一个添加了spring async标注的bean会在创建后被InfrastructureAdvisorAutoProxyCreator基于advisor进行代理增强，代理后便可在拦截器AnnotationAsyncExecutionInterceptor中对其方法进行拦截，然后执行async相关逻辑。

所以第一怀疑就是mailService没有经过InfrastructureAdvisorAutoProxyCreator的代理增强。果然调试发现，被AsyncTaskPoolConfig依赖的情况下在MailService的Bean实例化时，用于处理该Bean的PostBeanProcessor明显比没被AsyncTaskPoolConfig依赖时少，并且不含有InfrastructureAdvisorAutoProxyCreator。



而且，被依赖时会多打出来一行信息：
```
INFO   o.s.c.s.PostProcessorRegistrationDelegate$BeanPostProcessorChecker - Bean 'mailService' of type [com.forest10.spring.boot.family.async.MailService] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
```
据此推断，可能是MailService启动实例化时机过早，导致的后面那些BeanPostProcessor们来没来得及实例化及注册

2.3 BeanPostProcessor启动阶段对其依赖的Bean造成的影响

BeanPostProcessor的启动时机。在AbstractBeanFactory中维护了BeanPostProcessor的列表：
```
private final List<BeanPostProcessor> beanPostProcessors = new ArrayList<BeanPostProcessor>();
```
并实现了ConfigurableBeanFactory定义的方法：
```
void addBeanPostProcessor(BeanPostProcessor beanPostProcessor);
```
第一阶段是在启动时调用过程会调用AbstractApplicationContext.refresh()，其中的prepareBeanFactory方法中注册了ApplicationContextAwareProcessor、ApplicationListenerDetector：
```
beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
```
其中对已经注册的BeanFactoryPostProcessors挨个调用其postProcessBeanFactory方法，其中有一个ConfigurationClassPostProcessor，其postProcessBeanFactory方法中注册了一个ImportAwareBeanPostProcessor：
```
beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
```
最后在registerBeanPostProcessors方法中调用
```
PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
```
在该方法中，首先注册BeanPostProcessorChecker：
```
beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));
```
它会在Bean创建完后检查可在当前Bean上起作用的BeanPostProcessor个数与总的BeanPostProcessor个数，如果起作用的个数少于总数，则报出上面那句信息。

然后分成三个阶段依次实例化并注册实现了PriorityOrdered的BeanPostProcessor、实现了Ordered的BeanPostProcessor、没实现Ordered的BeanPostProcessor

```
// Separate between BeanPostProcessors that implement PriorityOrdered,
        // Ordered, and the rest.
        List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
        List<BeanPostProcessor> internalPostProcessors = new ArrayList<BeanPostProcessor>();
        List<String> orderedPostProcessorNames = new ArrayList<String>();
        List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
        for (String ppName : postProcessorNames) {
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
                priorityOrderedPostProcessors.add(pp);
                if (pp instanceof MergedBeanDefinitionPostProcessor) {
                    internalPostProcessors.add(pp);
                }
            }
            else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
                orderedPostProcessorNames.add(ppName);
            }
            else {
                nonOrderedPostProcessorNames.add(ppName);
            }
        }
 
 
        // First, register the BeanPostProcessors that implement PriorityOrdered.
        sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
        registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);
 
 
        // Next, register the BeanPostProcessors that implement Ordered.
        List<BeanPostProcessor> orderedPostProcessors = new ArrayList<BeanPostProcessor>();
        for (String ppName : orderedPostProcessorNames) {
            BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
            orderedPostProcessors.add(pp);
            if (pp instanceof MergedBeanDefinitionPostProcessor) {
                internalPostProcessors.add(pp);
            }
        }
        sortPostProcessors(orderedPostProcessors, beanFactory);
        registerBeanPostProcessors(beanFactory, orderedPostProcessors);
 
 
        // Now, register all regular BeanPostProcessors.
        List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
        for (String ppName : nonOrderedPostProcessorNames) {
            BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
            nonOrderedPostProcessors.add(pp);
            if (pp instanceof MergedBeanDefinitionPostProcessor) {
                internalPostProcessors.add(pp);
            }
        }
        registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);
 
 
        // Finally, re-register all internal BeanPostProcessors.
        sortPostProcessors(internalPostProcessors, beanFactory);
        registerBeanPostProcessors(beanFactory, internalPostProcessors);
 
 
        // Re-register post-processor for detecting inner beans as ApplicationListeners,
        // moving it to the end of the processor chain (for picking up proxies etc).
        beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
```



需要注意的是，除了第一个阶段，其他阶段同一个阶段的BeanPostProcessor是在全部实例化完成以后才会统一注册到beanFactory的，因此，同一个阶段的BeanPostProcessor及其依赖的Bean在实例化的时候是无法享受到相同阶段但是先实例化的BeanPostProcessor的“服务”的，因为它们还没有注册。

从上面调试与源代码分析，BeanPostProcessor的实例化与注册分为四个阶段，第一阶段applicationContext内置阶段、第二阶段priorityOrdered阶段、第三阶段Ordered阶段、第四阶段nonOrdered阶段。而BeanPostProcessor同时也是Bean，其注册之前一定先实例化。而且是分批实例化和注册，也就是属于同一批的BeanPostProcesser全部实例化完成后，再全部注册，不存在先实例化先注册的问题。而在实例化的时候其依赖的Bean同样要先实例化。

因此导致一个结果就是，被PriorityOrderedBeanPostProcessor所依赖的Bean其初始化时无法享受到PriorityOrdered、Ordered、和nonOrdered的BeanPostProcessor的服务。而被OrderedBeanPostProcessor所依赖的Bean无法享受Ordered、和nonOrdered的BeanPostProcessor的服务。最后被nonOrderedBeanPostProcessor所依赖的Bean无法享受到nonOrderedBeanPostProcessor的服务。



由于AsyncAnnotationBeanPostProcessor的启动阶段是Ordered(下面代码取自EnableAsync)，因此我们需要确保没有任何priorityOrdered和Ordered的BeanPostProcessor直接或间接的依赖到MailService。

```
/**
     * Indicate the order in which the {@link AsyncAnnotationBeanPostProcessor}
     * should be applied.
     * <p>The default is {@link Ordered#LOWEST_PRECEDENCE} in order to run
     * after all other post-processors, so that it can add an advisor to
     * existing proxies rather than double-proxy.
     */
    int order() default Ordered.LOWEST_PRECEDENCE;
```


----------


 - **总结**

BeanPostProcessor的启动时机。分为四个阶段，第一阶段context内置阶段、第二阶段priorityOrdered阶段、第三阶段Ordered阶段、第四阶段nonOrdered阶段。
而BeanPostProcessor同时也是Bean，其注册之前一定先实例化。而且是分批实例化和注册，也就是属于同一批的BeanPostProcesser全部实例化完成后，再全部注册，不存在先实例化先注册的问题。而在实例化的时候其依赖的Bean同样要先实例化。
BeanPostProcessor实例化时，自动依赖注入根据类型获得需要注入的Bean时，会将某些符合条件的Bean（FactoryBean并且其FactoryBeanFactory已经实例化的）先实例化，如果此FacotryBean又依赖其他普通Bean，会导致该Bean提前启动，造成误伤（无法享受部分BeanPostProcessor的后处理，例如典型的auto-proxy）


  [1]: http://blog.didispace.com/springbootmailsender
