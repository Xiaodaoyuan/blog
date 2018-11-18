---
title: springboot线程池优雅关闭
date: 2018-09-27 16:46:18
categories: web
tags: springboot
---

在我们停止springboot项目时，我们希望线程池中的任务能够继续执行完再完全停掉服务。一般有两种做法：

#### 线程池配置参数
在spring应用中，如果需要
停止服务，而线程池没有优雅的关闭，就会造成线程池中的任务被强行停止，导致部分任务执行失败。我们只需要在配置线程池时增加两个参数即可：
- waitForTasksToCompleteOnShutdown
- awaitTerminationSeconds
<!--more-->
具体代码如下：

```
    @Bean
    public ThreadPoolTaskExecutor treadPool() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(10);
        executor.setKeepAliveSeconds(60);
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(30);
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
```

---
#### 使用ApplicationListener<ContextClosedEvent>监听关闭事件

```
@Component
public class MyContextClosedHandler implements ApplicationListener<ContextClosedEvent>{
    @Autowired
    private ThreadPoolTaskExecutor executor;

    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        shutdownAndAwaitTermination(executor.getThreadPoolExecutor());
    }

    private void shutdownAndAwaitTermination(ExecutorService pool) {
        pool.shutdown(); // Disable new tasks from being submitted
        try {
            // Wait a while for existing tasks to terminate
            if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
                pool.shutdownNow(); // Cancel currently executing tasks
                // Wait a while for tasks to respond to being cancelled
                if (!pool.awaitTermination(30, TimeUnit.SECONDS))
                    System.err.println("Pool did not terminate");
            }
        } catch (InterruptedException ie) {
            // (Re-)Cancel if current thread also interrupted
            pool.shutdownNow();
            // Preserve interrupt status
            Thread.currentThread().interrupt();
        }
    }
}
```
---
