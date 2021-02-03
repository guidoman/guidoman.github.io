---
layout: post
title:  "How to setup a Kafka cluster with Docker on Amazon EC2"
date:   2021-02-02 18:00:00 +0100
categories: howto bigdata
---
# How to setup a Kafka cluster with Docker on Amazon EC2
In this post I show how to setup a cluster of Kafka servers using standard .

In this example we are going to setup a Kafka cluster of two nodes, i.e., the minimum number of nodes to setup a cluster. However, instructions described in this tutorial can be generalized very easily to larger clusters.

DISCLAIMER: as soon as your EC2 instances are running, __your account will be billed__.

## Setup EC2 instances
First of all you need to setup two EC2 instances. I won't describe here the details of how to setup an AWS account (if you still don't have one) and EC2 instances.

It's maybe enough to note here that, for a production environment, you have to choose carefully the __ instance type__ (e.g., `t2.large`) and the __size and type of storage__ (e.g., `500GB st1`).

The following instructions assume that your instances are running Amazon AMI Linux. However they are still valid for any other linux distro.

### Install Docker and Docker Compose
Connect to both instances with SSH and run

```
yum install docker
systemctl enable docker.service
systemctl start docker.service
```

In this post I will also use Docker Compose to start Docker containers. So you also need to install it. See the [official instructions](https://docs.docker.com/compose/install/). Anyway it should be enough to run something like:

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.28.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

where `1.28.2` is the latest version at the time of writing this (February 2021).

### Setup public IP addresses
To access Kafka node from outside, you need to setup an _Elastic IP address_ for both instances. Take note of the following fields:
- Public IPv4 address
- Public IPv4 DNS
- Private IPv4 addresses

In this example, I will use the following fake addresses:

Node A
- Public IPv4 address: A.B.C.D
- Public IPv4 DNS ec2-A-B-C-D.eu-west-1.compute.amazonaws.com
- Private IPv4 addresses 172.1.1.1

Node B
- Public IPv4 address: E.F.G.H
- Public IPv4 DNS ec2-E-F-G-H.eu-west-1.compute.amazonaws.com
- Private IPv4 addresses 172.2.2.2

### Setup firewall
Security group instances sg-xyz

VPC dashboard

Security > Security Groups

Select tab Inbound rules

## Configure Docker containers
permessi directory

Docker compose yml 1

Docker compose yml 2


