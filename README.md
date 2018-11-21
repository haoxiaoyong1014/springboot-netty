#### Netty介绍
> **Netty是一个NIO客户端服务器框架，可以快速轻松地开发协议服务器和客户端等网络应用程序。它极大地简化并简化了TCP和UDP套接字服务器等网络编程。
“快速简便”并不意味着最终的应用程序会受到可维护性或性能问题的影响。Netty经过精心设计，具有丰富的协议，如FTP，SMTP，HTTP以及各种二进制和基于文本的传统协议。因此，Netty成功地找到了一种在不妥协的情况下实现易于开发，性能，稳定性和灵活性的方法。Netty 版本3x(稳定,jdk1.5+),4x(推荐,稳定,jdk1.6+),5x(不推荐),新版本不是很稳定,所以这里使用的是 Netty4x 版本**

#### 项目依赖
```xml
 <dependency>
   <groupId>io.netty</groupId>
   <artifactId>netty-all</artifactId>
   <version>4.1.21.Final</version>
 </dependency>
```
#### IdleStateHandler
- Netty 可以使用 IdleStateHandler 来实现连接管理，当连接空闲时间太长（没有发送、接收消息）时则会触发一个事件，我们便可在该事件中实现心跳机制。

#### 编写服务端
```java
public class HeartBeatSimpleHandle extends SimpleChannelInboundHandler<CustomProtocol> {

    private final static Logger LOGGER = LoggerFactory.getLogger(HeartBeatSimpleHandle.class);
    private static final ByteBuf HEART_BEAT = Unpooled.unreleasableBuffer(Unpooled.copiedBuffer(new CustomProtocol(123456L, "pong").toString(), CharsetUtil.UTF_8));

    /**
     * 取消绑定
     * @param ctx
     * @throws Exception
     */
    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        NettySocketHolder.remove((NioSocketChannel) ctx.channel());
    }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent idleStateEvent = (IdleStateEvent) evt;
            if (idleStateEvent.state() == IdleState.READER_IDLE) {
                LOGGER.info("已经5秒没有收到信息！");
                //向客户端发送消息
                ctx.writeAndFlush(HEART_BEAT).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            }
        }
        super.userEventTriggered(ctx, evt);
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, CustomProtocol customProtocol) throws Exception {
        LOGGER.info("收到customProtocol={}", customProtocol);
        //我们调用writeAndFlush（Object）来逐字写入接收到的消息并刷新线路
        //ctx.writeAndFlush(customProtocol);
        //保存客户端与 Channel 之间的关系
        NettySocketHolder.put(customProtocol.getId(), (NioSocketChannel) ctx.channel());
    }
}

```
**对上面代码简要说明:** 
`HeartBeatSimpleHandle`继承了 ChannelInboundHandlerAdapter 的一个扩展(SimpleChannelInboundHandler),
而ChannelInboundHandlerAdapter是ChannelInboundHandler的一个实现
ChannelInboundHandler提供了可以重写的各种事件处理程序方法,包括channelRead0()方法.
目前，只需继承 SimpleChannelInboundHandler或ChannelInboundHandlerAdapter 而不是自己实现处理程序接口。
我们重写了channelRead0()方法,每当接收到新数据时，都会使用收到的消息调用此方法。
当有多个客户端连上来时，服务端需要区分开，不然响应消息就会发生混乱。
所以每当有个连接上来的时候，我们都将当前的 Channel 与连上的客户端 ID 进行关联（因此每个连上的客户端 ID 都必须唯一）。
这里采用了一个 Map 来保存这个关系，并且在断开连接时自动取消这个关联。
```java
public class NettySocketHolder {

    private static final Map<Long, NioSocketChannel> MAP = new ConcurrentHashMap<>(16);

    public static void put(Long id, NioSocketChannel socketChannel) {
        MAP.put(id, socketChannel);
    }

    public static NioSocketChannel get(Long id) {
        return MAP.get(id);
    }

    public static Map<Long, NioSocketChannel> getMAP() {
        return MAP;
    }

    public static void remove(NioSocketChannel nioSocketChannel) {
        MAP.entrySet().stream().filter(entry -> entry.getValue() == nioSocketChannel).forEach(entry -> MAP.remove(entry.getKey()));
    }
}
```
* 这里使用了jdk1.8的Lambda 表达式
#### 启动引导程序

