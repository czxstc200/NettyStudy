# Netty源码分析（六）：Pipeline的初始化与Handler的增删

## 1. 一个问题

1. Netty是如何判断ChannelHandler类型的

## 2. Pipeline的初始化

1. Pipeline在创建Channel的时候被创建
2. ChannelHandlerContext
3. head和tail

### 2.1 Pipeline的创建

`Pipeline`的创建在`AbstractChannel`的构造函数中被调用

```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
}
```

点进去查看`newChannelPipeline()`：

```java
protected DefaultChannelPipeline newChannelPipeline() {
    return new DefaultChannelPipeline(this);
}
```

继续：

```java
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);

    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```

这里的重点就是`tail`和`head`的创建

### 2.2 ChannelHandlerContext

`ChannelHandlerContext`是对`Handler`的一个进一步封装：

```java
public interface ChannelHandlerContext extends AttributeMap, ChannelInboundInvoker, ChannelOutboundInvoker {

    /**
     * Return the {@link Channel} which is bound to the {@link ChannelHandlerContext}.
     */
    Channel channel();

    /**
     * Returns the {@link EventExecutor} which is used to execute an arbitrary task.
     */
    EventExecutor executor();

    /**
     * The unique name of the {@link ChannelHandlerContext}.The name was used when then {@link ChannelHandler}
     * was added to the {@link ChannelPipeline}. This name can also be used to access the registered
     * {@link ChannelHandler} from the {@link ChannelPipeline}.
     */
    String name();

    /**
     * The {@link ChannelHandler} that is bound this {@link ChannelHandlerContext}.
     */
    ChannelHandler handler();

    /**
     * Return {@code true} if the {@link ChannelHandler} which belongs to this context was removed
     * from the {@link ChannelPipeline}. Note that this method is only meant to be called from with in the
     * {@link EventLoop}.
     */
    boolean isRemoved();

    @Override
    ChannelHandlerContext fireChannelRegistered();

    @Override
    ChannelHandlerContext fireChannelUnregistered();

    @Override
    ChannelHandlerContext fireChannelActive();

    @Override
    ChannelHandlerContext fireChannelInactive();

    @Override
    ChannelHandlerContext fireExceptionCaught(Throwable cause);

    @Override
    ChannelHandlerContext fireUserEventTriggered(Object evt);

    @Override
    ChannelHandlerContext fireChannelRead(Object msg);

    @Override
    ChannelHandlerContext fireChannelReadComplete();

    @Override
    ChannelHandlerContext fireChannelWritabilityChanged();

    @Override
    ChannelHandlerContext read();

    @Override
    ChannelHandlerContext flush();

    /**
     * Return the assigned {@link ChannelPipeline}
     */
    ChannelPipeline pipeline();

    /**
     * Return the assigned {@link ByteBufAllocator} which will be used to allocate {@link ByteBuf}s.
     */
    ByteBufAllocator alloc();

    /**
     * @deprecated Use {@link Channel#attr(AttributeKey)}
     */
    @Deprecated
    @Override
    <T> Attribute<T> attr(AttributeKey<T> key);

    /**
     * @deprecated Use {@link Channel#hasAttr(AttributeKey)}
     */
    @Deprecated
    @Override
    <T> boolean hasAttr(AttributeKey<T> key);
}
```

主要继承了`AttributeMap`, `ChannelInboundInvoker`和`ChannelOutboundInvoker`。

### 2.3 head和tail

#### 2.3.1 head节点

首先查看它的构造函数：

```java
HeadContext(DefaultChannelPipeline pipeline) {
    super(pipeline, null, HEAD_NAME, false, true);
    unsafe = pipeline.channel().unsafe();
    setAddComplete();
}
```

继续查看父类构造函数：

