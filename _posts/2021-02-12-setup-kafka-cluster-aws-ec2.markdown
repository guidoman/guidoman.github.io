---
layout: post
title:  "How to setup a Kafka cluster with Docker on Amazon EC2"
date:   2021-02-12 16:00:00 +0100
categories: howto bigdata
---
In this example we are going to setup a Kafka cluster of two nodes using Amazon EC2 instances. However, even if some configuration details are peculiar to AWS, instructions described in this tutorial can be generalized very easily to larger clusters and to other deployment environments.

⚠️ __WARNING__ ⚠️: as soon as your EC2 instances are running, <u>your Amazon account will be billed</u>.

## Setup EC2 instances
First of all you need to setup two EC2 instances. I won't describe here the details of how to setup an AWS account (if you still don't have one) and EC2 instances, but you can find lots of resources online (see e.g., the [official documentation](https://docs.aws.amazon.com/ec2/index.html)).

In general, for a production environment, you should evaluate carefully:
- the __instance type__ (e.g., `t2.large`)
- the __size and type of storage__ (e.g., `500GB st1`)

The following instructions assume that your instances are running _Amazon AMI Linux_. However, any other linux distro can be used.

### Install Docker and Docker Compose
Connect to both instances with SSH and run:
```
yum install docker
systemctl enable docker.service
systemctl start docker.service
```

In this post I will also use Docker Compose to start Docker containers. To install it, see the [official instructions](https://docs.docker.com/compose/install/). It should be enough to run something like the following:

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.28.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

where `1.28.2` is the latest version at the time of writing this (February 2021).

### Setup public IP addresses
To access Kafka nodes from outside, you need to setup _Elastic IP addresses_ for both instances. From the instances configuration page, take note of the following fields:
- Public IPv4 address
- Public IPv4 DNS
- Private IPv4 addresses

In this example, I will use the following fake addresses:

__Node A__
- Public IPv4 address: `A.B.C.D`
- Public IPv4 DNS: `ec2-A-B-C-D.us-east-1.compute.amazonaws.com`
- Private IPv4 address: `172.1.1.1`

__Node B__
- Public IPv4 address: `E.F.G.H`
- Public IPv4 DNS: `ec2-E-F-G-H.us-east-1.compute.amazonaws.com`
- Private IPv4 address: `172.2.2.2`

### Setup VPC
You need to configure your network to allow the two nodes to talk to each other, and to allow clients to reach the Kafka cluster nodes from the outside.

First of all, find the __security group__ of your EC2 instances (e.g., `sg-1234567890`). Then go to `VPC > SECURITY > Security Groups` and select that security group ID. Now click on button _Edit inbound rules_. You have to add the following rules:

Source                          | Type        | Protocol| Port range
---------                       |--------     |-------- |--------- 
Security group `sg-1234567890`  | All traffic | All     | All
_Your client IP address(es)_    | Custom TCP  | TCP     | 2181 (for Zookeeper)
_Your client IP address(es)_    | Custom TCP  | TCP     | 9092 (for Kafka)


The first rule could be more specific, i.e., you could open only ports 2181, 2888 and 3888. In the other two rules, you have to set the IP address of the client (or clients) that will connect to the Kafka cluster.

## Configure Docker containers
### Prepare the host
In this example I am using Docker images from [Bitnami](https://github.com/bitnami), but this configuration can be adapted very easily to other images. 

Bitnami images run in user mode, in particular with user ID `1001`. To allow Kafka and Zookeeper to persist files on your host filesystem, you have to map two directories to the docker containers (Everytime a Docker container is deleted and recreated, data present in its filesystem is lost).

Assuming that you have two directories `/data/zookeeper` and `/data/kafka` where you are going to store Zookeeper and Kafka files, respectively, run the following commands:
```
sudo chown 1001.1001 /data/zookeeper/
sudo chown 1001.1001 /data/kafka/
```

These commands will change the ownerships of the two directories, granting the container processes the permissions to read and write.

### Zookeeper and Kafka setup
We have to start one instance of Zookeeper and one instance of Kafka for each cluster node.

As mentioned above, we are using __Docker Compose__ to manage containers creation. Use the two files linked below as a template to create your configuration files (one for each node):

- [Docker Compose configuration for node 1](https://gist.github.com/guidoman/33cf16e2702887fb4dfedb3f4dbde440#file-kafka-node1-docker-compose-yml)
- [Docker Compose configuration for node 2](https://gist.github.com/guidoman/33cf16e2702887fb4dfedb3f4dbde440#file-kafka-node2-docker-compose-yml)

Let's go through the relevant details of the configuration.

In the `zookeeper` section:
- Each node is assigned a unique `ZOO_SERVER_ID` number.
- In the `ZOO_SERVERS` field you have to list all addresses of your cluster, in the position matching the `ZOO_SERVER_ID` number assigned above. Private IP addresses are OK here, since the communication between servers is only internal. In this field __you have to change IP addresses to match your configuration__.
  
In the `kafka` section:
- `KAFKA_BROKER_ID` is the unique identifier of the Kafka instance.
- `KAFKA_CFG_NUM_PARTITIONS` is set here to 1 by default, but in real applications you should create topics with many partitions (see below for an explanation).
- All configurations related to _replication factor_ are set to 2 (the minimum allowed). In this case we have a 2-nodes cluster, but with larger clusters you could increase it.
- The configuration of __listeners__ is crucial. Article [Kafka Listeners – Explained](https://www.confluent.io/blog/kafka-listeners-explained/) explains in fact everything about this theme. In this example I create a single listener (named `EXTERNAL`), that is used both for internal (between cluster nodes) and external (between clients and servers) communication. However, <u>this only works here in AWS because the DNS will resolve to the public IP address on the external interface, and to the private IP address on the internal interface</u>. In this field __you have to change DNS strings to match your configuration__.

### Start containers
Now put the first Docker Compose configuration file on the first instance, and the other configuration file on the second instance. Place them in some directory, rename them to `docker-compose.yml` and from inside that directory run:
```
sudo /usr/local/bin/docker-compose up -d
```
on both machines. Now by running `docker ps` you should see both Zookeeper and Kafka containers running on both instances. If not, you will need to debug the problem by checking containers output by using `docker logs`.

### Check if it's working
If you have configured everything correctly, both nodes should now talk to each other, and you should be able to reach both of them.

To test connection with Kafka, you can use a handy command line tool named [kafkacat](https://github.com/edenhill/kafkacat). To avoid installing it, since we are already using Docker, let's execute it as a container.

From the external host/network you have configured in the `VPC` section, try to reach both node 1:
```
docker run --rm --tty confluentinc/cp-kafkacat \
  kafkacat \
  -b ec2-A-B-C-D.us-east-1.compute.amazonaws.com:9092 \
  -L 
```
and node 2:
```
docker run --rm --tty confluentinc/cp-kafkacat \
  kafkacat \
  -b ec2-E-F-G-H.us-east-1.compute.amazonaws.com:9092 \
  -L
```
Both commands should run with no error, with an output similar to the following:
```
Metadata for all topics (from broker 1: ec2-A-B-C-D.us-east-1.compute.amazonaws.com:9092:9092/1):
 2 brokers:
  broker 2 at ec2-A-B-C-D.us-east-1.compute.amazonaws.com:9092
  broker 1 at ec2-E-F-G-H.us-east-1.compute.amazonaws.com:9092
...
```
## Create topics
In the configuration, I have set `KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE` to `true` to let you create topics automatically (with a default of 1 partition). However, since <u>using multiple partitions is the only way to parallelize Kafka IO operations</u>, I suggest to create topics manually and to choose carefully how many partitions to assign to each topic.

The best replication factor and number of partitions for your topic depend on many parameters, like e.g.:
- cluster size
- node instances hardware
- network speed
- application needs

I'm not covering here the details of this choice. Let's say for example that you want to create a topic with 10 partitions with a replication factor of 2. From a client with Kafka installed (e.g., `brew install kafka` on macOS), run the following:
```
kafka-topics --create \
  --zookeeper ec2-A-B-C-D.us-east-1.compute.amazonaws.com:2181 \
  --replication-factor 2 --partitions 10 \
  --topic my-test-topic
```
Again, it is required that the client machine has access to the Zookeeper port.

If you run again the `kafkacat` commands described above, you should be able to see your new topic in the command output.

## Conclusions
We have configured a working Kafka cluster, that you can use in production for any real application.

The configuration described here will work for Amazon EC2 (in particular see the setup of _Kafka listeners_). However, it can be also easily changed to work in any other deployment environment.