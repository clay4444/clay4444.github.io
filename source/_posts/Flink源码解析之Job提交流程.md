---
title: Flink源码解析之Job提交流程
tags:
  - Flink
abbrlink: c75f56
date: 2019-06-05 15:30:27
---

### 1.  wordcount 例子

```java
public class SocketTextStreamWordCount {
    public static void main(String[] args) throws Exception {
        /*if (args.length != 2) {
            System.err.println("USAGE:\nSocketTextStreamWordCount <hostname> <port>");
            return;
        }*/
        String hostName = "localhost";
        Integer port = 8088;

        // set up the execution environment
        /**
         * 这行代码会返回一个可用的执行环境。执行环境是整个flink程序执行的上下文，记录了相关配置（如并行度等），
         * 并提供了一系列方法，如读取输入流的方法，以及真正开始运行整个代码 的execute方法等。
         */
        final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // get input data
        DataStream<String> text = env.socketTextStream(hostName, port);

        text.flatMap(new LineSplitter()).setParallelism(1) // group by the tuple field "0" and sum up tuple field "1"
            .keyBy(0).sum(1).setParallelism(1).print();

        // execute program
        env.execute("Java WordCount from SocketTextStream Example");

    }

    /**
     * Implements the string tokenizer that splits sentences into words as a user-defined
     * <p>
     * FlatMapFunction. The function takes a line (String) and splits it in to
     * <p>
     * multiple pairs in the form of "(word,1)" (Tuple2&lt;String, Integer& gt;).
     */

    public static final class LineSplitter implements FlatMapFunction<String, Tuple2<String, Integer>> {

        @Override

        public void flatMap(String value, Collector<Tuple2<String, Integer>> out) { // normalize and split the line
            String[] tokens = value.toLowerCase().split("\\W+");
            // emit the pairs
            for (String token : tokens) {
                if (token.length() > 0) {
                    out.collect(new Tuple2<String, Integer>(token, 1));
                }
            }

        }
    }
}
```

<br/>

### 2.flink执行环境

程序的启动，从这句开始.

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
```

这行代码会返回一个可用的执行环境。执行环境是整个flink程序执行的上下文，记录了相关配 置（如并行度等），并提供了一系列方法，如读取输入流的方法，以及真正开始运行整个代码 的execute方法等。对于分布式流处理程序来说，我们在代码中定义的flatMap,keyBy等等操 作，事实上可以理解为一种声明，告诉整个程序我们采用了什么样的算子，而真正开启计算的代码不在此处。由于我们是在本地运行flink程序，因此这行代码会返回一个 LocalStreamEnvironment，最后我们要调用它的execute方法来开启真正的任务。我们先接着往下看。

<br/>

### 3. 算子（Operator）的注册（声明）

我们以flatMap为例, text.flatMap(new LineSplitter()) 这一句话跟进去是这样的：

```java
/**
	 * Applies a FlatMap transformation on a {@link DataStream}. The
	 * transformation calls a {@link FlatMapFunction} for each element of the
	 * DataStream. Each FlatMapFunction call can return any number of elements
	 * including none. The user can also extend {@link RichFlatMapFunction} to
	 * gain access to other features provided by the
	 * {@link org.apache.flink.api.common.functions.RichFunction} interface.
	 *
	 * @param flatMapper
	 *            The FlatMapFunction that is called for each element of the
	 *            DataStream
	 *
	 * @param <R>
	 *            output type
	 * @return The transformed {@link DataStream}.
	 *
	 * SingleOutputStreamOperator 是DataStream的子类，
	 * flink把每一个算子transform成一个对流的转换
	 * 并且注册到执行环境中，用于生成StreamGraph
	 */
