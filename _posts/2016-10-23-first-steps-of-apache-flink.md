---
layout: post
category: tech
title: Apache Flink的初步试用
---

[Apache Flink](http://flink.apache.org)是一个开源的支持大规模流式或批量数据处理的平台。
上周有段时间调研了一下Flink，看了一下官方文档和部分代码，重点是研究了下Checkpointing机制。
然而其中的技术细节比较多，作为一个初步调研，本文还是希望通过几个简单的例子，来说明如何将Flink使用起来。
具体地，本文包括如下几个内容：

* 如何在YARN上部署一个Flink集群
* 如何使用Flink消费Kafka上的数据
* Flink的任务异常恢复与任务重启的测试

<!--snapshot-->
## 在YARN上部署一个Flink集群

Flink集群的部署方式有很多种，本文选取了在YARN的部署方式。
测试的YARN集群是CDH版本的，为了避免以后可能出现的错误，笔者很自觉地从代码编译开始。
首先，我们从[GitHub](http://github.com/apache/flink)拉下项目源码，切换到想要的发行版本，使用如下命令进行编译打包。

{% highlight bash %}
mvn clean install -DskipTests -Pvendor-repos -Dhadoop.version=2.x.x-cdh5.x.x
{% endhighlight %}

然后，在build-target目录下，设定好hadoop或者yarn的相关配置，就可以在YARN上部署一个Flink集群了。

{% highlight bash %}
export HADOOP_CONF_DIR={Hadoop Configuration Directory}
./bin/yarn-session.sh -n 10 -jm 1024 -tm 2048 -d -qu default
{% endhighlight %}

在上述命令中，几个参数的意义如下：

* -n 10 一共启动10个TaskManager节点
* -jm 1024 JobManager的内存大小为1024M
* -tm 2048 TaskManager的内存大小为2048M
* -d 使用detached模式进行部署（部署完成后本地命令行可以退出）
* -qu default 部署到YARN的default队列中

执行完上述命令后，点开ApplicationMaster的链接，可以看到一个Flink Dashboard。
为了能够在后续提交任务到该集群，需要在这个Dashbord上找到并记录下JobManager对应的IP地址和端口号。

## 使用Flink消费Kafka的消息

### 连接Kafka并输出

Flink提供了FlinkKafkaConsumer08和FlinkKafkaConsumer09两个接口，分别用来从Apache Kafka 0.8.x和Apache Kafka 0.9.x拉取数据。
此外，本文使用的是Scala 10对应的API，因此需要添加这两个依赖：flink-streaming-scala_2.10和flink-connector-kafka-0.8_2.10。
首先，设定好kafka连接的相关属性：

{% highlight scala %}
val properties = {
  val prop = new Properties()
  prop.setProperty("bootstrap.servers", "test-01:6667,test-02:6667")
  prop.setProperty("zookeeper.connect","test-01:2181,test-02:2181/kafka")
  prop.setProperty("group.id", "flink-test")
}
val topic = "test_topic"
{% endhighlight%}

然后，连接Kafka并使用StringDeserializer输出Message的value：

{% highlight scala %}
val streamingEnv = StreamExecutionEnvironment.getExecutionEnvironment
val kafkaConsumer = new FlinkKafkaConsumer08[String](topic, new SimpleStringSchema, properties)
streamingEnv.addSource(kafkaConsumer)(BasicTypeInfo.STRING_TYPE_INFO).print()
streamingEnv.execute("test kafka")
{% endhighlight %}

打包好代码后，用如下命令提交到Flink集群上：

{% highlight bash %}
./bin/flink run -p 1 -c com.test.app.FlinkKafkaTest  -m host:port JAR_FILE_PATH
# -p 1, 这个任务使用一个task slot
# -c 任务的Class名
# -m JobManager对应的host和port
{% endhighlight %}

### 使用Flink备份Kafka的Topic

接下来我们看一个稍微复杂一点的例子：使用Flink将Kafka中消息逐小时地备份到HDFS中。
由于是直接备份消息，所以直接将消息地原始字节拿出来写入就行了，为此可以写一个简单的DeserializationSchema如下所示：

{% highlight scala %}
object BytesWritableDeserialization extends DeserializationSchema[FTuple[NullWritable, BytesWritable]] {

  override def isEndOfStream(t: FTuple[NullWritable, BytesWritable]): Boolean = false

  override def getProducedType: TypeInformation[FTuple[NullWritable, BytesWritable]] =
  new TupleTypeInfo[FTuple[NullWritable, BytesWritable]](new WritableTypeInfo[NullWritable](classOf[NullWritable]),
    new WritableTypeInfo[BytesWritable](classOf[BytesWritable]))

  override def deserialize(bytes: Array[Byte]): FTuple[NullWritable, BytesWritable] =
    FTuple.of(NullWritable.get(), new BytesWritable(bytes))

}
{% endhighlight %}

Flink提供了RollingSink，用于将数据以滚动的方式写入到HDFS中。
RollingSink提供了Exactly Once的实现，为了使用它，需要添加flink-connector-filesystem_2.10这个依赖。
由于是直接备份，将Kafka这个Source直接连接到RollingSink这个Sink，就可以完成我们的目标了。

{% highlight scala %}
val sink = new RollingSink[FTuple[NullWritable, BytesWritable]]("hdfs://user/test/temp_flink_out")
sink.setBucketer(new DateTimeBucketer("yyyy-MM-dd/HH"))
sink.setWriter(new SequenceFileWriter[NullWritable, BytesWritable]())
sink.setBatchSize(1024L * 1024 * 120)
val kafkaConsumer = new FlinkKafkaConsumer08[FTuple[NullWritable, BytesWritable]](topic, BytesWritableDeserialization, properties)
streamingEnv.addSource(kafkaConsumer)(BytesWritableDeserialization.getProducedType).addSink(sink)
streamingEnv.execute("kafka-backup")
{% endhighlight %}

## Flink任务的异常与恢复

Flink的Checkpointing机制能够不时的检查和保存Operator的状态。
在任务恢复时，每个Operator都会从最近完成的一个Checkpoint中恢复到当时所保存的状态中。

### 一个有状态的Operator的例子

下面的例子实现了一个RichFlatMapFunction. 这个operator将每个输入转换成Int，然后将其加到一个状态中去。
如果当前的和大于100，则将当前已经处理的事件个数和当前的和放到下一个阶段中去，然后将当前的和置为0。

{% highlight scala %}
object MaxThan100FlatMapper extends RichFlatMapFunction[String, (Long, Long)] with Checkpointed[FTuple[Long, Long]]{
  var state: FTuple[Long, Long] = null
  override def flatMap(value: String, out: Collector[(Long, Long)]): Unit = {
    val rlt = process(value)
    state.f1 = state.f1 + rlt
    state.f0 = state.f0 + 1
    if (state.f1 > 100) {
      out.collect((state.f0, state.f1))
      state.f1 = 0
    }
  }
  def process(value:String) = value.toInt
  override def open(parameters: Configuration): Unit = {
    state = Option(state).getOrElse(FTuple.of(0L, 0L))
  }
  override def restoreState(t: FTuple[Long, Long]): Unit = {
    state = FTuple.of(t.f0, t.f1)
  }
  override def snapshotState(l: Long, l1: Long): FTuple[Long, Long] = state.copy()
}
{% endhighlight %}

### Operator产生异常时Job的恢复

首先，需要设定Checkpointing机制和重启策略。

{% highlight scala %}
streamingEnv.enableCheckpointing(1000)
streamingEnv.setRestartStrategy(RestartStrategies.fixedDelayRestart(2, Time.of(5, TimeUnit.SECONDS)))
stringDataStream.keyBy[Long](x => 1L)(longInfo).flatMap(MaxThan100FlatMapper).print()
{% endhighlight %}

除此以外，我们还需要上述Operator的部分代码，让其能够产生异常:

{% highlight scala %}
var needThrowException: Boolean = true
def process(value:String) = {
  val rlt = value.toInt
  if (rlt >= 5 && needThrowException) {
    throw new Exception("Mocked Exception throwed")
  }
  rlt
}
override def restoreState(t: FTuple[Long, Long]): Unit = {
  LOG.info("restore from self state: " + t.toString)
  state = FTuple.of(t.f0, t.f1)
  needThrowException = false
}
{% endhighlight %}

经过测试，Job能够成功的从异常中恢复。

### 人工重启任务

Flink也提供了savepoint机制，以在人工重启Job的过程中，保存和恢复任务的状态。
{%  highlight scala %}
./bin/flink  savepoint -m host:port acdeaacf610361a6926c477d2207f837
# save the current status
./bin/flink run -p 1 -s jobmanager://savepoints/1  -c com.test.app.FlinkTest  -m host:port JAR_FILE_PATH
# start the job from saved point
{% endhighlight %}
