---
title: Flink源码解析之三层图结构
date: 2019-06-05 18:19:48
tags:
  - Flink
---

### 1.flink的图结构

事实上，flink总共提供了三种图的抽象，我们前面已经提到了StreamGraph和JobGraph，还 有一种是ExecutionGraph，是用于调度的基本数据结构。

<br/>

{% asset_img 二.jpg %}

上面这张图清晰的给出了flink各个图的工作原理和转换过程。其中最后一个物理执行图并非 flink的数据结构，而是程序开始执行后，各个task分布在不同的节点上，所形成的物理上的关 系表示。

<br/>

- 从JobGraph的图里可以看到，数据从上一个operator流到下一个operator的过程中，上 游作为生产者提供了IntermediateDataSet，而下游作为消费者需要JobEdge。事实 上，JobEdge是一个通信管道，连接了上游生产的dataset和下游的JobVertex节点。
- 在JobGraph转换到ExecutionGraph的过程中，主要发生了以下转变：
  1. 加入了并行度的概念，成为真正可调度的图结构
  2. 生成了与JobVertex对应的ExecutionJobVertex，ExecutionVertex，与 IntermediateDataSet对应的IntermediateResult和IntermediateResultPartition等， 并行将通过这些类实现
- ExecutionGraph已经可以用于调度任务。我们可以看到，flink根据该图生成了一一对应的 Task，每个task对应一个ExecutionGraph的一个ExecutionVertex。Task用InputGate、 InputChannel和ResultPartition对应了上面图中的IntermediateResult和 ExecutionEdge。

<br/>

那么，flink抽象出这三层图结构，四层执行逻辑的意义是什么呢？

StreamGraph是对用户逻辑的映射。JobGraph在此基础上进行了一些优化，比如把一部分操 作串成chain以提高效率。ExecutionGraph是为了调度存在的，加入了并行处理的概念。而在 此基础上真正执行的是Task及其相关结构。

<br/>

### 2. StreamGraph的生成

在第一节的算子注册部分，我们可以看到，flink把每一个算子transform成一个对流的转换 （比如上文中返回的SingleOutputStreamOperator是一个DataStream的子类），并且注册 到执行环境中，用于生成StreamGraph。实际生成StreamGraph的入口 是 StreamGraphGenerator.generate(env, transformations) 其中的transformations是一 个list，里面记录的就是我们在transform方法中放进来的算子。

<br/>

#### 2.1 StreamTransformation类代表了流的转换

StreamTransformation代表了从一个或多个DataStream生成新DataStream的操作。顺便，DataStream类在内部组合了一个StreamTransformation类，实际的转换操作均通过该类完成。

{% asset_img 三.jpg %}

<br/>

我们可以看到，从source到各种map,union再到sink操作全部被映射成了 StreamTransformation。 其映射过程如下所示：

{% asset_img 四.jpg %}

<br/>

以MapFunction为例：

- 首先，用户代码里定义的UDF会被当作其基类对待，然后交给StreamMap这个operator 做进一步包装。事实上，每一个Transformation都对应了一个StreamOperator。

- 由于map这个操作只接受一个输入，所以再被进一步包装为OneInputTransformation。

- 最后，将该transformation注册到执行环境中，当执行上文提到的generate方法时，生成 StreamGraph图结构。

- 另外，并不是每一个 StreamTransformation 都会转换成runtime层中的物理操作。 有一些只是逻辑概念，比如union、split/select、partition等。如下图所示的转换 树，在运行时会优化成下方的操作图。

  {% asset_img 五.jpg %}

<br/>

#### 2.2 StreamGraph生成函数分析

我们从StreamGraphGenerator.generate()方法往下看：