public <R> SingleOutputStreamOperator<R> flatMap(FlatMapFunction<T, R> flatMapper) {

    TypeInformation<R> outType = TypeExtractor.getFlatMapReturnTypes(clean(flatMapper),
                                                                     getType(), Utils.getCallLocationName(), true);

    //真正调用的方法，
    // 第一步：用户代码里定义的UDF会被当作其基类对待，然后交给StreamFlatMap这个operator 做进一步包装。
    // 事实上，每一个Transformation都对应了一个StreamOperator。
    return transform("Flat Map", outType, new StreamFlatMap<>(clean(flatMapper)));

}
```

<br/>

里面完成了两件事，一是用反射拿到了flatMap算子的输出类型，二是生成了一个Operator。 flink流式计算的核心概念，就是将数据从输入流一个个传递给Operator进行链式处理，最后交给输出流的过程。对数据的每一次处理在逻辑上成为一个operator，并且为了本地化处理的效率起见，operator之间也可以串成一个chain一起处理（可以参考责任链模式帮助理解）。

<br/>

我们也可以更改flink的设置，要求它不要对某个操作进行chain处理，或者从某个操作开启一个新chain等。

<br/>

上面代码中的最后一行transform方法的作用是返回一个SingleOutputStreamOperator，它继承了Datastream类并且定义了一些辅助方法，方便对流的操作。在返回之前，transform方法还把它注册到了执行环境中（后面生成执行图的时候还会用到它）。其他的操作，包括 keyBy，sum和print，都只是不同的算子，在这里出现都是一样的效果，即生成一个operator 并注册给执行环境用于生成DAG。

代码如下:

```java
/**
	 * Method for passing user defined operators along with the type
	 * information that will transform the DataStream.
	 *
	 * @param operatorName
	 *            name of the operator, for logging purposes
	 * @param outTypeInfo
	 *            the output type of the operator
	 * @param operator
	 *            the object containing the transformation logic
	 * @param <R>
	 *            type of the return stream
	 * @return the data stream constructed
	 */
@PublicEvolving
public <R> SingleOutputStreamOperator<R> transform(String operatorName, TypeInformation<R> outTypeInfo, OneInputStreamOperator<T, R> operator) {

    // read the output type of the input Transform to coax out errors about MissingTypeInfo
    transformation.getOutputType();

    //第一步在617 行
    //第二步：由于flatMap这个操作只接受一个输入，所以再被进一步包装为OneInputTransformation。
    OneInputTransformation<T, R> resultTransform = new OneInputTransformation<>(
        this.transformation,
        operatorName,
        operator,
        outTypeInfo,
        environment.getParallelism());

    @SuppressWarnings({ "unchecked", "rawtypes" })
    SingleOutputStreamOperator<R> returnStream = new SingleOutputStreamOperator(environment, resultTransform);

    // 最后，将该transformation注册到执行环境中，当执行上文提到的generate方法时，生成 StreamGraph图结构。
    getExecutionEnvironment().addOperator(resultTransform);

    //SingleOutputStreamOperator 也是 DataStream的子类，也就是返回了一个新的DataStream，
    //然后调用新的DataStream的某一个算子，又生成新的 StreamTransformation，继续加入到 StreamExecutionEnvironment的 transformations
    return returnStream;
}
```

<br/>

再来看一下OneInputTransformation的父类StreamTransformation

```java
/**
 * 代表了流的转换
 * StreamTransformation代表了从一个或多个DataStream生成新DataStream的操作。
 * 顺便，DataStream类在内部组合了一个StreamTransformation类，实际的转换操作均通过该类完成。
 *
 * 从source到各种map,union再到sink操作全部被映射成了 StreamTransformation。通过看它的子类可以看到；
 *
 * 其实可以理解为就是所有算子的公共父类；
 *
 * 所有的算子都是DataStream的一个方法，每次调用一个方法(也就是调用一个算子)，DataStream这个类就通过transform这个方法，把这个算子转换成
 * StreamTransformation，然后放到  StreamExecutionEnvironment  执行环境中的 transformations list容器中，最终生成StreamGraph的时候，
 * 会根据这个容器中的操作生成；
 */