```java
AbstractChannelHandlerContext(DefaultChannelPipeline pipeline, EventExecutor executor, String name,boolean inbound, boolean outbound) {
    this.name = ObjectUtil.checkNotNull(name, "name");
    this.pipeline = pipeline;
    this.executor = executor;
    this.inbound = inbound;
    this.outbound = outbound;
    // Its ordered if its driven by the EventLoop or the given Executor is an instanceof OrderedEventExecutor.
    ordered = executor == null || executor instanceof OrderedEventExecutor;
}
```

* 结合两者可以看出来，`head`节点是一个`outbound`的节点。 
* 之后，获取到channel的`unsafe`类。这个类根据Channel的不同，相应的函数逻辑可能也会有所不同。例如当Channel为服务端channel时，unsafe的read函数会读取新连接，为客户端channel时，会读取byte信息。
* 创建完`head`后，通过`setAddComplete()`将其设置为已经添加状态。

继续查看其它方法可以发现：

* `head`会把事件都往`tail`方向传播
* 在进行读写事件进行时，最终都会委托到`head`的`unsafe`进行操作

#### 2.3.2 tail节点

> A special catch-all handler that handles both bytes and messages.

首先看构造函数：

```java
TailContext(DefaultChannelPipeline pipeline) {
    super(pipeline, null, TAIL_NAME, true, false);
    setAddComplete();
}
```

继续查看父类构造函数：

```java
AbstractChannelHandlerContext(DefaultChannelPipeline pipeline, EventExecutor executor, String name,boolean inbound, boolean outbound) {
    this.name = ObjectUtil.checkNotNull(name, "name");
    this.pipeline = pipeline;
    this.executor = executor;
    this.inbound = inbound;
    this.outbound = outbound;
    // Its ordered if its driven by the EventLoop or the given Executor is an instanceof OrderedEventExecutor.
    ordered = executor == null || executor instanceof OrderedEventExecutor;
}
```

* 结合两者可以看出来，`tail`节点是一个`inbound`的节点。
* 创建完`tail`后，通过`setAddComplete()`将其设置为已经添加状态。

继续查看其它方法，可以看到大部分方法都为空。主要分析三个方法：

```java
@Override
public ChannelHandler handler() {
    return this;
}

@Override
public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
    onUnhandledInboundException(cause);
}

@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    onUnhandledInboundMessage(msg);
}
```

* `tail`对应的`handler`其实就是`tail`自身
* 在tail接受到异常或者是msg数据时，表示异常或读取的消息在前几个handler中没有处理，通过`onUnhandledInboundException`和`onUnhandledInboundMessage`方法会在log中打印消息进行提示。

可以看出来，**tail主要是做一个收尾的信息**。

## 3. 添加和删除ChannelHandler

### 3.1 添加Handler

首先查看一下用户通常添加Handler的操作：

```java
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
        .channel(NioServerSocketChannel.class)
        .childOption(ChannelOption.TCP_NODELAY, true)
        .childAttr(AttributeKey.newInstance("childAttr"), "childAttrValue")
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) {
                ch.pipeline().addLast(new OutBoundHandlerA());
                ch.pipeline().addLast(new OutBoundHandlerC());
                ch.pipeline().addLast(new OutBoundHandlerB());
            }
        });
```

可以看到主要是使用`addLast`方法。

#### 3.1.1 addLast方法

1. 判断是否重复添加
2. 创建节点并且添加至链表
3. 回调添加完成事件

首先一路跟进`addLast`方法，会得到：

```java
@Override
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        // 检测是否重复添加
        checkMultiplicity(handler);

        // 创建节点
        newCtx = newContext(group, filterName(name, handler), handler);

        // 添加至链表
        addLast0(newCtx);

        // If the registered is false it means that the channel was not registered on an eventloop yet.
        // In this case we add the context to the pipeline and add a task that will call
        // ChannelHandler.handlerAdded(...) once the channel is registered.
        if (!registered) {
            newCtx.setAddPending();
            callHandlerCallbackLater(newCtx, true);
            return this;
        }

        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            newCtx.setAddPending();
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    callHandlerAdded0(newCtx);
                }
            });
            return this;
        }
    }
    callHandlerAdded0(newCtx);
    return this;
}
```

