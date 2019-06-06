---
title: Flink源码解析之数据流转
date: 2019-06-06 17:23:29
tags:
  - Flink
---

## 数据流转——Flink的数据抽象及数据交换过程

本章打算讲一下flink底层是如何定义和在操作符之间传递数据的。

<br/>

### 1. flink的数据抽象

<br/>

#### 1.1 MemorySegment

Flink作为一个高效的流框架，为了避免JVM的固有缺陷（java对象存储密度低，FGC影响吞吐 和响应等），必然走上自主管理内存的道路。

这个 MemorySegment 就是Flink的内存抽象。默认情况下，一个MemorySegment可以被看做 是一个32kb大的内存块的抽象。这块内存既可以是JVM里的一个byte[]，也可以是堆外内存 （DirectByteBuffer）。

如果说byte[]数组和direct memory是最底层的存储，那么memorysegment就是在其上覆盖 的一层统一抽象。它定义了一系列抽象方法，用于控制和底层内存的交互，如：

```java
public abstract class MemorySegment {

    protected final byte[] heapMemory;

    /**
	 * The size in bytes of the memory segment.
	 */
    protected final int size;

    /**
	 * Optional owner of the memory segment.
	 */
    private final Object owner;

    // ------------------------------------------------------------------------
    // Memory Segment Operations
    // ------------------------------------------------------------------------

    /**
	 * Gets the size of the memory segment, in bytes.
	 *
	 * @return The size of the memory segment.
	 */
    public int size() {
        return size;
    }


    public void free() {
        // this ensures we can place no more data and trigger
        // the checks for the freed segment
        address = addressLimit + 1;
    }
    */
        public abstract ByteBuffer wrap(int offset, int length);
    .....
```

我们可以看到，它在提供了诸多直接操作内存的方法外，还提供了一个 wrap() 方法，将自己 包装成一个ByteBuffer，我们待会儿讲这个ByteBuffer。

Flink为MemorySegment提供了两个实现 类： HeapMemorySegment 和 HybridMemorySegment 。他们的区别在于前者只能分配堆内存，而后者能用来分配堆内和堆外内存。事实上，Flink框架里，只使用了后者。这是为什么呢？

如果HybridMemorySegment只能用于分配堆外内存的话，似乎更合常理。但是在JVM的世 界中，如果一个方法是一个虚方法，那么每次调用时，JVM都要花时间去确定调用的到底是哪 个子类实现的该虚方法（方法重写机制，不明白的去看JVM的invokeVirtual指令），也就意 味着每次都要去翻方法表；而如果该方法虽然是个虚方法，但实际上整个JVM里只有一个实现 （就是说只加载了一个子类进来），那么JVM会很聪明的把它去虚化处理，这样就不用每次调 用方法时去找方法表了，能够大大提升性能。但是只分配堆内或者堆外内存不能满足我们的需 要，所以就出现了HybridMemorySegment同时可以分配两种内存的设计。

我们可以看看HybridMemorySegment的构造代码：

```java
HybridMemorySegment(ByteBuffer buffer, Object owner) {
    super(checkBufferAndGetAddress(buffer), buffer.capacity(), owner);
    this.offHeapBuffer = buffer;
}
HybridMemorySegment(byte[] buffer, Object owner) {
    super(buffer, owner);
    this.offHeapBuffer = null;
}
```

其中，第一个构造函数的 checkBufferAndGetAddress() 方法能够得到direct buffer的内存地 址，因此可以操作堆外内存。

<br/>

#### 1.2 ByteBuffer 与 NetworkBufferPool

在 MemorySegment 这个抽象之上，Flink在数据从operator内的数据对象在向TaskManager上 转移，预备被发给下个节点的过程中，使用的抽象或者说内存对象是 Buffer 。

注意，这个Buffer是个flink接口，不是java.nio提供的那个Buffer抽象类。Flink在这一层面同 时使用了这两个同名概念，用来存储对象，直接看代码时到处都是各种xxxBuffer很容易混 淆：

- java提供的那个Buffer抽象类在这一层主要用于构建 HeapByteBuffer ，这个主要是当数据 从jvm里的一个对象被序列化成字节数组时用的；
- Flink的这个Buffer接口主要是一种 flink 层面用于传输数据和事件的统一抽象，其实现类 是 NetworkBuffer ，是对 MemorySegment 的包装。Flink在各个TaskManager之间传递数 据时，使用的是这一层的抽象。