```java
/**
	 * @param transformations  是一 个list，里面记录的就是我们在transform方法中放进来的算子。
	 * @return  生成StreamGraph
	 */
public static StreamGraph generate(StreamExecutionEnvironment env, List<StreamTransformation<?>> transformations) {
    return new StreamGraphGenerator(env).generateInternal(transformations);
}

/**
	 * This starts the actual transformation, beginning from the sinks.
	 */
private StreamGraph generateInternal(List<StreamTransformation<?>> transformations) {
    for (StreamTransformation<?> transformation: transformations) {
        // 处理每个算子
        transform(transformation);
    }
    return streamGraph;
}

// //注意，StreamGraph的生成是从sink开始的
private Collection<Integer> transform(StreamTransformation<?> transform) {

    if (alreadyTransformed.containsKey(transform)) {
        return alreadyTransformed.get(transform);
    }

    LOG.debug("Transforming " + transform);

    if (transform.getMaxParallelism() <= 0) {

        // if the max parallelism hasn't been set, then first use the job wide max parallelism
        // from theExecutionConfig.
        int globalMaxParallelismFromConfig = env.getConfig().getMaxParallelism();
        if (globalMaxParallelismFromConfig > 0) {
            transform.setMaxParallelism(globalMaxParallelismFromConfig);
        }
    }

    // call at least once to trigger exceptions about MissingTypeInfo
    transform.getOutputType();

    Collection<Integer> transformedIds;

    //这里对操作符的类型进行判断，并以此调用相应的处理逻辑.简而言之，处理的核心无非是递归的将该节点和节点的上游节点加入图

    //例如：map，filter等常用操作都是：OneInputStreamOperator
    if (transform instanceof OneInputTransformation<?, ?>) {
        transformedIds = transformOneInputTransform((OneInputTransformation<?, ?>) transform);
    } else if (transform instanceof TwoInputTransformation<?, ?, ?>) {
        transformedIds = transformTwoInputTransform((TwoInputTransformation<?, ?, ?>) transform);
    } else if (transform instanceof SourceTransformation<?>) {
        transformedIds = transformSource((SourceTransformation<?>) transform);
    } else if (transform instanceof SinkTransformation<?>) {
        transformedIds = transformSink((SinkTransformation<?>) transform);
    } else if (transform instanceof UnionTransformation<?>) {
        transformedIds = transformUnion((UnionTransformation<?>) transform);
    } else if (transform instanceof SplitTransformation<?>) {
        transformedIds = transformSplit((SplitTransformation<?>) transform);
    } else if (transform instanceof SelectTransformation<?>) {
        transformedIds = transformSelect((SelectTransformation<?>) transform);
    } else if (transform instanceof FeedbackTransformation<?>) {
        transformedIds = transformFeedback((FeedbackTransformation<?>) transform);
    } else if (transform instanceof CoFeedbackTransformation<?>) {
        transformedIds = transformCoFeedback((CoFeedbackTransformation<?>) transform);
    } else if (transform instanceof PartitionTransformation<?>) {
        transformedIds = transformPartition((PartitionTransformation<?>) transform);
    } else if (transform instanceof SideOutputTransformation<?>) {
        transformedIds = transformSideOutput((SideOutputTransformation<?>) transform);
    } else {
        throw new IllegalStateException("Unknown transformation: " + transform);
    }

    // need this check because the iterate transformation adds itself before
    // transforming the feedback edges
    // 注意这里和函数开始时的方法相对应，在有向图中要注意避免循环的产生
    if (!alreadyTransformed.containsKey(transform)) {
        alreadyTransformed.put(transform, transformedIds);
    }

    if (transform.getBufferTimeout() >= 0) {
        streamGraph.setBufferTimeout(transform.getId(), transform.getBufferTimeout());
    }
    if (transform.getUid() != null) {
        streamGraph.setTransformationUID(transform.getId(), transform.getUid());
    }
    if (transform.getUserProvidedNodeHash() != null) {
        streamGraph.setTransformationUserHash(transform.getId(), transform.getUserProvidedNodeHash());
    }

    if (transform.getMinResources() != null && transform.getPreferredResources() != null) {
        streamGraph.setResources(transform.getId(), transform.getMinResources(), transform.getPreferredResources());
    }

    return transformedIds;
}
```

<br/>

因为map，filter等常用操作都是OneInputStreamOperator,我们就来看 看 transformOneInputTransform((OneInputTransformation<?, ?>) transform) 方法。

