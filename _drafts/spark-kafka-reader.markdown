---
layout: post
title:  "Read data with Apache Spark from Kafka"
date:   2021-08-04 16:00:00 +0200
categories: howto bigdata
---
## Introduction
In this brief tutorial I'm going to explain how to use Kafka as a data source for Spark.

In general, each message in Kafka belongs to a topic. Each topic is then split into one or more partitions. The number of partitions to assign to a topic is configurable, and it usually depends on your server or cluster configuration.

New messages are appended to a log file for a partition and topic. Each message is assigned a progressive number, called __offset__. When a client reads a message, the original message is not deleted from the partition. Old messages are instead deleted periodically according to a configurable logic, e.g., age, partition file size, and so on.

A Kafka client can choose to read all messages of a partition, or only a subset. For convenience, Kafka provides to clients a way to store the offset of the last read offset, for each topic/partition -- this is the __consumer group__ concept. However, this is not mandatory, i.e., clients can handle this manually.

## Read Kafka data with Spark
When reading data from Kafka in Spark, you have to manage offset for each topic/partition by yourself.

To summarize, the steps are the following:
- for each partition get the last offset you read the last time, or leave empty if this is the first time you read data
- for each partition get from Kafka the current offset of the last message -- this may be optional, see below
- create two JSON string, one for starting offsets, and one for ending offsets, with the offset values for each partition
- read data and get the offset of the most recent message just read, for each partition
- save most recent offset values

The format of the JSON to pass to the Spark reader is the following:
{% highlight json %}
{
    "myTopic": {
        "0": 100,
        "1": 200,
        "2": 300
    }
}
{% endhighlight %}
where in this example, for topic `myTopic` the offset value for partition 0 is 100, for partition 1 is 200, and so on.

### Get the last read offset
Retrieve saved offset, start from 0
{% highlight java %}
long retrieveKafkaInputOffset(String topic, int partition) {
    // TODO:
    // retrieve stored maxOffset for topic/partition,
    // return `null` if not available
}
{% endhighlight %}

### Get the most recent offset now on Kafka
For more control, I get the exact max offset for topic:
{% highlight java %}
long getEndingOffset(String topicName, String kafkaBrokers, int partition) {
    Properties props = new Properties();
    props.put("bootstrap.servers", kafkaBrokers);
    props.put("key.deserializer", StringDeserializer.class.getName());
    props.put("value.deserializer", StringDeserializer.class.getName());
    KafkaConsumer<String, String> kafkaConsumer = null;
    try {
        kafkaConsumer = new KafkaConsumer<>(props);
        TopicPartition actualTopicPartition = new TopicPartition(topicName, partition);
        Set<TopicPartition> partitions = new HashSet<TopicPartition>();
        partitions.add(actualTopicPartition);
        kafkaConsumer.assign(partitions);
        kafkaConsumer.seekToEnd(partitions);
        return kafkaConsumer.position(actualTopicPartition);
    } finally {
        if (kafkaConsumer != null) {
            kafkaConsumer.close();
        }
    }
}
{% endhighlight %}

### Configure the reader and read data from it
In the following code, I assume that I have an optional array of starting offsets. If the array is `null`, I just tell Spark to read from the beginning (i.e., the oldest message present in the partition).

The `i`-th element of the `startingOffsets` array is the starting offset of partition `i`.

{% highlight java %}
private static Dataset<Row> readKafkaInput(SparkSession spark, String kafkaServers,
        String topicName, int nPartitions, long[] startingOffsets) {
    DataFrameReader reader = spark.read()
            .format("kafka")
            .option("kafka.bootstrap.servers", kafkaServers)
            .option("subscribe", topicName);

    if (startingOffsets != null) {
        // Configure reader with starting offsets
        assert startingOffsets.length == nPartitions;
        List<String> partitionOffsetList = new ArrayList<>();
        for (int partition = 0; partition < nPartitions; partition++) {
            long startingOffset = startingOffsets[partition];
            partitionOffsetList.add("\"" + partition + "\": " + startingOffset);
        }
        String startingOffsetJson =
            "{\"" + topicName + "\": {" + String.join(",", partitionOffsetList) + "}}";
        reader = reader.option("startingOffsets", startingOffsetJson);
    }

    // Configure reader with ending offsets
    List<String> partitionOffsetList = new ArrayList<>();
    for (int partition = 0; partition < nPartitions; partition++) {
        long endingOffset = getEndingOffset(topicName, kafkaServers, partition);
        partitionOffsetList.add("\"" + partition + "\": " + endingOffset);
    }
    String endingOffsetJson =
            "{\"" + topicName + "\": {" + String.join(",", partitionOffsetList) + "}}";
    reader = reader.option("endingOffsets", endingOffsetJson);

    return reader.load();
}
{% endhighlight %}

### Save the offset of the last read message
Save the last read offset for the next time:
{% highlight java %}
void persistKafkaInputOffset(Dataset<Row> kafkaInput,
        String topic, int partition) {
    Row minMaxOffset = kafkaInput.agg(
        max("offset").alias("max_offset"))
        .collectAsList()
        .get(0);

    Long maxOffset = minMaxOffset.getAs("max_offset");
    System.out.println("Max read offset: " + maxOffset);

    // TODO store maxOffset for topic/partition somewhere
    // ...
}
{% endhighlight %}

## Final step
Now you have your Dataset returned by method `readKafkaInput()`. You have to decode it in the following way:
{% highlight java %}
Dataset<String> strData = kafkaInput
    .selectExpr("CAST(value AS STRING)")
    .as(Encoders.STRING());
{% endhighlight %}

Now you can use data as normal. For example, if Kafka message contain JSON data, you could simply do the followint to decode it:
{% highlight java %}
Dataset<Row> inputData = spark.sqlContext().read().json(strData);
{% endhighlight %}