因为Buffer的底层是MemorySegment，这可能不是JVM所管理的，所以为了知道什么时候一 个Buffer用完了可以回收，Flink引入了引用计数的概念，当确认这个buffer没有人引用，就可以回收这一片MemorySegment用于别的地方了。

```java
public abstract class AbstractReferenceCountedByteBuf extends AbstractByteBuf {
    private static final AtomicIntegerFieldUpdater<AbstractReferenceCountedByteBuf> refCntUpdater;
    private volatile int refCnt = 1;
   ....
```

为了方便管理 NetworkBuffer ，Flink 提供了 BufferPoolFactory ，并且提供了唯一实现 NetworkBufferPool ，这是个工厂模式的应用。

NetworkBufferPool在每个TaskManager上只有一个，负责所有子task的内存管理。其实例化时就会尝试获取所有可由它管理的内存（对于堆内存来说，直接获取所有内存并放入老年代，并令用户对象只在新生代存活，可以极大程度的减少Full GC），我们看看其构造方法：

```java
public NetworkBufferPool(int numberOfSegmentsToAllocate, int segmentSize) {

    this.totalNumberOfMemorySegments = numberOfSegmentsToAllocate;
    this.memorySegmentSize = segmentSize;

    final long sizeInLong = (long) segmentSize;

    try {
        this.availableMemorySegments = new ArrayBlockingQueue<>(numberOfSegmentsToAllocate);
    }
    catch (OutOfMemoryError err) {
        throw new OutOfMemoryError("Could not allocate buffer queue of length "
                                   + numberOfSegmentsToAllocate + " - " + err.getMessage());
    }

    try {
        for (int i = 0; i < numberOfSegmentsToAllocate; i++) {
            ByteBuffer memory = ByteBuffer.allocateDirect(segmentSize);
            availableMemorySegments.add(MemorySegmentFactory.wrapPooledOffHeapMemory(memory, null));
        }
    }
   ......

    long allocatedMb = (sizeInLong * availableMemorySegments.size()) >> 20;

    LOG.info("Allocated {} MB for network buffer pool (number of memory segments: {}, bytes per segment: {}).",
             allocatedMb, availableMemorySegments.size(), segmentSize);
}
```

<br/>

由于NetworkBufferPool只是个工厂，实际的内存池是 LocalBufferPool 。每个 TaskManager都只有一个NetworkBufferPool工厂，但是上面运行的每个task都要有一个和 其他task隔离的LocalBufferPool池，这从逻辑上很好理解。另外，NetworkBufferPool会计算自己所拥有的所有内存分片数，在分配新的内存池时对每个内存池应该占有的内存分片数重分配，步骤是：

- 首先，从整个工厂管理的内存片中拿出所有的内存池所需要的最少Buffer数目总和
- 如果正好分配完，就结束
- 其次，把所有的剩下的没分配的内存片，按照每个LocalBufferPool内存池的剩余想要容量 大小进行按比例分配
- 剩余想要容量大小是这么个东西：如果该内存池至少需要3个buffer，最大需要10个 buffer，那么它的剩余想要容量就是7

实现代码如下：

