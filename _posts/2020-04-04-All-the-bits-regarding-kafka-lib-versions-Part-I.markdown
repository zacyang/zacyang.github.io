---
layout: post
title:  "All the bits regarding kafka lib versions Part-I"
date:   2020-04-04 10:29:11
categories: Kafka
---

## Background

> I have been building kafka as a data platform for Afterpay for 14 month now. I thought it would be nice to share some of common track one could face during the impmentation

This post will cover the follow issues:

+ Kafka server and Kafka client versions
+ Schema registry/Avro version
+ Compatibility of Test support libararies 

***

## Kafka server and Kafka client versions

## Schema registry/Avro version

## Compatibility of Test support libararies 
For testing kafka specifically the events flow. e.g when you send out an event and assert what you published by consume it afterward.
There are basically 2 ways of doing it.

1. Use embed kafka from spring-kafka-test
2. Use real kafka with all the schema-registry and connectors

Depends on how you want to test it, spring-kafka-test and kakfa-client have a version locked list of compatible versions.

see [Spring-kafka-test Appendix A](https://docs.spring.io/spring-kafka/docs/2.1.x/reference/html/deps-for-11x.html) 

*Problem*
Anyway, from time to time you might bump into the following issue:

```
<XXX Spring bean creation>Exception
Caused by: java.lang.NoSuchMethodError: scala.Predef$.refArrayOps([Ljava/lang/Object;)Lscala/collection/mutable/ArrayOps;
	at kafka.cluster.EndPoint$.<init>(EndPoint.scala:32)
	at kafka.cluster.EndPoint$.<clinit>(EndPoint.scala)
	at kafka.server.Defaults$.<init>(KafkaConfig.scala:68)
	at kafka.server.Defaults$.<clinit>(KafkaConfig.scala)
	at kafka.server.KafkaConfig$.<init>(KafkaConfig.scala:781)
	at kafka.server.KafkaConfig$.<clinit>(KafkaConfig.scala)
	at kafka.utils.TestUtils$.createBrokerConfig(TestUtils.scala:234)
	at kafka.utils.TestUtils.createBrokerConfig(TestUtils.scala)
	... 86 more
```

so the root cause is basically incompatible versions, but this time is scala
note, spring-kafka-test brings its own scala version, in case of 2.1.X it's sacla 2.11.X, however, should your other depedencies have higher version of scala it would be overrided. 

For example, if you have some libary comes with scala 2.12.x as tranparent depdency then this exception is thrown.

*Solution*
Use `gradle dependecies` or mvn equivent to pin point the libary brought the higher version of scala in and then exclude it.

e.g in gradle
```
    compile(project(":someOtherProject")) {
        exclude group: "org.scala-lang"
    }
```


---------

#### Summary