```java
@Component
public class HeartBeatServer {

    private final static Logger LOGGER = LoggerFactory.getLogger(HeartBeatServer.class);
    private EventLoopGroup boss = new NioEventLoopGroup(); //(1)
    private EventLoopGroup work = new NioEventLoopGroup();

    @Value("${netty.server.port}")
    private int nettyPort;

    /**
     * 启动 Netty
     *
     * @return
     * @throws InterruptedException
     */
    @PostConstruct
    public void start() throws InterruptedException {
        ServerBootstrap bootstrap = new ServerBootstrap() //(2)
                .group(boss, work)
                .channel(NioServerSocketChannel.class)// (3)
                .localAddress(new InetSocketAddress(nettyPort))
                //保持长连接
                .childOption(ChannelOption.SO_KEEPALIVE, true)//(4)
                .childHandler(new HeartbeatInitializer());// (5)
        //绑定并开始接受传入的连接。
        ChannelFuture future = bootstrap.bind().sync();//(6)
        if (future.isSuccess()) {
            LOGGER.info("启动 Netty 成功");
        }
    }

    /**
     * 销毁
     */
    @PreDestroy
    public void destroy() {                        //(7)
        boss.shutdownGracefully().syncUninterruptibly();
        work.shutdownGracefully().syncUninterruptibly();
        LOGGER.info("关闭 Netty 成功");
    }
}
```
**对上面代码进行简要说明**

(1),NioEventLoopGroup是一个处理I / O操作的多线程事件循环。 Netty为不同类型的传输提供各种EventLoopGroup实现。我们在此示例中实现了服务器端应用程序，因此将使用两个NioEventLoopGroup。第一个，通常称为“老板”，接受传入连接。第二个，通常称为“工人”，一旦老板接受连接并将接受的连接注册到工作人员，就处理被接受连接的流量。使用了多少个线程以及它们如何映射到创建的Channels取决于EventLoopGroup实现，甚至可以通过构造函数进行配置。

(2),ServerBootstrap是一个设置服务器的帮助程序类。

(3), 在这里，我们指定使用`NioServerSocketChannel`类，该类用于实例化新的Channel以接受传入的连接。

(4),保持长连接

(5), childHandler()方法需要一个ChannelInitializer类,ChannelInitializer是一个特殊的处理程序，旨在帮助用户配置新的Channel,您最有可能希望通过添加一些处理程序（如DiscardServerHandler）来配置新Channel的ChannelPipeline，以实现您的网络应用程序。 随着应用程序变得复杂，您可能会向管道添加更多处理程序，并最终将此匿名类提取到顶级类中。
这里我们用HeartbeatInitializer类继承了ChannelInitializer类并重写了initChannel()方法

```java
public class HeartbeatInitializer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel channel) throws Exception {
        channel.pipeline()
                //五秒没有收到消息 将IdleStateHandler 添加到 ChannelPipeline 中
                .addLast(new IdleStateHandler(5, 0, 0))//(8)
                .addLast(new HeartbeatDecoder())//(9)
                .addLast(new HeartBeatSimpleHandle());//(10)
    }
}
```
(6).绑定到端口并启动服务器。

(7).当程序关闭时优雅的关闭 Neety

(8).将IdleStateHandler 添加到 ChannelPipeline 中，也会有一个定时任务，每5秒校验一次是否有收到消息，否则就主动发送一次请求。

(9),服务端解码器,服务端与客户端采用的是自定义的 POJO 进行通讯的,所以需要在客户端进行编码，服务端进行解码，也都只需要各自实现一个编解码器即可。(下面也会说到)

