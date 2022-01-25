#  尚硅谷大数据Flink（Java）和（Scala）

> 来源：B站视频、尚硅谷公众号提供的资料以及个人小结笔记。
>
> [Java版Flink教程](https://www.bilibili.com/video/BV1qy4y1q728)
>
> [Scala版Flink教程](https://www.bilibili.com/video/BV1Qp4y1Y7YN)
>
> > Flink官网地址<https://flink.apache.org>

# 第一章 Flink的简介

## 1.1 Flink是什么？

![](https://flink.apache.org/img/flink-header-logo.svg)

Apache Flink 是一个<font color=red>框架</font>和<font color=red>分布式</font>处理引擎，用于在<font color=red>无界和有界数据流</font>上进行有<font color=red>状态</font>的计算。

## 1.2 Flink的特点

### 1.2.1 事件型驱动（Event-driven）

事件驱动型应用是一类具有状态的应用，它从一个或多个事件流提取数据，并根据到来的事件触发计算、状态更新或其他外部动作。

![](https://flink.apache.org/img/usecases-eventdrivenapps.png)

### 1.2.2 流与批的世界观

在Flink的世界观中，一切都是由流组成的，离线数据是有界的流；实时数据是一个没有界限的流。

![](https://nightlies.apache.org/flink/flink-docs-release-1.14/fig/bounded-unbounded.png)



### 1.2.3 分层API

Flink 为流式/批式处理应用程序的开发提供了不同级别的抽象。

**越顶层越抽象，表达含义越简明，使用越方便**

**越底层越具体，表达能力越丰富，使用越灵活**

![](https://flink.apache.org/img/api-stack.png)

* 

## 1.3 Flink VS Spark Streaming

* 数据模型
  * spark采用RDD模型，spark streaming的DataStream实际上也就是一组组小批数据RDD的集合
  * flink基本数据模型是数据流，以及事件（Event）序列
* 运行时架构
  * spark是批计算，将DAG划分为不同的stage，一个完成后才可以计算下一个
  * flink是标准的流执行模式，一个事件在一个节点处理完后可以直接发往下一个节点进行处理

# 第二章 快速上手

## 2.1 IDEA中搭建maven工程导入相关依赖

说明：Flink的底层是用Java编写的，但是Flink包下runtime的这个组件用到akka工具。akka底层是用scala写的。所以这里也要引入scala版本。

> Flink运行报错 <font color=red>No ExecutorFactory found to execute the application</font>
>
> 从Flink1.11开始，移除了flink-streaming-java对flink-clients的依赖，需要手动加入clients依赖。

<!-- tabs:start -->

#### **Java**

```xml
<properties>
    <flink.version>1.13.0</flink.version>
    <java.version>1.8</java.version>
    <scala.binary.version>2.12</scala.binary.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-java</artifactId>
        <version>${flink.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-streaming-java_${scala.binary.version}</artifactId>
        <version>${flink.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-clients_${scala.binary.version}</artifactId>
        <version>${flink.version}</version>
    </dependency>
</dependencies>
```

#### **Scala**

```xml
<properties>
    <flink.version>1.13.0</flink.version>
    <scala.binary.version>2.12</scala.binary.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-scala_${scala.binary.version}</artifactId>
        <version>${flink.version}</version>
    </dependency>
    <!-- https://mvnrepository.com/artifact/org.apache.flink/flink-streaming-scala -->
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-streaming-scala_${scala.binary.version}</artifactId>
        <version>${flink.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-clients_${scala.binary.version}</artifactId>
        <version>${flink.version}</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <!-- 该插件用于将 Scala 代码编译成 class 文件 -->
        <plugin>
            <groupId>net.alchim31.maven</groupId>
            <artifactId>scala-maven-plugin</artifactId>
            <version>4.4.0</version>
            <executions>
                <execution>
                    <!-- 声明绑定到 maven 的 compile 阶段 -->
                    <goals>
                        <goal>compile</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-assembly-plugin</artifactId>
            <version>3.3.0</version>
            <configuration>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
            </configuration>
            <executions>
                <execution>
                    <id>make-assembly</id>
                    <phase>package</phase>
                    <goals>
                        <goal>single</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

<!-- tabs:end -->

## 2.2 批处理wordcount

<!-- tabs:start -->

#### **Java**

```java
import org.apache.flink.api.java.ExecutionEnvironment;
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.DataSet;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.util.Collector;

// 批处理word count
public class WordCount {
    public static void main(String[] args) throws Exception {
        // 创建执行环境
        ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

        // 从文件中读取数据
        String inputPath = "src/main/resources/hello.txt";
        DataSet<String> inputDataSet = env.readTextFile(inputPath);

        // 对数据集进行处理，按空格分词展开，转换成(word,1)二元组进行统计
        DataSet<Tuple2<String, Integer>> resultSet = inputDataSet.flatMap(new MyFlatMapper())
                .groupBy(0)       // 按照第一个位置的word分组
                .sum(1);          // 将第二个位置上的数据求和

        resultSet.print();
    }

    // 自定义类，实现FlatMapFunction接口
    public static class MyFlatMapper implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String value, Collector<Tuple2<String, Integer>> out) throws Exception {
            // 按空格分词
            String[] words = value.split(" ");
            // 遍历所有word，包成二元组输出
            for (String word : words) {
                out.collect(new Tuple2<>(word, 1));
            }
        }
    }
}
```

#### **Scala**

```scala
import org.apache.flink.api.scala.ExecutionEnvironment
import org.apache.flink.api.scala._

// 批处理的wordcount
object WordCount {
  def main(args: Array[String]): Unit = {
    // 创建执行环境
    val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

    // 从文件中读取数据
    val inputPath = "src/main/resources/hello.txt"
    val inputDataSet: DataSet[String] = env.readTextFile(inputPath)

    //对数据进行转换处理，先分词。在按照word进行分组，最后进行聚合统计
    val resultDataSet: AggregateDataSet[(String, Int)] = inputDataSet
      .flatMap(_.split(" "))
      .map((_, 1))
      .groupBy(0)  		// 以第一个元素作为key，进行分组
      .sum(1)      		// 对所有数据的第二个元素求和

    // 打印输出
    resultDataSet.print()
  }
}
```

<!-- tabs:end -->

运行结果：

<!-- tabs:start -->

#### **Java**

```java
(scala,1)
(flink,1)
(world,1)
(hello,4)
(and,1)
(fine,1)
(how,1)
(spark,1)
(you,3)
(are,1)
(thank,1)
```

#### **Scala**

```scala

```

<!-- tabs:end -->

## 2.3 流处理wordcount

### 文件测试

不同于批处理方式，来了一组在处理。而是流式处理，来一个处理一个。

* 这里没有批处理的groupBy方法。而是使用流处理的keyBy方法。按照指定key进行不同的划分（按照当前key的hashcode对数据进行重分区的操作）
* 流式处理方式。是先把任务启动起来等待数据来。所以要调`execute()`方法执行任务。

<!-- tabs:start -->

#### **Java**

```java
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class StreamWordCount {
    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        // 设置并行度
        env.setParallelism(1);

        // 从文件中读取数据
        String inputPath = "src/main/resources/hello.txt";
        DataStream<String> inputDataStream = env.readTextFile(inputPath);

        // 基于数据流进行转换计算
        DataStream<Tuple2<String, Integer>> resultStream = inputDataStream
                .flatMap(new WordCount.MyFlatMapper())
                .keyBy(0)
                .sum(1);

        resultStream.print();

        // 执行任务
        env.execute();
    }
}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

运行结果：

可以看出每一个word不止一个输出，尽管读的是文件也是来一个处理一个。相当于中间保存了每一个word的状态。在之前的状态基础上叠加上来。

每个数字代表的并行执行的线程编号。可以手动设置并行度。

<!-- tabs:start -->

#### **Java**

```java
13> (flink,1)
11> (how,1)
5> (hello,1)
9> (fine,1)
1> (spark,1)
10> (you,1)
10> (you,2)
1> (scala,1)
8> (are,1)
6> (thank,1)
9> (world,1)
5> (hello,2)
15> (and,1)
10> (you,3)
5> (hello,3)
5> (hello,4)
```

#### **Scala**

```scala

```

<!-- tabs:end -->

### 流式数据源测试

使用一个简单易用的小工具nc（netcat）实现流式输出的功能。

在Linux环境里自带的。window需要安装

利用nc命令`nc -lk <port>`在终端启动一个socket服务

```shell
nc -lk 7777
```

<!-- tabs:start -->

#### **Java**

```java
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.api.java.utils.ParameterTool;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class StreamWordCount {
    public static void main(String[] args) throws Exception {
        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        // 设置并行度
        env.setParallelism(1);

        // 从文件中读取数据
//        String inputPath = "src/main/resources/hello.txt";
//        DataStream<String> inputDataStream = env.readTextFile(inputPath);
        
        // 用parameter tool工具从程序启动参数中提取配置项
        ParameterTool parameterTool = ParameterTool.fromArgs(args);
        String hostname = parameterTool.get("hostname");
        int port = parameterTool.getInt("port");

        // 从socket文本流读取数据
        DataStreamSource<String> inputDataStream = env.socketTextStream(hostname, port);


        // 基于数据流进行转换计算
        DataStream<Tuple2<String, Integer>> resultStream = inputDataStream
                .flatMap(new WordCount.MyFlatMapper())
                .keyBy(0)
                .sum(1);

        resultStream.print();

        // 执行任务
        env.execute();
    }
}

```

#### **Scala**

```scala

```

<!-- tabs:end -->

# 第三章 Flink的部署

## 3.1 Standalone模式



## 3.2 Yarn模式

## 3.3 Kubernetes部署

# 第四章 Flink的运行架构

## 4.1 Flink运行的组件

## 4.2 任务提交流程

## 4.3 任务调度原理



### 4.3.1 TaskManger与Slots

![](https://nightlies.apache.org/flink/flink-docs-release-1.14/fig/tasks_slots.svg)



![](https://nightlies.apache.org/flink/flink-docs-release-1.14/fig/slot_sharing.svg)

### 4.3.2 程序与数据流（DataFlow）

![](https://nightlies.apache.org/flink/flink-docs-master/fig/program_dataflow.svg)

### 4.3.3 执行图（ExecutionGraph）

### 4.3.4 并行度（Parallelism）

### 4.3.5 任务链（Operator Chains）

![](https://nightlies.apache.org/flink/flink-docs-master/fig/parallel_dataflow.svg)

# 第五章 Flink流处理API

## 5.1 Environment

### 5.1.1 getExecutionEnvironment

创建一个执行环境，表示当前执行程序的上下文。 如果程序是独立调用的，则此方法返回本地执行环境；如果从命令行客户端调用程序以提交到集群，则此方法返回此集群的执行环境，也就是说，getExecutionEnvironment 会根据查询运行的方式决定返回什么样的运行环境，是最常用的一种创建执行环境的方式。

<!-- tabs:start -->

#### **Java**

```java
// 批处理执行环境
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// 流处理执行环境
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
```

#### **Scala**

```scala
// 批处理执行环境
val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

// 流处理执行环境
val env = StreamExecutionEnvironment.getExecutionEnvironment
```

<!-- tabs:end -->

如果没有设置并行度，会以flink-conf.yaml中的配置为准，默认是1。

```yaml
# The parallelism used for programs that did not specify and other parallelism.

parallelism.default: 1
```

### 5.1.2 createLocalEnvironment

返回本地执行环境，需要在调用时指定默认的并行度。

<!-- tabs:start -->

#### **Java**

```java
LocalStreamEnvironment env = StreamExecutionEnvironment.createLocalEnvironment(1);
```

#### **Scala**

```scala
val env = StreamExecutionEnvironment.createLocalEnvironment(1)
```

<!-- tabs:end -->

### 5.1.3 createRemoteEnvironment

返回集群执行环境，将 Jar 提交到远程服务器。需要在调用时指定 JobManager 的 IP 和端口号，并指定要在集群中运行的 Jar 包。

<!-- tabs:start -->

#### **Java**

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.createRemoteEnvironment(
    "jobmanage-hostname", 
    6123, 
    "YOURPATH//wordcount.jar");
```

#### **Scala**

```scala
val env = StreamExecutionEnvironment.createRemoteEnvironment(
    "jobmanage-hostname", 
    6123,
    "YOURPATH//wordcount.jar")
```

<!-- tabs:end -->

## 5.2 Source

### 5.2.1 从集合中读取数据

<!-- tabs:start -->

#### **Java**

```java
import com.atguigu.beans.SensorReading;// 定义SensorReading类
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

import java.util.Arrays;

public class SourceTest1_Collection {
    public static void main(String[] args) throws Exception {
        // 创建执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        
        // 可以设置并行度为1，让一条流上的数据顺序输出
        env.setParallelism(1);

        // 从集合中读取数据
        DataStream<SensorReading> dataStream = env.fromCollection(Arrays.asList(
                new SensorReading("sensor_1", 1547718199L, 35.8),
                new SensorReading("sensor_6", 1547718201L, 15.4),
                new SensorReading("sensor_7", 1547718202L, 6.7),
                new SensorReading("sensor_10", 1547718205L, 38.1)
        ));

        // 直接指定元素
        DataStream<Integer> integerDataStream = env.fromElements(1, 2, 4, 67, 189);

        // 打印输出
        dataStream.print("data");
        integerDataStream.print("int");

        // 执行任务
        env.execute("jobName");
    }
}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

运行结果

<!-- tabs:start -->

#### **Java**

```java
int:2> 67
int:3> 189
int:1> 4
data:16> SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
data:2> SensorReading{id='sensor_7', timestamp=1547718202, temperature=6.7}
data:1> SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
data:3> SensorReading{id='sensor_10', timestamp=1547718205, temperature=38.1}
int:15> 1
int:16> 2
```

#### **Scala**

```scala

```

<!-- tabs:end -->

### 5.2.2 从文件中读取数据

<!-- tabs:start -->

#### **Java**

```java
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

// 从文件中读取数据
public class SourceTest2_File {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // 从文件中读取数据
        DataStream<String> dataStream = env.readTextFile("src/main/resources/sensor.txt");

        // 打印输出
        dataStream.print();

        env.execute("从文件中读取数据");
    }
}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

sensor.txt的内容

<!-- tabs:start -->

#### **Java**

```java
sensor_1,1547718199L,35.8
sensor_6,1547718201L,15.4
sensor_7,1547718202L,6.7
sensor_10,1547718205,38.1
```

#### **Scala**

```scala

```

<!-- tabs:end -->

运行结果

<!-- tabs:start -->

#### **Java**

```java
7> sensor_7,1547718202L,6.7
16> sensor_1,1547718199L,35.8
3> sensor_6,1547718201L,15.4
11> sensor_10,1547718205,38.1
```

#### **Scala**

```scala

```

<!-- tabs:end -->

### 5.2.3 从Kafka消息队列读取数据

引入flink-connector-kafka依赖

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-kafka_${scala.binary.version}</artifactId>
    <version>${flink.version}</version>
</dependency>
```

代码实现

<!-- tabs:start -->

#### **Java**

```java
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer;

import java.util.Properties;

public class SourceTest3_Kafka {
    public static void main(String[] args) throws Exception {
        // 获取执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // Kafka配置项
        Properties properties = new Properties();
        properties.setProperty("bootstrap.servers","hadoop102");
        // 次要配置
        properties.setProperty("group.id", "consumer-group");
        properties.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        properties.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        properties.setProperty("auto.offset.reset", "latest");

        // 从Kafka中读取数据
        DataStream<String> dataStream = env.addSource(new FlinkKafkaConsumer<String>(
                "sensor",
                new SimpleStringSchema(),
                properties
        ));

        // 打印输出
        dataStream.print();

        // 执行任务
        env.execute("读取Kafka数据");
    }
}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

1）启动Zookeeper

```shell
$ ./bin/zkServer.sh start
```

2）启动Kafka

```shell
$ ./bin/kafka-server-start.sh -daemon ./config/server.properties
```

3）启动kafka生产者

```shell
$ ./bin/kafka-console-producer.sh --broker-list hadoop102:9092 --topic sensor
```

4）测试

```shell
$ bin/kafka-console-producer.sh --broker-list hadoop102:9092  --topic sensor
>sensor_1,1547718199,35.8
>sensor_6,1547718201,15.4
>sensor_7,1547718202L,6.7
>sensor_10,1547718205,38.1
>
```

结果输出

<!-- tabs:start -->

#### **Java**

```java
sensor_1,1547718199,35.8
sensor_6,1547718201,15.4
sensor_7,1547718202,6.7
sensor_10,1547718205,38.1
```

#### **Scala**

```scala

```

<!-- tabs:end -->

### 5.2.4 自定义Source

除了以上的 source 数据来源，我们还可以自定义 source。需要做的，只是传入一个 SourceFunction 就可以。

<!-- tabs:start -->

#### **Java**

```java
import com.atguigu.beans.SensorReading;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.SourceFunction;

import java.util.HashMap;
import java.util.Random;

public class SourceTest4_UDF {
    public static void main(String[] args) throws Exception {
        // 获取执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        DataStream<SensorReading> dataStream = env.addSource(new MySensorSource());

        // 打印输出
        dataStream.print();

        // 执行任务
        env.execute("自定义Source Test");
    }

    // 实现自定义的SourceFunction
    public static class MySensorSource implements SourceFunction<SensorReading> {
        // 定义一个标识位，用来控制数据的产生
        private boolean running = true;

        @Override
        public void run(SourceContext<SensorReading> ctx) throws Exception {
            // 定义一个随机数发生器
            Random random = new Random();

            // 设置10个传感器的初始温度
            HashMap<String, Double> sensorTempMap = new HashMap<>();
            for (int i = 0; i < 10; i++) {
                sensorTempMap.put("sensor_" + (i+1), 60 + random.nextGaussian() * 20);
            }

            while (running) {
                for (String sensorId: sensorTempMap.keySet()){
                    // 在当前温度基础上随机波动
                    double newTemp = sensorTempMap.get(sensorId) + random.nextGaussian();
                    sensorTempMap.put(sensorId,newTemp);
                    ctx.collect(new SensorReading(sensorId,System.currentTimeMillis(),newTemp));
                }
                // 控制输出频率
                Thread.sleep(1000L);
            }
        }

        @Override
        public void cancel() {
            running = false;
        }
    }
}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

## 5.3 Transform

### 5.3.1 简单转化算子（map flatmap Filter）

<!-- tabs:start -->

#### **Java**

```java
import org.apache.flink.api.common.functions.FilterFunction;
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;

public class TransformTest1_Base {
    public static void main(String[] args) throws Exception {
        // 获取执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        // 从文件中读取数据
        DataStream<String> inputStream = env.readTextFile("src/main/resources/sensor.txt");

        // 1.map 把String转换成长度输出
        DataStream<Integer> mapStream = inputStream.map(new MapFunction<String, Integer>() {
            @Override
            public Integer map(String value) throws Exception {
                return value.length();
            }
        });

        // 2.flatmap 按照逗号分字段
        DataStream<String> flatMapStream = inputStream.flatMap(new FlatMapFunction<String, String>() {
            @Override
            public void flatMap(String value, Collector<String> out) throws Exception {
                String[] fields = value.split(",");
                for (String field : fields) {
                    out.collect(field);
                }
            }
        });

        // 3.filter 筛选senor_1开头的id对应数据
        DataStream<String> filterStream = inputStream.filter(new FilterFunction<String>() {
            @Override
            public boolean filter(String value) throws Exception {
                return value.startsWith("sensor_1");
            }
        });

        // 打印输出
        mapStream.print("map");
        flatMapStream.print("flatMap");
        filterStream.print("filter");

        env.execute("简单转化算子 Test");
    }
}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

运行结果

<!-- tabs:start -->

#### **Java**

```java
map> 24
flatMap> sensor_1
flatMap> 1547718199
flatMap> 35.8
filter> sensor_1,1547718199,35.8
map> 24
flatMap> sensor_6
flatMap> 1547718201
flatMap> 15.4
map> 23
flatMap> sensor_7
flatMap> 1547718202
flatMap> 6.7
map> 25
flatMap> sensor_10
flatMap> 1547718205
flatMap> 38.1
filter> sensor_10,1547718205,38.1
```

#### **Scala**

```scala

```

<!-- tabs:end -->

### 5.3.2 KeyBy

**DataStream **→ **KeyedStream**：逻辑地将一个流拆分成不相交的分区，每个分区包含具有相同 key 的元素，在内部以 hash 的形式实现的。

### 5.3.3 滚动聚合算子（Rolling Aggregation）

这些算子可以针对KeyStream的每一个支流做聚合

* sum()
* min()
* max()
* minBy()
* maxBy()

<!-- tabs:start -->

#### **Java**

```java
import com.atguigu.beans.SensorReading;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.java.tuple.Tuple;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class TransformTest2_RollingAggregation {
    public static void main(String[] args) throws Exception {
        // 获取执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        // 从文件中读取数据
        DataStream<String> inputStream = env.readTextFile("src/main/resources/sensor.txt");

        // 转换成SensorReading类型
        // 实现一: 匿名内部类
//        DataStream<SensorReading> dataStream = inputStream.map(new MapFunction<String, SensorReading>() {
//            @Override
//            public SensorReading map(String value) throws Exception {
//                String[] fields = value.split(",");
//                return new SensorReading(fields[0], new Long(fields[1]),new Double(fields[2]));
//            }
//        });

        // 实现二: lambda表达式
        DataStream<SensorReading> dataStream = inputStream.map(line -> {
            String[] fields = line.split(",");
            return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
        });

        // 分组
        KeyedStream<SensorReading, Tuple> keyedStream = dataStream.keyBy("id");
//        KeyedStream<SensorReading, String> keyedStream1 = dataStream.keyBy(SensorReading::getId);

        // 滚动聚合，取当前最大的温度值
        DataStream<SensorReading> resultStream = keyedStream.maxBy("temperature");

        // 打印输出
        resultStream.print();

        env.execute("RollingAggregation Test");
    }
}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

新增一些测试数据sensor.txt

<!-- tabs:start -->

#### **Java**

```java
sensor_1,1547718199,35.8
sensor_6,1547718201,15.4
sensor_7,1547718202,6.7
sensor_10,1547718205,38.1
sensor_6,1547718207,36.3
sensor_6,1547718209,32.8
sensor_6,1547718212,37.1
```

#### **Scala**

```java

```

<!-- tabs:end -->

运行结果：滚动更新，按照tempature字段，每次输出最大值。

<!-- tabs:start -->

#### **Java**

```java
SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
SensorReading{id='sensor_7', timestamp=1547718202, temperature=6.7}
SensorReading{id='sensor_10', timestamp=1547718205, temperature=38.1}
SensorReading{id='sensor_6', timestamp=1547718207, temperature=36.3}
SensorReading{id='sensor_6', timestamp=1547718207, temperature=36.3}
SensorReading{id='sensor_6', timestamp=1547718212, temperature=37.1}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

### 5.3.4 Reduce

**KeyedStream** **→** **DataStream**：一个分组数据流的聚合操作，合并当前的元素和上次聚合的结果，产生一个新的值，返回的流中包含每一次聚合的结果，而不是只返回最后一次聚合的最终结果。

<!-- tabs:start -->

#### **Java**

```java
import com.atguigu.beans.SensorReading;
import org.apache.flink.api.common.functions.ReduceFunction;
import org.apache.flink.api.java.tuple.Tuple;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class TransformTest3_Reduce {
    public static void main(String[] args) throws Exception {
        // 获取执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        // 从文件中读取数据
        DataStream<String> inputStream = env.readTextFile("src/main/resources/sensor.txt");

        // 转换成SensorReading类型
        DataStream<SensorReading> dataStream = inputStream.map(line -> {
            String[] fields = line.split(",");
            return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
        });

        // 分组
        KeyedStream<SensorReading, Tuple> keyedStream = dataStream.keyBy("id");

        // reduce聚合，取最大的温度值，以及当前最新的时间戳
//        DataStream resultStream = keyedStream.reduce(new ReduceFunction<SensorReading>() {
//            @Override
//            public SensorReading reduce(SensorReading value1, SensorReading value2) throws Exception {
//                return new SensorReading(
//                        value1.getId(),
//                        value2.getTimestamp(),
//                        Math.max(value1.getTemperature(), value2.getTemperature()));
//            }
//        });

        DataStream resultStream = keyedStream.reduce((value1, value2) -> new SensorReading(
                value1.getId(),
                value2.getTimestamp(),
                Math.max(value1.getTemperature(), value2.getTemperature())));

        // 打印输出
        resultStream.print();

        env.execute("RollingAggregation Test");
    }
}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

reduce更加一般化的聚合操作

运行结果：不同于聚合操作。输出最大值并且输出当前的时间戳。

<!-- tabs:start -->

#### **Java**

```java
SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
SensorReading{id='sensor_7', timestamp=1547718202, temperature=6.7}
SensorReading{id='sensor_10', timestamp=1547718205, temperature=38.1}
SensorReading{id='sensor_6', timestamp=1547718207, temperature=36.3}
SensorReading{id='sensor_6', timestamp=1547718209, temperature=36.3}
SensorReading{id='sensor_6', timestamp=1547718212, temperature=37.1}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

### 5.3.5 Split 和 Select（新版已删除）

> <font color=red>注意</font>：新版本这两个方法已被删除。使用新版本的侧输出流调用底层（ProcessFunction API 使用 SideOutput）进行分流操作。

`Split`

**DataStream** → **SplitStream**：根据某些特征把一个 DataStream 拆分成两个或者多个 DataStream。

`Select`

**SplitStream** → **DataStream**：从一个 SplitStream 中获取一个或者多个DataStream。

需求：传感器数据按照温度高低（以 30 度为界），拆分成两个流。

> 以下使用的是flink-1.10.1版本实现的

<!-- tabs:start -->

#### **Java**

```java
import com.atguigu.beans.SensorReading;
import org.apache.flink.streaming.api.collector.selector.OutputSelector;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SplitStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

import java.util.Collections;

public class TransformTest4_MultipleStreams {
    public static void main(String[] args) throws Exception {
        // 获取执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        // 从文件中读取数据
        DataStream<String> inputStream = env.readTextFile("src/main/resources/sensor.txt");

        // 转换成SensorReading类型
        DataStream<SensorReading> dataStream = inputStream.map(line -> {
            String[] fields = line.split(",");
            return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
        });

        // 1.分流，按照温度值30°为界分为两条流
        SplitStream<SensorReading> splitStream = dataStream.split(new OutputSelector<SensorReading>() {
            @Override
            public Iterable<String> select(SensorReading value) {
                return (value.getTemperature() > 30) ? Collections.singletonList("high") : Collections.singletonList("low");
            }
        });

        DataStream<SensorReading> highTempStream = splitStream.select("high");
        DataStream<SensorReading> lowTempStream = splitStream.select("low");
        DataStream<SensorReading> allTempStream = splitStream.select("high", "low");

        highTempStream.print("high");
        lowTempStream.print("low");
        allTempStream.print("all");

        env.execute("多流转化算子");
    }
}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

运行结果

<!-- tabs:start -->

#### **Java**

```java
high> SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
all> SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
low> SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
all> SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
low> SensorReading{id='sensor_7', timestamp=1547718202, temperature=6.7}
all> SensorReading{id='sensor_7', timestamp=1547718202, temperature=6.7}
high> SensorReading{id='sensor_10', timestamp=1547718205, temperature=38.1}
all> SensorReading{id='sensor_10', timestamp=1547718205, temperature=38.1}
high> SensorReading{id='sensor_6', timestamp=1547718207, temperature=36.3}
all> SensorReading{id='sensor_6', timestamp=1547718207, temperature=36.3}
high> SensorReading{id='sensor_6', timestamp=1547718209, temperature=32.8}
all> SensorReading{id='sensor_6', timestamp=1547718209, temperature=32.8}
high> SensorReading{id='sensor_6', timestamp=1547718212, temperature=37.1}
all> SensorReading{id='sensor_6', timestamp=1547718212, temperature=37.1}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

### 5.3.6 Connect 和 CoMap

**DataStream,DataStream** **→** **ConnectedStreams**：连接两个保持他们类型的数据流，两个数据流被 Connect 之后，只是被放在了一个同一个流中，内部依然保持各自的数据和形式不发生任何变化，两个流相互独立。

**ConnectedStreams → DataStream**：作用于 ConnectedStreams 上，功能与 map和 flatMap 一样，对 ConnectedStreams 中的每一个 Stream 分别进行 map 和 flatMap 处理。

<!-- tabs:start -->

#### **Java**

```java
import com.atguigu.beans.SensorReading;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.api.java.tuple.Tuple3;
import org.apache.flink.streaming.api.collector.selector.OutputSelector;
import org.apache.flink.streaming.api.datastream.ConnectedStreams;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.datastream.SplitStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.co.CoMapFunction;

import java.util.Collections;

public class TransformTest4_MultipleStreams {
    public static void main(String[] args) throws Exception {
        // 获取执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        // 从文件中读取数据
        DataStream<String> inputStream = env.readTextFile("src/main/resources/sensor.txt");

        // 转换成SensorReading类型
        DataStream<SensorReading> dataStream = inputStream.map(line -> {
            String[] fields = line.split(",");
            return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
        });

        // 1.分流，按照温度值30°为界分为两条流
        SplitStream<SensorReading> splitStream = dataStream.split(new OutputSelector<SensorReading>() {
            @Override
            public Iterable<String> select(SensorReading value) {
                return (value.getTemperature() > 30) ? Collections.singletonList("high") : Collections.singletonList("low");
            }
        });

        DataStream<SensorReading> highTempStream = splitStream.select("high");
        DataStream<SensorReading> lowTempStream = splitStream.select("low");
        DataStream<SensorReading> allTempStream = splitStream.select("high", "low");

        highTempStream.print("high");
        lowTempStream.print("low");
        allTempStream.print("all");

        // 2. 合流connect，将高温流转换成二元组类型，与低温流连接合并之后，输出状态信息
        DataStream<Tuple2<String, Double>> warningStream = highTempStream.map(new MapFunction<SensorReading, Tuple2<String, Double>>() {
            @Override
            public Tuple2<String, Double> map(SensorReading value) throws Exception {
                return new Tuple2<>(value.getId(), value.getTemperature());
            }
        });

        ConnectedStreams<Tuple2<String, Double>, SensorReading> connectedStreams = warningStream.connect(lowTempStream);

        SingleOutputStreamOperator<Object> resultStream = connectedStreams.map(new CoMapFunction<Tuple2<String, Double>, SensorReading, Object>() {
            @Override
            public Object map1(Tuple2<String, Double> value) throws Exception {
                return new Tuple3<>(value.f0, value.f1, "high temp warning");
            }

            @Override
            public Object map2(SensorReading value) throws Exception {
                return new Tuple2<>(value.getId(), "normal");
            }
        });

        resultStream.print();

        env.execute("多流转化算子");
    }
}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

运行结果

<!-- tabs:start -->

#### **Java**

```java
(sensor_1,35.8,high temp warning)
(sensor_6,normal)
(sensor_10,38.1,high temp warning)
(sensor_7,normal)
(sensor_6,36.3,high temp warning)
(sensor_6,32.8,high temp warning)
(sensor_6,37.1,high temp warning)
```

#### **Scala**

```scala

```

<!-- tabs:end -->

### 5.3.7 Union

**DataStream** **→** **DataStream**：对两个或者两个以上的 DataStream 进行 union 操作，产生一个包含所有 DataStream 元素的新 DataStream。

<!-- tabs:start -->

#### **Java**

```java
DataStream<SensorReading> unionStream = highTempStream.union(lowTempStream, allTempStream);
```

#### **Scala**

```scala

```

<!-- tabs:end -->

**Connect** 与 **Union** 区别：

1．Union 之前两个流的类型必须是一样，Connect 可以不一样，在之后的 coMap中再去调整成为一样的。

2．Connect 只能操作两个流，Union 可以操作多个。

## 5.4 支持的数据类型

Flink 流应用程序处理的是以数据对象表示的事件流。所以在 Flink 内部，我们需要能够处理这些对象。它们需要被<font color=red>序列化和反序列化</font>，以便通过网络传送它们；或者从状态后端、检查点和保存点读取它们。为了有效地做到这一点，Flink 需要明确知道应用程序所处理的数据类型。Flink 使用类型信息的概念来表示数据类型，并为每个数据类型生成特定的序列化器、反序列化器和比较器。

Flink 还具有一个类型提取系统，该系统分析函数的输入和返回类型，以自动获取类型信息，从而获得序列化器和反序列化器。但是，在某些情况下，例如 lambda 函数或泛型类型，需要显式地提供类型信息，才能使应用程序正常工作或提高其性能。

Flink 支持 Java 和 Scala 中所有常见数据类型。使用最广泛的类型有以下几种。

### 5.4.1 基础数据类型

Flink 支持所有的 Java 和 Scala 基础数据类型，Int, Double, Long, String, …

<!-- tabs:start -->

#### **Java**

```java
DataStream<Integer> numberStream = env.fromElements(1, 2, 3, 4);
numberStream.map(data -> data * 2);
```

#### **Scala**

```scala
val numbers: DataStream[Long] = env.fromElements(1L, 2L, 3L, 4L)
numbers.map(n => n + 1)
```

<!-- tabs:end -->

### 5.4.2 Java和Scala元组（Tuples）

java的元组类型由Flink的包提供，提供Tuple0~Tuple25

<!-- tabs:start -->

#### **Java**

```java
DataStream<Tuple2<String, Integer>> personStream = env.fromElements(
    new Tuple2("zhen", 17),
    new Tuple2("han", 21));
personStream.filter(p -> p.f1 > 18);
```

#### **Scala**

```scala
val persons: DataStream[(String, Integer)] = env.fromElements(
    ("zhen", 17), 
    ("han", 21)
)
persons.filter(p => p._2 > 18)
```

<!-- tabs:end -->

### 5.4.3 Scala样例类（case classes）

```scala
case class Person(name: String, age: Int)

val persons: DataStream[Person] = env.fromElements(
    Person("zhen", 17),
    Person("han", 21)
)
persons.filter(p => p.age > 18)
```

### 5.4.4 Java简单对象（POJOs）

要求：必须提供无参的构造方法。属性字段必须是public或者是private（提供get、set方法）

```java
public class Person {
    public String name;
    public int age;

    public Person() {}

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}

DataStream<Person> persons = env.fromElements(
    new Person("zhen", 17),
    new Person("han", 21));
```

### 5.4.5 其他

Flink 对 Java 和 Scala 中的一些特殊目的的类型也都是支持的，比如 Java 的 ArrayList，HashMap，Enum 等等。

## 5.5 实现UDF函数——更细粒度的控制流

### 5.5.1 函数类（Function Classes）

Flink 暴露了所有 udf 函数的接口(实现方式为接口或者抽象类)。例如 MapFunction, FilterFunction, ProcessFunction 等等。

下面例子实现了 FilterFunction 接口：

<!-- tabs:start -->

#### **Java**

```java
public static class FlinkFilter implements FilterFunction<String> {
    @Override
    public boolean filter(String value) throws Exception {
        return value.contains("flink");
    }
}

DataStream<String> flinkTweets = tweets.filter(new FlinkFilter());
```

#### **Scala**

```scala
class FilterFilter extends FilterFunction[String] {
  override def filter(value: String): Boolean = {
    value.contains("flink")
  }
}

val flinkTweets = tweets.filter(new FlinkFilter)
```

<!-- tabs:end -->

还可以将函数实现匿名类

<!-- tabs:start -->

#### **Java**

```java
DataStream<String> flinkTweets = tweets.filter(new FilterFunction<String>() {
    @Override
    public boolean filter(String value) throws Exception {
        return value.contains("flink");
    }
});
```

#### **Scala**

```scala
val flinkTweets = tweets.filter(new RichFilterFunction[String] {
    override def filter(value: String): Boolean = {
        value.contains("flink")
    }
})
```

<!-- tabs:end -->

实现单独的类，通用性更强。比如：我们 filter 的字符串"flink"还可以当作参数传进去。

<!-- tabs:start -->

#### **Java**

```java
DataStream<String> tweets = env.readTextFile("INPUT_FILE ");

DataStream<String> flinkTweets = tweets.filter(new KeyWordFilter("flink"));

public static class KeyWordFilter implements FilterFunction<String> {
    private String keyWord;

    KeyWordFilter(String keyWord) { this.keyWord = keyWord; }

    @Override
    public boolean filter(String value) throws Exception {
        return value.contains(this.keyWord);
    }
}
```

#### **Scala**

```scala
val tweets: DataStream[String] = env.readTextFile("INPUT_FILE ");

val flinkTweets = tweets.filter(new KeywordFilter("flink"))

class KeywordFilter(keyWord: String) extends FilterFunction[String] {
  override def filter(value: String): Boolean = {
    value.contains(keyWord)
  }
}
```

<!-- tabs:end -->

### 5.5.2 匿名函数（Lambda Function）

<!-- tabs:start -->

#### **Java**

```java
DataStream<String> tweets = env.readTextFile("INPUT_FILE");

DataStream<String> flinkTweets = tweets.filter(tweet -> tweet.contains("flink"));
```

#### **Scala**

```scala
val tweets: DataStream[String] = env.readTextFile("INPUT_FILE ")

val flinkTweets = tweets.filter(_.contains("flink"))
```

<!-- tabs:end -->

### 5.5.3 富函数（Rich Functions）

“富函数”是 DataStream API 提供的一个函数类的接口，所有 Flink 函数类都有其 Rich 版本。它与常规函数的不同在于，<font color=red>可以获取运行环境的上下文，并拥有一些生命周期方法，所以可以实现更复杂的功能</font>。

* RichMapFunction 

* RichFlatMapFunction 

* RichFilterFunction 

* …

Rich Function 有一个生命周期的概念。典型的**生命周期方法**有：

* `open()`方法是 rich function 的初始化方法，当一个算子例如 map 或者 filter 被调用之前 `open()` 会被调用。
* `close()` 方法是生命周期中的最后一个调用的方法，做一些清理工作。
* `getRuntimeContext()` 方法提供了函数的 RuntimeContext 的一些信息，例如函数执行的并行度，任务的名字，以及 state 状态。

<!-- tabs:start -->

#### **Java**

```java
public static class MyMapFunction extends RichMapFunction<SensorReading, Tuple2<Integer, String>> {
    @Override
    public Tuple2<Integer, String> map(SensorReading value) throws Exception {
        return new Tuple2<>(getRuntimeContext().getIndexOfThisSubtask(), value.getId());
    }

    @Override
    public void open(Configuration parameters) throws Exception {
        System.out.println("open");
        // 以下可以做一些初始化工作，例如建立一个和 HDFS 的连接
    }

    @Override
    public void close() throws Exception {
        System.out.println("close");
        // 以下做一些清理工作，例如断开和 HDFS 的连接
    }
}
```

#### **Scala**

```scala
class MyFlatMap extends RichFlatMapFunction[Int, (Int, Int)] {
  var subTaskIndex = 0

  override def open(configuration: Configuration): Unit = {
    subTaskIndex = getRuntimeContext.getIndexOfThisSubtask
    // 以下可以做一些初始化工作，例如建立一个和 HDFS 的连接
  }

  override def flatMap(in: Int, out: Collector[(Int, Int)]): Unit = {
    if (in % 2 == subTaskIndex) {
      out.collect((subTaskIndex, in))
    }
  }

  override def close(): Unit = {
    // 以下做一些清理工作，例如断开和 HDFS 的连接。
  }
}
```

<!-- tabs:end -->

测试代码

<!-- tabs:start -->

#### **Java**

```java
import com.atguigu.beans.SensorReading;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.common.functions.RichMapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class TransformTest5_RichFunction {
    public static void main(String[] args) throws Exception {
        // 获取执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(4);

        // 从文件中读取数据
        DataStream<String> inputStream = env.readTextFile("src/main/resources/sensor.txt");

        // 转换成SensorReading类型
        DataStream<SensorReading> dataStream = inputStream.map(line -> {
            String[] fields = line.split(",");
            return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
        });

        DataStream<Tuple2<String, Integer>> resultStream = dataStream.map(new MyMapper());

        resultStream.print();

        env.execute("RichFunction Test");
    }

    public static class MyMapper0 implements MapFunction<SensorReading, Tuple2<String, Integer>> {
        @Override
        public Tuple2<String, Integer> map(SensorReading value) throws Exception {
            return new Tuple2<>(value.getId(), value.getId().length());
        }
    }

    // 实现自定义富函数类
    public static class MyMapper extends RichMapFunction<SensorReading, Tuple2<String, Integer>> {
        @Override
        public Tuple2<String, Integer> map(SensorReading value) throws Exception {
            // 获取状态
//            getRuntimeContext().getState();
            return new Tuple2<>(value.getId(), getRuntimeContext().getIndexOfThisSubtask());
        }

        @Override
        public void open(Configuration parameters) throws Exception {
            // 初始化工作，一般是定义状态，或者建立数据库连接
            System.out.println("open");
        }

        @Override
        public void close() throws Exception {
            // 一般是关闭连接和清空状态的收尾操作
            System.out.println("close");
        }
    }
}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

运行结果

由于设置了执行环境env的并行度为4，所以有4个slot执行自定义的RichFunction，输出4次open和close

<!-- tabs:start -->

#### **Java**

```java
open
open
open
open
3> (sensor_1,2)
4> (sensor_7,3)
2> (sensor_6,1)
1> (sensor_6,0)
1> (sensor_6,0)
4> (sensor_10,3)
3> (sensor_6,2)
close
close
close
close
```

#### **Scala**

```scala

```

<!-- tabs:end -->

### 5.5.4 数据重分区

<!-- tabs:start -->

#### **Java**

```java
import com.atguigu.beans.SensorReading;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class TransformTest6_Partition {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(4);

        // 从文件中读取数据
        DataStream<String> inputStream = env.readTextFile("src/main/resources/sensor.txt");

        // 转换成SensorReading类型
        DataStream<SensorReading> dataStream = inputStream.map(line -> {
            String[] fields = line.split(",");
            return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
        });

        dataStream.print("input");

        // 1.shuffle        随机打乱，然后分区
        DataStream<String> shuffleStream = inputStream.shuffle();

        shuffleStream.print("shuffle");

        // 2.keyBy          按照Hashcode取模分区
        dataStream.keyBy("id").print("KeyBy");

        // 3.global         全部发送到第一个分区
        dataStream.global().print("global");

        env.execute("重分区");
    }
}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

测试结果

<!-- tabs:start -->

#### **Java**

```java
input:1> SensorReading{id='sensor_7', timestamp=1547718202, temperature=6.7}
input:3> SensorReading{id='sensor_6', timestamp=1547718212, temperature=37.1}
input:2> SensorReading{id='sensor_6', timestamp=1547718207, temperature=36.3}
input:4> SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
input:2> SensorReading{id='sensor_6', timestamp=1547718209, temperature=32.8}
input:4> SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
input:1> SensorReading{id='sensor_10', timestamp=1547718205, temperature=38.1}
shuffle:4> sensor_6,1547718207,36.3
shuffle:4> sensor_6,1547718212,37.1
shuffle:3> sensor_6,1547718209,32.8
shuffle:4> sensor_7,1547718202,6.7
shuffle:3> sensor_10,1547718205,38.1
KeyBy:3> SensorReading{id='sensor_6', timestamp=1547718212, temperature=37.1}
shuffle:3> sensor_6,1547718201,15.4
global:1> SensorReading{id='sensor_6', timestamp=1547718212, temperature=37.1}
KeyBy:3> SensorReading{id='sensor_6', timestamp=1547718207, temperature=36.3}
global:1> SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
KeyBy:3> SensorReading{id='sensor_6', timestamp=1547718209, temperature=32.8}
KeyBy:3> SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
KeyBy:3> SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
KeyBy:4> SensorReading{id='sensor_7', timestamp=1547718202, temperature=6.7}
global:1> SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
shuffle:4> sensor_1,1547718199,35.8
global:1> SensorReading{id='sensor_6', timestamp=1547718207, temperature=36.3}
KeyBy:2> SensorReading{id='sensor_10', timestamp=1547718205, temperature=38.1}
global:1> SensorReading{id='sensor_6', timestamp=1547718209, temperature=32.8}
global:1> SensorReading{id='sensor_7', timestamp=1547718202, temperature=6.7}
global:1> SensorReading{id='sensor_10', timestamp=1547718205, temperature=38.1}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

## 5.6 Sink

Flink 没有类似于 spark 中 foreach 方法，让用户进行迭代的操作。虽有对外的输出操作都要利用 Sink 完成。最后通过类似如下方式完成整个任务最终输出操作。

```java
stream.addSink(new MySink(xxxx))
```

 官方提供了一部分的框架的sink。除此以外，需要用户自定义实现sink。

### 5.6.1 Kafka

引入pom依赖

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-kafka_${scala.binary.version}</artifactId>
    <version>${flink.version}</version>
</dependency>
```

<!-- tabs:start -->

#### **Java**

```java
import com.atguigu.beans.SensorReading;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducer;

import java.util.Properties;

public class SinkTest1_Kafka {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        // 从文件中读取数据
//        DataStream<String> inputStream = env.readTextFile("src/main/resources/sensor.txt");

        // Kafka配置项
        Properties properties = new Properties();
        properties.setProperty("bootstrap.servers","hadoop102");
        // 次要配置
        properties.setProperty("group.id", "consumer-group");
        properties.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        properties.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        properties.setProperty("auto.offset.reset", "latest");

        // 从Kafka中读取数据
        DataStream<String> inputStream = env.addSource(new FlinkKafkaConsumer<String>(
                "sensor",
                new SimpleStringSchema(),
                properties
        ));

        DataStream<String> dataStream = inputStream.map(line -> {
            String[] fields = line.split(",");
            return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2])).toString();
        });

        dataStream.addSink(new FlinkKafkaProducer<String>("hadoop102:9092", "sinkTest",new SimpleStringSchema()));

        env.execute("KafkaSink Test");
    }
}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

测试如下

1）启动Zookeeper

```shell
$ ./bin/zkServer.sh start
```

2）启动Kafka

```shell
$ ./bin/kafka-server-start.sh -daemon ./config/server.properties
```

3）启动kafka生产者

```shell
$ ./bin/kafka-console-producer.sh --broker-list hadoop102:9092 --topic sensor
```

4）启动kafka消费者

```shell
$ ./bin/kafka-console-consumer.sh --bootstrap-server hadoop102:9092 --topic sinktest
```

5）运行Flink程序，在kafka生产者console输入数据，查看kafka消费者console的输出结果

输入(kafka生产者console)

```shell
$ ./bin/kafka-console-producer.sh --broker-list hadoop102:9092 --topic sensor
>sensor_1,1547718199,35.8
>sensor_6,1547718201,15.4
>
```

输出(kafka消费者console)

```shell
$ ./bin/kafka-console-consumer.sh --bootstrap-server hadoop102:9092 --topic sinktest
>sensor_1,1547718199,35.8
>sensor_6,1547718201,15.4
>
```

这里类似于做了一个数据管道（pipeline），Kafka进Kafka出。可以看做用Flink做了一个ETL过程。

### 5.6.2 Redis

> 查询Flink-Redis连接器 [flink-connector-redis](https://mvnrepository.com/search?q=flink-connector-redis)

导入依赖

```xml
<!-- https://mvnrepository.com/artifact/org.apache.bahir/flink-connector-redis -->
<dependency>
    <groupId>org.apache.bahir</groupId>
    <artifactId>flink-connector-redis_${scala.binary.version}</artifactId>
    <version>1.0</version>
</dependency>
```

<!-- tabs:start -->

#### **Java**

```java
import com.atguigu.beans.SensorReading;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.redis.RedisSink;
import org.apache.flink.streaming.connectors.redis.common.config.FlinkJedisPoolConfig;
import org.apache.flink.streaming.connectors.redis.common.mapper.RedisCommand;
import org.apache.flink.streaming.connectors.redis.common.mapper.RedisCommandDescription;
import org.apache.flink.streaming.connectors.redis.common.mapper.RedisMapper;

public class SinkTest2_Redis {
    public static void main(String[] args) throws Exception {
        // 获取执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        // 从文件中读取数据
        DataStream<String> inputStream = env.readTextFile("src/main/resources/sensor.txt");

        // 转换成SensorReading类型
        DataStream<SensorReading> dataStream = inputStream.map(line -> {
            String[] fields = line.split(",");
            return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
        });

        // 定义jedis连接配置
        FlinkJedisPoolConfig config = new FlinkJedisPoolConfig.Builder()
                .setHost("hadoop102")
                .setPort(6379)
                .build();


        dataStream.addSink(new RedisSink<>(config,new MyRedisMapper()));

        env.execute("RedisSink Test");
    }

    public static class MyRedisMapper implements RedisMapper<SensorReading>{
        // 定义保存数据到Redis的命令，存成Hash表，Hset sensor_temp id temperature
        @Override
        public RedisCommandDescription getCommandDescription() {
            return new RedisCommandDescription(RedisCommand.HSET,"sensor_temp");
        }

        @Override
        public String getKeyFromData(SensorReading data) {
            return data.getId();
        }

        @Override
        public String getValueFromData(SensorReading data) {
            return data.getTemperature().toString();
        }
    }
}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

测试

1）启动redis-server

```shell
$ ./redis-serve
```

2）启动redis-cli

```shell
$ ./redis-cli
```

3）查看redis数据

```shell
hadoop102:6379>hgetall sensor_temp
1) "sensor_1"
2) "37.1"
3) "sensor_6"
4) "15.4"
5) "sensor_7"
6) "6.7"
7) "sensor_10"
8) "38.1"
```

### 5.6.3 Elasticsearch

导入依赖。这里ES版本是7

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-elasticsearch7_${scala.binary.version}</artifactId>
    <version>${flink.version}</version>
</dependency>
```

<!-- tabs:start -->

#### **Java**

```java
import com.atguigu.beans.SensorReading;
import org.apache.flink.api.common.functions.RuntimeContext;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.elasticsearch.ElasticsearchSinkFunction;
import org.apache.flink.streaming.connectors.elasticsearch.RequestIndexer;
import org.apache.flink.streaming.connectors.elasticsearch7.ElasticsearchSink;
import org.apache.http.HttpHost;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.client.Requests;

import java.util.ArrayList;
import java.util.HashMap;

public class SinkTest3_Es {
    public static void main(String[] args) throws Exception {
        // 获取执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        // 从文件中读取数据
        DataStream<String> inputStream = env.readTextFile("src/main/resources/sensor.txt");

        // 转换成SensorReading类型
        DataStream<SensorReading> dataStream = inputStream.map(line -> {
            String[] fields = line.split(",");
            return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
        });

        // 定义es的连接配置
        ArrayList<HttpHost> httpHosts = new ArrayList<>();
        httpHosts.add(new HttpHost("hadoop102", 9200));

        dataStream.addSink(new ElasticsearchSink.Builder<SensorReading>(httpHosts, new MyEsSinkFunction()).build());

        env.execute("EsSink Test");
    }

    // 实现自定义的ES写入操作
    public static class MyEsSinkFunction implements ElasticsearchSinkFunction<SensorReading> {
        @Override
        public void process(SensorReading element, RuntimeContext ctx, RequestIndexer indexer) {
            // 定义写入的数据source
            HashMap<String, String> dataSource = new HashMap<>();
            dataSource.put("id", element.getId());
            dataSource.put("temp", element.getTemperature().toString());
            dataSource.put("ts", element.getTimestamp().toString());

            // 创建请求，作为向Es发起的写入命令
            IndexRequest indexRequest = Requests.indexRequest()
                    .index("sensor")
//                    .type()         //ES7统一type就是_doc，不再允许指定type
                    .source(dataSource);

            // 用index发送请求
            indexer.add(indexRequest);
        }
    }
}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

测试

1）启动Es

```shell
$ ./bin/elasticsearch
```

2）启动程序。使用 curl 查看数据

```shell
$ curl "hadoop102:9200/sensor/_search?pretty"
```

### 5.6.4 JDBC 自定义sink

> [JDBC Connector](https://nightlies.apache.org/flink/flink-docs-release-1.13/zh/docs/connectors/datastream/jdbc/)。官方连接器没有Mysql所以我们自定义实现。

测试Mysql连接

导入pom依赖

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.21</version>
</dependency>
```

<!-- tabs:start -->

#### **Java**

```java
import com.atguigu.beans.SensorReading;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.sink.RichSinkFunction;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;

public class SinkTest4_Jdbc {
    public static void main(String[] args) throws Exception {
        // 获取执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        // 从文件中读取数据
        DataStream<String> inputStream = env.readTextFile("src/main/resources/sensor.txt");

        // 转换成SensorReading类型
        DataStream<SensorReading> dataStream = inputStream.map(line -> {
            String[] fields = line.split(",");
            return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
        });

        dataStream.addSink(new MyJdbcSink());

        env.execute("JdbcSink Test");
    }

    // 实现自定义的SinkFunction
    public static class MyJdbcSink extends RichSinkFunction<SensorReading> {
        // 声明连接和预编译语句
        Connection connection = null;
        PreparedStatement insertStmt = null;
        PreparedStatement updateStmt = null;

        @Override
        public void open(Configuration parameters) throws Exception {
            connection = DriverManager.getConnection("jdbc:mysql://localhost3306/flink_test", "root", "000000");
            insertStmt = connection.prepareStatement("insert into sersor_temp(id,temp) values(?,?)");
            updateStmt = connection.prepareStatement("update sensor_temp set temp = ? where id = ?");
        }

        // 每来一条数据，调用连接，执行sql
        @Override
        public void invoke(SensorReading value) throws Exception {
            // 直接执行更新语句，如果没有更新那么就插入
            updateStmt.setDouble(1, value.getTemperature());
            updateStmt.setString(2, value.getId());
            updateStmt.execute();
            if (updateStmt.getUpdateCount() == 0) {
                insertStmt.setString(1, value.getId());
                insertStmt.setDouble(2, value.getTemperature());
                insertStmt.execute();
            }
        }

        @Override
        public void close() throws Exception {
            insertStmt.close();
            updateStmt.close();
            connection.close();
        }
    }
}
```

#### **Scala**

```scala

```

<!-- tabs:end -->

测试

1）启动Mysql，新建数据库

```sql
CREATE DATABASE `flink_test` DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
```

2）新建表

```sql
CREATE TABLE `sensor_temp` (
  `id` varchar(32) NOT NULL,
  `temp` double NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

3）运行程序，查看Mysql里的数据（轮询查询，可以看出sensor在一直变化）

```sql
mysql> SELECT * FROM sensor_temp;
+-----------+--------------------+
| id        | temp               |
+-----------+--------------------+
| sensor_3  | 20.489172407885917 |
| sensor_10 |  73.01289164711463 |
| sensor_4  | 43.402500895809744 |
| sensor_1  |  6.894772325662007 |
| sensor_2  | 101.79309911751122 |
| sensor_7  | 63.070612021580324 |
| sensor_8  |  63.82606628090501 |
| sensor_5  |  57.67115738487047 |
| sensor_6  |  50.84442627975055 |
| sensor_9  |  52.58400793021675 |
+-----------+--------------------+
10 rows in set (0.00 sec)

mysql> SELECT * FROM sensor_temp;
+-----------+--------------------+
| id        | temp               |
+-----------+--------------------+
| sensor_3  | 19.498209543035923 |
| sensor_10 |  71.92981963197121 |
| sensor_4  | 43.566017489470426 |
| sensor_1  |  6.378208186786803 |
| sensor_2  | 101.71010087830145 |
| sensor_7  |  62.11402602179431 |
| sensor_8  |  64.33196455020062 |
| sensor_5  |  56.39071692662006 |
| sensor_6  | 48.952784757264894 |
| sensor_9  | 52.078086096436685 |
+-----------+--------------------+
10 rows in set (0.00 sec)
```



# 第六章 Flink中的Window

## 6.1 Window

### 6.1.1 Window概述

![](https://flink.apache.org/img/bounded-unbounded.png)

* 一般真实的流都是无界的，怎样处理无界的数据？
* 可以把无限的数据流进行切分，得到有限的数据集进行处理——也就是得到有界流
* 窗口（window）就是<font color=red>将无限流切割为有限流</font>的一种方式，他会将流数据分发到有限大小的桶（bucket）中进行分析

### 6.1.2 Window的类型

* 时间窗口（Time Window）
  * 滚动时间窗口
  * 滑动时间窗口
  * 会话窗口
* 计数窗口（Count Window）
  * 滚动计数窗口
  * 滑动计数窗口

**滚动窗口（Tumbling Windows）**

![](https://nightlies.apache.org/flink/flink-docs-release-1.14/fig/tumbling-windows.svg)

* 将数据依据固定的窗口长度对数据进行切分
* <font color=red>时间对齐，窗口长度固定，没有重叠</font>。

**滑动窗口（Sliding Windows）**

![](https://nightlies.apache.org/flink/flink-docs-release-1.14/fig/sliding-windows.svg)

* 滑动窗口是固定窗口的更广义的一种形式，滑动窗口由固定的窗口和滑动间隔组成
* <font color=red>窗口长度固定，可以有重叠</font>。

**会话窗口（Session Windows）**

![](https://nightlies.apache.org/flink/flink-docs-release-1.14/fig/session-windows.svg)

* 由一系列事件组合一个指定时间长度的timeout间隙组成，也就是一段时间没有接收到新的数据就会生成新的窗口
* <font color=red>特点：时间无对齐</font>

## 6.2 Window API

* 窗口分配器——`window()`方法

* 我们可以用 `.window()` 来定义一个窗口，然后基于这个window去做一些聚合或者其他处理操作。

  <font color=red>注意</font>：window() 方法必须在 keyBy 之后才能使用

* Flink提供了更加简单的 `.timeWindow()` 和 `.countWindow()` 方法，用于定义时间窗口和计数窗口。

<!-- tabs:start -->

#### **Java**

```java
DataStream<Tuple2<String,Double>> minTempPerWindowStream = datastream
    .map(new MyMapper())
    .keyBy(data -> data.f0)
    .timeWindow(Time.seconds(15))
    .minBy(1);
```

#### **Scala**

```scala
val minTempPerWindow = dataStream
	.map(r => (r.id, r.temperature))
	.keyBy(_._1).timeWindow(Time.seconds(15))
	.reduce((r1, r2) => (r1._1, r1._2.min(r2._2)))
```

<!-- tabs:end -->

### 6.2.1 TimeWindow

TimeWindow 是将指定时间范围内的所有数据组成一个 window，一次对一个 window 里面的所有数据进行计算。

**1）滚动窗口**

Flink 默认的时间窗口根据 Processing Time 进行窗口的划分，将 Flink 获取到的数据根据进入 Flink 的时间划分到不同的窗口中。

<!-- tabs:start -->

#### **Java**

```java
DataStream<Tuple2<String, Double>> minTempPerWindowStream = dataStream
    .map(new MapFunction<SensorReading, Tuple2<String, Double>>() {
        @Override
        public Tuple2<String, Double> map(SensorReading value) throws
            Exception {
            return new Tuple2<>(value.getId(), value.getTemperature());
        }
    })
    .keyBy(data -> data.f0)
    .timeWindow(Time.seconds(15))
    .minBy(1);
```

#### **Scala**

```scala
val minTempPerWindow = dataStream
	.map(r => (r.id, r.temperature))
	.keyBy(_._1).timeWindow(Time.seconds(15))
	.reduce((r1, r2) => (r1._1, r1._2.min(r2._2)))
```

<!-- tabs:end -->

时间间隔可以通过 `Time.milliseconds(x)`，`Time.seconds(x)`，`Time.minutes(x) `等其中的一个来指定。

**2）滑动窗口**

<!-- tabs:start -->

#### **Java**

```java
DataStream<SensorReading> minTempPerWindowStream = dataStream
    .keyBy(SensorReading::getId)
    .timeWindow(Time.seconds(15), Time.seconds(5))
    .minBy("temperature");
```

#### **Scala**

```scala
val minTempPerWindow: DataStream[(String, Double)] = dataStream
	.map(r => (r.id, r.temperature))
	.keyBy(_._1)
	.timeWindow(Time.seconds(15), Time.seconds(5))
	.reduce((r1, r2) => (r1._1, r1._2.min(r2._2)))
//	  .window(SlidingEventTimeWindows.of(Time.seconds(15), Time.seconds(5))
```

<!-- tabs:end -->

时间间隔可以通过 `Time.milliseconds(x)`，`Time.seconds(x)`，`Time.minutes(x) `等其中的一个来指定。

### 6.2.2 CountWindow

CountWindow 根据窗口中相同 key 元素的数量来触发执行，执行时只计算元素数量达到窗口大小的 key 对应的结果。 

<font color=red>注意</font>：CountWindow 的 window_size 指的是相同 Key 的元素的个数，不是输入的所有元素的总数。

**1）滚动窗口**

默认的 CountWindow 是一个滚动窗口，只需要指定窗口大小即可，**当元素数量达到窗口大小时，就会触发窗口的执行**。

<!-- tabs:start -->

#### **Java**

```java
DataStream<SensorReading> minTempPerWindowStream = dataStream
    .keyBy(SensorReading::getId)
    .countWindow(5)
    .minBy("temperature");
```

#### **Scala**

```scala
val minTempPerWindow: DataStream[(String, Double)] = dataStream
	.map(r => (r.id, r.temperature))
	.keyBy(_._1)
	.countWindow(5)
	.reduce((r1, r2) => (r1._1, r1._2.max(r2._2)))
```

<!-- tabs:end -->

**2）滑动窗口**

滑动窗口和滚动窗口的函数名是完全一致的，只是在传参数时需要传入两个参数，一个是 window_size，一个是 sliding_size。 

下面代码中的 sliding_size 设置为了 2，也就是说，每收到两个相同 key 的数据就计算一次，每一次计算的 window 范围是 10 个元素。

<!-- tabs:start -->

#### **Java**

```java
DataStream<SensorReading> minTempPerWindowStream = dataStream
    .keyBy(SensorReading::getId)
    .countWindow(10, 2)
    .minBy("temperature");
```

#### **Scala**

```scala
val keyedStream: KeyedStream[(String, Int), Tuple] = dataStream
	.map(r => (r.id, r.temperature))
	.keyBy(0) //每当某一个 key 的个数达到 2 的时候,触发计算，计算最近该 key 最近 10 个元素的内容

val windowedStream: WindowedStream[(String, Int), Tuple, GlobalWindow] = keyedStream.countWindow(10, 2)
val sumDataStream: DataStream[(String, Int)] = windowedStream.sum(1)
```

<!-- tabs:end -->

### 6.2.3 window function

window function 定义了要对窗口中收集的数据做的计算操作，主要可以分为两类：

* 增量聚合函数（incremental aggregation functions） 

每条数据到来就进行计算，保持一个简单的状态。典型的增量聚合函数有 

ReduceFunction, AggregateFunction。 

* 全窗口函数（full window functions） 

先把窗口所有数据收集起来，等到计算的时候会遍历所有数据。 

ProcessWindowFunction 就是一个全窗口函数。

#  第七章 Flink的时间语义和Watermark

**1）滚动窗口**

<!-- tabs:start -->

#### **Java**

```java

```

#### **Scala**

```scala

```

<!-- tabs:end -->

**2）滑动窗口**

<!-- tabs:start -->

#### **Java**

```java

```

#### **Scala**

```scala

```

<!-- tabs:end -->

## 7.1 Flink中的时间语义

![](https://nightlies.apache.org/flink/flink-docs-master/fig/event_processing_time.svg)

## 7.2 EventTime的引入

## 7.3 Watermark

<!-- tabs:start -->

#### **Java**

```java

```

#### **Scala**

```scala

```

<!-- tabs:end -->

## 7.4 EventTime在window中的使用

<!-- tabs:start -->

#### **Java**

```java

```

#### **Scala**

```scala

```

<!-- tabs:end -->

# 第八章 ProcessFunction API（底层API）

## 8.1 KeyedProcessFunction

## 8.2 TimeService和定时器（Timers）

## 8.3 侧输出流（SideOutput）

## 8.4 CoProcessFunction

<!-- tabs:start -->

#### **Java**

```java

```

#### **Scala**

```scala

```

<!-- tabs:end -->

# 第九章 状态编程和容错机制

## 9.1 有状态的算子和应用程序

## 9.2 状态一致性

## 9.3 检查点（checkpoint）

## 9.4 选择一个状态后端（state backend）

<!-- tabs:start -->

#### **Java**

```java

```

#### **Scala**

```scala

```

<!-- tabs:end -->

# 第十章 Flink的Table API和SQL
<!-- tabs:start -->

#### **Java**

```java

```

#### **Scala**

```scala

```

<!-- tabs:end -->

# 第十一章 Flink CEP

<!-- tabs:start -->

#### **Java**

```java

```

#### **Scala**

```scala

```

<!-- tabs:end -->

# 第十二章 

<!-- tabs:start -->

#### **Java**

```java

```

#### **Scala**

```scala

```

<!-- tabs:end -->