可以看到有一个`checkMultiplicity(handler)`，用于检测是否重复添加：

```java
private static void checkMultiplicity(ChannelHandler handler) {
    if (handler instanceof ChannelHandlerAdapter) {
        ChannelHandlerAdapter h = (ChannelHandlerAdapter) handler;
        if (!h.isSharable() && h.added) {
            throw new ChannelPipelineException(
                    h.getClass().getName() +
                    " is not a @Sharable handler, so can't be added or removed multiple times.");
        }
        h.added = true;
    }
}
```

主要通过一个`added`变量来确定是否已经开始添加该handler。此外，还需要判断`isSharable()`。这里主要是观察Handler是否有`@Sharable`注解。 接着继续到第二步：创建节点并且添加至链表。首先查看`newContext`方法：

```java
private AbstractChannelHandlerContext newContext(EventExecutorGroup group, String name, ChannelHandler handler) {
    return new DefaultChannelHandlerContext(this, childExecutor(group), name, handler);
}
```

然后是`addLast0`方法：

```java
private void addLast0(AbstractChannelHandlerContext newCtx) {
    AbstractChannelHandlerContext prev = tail.prev;
    newCtx.prev = prev;
    newCtx.next = tail;
    prev.next = newCtx;
    tail.prev = newCtx;
}
```

在tail前面插入了新建的`newCtx`。到这里第二步就算是完成了。 最后是回调添加完成事件。这里的主要方法是`callHandlerAdded0`：

```java
private void callHandlerAdded0(final AbstractChannelHandlerContext ctx) {
    try {
        ctx.handler().handlerAdded(ctx);
        ctx.setAddComplete();
    } catch (Throwable t) {
        boolean removed = false;
        try {
            remove0(ctx);
            try {
                ctx.handler().handlerRemoved(ctx);
            } finally {
                ctx.setRemoved();
            }
            removed = true;
        } catch (Throwable t2) {
            if (logger.isWarnEnabled()) {
                logger.warn("Failed to remove a handler: " + ctx.name(), t2);
            }
        }

        if (removed) {
            fireExceptionCaught(new ChannelPipelineException(
                    ctx.handler().getClass().getName() +
                    ".handlerAdded() has thrown an exception; removed.", t));
        } else {
            fireExceptionCaught(new ChannelPipelineException(
                    ctx.handler().getClass().getName() +
                    ".handlerAdded() has thrown an exception; also failed to remove.", t));
        }
    }
}
```

* 首先是调用handler的`handlerAdded(ctx)`方法，这部分内容可以由用于自行编码实现
* 其次是通过CAS操作修改ctx的状态，设为已经添加。

有一个比较特殊的Handler——`ChannelInitializer`。查看它的`handlerAdded(ctx)`方法：

```java
@Override
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
    if (ctx.channel().isRegistered()) {
        // This should always be true with our current DefaultChannelPipeline implementation.
        // The good thing about calling initChannel(...) in handlerAdded(...) is that there will be no ordering
        // suprises if a ChannelInitializer will add another ChannelInitializer. This is as all handlers
        // will be added in the expected order.
        initChannel(ctx);
    }
}
```

继续看`initChannel(ctx)`：

```java
@SuppressWarnings("unchecked")
private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
    // 判断该handler是否已经被执行过
    if (initMap.putIfAbsent(ctx, Boolean.TRUE) == null) { // Guard against re-entrance.
        try {
            initChannel((C) ctx.channel());
        } catch (Throwable cause) {
            // Explicitly call exceptionCaught(...) as we removed the handler before calling initChannel(...).
            // We do so to prevent multiple calls to initChannel(...).
            exceptionCaught(ctx, cause);
        } finally {
            remove(ctx);
        }
        return true;
    }
    return false;
}
```

这里的`initChannel((C) ctx.channel())`就是由用户进行override的方法，例如添加一些其他handler等操作。在初始化完成后，调用`remove`将该节点从`pipeline`中移除。删除节点在后面会进行分析。

