## 前言

Netty作为一款高性能的网络通信框架,广泛应用在RPC框架,MQ组件和游戏行业的基础通信, 每款框架都有不同的参数和配置来满足不同的使用场景,也有相应的最佳实践方式,这里整理了近些年在使用Netty过程中遇到的一些问题和最佳的解决方式,也欢迎大家提pr补充你们遇到的问题和对应的解决方式,帮助大家一起用好Netty.

## 线程篇

### bossGroup和workerGroup

Netty服务端有2个线程池,`bossGroup` 和 `workerGroup`,  `bossGroup` 线程负责处理IO的Accept事件,接受远程客户端的连接;而 `workerGroup` 处理已连接的客户端的IO读写。`bossGroup` 和` workerGroup`对应源码的 `EventLoop` ,而每个 `EventLoop` 是一个只有单个线程的线程池,每个`channel`被激活后会注册到一个`EventLoop`上,这就保证了后续该 `channel` 的操作都是线程安全的,尽量减少了锁的使用, 提高并发性能.

> EventLoopGroup的默认大小都是是2倍的CPU核数，但这并不是一个恒定的最佳数量，为了避免线程上下文切换，只要能满足要求，这个值其实越少越好。
> Boss Group每个端口平时好像就只占1条线程，无论配了多少。

因此对于每个Server端口, `BossGroup` 配置一个线程即可, `workerGroup` 可以根据请求量的大小适当调整。

### 业务线程池

Netty线程的数量一般固定且较少，所以很怕线程被堵塞，比如同步的数据库查询，比如下游的服务调用，此时就要把处理放到一个业务线程池里操作，即使要付出线程上下文切换的代价，甚至还有些 `ThreadLocal` 需要复制。

最佳实践: 在你的`ChannelInboundHandle`的`channelRead0`方法里，通过自定义线程池去处理请求:
``` java
  protected void channelRead0(final ChannelHandlerContext ctx, final Message request) throws Exception {
        executor.execute(new Runnable() {
          @Override
          public void run() {
            handleRequest(ctx, request);
          }
        });
}
```

### write后不要修改对象

在Netty 4里 `inbound` 和 `outbound` 都是在 `EventLoop`(IO线程)里执行, 如果在业务线程里通过 `channel.writeAndFlush` 向网络写出一些东西的时候，netty 会往这个 `channel` 的 `EventLoop`里提交一个写出的任务,那也就是业务线程和IO线程是异步执行的,如果你在调用 `writeAndFlush(message)` 后又修改了message对象, 最后导致的结果可能和你预期的不一致, 需要在写之前深拷贝一份。

###  定时任务

像发送超时控制之类的一次性任务，不要使用JDK的`ScheduledExecutorService` ，而是如下：
``` java
ctx.executor().schedule(new MyTimeoutTask(p), 30, TimeUnit.SECONDS)
``` 
JDK的 `ScheduledExecutorService` 是一个大池子，多线程争抢并发锁,上面的写法，`TimeoutTask` 只属于当前的 `EventLoop` ，没有任何锁;如果发送成功，需要从长长Queue里找回任务来取消掉它,现在每个 `EventLoop` 一条 `Queue` ，长度只有原来的N分之一。

## 连接管理

## 连接池

为了提高通信效率，我们需要考虑复用连接，减少 TCP 三次握手的次数，因此需要有连接管理的机制。一般需要建立多个连接来提高通信效率，我们需要设计一个针对某个连接地址（IP 与 Port 唯一确定的地址）建立特定数目连接的实现，同时保存在一个连接池里。