```java
public class HeartbeatDecoder extends ByteToMessageDecoder {
    @Override
    protected void decode(ChannelHandlerContext channelHandlerContext, ByteBuf byteBuf, List<Object> list) throws Exception {
        long id = byteBuf.readLong();
        byte[] bytes = new byte[byteBuf.readableBytes()];
        byteBuf.readBytes(bytes);
        String content = new String(bytes);
        CustomProtocol customProtocol = new CustomProtocol();
        customProtocol.setId(id);
        customProtocol.setContent(content);
        list.add(customProtocol);
    }
}
```
(10),`HeartBeatSimpleHandle`继承了 ChannelInboundHandlerAdapter 的一个扩展(SimpleChannelInboundHandler)

#### 编写客户端
* 当客户端空闲了 N 秒没有给服务端发送消息时会自动发送一个心跳来维持连接。

```java
/**
 * Created by haoxy on 2018/10/17.
 * E-mail:hxyHelloWorld@163.com
 * github:https://github.com/haoxiaoyong1014
 */
public class EchoClientHandle extends SimpleChannelInboundHandler<ByteBuf> {

    private final static Logger LOGGER = LoggerFactory.getLogger(EchoClientHandle.class);

    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent idleStateEvent = (IdleStateEvent) evt;
            if (idleStateEvent.state() == IdleState.WRITER_IDLE) {
                LOGGER.info("已经10秒没收到消息了");
                //向服务端发送消息
                CustomProtocol heartBeat = SpringBeanFactory.getBean("heartBeat",CustomProtocol.class);
                ctx.writeAndFlush(heartBeat).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            }

        }
        super.userEventTriggered(ctx, evt);
    }

    /**
     *  每当从服务端接收到新数据时，都会使用收到的消息调用此方法 channelRead0(),在此示例中，接收消息的类型是ByteBuf。
     * @param channelHandlerContext
     * @param byteBuf
     * @throws Exception
     */
    @Override
    protected void channelRead0(ChannelHandlerContext channelHandlerContext, ByteBuf byteBuf) throws Exception {
        //从服务端收到消息时被调用
        LOGGER.info("客户端收到消息={}", byteBuf.toString(CharsetUtil.UTF_8));
    }
}

```
**对上面代码简要说明:** 
EchoClientHandle继承了 ChannelInboundHandlerAdapter 的一个扩展(SimpleChannelInboundHandler),
而ChannelInboundHandlerAdapter是ChannelInboundHandler的一个实现
ChannelInboundHandler提供了可以重写的各种事件处理程序方法,包括channelRead0()方法.
目前，只需继承 SimpleChannelInboundHandler或ChannelInboundHandlerAdapter 而不是自己实现处理程序接口。
我们重写了channelRead0()方法,每当接收到新数据时，都会使用收到的消息调用此方法。
由于整合了 SpringBoot,在这里我们向服务端发送的是一个 单例的 Bean(上面所说的 pojo),所涉及到的类有:

