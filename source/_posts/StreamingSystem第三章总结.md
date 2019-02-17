---
title: StreamingSystem第三章总结
categories:
  - StreamingSystem读书笔记
abbrlink: 98dde65a
date: 2019-02-15 12:32:11
---

从流处理系统的底层机制的角度来看，查看这些机制将有助于我们激发，理解和应用水印的概念。我们将讨论**如何在数据入口处创建水印**，**它们如何在数据处理管道中传播**，以及**它们如何影响输出时间戳**。我们还演示了水印提供了什么样必要的保证，以便在处理无界数据时回答where和when的问题

<br/>

### 定义

何时可以安全地调用事件时间窗口，这意味着窗口不再需要更多数据。为此，我们想要描述管道相对于其无界输入所取得的进展。

第一章尝试将事件时间映射为处理时间，但是这种要保证数据的瞬时性，即事件时间等于处理时间，现实中不可能实现

<br/>

另一种直观但最终不正确的方法是考虑数据pipline处理的消息速率。虽然这是一个有趣的指标，但是随着输入的变化，预期结果的可变性，可用于处理的资源等，速率可能会随意变化

<br/>

我们需要更有力的衡量数据处理进展的工具。为此，我们对流数据做出一个基本假设：每条消息都有一个相关的逻辑事件时间戳。在连续到达无界数据的情况下，这种假设是合理的，因为这意味着连续生成输入数据。在大多数情况下，我们可以将原始事件发生的时间作为其逻辑事件时间戳。使用包含事件时间戳的所有输入消息，我们可以检查任何数据管道中的此类时间戳的分布。这样的管道可以被分发以在许多代理上并行处理并消耗输入消息而不保证各个分片之间的排序。因此，此管道中正在处理的消息的事件时间戳将形成分布，如图3-1所示。

消息由管道摄取，处理并最终标记为已完成。每条消息都是“在飞行中”，意味着它已被接收但尚未完成，或“已完成”，这意味着此消息已经处理完成。如果我们按事件时间检查消息的分布，它将如图3-1所示。随着时间的推移，更多的消息将被添加到右侧的“飞行中”分布中，并且来自分布的“飞行中”部分的更多消息将被完成并移动到“完成”分布中

{% asset_img figure3-1.png %}

<br/>

此分布上有一个关键点，位于“飞行中”分布的最左边，对应于我们管道的任何未处理消息的最早事件时间戳。我们使用此值来定义水印

水印是尚未完成的最早的单调增加的时间戳

两个有用的基本属性

1. 完整性：如果水印超过某个时间戳T，我们可以通过其单调属性来保证在T处或T处之后不会再对之后的事件数据进行处理。因此，我们可以在T处或之后正确地发出任何聚合。换句话说，水印允许我们知道何时关闭窗口是正确的。
2. 可见性：如果由于任何原因消息在我们的管道中卡住，则水印无法前进。此外，我们将通过检查阻止水印前进的消息来找到问题的根源。问题对我们来说，是可见的。

<br/>

<br/>

### 数据源处watermark 的创建

这些水印来自哪里？要为数据源建立水印，我们必须为从该源进入管道的每条消息分配逻辑事件时间戳（事件时间）

<br/>

继续看第二章中计算团队得分然后求和的例子，左图是完美式水印，右图是启发式水印，区别特征是完美的水印确保水印能够解释所有数据，而启发式水印则允许产生一些延迟元素。

{% asset_img figure3-2.png %}

<br/>

在水印一旦创建完成后，水印在整个管道中的状态将保持不变。至于到底是创建完美式水印还是启发式水印，它在很大程度上取决于数据源的性质。为了了解原因，让我们看一下每种水印创建的几个例子

<br/>

#### 完美水印的创建

完美的水印创建将时间戳分配给传入的消息，使得产生的水印严格保证不会再从该源再次看到事件时间小于水印的数据。使用完美水印创建的管道永远不必处理延迟数据；也就是说，在水印之后到达的数据永远比之前的数据事件时间要大。然而，完美的水印创建需要对数据源有完全的了解，因此对于许多真实的分布式输入源是不切实际的。以下是一些可以创建完美水印的用例示例

