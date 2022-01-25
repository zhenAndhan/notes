# Flink（Java版）

> Flink官网地址<https://flink.apache.org>
>
> 以下使用的是Flink1.13版本的

# 一、Flink流处理简介

## 1.1 Flink是什么

![](https://flink.apache.org/img/flink-header-logo.svg)

Apache Flink 是一个<font color=red>框架</font>和<font color=red>分布式</font>处理引擎，用于在<font color=red>无界和有界数据流</font>上进行有<font color=red>状态</font>的计算。

## 1.2 为什么选择Flink

* 流数据更真实地反映了我们的生活方式
* 传统的数据架构是基于有限数据集的
* 我们的目标
  *  低延迟（Spark Streaming 的延迟是秒级，Flink 延迟是毫秒级）
  *  高吞吐
  * 结果的准确性和良好的容错性（exactly-once） 

## 1.3 Flink的主要特点

### 事件驱动（Event-driven） 

![](https://flink.apache.org/img/usecases-eventdrivenapps.png)

### 基于流的世界观

在 Flink 的世界观中，一切都是由流组成的，离线数据是有界的流；实时数据是一个没有界限的流：这就是所谓的界流和无界流。

![](https://nightlies.apache.org/flink/flink-docs-release-1.14/fig/bounded-unbounded.png)

### 分层API

* **越顶层越抽象，表达含义越简明，使用越方便**
* **越底层越具体，表达能力越丰富，使用越灵活**

![](https://nightlies.apache.org/flink/flink-docs-release-1.14/fig/levels_of_abstraction.svg)

## 1.4 Flink的其他特点

*  支持<font color=red>事件时间（event-time）</font>和<font color=red>处理时间（processing-time）</font>语义
*  <font color=red>精确一次（exactly-once）的状态一致性保证</font>
*  <font color=red>低延迟</font>，每秒处理数百万个事件，毫秒级延迟（实际上就是没有延迟）
*  与众多常用<font color=red>存储系统的连接</font>（ES、HBase、MySQL、Redis…）
*  <font color=red>高可用</font>（zookeeper），动态扩展，实现 7*24 小时全天候运行

## 1.5 Flink vs Spark Streaming

* 流（stream）和微批
* 数据模型
  * Spark 采用 RDD 模型，Spark Streaming 的 DataStream 实际上也就是一组组小批数据 RDD 的集合
  * Flink 基本数据模型是数据流，以及事件（Event）序列（Integer、String、Long、POJO Class）
* 运行时架构
  * Spark 是批计算，将 DAG 划分为不同的 Stage，一个 Stage 完成后才可以计算下一个 Stage
  * Flink 是标准的流执行模式，一个事件在一个节点处理完后可以直接发往下一个节点进行处理

## 1.6 Flink上手Word Count

在IDEA中搭建maven工程导入相关依赖

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
```

Wordcount程序实现

```java
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.common.functions.ReduceFunction;
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;

// word count
public class WordCount {
    // 记得抛出异常
    public static void main(String[] args) throws Exception {
        // 获取流处理的运行时环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        // 设置并行任务的数量为1
        env.setParallelism(1);

        // 读取数据源
        // 从socket读取数据然后处理
        // 先在终端启动 `nc -lk 9999`
        DataStreamSource<String> stream = env.socketTextStream("localhost", 9999);

        // 从离线数据读取数据然后处理
//        DataStreamSource<String> stream = env.fromElements("hello world", "hello world");

        // map操作
        // 这里使用的flatMap方法
        // map: 针对流中的每一个元素，输出一个元素
        // flatMap: 针对流中的每一个元素，输出0个、1个或多个元素
        SingleOutputStreamOperator<WordWithCount> mappedStream = stream
                // 输入泛型: String; 输出泛型: WordWithCount
                .flatMap(new FlatMapFunction<String, WordWithCount>() {
                    @Override
                    public void flatMap(String value, Collector<WordWithCount> out) throws Exception {
                        String[] words = value.split(" ");
                        // 使用collect方法向下游发送数据
                        for (String word : words) {
                            out.collect(new WordWithCount(word, 1L));
                        }
                    }
                });

        // 分组: shuffle
        KeyedStream<WordWithCount, String> keyedStream = mappedStream
                // 第一个泛型: 流中元素的泛型
                // 第二个泛型: key的泛型
                .keyBy(new KeySelector<WordWithCount, String>() {
                    @Override
                    public String getKey(WordWithCount value) throws Exception {
                        return value.word;
                    }
                });

        // reduce操作
        // reduce会维护一个累加器
        // 第一条数据到来，作为累加器输出
        // 第二条数据到来，和累加器进行聚合操作，然后输出累加器
        // 累加器和流中元素的类型是一样的
        SingleOutputStreamOperator<WordWithCount> result = keyedStream
                .reduce(new ReduceFunction<WordWithCount>() {
                    // 定义了聚合的逻辑
                    @Override
                    public WordWithCount reduce(WordWithCount value1, WordWithCount value2) throws Exception {
                        return new WordWithCount(value1.word, value1.count + value2.count);
                    }
                });

        // 执行程序
        env.execute("word count");
    }

    // POJO类
    // 1. 必须是公有类
    // 2. 所有字段必须是public
    // 3. 必须有空构造器
    // 模拟了case class
    public static class WordWithCount{
        public String word;
        public Long count;

        public WordWithCount() {
        }

        public WordWithCount(String word, Long count) {
            this.word = word;
            this.count = count;
        }

        @Override
        public String toString() {
            return "WordWithCount{" +
                    "word='" + word + '\'' +
                    ", count='" + count + '\'' +
                    '}';
        }
    }
}
```

使用一个简单易用的小工具nc（netcat）实现流式输出的功能。

在Linux环境里自带的。window需要安装

利用nc命令`nc -lk <port>`在终端启动一个socket服务。然后再运行程序

```shell
nc -lk 9999
```

在终端输入数据

```shell
hello world
hello world
```

程序输出

```java
WordWithCount{word='hello', count=1};
WordWithCount{word='world', count=1};
WordWithCount{word='hello', count=2};
WordWithCount{word='world', count=2};
```

# 二、Flink运行架构

## 2.1 Flink运行时的组件

Flink 运行时由两种类型的进程组成：一个 <font color=red>JobManager</font> 和一个或者多个  <font color=red>TaskManager</font>。 典型的 **Master-Slave** 架构



## 2.2 任务提交流程

## 2.3 任务调度原理

![](https://nightlies.apache.org/flink/flink-docs-release-1.14/fig/processes.svg)

# 三、Flink DataStream API