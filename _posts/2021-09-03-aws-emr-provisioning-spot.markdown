---
layout: post
title:  "How to provision an EMR cluster using On-Demand and Spot EC2 Instances"
date:   2021-09-03 16:00:00 +0200
categories: howto bigdata
---

In this tutorial I'm going to explain how to provision an [Amazon EMR](https://aws.amazon.com/emr) cluster using EC2 Spot Instances.

⚠️ __WARNING__ ⚠️: as soon as your EC2 instances are running, <u>your Amazon account will be billed</u>.

You can find the <u>full Java source code</u> in [this Gist](https://gist.github.com/guidoman/a3204d895ea28c5db651283e4e64b0bb).

## Introduction
Amazon EMR is the AWS solution for Big Data using open source tools such as [Apache Spark](https://spark.apache.org). In a matter of minutes you can provision and run clusters with up to thousands of nodes.

In general, Big Data processing requires a lot of computational resources, so running big clusters could be very expensive. EC2 Spot Instances provide an easy way to spend less money. As the [official documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html) states:

> Amazon EC2 Spot Instances are spare EC2 compute capacity in the AWS Cloud that are available to you at savings of up to 90% off compared to On-Demand prices.

However, Spot Instances can be terminated at any time. This can cause your application to crash, to loose data or get corrupted results.

An effective way to solve this problem is to mix On-Demand and Spot Instances in the same cluster.

## Cluster nodes configuration
In general, your EMR cluster can have three types of nodes: master, core and task.

From the [official documentation](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-master-core-task-nodes.html):

__Master nodes__
> The master node manages the cluster and typically runs master components of distributed applications.

__Core nodes__
> Core nodes are managed by the master node. Core nodes run the Data Node daemon to coordinate data storage as part of the Hadoop Distributed File System (HDFS). They also run the Task Tracker daemon and perform other parallel computation tasks on data that installed applications require.

__Task nodes__
> You can use task nodes to add power to perform parallel computation tasks on data, such as Hadoop MapReduce tasks and Spark executors. Task nodes don't run the Data Node daemon, nor do they store data in HDFS. 

And also:

> Because Spot Instances are often used to run task nodes, Amazon EMR has default functionality for scheduling YARN jobs so that running jobs do not fail when task nodes running on Spot Instances are terminated. Amazon EMR does this by allowing application master processes to run only on core nodes. The application master process controls running jobs and needs to stay alive for the life of the job. 

Depending on your workload and application type, Amazon suggests different configurations. You should see first [Cluster configuration guidelines and best practices](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-plan-instances-guidelines.html), and then [Instance configurations for application scenarios](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-plan-instances-guidelines.html#emr-plan-spot-scenarios).

In this tutorial, I'm going to explain how to provision a simple cluster with the following configuration:
* one master node On-Demand
* two core nodes On-Demand
* many (!) task nodes Spot

To do this, I will use the Amazon EMR __Java__ API (full source code [here](https://gist.github.com/guidoman/a3204d895ea28c5db651283e4e64b0bb)).

## Choosing instance types
Which instance types to use is an important choice that should be taken carefully. In general, it depends on the type of your application and workload, and on your budget.

Besides, when choosing Spot Instances, you should also take into account the availability in your region. Instances with low availability will be terminated more often, so many jobs will fail and your application will require more time (and more money).

The <u>full list of EC2 instance types</u> (with description) is here: [Amazon EC2 Instance Types](https://aws.amazon.com/ec2/instance-types/). However, not all instances are supported by EMR. Check the full list here: [Instance types supported by EMR](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-supported-instance-types.html).

To choose which Spot Instances to use, the following page provides detailed information about <u>types of Spot Instances and their frequency of interruption in your region</u>: [Spot Instance Advisor](https://aws.amazon.com/ec2/spot/instance-advisor/).

For additional information about prices, check [Amazon EC2 Spot Instances Pricing](https://aws.amazon.com/ec2/spot/pricing/) for current prices in your region. For Spot Instances in particular, the best type is usually a trade-off between frequency of interruption and price. The [Spot Instance pricing history](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances-history.html) is also very helpful.

## Cluster provisioning

### Configuration
First of all, I suggest to create a first EMR test cluster by hand. This will help you create what you require later, in particular a __key pair__, default __EMR roles__ and a __subnet__.

We also have to choose the EMR version to use and a S3 bucket where to store logs.

{% highlight java %}
String emrVersion = "emr-5.30.1";
String logFolder = "s3://your-bucket/emrlogs/";
String subnetID = "your-subnet-id";
String keyPairName = "your-emr-key-pair";
{% endhighlight %}

### Configuring instance types
According to what I've explained above, for this example I choose the following instances for master and core nodes:

{% highlight java %}
String masterInstanceType = "m5.xlarge";
String coreInstanceType = "i3.2xlarge";
{% endhighlight %}

As mentioned above, these will be On-Demand Instances.

When configuring Spot Instances, you have to set their weighted capacity, i.e., how much weight each instance type has when creating the Instance Fleet. In this example I choose the following instance types with their respective weights:

{% highlight java %}
SpotInstance[] taskFleetInstances = new SpotInstance[] {
    new SpotInstance("r4.2xlarge", 8),
    new SpotInstance("r4.4xlarge", 16)
};
{% endhighlight %}

Note: `SpotInstance` is just a bean I created to store this information.

Let me explain how weighted capacity works. In this example the weight of `r4.2xlarge` is 8 and that of `r4.4xlarge` is 16. First of all, even if these numbers are arbitrary (they could be for example 1 and 2), I choose here to use <u>as weight the number of vCPUs</u> of each instance type. I think this way is easy to understand and less error-prone. This is also the default when creating a cluster in the AWS control panel.

Later, when you create your "instance fleet" (see below) you choose a <u>total capacity</u>. EC2 will fill the specified total capacity with Spot Instances according to their weights. So, for example, if you request a total capacity of 32, you may get 2 `r4.2xlarge` instances and 1 `r4.4xlarge`, or 4 `r4.2xlarge`, and so on.

### Basic cluster provisioning
First of all you create an `AmazonElasticMapReduce` object using your role credentials, and then you setup <u>cluster steps</u>, i.e., "actions" that your cluster will perform. You need at least one step, your "main" step. In this example, the main step is an Apache Spark application, deployed as a JAR file on S3.

{% highlight java %}
String jarPath = "s3://your-bucket/your-job.jar";
List<String> argsList = new ArrayList<>();
argsList.add("arg1");
argsList.add("arg2");

StepConfig mainStep = new StepConfig()
    .withName("your-main-step")
    .withActionOnFailure("TERMINATE_JOB_FLOW")
    .withHadoopJarStep(new HadoopJarStepConfig().withJar(jarPath).withArgs(argsList));
{% endhighlight %}

Note also that I configure an optional debugging step.

### Task nodes
When creating Spot Instances, you are actually making a bid. In this example I choose a maximum price as a percentage of the full price of On-Demand Instances:

{% highlight java %}
double spotPricePerc = 0.4;
{% endhighlight %}

that is, at least 40% of the full price. Note that Spot prices are usually up to 80% less than the full price.

I didn't mention that <u>in a single instance fleet you can mix Spot and On-Demand instances</u>. For task nodes, you may choose to have a few On-Demand nodes always available. This depends on your application. <u>In many scenarios I think the best choice would be to allocate all task nodes as Spot Instances</u>, trying to select types with low frequency of interruption if possible.

{% highlight java %}
InstanceFleetConfig taskFleetConfig =
    new InstanceFleetConfig().withName("Task").withInstanceFleetType(InstanceFleetType.TASK)
        .withTargetSpotCapacity(tasksSpotCapacity)
        .withLaunchSpecifications(
            new InstanceFleetProvisioningSpecifications()
                .withSpotSpecification(
                    new SpotProvisioningSpecification()
                        .withAllocationStrategy("capacity-optimized")
                        .withTimeoutDurationMinutes(15)
                        .withTimeoutAction(SpotProvisioningTimeoutAction.TERMINATE_CLUSTER)))
        .withInstanceTypeConfigs(taskFleetInstanceConfigs);
if (tasksOnDemandCapacity > 0) {
    System.out.println(
        "Tasks capacity ON DEMAND = " + tasksOnDemandCapacity + ", SPOT = " + tasksSpotCapacity);
    taskFleetConfig = taskFleetConfig.withTargetOnDemandCapacity(tasksOnDemandCapacity);
}
{% endhighlight %}

I configure an instance fleet with a capacity of `tasksSpotCapacity` units of Spot Instances, and an optional capacity of `tasksOnDemandCapacity` units of On-Demand Instances.

The `capacity-optimized` allocation strategy will choose Spot Instance types according to their availability. This will lower the probability that your instances will be terminated. The default strategy is instead to choose instances by price (the more it is discounted the better).

I also choose to wait at most `15` minutes for the bidding. After this timeout without the bid to complete, cluster provisioning will fail.

### Final cluster configuration
In the last step I configure the other two node types, master and core, and I create the request object to be passed to the API:

{% highlight java %}
RunJobFlowRequest request = new RunJobFlowRequest()
    .withName("your-job-name")
    .withReleaseLabel(emrVersion).withApplications(new Application().withName("Spark"))
    .withSteps(enableDebugging, mainStep)
    .withLogUri(logFolder)
    .withServiceRole("EMR_DefaultRole")
    .withJobFlowRole("EMR_EC2_DefaultRole")
    .withInstances(
        new JobFlowInstancesConfig()
            .withEc2SubnetId(subnetID)
            .withEc2KeyName(keyPairName)
            .withKeepJobFlowAliveWhenNoSteps(false)
            .withInstanceFleets(
                new InstanceFleetConfig().withName("Master")
                    .withInstanceFleetType(InstanceFleetType.MASTER)
                    .withInstanceTypeConfigs(
                        new InstanceTypeConfig().withInstanceType(masterInstanceType))
                    .withTargetOnDemandCapacity(1),
                new InstanceFleetConfig().withName("Core")
                    .withInstanceFleetType(InstanceFleetType.CORE)
                    .withInstanceTypeConfigs(
                        new InstanceTypeConfig().withInstanceType(coreInstanceType))
                    .withTargetOnDemandCapacity(1),
                taskFleetConfig));
{% endhighlight %}

The last step is to invoke the API:

{% highlight java %}
RunJobFlowResult result = emr.runJobFlow(request);
System.out.println("The cluster ID is " + result);
{% endhighlight %}