1. 将进入数据处理系统的时间指定为数据的事件时间，这样的源可以创建完美的水印。在这种情况下，源水印只是跟踪管道所观察到的当前处理时间。这基本上是2016年之前所有窗口化的流媒体系统所使用的方法，因为事件时间是从一个单调增加的源（实际进入系统的时间）分配的，所以系统因此完全了解数据流中接下来会有哪些时间戳。结果，事件时间进度和窗口语义变得非常容易推理。当然，缺点是水印与数据本身的事件时间无关，数据本身的产生时间（事件时间）被全部丢弃，而水印只是跟踪数据相对于其到达系统的进度；
2. **按照时间排好序的静态日志集**，（例如，具有静态分区（分区数不变）的Apache Kafka主题，其中源的每个分区包含单调增加的事件时间）将是创建完美水印的相对简单的源。为此，源将简单地跟踪已知所有topic上的未处理数据的最小事件时间（即，取每个分区中最近的记录的事件时间中的最小值）；与前面提到的入口时间戳类似，由于已知跨越静态分区集的事件时间单调增加，因此系统完全了解下一个时间戳将会到来。这实际上是一种有界无序处理的形式，已知所有topic中的无序数据受从所有topic中观察到的最小事件时间的限制；通常，您可以保证在分区内单调增加时间戳的唯一方法是，在将数据写入其中时分配这些分区中的时间戳，例如，通过web前端将事件直接记录到Kafka中。虽然仍然是有限的做法，但这肯定比到达数据处理系统时的入口时间戳更有用，因为水印跟踪的是数据的有意义的事件时间。

<br/>

#### 启发式水印的创建

另一方面，启发式水印创建产生的水印仅仅是对不会再次看到事件时间小于当前水印的估计。使用启发式水印创建的管道可能需要处理一些延迟数据。延迟数据是在水印代表的事件时间之后到达的任何数据。只有启发式水印创建才有可能产生延迟数据。如果启发式算法相当好，则后期数据的数量可能非常小，并且水印仍然可用作完成估计的度量，如果要支持需要完全正确性的场景（例如，诸如计费之类的事情），系统仍然需要为用户提供处理延迟数据的方法。

<br/>

对于许多现实世界的分布式输入源，构建完美水印在计算上或操作上都是不切实际的，但仍然可以通过利用输入数据源的结构特征来构建高度准确的启发式水印。给出以下两个例子：

<br/>

1. 动态的（动态的意思可以理解为在kafka中topic的个数是不固定的，随时可能增加）按时间顺序排列的日志，考虑一组动态结构化日志文件（每个单独的文件包含相对于同一文件中的其他记录单调增加的事件时间但没有文件之间事件时间固定关系的记录），其中包含完整的预期日志文件集（即，在Kafka的说法中，分区在运行时是未知的。这些投入通常存在于由许多独立团队构建和管理的全球规模服务中。在这种用例中，在输入上创建完美的水印是难以处理的，但是创建精确的启发式水印是非常有可能的。通过跟踪现有日志文件集中未处理数据的最小事件时间，监控增长率，并利用网络拓扑和带宽可用性等外部信息，您可以创建非常准确的水印，即使缺乏对所有日期文件的全部了解。这种类型的输入源是Google中最常见的无界数据集类型之一，因此我们在为这些场景创建和分析水印质量方面拥有丰富的经验，并且已经看到它们在许多用例中都具有良好的效果。
2. google 云服务数据，略

<br/>

考虑用户玩移动游戏的示例，并将他们的分数发送到我们的数据pipline进行处理，您通常可以假设对于使用移动设备进行输入的任何数据源，通常不可能提供完美的水印。由于设备长时间脱机的问题，没有办法对这种数据源提供任何合理的绝对完整性估计。但是，您可以设想构建一个水印，准确估计当前在线设备的数据输入完整性，从提供低延迟结果的角度来看，积极在线的用户可能是我们最关心的用户子集，因此这种情况不像您想象的那么糟糕

<br/>

从广义上讲，对数据源的了解越多，启发式越好，延迟数据就越少。鉴于源的类型，事件的分布和使用模式会有很大差异，因此没有一个通用的解决方案。但在任何一种情况下（完美或启发式），在输入源处创建水印之后，系统可以完美地通过数据pipline 传播水印。这意味着完美的水印将保持完美的下游，启发式水印也将严格保持与启动时一样的启发式。这是watermark的好处：

