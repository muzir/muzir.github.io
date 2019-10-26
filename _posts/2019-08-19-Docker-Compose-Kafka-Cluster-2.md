---
layout: post
title:  Kafka Cluster with Docker Compose
categories: [docker, docker-compose, kafka]
fullview: false
---
We introduce to run and test kafka cluster in local in the [first part of the Kafka Cluster series](https://muzir.github.io/http://muzir.github.io/spring/docker/docker-compose/kafka/2019/08/19/Spring-Boot-Kafka-Cluster-1.html){:target="_blank"}. In this post we will take a bit closer to 
how to use this Kafka Cluster as a middleware between different applications. Basically we will have a project which produce a message and in another project consume these messages and
persist to database.

# Introduction

Currently we have two projects as consumer and producer. These two projects communicating via Kafka. Producer produce productChange message and publish it to ```Product.change``` topic.
Consumer listen same topic check with product name if it hasn't save before then save it to ```product``` table. In current configuration we have one producer and 3 consumers.
When we will create ```Product.change``` topic, we will create it with ```--replication-factor``` 3 and ```--partitions``` as 3. So the topic will be replicated in 3 kafka brokers. And in each broker
it will has 3 partitions.
 
# Configuration 

We have 3 ```docker-compose.yaml``` files. One of them for kafka cluster, another one is for producer and last one for consumer. Kafka cluster ```docker-compose.yaml``` is same with the previous post so if you 
want to check it in detail check the previous post in the [first part of the Kafka Cluster series](https://muzir.github.io/http://muzir.github.io/spring/docker/docker-compose/kafka/2019/08/19/Spring-Boot-Kafka-Cluster-1.html#configureKafka){:target="_blank"}.

## Producer Docker Compose Configuration

Basically producer have service configuration and networks configuration. In service configuration point out project main folder with ```build``` so docker know to where to find 
the project folder when it wants to build the image. ```ports``` when application started which port it will serve and in environment define the environment variables which 
will be used inside producer project. In ```networks``` instead of use the default docker network we explicitly define the network name. Because we want that all docker-compose.yamls 
which will be used in consumer and producer should be in same network. If you are running consumer, producer and kafka cluster via same docker-compose.yaml we don't need to define network
they will all use same network. But in our case we have to define the network name in each docker-compose.yaml file. 


```yaml

version: '3.7'
services:
  spring-boot-kafka-cluster-producer:
    build: ../../producer
    ports:
      - "12348:12348"
    environment:
      SPRING_PROFILES_ACTIVE: dev
      JAVA_HEAP_SIZE_MB: 2048
      ENV_KAFKA_BOOTSTRAP_SERVERS: PLAINTEXT://kafka-1:19092,PLAINTEXT://kafka-2:29092,PLAINTEXT://kafka-3:39092
      ENV_APPLICATION_CLIENTID: producer-1

networks:
  default:
    external:
      name: kafka_default

```

## Consumer Docker Compose Configuration



Also consumer project is using a postgres database if you want to check it configuration in detail, you can check [configure postgres section in spring boot docker post.](https://muzir.github.io/spring/docker/docker-compose/postgres/2019/03/24/Spring-Boot-Docker.html#configurePostgres){:target="_blank"}



# Result


You can find the all project [on Github](https://github.com/muzir/softwareLabs/tree/master/spring-boot-kafka-cluster){:target="_blank"}

# References


Happy coding :) 