```java
private void remove(ChannelHandlerContext ctx) {
    try {
        ChannelPipeline pipeline = ctx.pipeline();
        if (pipeline.context(this) != null) {
            pipeline.remove(this);
        }
    } finally {
        initMap.remove(ctx);
    }
}
```

### 3.2 删除Handler

删除Handler主要经过三个步骤： 1. 找到节点 2. 链表节点的删除 3. 调用handler的删除回调函数

除了`ChannelInitializer`以外，还有一个场景需要进行删除handler的操作，就是权限校验。权限校验的例子：

```java
protected void channelRead0(ChannelHandlerContext ctx, ByteBuf password) throws Exception {
    // 密码正确，删除handler
    if (paas(password)) {
        ctx.pipeline().remove(this);
    } else {// 密码错误，关闭channel
        ctx.close();
    }
}
```

借此来分析删除handler的操作，首先跟进`remove(this)`:

```java
public final ChannelPipeline remove(ChannelHandler handler) {
    remove(getContextOrDie(handler));
    return this;
}
```

跟进`getContextOrDie`方法，这个方法获取到context或者是抛出异常：

```java
private AbstractChannelHandlerContext getContextOrDie(ChannelHandler handler) {
    AbstractChannelHandlerContext ctx = (AbstractChannelHandlerContext) context(handler);
    if (ctx == null) {
        throw new NoSuchElementException(handler.getClass().getName());
    } else {
        return ctx;
    }
}
```

继续跟进`context`：

```java
public final ChannelHandlerContext context(ChannelHandler handler) {
    if (handler == null) {
        throw new NullPointerException("handler");
    }

    AbstractChannelHandlerContext ctx = head.next;
    for (;;) {

        if (ctx == null) {
            return null;
        }

        if (ctx.handler() == handler) {
            return ctx;
        }

        ctx = ctx.next;
    }
}
```

即从`head`开始，顺序判断是否和所需handler是同一个handler。到这里就找到了我们所需要的的节点。 第二个过程是删除链表中的指定节点，继续查看`remove`方法：

```java
private AbstractChannelHandlerContext remove(final AbstractChannelHandlerContext ctx) {
    assert ctx != head && ctx != tail;

    synchronized (this) {
        // 删除链表中的ctx节点
        remove0(ctx);

        // If the registered is false it means that the channel was not registered on an eventloop yet.
        // In this case we remove the context from the pipeline and add a task that will call
        // ChannelHandler.handlerRemoved(...) once the channel is registered.
        if (!registered) {
            callHandlerCallbackLater(ctx, false);
            return ctx;
        }

        // 调用handler删除的回调函数
        EventExecutor executor = ctx.executor();
        if (!executor.inEventLoop()) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    callHandlerRemoved0(ctx);
                }
            });
            return ctx;
        }
    }
    callHandlerRemoved0(ctx);
    return ctx;
}
```

继续查看`remove0`:

```java
private static void remove0(AbstractChannelHandlerContext ctx) {
    AbstractChannelHandlerContext prev = ctx.prev;
    AbstractChannelHandlerContext next = ctx.next;
    prev.next = next;
    next.prev = prev;
}
```

这里也比较好理解。至此，第二部分的删除链表节点也完成了，还剩下最后一步，调用相应的回调函数。继续查看`callHandlerRemoved0`：

```java
private void callHandlerRemoved0(final AbstractChannelHandlerContext ctx) {
    // Notify the complete removal.
    try {
        try {
            ctx.handler().handlerRemoved(ctx);
        } finally {
            ctx.setRemoved();
        }
    } catch (Throwable t) {
        fireExceptionCaught(new ChannelPipelineException(
                ctx.handler().getClass().getName() + ".handlerRemoved() has thrown an exception.", t));
    }
}
```

* 调用handler的`handlerRemoved`方法
* 设置ctx的状态为`removed`状态