**您可以将管道中完整性跟踪的复杂性完全降低到在源头创建水印的问题**

<br/>

到目前为止，我们只考虑了单个操作或阶段环境中输入的水印。但是，大多数真实世界的管道都包含多个阶段。了解水印如何在各个阶段传播对于理解它们如何影响整个管道以及如何观察结果延迟非常重要。

<br/>

<br/>

### watermark 的传播

生产任务的pipeline中通常有多个stage，在源头产生的watermark会在pipeline的多个stage间传递。了解watermark如何在一个pipeline的多个stage间进行传递，可以更好的了解watermark对整个pipeline的影响，以及对pipeline结果延时的影响。我们在pipeline的各stage的边界上对watermark做如下定义：

 <br/>

<br/>

1. 输入watermark（An input watermark）：捕捉上游各阶段数据处理进度。对源头算子，input watermark是个特殊的function，对进入的数据产生watermark。对非源头算子，input watermark是上游stage中，所有shard/partition/instance产生的最小的watermark
2. 输出watermark（An output watermark）：捕捉本stage的数据进度，实质上指本stage中，所有input watermark的最小值，和本stage中所有非late event的数据的event time。比如，该stage中，被缓存起来等待做聚合的数据等。

 <br/>

**通过每个stage的input watermark和output watermark可以计算出该stage在event time上产生的延时（event time lantency = output watermark-input watermark）.比如，一个10s的聚合窗口，会产生一个>=10s的延时。管道中，每个stage根据具体的操作会对event time做相应延时。**

 <br/>

每个stage内的操作并不是线性递增的。概念上，每个stage的操作都可以被分为几个组件（components），每个组件都会影响pipeline的输出watermark。每个组件的特性与具体的实现方式和包含的算子相关。理论上，这类算子会缓存数据，直到触发某个计算。比如缓存一部分数据并将其存入状态（state）中，直到触发聚合计算，并将计算结果写入下游stage。

{% asset_img figure3-3.png %}

<br/>

上图为流计算系统中包含待处理数据缓存功能组件的stage示意图。每个component都有watermark。而整个stage的watermark就是所有buffer的watermark的最小值。在该例子中，watermark可以是以下项的最小值：

- 每个source的watermark（Per-source watermark） - 每个发送数据的stage.
- 每个外部数据源的watermark（Per-external input watermark） - pipeline之外的数据源
- 每个状态组件的watermark（Per-state component watermark） - 每种需要写入的state类型
- 每个输出buffer的watermark（Per-output buffer watermark） - 每个接收stage

<br/>

这种精度的watermark能够更好的描述系统内部状态。能够更简单的跟踪数据在系统各个buffer中的流转状态，有助于排查数据堵塞问题。

 <br/>

#### 理解watermakr的传播

我们用一个例子来更好理解输入和输入watermark的关系以及他们如何影响watermark在计算pipeline中的传播。例子：通过计算用户玩游戏的时间来评估用户对游戏的参与度。假设游戏有pc端和手机端这两个数据源。就需要写两个pipeline来分别对每个数据源计算用户游戏得分。虽然计算逻辑相同，但由于数据源不同，会导致watermark非常不同。 计算用户游戏session时长的伪代码如下（session window）：

```
PCollection<Double> mobileSessions = IO.read(new MobileInputSource())
.apply(Window.into(Sessions.withGapDuration(Duration.standardMinutes(1))
               .triggering(AtWatermark())
               .discardingFiredPanes())
  .apply(CalculateWindowLength());
PCollection<Double> consoleSessions = IO.read(new ConsoleInputSource())
.apply(Window.into(Sessions.withGapDuration(Duration.standardMinutes(1))
               .triggering(AtWatermark())
               .discardingFiredPanes())
  .apply(CalculateWindowLength());
```

<br/>

该计算中，先按user进行分区，再对每个user开个中断时长为1min的session window，然后计算这个session的长度，即得到用户玩游戏的时长。两个pipeline的输出如下：

 <br/>

{% asset_img figure3-4.png %}

<br/>

上图表示每个用户的session时长。

<br/>

接着，我们求一下游戏用户的平均游戏时长，首先每个用户的session时长，再将两个流的数据合并成一个流，求固定窗口内的session时长，求手机端和PC端用户时长的伪代码如下：