```java
// 单个输入类型：filter、map等算子都属于此类型；
private <IN, OUT> Collection<Integer> transformOneInputTransform(OneInputTransformation<IN, OUT> transform) {

    // 核心：递归调用：先处理这个算子的input，所以说StreamGraph的生成是从sink开始的
    Collection<Integer> inputIds = transform(transform.getInput());

    // the recursive call might have already transformed this
    if (alreadyTransformed.containsKey(transform)) {
        return alreadyTransformed.get(transform);
    }

    //这里是获取slotSharingGroup。这个group用来定义当前我们在处理的这个操作符可以跟什么操作符chain到一个slot里进行操作
    // 因为有时候我们可能不满意flink替我们做的chain聚合
    // 一个slot就是一个执行task的基本容器
    String slotSharingGroup = determineSlotSharingGroup(transform.getSlotSharingGroup(), inputIds);

    //把该operator加入图
    streamGraph.addOperator(transform.getId(),
                            slotSharingGroup,
                            transform.getCoLocationGroupKey(),
                            transform.getOperator(),
                            transform.getInputType(),
                            transform.getOutputType(),
                            transform.getName());

    //对于keyedStream，我们还要记录它的keySelector方法
    //flink并不真正为每个keyedStream保存一个key，而是每次需要用到key的时候都使用keySelector方法进行计算
    //因此，我们自定义的keySelector方法需要保证幂等性
    //到后面介绍keyGroup的时候我们还会再次提到这一点
    if (transform.getStateKeySelector() != null) {
        TypeSerializer<?> keySerializer = transform.getStateKeyType().createSerializer(env.getConfig());
        streamGraph.setOneInputStateKey(transform.getId(), transform.getStateKeySelector(), keySerializer);
    }

    streamGraph.setParallelism(transform.getId(), transform.getParallelism());
    streamGraph.setMaxParallelism(transform.getId(), transform.getMaxParallelism());

    //为当前节点和它的依赖节点建立边
    //这里可以看到之前提到的select union partition等逻辑节点被合并入edge 的过程
    for (Integer inputId: inputIds) {
        streamGraph.addEdge(inputId, transform.getId(), 0);
    }

    return Collections.singleton(transform.getId());
}


// StreamGraph.java
public void addEdge(Integer upStreamVertexID, Integer downStreamVertexID, int typeNumber) {
    addEdgeInternal(upStreamVertexID,
                    downStreamVertexID,
                    typeNumber,
                    null,
                    new ArrayList<String>(),
                    null);

}

//addEdge的实现，会合并一些逻辑节点
private void addEdgeInternal(Integer upStreamVertexID,
                             Integer downStreamVertexID,
                             int typeNumber,
                             StreamPartitioner<?> partitioner,
                             List<String> outputNames,
                             OutputTag outputTag) {

    //如果输入边是侧输出节点，则把side的输入边作为本节点的输入边，并递归调用
    if (virtualSideOutputNodes.containsKey(upStreamVertexID)) {
        int virtualId = upStreamVertexID;
        upStreamVertexID = virtualSideOutputNodes.get(virtualId).f0;
        if (outputTag == null) {
            outputTag = virtualSideOutputNodes.get(virtualId).f1;
        }
        addEdgeInternal(upStreamVertexID, downStreamVertexID, typeNumber, partitioner, null, outputTag);

        //如果输入边是select，则把select的输入边作为本节点的输入边
    } else if (virtualSelectNodes.containsKey(upStreamVertexID)) {
        int virtualId = upStreamVertexID;
        upStreamVertexID = virtualSelectNodes.get(virtualId).f0;
        if (outputNames.isEmpty()) {
            // selections that happen downstream override earlier selections
            outputNames = virtualSelectNodes.get(virtualId).f1;
        }
        addEdgeInternal(upStreamVertexID, downStreamVertexID, typeNumber, partitioner, outputNames, outputTag);

        //如果是partition节点
    } else if (virtualPartitionNodes.containsKey(upStreamVertexID)) {
        int virtualId = upStreamVertexID;
        upStreamVertexID = virtualPartitionNodes.get(virtualId).f0;
        if (partitioner == null) {
            partitioner = virtualPartitionNodes.get(virtualId).f1;
        }
        addEdgeInternal(upStreamVertexID, downStreamVertexID, typeNumber, partitioner, outputNames, outputTag);
    } else {

        // //正常的edge处理逻辑
        StreamNode upstreamNode = getStreamNode(upStreamVertexID);
        StreamNode downstreamNode = getStreamNode(downStreamVertexID);

        // If no partitioner was specified and the parallelism of upstream and downstream
        // operator matches use forward partitioning, use rebalance otherwise.
        if (partitioner == null && upstreamNode.getParallelism() == downstreamNode.getParallelism()) {
            partitioner = new ForwardPartitioner<Object>();
        } else if (partitioner == null) {
            partitioner = new RebalancePartitioner<Object>();
        }

        if (partitioner instanceof ForwardPartitioner) {
            if (upstreamNode.getParallelism() != downstreamNode.getParallelism()) {
                throw new UnsupportedOperationException("Forward partitioning does not allow " +
                                                        "change of parallelism. Upstream operation: " + upstreamNode + " parallelism: " + upstreamNode.getParallelism() +
                                                        ", downstream operation: " + downstreamNode + " parallelism: " + downstreamNode.getParallelism() +
                                                        " You must use another partitioning strategy, such as broadcast, rebalance, shuffle or global.");
            }
        }

        StreamEdge edge = new StreamEdge(upstreamNode, downstreamNode, typeNumber, outputNames, partitioner, outputTag);

        getStreamNode(edge.getSourceId()).addOutEdge(edge);
        getStreamNode(edge.getTargetId()).addInEdge(edge);
    }
}
```