```java
/**
	 * NetworkBufferPool只是个工厂，实际的内存池是 LocalBufferPool
	 * 每个TaskManager都只有一个NetworkBufferPool工厂，但是上面运行的每个task都要有一个和其他task隔离的LocalBufferPool池，
	 * 这个方法的作用就是：分配新的内存池(新增一个task)时所需要做的操作，也就是对所有内存分片(NetworkBuffer) 重分配的过程
	 */
private void redistributeBuffers() throws IOException {
    assert Thread.holdsLock(factoryLock);

    // All buffers, which are not among the required ones
    final int numAvailableMemorySegment = totalNumberOfMemorySegments - numTotalRequiredBuffers;

    if (numAvailableMemorySegment == 0) {
        // in this case, we need to redistribute buffers so that every pool gets its minimum
        for (LocalBufferPool bufferPool : allBufferPools) {
            bufferPool.setNumBuffers(bufferPool.getNumberOfRequiredMemorySegments());
        }
        return;
    }

    /*
		 * With buffer pools being potentially limited, let's distribute the available memory
		 * segments based on the capacity of each buffer pool, i.e. the maximum number of segments
		 * an unlimited buffer pool can take is numAvailableMemorySegment, for limited buffer pools
		 * it may be less. Based on this and the sum of all these values (totalCapacity), we build
		 * a ratio that we use to distribute the buffers.
		 */

    long totalCapacity = 0; // long to avoid int overflow

    for (LocalBufferPool bufferPool : allBufferPools) {
        int excessMax = bufferPool.getMaxNumberOfMemorySegments() -
            bufferPool.getNumberOfRequiredMemorySegments();
        totalCapacity += Math.min(numAvailableMemorySegment, excessMax);
    }

    // no capacity to receive additional buffers?
    if (totalCapacity == 0) {
        return; // necessary to avoid div by zero when nothing to re-distribute
    }

    // since one of the arguments of 'min(a,b)' is a positive int, this is actually
    // guaranteed to be within the 'int' domain
    // (we use a checked downCast to handle possible bugs more gracefully).
    final int memorySegmentsToDistribute = MathUtils.checkedDownCast(
        Math.min(numAvailableMemorySegment, totalCapacity));

    long totalPartsUsed = 0; // of totalCapacity
    int numDistributedMemorySegment = 0;
    for (LocalBufferPool bufferPool : allBufferPools) {
        int excessMax = bufferPool.getMaxNumberOfMemorySegments() -
            bufferPool.getNumberOfRequiredMemorySegments();

        // shortcut
        if (excessMax == 0) {
            continue;
        }

        totalPartsUsed += Math.min(numAvailableMemorySegment, excessMax);

        // avoid remaining buffers by looking at the total capacity that should have been
        // re-distributed up until here
        // the downcast will always succeed, because both arguments of the subtraction are in the 'int' domain
        final int mySize = MathUtils.checkedDownCast(
            memorySegmentsToDistribute * totalPartsUsed / totalCapacity - numDistributedMemorySegment);

        numDistributedMemorySegment += mySize;
        bufferPool.setNumBuffers(bufferPool.getNumberOfRequiredMemorySegments() + mySize);
    }

    assert (totalPartsUsed == totalCapacity);
    assert (numDistributedMemorySegment == memorySegmentsToDistribute);
}
```

接下来说说这个 LocalBufferPool 内存池。 LocalBufferPool的逻辑想想无非是增删改查，值得说的是其fields：

```java
/** Global network buffer pool to get buffers from. */
private final NetworkBufferPool networkBufferPool;

/** 该内存池需要的最少内存片数目*/
private final int numberOfRequiredMemorySegments;

/** * 当前已经获得的内存片中，还没有写入数据的空白内存片 */
private final ArrayDeque<MemorySegment> availableMemorySegments = new ArrayDeque<MemorySegment>();

/** * 注册的所有监控buffer可用性的监听器 */
private final ArrayDeque<BufferListener> registeredListeners = new ArrayDeque<>();

/** 能给内存池分配的最大分片数*/
private final int maxNumberOfMemorySegments;

/** 当前内存池大小 */
private int currentPoolSize;

/** * 所有经由NetworkBufferPool分配的，被本内存池引用到的（非直接获得的）分片数 */
private int numberOfRequestedMemorySegments;

private boolean isDestroyed;

private BufferPoolOwner owner;
```

承接NetworkBufferPool的重分配方法，我们来看看LocalBufferPool的 setNumBuffers() 方 法，代码很短，逻辑也相当简单，就不展开说了：

```java
@Override
public void setNumBuffers(int numBuffers) throws IOException {
    synchronized (availableMemorySegments) {
        checkArgument(numBuffers >= numberOfRequiredMemorySegments,
                      "Buffer pool needs at least %s buffers, but tried to set to %s",
                      numberOfRequiredMemorySegments, numBuffers);

        if (numBuffers > maxNumberOfMemorySegments) {
            currentPoolSize = maxNumberOfMemorySegments;
        } else {
            currentPoolSize = numBuffers;
        }

        returnExcessMemorySegments();

        // If there is a registered owner and we have still requested more buffers than our
        // size, trigger a recycle via the owner.
        if (owner != null && numberOfRequestedMemorySegments > currentPoolSize) {
            owner.releaseMemory(numberOfRequestedMemorySegments - currentPoolSize);
        }
    }
}
```