```
PCollection<Double> mobileSessions = IO.read(new MobileInputSource())
.apply(Window.into(Sessions.withGapDuration(Duration.standardMinutes(1))
               .triggering(AtWatermark())
               .discardingFiredPanes())
  .apply(CalculateWindowLength());
PCollection<Double> consoleSessions = IO.read(new ConsoleInputSource())
.apply(Window.into(Sessions.withGapDuration(Duration.standardMinutes(1))
               .triggering(AtWatermark())
               .discardingFiredPanes())
  .apply(CalculateWindowLength());
PCollection<Float> averageSessionLengths = PCollectionList
  .of(mobileSessions).and(consoleSessions)
  .apply(Flatten.pCollections())
  .apply(Window.into(FixedWindows.of(Duration.standardMinutes(2)))
               .triggering(AtWatermark())
  .apply(Mean.globally());
```

<br/>

<br/>

图示如下

{% asset_img figure3-5.png %}

<br/>

两个重要的点：

- 输出watermark一定比输入watermark早。
- 这个例子中，求平均时长的stage的watermark是两个输入中watermark较早的那个。

<br/> <br/>

pipeline中，下游各stage的watermark一定比上游小（早）。这个例子中，pipeline的逻辑修改起来非常简单，通过研究这个pipeline，我们再看研究watermark的另一个问题：输出时间戳（output timestamp）

 <br/>

#### watermark传播和输出时间戳

 在图3-5中，隐藏了输出时间戳的一些细节。但是如果仔细观察图中的第二阶段，可以看到第一阶段的每个输出都被分配了一个与其窗口末端相匹配的时间戳。虽然这是输出时间戳的一个相当自然的选择，但它不是唯一有效的选择。正如您在本章前面所知，水印永远不会被允许向后移动（单调递增，永远不可能减小）。鉴于该限制，您可以推断出给定窗口的有效输出时间戳范围以窗口中最早的非延迟数据的时间戳开始（因为只有非迟到数据才保证比水印要小）并一直延伸到正无穷大。这有很多选择。然而，在实践中，在大多数情况下，往往只有少数选择是有意义的：

- 窗口结束时间：将窗口结束时间作为这个窗口输出数据的时间戳。这种方式系统的性能最好，因为这种方式不会阻塞输出watermark
- 窗口中第一个非迟到数据的时间戳：用窗口中第一个非late event的数据的时间戳，作为窗口所有输出数据的时间戳是一种非常保守的方式，但是会对系统性能有一定影响，因为这种方式会阻塞 output watermark
- 用窗口中某个元素的时间戳：在某种用例中，比如一个查询流join一个点击流，有时希望用查询流的时间戳做watermark，有时又希望用点击流的时间戳做watermark。 

<br/>

接下来我们用个例子来说明输出时间戳在整个pipeline中的作用，用窗口中第一个非late event数据的时间戳作为窗口输出数据的时间戳的伪代码如下：

```
PCollection<Double> mobileSessions = IO.read(new MobileInputSource())
.apply(Window.into(Sessions.withGapDuration(Duration.standardMinutes(1))
               .triggering(AtWatermark())
               .withTimestampCombiner(EARLIEST)
               .discardingFiredPanes())
  .apply(CalculateWindowLength());
PCollection<Double> consoleSessions = IO.read(new ConsoleInputSource())
.apply(Window.into(Sessions.withGapDuration(Duration.standardMinutes(1))
               .triggering(AtWatermark())
               .withTimestampCombiner(EARLIEST)
               .discardingFiredPanes())
  .apply(CalculateWindowLength());
PCollection<Float> averageSessionLengths = PCollectionList
  .of(mobileSessions).and(consoleSessions)
  .apply(Flatten.pCollections())
  .apply(Window.into(FixedWindows.of(Duration.standardMinutes(2)))
               .triggering(AtWatermark())
  .apply(Mean.globally());
```

<br/>

{% asset_img figure3-6.png %}

这个图中，与上个图（ figure3-5.png）相比，watermark被hold住了，因为上个图片中的时间戳选的是窗口结束，但是这个图片中，选第一个非late event的数据的时间戳。同时可以明显看出，第二个stage的watermark也被hold住了。 

