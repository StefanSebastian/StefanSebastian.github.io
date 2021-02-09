---
layout: post
title:  "How to Dockerize Kafka for development"
date:   2021-02-01 19:44:40 +0200
---

This post will present a simple way to create a Kafka + Zookeeper Docker image and run multiple containers which maps configuration and logs to host directories. I had the need to dockerize Kafka while developing an application on Windows which relies on this technology. At the time of writing Kafka still has some issues on the Windows environment, like [this one](https://github.com/apache/kafka/pull/6329) which causes crashes because Windows does not allow renaming files that are locked.

My main requirements were:
1. run Kafka on my Windows development machine
2. have the possibility to create a local cluster
3. expose zookeeper and kafka endpoints between multiple instances of cluster and to the application


The format of the Dockerfile:

{% highlight docker %}
FROM openjdk:8u212-jre-alpine

ARG kafka_dir=kafka_2.12-2.1.1

ENV KAFKA_HOME=/app/kafka
ENV CONFIG_DIR=/app/config
ENV LOG_DIR=/app/logs
ENV DATA_DIR=/app/data

COPY ${kafka_dir} ${KAFKA_HOME}
EXPOSE 2181 2888 3888 9092
RUN apk add --no-cache --upgrade bash
{% endhighlight %}

This is a pretty straightforward Dockerfile. I have started from a lightweight alpine distribution that includes open jdk 8. Then I downloaded a Kafka distribution from the [official website](https://kafka.apache.org/downloads) which is copied into the image. The exposed ports are for [zookeeper cluster](https://zookeeper.apache.org/doc/r3.1.2/zookeeperStarted.html) and for Kafka clients. Directories are created to be mapped to host (logs and config) and to docker volumes (data). You can build the image using the following command:  docker image build . -t demo/kafka.

The docker-compose file:

{% highlight docker %}
version: "3.9"
services:
  zookeeper_service:
    image: demo/kafka
    ports:
      - "10.0.100.90:2181:2181"
      - "10.0.100.90:2888:2888"
      - "10.0.100.90:3888:3888"
    volumes:
      - ./config:/app/config
      - ./logs/zk:/app/logs
      - zk_data:/app/data
    command: /app/kafka/bin/zookeeper-server-start.sh /app/config/zookeeper.properties
    restart: always
  kafka_service:
    image: demo/kafka
    ports:
      - "10.0.100.90:9092:9092"
    volumes:
      - ./config:/app/config
      - ./logs/kafka:/app/logs
      - kfk_data:/app/data
    command: /app/kafka/bin/kafka-server-start.sh /app/config/server.properties
    restart: always
    extra_hosts:
      - "host:10.0.100.90"
volumes:
  zk_data:
  kfk_data:
{% endhighlight %}

The docker-compose file is a bit more verbose. The same docker image is used for two separate containers, one for zookeeper and one for kafka. This is mostly to keep in line with the Docker philosophy of having one responsability per container, but they can also be combined into one. For networking I use a virtual IP created with the [Microsoft Loopback Adapter](https://docs.microsoft.com/en-us/troubleshoot/windows-server/deployment/microsoft-loopback-adapter-rename) : 10.0.100.90 into which I map ports from inside the containers.

The zookeeper service defines the mapping for our zookeeper instance. The ports mapped are 2181 (for zk clients), 2888 and 3888 (used internally by zk for clusters, although they are not need for this example). Config and log directories are mapped from host and must be created (in this case I have created two directories in the same location from where I am running the docker-compose file). The data directory is a Docker volume. The configuration file for zookeeper, which comes from the config directory, on the host, is zookeeper.properties and contains :
{% highlight properties %}
dataDir=/app/data/zk
clientPort=2181
{% endhighlight %}

The kafka service is very similar in configuration to the previous one, except that it uses a different port and different file paths so as not to overlap. An extra configuration is the extra_hosts option which adds a mapping to the address on which zookeeper is also running so it can be accessed from within the container. The configuration file for kafka is server.properties and the parts which you need to modify are presented below:
{% highlight properties %}
listeners=PLAINTEXT://:9092
log.dirs=/app/data/kafka
zookeeper.connect=host:2181
{% endhighlight %}

Now that the configuration is complete you can run the service using the commands from docker-compose: docker-compose up (or with -d flag to run in detached mode).