<br/>

### 3. JobGraph 的生成

flink会根据上一步生成的StreamGraph生成JobGraph，然后将JobGraph发送到server端进 行ExecutionGraph的解析。

<br/>

#### 3.1 JobGraph生成源码

与StreamGraph类似，JobGraph的入口方法是 StreamingJobGraphGenerator.createJobGraph() 。我们直接来看源码。

```java
private JobGraph createJobGraph() {

    // make sure that all vertices start immediately
    // 设置启动模式为所有节点均在一开始就启动
    jobGraph.setScheduleMode(ScheduleMode.EAGER);

    // Generate deterministic hashes for the nodes in order to identify them across
    // 为每个节点生成hash id
    Map<Integer, byte[]> hashes = defaultStreamGraphHasher.traverseStreamGraphAndGenerateHashes(streamGraph);

    // Generate legacy version hashes for backwards compatibility
    // 为了保持兼容性创建的hash
    List<Map<Integer, byte[]>> legacyHashes = new ArrayList<>(legacyStreamGraphHashers.size());
    for (StreamGraphHasher hasher : legacyStreamGraphHashers) {
        legacyHashes.add(hasher.traverseStreamGraphAndGenerateHashes(streamGraph));
    }

    Map<Integer, List<Tuple2<byte[], byte[]>>> chainedOperatorHashes = new HashMap<>();

    //生成jobvertex，串成chain等
    //这里的逻辑大致可以理解为，挨个遍历节点，如果该节点是一个chain的头节点，
    // 就生成一个JobVertex，如果不是头节点，就要把自身配置并入头节点，然后把头节点和自己的出边相连；
    // 对于不能chain的节点，当作只有头节点处理即可
    setChaining(hashes, legacyHashes, chainedOperatorHashes);

    // 设置输入边edge
    setPhysicalEdges();

    //设置slot共享group
    setSlotSharingAndCoLocation();

    //配置检查点
    configureCheckpointing();

    JobGraphGenerator.addUserArtifactEntries(streamGraph.getEnvironment().getCachedFiles(), jobGraph);

    // set the ExecutionConfig last when it has been finalized
    try {
        // 传递执行环境配置
        jobGraph.setExecutionConfig(streamGraph.getExecutionConfig());
    }
    catch (IOException e) {
        throw new IllegalConfigurationException("Could not serialize the ExecutionConfig." +
                                                "This indicates that non-serializable types (like custom serializers) were registered");
    }

    return jobGraph;
}
```

<br/>

#### 3.2 operator chain的逻辑

**为了更高效地分布式执行，Flink会尽可能地将operator的subtask链接（chain）在一起 形成task。每个task在一个线程中执行。将operators链接成task是非常有效的优化：它 能减少线程之间的切换，减少消息的序列化/反序列化，减少数据在缓冲区的交换，减少了延迟的同时提高整体的吞吐量**

{% asset_img 六.jpg %}

<br/>

上图中将KeyAggregation和Sink两个operator进行了合并，因为这两个合并后并不会改变整 体的拓扑结构。但是，并不是任意两个 operator 就能 chain 一起的,其条件还是很苛刻的：

- 上下游的并行度一致
- 下游节点的入度为1 （也就是说下游节点没有来自其他节点的输入）
- 上下游节点都在同一个 slot group 中（下面会解释 slot group）
- 下游节点的 chain 策略为 ALWAYS（可以与上下游链接，map、flatmap、filter等 默认是ALWAYS）
- 上游节点的 chain 策略为 ALWAYS 或 HEAD（只能与下游链接，不能与上游链 接，Source默认是HEAD）
- 两个节点间数据分区方式是 forward（参考理解数据流的分区）
- 用户没有禁用 chain

<br/>

flink的chain逻辑是一种很常见的设计，比如spring的interceptor也是类似的实现方式。通过把操作符串成一个大操作符，flink避免了把数据序列化后通过网络发送给其他节点的开销，能够大大增强效率。

<br/>

### 3.3 JobGraph的提交

前面已经提到，JobGraph的提交依赖于JobClient和JobManager之间的异步通信，如图所示：

{% asset_img 七.jpg %}