<br/>

#### 1.3 RecordWriter与Record

我们接着往高层抽象走，刚刚提到了最底层内存抽象是MemorySegment，用于数据传输的 是Buffer，那么，承上启下对接从Java对象转为Buffer的中间对象是什么呢？

是 StreamRecord 。

从 StreamRecord<T> 这个类名字就可以看出来，这个类就是个wrap，里面保存了原始的Java 对象。另外，StreamRecord还保存了一个timestamp。

那么这个对象是怎么变成 LocalBufferPool 内存池里的一个大号字节数组的呢？借助了 RecordWriter 这个类。

我们直接来看把数据序列化交出去的方法：

```java
private void sendToTarget(T record, int targetChannel) throws IOException, InterruptedException {
    RecordSerializer<T> serializer = serializers[targetChannel];

    SerializationResult result = serializer.addRecord(record);

    while (result.isFullBuffer()) {
        if (tryFinishCurrentBufferBuilder(targetChannel, serializer)) {
            // If this was a full record, we are done. Not breaking
            // out of the loop at this point will lead to another
            // buffer request before breaking out (that would not be
            // a problem per se, but it can lead to stalls in the
            // pipeline).
            if (result.isFullRecord()) {
                break;
            }
        }
        BufferBuilder bufferBuilder = requestNewBufferBuilder(targetChannel);

        result = serializer.continueWritingWithNextBufferBuilder(bufferBuilder);
    }
    checkState(!serializer.hasSerializedData(), "All data should be written at once");

    if (flushAlways) {
        targetPartition.flush(targetChannel);
    }
}
```

先说最后一行，如果配置为flushAlways，那么会立刻把元素发送出去，但是这样吞吐量会下 降；Flink的默认设置其实也不是一个元素一个元素的发送，是单独起了一个线程，每隔固定时 间flush一次所有channel，较真起来也算是mini batch了。

再说序列化那一句: SerializationResult result = serializer.addRecord(record); 。在 这行代码中，Flink把对象调用该对象所属的序列化器序列化为字节数组。

<br/>

### 2. 数据流转过程

上一节讲了各层数据的抽象，这一节讲讲数据在各个task之间exchange的过程。

<br/>

#### 2.1 整体过程

看这张图：

{% asset_img 一.jpg %}

1. 第一步必然是准备一个ResultPartition；
2. 通知JobMaster；
3. JobMaster通知下游节点；如果下游节点尚未部署，则部署之；
4. 下游节点向上游请求数据
5. 开始传输数据

<br/>

#### 2.2 数据跨task传递

本节讲一下算子之间具体的数据传输过程。也先上一张图：

{% asset_img 二.jpg %}

数据在task之间传递有如下几步：

1. 数据在本operator处理完后，交给 RecordWriter 。每条记录都要选择一个下游节点，所 以要经过 ChannelSelector 。
2. 每个channel都有一个serializer（我认为这应该是为了避免多线程写的麻烦），把这条 Record序列化为ByteBuffer
3. 接下来数据被写入ResultPartition下的各个subPartition里，此时该数据已经存入 DirectBuffer（MemorySegment）
4. 单独的线程控制数据的flush速度，一旦触发flush，则通过Netty的nio通道向对端写入
5. 对端的netty client接收到数据，decode出来，把数据拷贝到buffer里，然后通 知 InputChannel
6. 有可用的数据时，下游算子从阻塞醒来，从InputChannel取出buffer，再解序列化成 record，交给算子执行用户代码

<br/>

数据在不同机器的算子之间传递的步骤就是以上这些。

了解了步骤之后，再来看一下部分关键代码：

首先是把数据交给recordwriter。

```java
//RecordWriterOutput.java
@Override
public void collect(StreamRecord<OUT> record) {
    if (this.outputTag != null) {
        // we are only responsible for emitting to the main input
        return;
    }
	//这里可以看到把记录交给了recordwriter
    pushToRecordWriter(record);
}
```

然后recordwriter把数据发送到对应的通道。