```

<br/>

### 4.程序的执行

程序执行即 env.execute("Java WordCount from SocketTextStream Example") 这行代码。

<br/>

#### 4.1 本地模式下的execute方法

这行代码主要做了以下事情：

1. 生成StreamGraph。代表程序的拓扑结构，是从用户代码直接生成的图。
2. 生成JobGraph。这个图是要交给flink去生成task的图。
3. 生成一系列配置。
4. 将 JobGraph 和配置交给 flink 集群去运行。如果不是本地运行的话，还会把jar文件通过网络发给其他节点。
5. 以本地模式运行的话，可以看到启动过程，如启动性能度量、web模块、JobManager、 ResourceManager、taskManager等等
6. 启动任务。值得一提的是在启动任务之前，先启动了一个用户类加载器，这个类加载器可以用来做一些在运行时动态加载类的工作。

<br/>

#### 4.2 远程模式（RemoteEnvironment）的execute方法

远程模式的程序执行更加有趣一点。第一步仍然是获取StreamGraph，然后调用 executeRemotely方法进行远程执行。

该方法首先创建一个用户代码加载器

```java
ClassLoader usercodeClassLoader = JobWithJars.buildUserCodeClassLoader (jarFiles, globalClasspaths, getClass().getClassLoader());
```

然后创建一系列配置，交给Client对象。Client这个词有意思，看见它就知道这里绝对是跟远 程集群打交道的客户端。

```java
				ClusterClient client;

try { client = new StandaloneClusterClient(configuration); client.setPrintStatusDuringExecution(getConfig().isSysoutLo ggingEnabled());

    }

}

try { return client.run(streamGraph, jarFiles, globalClasspaths, usercodeClassLoader).getJobExecutionResult();

    }
```

<br/>

client的run方法首先生成一个JobGraph，然后将其传递给JobClient。关于Client、 JobClient、JobManager到底谁管谁，可以看这张图：

{% asset_img 一.jpg %}

<br/>

确切的说，JobClient负责以异步的方式和JobManager通信（Actor是scala的异步模块）， 具体的通信任务由JobClientActor完成。相对应的，JobManager的通信任务也由一个Actor 完成。

<br/>

```java
JobListeningContext jobListeningContext = submitJob( actorSystem,config,highAvailabilityServices,jobGraph,ti meout,sysoutLogUpdates, classLoader);

return awaitJobResult(jobListeningContext);
```

可以看到，该方法阻塞在awaitJobResult方法上，并最终返回了一个JobListeningContext， 透过这个Context可以得到程序运行的状态和结果。

<br/>

#### 4.3 程序的启动过程

上面提到，整个程序真正意义上开始执行，是这里：

```java
env.execute("Java WordCount from SocketTextStream Example");
```

远程模式和本地模式有一点不同，我们先按本地模式来调试。 我们跟进源码，（在本地调试模式下）会启动一个miniCluster，然后开始执行代码：

```java
@Override
public JobExecutionResult execute(String jobName) throws Exception {

    // 生成各种图结构
    // transform the streaming program into a JobGraph
    .......
        try {
            //启动集群，包括启动JobMaster，进行leader选举等等
            miniCluster.start();
            configuration.setInteger(RestOptions.PORT, miniCluster.getRestAddress().getPort());

            //提交任务到JobMaster
            return miniCluster.executeJobBlocking(jobGraph);
        }
    finally {
        transformations.clear();
        miniCluster.close();
    }
}
```

<br/>

这个方法里有一部分逻辑是与生成图结构相关的，我们放在第二章里讲；现在我们先接着往里 跟：

```java
// MiniCluster.java
@Override
	public JobExecutionResult executeJobBlocking(JobGraph job) throws JobExecutionException, InterruptedException {
		checkNotNull(job, "job is null");


		//在这里，最终把job提交给了jobMaster,这段代码核心逻辑就是调用这个 submitJob 方法。
		final CompletableFuture<JobSubmissionResult> submissionFuture = submitJob(job);

		final CompletableFuture<JobResult> jobResultFuture = submissionFuture.thenCompose(
			(JobSubmissionResult ignored) -> requestJobResult(job.getJobID()));

	}
