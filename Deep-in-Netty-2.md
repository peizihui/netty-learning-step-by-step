# 3. NioEventLoop

**问题**

- 默认情况下，Netty服务端起多少线程？何时启动？
- Netty是如果解决jdk空转轮询bug的？ （问题都不明白）
- Netty如何保证异步串行无锁化？



# 3.1 NioEventLoop 创建

![image-20200426191054733](http://q8xc9za4f.bkt.clouddn.com/cloudflare/image-20200426191054733.png)







# 3.2 NioEventLoop 启动



```
// 1代表
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
// 不传默认为0
EventLoopGroup workerGroup = new NioEventLoopGroup();
```







```
// io.netty.util.concurrent.MultithreadEventExecutorGroup

protected MultithreadEventExecutorGroup(int nThreads, Executor executor, EventExecutorChooserFactory chooserFactory, Object... args) {
    this.terminatedChildren = new AtomicInteger();
    this.terminationFuture = new DefaultPromise(GlobalEventExecutor.INSTANCE);
    if (nThreads <= 0) {
        throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
    } else {
    
    //Step 1. 创建线程创建器
    	// 每次执行任务都会创建一个线程实体；	
        if (executor == null) {
            executor = new ThreadPerTaskExecutor(this.newDefaultThreadFactory());
        }

        this.children = new EventExecutor[nThreads];

	//for start 
	// Step2  构造NioEventLoop   		
        int j;
        for(int i = 0; i < nThreads; ++i) {
            boolean success = false;
            boolean var18 = false;

            try {
                var18 = true;
                this.children[i] = this.newChild((Executor)executor, args);
                success = true;
                var18 = false;
            } catch (Exception var19) {
                throw new IllegalStateException("failed to create a child event loop", var19);
            } finally {
                if (var18) {
                    if (!success) {
                        int j;
                        for(j = 0; j < i; ++j) {
                            this.children[j].shutdownGracefully();
                        }

                        for(j = 0; j < i; ++j) {
                            EventExecutor e = this.children[j];

                            try {
                                while(!e.isTerminated()) {
                                    e.awaitTermination(2147483647L, TimeUnit.SECONDS);
                                }
                            } catch (InterruptedException var20) {
                                Thread.currentThread().interrupt();
                                break;
                            }
                        }
                    }

                }
            }

            if (!success) {
                for(j = 0; j < i; ++j) {
                    this.children[j].shutdownGracefully();
                }

                for(j = 0; j < i; ++j) {
                    EventExecutor e = this.children[j];

                    try {
                        while(!e.isTerminated()) {
                            e.awaitTermination(2147483647L, TimeUnit.SECONDS);
                        }
                    } catch (InterruptedException var22) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            }
        }
        // for end;
        

		//Step 3. chooserFactory.newChooser 创建线程选择器
        this.chooser chooserFactory.newChooser(this.children);
        FutureListener<Object> terminationListener = new FutureListener<Object>() {
            public void operationComplete(Future<Object> future) throws Exception {
                if (MultithreadEventExecutorGroup.this.terminatedChildren.incrementAndGet() == MultithreadEventExecutorGroup.this.children.length) {
                    MultithreadEventExecutorGroup.this.terminationFuture.setSuccess((Object)null);
                }

            }
        };
        EventExecutor[] arr$ = this.children;
        j = arr$.length;

        for(int i$ = 0; i$ < j; ++i$) {
            EventExecutor e = arr$[i$];
            e.terminationFuture().addListener(terminationListener);
        }

        Set<EventExecutor> childrenSet = new LinkedHashSet(this.children.length);
        Collections.addAll(childrenSet, this.children);
        this.readonlyChildren = Collections.unmodifiableSet(childrenSet);
    }
}
```

### 3.1.1 ThreadPerTaskExecutor

- 每次执行任务都会创建一个线程实体；

```
     executor = new ThreadPerTaskExecutor(this.newDefaultThreadFactory());
```



​		NioEventLoop线程命名规则nioEventLoop-1-xx;



### 3.1.2  newchild()

- 保存线程执行器ThreadPerTaskExecutor

- 创建一个MpscQueue

- 创建一个selector

  ```
  this.chooser = chooserFactory.newChooser(this.children);
  ```

- ```
  // io.netty.util.concurrent.DefaultEventExecutorChooserFactory
  
  
  public EventExecutorChooser newChooser(EventExecutor[] executors) {
          return (EventExecutorChooser)(isPowerOfTwo(executors.length) ? 
  // 优化
  new DefaultEventExecutorChooserFactory.PowerOfTowEventExecutorChooser(executors) :
  // 普通        
  new DefaultEventExecutorChooserFactory.GenericEventExecutorChooser(executors));
      }
      
  
  
  
  
  ```



**判断是否是二的幂**

```
private static boolean isPowerOfTwo(int val) {
    return (val & -val) == val;
}
```



**PowerOfTowEventExecutorChooser() 是如何优化的**;

- & 运算比%运算更高效；



![image-20200426211600047](http://q8xc9za4f.bkt.clouddn.com/cloudflare/image-20200426211600047.png)



### 3.1.3 NioEventLoop 启动



![image-20200427073450761](http://q8xc9za4f.bkt.clouddn.com/cloudflare/image-20200427073450761.png)



![image-20200427073605397](http://q8xc9za4f.bkt.clouddn.com/cloudflare/image-20200427073605397.png)



- 服务端启动绑定端口
- 新连接接入通过chooser绑定一个NioEventLoop 



#### 3.1.3.1 服务端启动绑定端口





![image-20200426212051546](http://q8xc9za4f.bkt.clouddn.com/cloudflare/image-20200426212051546.png)







```

// io.netty.bootstrap.AbstractBootstrap


private static void doBind0(final ChannelFuture regFuture, final Channel channel, final SocketAddress localAddress, final ChannelPromise promise) {

// 调用channel.eventLoop,在netty启动过程中，通过register()方法绑定上去




//Step 3.1.3 -2   new Runnable 其实是一个Task，
 channel.eventLoop().execute(new Runnable() {
        public void run() {
            if (regFuture.isSuccess()) {
 // Task 做什么事情，其实就是绑定端口           
 channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }

        }
    });


}
```



**紧跟上文代码Step 3.1.3 -2   new Runnable 其实是一个Task**

**io.netty.util.concurrent.SingleThreadEventExecutor**

****

```
//io.netty.util.concurrent.SingleThreadEventExecutor

public void execute(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    } else {
      //Step 3.1.3 -2-1 判断是不是·
      //  inEventLoop 线程没有创建，
        boolean inEventLoop = this.inEventLoop();
        if (inEventLoop) {
            this.addTask(task);
        } else {
        //Step 3.1.3 -2-2 程开始创建；
            this.startThread();
            this.addTask(task);
            if (this.isShutdown() && this.removeTask(task)) {
                reject();
            }
        }

        if (!this.addTaskWakesUp && this.wakesUpForTask(task)) {
            this.wakeup(inEventLoop);
        }

    }
}
```

**io.netty.util.concurrent.SingleThreadEventExecutor**

**startThread    // 紧跟 Step 3.1.3 -2-2 程开始创建**

```
// 判断当前线程是否未创建
private void startThread() {
// 如果未启动，通过CS的方法进行线程的启动；
        if (STATE_UPDATER.get(this) == 1 && STATE_UPDATER.compareAndSet(this, 1, 2)) {
        // 创建线程；
        //  紧跟 Step 3.1.3 -2-3 开始新的线程
            this.doStartThread();
        }

    }
```





**io.netty.util.concurrent.SingleThreadEventExecutor**

```
private void doStartThread() {
		// 断言，启动线程，线程是以前没有创建的；
    assert this.thread == null;
    
    // executor ,前面创建Nio
	// 
    this.executor.execute(new Runnable() {
        public void run() {
            SingleThreadEventExecutor.this.thread = Thread.currentThread();
            if (SingleThreadEventExecutor.this.interrupted) {
                SingleThreadEventExecutor.this.thread.interrupt();
            }

            boolean success = false;
            SingleThreadEventExecutor.this.updateLastExecutionTime();
            boolean var112 = false;

            int oldState;
            label1686: {
                try {
                    var112 = true;
                    SingleThreadEventExecutor.this.run();
                    success = true;
                    var112 = false;
                    break label1686;
                } catch (Throwable var119) {
                    SingleThreadEventExecutor.logger.warn("Unexpected exception from an event executor: ", var119);
                    var112 = false;
                } finally {
                    if (var112) {
                        int oldStatex;
                        do {
                            oldStatex = SingleThreadEventExecutor.STATE_UPDATER.get(SingleThreadEventExecutor.this);
                        } while(oldStatex < 3 && !SingleThreadEventExecutor.STATE_UPDATER.compareAndSet(SingleThreadEventExecutor.this, oldStatex, 3));

                        if (success && SingleThreadEventExecutor.this.gracefulShutdownStartTime == 0L) {
                            SingleThreadEventExecutor.logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " + SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must be called " + "before run() implementation terminates.");
                        }

                        try {
                            while(!SingleThreadEventExecutor.this.confirmShutdown()) {
                            }
                        } finally {
                            try {
                                SingleThreadEventExecutor.this.cleanup();
                            } finally {
                                SingleThreadEventExecutor.STATE_UPDATER.set(SingleThreadEventExecutor.this, 5);
                                SingleThreadEventExecutor.this.threadLock.release();
                                if (!SingleThreadEventExecutor.this.taskQueue.isEmpty()) {
                                    SingleThreadEventExecutor.logger.warn("An event executor terminated with non-empty task queue (" + SingleThreadEventExecutor.this.taskQueue.size() + ')');
                                }

                                SingleThreadEventExecutor.this.terminationFuture.setSuccess((Object)null);
                            }
                        }

                    }
                }

                do {
                    oldState = SingleThreadEventExecutor.STATE_UPDATER.get(SingleThreadEventExecutor.this);
                } while(oldState < 3 && !SingleThreadEventExecutor.STATE_UPDATER.compareAndSet(SingleThreadEventExecutor.this, oldState, 3));

                if (success && SingleThreadEventExecutor.this.gracefulShutdownStartTime == 0L) {
                    SingleThreadEventExecutor.logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " + SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must be called " + "before run() implementation terminates.");
                }

                try {
                    while(!SingleThreadEventExecutor.this.confirmShutdown()) {
                    }

                    return;
                } finally {
                    try {
                        SingleThreadEventExecutor.this.cleanup();
                    } finally {
                        SingleThreadEventExecutor.STATE_UPDATER.set(SingleThreadEventExecutor.this, 5);
                        SingleThreadEventExecutor.this.threadLock.release();
                        if (!SingleThreadEventExecutor.this.taskQueue.isEmpty()) {
                            SingleThreadEventExecutor.logger.warn("An event executor terminated with non-empty task queue (" + SingleThreadEventExecutor.this.taskQueue.size() + ')');
                        }

                        SingleThreadEventExecutor.this.terminationFuture.setSuccess((Object)null);
                    }
                }
            }

            do {
                oldState = SingleThreadEventExecutor.STATE_UPDATER.get(SingleThreadEventExecutor.this);
            } while(oldState < 3 && !SingleThreadEventExecutor.STATE_UPDATER.compareAndSet(SingleThreadEventExecutor.this, oldState, 3));

            if (success && SingleThreadEventExecutor.this.gracefulShutdownStartTime == 0L) {
                SingleThreadEventExecutor.logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " + SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must be called " + "before run() implementation terminates.");
            }

            try {
                while(!SingleThreadEventExecutor.this.confirmShutdown()) {
                }
            } finally {
                try {
                    SingleThreadEventExecutor.this.cleanup();
                } finally {
                    SingleThreadEventExecutor.STATE_UPDATER.set(SingleThreadEventExecutor.this, 5);
                    SingleThreadEventExecutor.this.threadLock.release();
                    if (!SingleThreadEventExecutor.this.taskQueue.isEmpty()) {
                        SingleThreadEventExecutor.logger.warn("An event executor terminated with non-empty task queue (" + SingleThreadEventExecutor.this.taskQueue.size() + ')');
                    }

                    SingleThreadEventExecutor.this.terminationFuture.setSuccess((Object)null);
                }
            }

        }
    });
}


```

####3.1.3.2 新连接接入通过chooser绑定一个NioEventLoop





#3.3 NioEventLoop 执行逻辑

### 3.3.1  NioEventLoop.run(); 

![image-20200426230319128](http://q8xc9za4f.bkt.clouddn.com/cloudflare/image-20200426230319128.png)

```
run() -> for(;;)
```



**io.netty.channel.nio.NioEventLoop**



```
 protected void run() {
        while(true) {
            while(true) {
                try {
                    switch(this.selectStrategy.calculateStrategy(this.selectNowSupplier, this.hasTasks())) {
                    case -2:
                        continue;
                    case -1:
                    // 轮询注册到selector 的IO事件
                    // 标注进行select操作，而且是未唤醒状态；
                        this.select(this.wakenUp.getAndSet(false));
                        if (this.wakenUp.get()) {
                            this.selector.wakeup();
                        }
                    default:
                        this.cancelledKeys = 0;
                        this.needsToSelectAgain = false;
                        //  this.ioRatio 没有设置默认50；
                        int ioRatio = this.ioRatio;
                       // 
                        if (ioRatio == 100) {
                            try {
                            // 处理IO相关的逻辑
                                this.processSelectedKeys();
                            } finally {
                            // 处理外部线程扔到taskqu 里面的任务；
                                this.runAllTasks();
                            }
                        } else {
                            long ioStartTime = System.nanoTime();
                            boolean var13 = false;

                            try {
                                var13 = true;
                                this.processSelectedKeys();
                                var13 = false;
                            } finally {
                                if (var13) {
                                    long ioTime = System.nanoTime() - ioStartTime;
                                    this.runAllTasks(ioTime * (long)(100 - ioRatio) / (long)ioRatio);
                                }
                            }

                            long ioTime = System.nanoTime() - ioStartTime;
                            this.runAllTasks(ioTime * (long)(100 - ioRatio) / (long)ioRatio);
                        }
                    }
                } catch (Throwable var21) {
                    handleLoopException(var21);
                }

                try {
                    if (this.isShuttingDown()) {
                        this.closeAll();
                        if (this.confirmShutdown()) {
                            return;
                        }
                    }
                } catch (Throwable var18) {
                    handleLoopException(var18);
                }
            }
        }
    }


```





## 3.4 select()方法执行逻辑

- deadline 以及任务穿插逻辑处理；

- 阻塞式select

- 避免jdk空轮询的bug

  

### 3.4.1 deadline 以及任务穿插逻辑处理；



```
 protected void run() {
        while(true) {
            while(true) {
                try {
                    switch(this.selectStrategy.calculateStrategy(this.selectNowSupplier, this.hasTasks())) {
                    case -2:
                        continue;
                    case -1:
                    // 轮询注册到selector 的IO事件
                    // 标注进行select操作，而且是未唤醒状态；
                        this.select(this.wakenUp.getAndSet(false));
                        if (this.wakenUp.get()) {
                            this.selector.wakeup();
                        }
                    default:
                        this.cancelledKeys = 0;
                        this.needsToSelectAgain = false;
                        //  this.ioRatio 没有设置默认50；
                        int ioRatio = this.ioRatio;
                       // 
                        if (ioRatio == 100) {
                            try {
                            // 处理IO相关的逻辑
                                this.processSelectedKeys();
                            } finally {
                            // 处理外部线程扔到taskqu 里面的任务；
                                this.runAllTasks();
                            }
                        } else {
```

**总结** 以上代码注意两点

```
// 轮询注册到selector的IO事件
// 标注进行select操作，而且是未唤醒状态；
```



```
// io.netty.channel.nio.NioEventLoop;


private void select(boolean oldWakenUp) throws IOException {
    Selector selector = this.selector;

    try {
        int selectCnt = 0;
        long currentTimeNanos = System.nanoTime();
        //  delayNanos 计算第一个定时任务执行的时间；
        long selectDeadLineNanos = currentTimeNanos + this.delayNanos(currentTimeNanos);

	   // 计算当前是否超时，如果超时，一次没有进行select的话，
        while(true) {
            long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
            // 如果超时的，一次没有select的话，就进行一次非阻塞的select方法；
            if (timeoutMillis <= 0L) {
                if (selectCnt == 0) {
                    selector.selectNow();
                    selectCnt = 1;
                }
                break;
            }

			// 
            if (this.hasTasks() && this.wakenUp.compareAndSet(false, true)) {
                selector.selectNow();
                selectCnt = 1;
                break;
            }

            int selectedKeys = selector.select(timeoutMillis);
            ++selectCnt;
            if (selectedKeys != 0 || oldWakenUp || this.wakenUp.get() || this.hasTasks() || this.hasScheduledTasks()) {
                break;
            }

            if (Thread.interrupted()) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurely because Thread.currentThread().interrupt() was called. Use NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
                }

                selectCnt = 1;
                break;
            }

            long time = System.nanoTime();
            if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                selectCnt = 1;
            } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 && selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
                logger.warn("Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.", selectCnt, selector);
                this.rebuildSelector();
                selector = this.selector;
                selector.selectNow();
                selectCnt = 1;
                break;
            }

            currentTimeNanos = time;
        }

        if (selectCnt > 3 && logger.isDebugEnabled()) {
            logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.", selectCnt - 1, selector);
        }
    } catch (CancelledKeyException var13) {
        if (logger.isDebugEnabled()) {
            logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?", selector, var13);
        }
    }

}
```



### 3.4.2 阻塞式select







### 3.4.3 避免jdk空轮询的bug