在submitJobAndWait方法中，其首先会创建一个JobClientActor的ActorRef,然后向其发起 一个SubmitJobAndWait消息，该消息将JobGraph的实例提交给JobClientActor。发起模式 是ask，它表示需要一个应答消息。

```java
//JobClient.java
Future<Object> future = Patterns.ask(jobClientActor, new JobClientMessa ges.SubmitJobAndWait(jobGraph), new Timeout(AkkaUtils.INF_TIMEOUT())); answer = Await.result(future, AkkaUtils.INF_TIMEOUT());
```

该SubmitJobAndWait消息被JobClientActor接收后，最终通过调用tryToSubmitJob方法触 发真正的提交动作。当JobManager的actor接收到来自client端的请求后，会执行一个 submitJob方法，主要做以下事情：

- 向BlobLibraryCacheManager注册该Job
- 构建ExecutionGraph对象；
- 对JobGraph中的每个顶点进行初始化
- 将DAG拓扑中从source开始排序，排序后的顶点集合附加到Exec> - utionGraph对象；
- 获取检查点相关的配置，并将其设置到ExecutionGraph对象；
- 向ExecutionGraph注册相关的listener；
- 执行恢复操作或者将JobGraph信息写入SubmittedJobGraphStore以在后续用于恢 复目的；
- 响应给客户端JobSubmitSuccess消息；
- 对ExecutionGraph对象进行调度执行；

最后，JobManger会返回消息给JobClient，通知该任务是否提交成功。

<br/>

### 4.ExecutionGraph的生成

与StreamGraph和JobGraph不同，ExecutionGraph并不是在我们的客户端程序生成，而是 在服务端（JobManager处）生成的，顺便flink只维护一个JobManager。其入口代码 是 ExecutionGraphBuilder.buildGraph（...） 该方法长200多行，其中一大半是checkpoiont的相关逻辑，我们暂且略过，直接看核心方 法 executionGraph.attachJobGraph(sortedTopology) 因为ExecutionGraph事实上只是改动了JobGraph的每个节点，而没有对整个拓扑结构进行变 动，所以代码里只是挨个遍历jobVertex并进行处理：

```java
public void attachJobGraph(List<JobVertex> topologiallySorted) throws JobException {

		LOG.debug("Attaching {} topologically sorted vertices to existing job graph with {} " +
				"vertices and {} intermediate results.",
				topologiallySorted.size(), tasks.size(), intermediateResults.size());

		final ArrayList<ExecutionJobVertex> newExecJobVertices = new ArrayList<>(topologiallySorted.size());
		final long createTimestamp = System.currentTimeMillis();

		for (JobVertex jobVertex : topologiallySorted) {

			if (jobVertex.isInputVertex() && !jobVertex.isStoppable()) {
				this.isStoppable = false;
			}

			//在这里生成ExecutionGraph的每个节点
			//首先是进行了一堆赋值，将任务信息交给要生成的图节点，以及设定并行度等等
			//然后是创建本节点的IntermediateResult，根据本节点的下游节点的个数确定创建几份
			//最后是根据设定好的并行度创建用于执行task的ExecutionVertex
			//如果job有设定inputsplit的话，这里还要指定inputsplits
			ExecutionJobVertex ejv = new ExecutionJobVertex(
				this,
				jobVertex,
				1,
				rpcTimeout,
				globalModVersion,
				createTimestamp);

			//这里要处理所有的JobEdge
			//对每个edge，获取对应的intermediateResult，并记录到本节点的输入上
			//最后，把每个ExecutorVertex和对应的IntermediateResult关联起来
			ejv.connectToPredecessors(this.intermediateResults);

			ExecutionJobVertex previousTask = this.tasks.putIfAbsent(jobVertex.getID(), ejv);
			if (previousTask != null) {
				throw new JobException(String.format("Encountered two job vertices with ID %s : previous=[%s] / new=[%s]",
						jobVertex.getID(), ejv, previousTask));
			}

			for (IntermediateResult res : ejv.getProducedDataSets()) {
				IntermediateResult previousDataSet = this.intermediateResults.putIfAbsent(res.getId(), res);
				if (previousDataSet != null) {
					throw new JobException(String.format("Encountered two intermediate data set with ID %s : previous=[%s] / new=[%s]",
							res.getId(), res, previousDataSet));
				}
			}

			this.verticesInCreationOrder.add(ejv);
			this.numVerticesTotal += ejv.getParallelism();
			newExecJobVertices.add(ejv);
		}

		terminationFuture = new CompletableFuture<>();
		failoverStrategy.notifyNewVertices(newExecJobVertices);
	}
```

至此，ExecutorGraph就创建完成了。