<br/>

两点值得注意：

- 水印延迟：与图3-5相比，图3-6中的水印进展得慢得多。这是因为第一阶段的输出水印一直被保持为每个窗口中第一个非延迟事件的事件时间戳，（也就是我们看到的第二幅图的1.6内条笔直的线，代表watermark一直没有前进，即处理时间在增长，但是处理的事件时间一直被阻塞）直到该窗口的输入完成，才允许输出水印前进（1.6之后水印开始水平，代表watermark 开始快递处理）
- 语义差别：因为输出时间戳现在被分配为匹配会话中最早的nonlate元素的事件时间，所以此时当我们计算下一阶段的全局会话长度平均值时，每个session window 的输出时间戳通常会出现在不同的fix window桶中，这也是导致最终计算结果不一致的原因。到目前为止我们看到的两个选项中没有一个本质上是对或错的，他们只是两种不同的角度。但重要的是要明白它们的意义是不用的，并且熟悉不同的选择会带来不同的结果，以便在时机成熟时为您的特定用例做出正确的选择。

<br/>

#### 窗口重叠的复杂case

 如何处理重叠窗口的输出时间戳非常重要和复杂。如果将输出数据时间戳设为窗口中最早非late event数据的时间戳，会导致下游窗口有很大延时。假设一个有两个stage的pipeline，一个元素通过三个连续的滑窗。理想的语义是：

- 第一个窗口在第一个stage结束，并输出到下游
- 第一个窗口在第二个stage结束，输出到下游
- 一段时间之后，第二个窗口在第一个stage结束，然后循环往复。。。

但真正的语义是：

- 第一个窗口在第一个stage结束，并输出到下游
- 第一个窗口在第二个stage无法输出，因为其输入watermark被第二个和第三个窗口拖住了，因为用了窗口中最早数据的时间戳作为窗口输出时间戳
- 第二个窗口在第一个stage结束，并输出到下游
- 第一个窗口和第二个窗口在第二个stage仍然不能输出，被上游第三个窗口拖住
- 第三个窗口在第一个stage结束，并输出到下游
- 第一/二/三个窗口在第二个stage同时输出

这种方式虽然结果是对的，但会大大增加不必要的延时。

<br/>

<br/>

### 百分比watermark

我们可以分析event time的分布来得到窗口更好的触发时间。也就是说，在某个时间点，我已经处理了某个event time之前百分之多少的数据（这是依据event time的分布来保证的）

 <br/>

这个方案有什么好处？如果对于业务逻辑“大多数”正确就足够了，百分位水印提供了一种机制，通过该机制，水印可以通过从水印中丢弃分布的长尾中的异常值的方式，来跟踪最小事件时间戳，以使得watermark可以更快更平稳地前进。

<br/>

图3-9显示了事件时间的紧凑分布，其中第90百分位水印接近第100百分位数。图3-10显示了异常值进一步落后的情况，因此第90百分位水印显著领先于第100百分位水印。通过从水印中丢弃异常数据，百分位水印仍然可以跟踪分布的大部分消息而不会被异常值延迟。

<br/>

{% asset_img figure3-9.png %}

<br/>

{% asset_img figure3-10.png %}

<br/>

<br/>

看个例子，在2分钟固定窗口中，可以根据输入数据百分比画几条线：

 {% asset_img figure3-11.png %}

<br/>

该图展示了33%，66%和100% watermark。33%和66%的watermark使窗口更早输出，当然后果是更多的数据成了late event。比如[12:00,12:02)的窗口中，33%的watermark，只有4个元素会被计算，其他元素都是late event，但是窗口在处理时间的12:06就会输出。66%的watermark有7个元素，在12:07输出，而100% watermark有10个元素，窗口要在12:08才会输出。百分比watermark为我们提供了一种平衡输出延时和结果正确性的方式。

<br/> <br/>

### 处理时间 watermark

event time上的watermark可以帮助我们确定最早未处理的数据与当前的整体延迟，但不能解决这个延时到底是由于系统问题还是数据问题导致的。由此我们引入process time上的watermark。 

<br/>

process time上的watermark的定义与event time watermark定义完全相同，使用最早的还未完成的算子的时间作为process time上的watermark，其**与当前时间的延时**，有几种原因：从一个stage到另一个stage的数据发送被堵住了，访问state的IO被堵住了，处理过程中有异常。总结来说，就是系统有问题了；

