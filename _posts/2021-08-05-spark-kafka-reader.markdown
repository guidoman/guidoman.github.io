---
layout: post
title:  "Read data with Apache Spark from Kafka"
date:   2021-08-05 17:30:00 +0200
categories: howto bigdata
---
## Introduction
In this brief tutorial I'm going to explain how to use Kafka as a data source for Spark.

In general, each message in Kafka belongs to a __topic__. Each topic is then split into one or more __partitions__. The number of partitions to assign to a topic is configurable, and it usually depends on your server or cluster configuration.

New messages are appended to a separate log file for each partition and topic. Each message is assigned a progressive identifier, called __offset__. When a client reads a message, the original message is not deleted from the partition. Old messages are instead deleted periodically according to some configurable logic, e.g., age, partition file size, and so on.

A Kafka client can choose to read all messages of a partition, or only a subset. For convenience, Kafka provides to clients a way to store the offset of the last read offset, for each topic/partition -- the so-called __consumer group__ concept. However, this is not mandatory, i.e., clients can handle this manually.

## Read Kafka data with Spark
When reading data from Kafka in Spark, __you have to manage offsets for each topic/partition by yourself__.

The key takeaway here is that you have to __store persistently__ the offset of the last message you read from Kafka. Spark won't do this for you. When you read new data, you have to pass to Spark the last offsets you read the previous time. This reader option is named `startingOffsets`.

Once you've read new data, you retrieve the most recent offsets directly from the dataset and you store them somewhere. You will use this information the next time.

Optionally, you may choose to pass to Spark also the `endingOffsets`. The reader will read messages up to these offsets. I'll show below also how to retrieve this information directly from Kafka. You may choose to do this, e.g., if you want more control on how many data are you reading each time.

In the following sections, I will explain these steps in detail, along with some example __Java code__.

### Get the last read offsets
Let's start by implementing two methods:
- one that retrieves the previously stored offset for a given topic/partiion. If we haven't read any data yet, the result will be empty
- the other that stores that data persistently

The method to retrieve data:

{% highlight java %}
Long retrieveKafkaInputOffset(String topic, int partition) {
    // TODO:
    // retrieve stored maxOffset for topic/partition,
    // return `null` if not available
    // ...
}
{% endhighlight %}

And the other one:

{% highlight java %}
void persistKafkaInputOffset(String topic, int partition, long offset) {
    // TODO store maxOffset for topic/partition somewhere
    // ...
}
{% endhighlight %}

### Get the most recent offset now on Kafka
As explained above, you may choose to get from Kafka the most recent offsets and pass them to Spark. The following code uses the default Java Kafka client to do this.

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
Now you should have a list of starting offsets (one for each partition). Of course, the first time you read data this list will be empty. You may have also a list of "ending offsets" (as explained above).

You have now to convert this data into a JSON string and pass it to the Spark reader. The format of this JSON is the following:

{% highlight json %}
{
    "myTopic": {
        "0": 100,
        "1": 200,
        "2": 300
    }
}
{% endhighlight %}
where, in this example, for topic `myTopic` the offset value for partition 0 is 100, for partition 1 it is 200, and so on.

I choose to put my offsets in a Java array, where the `i`-th element of the array is the offset of partition `i`. The following method will create the required JSON string:

{% highlight java %}
String toJson(String topicName, long[] offsets) {
    List<String> partitionOffsetList = new ArrayList<>();
    int nPartitions = offsets.length;
    for (int partition = 0; partition < nPartitions; partition++) {
        long offset = offsets[partition];
        partitionOffsetList.add("\"" + partition + "\": " + offset);
    }
    return "{\"" + topicName
            + "\": {" + String.join(",", partitionOffsetList)
            + "}}";
}
{% endhighlight %}

In the following method a create a Spark reader. The two arrays `startingOffsets` and `endingOffsets` are optional. If not `null`, I use them to create the JSON to pass to the reader.

{% highlight java %}
Dataset<Row> readKafkaInput(SparkSession spark, String kafkaServers,
        String topicName, int nPartitions, long[] startingOffsets, long[] endingOffsets) {
    DataFrameReader reader = spark.read()
            .format("kafka")
            .option("kafka.bootstrap.servers", kafkaServers)
            .option("subscribe", topicName);
    if (startingOffsets != null) {
        reader = reader.option("startingOffsets", toJson(startingOffsets));
    }
    if (endingOffsets != null) {
        reader = reader.option("endingOffsets", toJson(endingOffsets));
    }
    return reader.load();
}
{% endhighlight %}

### Save the offset of the last read message
If you haven't read the `endingOffsets` directly from Kafka and passed them as option to the reader, you must retrieve them from the dataset you've just read:

{% highlight java %}
void getMaxOffsets(Dataset<Row> kafkaInput, String topicName) {
    List<Row> maxOffsets = kafkaInput.groupBy("partition")
            .agg(max("offset").alias("max_offset"))
            .collectAsList();
    for (Row row : maxOffsets) {
        int partition = row.getAs("partition");
        long maxOffset = row.getAs("max_offset");
        // TODO store maxOffset for topic/partition somewhere
        // ...
    }
}
{% endhighlight %}

## Final step
To use data returned by `readKafkaInput()`, you need to decode it to string in the following way:

{% highlight java %}
Dataset<String> strData = kafkaInput
    .selectExpr("CAST(value AS STRING)")
    .as(Encoders.STRING());
{% endhighlight %}

Now you can use it as normal. For example, if Kafka messages contain data in JSON format, you could simply do the following to parse each row and convert it to a `Row` dataset:

{% highlight java %}
Dataset<Row> inputData = spark.sqlContext().read().json(strData);
{% endhighlight %}
