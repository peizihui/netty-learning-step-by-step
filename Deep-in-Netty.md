# 0.overview

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



## 2.2 在哪里accept 连接？