<br/>

 {% asset_img figure3-12.png %}

<br/>

上图表示了一个pipeline中，event time上的延时一直在增加，但是并不知道为啥增加。

 <br/>

 {% asset_img figure3-13.png %}

结合event time和process time的延时，我们可以推断出由于某种原因导致系统无法及时处理数据。比如网络问题或者系统异常等等。process time上的延时就表明了由于某些系统异常导致了数据无法被及时处理，需要引起管理员的注意。

<br/>

 {% asset_img figure3-14.png %}

 这个例子中，process time延时正常，但是event time上延时很高，说明系统是正常的，可能是由于系统中由于一些原因缓存了一些数据才导致了输出延时增长。

 <br/>

{% asset_img figure3-15.png %}

上图表示固定窗口的event time和process time的延时曲线。process time上延时一直是正常的，event time上延时会随着窗口缓存数据而增加，一旦窗口被触发输出，event time上的延时又变小。

因此process time上的watermark是区分数据延时或系统延时的非常好的工具。同时，在系统实现层面，process time上的watermark通常在系统垃圾回收会临时状态清理过程中使用。

<br/>

<br/>

### 案例解析

 上文详述了watermark的理论知识，接下来我们看看现在主流的流计算系统是如何在正确性和延时之间做权衡，来实现这些理论的。

<br/> <br/>

#### Google Cloud Dataflow 中的 Watermark

 Dataflow每个stage，都将输入的数据按照key的范围分片，每个物理worker只负责一个key分区。当pipeline中又GroupByKey操作时，数据必须按照key被分发到相应的worker上进行计算，其逻辑示意图如下：

 {% asset_img figure3-16.png %}

<br/>

上图表明，在每个stage，数据都会被按key进行shuffle。其数据物理分区示意图如下：

  {% asset_img figure3-17.png %}

<br/>

每个worker中都负责两个stage的计算，每个stage计算完成后，数据都需要按key被分发到相应的worker进行下一阶段的计算。

Dataflow记录每个stage中每个组件的的每个key分区的watermark，watermark的聚合操作，会分析所有key分区的watermark的最小值，保证：

- 所有分区都有watermark。如果有某个分区没有watermark，整个pipeline的watermark都不回前进
- 保证分区的watermark单调递增。

Dataflow中有个中心化的聚合代理（agent），来处理watermark的聚合。agent是分布式的，可以更有效的做聚合操作。从正确行的角度来看，只有这个聚合agent才知道真正正确的watermark。

这种分布式watermark聚合的架构中，如何保证watermark的正确性是一个非常大的挑战。首先，必须保证watermark不能被提前触发，否则会造成很多的late event。每个worker都会被分配一段key空间，这个worker需要负责这段key的数据的state的更新/清理工作。在更新某个worker进程的watermark的时候，必须保证这个worker还在负责这段key的state的管理。否则，watermark更新协议就必须负责校验state的归属情况。

<br/>

#### Apache Flink中的watermark

 Flink是一个开源的流处理框架，能够支持分布式，高性能，高可用的流计算应用，且保证数据正确性。flink的计算管道示意图如下：

 {% asset_img figure3-18.png %}

<br/>

 这个管道中有两个数据源，每个数据源都会产生一个watermark检查点，会与数据一起被发送到下游。比如，对数据源A来说，输出53这个watermark就意味这，不会再输出53之前的数据。下游的keyby算子会收到53这个watermark与其对应的数据。从operator的视角来看，他会不断的接到数据及watermark，并不断输出数据和watermark。

分布式发送watermark的方式，与DataFlow中心化管理watermark的方式相比，有几个优势：

- 减少watermark传输的时延
- 没有watermark聚合agent这种单点问题
- 可扩展性更强
  当然DataFlow中心化管理watermark的方式也有一些优点：
- Single source of “truth”：在一些debug场景中，如果能从一个地方拿到所有数据的watermark处理信息，会非常方便分析问题。而分布式watermark架构中，debug就非常麻烦
- 创建源头watermark：在某些场景中，需要其他系统的一些信息来产生watermark，使用中心化方式处理watermark就很方便了