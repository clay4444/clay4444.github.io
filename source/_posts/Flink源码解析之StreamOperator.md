---
title: Flink源码解析之StreamOperator
tags:
  - Flink
abbrlink: 118b9a37
date: 2019-06-06 11:48:11
---

## StreamOperator的抽象与实现

<br/>

### 1. 数据源的逻辑——StreamSource与时间模型

StreamSource抽象了一个数据源，并且指定了一些如何处理数据的模式。

```java
public class StreamSource<OUT, SRC extends SourceFunction<OUT>>
		extends AbstractUdfStreamOperator<OUT, SRC> implements StreamOperator<OUT> {

	....

	public void run(final Object lockingObject, final StreamStatusMaintainer streamStatusMaintainer) throws Exception {
		run(lockingObject, streamStatusMaintainer, output);
	}

	public void run(final Object lockingObject,
			final StreamStatusMaintainer streamStatusMaintainer,
			final Output<StreamRecord<OUT>> collector) throws Exception {

		final TimeCharacteristic timeCharacteristic = getOperatorConfig().getTimeCharacteristic();

		LatencyMarksEmitter latencyEmitter = null;
		if (getExecutionConfig().isLatencyTrackingEnabled()) {
			latencyEmitter = new LatencyMarksEmitter<>(
				getProcessingTimeService(),
				collector,
				getExecutionConfig().getLatencyTrackingInterval(),
				this.getOperatorID(),
				getRuntimeContext().getIndexOfThisSubtask());
		}

		final long watermarkInterval = getRuntimeContext().getExecutionConfig().getAutoWatermarkInterval();

		this.ctx = StreamSourceContexts.getSourceContext(
			timeCharacteristic,
			getProcessingTimeService(),
			lockingObject,
			streamStatusMaintainer,
			collector,
			watermarkInterval,
			-1);

		try {
			//在StreamSource生成上下文之后，接下来就是把上下文交给SourceFunction去执行:
			userFunction.run(ctx);

			// if we get here, then the user function either exited after being done (finite source)
			// or the function was canceled or stopped. For the finite source case, we should emit
			// a final watermark that indicates that we reached the end of event-time
			if (!isCanceledOrStopped()) {
				ctx.emitWatermark(Watermark.MAX_WATERMARK);
			}
		} finally {
			// make sure that the context is closed in any case
			ctx.close();
			if (latencyEmitter != null) {
				latencyEmitter.close();
			}
		}
	}

	....

	private static class LatencyMarksEmitter<OUT> {
		private final ScheduledFuture<?> latencyMarkTimer;

		public LatencyMarksEmitter(
				final ProcessingTimeService processingTimeService,
				final Output<StreamRecord<OUT>> output,
				long latencyTrackingInterval,
				final OperatorID operatorId,
				final int subtaskIndex) {

			latencyMarkTimer = processingTimeService.scheduleAtFixedRate(
				new ProcessingTimeCallback() {
					@Override
					public void onProcessingTime(long timestamp) throws Exception {
						try {
							// ProcessingTimeService callbacks are executed under the checkpointing lock
							output.emitLatencyMarker(new LatencyMarker(timestamp, operatorId, subtaskIndex));
						} catch (Throwable t) {
							// we catch the Throwables here so that we don't trigger the processing
							// timer services async exception handler
							LOG.warn("Error while emitting latency marker.", t);
						}
					}
				},
				0L,
				latencyTrackingInterval);
		}

		public void close() {
			latencyMarkTimer.cancel(true);
		}
	}
}
```

<br/>

在StreamSource生成上下文之后，接下来就是把上下文交给SourceFunction去执行:

```java
userFunction.run(ctx);
```

SourceFunction是对Function的一个抽象，就好像MapFunction，KeyByFunction一样，用 户选择实现这些函数，然后flink框架就能利用这些函数进行计算，完成用户逻辑。 我们的wordcount程序使用了flink提供的一个 SocketTextStreamFunction 。我们可以看一下它的实现逻辑，对source如何运行有一个基本的认识：

