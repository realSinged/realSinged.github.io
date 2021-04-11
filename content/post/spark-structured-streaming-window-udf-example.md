---
title: "使用Spark Structured Streaming的Window和UDF进行数据处理"
date: 2021-04-11T18:03:48+08:00
draft: false
---

最近经历了一个流数据处理的项目，数据源是kafka，需要聚合并分组时间段内的数据，然后基于聚合好的数据集来做分析， 最后再将计算结果数据。经过一番调研，最终选择了spark sructured streaming。因为之前没接触过spark，调研也花了好一番功夫，google了好久，关于spark structured streaming的中Group、Window和UDF基本都是分开使用或讨论。将这三者结合起来的案例少之又少，我也是费了好一番功夫才弄出来（可能是鄙人太菜了，大雾）。 鉴于此，我把项目中的关于Group + Window + UDF的使用，抽了一个demo出来，作成此文。
## 什么是Spark Structured Streaming
Structured是基于Spark SQL引擎构建的可伸缩、可容错的流处理引擎。你可以像对静态数据进行批处理计算一样，来进行流数据计算。当流数据持续到达时，Spark SQL引擎将负责递增地，连续地运行它并更新最终结果。您可以在Scala，Java，Python或R中使用Dataset/DataFrame API来表示流聚合，事件时间窗口，流的批处理等。计算是在同一优化的Spark SQL引擎上执行的。系统通过检查点和预写日志（WAL）来确保端到端的正好一次的容错保证。简而言之，Structured Streaming提供了快速，可扩展，容错，端到端的精确一次流处理，而用户无需关心流。

以上内容摘抄自[Spark官网](),可自行跳转查看。

### Group Function
基于用户指定的方式来聚合DataFrame，[链接在这里](https://spark.apache.org/docs/3.1.1/sql-ref-syntax-qry-select-groupby.html#content)，本文主要使用了基于[窗口的聚合](https://spark.apache.org/docs/3.1.1/structured-streaming-programming-guide.html#basic-operations---selection-projection-aggregation)。
### Window Function
对于流数据，一种是滑动窗口（Sliding Window），一种是滚动窗口(Rolling Window)，就是基于时间对一段时间内聚合。官网比我说的明白，直接上[链接](http://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#window-operations-on-event-time)。本文示例中使用的是滑动窗口
### UDF
全称User-Defined Functions。简单来说，就是可以用户自定义方法来处理DataFrame，给Spark数据处理提供了极强的扩展性。直接上[链接](https://spark.apache.org/docs/3.1.1/sql-ref-functions.html#udfs-user-defined-functions)

## 外部依赖

### Spark 
从[这里](https://spark.apache.org/downloads.html)下载spark，使用的版本是2.4.7，解压到任意位置。然后配置Spark Home的环境变量。 export SPARK_HOME=/the/path/to/spark 即可。

### Kafka
上游输入数据源和下游输出数据源，本地测试可以直接用Docker拉起来即可，以下是docker-compose.yaml文件内容。

    version: '3.7'
    services:
      zookeeper:
        image: wurstmeister/zookeeper
        volumes:
          - ./zookeeper-data:/data
        ports:
          - "2181:2181"
      restart: always

    kafka:
      image: wurstmeister/kafka
      ports:
        - "9092:9092"
      environment:
        KAFKA_BROKER_ID: 0
        #may be you should always change this to you local ip
        KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://127.0.0.1:9092
        #KAFKA_CREATE_TOPICS: "kafeidou:2:0" #kafka启动后初始化一个有2个partition(分区)0个副本名叫kafeidou的topic
        KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
        KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      volumes:
        - ./kafka-logs:/kafka
      depends_on:
        - zookeeper

    networks:
      app-tier:
        driver: bridge

## 使用

1. 订阅kafka数据源

        sdf = (
            spark.readStream.format("kafka")
            .option("kafka.bootstrap.servers", kafka_address)
            .option("subscribe", kafka_source_topics)
            .option("kafka.reconnect.backoff.ms", 2000)
            .option("kafka.reconnect.backoff.max.ms", 10000)
            .option("failOnDataLoss", False)
            .option("backpressure.enabled", True)
            .load()
        )
2. 使用Spark UDF将kafka数据源数据转化成DataFrame

        @F.udf(returnType=transferSchema)
        def transferDF(data):
        #Kafka中数据形式为 {"timestamp": "2006-01-02Z03:04:05", "school": "cambridge", "major": "computer science", "name": "hanmeimei", "extra": ""}
        obj = json.loads(data)
        return (
            datetime.strptime(obj["timestamp"], "%Y-%m-%dT%H:%M:%S.%f"),
            obj["school"],
            obj["major"],
            data,
        )
3. 基于Spark Structured Streaming的窗口来对数据进行分组，分组条件为school和major。 并且设置窗口水位。

        # 水位设置，将滑动窗口中数据以camera_id和region_id分组
        window_group = sdf.withWatermark("timestamp", WATERMARK_DELAY_THRESHOLD).groupBy(
            F.window(F.col("timestamp"), WINDOW_DURATION, WINDOW_SLIDE_DURATION),
            F.col("school"),
            F.col("major"),
        )


4. 将窗口数据转化为UDF可以处理的DataSet。

        result_df_set = window_group.agg(
            handle_grouped_data(
                F.collect_list("data"), F.collect_set("window")
            ).alias(  # 使用窗口数据
                "value_set"
                )

5. 使用UDF来处理聚合好的窗口分组数据，这里面即是用户自定义的数据处理算法及逻辑。

        @F.udf(T.ArrayType(T.BinaryType()))
        def handle_grouped_data(data_list, window) -> T.ArrayType(T.BinaryType()):
            if not window or len(window) != 1:
                print("handle_grouped_data, should have only one window instance")
                return None
            if len(data_list) == 0:
                return None
            sample = data_list[0]
            sample_dict = json.loads(sample)
            print(
                f"date_list length:{len(data_list)},school: {sample_dict['school']}, major: {sample_dict['major']}, window start:{window[0].start}, window end:{window[0].end}"
            )

6. 将处理完成后返回的数据集再拆成DataFrame

        result_df_set = window_group.agg(
        handle_grouped_data(
            F.collect_list("data"), F.collect_set("window")
            ).alias(  # 使用窗口数据
                "value_set"
            )
        ).withColumn(
            "value", F.explode(F.col("value_set"))
        )  # 拆分处理后的数据集

7. 将处理结果输出到Kafka

        query = (
            sdf.writeStream.format("kafka")
            .option("kafka.bootstrap.servers", kafka_address)
            .option("topic", kafka_sink_topic)
            .option("kafka.security.protocol", "SASL_PLAINTEXT")
            .option("kafka.sasl.mechanism", "PLAIN")
            .option("kafka.sasl.jaas.config", kafka_jaas_config)
            .option("kafka.reconnect.backoff.ms", 2000)
            .option("kafka.reconnect.backoff.max.ms", 10000)
            .option("checkpointLocation", spark_check_point_dir)
            .start()
        )

## [完整Demo链接](https://github.com/realSinged/spark-structured-streaming-window-udf-example)

可以使用Demo中的kafka-consumer造一些数据供example来使用。