客户端启动时可以预先和服务端建立好多个连接, 这样子需要用的时候可以避免建立连接的开销;获取连接时调用`channel.isAvailable()`判断连接是否可用,如果不可用则重连;
``` java
  if (channel.isAvailable()) {
    if (channel.isWriteable()) {
      return channel.getChannel();
    } else {
      // 写 缓冲区溢出的话选下一个链接
      unWriteableCnt++;
    }
  } else {
    channel.reConnect((int) (maxReqConnTimeout - totalPassedTimeMs));
    if (channel.isAvailable()) {
      return channel.getChannel();
    } else {
      reconnCnt++;
    }
  }
```
* 连接数在满足传输吞吐量的情况下，越少越好。

   举个例子，在我的Proxy测试场景里：

    >2条连接时，只能有40k QPS。
    >48条连接，升到62k QPS，CPU烧了28%
    >4条连接，QPS反而上升到68k ，而且CPU降到20%。
    
 ### 连接超时配置
 
 在使用Netty编写客户端的时候，我们一般会有类似这样的代码：
``` java
bootstrap.connect(address).await(1000, TimeUnit.MILLISECONDS)
```
向对端发起一个连接，超时等待 1 秒钟。如果 1 秒钟没有连接上则重连或者做其他处理。而其实在bootstrap的选项里，还有这样的一项：
``` java
bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000);
```
如果这两个值设置的不一致，在await的时候较短，而option里设置的较长就出问题了。这个时候你会发现connect里已经超时了，你以为连接失败了，但实际上await超时 `Netty` 并不会帮你取消正在连接的链接。这个时候如果第2秒的时候连上了对端服务器，那么你刚才的判断就失误了。如果你根据`connect(address).await(1000, TimeUnit.MILLISECONDS)` 来决定是否重连，很有可能你就建立了两个连接，而且很有可能你的 `handler` 就在这两个 `channel` 里共享起来了.所以建议不要在 `await` 上设置超时，而总是使用 `option` 上的选项来设置。

或者通过 `ChannelFuture` 来把超时了但最终连接的channel关闭:
``` java
final ChannelFuture cf = bootstrap.connect();
 boolean result = cf.awaitUninterruptibly(1000, TimeUnit.MILLISECONDS);
 if (result && cf.isSuccess()) {
    return cf.channel();
  } else {
        // 可能超时的但是最终成功的 channel 关闭掉
        cf.addListener(new GenericFutureListener<Future<? super Void>>() {
          @Override
          public void operationComplete(Future<? super Void> future) throws Exception {
            if (future.isSuccess()) {
              cf.channel().close();
            } 
          }
        });
        }

```

###  Netty Native

`Netty Native`用C++编写JNI调用的`Socket Transport`，是由`Twitter`将`Tomcat Native`的移植过来，现在还时不时和汤姆家同步一下代码。

经测试，的确比`JDK NIO`更省CPU。

也许有人问，JDK的NIO也用EPOLL啊，大家有什么不同？ `Norman Maurer`这么说的：

>* Netty的 `epoll transport`使用 `edge-triggered` 而 JDK NIO 使用 `level-triggered`
>* C代码，更少GC，更少`synchronized`
>* 暴露了更多的`Socket`配置参数

用法倒是简单，只要几个类名替换一下:
``` java
if (Epoll.isAvailable()) {
  acceptorGroup = new EpollEventLoopGroup(config.getAcceptorSize(),
      new DefaultThreadFactory("acceptor-" ));
  selectorGroup = new EpollEventLoopGroup(config.getSelectorSize(),
      new DefaultThreadFactory("selector-"));
  bootstrap.group(acceptorGroup, selectorGroup).channel(EpollServerSocketChannel.class);
} else {
  acceptorGroup = new NioEventLoopGroup(config.getAcceptorSize(),
      new DefaultThreadFactory("acceptor-"));
  selectorGroup = new NioEventLoopGroup(config.getSelectorSize(),
      new DefaultThreadFactory("selector-"));
  bootstrap.group(acceptorGroup, selectorGroup).channel(NioServerSocketChannel.class);
}
```
### Socket参数