```

<br/>

正如我在注释里写的，这一段代码核心逻辑就是调用那个 submitJob 方法。那么我们再接着看 这个方法：

```java
// 核心方法
public CompletableFuture<JobSubmissionResult> submitJob(JobGraph jobGraph) {
    final DispatcherGateway dispatcherGateway;
    try {
        dispatcherGateway = getDispatcherGateway();
    } catch (LeaderRetrievalException | InterruptedException e) {
        ExceptionUtils.checkInterrupted(e);
        return FutureUtils.completedExceptionally(e);
    }

    // we have to allow queued scheduling in Flip-6 mode because we need to request slots
    // from the ResourceManager
    jobGraph.setAllowQueuedScheduling(true);

    final CompletableFuture<InetSocketAddress> blobServerAddressFuture = createBlobServerAddress(dispatcherGateway);

    final CompletableFuture<Void> jarUploadFuture = uploadAndSetJobFiles(blobServerAddressFuture, jobGraph);

    final CompletableFuture<Acknowledge> acknowledgeCompletableFuture = jarUploadFuture.thenCompose(
        //在这里执行了真正的submit操作
        /**
			 * 这里的 Dispatcher 是一个接收job，然后指派JobMaster去启动任务的类,
			 * 我们可以看看它的 类结构，有两个实现。在本地环境下启动的是 MiniDispatcher ，
			 * 在集群上提交任务时，集群 上启动的是 StandaloneDispatcher
			 */
        (Void ack) -> dispatcherGateway.submitJob(jobGraph, rpcTimeout));

    return acknowledgeCompletableFuture.thenApply(
        (Acknowledge ignored) -> new JobSubmissionResult(jobGraph.getJobID()));
}
```

<br/>

这里的 Dispatcher 是一个接收job，然后指派JobMaster去启动任务的类,我们可以看看它的 类结构，有两个实现。在本地环境下启动的是 MiniDispatcher ，在集群上提交任务时，集群 上启动的是 StandaloneDispatcher 。

<br/>

那么这个Dispatcher又做了什么呢？它启动了一个 JobManagerRunner （这里我要吐槽Flink的 命名，这个东西应该叫做JobMasterRunner才对，flink里的JobMaster和JobManager不是 一个东西），委托JobManagerRunner去启动该Job的 JobMaster 。我们看一下对应的代 码：

<br/>

```java
//jobManagerRunner.java
/**
 * The runner for the job manager. It deals with job level leader election and make underlying job manager
 * properly reacted.
 *
 * Dispatcher 启动的，其实这个东西应该叫做 JobMasterRunner 才对，flink里的JobMaster和JobManager不是同一个组件
 * 委托JobManagerRunner去启动该Job的 JobMaster
 */
public class JobManagerRunner implements LeaderContender, OnCompletionActions, AutoCloseableAsync {

    private static final Logger log = LoggerFactory.getLogger(JobManagerRunner.class);
    
    //主要是启动 JobMaster
	//最终调用到 JobMaster类的executionGraph.scheduleForExecution(); 方法，
	private final JobMaster jobMaster;
    
    private void verifyJobSchedulingStatusAndStartJobManager(UUID leaderSessionId) throws Exception {
        ....

            final CompletableFuture<Acknowledge> startFuture = jobMaster.start(new JobMasterId(leaderSessionId), rpcTimeout);
        ....
    }
```

<br/>

然后，JobMaster经过了一堆方法嵌套之后，执行到了这里：

```java
// JobMaster.java
private void scheduleExecutionGraph() {
    checkState(jobStatusListener == null);
    // register self as job status change listener
    jobStatusListener = new JobManagerJobStatusListener();
    executionGraph.registerJobStatusListener(jobStatusListener);

    try {
        //executionGraph 最后一层图结构
        executionGraph.scheduleForExecution();
    }
    catch (Throwable t) {
        executionGraph.failGlobal(t);
    }
}
```

<br/>

我们知道，flink的框架里有三层图结构，其中ExecutionGraph就是真正被执行的那一层，所 以到这里为止，一个任务从提交到真正执行的流程就走完了，我们再回顾一下（顺便提一下远 程提交时的流程区别）：

1. 客户端代码的execute方法执行；
2. 本地环境下，MiniCluster完成了大部分任务，直接把任务委派给了MiniDispatcher；
3. 远程环境下，启动了一个 RestClusterClient ，这个类会以HTTP Rest的方式把用户代码 提交到集群上；
4. 远程环境下，请求发到集群上之后，必然有个handler去处理，在这里 是 JobSubmitHandler 。这个类接手了请求后，委派StandaloneDispatcher启动job，到 这里之后，本地提交和远程提交的逻辑往后又统一了；
5. Dispatcher接手job之后，会实例化一个 JobManagerRunner ，然后用这个runner启动 job；
6. JobManagerRunner接下来把job交给了 JobMaster 去处理；
7. JobMaster使用 ExecutionGraph 的方法启动了整个执行图；整个任务就启动起来了。