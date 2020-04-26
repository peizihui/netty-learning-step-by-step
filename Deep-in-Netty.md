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



## 2.2 在哪里accept 连接？