`TCP/Socket`的大路设置，无非 `SO_REUSEADDR`， `TCP_NODELAY`， `SO_KEEPALIVE` 。另外还有`SO_LINGER` ， `SO_TIMEOUT`, `SO_BACKLOG`, `SO_SNDBUF`, `SO_RCVBUF`;这几个参数的作用可以参考[Linux下高性能网络编程中的几个TCP/IP选项](https://blog.csdn.net/zlxfogger/article/details/44922993)

server配置:
``` java
bootstrap.option(ChannelOption.SO_BACKLOG, config.getBacklog());
bootstrap.childOption(ChannelOption.SO_KEEPALIVE, true);
bootstrap.childOption(ChannelOption.TCP_NODELAY, true);
bootstrap.childOption(ChannelOption.WRITE_BUFFER_WATER_MARK, new WriteBufferWaterMark(8 * 1024, 32 * 1024));
```
client配置:
``` java
bootstrap.option(ChannelOption.SO_REUSEADDR, true);
bootstrap.option(ChannelOption.TCP_NODELAY, true);
bootstrap.option(ChannelOption.SO_KEEPALIVE, true);
bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, config.getReqConnTimeoutMs());
bootstrap.option(ChannelOption.SO_SNDBUF, 4096);
bootstrap.option(ChannelOption.SO_RCVBUF, 4096);
bootstrap.option(ChannelOption.WRITE_BUFFER_WATER_MARK, new WriteBufferWaterMark(8 * 1024, 32 * 1024));
```

## 内存篇

### 堆外内存

堆外内存是IO框架的绝配，但堆外内存的分配销毁不易，所以使用内存池能大幅提高性能，也告别了频繁的GC。
> Netty里主要有四种 `ByteBuf` ，其中 `UnpooledHeapByteBuf` 底下的byte[]能够依赖JVM GC自然回收；而 `UnpooledDirectByteBuf` 底下是 `DirectByteBuffer` ，除了等JVM GC，最好也能主动进行回收；而`PooledHeapByteBuf` 和 `PooledDirectByteBuf` ，则必须要主动将用完的 `byte[]/ByteBuffer` 放回池里，否则内存就要爆掉。

分配内存推荐的几种方式:
``` java
//堆内内存
ByteBuf buf = ctx.alloc().buffer();
ByteBuf buf = channel.alloc().buffer();
ByteBuf buf = PooledByteBufAllocator.DEFAULT.buffer();
//堆外内存
ByteBuf buf = ctx.alloc().directBuffer();
ByteBuf buf = channel.alloc().directBuffer();
ByteBuf buf = PooledByteBufAllocator.DEFAULT.directBuffer();
```
在Netty里，因为Handler链的存在，ByteBuf经常要传递到下一个Hanlder去而不复还，谁是最后使用者，谁负责释放,Netty提供了`ReferenceCountUtil.release(buf)`方法来释放ByteBuf，推荐大家看一下这篇文章: [Netty之有效规避内存泄漏
](http://calvin1978.blogcn.com/articles/netty-leak.html)

#### 禁用Netty对象缓存机制

Netty的无锁化设计使得对象可以在线程内无锁的被回收重用,但有些场景里，某线程只创建对象，要到另一个线程里释放，一不小心，你就会发现应用缓缓变慢，`heap dump` 时看到好多 `RecyleHandler` 对象。比如有些RPC框架为了减轻 io 线程的压力会把序列化放在业务线程,如下的这段代码如果在业务线程里执行,就会有[内存泄漏](https://jacobke.github.io/2017/09/13/netty-recycler-cache/)的问题:
``` java
PooledByteBufAllocator allocator = PooledByteBufAllocator.DEFAULT;
ByteBuf buffer = allocator.buffer();
User user = new User();
serialization.serialize(buffer, user);
channel.writeAndFlush(buffer);
```
分配的时候是在业务线程，也就是说从业务线程的 `thread local`对应的池里分配的，而回收的时候是在 IO 线程,这两个是不同的线程,池的作用完全丧失了，一个线程不断地去分配, 然后不断地转移到另外一个池。netty 貌似直到4.0.40才修复了这个问题,相关的issue可以看下这个[Netty4.0.28 mem leak for io.netty.util.Recycler](https://github.com/netty/netty/issues/5563)

但有时觉得这么辛苦的重用一个对象,不如干脆禁止掉这个功能，4.0.0.33之后的版本可以在启动参数里加入 `-Dio.netty.recycler.maxCapacity.default=0`禁用对象缓存,4.1之后的版本要使用`-Dio.netty.recycler.maxCapacity=0`才行。

#### arene参数配置

netty内部使用 `arena` 来管理 `ByteBuf` 的分配,相关参数如下:

``` java
  /**
   * 配置 netty arena个数, 每一个 arena 占用8mb
   * 如果为0， 则内部会使用 Unpooled直接取
   */
  public static void configArena(int directArenaCnt, int heapArenaCnt) {
    //堆外内存Arena数,如果为0，最终分配的是UnpooledDirectByteBuf
    System.setProperty("io.netty.allocator.numDirectArenas",String.valueOf(directArenaCnt));
    //堆内内存Arena数,如果为0，最终分配的是UnpooledHeapByteBuf
    System.setProperty("io.netty.allocator.numHeapArenas",String.valueOf(heapArenaCnt));
    System.setProperty("io.netty.allocator.pageSize", "4096");
  }
```

建议在测试的时候将内存泄漏检查级别开到最高,可以通过参数`-Dio.netty.leakDetectionLevel=ADVANCED`,也可以直接调用Netty的API `ResourceLeakDetector.setLevel(ResourceLeakDetector.Level.ADVANCED)`,同时监控Netty的logger有没有输出memory leak信息。
如果发生了内存泄漏,大概率会打印类似下面的日志:

> 2018-02-09 20:15:29  [ nioEventLoopGroup-1-0:635052 ] - [ ERROR ]  LEAK: ByteBuf.release() was not called before it's garbage-collected. See http://netty.io/wiki/reference-counted-objects.html for more information.
Recent access records: 5   io.netty.buffer.AdvancedLeakAwareByteBuf.readBytes(AdvancedLeakAwareByteBuf.java:435)
com.ezlippi.nettyServer.ServerHandler.channelRead(ObdServerHandler.java:31)

### 避免WriteAndFlushTask过多

在使用 `Channel` 写数据之前，建议使用 `isWritable()` 方法来判断一下当前 `ChannelOutboundBuffer` 里的写缓存水位，防止 OOM 发生。写缓存的水位可以在初始化BootStrap的时候设置:
``` java
bootstrap.childOption(ChannelOption.WRITE_BUFFER_WATER_MARK, new WriteBufferWaterMark(8 * 1024, 32 * 1024));
```    
当写到ChannelOutBoundBuffer的数据量大于高水位时,会激活`pipeline.fireChannelWritabilityChanged()`,调用`isWritable` 会返回false；

同时,还有一个需要注意的是 `NioEventLoop` 的队列默认是无限大的,如果没有调用 `isWritable` 判断,会不断的提交 `WriteAndFlushTask` 到 `EventLoop` 的队列中,在对方接收比较慢的场景就可能出现OOM，可以通过系统属性配置下:
``` java
  public static void configCommonProp() {       System.setProperty("io.netty.eventLoop.maxPendingTasks", "4096");
System.setProperty("io.netty.eventexecutor.maxPendingTasks", "4096");
  }
```
## 其他建议

* 对于无状态的 `ChannelHandler` ，通过`ChannelHandler.Sharable`注解设置成共享模式，比如我们的事件处理器可以设置为共享，减少不同的 `Channel` 对应的 `ChannelPipeline` 里生成的对象个数:
    ``` java
        @ChannelHandler.Sharable
        public class NettyServerDefaultChannelHandler   extends SimpleChannelInboundHandler<Message> {
        }
    ```

* 正确使用 `ChannelHandlerContext` 的 `ctx.write()` 与 `ctx.channel().write()` 方法。前者是从当前 Handler 的下一个 `Handler`开始处理，而后者会从 tail 开始处理。大多情况下使用 `ctx.write()` 即可。

参考文档:

* [Netty高性能编程备忘录](http://calvin1978.blogcn.com/articles/netty-performance.html)
* [蚂蚁通信框架实践](http://www.jiangxinlingdu.com/netty/2018/11/23/bolt.html)
* [Netty内存管理](https://gsmtoday.github.io/2017/09/03/netty-memory-pool-md/)
