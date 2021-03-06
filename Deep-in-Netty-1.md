

**定义作用** ：

- 异步实际驱动框架，用于快速开发高性能服务端和客户端

- 封装了JDK底层BIO和NIO模型，提供高度可用的API;
- 自带编解码器解决拆包粘包问题，用户只用关心业务逻辑
- 精心设计的reactor线程模型支持高并发海量连接
- 自带各种协议栈

https://netty.io/



![image-20200426103628763](http://q8xc9za4f.bkt.clouddn.com/cloudflare/image-20200426103628763.png)

    实际应用：

![image-20200426131400045](http://q8xc9za4f.bkt.clouddn.com/cloudflare/image-20200426131400045.png)



#1. Netty 基本组件



![image-20200426134050399](http://q8xc9za4f.bkt.clouddn.com/cloudflare/image-20200426134050399.png)

# 2. Netty 服务端启动问题；

## 2.1 服务端的socket在哪儿初始化？

### 2.1.1 创建channel 过程；

服务端；参考ch03代码；server

```
     
       ServerBootstrap b = new ServerBootstrap();
       // channel(NioServerSocketChannel.class)  11；  
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childOption(ChannelOption.TCP_NODELAY, true)
                    .childAttr(AttributeKey.newInstance("childAttr"), "childAttrValue")
                    .handler(new ServerHandler())
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) {
                            ch.pipeline().addLast(new AuthHandler());
                            //..

                        }
                    });


     
     
     
     ChannelFuture f = b.bind(8888).sync();
```

```
#  io.netty.bootstrap.AbstractBootstrap;  


  public ChannelFuture bind(int inetPort) {
        return this.bind(new InetSocketAddress(inetPort));
    }
    
    
    
    
    private ChannelFuture doBind(final SocketAddress localAddress) {
        final ChannelFuture regFuture = this.initAndRegister();
        final Channel channel = regFuture.channel();
        
        
        
        
        final ChannelFuture initAndRegister() {
        Channel channel = null;

        try {
            channel = this.channelFactory.newChannel();
            this.init(channel);
        } catch (Throwable var3) {
            if (channel != null) {
                channel.unsafe().closeForcibly();
            }

            return (new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE)).setFailure(var3);
        }
```







```
// 对应 hannel(NioServerSocketChannel.class) 11的实现；
public B channel(Class<? extends C> channelClass) {
    if (channelClass == null) {
        throw new NullPointerException("channelClass");
    } else {
     // NioServerSocketChannel类型的channelClass 通过反射方式创建；
    return this.channelFactory((io.netty.channel.ChannelFactory)(new ReflectiveChannelFactory(channelClass)));
    }
}
```



**NioServerSocketChannel **

```
public class NioServerSocketChannel extends AbstractNioMessageChannel implements 
ServerSocketChannel
```

![image-20200426152133058](http://q8xc9za4f.bkt.clouddn.com/cloudflare/image-20200426152133058.png)





**NioServerSocketChannel  methord**

![image-20200426152420546](http://q8xc9za4f.bkt.clouddn.com/cloudflare/image-20200426152420546.png)



### 2.1.2 反射创建服务端Channel过程；





![image-20200426152554417](http://q8xc9za4f.bkt.clouddn.com/cloudflare/image-20200426152554417.png)





#### 2.1.2.1  newSocket() 

```
//  1.  NioServerSocketChannel   默认构造函数;
    public NioServerSocketChannel() {
        this(newSocket(DEFAULT_SELECTOR_PROVIDER));
    }
    
// 2. NioServerSocketChannel newSocket 具体实现方法； 参数SelectorProvider provider；
  private static java.nio.channels.ServerSocketChannel newSocket(SelectorProvider provider) {
        try {
            return provider.openServerSocketChannel();
        } catch (IOException var2) {
            throw new ChannelException("Failed to open a server socket.", var2);
        }
    }
    
//3. 实现  java.nio.channels.spi.SelectorProvider.openServerSocketChannel 



```



####2.1.2.2 NioServerSocketChannelConfig 创建tcp参数配置类

**作用** ： 传递TCP相关的参数；





```
private final class NioServerSocketChannelConfig extends DefaultServerSocketChannelConfig {
    private NioServerSocketChannelConfig(NioServerSocketChannel channel, ServerSocket javaSocket) {
        super(channel, javaSocket);
    }

    protected void autoReadCleared() {
        NioServerSocketChannel.this.clearReadPending();
    }
}
```





####2.1.2.3  AbstractNioChannel

```
// NioServerSocketChannel 在其默认构造方法中；

public NioServerSocketChannel(java.nio.channels.ServerSocketChannel channel) {
    super((Channel)null, channel, 16);
    this.config = new NioServerSocketChannel.NioServerSocketChannelConfig(this, this.javaChannel().socket());
}
```



```
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    this.readInterestOp = readInterestOp;

    try {
    // 设置服务端channel非阻塞过程
    
        ch.configureBlocking(false);
    } catch (IOException var7) {
        try {
            ch.close();
        } catch (IOException var6) {
            if (logger.isWarnEnabled()) {
                logger.warn("Failed to close a partially initialized socket.", var6);
            }
        }

        throw new ChannelException("Failed to enter non-blocking mode.", var7);
    }
}
```

#### 

####2.1.2.4 AbstractChannel

 			服务端，客户端都要继承该channel;



```
io.netty.channel


protected AbstractChannel(Channel parent) {
    this.parent = parent;
    this.id = this.newId();     	 //	id changeID唯一标识；
    this.unsafe = this.newUnsafe();   // netty 特有方法；
    this.pipeline = this.newChannelPipeline(); // 和服务端客户端相关的逻辑链；
}
```



**总结**  

- newSocket 反射 创建底层jdk channel 
- 创建channel 相关的config参数类,tcp
- 然后configureBlocking（false） 设置为非阻塞模式
- 一起就绪，创建channel 最重要属性pipeline



###2.2  初始化服务端Channel

`

![image-20200426164740637](http://q8xc9za4f.bkt.clouddn.com/cloudflare/image-20200426164740637.png)

# 



### 2.2.1







## 2.3 注册selector

![image-20200426171529518](http://q8xc9za4f.bkt.clouddn.com/cloudflare/image-20200426171529518.png)

##  



```
// io.netty.channel.AbstractChannel

public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    if (eventLoop == null) {
        throw new NullPointerException("eventLoop");
    } else if (AbstractChannel.this.isRegistered()) {
        promise.setFailure(new IllegalStateException("registered to an event loop already"));
    } else if (!AbstractChannel.this.isCompatible(eventLoop)) {
        promise.setFailure(new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
    } else 
    		// Step1. 绑定线程；简单的赋值操作，告诉channel所有的事件IO操作交给eventLoop 处理；
        AbstractChannel.this.eventLoop = eventLoop;
        if (eventLoop.inEventLoop()) {
         	//Step2. 实际注册；
            this.register0(promise);
        } else {
            try {
                eventLoop.execute(new Runnable() {
                    public void run() {
                        AbstractUnsafe.this.register0(promise);
                    }
                });
            } catch (Throwable var4) {
                AbstractChannel.logger.warn("Force-closing a channel whose registration task was not accepted by an event loop: {}", AbstractChannel.this, var4);
                this.closeForcibly();
                AbstractChannel.this.closeFuture.setClosed();
                this.safeSetFailure(promise, var4);
            }
        }

    }
}
```



**register0**

```
// AbstractChannel
// 跟着Step2    this.register0(promise);
private void register0(ChannelPromise promise) {
    try {
        if (!promise.setUncancellable() || !this.ensureOpen(promise)) {
            return;
        }

        boolean firstRegistration = this.neverRegistered;
        //Step2.1 调用JDK底层事件；
        AbstractChannel.this.doRegister();
        this.neverRegistered = false;
        AbstractChannel.this.registered = true;
        //Step2.2  事件回调，添加chanleHanlder到channel的时候，添加用户回调；
        AbstractChannel.this.pipeline.invokeHandlerAddedIfNeeded();
        this.safeSetSuccess(promise);
        //Step2.3 把channel注册成功这个事件，传播到用户的代码中；
        AbstractChannel.this.pipeline.fireChannelRegistered();
        if (AbstractChannel.this.isActive()) {
            if (firstRegistration) {
                AbstractChannel.this.pipeline.fireChannelActive();
            } else if (AbstractChannel.this.config().isAutoRead()) {
                this.beginRead();
            }
        }
    } catch (Throwable var3) {
        this.closeForcibly();
        AbstractChannel.this.closeFuture.setClosed();
        this.safeSetFailure(promise, var3);
    }

}
```



**AbstractChannel.this.doRegister() 紧跟上面代码Step2.1**

```
// 

protected void doRegister() throws Exception {
    boolean selected = false;

    while(true) {
        try {
           // 注册selector 调用jdk底层selector方法；
           // javaChannel 2.1所讲创建channel会创建一个jdk底层的channel;
           // 参数意义
           		1. selector
           		2. 注册时关心的事件，0，代表不关心任何时事件；
           		3.  this 代表服务端channel,通过txshahment ,绑定到selector
           
            this.selectionKey = this.javaChannel().register(this.eventLoop().selector, 0, this);
            return;
        } catch (CancelledKeyException var3) {
            if (selected) {
                throw var3;
            }

            this.eventLoop().selectNow();
            selected = true;
        }
    }
}
```

## 2.4 端口绑定；

![image-20200426180222110](Deep-in-Netty.assets/image-20200426180222110.png)







```
// io.netty.bootstrap.AbstractBootstrap

// Step1.
private ChannelFuture doBind(final SocketAddress localAddress) {
    final ChannelFuture regFuture = this.initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    } else if (regFuture.isDone()) {
        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        final AbstractBootstrap.PendingRegistrationPromise promise = new AbstractBootstrap.PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            public void operationComplete(ChannelFuture future) throws Exception {
                Throwable cause = future.cause();
                if (cause != null) {
                    promise.setFailure(cause);
                } else {
                    promise.registered();
                    AbstractBootstrap.doBind0(regFuture, channel, localAddress, promise);
                }

            }
        });
        return promise;
、、    }
}
```









```
// io.netty.channel.AbstractChannel 方法；


public final void bind(SocketAddress localAddress, ChannelPromise promise) {
    this.assertEventLoop();
    if (promise.setUncancellable() && this.ensureOpen(promise)) {
        if (Boolean.TRUE.equals(AbstractChannel.this.config().getOption(ChannelOption.SO_BROADCAST)) && localAddress instanceof InetSocketAddress && !((InetSocketAddress)localAddress).getAddress().isAnyLocalAddress() && !PlatformDependent.isWindows() && !PlatformDependent.isRoot()) {
            AbstractChannel.logger.warn("A non-root user can't receive a broadcast packet if the socket is not bound to a wildcard address; binding to a non-wildcard address (" + localAddress + ") anyway as requested.");
        }
		// 端口绑定未完成此时返回false;
        boolean wasActive = AbstractChannel.this.isActive();

        try {
        // Step5 绑定底层JDK方法；
            AbstractChannel.this.doBind(localAddress);
        } catch (Throwable var5) {
            this.safeSetFailure(promise, var5);
            this.closeIfClosed();
            return;
        }
		// 传播事件；端口绑定之前不是active(true)，端口绑定之后是active,返回true;
		// 此时触发channelActive事件；
		// 从pipeline开始传播；
		
        if (!wasActive && AbstractChannel.this.isActive()) {
            this.invokeLater(new Runnable() {
                public void run() {
                    AbstractChannel.this.pipeline.fireChannelActive();
                }
            });
        }

        this.safeSetSuccess(promise);
    }
}
```



**紧跟上面Step5**

```
// io.netty.channel.socket.nio.NioServerSocketChannel;

protected void doBind(SocketAddress localAddress) throws Exception {
    if (PlatformDependent.javaVersion() >= 7) {
    // 创建jdk channel ，端口进行绑定；
        this.javaChannel().bind(localAddress, this.config.getBacklog());
    } else {
    // 
        this.javaChannel().socket().bind(localAddress, this.config.getBacklog());
    }

}
```



## 在哪里accept 连接？



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
                        this.select(this.wakenUp.getAndSet(false));
                        if (this.wakenUp.get()) {
                            this.selector.wakeup();
                        }
                    default:
                        this.cancelledKeys = 0;
                        this.needsToSelectAgain = false;
                        int ioRatio = this.ioRatio;
                        if (ioRatio == 100) {
                            try {
                                this.processSelectedKeys();
                            } finally {
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