```java
/**
	 * 真正执行的方法，这个run 方法 是 Function 基类的，不是Thread的，
	 * tm 反射初始化 invokable 类，也就是几种task中的的一种（flink中制定了所有类型的task，比如 StreamTask），然后调用 StreamTask 的invokable方法，
	 * 最终调用 StreamTask 的run方法，最终调用到这里；
	 */
@Override
public void run(SourceContext<String> ctx) throws Exception {
    final StringBuilder buffer = new StringBuilder();
    long attempt = 0;

    while (isRunning) {

        try (Socket socket = new Socket()) {
            currentSocket = socket;

            LOG.info("Connecting to server socket " + hostname + ':' + port);
            socket.connect(new InetSocketAddress(hostname, port), CONNECTION_TIMEOUT_TIME);
            try (BufferedReader reader = new BufferedReader(new InputStreamReader(socket.getInputStream()))) {

                char[] cbuf = new char[8192];
                int bytesRead;
                //核心逻辑就是一直读inputSocket，然后交给collect方法
                while (isRunning && (bytesRead = reader.read(cbuf)) != -1) {
                    buffer.append(cbuf, 0, bytesRead);
                    int delimPos;
                    while (buffer.length() >= delimiter.length() && (delimPos = buffer.indexOf(delimiter)) != -1) {
                        String record = buffer.substring(0, delimPos);
                        // truncate trailing carriage return
                        if (delimiter.equals("\n") && record.endsWith("\r")) {
                            record = record.substring(0, record.length() - 1);
                        }
                        //读到数据后，把数据交给collect方法，collect方法负责 把数据交到合适的位置（如发布为br变量，或者交给下个operator，或者通过网络发出去）
                        //整段代码里，只有collect方法有些复杂度，后面我们在讲到flink的对象机制时会结合来讲，此处知道collect方法会收集结果，然后发送给接收者即可
                        ctx.collect(record);
                        buffer.delete(0, delimPos + delimiter.length());
                    }
                }
            }
        }

        // if we dropped out of this loop due to an EOF, sleep and retry
        if (isRunning) {
            attempt++;
            if (maxNumRetries == -1 || attempt < maxNumRetries) {
                LOG.warn("Lost connection to server socket. Retrying in " + delayBetweenRetries + " msecs...");
                Thread.sleep(delayBetweenRetries);
            }
            else {
                // this should probably be here, but some examples expect simple exists of the stream source
                // throw new EOFException("Reached end of stream and reconnects are not enabled.");
                break;
            }
        }
    }

    // collect trailing data
    if (buffer.length() > 0) {
        ctx.collect(buffer.toString());
    }
}
```

整段代码里，只有collect方法有些复杂度，后面我们在讲到flink的对象机制时会结合来讲，此 处知道collect方法会收集结果，然后发送给接收者即可。在我们的wordcount里，这个算子 的接收者就是被chain在一起的flatmap算子，不记得这个示例程序的话，可以返回第一章去看 一下。

<br/>

### 2. 从数据输入到数据处理——OneInputStreamOperator & AbstractUdfStreamOperator

StreamSource是用来开启整个流的算子，而承接输入数据并进行处理的算子就是 OneInputStreamOperator、TwoInputStreamOperator等。

{% asset_img 一.jpg %}

<br/>

整个StreamOperator的继承关系如上图所示（图很大，建议点开放大看）。 OneInputStreamOperator这个接口的逻辑很简单：

```java
public interface OneInputStreamOperator<IN, OUT> extends StreamOperator<OUT> {

    /**
	 * Processes one element that arrived at this operator.
	 * This method is guaranteed to not be called concurrently with other methods of the operator.
	 */
    void processElement(StreamRecord<IN> element) throws Exception;

    /**
	 * Processes a {@link Watermark}.
	 * This method is guaranteed to not be called concurrently with other methods of the operator.
	 *
	 * @see org.apache.flink.streaming.api.watermark.Watermark
	 */
    void processWatermark(Watermark mark) throws Exception;

    void processLatencyMarker(LatencyMarker latencyMarker) throws Exception;
}
```

<br/>

而实现了这个接口的StreamFlatMap算子也很简单，没什么可说的：

```java
public class StreamFlatMap<IN, OUT>
		extends AbstractUdfStreamOperator<OUT, FlatMapFunction<IN, OUT>>
		implements OneInputStreamOperator<IN, OUT> {

	private static final long serialVersionUID = 1L;

	private transient TimestampedCollector<OUT> collector;

	public StreamFlatMap(FlatMapFunction<IN, OUT> flatMapper) {
		super(flatMapper);
		chainingStrategy = ChainingStrategy.ALWAYS;
	}

	@Override
	public void open() throws Exception {
		super.open();
		collector = new TimestampedCollector<>(output);
	}

	@Override
	public void processElement(StreamRecord<IN> element) throws Exception {
		collector.setTimestamp(element);
		userFunction.flatMap(element.getValue(), collector);
	}
}
```

从类图里可以看到，flink为我们封装了一个算子的基类 AbstractUdfStreamOperator ，提供 了一些通用功能，比如把context赋给算子，保存快照等等，其中最为大家了解的应该是这两 个：

```java
@Override
public void open() throws Exception {
    super.open();
    FunctionUtils.openFunction(userFunction, new Configuration());
}

@Override
public void close() throws Exception {
    super.close();
    functionsClosed = true;
    FunctionUtils.closeFunction(userFunction);
}
```

这两个就是flink提供的 Rich***Function 系列算子的open和close方法被执行的地方。

<br/>

### 3. StreamSink

StreamSink着实没什么可说的，逻辑很简单，值得一提的只有两个方法：

```java
// 继承自StreamOperator的方法
@Override
public void processElement(StreamRecord<IN> element) throws Exception {
    sinkContext.element = element;
    userFunction.invoke(element.getValue(), sinkContext);
}

//用来计算延迟的，前面提到StreamSource会产生
//LateMarker，用于记录数据计算时间，就是在这里完成了计算。
@Override
protected void reportOrForwardLatencyMarker(LatencyMarker marker) {
    // all operators are tracking latencies
    this.latencyStats.reportLatency(marker);

    // sinks don't forward latency markers
}
```

其中， processElement 是继承自StreamOperator的方 法。 reportOrForwardLatencyMarker 是用来计算延迟的，前面提到StreamSource会产生

LateMarker，用于记录数据计算时间，就是在这里完成了计算。

<br/>

### 4. 其他算子