**SpringBeanFactory** 
```java
@Component
public final class SpringBeanFactory implements ApplicationContextAware {

    private static ApplicationContext context;

    public static <T> T getBean(Class<T> c){
        return context.getBean(c);
    }


    public static <T> T getBean(String name,Class<T> clazz){
        return context.getBean(name,clazz);
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        context = applicationContext;
    }
}

```
**CustomProtocol**
```java
public class CustomProtocol implements Serializable {
    private static final long serialVersionUID = 290429819350651974L;
    private long id;
    private String content;
    //省去 get set方法
```
**HeartBeatConfig**
* 将CustomProtocol设置为一个 Bean,并赋值
```java
@Configuration
public class HeartBeatConfig {

    @Value("${channel.id}")
    private long id;

    @Bean(value = "heartBeat")
    public CustomProtocol heartBeat(){
        return new CustomProtocol(id,"ping") ;
    }
}
```
**当然我们还需要一个启动引导类**
```java
@Component
public class HeartbeatClient {
    private final static Logger LOGGER = LoggerFactory.getLogger(HeartbeatClient.class);
    private EventLoopGroup group = new NioEventLoopGroup();//(1)
    @Value("${netty.server.port}")
    private int nettyPort;
    @Value("${netty.server.host}")
    private String host;

    private SocketChannel socketChannel;

    @PostConstruct
    public void start() throws InterruptedException {
        Bootstrap bootstrap = new Bootstrap();//(2)
        /**
         * NioSocketChannel用于创建客户端通道，而不是NioServerSocketChannel。
         * 请注意，我们不像在ServerBootstrap中那样使用childOption()，因为客户端SocketChannel没有父服务器。
         */
        bootstrap.group(group)
        .channel(NioSocketChannel.class)//(3)
        .handler(new CustomerHandleInitializer());//(4)
        /**
         * 启动客户端
         * 我们应该调用connect()方法而不是bind()方法。
         */
        ChannelFuture future = bootstrap.connect(host, nettyPort).sync();//(5)
        if (future.isSuccess()) {
            LOGGER.info("启动 Netty 成功");
        }

        socketChannel = (SocketChannel) future.channel();

    }

}
```
**对上面代码进行简要说明:**
Netty中服务器和客户端之间最大和唯一的区别是使用了不同的Bootstrap和Channel实现。

(1),如果只指定一个EventLoopGroup，它将同时用作boss组和worker组。 但是，老板工作者不会用于客户端。

(2),Bootstrap(客户端使用)类似于ServerBootstrap(服务端使用)，不同之处在于它适用于非服务器通道

(3),NioSocketChannel用于创建客户端通道，而不是NioServerSocketChannel(服务端)

(4),客户端handler()方法与服务端childHandler()同样都是需要一个ChannelInitializer类,ChannelInitializer是一个特殊的处理程序,这里用CustomerHandleInitializer继承了ChannelInitializer

```java
public class CustomerHandleInitializer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel channel) throws Exception {
        channel.pipeline()
                //10 秒没发送消息 将IdleStateHandler 添加到 ChannelPipeline 中
                .addLast(new IdleStateHandler(0, 10, 0))//(6)
                .addLast(new HeartbeatEncode())//(7)
                .addLast(new EchoClientHandle());//(8)
    }
}
```
(5),我们这里应该调用connect()方法而不是bind()方法。

(6),客户端的心跳其实也是类似，也需要在 ChannelPipeline 中添加一个 IdleStateHandler 

(7),客户端编码器

```java
public class HeartbeatEncode extends MessageToByteEncoder<CustomProtocol> {
    @Override
    protected void encode(ChannelHandlerContext channelHandlerContext, CustomProtocol customProtocol, ByteBuf byteBuf) throws Exception {
        byteBuf.writeLong(customProtocol.getId()) ;
        byteBuf.writeBytes(customProtocol.getContent().getBytes()) ;
    }
}
```
(8),EchoClientHandle继承了 ChannelInboundHandlerAdapter 的一个扩展(SimpleChannelInboundHandler)

**这里贴出客户端与服务端的配置文件**
* 客户端
```java
# web port
server.port=8082
# 通道 ID
channel.id=100

netty.server.host=127.0.0.1
netty.server.port=11211
```
* 服务端
```java
# web port
server.port=8081
netty.server.port=11211
```
**启动客户端以及服务端**
![image](https://github.com/haoxiaoyong1014/best-pay-demo/raw/master/src/main/java/com/github/lly835/Images/x1.jpeg)
![image](https://github.com/haoxiaoyong1014/best-pay-demo/raw/master/src/main/java/com/github/lly835/Images/x2.jpeg)



* [x] [参考文章](https://crossoverjie.top/2018/05/24/netty/Netty(1)TCP-Heartbeat/)

* [x] [需要了解更多参考](https://netty.io/index.html)