```java
//RecordWriter.java
public void emit(T record) throws IOException, InterruptedException {
    //channelselector登场了
    for (int targetChannel : channelSelector.selectChannels(record, numChannels)) {
        sendToTarget(record, targetChannel);
    }
}
private void sendToTarget(T record, int targetChannel) throws IOException, InterruptedException {
    //选择序列化器并序列化数据
    RecordSerializer<T> serializer = serializers[targetChannel];

    SerializationResult result = serializer.addRecord(record);

    while (result.isFullBuffer()) {
        if (tryFinishCurrentBufferBuilder(targetChannel, serializer)) {
            // If this was a full record, we are done. Not breaking
            // out of the loop at this point will lead to another
            // buffer request before breaking out (that would not be
            // a problem per se, but it can lead to stalls in the
            // pipeline).
            if (result.isFullRecord()) {
                break;
            }
        }
        BufferBuilder bufferBuilder = requestNewBufferBuilder(targetChannel);

        //写入channel
        result = serializer.continueWritingWithNextBufferBuilder(bufferBuilder);
    }
    checkState(!serializer.hasSerializedData(), "All data should be written at once");

    if (flushAlways) {
        targetPartition.flush(targetChannel);
    }
}
```

接下来是把数据推给底层设施（netty）的过程：

```java
//ResultPartition.java
@Override
public void flushAll() {
    for (ResultSubpartition subpartition : subpartitions) {
        subpartition.flush();
    }
}
//PartitionRequestQueue.java
void notifyReaderNonEmpty(final NetworkSequenceViewReader reader) {
    //这里交给了netty server线程去推
    ctx.executor().execute(new Runnable() {
        @Override
        public void run() {
            ctx.pipeline().fireUserEventTriggered(reader);
        }
    });
}
```

netty相关的部分：

```java
//AbstractChannelHandlerContext.java
public ChannelHandlerContext fireChannelRegistered() {
    final AbstractChannelHandlerContext next = findContextInbound();
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeChannelRegistered();
    } else {
        executor.execute(new OneTimeTask() {
            @Override
            public void run() {
                next.invokeChannelRegistered();
            }
        });
    }
    return this;
}
```

最后真实的写入：

```java
//PartittionRequesetQueue.java
private void enqueueAvailableReader(final NetworkSequenceViewReader reader) throws Exception {
    if (reader.isRegisteredAsAvailable() || !reader.isAvailable()) {
        return;
    }
    // Queue an available reader for consumption. If the queue is empty,
    // we try trigger the actual write. Otherwise this will be handled by
    // the writeAndFlushNextMessageIfPossible calls.
    boolean triggerWrite = availableReaders.isEmpty();
    registerAvailableReader(reader);

    if (triggerWrite) {
        writeAndFlushNextMessageIfPossible(ctx.channel());
    }
}

private void writeAndFlushNextMessageIfPossible(final Channel channel) throws IOException {
    ....
            next = reader.getNextBuffer();
            if (next == null) {
                if (!reader.isReleased()) {
                    continue;
                }
                markAsReleased(reader.getReceiverId());

                Throwable cause = reader.getFailureCause();
                if (cause != null) {
                    ErrorResponse msg = new ErrorResponse(
                        new ProducerFailedException(cause),
                        reader.getReceiverId());

                    ctx.writeAndFlush(msg);
                }
            } else {
                // This channel was now removed from the available reader queue.
                // We re-add it into the queue if it is still available
                if (next.moreAvailable()) {
                    registerAvailableReader(reader);
                }

                BufferResponse msg = new BufferResponse(
                    next.buffer(),
                    reader.getSequenceNumber(),
                    reader.getReceiverId(),
                    next.buffersInBacklog());

                if (isEndOfPartitionEvent(next.buffer())) {
                    reader.notifySubpartitionConsumed();
                    reader.releaseAllResources();

                    markAsReleased(reader.getReceiverId());
                }

                // Write and flush and wait until this is done before
                // trying to continue with the next buffer.
                channel.writeAndFlush(msg).addListener(writeListener);

                return;
            }
        }
    } catch (Throwable t) {
        if (next != null) {
            next.buffer().recycleBuffer();
        }

        throw new IOException(t.getMessage(), t);
    }
}
```

上面这段代码里第二个方法中调用的 writeAndFlush(msg) 就是真正往netty的nio通道里写入 的地方了。在这里，写入的是一个RemoteInputChannel，对应的就是下游节点的InputGate 的channels。

有写就有读，nio通道的另一端需要读入buffer，代码如下：

