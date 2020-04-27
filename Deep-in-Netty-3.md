Netty 处理新连接



# 1.问题

- Netty是在哪里检测有新连接接入的？

- 新连接是怎样注册到NioEventLoop 线程的？

  

![image-20200427102657540](http://q8xc9za4f.bkt.clouddn.com/cloudflare/image-20200427102657540.png)



# 2. 检测新连接



![image-20200427102822672](http://q8xc9za4f.bkt.clouddn.com/cloudflare/image-20200427102822672.png)



# 3. 创建NioSocketChannel 



new NioSocketChannel(parent,ch)[入口]

