# Netty的ServerBootstrap启动源码分析



## 1. ServerBootstrap启动代码

```java
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
        .channel(NioServerSocketChannel.class) //为启动类赋值一个ChannelFactory
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

f.channel().closeFuture().sync();
```

## 2. 两个问题

1. 服务端的Socket在哪里初始化？
2. 在哪里Accept连接？

## 3. 服务端启动主要过程

1. 创建服务端的Channel
2. 初始化服务端的Channel
3. 注册Selector
4. 端口绑定

### 3.1. 创建服务端的Channel

#### 3.1.1. 创建NioServerSocketChannel

服务端创建`NioServerSocketChannel`的流程大致如下：

![](.gitbook/assets/wx20190501-001819-2x.png)

首先看`bind()`方法，一步一步点下去：

```java
public ChannelFuture bind(int inetPort) {
    return this.bind(new InetSocketAddress(inetPort));
}
```

```java
public ChannelFuture bind(SocketAddress localAddress) {
    this.validate();
    if (localAddress == null) {
        throw new NullPointerException("localAddress");
    } else {
        return this.doBind(localAddress);
    }
}
```

```java
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
    }
}
```

这里出现了`initAndRegister()`方法，继续点进去：

```java
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

    ChannelFuture regFuture = this.config().group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }

    return regFuture;
}
```

注意到`channel = this.channelFactory.newChannel();`，通过工厂来获取一个Channel，继续点击去查看`newChannel()`方法：

```java
public T newChannel() {
    try {
        return (Channel)this.clazz.newInstance();
    } catch (Throwable var2) {
        throw new ChannelException("Unable to create Channel from class " + this.clazz, var2);
    }
}
```

这里是通过反射来获取Channel实例的，那么这个`clazz`和`channelFactory`又是在哪里传进来的呢？这里需要我们回到`ServerBootstrap.channel()`，其代码如下：

```java
public B channel(Class<? extends C> channelClass) {
    if (channelClass == null) {
        throw new NullPointerException("channelClass");
    } else {
        return this.channelFactory((io.netty.channel.ChannelFactory)(new ReflectiveChannelFactory(channelClass)));
    }
}
```

其中`channelFactory()`只是将channelFactory进行赋值，并且返回`ServerBootstrap`本身。可以看到，该方法中将channelClass封装成了一个`ReflectiveChannelFactory`，这个channelClass就是我们传进来的`NioServerSocketChannel.class`。我们点进去`ReflectiveChannelFactory`查看一下它的构造函数：

```java
public ReflectiveChannelFactory(Class<? extends T> clazz) {
    if (clazz == null) {
        throw new NullPointerException("clazz");
    } else {
        this.clazz = clazz;
    }
}
```

到这里，就可以通过Factory来进行`NioServerSocketChannel`的获取。但是`NioServerSocketChannel`只是Netty对Java底层的Channel所做的一个封装，我们需要进一步点进去查看`NioServerSocketChannel`。

#### 3.1.2. NioServerSocketChannel的构造

这里主要是用`NioServerSocketChannel`来对JDK提供的Channel进行一个封装：

![](.gitbook/assets/2.png)

首先是`NioServerSccketChannel`的构造方法：

```java
public NioServerSocketChannel() {
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
```

点进去查看`newSocket`方法：

```java
private static java.nio.channels.ServerSocketChannel newSocket(SelectorProvider provider) {
    try {
        return provider.openServerSocketChannel();
    } catch (IOException var2) {
        throw new ChannelException("Failed to open a server socket.", var2);
    }
}
```

这里调用JDK底层的方法打开一个Channel。其中`DEFAULT_SELECTOR_PROVIDER`是通过`SelectorProvider.provider()`来进行创建的。至此，我们就获得了一个JDK底层真正的Channel了。

回到构造方法上，继续点进去：

```java
public NioServerSocketChannel(java.nio.channels.ServerSocketChannel channel) {
    super((Channel)null, channel, 16);
    this.config = new NioServerSocketChannel.NioServerSocketChannelConfig(this, this.javaChannel().socket());
}
```

分为两个部分： 1. 继续调用父类构造函数 2. 配置config。Config类中包含了TCP的各项配置信息，例如backlog等

继续点到父类查看对应的构造函数：

```java
protected AbstractNioMessageChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent, ch, readInterestOp);
}
```

继续：

```java
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    this.readInterestOp = readInterestOp;

    try {
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

这里做了这几件事： 1. 调用父类构造函数 2. 为channel和readInterestOp进行赋值 3. 设置channel为非阻塞

我们继续查看父类的构造函数：

```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    this.id = this.newId();
    this.unsafe = this.newUnsafe();
    this.pipeline = this.newChannelPipeline();
}
```

在这里进行了`id`、`unsafe`和`pipeline`的初始化。其中unsafe封装了一些底层的有关TCP读写相关的方法，pipeline在接下来会提到，是Netty中很重要的一个组件。

### 3.2. 初始化服务端的Channel