```java
//CreditBasedPartitionRequestClientHandler.java
private void decodeMsg(Object msg) throws Throwable {
    final Class<?> msgClazz = msg.getClass();

    // ---- Buffer --------------------------------------------------------
    if (msgClazz == NettyMessage.BufferResponse.class) {
        NettyMessage.BufferResponse bufferOrEvent = (NettyMessage.BufferResponse) msg;

        RemoteInputChannel inputChannel = inputChannels.get(bufferOrEvent.receiverId);
        if (inputChannel == null) {
            bufferOrEvent.releaseBuffer();

            cancelRequestFor(bufferOrEvent.receiverId);

            return;
        }

        decodeBufferOrEvent(inputChannel, bufferOrEvent);
    } 
}
```

插一句，Flink其实做阻塞和获取数据的方式非常自然，利用了生产者和消费者模型，当获取不 到数据时，消费者自然阻塞；当数据被加入队列，消费者被notify。Flink的背压机制也是借此 实现。

然后在这里又反序列化成 StreamRecord ：

```java
//StreamElementSerializer.java
public StreamElement deserialize(DataInputView source) throws IOException {
    int tag = source.readByte();
    if (tag == TAG_REC_WITH_TIMESTAMP) {
        long timestamp = source.readLong();
        return new StreamRecord<T>(typeSerializer.deserialize(source), timestamp);
    }
    else if (tag == TAG_REC_WITHOUT_TIMESTAMP) {
        return new StreamRecord<T>(typeSerializer.deserialize(source));
    }
    else if (tag == TAG_WATERMARK) {
        return new Watermark(source.readLong());
    }
    else if (tag == TAG_STREAM_STATUS) {
        return new StreamStatus(source.readInt());
    }
    else if (tag == TAG_LATENCY_MARKER) {
        return new LatencyMarker(source.readLong(), new OperatorID(source.readLong(), source.readLong()), source.readInt());
    }
    else {
        throw new IOException("Corrupt stream, found tag: " + tag);
    }
}
```

然后再次在 StreamInputProcessor.processInput() 循环中得到处理。

至此，数据在跨jvm的节点之间的流转过程就讲完了。

<br/>

### 3. Credit漫谈

在看上一部分的代码时，有一个小细节不知道读者有没有注意到，我们的数据发送端的代码叫做PartittionRequesetQueue.java ，而我们的接收端却起了一个完全不相干的名 字：CreditBasedPartitionRequestClientHandler.java 。为什么前面加了CreditBased的前缀呢？

<br/>

#### 3.1 背压问题

在流模型中，我们期待数据是像水流一样平滑的流过我们的引擎，但现实生活不会这么美好。 数据的上游可能因为各种原因数据量暴增，远远超出了下游的瞬时处理能力（回忆一下98年大 洪水），导致系统崩溃。

那么框架应该怎么应对呢？和人类处理自然灾害的方式类似，我们修建了三峡大坝，当洪水来 临时把大量的水囤积在大坝里；对于Flink来说，就是在数据的接收端和发送端放置了缓存池， 用以缓冲数据，并且设置闸门阻止数据向下流。

那么Flink又是如何处理背压的呢？答案也是靠这些缓冲池。

{% asset_img 三.jpg %}

这张图说明了Flink在生产和消费数据时的大致情况。 ResultPartition 和 InputGate 在输出 和输入数据时，都要向 NetworkBufferPool 申请一块 MemorySegment 作为缓存池。

接下来的情况和生产者消费者很类似。当数据发送太多，下游处理不过来了，那么首先InputChannel会被填满，然后是InputChannel能申请到的内存达到最大，于是下游停止读取数据，上游负责发送数据的nettyServer会得到响应停止从ResultSubPartition读取缓存，那么ResultPartition很快也将存满数据不能被消费，从而生产数据的逻辑被阻塞在获取新buffer 上，非常自然地形成背压的效果。

Flink自己做了个试验用以说明这个机制的效果：

{% asset_img 四.jpg %}

我们首先设置生产者的发送速度为60%，然后下游的算子以同样的速度处理数据。然后我们将 下游算子的处理速度降低到30%，可以看到上游的生产者的数据产生曲线几乎与消费者同步下滑。而后当我们解除限速，整个流的速度立刻提高到了100%。