---
layout: post
title:  "Monitor your ML model in Apache Spark with Deequ"
date:   2021-02-26 14:00:00 +0100
categories: howto bigdata
---
Once you have built and trained your ML model, you have to maintain it in production. [This great post](https://eugeneyan.com/writing/practical-guide-to-maintaining-machine-learning/) provides a detailed overview of many options and challenges to do this effectively. 

In general, one crucial task is to __validate your incoming data__, i.e., you have to check and assure that your input data is clean, before using it to train your model.

You can always perform this task manually, by coding these checks by yourself. However, if you are using [Apache Spark](http://spark.apache.org/), you should consider to use the [Deequ library](https://github.com/awslabs/deequ).

In this post I am giving a quick overview of why you should monitor your ML pipeline, and I'm providing some __example Java code__ to implement input data validation using the Deequ library.

## Monitoring ML in production overview
Generally speaking, there are three important things to do when monitoring a ML pipeline.

1. __Validate your input data before training your model__ 

   You perform basic checks on:
   - Data format: schema, format, etc...
   - Data and features distribution: null values, duplicated values, minimum and maximum, categorical values, etc...

2. __Validate your model before deployment__
   
   By keeping a hold-out dataset, you evaluate performance of the new model on this set. If this check fails, you should break the pipeline, i.e., not deploy the model. 

3. __Rollback easily with CI/CD__
   
   By setting up a continuous integration/continuous deployment (CI/CD) pipeline, whenever a check fails, you can easily break the deployment and rollback to the previous working model.

## Validate your input data
I will describe here how you can implement the first step ("validate your input data") of the monitoring pipeline described above in Apache Spark.

### Deequ
Deequ is library developed by the [AWS Labs](https://github.com/awslabs). It is a built for Apache Spark, and implemented in Scala.

According the the [project homepage](https://github.com/awslabs/deequ):
> Deequ's purpose is to "unit-test" data to find errors early, before the data gets fed to consuming systems or machine learning algorithms.

The purpose of this library is to define data checks in a _declarative way_ and to generate and run Spark code that will perform these checks for you automatically.

### Example Java code
You can find the full list of examples [here](https://github.com/awslabs/deequ/tree/master/src/main/scala/com/amazon/deequ/examples) on GitHub.

All examples are implemented in Scala. If you are creating Spark jobs using Java, you may find the following example code useful.

<script src="https://gist.github.com/guidoman/323310a3b529db10429831398ed6c044.js"></script>

This is a super-easy "Hello, world" example, just to help you setup your Java code and start experimenting with Deequ.

To summarize:
- Create a standard Spark job in Spark. To run it, you have to `spark-submit` it.
- Create a new `SparkSession` and use it to read some dataset (a Parquet in this example).
- Create an instance of `VerificationSuite`. You will add all the checks you need to this object. In this example, I just check that some column named `data_id` is complete, i.e., it contains no null values.
- By running the `VerificationSuite` you get back a `VerificationResult` that contains the results of your checks.

## Conclusions
In this post I have explained the basic steps to monitor your ML pipeline. I have also described how you can create an Apache Spark job to inspect and check your input data using the Amazon Deequ library.