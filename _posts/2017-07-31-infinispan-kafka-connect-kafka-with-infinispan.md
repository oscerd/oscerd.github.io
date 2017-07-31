---
layout: post
title: Introducing Infinispan-Kafka, connect your Kafka cluster with Infinispan
---

In the last couple of months I worked on a side project: Infinispan-Kafka. This project is based on the [Kafka Connect](https://kafka.apache.org/documentation/#connect) tool: Kafka Connect is a tool for streaming data between Apache Kafka and other systems. There are two sides where data can be streamed: from Kafka to a different system (Sink Connector) and from a different system to Kafka (Source Connector). The Infinispan Kafka project implements only the Sink Connector (for the moment).

### Basic Idea

In Infinispan, through the [Protostream](https://github.com/infinispan/protostream/) and [Infinispan Remote Querying](http://infinispan.org/docs/stable/user_guide/user_guide.html#query.remote) an end-user is able to remote querying Infinispan in a language-neutral manner, by using Protobuf. The Infinispan client must be configured to use a dedicated marshaller, ProtoStreamMarshaller and this one will use the ProtoStream library for encoding objects. ProtoStream library has to be instructed on how to marshall the message types, here is the [documentation](http://infinispan.org/docs/stable/user_guide/user_guide.html#storing_protobuf_encoded_entities).

So the idea behind the Infinispan-Kafka project is to use the Protostream library and the ProtoStreamMarshaller to define what kind of data can be saved in Infinispan cache from Kafka and let Infinispan manage the marshalling/storing, creating a contract between Infinispan and Kafka.

The Infinispan-Kafka connector use the following configuration properties (for the moment):


| Name                                 | Description                                                                                                            | Type    | Default      | Importance |
|--------------------------------------|------------------------------------------------------------------------------------------------------------------------|-------- |------------- |------------|
| infinispan.connection.hosts          | List of comma separated Infinispan hosts                                                                               | string  | localhost    | high       |
| infinispan.connection.hotrod.port    | Infinispan Hot Rod port                                                                                                | int     | 11222        | high       |
| infinispan.connection.cache.name     | Infinispan Cache name of use                                                                                           | String  | default      | medium     |
| infinispan.use.proto                 | If true, the Remote Cache Manager will be configured to use protostream schemas                                        | boolean | false        | medium     |
| infinispan.proto.marshaller.class    | If infinispan.use.proto is true, this option has to contain an annotated protostream class to be used                  | Class   | String.class | medium     |
| infinispan.cache.force.return.values | By default, previously existing values for Map operations are not returned, if set to true the values will be returned | boolean | false        | low        |

an example of annotated protostream class is the following


```java
package org.infinispan.kafka;

import java.io.Serializable;

import org.infinispan.protostream.annotations.ProtoDoc;
import org.infinispan.protostream.annotations.ProtoField;

@ProtoDoc("@Indexed")
public class Author implements Serializable {

   private String name;

   @ProtoField(number = 1, required = true)
   public String getName() {
      return name;
   }

   public void setName(String name) {
      this.name = name;
   }

   @Override
   public String toString() {
      return "Author [name=" + name + "]";
   }
}
```

### A running example

Lets see how the connector works in a real example. You'll need:

- Infinispan server 9.1.0.Final
- Kafka 0.11.0.0 running

and these three projects 

- https://github.com/oscerd/infinispan-kafka-demo
- https://github.com/oscerd/infinispan-kafka-producer
- https://github.com/oscerd/camel-infinispan-kafka-demo

But, since infinispan-kafka is not yet released, you'll need to build the project to have it available in your Local Maven repository.

So fork or clone the [Infinispan-Kafka](https://github.com/infinispan/infinispan-kafka) project and run 

```bash
> mvn clean install
```

Now lets see what is inside the different projects.

### The Infinispan-kafka demo

In the repository you'll find a [properties file](https://github.com/oscerd/infinispan-kafka-demo/tree/master/config) for the Infinispan-Kafka sink connector and a Protostream annotated class named [Author](https://github.com/oscerd/infinispan-kafka-demo/tree/master/src/main).

The configuration is the following:

```text
name=InfinispanSinkConnector
topics=test
tasks.max=1
connector.class=org.infinispan.kafka.InfinispanSinkConnector
key.converter=org.apache.kafka.connect.storage.StringConverter
value.converter=org.apache.kafka.connect.storage.StringConverter

infinispan.connection.hosts=127.0.0.1
infinispan.connection.hotrod.port=11222
infinispan.connection.cache.name=default
infinispan.cache.force.return.values=true
infinispan.use.proto=true
infinispan.proto.marshaller.class=org.infinispan.kafka.Author
```

We will use the topic test from our Kafka cluster


### The Infinispan-kafka producer

In this project we define a [Simple Producer](https://github.com/oscerd/infinispan-kafka-producer/blob/master/src/main/java/com/github/oscerd/SimpleProducer.java) that will send record to our Kafka cluster in the topic test. We will send five record of Author.

### The Camel-infispan-kafka demo

In this project we define a [Standalone camel route](https://github.com/oscerd/camel-infinispan-kafka-demo/blob/master/src/main/java/com/github/oscerd/camel/infinispan/kafka/demo/CamelInfinispanRoute.java). This route will run every 10 seconds and it will query the Infinispan cache with a Remote Query defined [here](https://github.com/oscerd/camel-infinispan-kafka-demo/blob/master/src/main/java/com/github/oscerd/camel/infinispan/kafka/demo/InfinispanKafkaQueryBuilder.java).

### Run the example now!

Now that we have a full view of this demo we can run it!

Let's start from the servers.

First we need to start the Infinispan server, in this case in standalone mode.

```bash
>infinispan-server-9.1.0.Final/bin$ ./standalone.sh 
=========================================================================

  JBoss Bootstrap Environment

  JBOSS_HOME: /home/oscerd/playground/infinispan-server-9.1.0.Final

  JAVA: /usr/lib/jvm/jdk1.8.0_65//bin/java

  JAVA_OPTS:  -server  -server -Xms64m -Xmx512m -Djava.net.preferIPv4Stack=true -Djboss.modules.system.pkgs=org.jboss.byteman -Djava.awt.headless=true

=========================================================================

11:36:28,886 INFO  [org.jboss.modules] (main) JBoss Modules version 1.5.2.Final
11:36:29,046 INFO  [org.jboss.msc] (main) JBoss MSC version 1.2.6.Final
11:36:29,104 INFO  [org.jboss.as] (MSC service thread 1-6) WFLYSRV0049: Infinispan Server 9.1.0.Final (WildFly Core 2.2.0.Final) starting
11:36:29,788 INFO  [org.jboss.as.server] (Controller Boot Thread) WFLYSRV0039: Creating http management service using socket-binding (management-http)
11:36:29,801 INFO  [org.xnio] (MSC service thread 1-6) XNIO version 3.4.0.Final
11:36:29,806 INFO  [org.xnio.nio] (MSC service thread 1-6) XNIO NIO Implementation Version 3.4.0.Final
11:36:29,823 INFO  [org.jboss.as.clustering.infinispan] (ServerService Thread Pool -- 20) Activating Infinispan subsystem.
11:36:29,823 INFO  [org.wildfly.extension.io] (ServerService Thread Pool -- 19) WFLYIO001: Worker 'default' has auto-configured to 16 core threads with 128 task threads based on your 8 available processors
11:36:29,835 INFO  [org.jboss.as.connector.subsystems.datasources] (ServerService Thread Pool -- 18) WFLYJCA0004: Deploying JDBC-compliant driver class org.h2.Driver (version 1.3)
11:36:29,843 INFO  [org.jboss.as.naming] (ServerService Thread Pool -- 25) WFLYNAM0001: Activating Naming Subsystem
11:36:29,848 WARN  [org.jboss.as.txn] (ServerService Thread Pool -- 29) WFLYTX0013: Node identifier property is set to the default value. Please make sure it is unique.
11:36:29,849 INFO  [org.jboss.as.connector] (MSC service thread 1-8) WFLYJCA0009: Starting JCA Subsystem (WildFly/IronJacamar 1.3.4.Final)
11:36:29,851 INFO  [org.jboss.as.connector.deployers.jdbc] (MSC service thread 1-3) WFLYJCA0018: Started Driver service with driver-name = h2
11:36:29,860 INFO  [org.jboss.as.security] (ServerService Thread Pool -- 27) WFLYSEC0002: Activating Security Subsystem
11:36:29,861 INFO  [org.jboss.remoting] (MSC service thread 1-6) JBoss Remoting version 4.0.21.Final
11:36:29,872 INFO  [org.jboss.as.security] (MSC service thread 1-2) WFLYSEC0001: Current PicketBox version=4.9.6.Final
11:36:29,876 INFO  [org.jboss.as.naming] (MSC service thread 1-7) WFLYNAM0003: Starting Naming Service
11:36:30,079 INFO  [org.jboss.as.server.deployment.scanner] (MSC service thread 1-4) WFLYDS0013: Started FileSystemDeploymentService for directory /home/oscerd/playground/infinispan-server-9.1.0.Final/standalone/deployments
11:36:30,394 INFO  [org.infinispan.factories.GlobalComponentRegistry] (MSC service thread 1-5) ISPN000128: Infinispan version: Infinispan 'Bastille' 9.1.0.Final
11:36:30,535 INFO  [org.jboss.as.connector.subsystems.datasources] (MSC service thread 1-8) WFLYJCA0001: Bound data source [java:jboss/datasources/ExampleDS]
11:36:30,745 INFO  [org.jboss.as.clustering.infinispan] (MSC service thread 1-6) DGISPN0001: Started default cache from local container
11:36:30,746 INFO  [org.jboss.as.clustering.infinispan] (MSC service thread 1-7) DGISPN0001: Started namedCache cache from local container
11:36:30,763 INFO  [org.infinispan.server.endpoint] (MSC service thread 1-3) DGENDPT10000: HotRodServer starting
11:36:30,766 INFO  [org.infinispan.server.endpoint] (MSC service thread 1-3) DGENDPT10001: HotRodServer listening on 127.0.0.1:11222
11:36:30,770 INFO  [org.infinispan.server.endpoint] (MSC service thread 1-5) DGENDPT10000: REST starting
11:36:31,072 INFO  [org.infinispan.server.endpoint] (MSC service thread 1-5) DGENDPT10002: REST listening on 127.0.0.1:8080 (mapped to rest)
11:36:31,258 INFO  [org.jboss.as] (Controller Boot Thread) WFLYSRV0060: Http management interface listening on http://127.0.0.1:9990/management
11:36:31,259 INFO  [org.jboss.as] (Controller Boot Thread) WFLYSRV0051: Admin console listening on http://127.0.0.1:9990
11:36:31,259 INFO  [org.jboss.as] (Controller Boot Thread) WFLYSRV0025: Infinispan Server 9.1.0.Final (WildFly Core 2.2.0.Final) started in 2602ms - Started 152 of 163 services (49 services are lazy, passive or on-demand)
```

Next we need to start our Kafka server, as always we'll need to start zookeeper first and then the server.

```bash
>kafka_2.12-0.11.0.0$ bin/zookeeper-server-start.sh config/zookeeper.properties
[2017-07-31 11:38:55,401] INFO Reading configuration from: config/zookeeper.properties (org.apache.zookeeper.server.quorum.QuorumPeerConfig)
[2017-07-31 11:38:55,404] INFO autopurge.snapRetainCount set to 3 (org.apache.zookeeper.server.DatadirCleanupManager)
[2017-07-31 11:38:55,404] INFO autopurge.purgeInterval set to 0 (org.apache.zookeeper.server.DatadirCleanupManager)
[2017-07-31 11:38:55,404] INFO Purge task is not scheduled. (org.apache.zookeeper.server.DatadirCleanupManager)
[2017-07-31 11:38:55,404] WARN Either no config or no quorum defined in config, running  in standalone mode (org.apache.zookeeper.server.quorum.QuorumPeerMain)
[2017-07-31 11:38:55,417] INFO Reading configuration from: config/zookeeper.properties (org.apache.zookeeper.server.quorum.QuorumPeerConfig)
[2017-07-31 11:38:55,417] INFO Starting server (org.apache.zookeeper.server.ZooKeeperServerMain)
[2017-07-31 11:38:55,728] INFO Server environment:zookeeper.version=3.4.10-39d3a4f269333c922ed3db283be479f9deacaa0f, built on 03/23/2017 10:13 GMT (org.apache.zookeeper.server.ZooKeeperServer)
[2017-07-31 11:38:55,728] INFO Server environment:host.name=ghost (org.apache.zookeeper.server.ZooKeeperServer)
[2017-07-31 11:38:55,728] INFO Server environment:java.version=1.8.0_65 (org.apache.zookeeper.server.ZooKeeperServer)
.
.
.
.
```

And the server

```bash
>kafka_2.12-0.11.0.0$ bin/kafka-server-start.sh config/server.properties
[2017-07-31 11:40:31,244] INFO KafkaConfig values: 
	advertised.host.name = null
	advertised.listeners = null
	advertised.port = null
	alter.config.policy.class.name = null
	authorizer.class.name = 
	auto.create.topics.enable = true
	auto.leader.rebalance.enable = true
	background.threads = 10
	broker.id = 0
	broker.id.generation.enable = true
	broker.rack = null
	compression.type = producer
	connections.max.idle.ms = 600000
	controlled.shutdown.enable = true
	controlled.shutdown.max.retries = 3
	controlled.shutdown.retry.backoff.ms = 5000
	controller.socket.timeout.ms = 30000
	create.topic.policy.class.name = null
	default.replication.factor = 1
.
.
.
.
.
.
.
```

Next we'll need to create a new topic named test.

```bash
>kafka_2.12-0.11.0.0$ bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
Created topic "test".
```

The servers are now up and running. At this point we need to start the Infinispan-Kafka connector demo.

```
>infinispan-kafka-demo$ mvn clean package
[INFO] Scanning for projects...
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] Building infinispan-kafka-demo 0.0.1-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO] 
[INFO] --- maven-clean-plugin:2.5:clean (default-clean) @ infinispan-kafka-demo ---
[INFO] Deleting /home/oscerd/workspace/miscellanea/infinispan-kafka-demo/target
[INFO] 
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ infinispan-kafka-demo ---
[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!
[INFO] Copying 0 resource
[INFO] 
[INFO] --- maven-compiler-plugin:2.5.1:compile (default-compile) @ infinispan-kafka-demo ---
[WARNING] File encoding has not been set, using platform encoding UTF-8, i.e. build is platform dependent!
[INFO] Compiling 1 source file to /home/oscerd/workspace/miscellanea/infinispan-kafka-demo/target/classes
[INFO] 
[INFO] --- maven-resources-plugin:2.6:testResources (default-testResources) @ infinispan-kafka-demo ---
[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!
[INFO] skip non existing resourceDirectory /home/oscerd/workspace/miscellanea/infinispan-kafka-demo/src/test/resources
[INFO] 
[INFO] --- maven-compiler-plugin:2.5.1:testCompile (default-testCompile) @ infinispan-kafka-demo ---
[INFO] No sources to compile
[INFO] 
[INFO] --- maven-surefire-plugin:2.12.4:test (default-test) @ infinispan-kafka-demo ---
[INFO] No tests to run.
[INFO] 
[INFO] --- maven-jar-plugin:2.4:jar (default-jar) @ infinispan-kafka-demo ---
[INFO] Building jar: /home/oscerd/workspace/miscellanea/infinispan-kafka-demo/target/infinispan-kafka-demo-0.0.1-SNAPSHOT.jar
[INFO] 
[INFO] --- maven-assembly-plugin:2.5.3:single (make-assembly) @ infinispan-kafka-demo ---
[INFO] Reading assembly descriptor: src/main/assembly/package.xml
[WARNING] The following patterns were never triggered in this artifact exclusion filter:
o  'org.apache.kafka:connect-api'

[INFO] Copying files to /home/oscerd/workspace/miscellanea/infinispan-kafka-demo/target/infinispan-kafka-demo-0.0.1-SNAPSHOT-package
[WARNING] Assembly file: /home/oscerd/workspace/miscellanea/infinispan-kafka-demo/target/infinispan-kafka-demo-0.0.1-SNAPSHOT-package is not a regular file (it may be a directory). It cannot be attached to the project build for installation or deployment.
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 2.799 s
[INFO] Finished at: 2017-07-31T11:50:11+02:00
[INFO] Final Memory: 30M/530M
[INFO] ------------------------------------------------------------------------
>infinispan-kafka-demo$ export CLASSPATH="$(find target/ -type f -name '*.jar'| grep '\-package' | tr '\n' ':')"
```

We can start the connector now with the configuration of the demo project in this way:

```bash
>infinispan-kafka-demo$ kafka_2.12-0.11.0.0/bin/connect-standalone.sh kafka_2.12-0.11.0.0/config/connect-standalone.properties config/InfinispanSinkConnector.properties 
[2017-07-31 11:58:37,893] INFO Registered loader: sun.misc.Launcher$AppClassLoader@18b4aac2 (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:199)
[2017-07-31 11:58:37,895] INFO Added plugin 'org.apache.kafka.connect.file.FileStreamSinkConnector' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,895] INFO Added plugin 'org.apache.kafka.connect.file.FileStreamSourceConnector' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,896] INFO Added plugin 'org.apache.kafka.connect.tools.VerifiableSinkConnector' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,896] INFO Added plugin 'org.apache.kafka.connect.tools.SchemaSourceConnector' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,896] INFO Added plugin 'org.apache.kafka.connect.tools.MockSinkConnector' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,896] INFO Added plugin 'org.apache.kafka.connect.tools.VerifiableSourceConnector' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,896] INFO Added plugin 'org.apache.kafka.connect.tools.MockConnector' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,896] INFO Added plugin 'org.infinispan.kafka.InfinispanSinkConnector' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,896] INFO Added plugin 'org.apache.kafka.connect.tools.MockSourceConnector' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,896] INFO Added plugin 'org.apache.kafka.connect.json.JsonConverter' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,896] INFO Added plugin 'org.apache.kafka.connect.storage.StringConverter' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,896] INFO Added plugin 'org.apache.kafka.connect.converters.ByteArrayConverter' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,896] INFO Added plugin 'org.apache.kafka.connect.transforms.MaskField$Key' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,896] INFO Added plugin 'org.apache.kafka.connect.transforms.ExtractField$Key' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,896] INFO Added plugin 'org.apache.kafka.connect.transforms.SetSchemaMetadata$Value' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,897] INFO Added plugin 'org.apache.kafka.connect.transforms.Flatten$Key' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,897] INFO Added plugin 'org.apache.kafka.connect.transforms.Cast$Key' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,897] INFO Added plugin 'org.apache.kafka.connect.transforms.ExtractField$Value' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,898] INFO Added plugin 'org.apache.kafka.connect.transforms.ReplaceField$Key' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,898] INFO Added plugin 'org.apache.kafka.connect.transforms.MaskField$Value' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,898] INFO Added plugin 'org.apache.kafka.connect.transforms.Flatten$Value' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,898] INFO Added plugin 'org.apache.kafka.connect.transforms.HoistField$Key' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,898] INFO Added plugin 'org.apache.kafka.connect.transforms.ReplaceField$Value' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,898] INFO Added plugin 'org.apache.kafka.connect.transforms.TimestampConverter$Value' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,898] INFO Added plugin 'org.apache.kafka.connect.transforms.Cast$Value' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,898] INFO Added plugin 'org.apache.kafka.connect.transforms.TimestampConverter$Key' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,898] INFO Added plugin 'org.apache.kafka.connect.transforms.InsertField$Key' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,898] INFO Added plugin 'org.apache.kafka.connect.transforms.ValueToKey' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,898] INFO Added plugin 'org.apache.kafka.connect.transforms.InsertField$Value' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,898] INFO Added plugin 'org.apache.kafka.connect.transforms.RegexRouter' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,898] INFO Added plugin 'org.apache.kafka.connect.transforms.HoistField$Value' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,898] INFO Added plugin 'org.apache.kafka.connect.transforms.SetSchemaMetadata$Key' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,898] INFO Added plugin 'org.apache.kafka.connect.transforms.TimestampRouter' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:132)
[2017-07-31 11:58:37,899] INFO Added aliases 'FileStreamSinkConnector' and 'FileStreamSink' to plugin 'org.apache.kafka.connect.file.FileStreamSinkConnector' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:293)
[2017-07-31 11:58:37,899] INFO Added aliases 'FileStreamSourceConnector' and 'FileStreamSource' to plugin 'org.apache.kafka.connect.file.FileStreamSourceConnector' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:293)
[2017-07-31 11:58:37,899] INFO Added aliases 'MockConnector' and 'Mock' to plugin 'org.apache.kafka.connect.tools.MockConnector' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:293)
[2017-07-31 11:58:37,899] INFO Added aliases 'MockSinkConnector' and 'MockSink' to plugin 'org.apache.kafka.connect.tools.MockSinkConnector' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:293)
[2017-07-31 11:58:37,899] INFO Added aliases 'MockSourceConnector' and 'MockSource' to plugin 'org.apache.kafka.connect.tools.MockSourceConnector' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:293)
[2017-07-31 11:58:37,900] INFO Added aliases 'SchemaSourceConnector' and 'SchemaSource' to plugin 'org.apache.kafka.connect.tools.SchemaSourceConnector' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:293)
[2017-07-31 11:58:37,900] INFO Added aliases 'VerifiableSinkConnector' and 'VerifiableSink' to plugin 'org.apache.kafka.connect.tools.VerifiableSinkConnector' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:293)
[2017-07-31 11:58:37,900] INFO Added aliases 'VerifiableSourceConnector' and 'VerifiableSource' to plugin 'org.apache.kafka.connect.tools.VerifiableSourceConnector' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:293)
[2017-07-31 11:58:37,900] INFO Added aliases 'InfinispanSinkConnector' and 'InfinispanSink' to plugin 'org.infinispan.kafka.InfinispanSinkConnector' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:293)
[2017-07-31 11:58:37,900] INFO Added aliases 'ByteArrayConverter' and 'ByteArray' to plugin 'org.apache.kafka.connect.converters.ByteArrayConverter' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:293)
[2017-07-31 11:58:37,900] INFO Added aliases 'JsonConverter' and 'Json' to plugin 'org.apache.kafka.connect.json.JsonConverter' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:293)
[2017-07-31 11:58:37,900] INFO Added aliases 'StringConverter' and 'String' to plugin 'org.apache.kafka.connect.storage.StringConverter' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:293)
[2017-07-31 11:58:37,900] INFO Added alias 'RegexRouter' to plugin 'org.apache.kafka.connect.transforms.RegexRouter' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:290)
[2017-07-31 11:58:37,901] INFO Added alias 'TimestampRouter' to plugin 'org.apache.kafka.connect.transforms.TimestampRouter' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:290)
[2017-07-31 11:58:37,901] INFO Added alias 'ValueToKey' to plugin 'org.apache.kafka.connect.transforms.ValueToKey' (org.apache.kafka.connect.runtime.isolation.DelegatingClassLoader:290)
[2017-07-31 11:58:37,912] INFO StandaloneConfig values: 
	access.control.allow.methods = 
	access.control.allow.origin = 
	bootstrap.servers = [localhost:9092]
	internal.key.converter = class org.apache.kafka.connect.json.JsonConverter
	internal.value.converter = class org.apache.kafka.connect.json.JsonConverter
	key.converter = class org.apache.kafka.connect.json.JsonConverter
	offset.flush.interval.ms = 10000
	offset.flush.timeout.ms = 5000
	offset.storage.file.filename = /tmp/connect.offsets
	plugin.path = null
	rest.advertised.host.name = null
	rest.advertised.port = null
	rest.host.name = null
	rest.port = 8083
	task.shutdown.graceful.timeout.ms = 5000
	value.converter = class org.apache.kafka.connect.json.JsonConverter
 (org.apache.kafka.connect.runtime.standalone.StandaloneConfig:223)
[2017-07-31 11:58:38,004] INFO Logging initialized @2593ms (org.eclipse.jetty.util.log:186)
[2017-07-31 11:58:38,153] INFO Kafka Connect starting (org.apache.kafka.connect.runtime.Connect:49)
[2017-07-31 11:58:38,153] INFO Herder starting (org.apache.kafka.connect.runtime.standalone.StandaloneHerder:70)
[2017-07-31 11:58:38,153] INFO Worker starting (org.apache.kafka.connect.runtime.Worker:144)
[2017-07-31 11:58:38,154] INFO Starting FileOffsetBackingStore with file /tmp/connect.offsets (org.apache.kafka.connect.storage.FileOffsetBackingStore:59)
[2017-07-31 11:58:38,155] INFO Worker started (org.apache.kafka.connect.runtime.Worker:149)
[2017-07-31 11:58:38,155] INFO Herder started (org.apache.kafka.connect.runtime.standalone.StandaloneHerder:72)
[2017-07-31 11:58:38,155] INFO Starting REST server (org.apache.kafka.connect.runtime.rest.RestServer:98)
[2017-07-31 11:58:38,218] INFO jetty-9.2.15.v20160210 (org.eclipse.jetty.server.Server:327)
Jul 31, 2017 11:58:38 AM org.glassfish.jersey.internal.Errors logErrors
WARNING: The following warnings have been detected: WARNING: The (sub)resource method createConnector in org.apache.kafka.connect.runtime.rest.resources.ConnectorsResource contains empty path annotation.
WARNING: The (sub)resource method listConnectors in org.apache.kafka.connect.runtime.rest.resources.ConnectorsResource contains empty path annotation.
WARNING: The (sub)resource method listConnectorPlugins in org.apache.kafka.connect.runtime.rest.resources.ConnectorPluginsResource contains empty path annotation.
WARNING: The (sub)resource method serverInfo in org.apache.kafka.connect.runtime.rest.resources.RootResource contains empty path annotation.

[2017-07-31 11:58:38,561] INFO Started o.e.j.s.ServletContextHandler@5dab9949{/,null,AVAILABLE} (org.eclipse.jetty.server.handler.ContextHandler:744)
[2017-07-31 11:58:38,568] INFO Started ServerConnector@740c2895{HTTP/1.1}{0.0.0.0:8083} (org.eclipse.jetty.server.ServerConnector:266)
[2017-07-31 11:58:38,569] INFO Started @3158ms (org.eclipse.jetty.server.Server:379)
[2017-07-31 11:58:38,569] INFO REST server listening at http://192.168.1.10:8083/, advertising URL http://192.168.1.10:8083/ (org.apache.kafka.connect.runtime.rest.RestServer:150)
[2017-07-31 11:58:38,569] INFO Kafka Connect started (org.apache.kafka.connect.runtime.Connect:55)
[2017-07-31 11:58:38,575] INFO ConnectorConfig values: 
	connector.class = org.infinispan.kafka.InfinispanSinkConnector
	key.converter = class org.apache.kafka.connect.storage.StringConverter
	name = InfinispanSinkConnector
	tasks.max = 1
	transforms = null
	value.converter = class org.apache.kafka.connect.storage.StringConverter
 (org.apache.kafka.connect.runtime.ConnectorConfig:223)
[2017-07-31 11:58:38,576] INFO EnrichedConnectorConfig values: 
	connector.class = org.infinispan.kafka.InfinispanSinkConnector
	key.converter = class org.apache.kafka.connect.storage.StringConverter
	name = InfinispanSinkConnector
	tasks.max = 1
	transforms = null
	value.converter = class org.apache.kafka.connect.storage.StringConverter
 (org.apache.kafka.connect.runtime.ConnectorConfig$EnrichedConnectorConfig:223)
[2017-07-31 11:58:38,576] INFO Creating connector InfinispanSinkConnector of type org.infinispan.kafka.InfinispanSinkConnector (org.apache.kafka.connect.runtime.Worker:204)
[2017-07-31 11:58:38,576] INFO Instantiated connector InfinispanSinkConnector with version 0.0.1-SNAPSHOT of type class org.infinispan.kafka.InfinispanSinkConnector (org.apache.kafka.connect.runtime.Worker:207)
[2017-07-31 11:58:38,577] INFO Finished creating connector InfinispanSinkConnector (org.apache.kafka.connect.runtime.Worker:225)
[2017-07-31 11:58:38,578] INFO SinkConnectorConfig values: 
	connector.class = org.infinispan.kafka.InfinispanSinkConnector
	key.converter = class org.apache.kafka.connect.storage.StringConverter
	name = InfinispanSinkConnector
	tasks.max = 1
	topics = [test]
	transforms = null
	value.converter = class org.apache.kafka.connect.storage.StringConverter
 (org.apache.kafka.connect.runtime.SinkConnectorConfig:223)
[2017-07-31 11:58:38,578] INFO EnrichedConnectorConfig values: 
	connector.class = org.infinispan.kafka.InfinispanSinkConnector
	key.converter = class org.apache.kafka.connect.storage.StringConverter
	name = InfinispanSinkConnector
	tasks.max = 1
	topics = [test]
	transforms = null
	value.converter = class org.apache.kafka.connect.storage.StringConverter
 (org.apache.kafka.connect.runtime.ConnectorConfig$EnrichedConnectorConfig:223)
[2017-07-31 11:58:38,578] INFO Setting task configurations for 1 workers. (org.infinispan.kafka.InfinispanSinkConnector:50)
[2017-07-31 11:58:38,579] INFO Creating task InfinispanSinkConnector-0 (org.apache.kafka.connect.runtime.Worker:358)
[2017-07-31 11:58:38,579] INFO ConnectorConfig values: 
	connector.class = org.infinispan.kafka.InfinispanSinkConnector
	key.converter = class org.apache.kafka.connect.storage.StringConverter
	name = InfinispanSinkConnector
	tasks.max = 1
	transforms = null
	value.converter = class org.apache.kafka.connect.storage.StringConverter
 (org.apache.kafka.connect.runtime.ConnectorConfig:223)
[2017-07-31 11:58:38,579] INFO EnrichedConnectorConfig values: 
	connector.class = org.infinispan.kafka.InfinispanSinkConnector
	key.converter = class org.apache.kafka.connect.storage.StringConverter
	name = InfinispanSinkConnector
	tasks.max = 1
	transforms = null
	value.converter = class org.apache.kafka.connect.storage.StringConverter
 (org.apache.kafka.connect.runtime.ConnectorConfig$EnrichedConnectorConfig:223)
[2017-07-31 11:58:38,580] INFO TaskConfig values: 
	task.class = class org.infinispan.kafka.InfinispanSinkTask
 (org.apache.kafka.connect.runtime.TaskConfig:223)
[2017-07-31 11:58:38,580] INFO Instantiated task InfinispanSinkConnector-0 with version 0.0.1-SNAPSHOT of type org.infinispan.kafka.InfinispanSinkTask (org.apache.kafka.connect.runtime.Worker:373)
[2017-07-31 11:58:38,587] INFO ConsumerConfig values: 
	auto.commit.interval.ms = 5000
	auto.offset.reset = earliest
	bootstrap.servers = [localhost:9092]
	check.crcs = true
	client.id = 
	connections.max.idle.ms = 540000
	enable.auto.commit = false
	exclude.internal.topics = true
	fetch.max.bytes = 52428800
	fetch.max.wait.ms = 500
	fetch.min.bytes = 1
	group.id = connect-InfinispanSinkConnector
	heartbeat.interval.ms = 3000
	interceptor.classes = null
	internal.leave.group.on.close = true
	isolation.level = read_uncommitted
	key.deserializer = class org.apache.kafka.common.serialization.ByteArrayDeserializer
	max.partition.fetch.bytes = 1048576
	max.poll.interval.ms = 300000
	max.poll.records = 500
	metadata.max.age.ms = 300000
	metric.reporters = []
	metrics.num.samples = 2
	metrics.recording.level = INFO
	metrics.sample.window.ms = 30000
	partition.assignment.strategy = [class org.apache.kafka.clients.consumer.RangeAssignor]
	receive.buffer.bytes = 65536
	reconnect.backoff.max.ms = 1000
	reconnect.backoff.ms = 50
	request.timeout.ms = 305000
	retry.backoff.ms = 100
	sasl.jaas.config = null
	sasl.kerberos.kinit.cmd = /usr/bin/kinit
	sasl.kerberos.min.time.before.relogin = 60000
	sasl.kerberos.service.name = null
	sasl.kerberos.ticket.renew.jitter = 0.05
	sasl.kerberos.ticket.renew.window.factor = 0.8
	sasl.mechanism = GSSAPI
	security.protocol = PLAINTEXT
	send.buffer.bytes = 131072
	session.timeout.ms = 10000
	ssl.cipher.suites = null
	ssl.enabled.protocols = [TLSv1.2, TLSv1.1, TLSv1]
	ssl.endpoint.identification.algorithm = null
	ssl.key.password = null
	ssl.keymanager.algorithm = SunX509
	ssl.keystore.location = null
	ssl.keystore.password = null
	ssl.keystore.type = JKS
	ssl.protocol = TLS
	ssl.provider = null
	ssl.secure.random.implementation = null
	ssl.trustmanager.algorithm = PKIX
	ssl.truststore.location = null
	ssl.truststore.password = null
	ssl.truststore.type = JKS
	value.deserializer = class org.apache.kafka.common.serialization.ByteArrayDeserializer
 (org.apache.kafka.clients.consumer.ConsumerConfig:223)
[2017-07-31 11:58:38,631] INFO Kafka version : 0.11.0.0 (org.apache.kafka.common.utils.AppInfoParser:83)
[2017-07-31 11:58:38,631] INFO Kafka commitId : cb8625948210849f (org.apache.kafka.common.utils.AppInfoParser:84)
[2017-07-31 11:58:38,633] INFO Created connector InfinispanSinkConnector (org.apache.kafka.connect.cli.ConnectStandalone:91)
[2017-07-31 11:58:38,634] INFO InfinispanSinkConnectorConfig values: 
	infinispan.cache.force.return.values = true
	infinispan.connection.cache.name = default
	infinispan.connection.hosts = 127.0.0.1
	infinispan.connection.hotrod.port = 11222
	infinispan.proto.marshaller.class = class org.infinispan.kafka.Author
	infinispan.use.proto = true
 (org.infinispan.kafka.InfinispanSinkConnectorConfig:223)
[2017-07-31 11:58:38,661] INFO Adding protostream (org.infinispan.kafka.InfinispanSinkTask:90)
[2017-07-31 11:58:38,749] INFO ISPN004021: Infinispan version: 9.1.0.Final (org.infinispan.client.hotrod.RemoteCacheManager:212)
[2017-07-31 11:58:38,908] INFO Sink task WorkerSinkTask{id=InfinispanSinkConnector-0} finished initialization and start (org.apache.kafka.connect.runtime.WorkerSinkTask:233)
[2017-07-31 11:58:38,996] INFO Discovered coordinator ghost:9092 (id: 2147483647 rack: null) for group connect-InfinispanSinkConnector. (org.apache.kafka.clients.consumer.internals.AbstractCoordinator:597)
[2017-07-31 11:58:38,998] INFO Revoking previously assigned partitions [] for group connect-InfinispanSinkConnector (org.apache.kafka.clients.consumer.internals.ConsumerCoordinator:419)
[2017-07-31 11:58:38,998] INFO (Re-)joining group connect-InfinispanSinkConnector (org.apache.kafka.clients.consumer.internals.AbstractCoordinator:432)
[2017-07-31 11:58:39,056] INFO Successfully joined group connect-InfinispanSinkConnector with generation 4 (org.apache.kafka.clients.consumer.internals.AbstractCoordinator:399)
[2017-07-31 11:58:39,057] INFO Setting newly assigned partitions [test-0] for group connect-InfinispanSinkConnector (org.apache.kafka.clients.consumer.internals.ConsumerCoordinator:262)
```

Lets run the camel route to query the Infinispan cache and see the result in this moment.

```
>camel-infinispan-kafka-demo$ mvn clean compile exec:java
[INFO] Scanning for projects...
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] Building Camel example with Infinispan 2.20.0-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO] 
[INFO] --- maven-clean-plugin:2.5:clean (default-clean) @ camel-infinispan-kafka-demo ---
[INFO] Deleting /home/oscerd/workspace/miscellanea/camel-infinispan-kafka-demo/target
[INFO] 
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ camel-infinispan-kafka-demo ---
[WARNING] File encoding has not been set, using platform encoding UTF-8, i.e. build is platform dependent!
[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!
[INFO] Copying 1 resource
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ camel-infinispan-kafka-demo ---
[INFO] Changes detected - recompiling the module!
[WARNING] File encoding has not been set, using platform encoding UTF-8, i.e. build is platform dependent!
[INFO] Compiling 4 source files to /home/oscerd/workspace/miscellanea/camel-infinispan-kafka-demo/target/classes
[WARNING] /home/oscerd/workspace/miscellanea/camel-infinispan-kafka-demo/src/main/java/com/github/oscerd/camel/infinispan/kafka/demo/CamelInfinispanRoute.java: /home/oscerd/workspace/miscellanea/camel-infinispan-kafka-demo/src/main/java/com/github/oscerd/camel/infinispan/kafka/demo/CamelInfinispanRoute.java uses unchecked or unsafe operations.
[WARNING] /home/oscerd/workspace/miscellanea/camel-infinispan-kafka-demo/src/main/java/com/github/oscerd/camel/infinispan/kafka/demo/CamelInfinispanRoute.java: Recompile with -Xlint:unchecked for details.
[INFO] 
[INFO] --- exec-maven-plugin:1.5.0:java (default-cli) @ camel-infinispan-kafka-demo ---
[.kafka.demo.Application.main()] RemoteCacheManager             INFO  ISPN004021: Infinispan version: 9.1.0.Final
[.kafka.demo.Application.main()] DefaultCamelContext            INFO  Apache Camel 2.20.0-SNAPSHOT (CamelContext: camel-1) is starting
[.kafka.demo.Application.main()] ManagedManagementStrategy      INFO  JMX is enabled
[.kafka.demo.Application.main()] DefaultTypeConverter           INFO  Type converters loaded (core: 192, classpath: 0)
[.kafka.demo.Application.main()] DefaultCamelContext            INFO  StreamCaching is not in use. If using streams then its recommended to enable stream caching. See more details at http://camel.apache.org/stream-caching.html
[.kafka.demo.Application.main()] DefaultCamelContext            INFO  Route: route1 started and consuming from: timer://foo?period=10000&repeatCount=0
[.kafka.demo.Application.main()] DefaultCamelContext            INFO  Total 1 routes, of which 1 are started.
[.kafka.demo.Application.main()] DefaultCamelContext            INFO  Apache Camel 2.20.0-SNAPSHOT (CamelContext: camel-1) started in 0.221 seconds
Starting Camel. Use ctrl + c to terminate the JVM.

[.kafka.demo.Application.main()] DefaultCamelContext            INFO  Apache Camel 2.20.0-SNAPSHOT (CamelContext: camel-2) is starting
[.kafka.demo.Application.main()] ManagedManagementStrategy      INFO  JMX is enabled
[.kafka.demo.Application.main()] DefaultTypeConverter           INFO  Type converters loaded (core: 192, classpath: 0)
[.kafka.demo.Application.main()] DefaultCamelContext            INFO  StreamCaching is not in use. If using streams then its recommended to enable stream caching. See more details at http://camel.apache.org/stream-caching.html
[.kafka.demo.Application.main()] DefaultCamelContext            INFO  Total 0 routes, of which 0 are started.
[.kafka.demo.Application.main()] DefaultCamelContext            INFO  Apache Camel 2.20.0-SNAPSHOT (CamelContext: camel-2) started in 0.012 seconds
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result size 0
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result content []
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result size 0
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result content []
.
.
.
.
.
```

We don't have data because no records has been sent to Kafka topic. Lets add some data through the Infinispan-Kafka producer.

```bash
>infinispan-kafka-producer$ mvn clean compile exec:exec
[INFO] Scanning for projects...
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] Building Infinispan Kafka Producer 0.0.1-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO] 
[INFO] --- maven-clean-plugin:2.5:clean (default-clean) @ infinispan-kafka-producer ---
[INFO] Deleting /home/oscerd/workspace/miscellanea/infinispan-kafka-producer/target
[INFO] 
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ infinispan-kafka-producer ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 1 resource
[INFO] 
[INFO] --- maven-compiler-plugin:3.2:compile (default-compile) @ infinispan-kafka-producer ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 2 source files to /home/oscerd/workspace/miscellanea/infinispan-kafka-producer/target/classes
[INFO] 
[INFO] --- exec-maven-plugin:1.5.0:exec (default-cli) @ infinispan-kafka-producer ---
2017-07-31 12:03:12 INFO  ProducerConfig:223 - ProducerConfig values: 
	acks = 1
	batch.size = 16384
	bootstrap.servers = [localhost:9092]
	buffer.memory = 33554432
	client.id = 
	compression.type = none
	connections.max.idle.ms = 540000
	enable.idempotence = false
	interceptor.classes = null
	key.serializer = class org.apache.kafka.common.serialization.StringSerializer
	linger.ms = 0
	max.block.ms = 60000
	max.in.flight.requests.per.connection = 5
	max.request.size = 1048576
	metadata.max.age.ms = 300000
	metric.reporters = []
	metrics.num.samples = 2
	metrics.recording.level = INFO
	metrics.sample.window.ms = 30000
	partitioner.class = class org.apache.kafka.clients.producer.internals.DefaultPartitioner
	receive.buffer.bytes = 32768
	reconnect.backoff.max.ms = 1000
	reconnect.backoff.ms = 50
	request.timeout.ms = 30000
	retries = 0
	retry.backoff.ms = 100
	sasl.jaas.config = null
	sasl.kerberos.kinit.cmd = /usr/bin/kinit
	sasl.kerberos.min.time.before.relogin = 60000
	sasl.kerberos.service.name = null
	sasl.kerberos.ticket.renew.jitter = 0.05
	sasl.kerberos.ticket.renew.window.factor = 0.8
	sasl.mechanism = GSSAPI
	security.protocol = PLAINTEXT
	send.buffer.bytes = 131072
	ssl.cipher.suites = null
	ssl.enabled.protocols = [TLSv1.2, TLSv1.1, TLSv1]
	ssl.endpoint.identification.algorithm = null
	ssl.key.password = null
	ssl.keymanager.algorithm = SunX509
	ssl.keystore.location = null
	ssl.keystore.password = null
	ssl.keystore.type = JKS
	ssl.protocol = TLS
	ssl.provider = null
	ssl.secure.random.implementation = null
	ssl.trustmanager.algorithm = PKIX
	ssl.truststore.location = null
	ssl.truststore.password = null
	ssl.truststore.type = JKS
	transaction.timeout.ms = 60000
	transactional.id = null
	value.serializer = class org.apache.kafka.common.serialization.StringSerializer

2017-07-31 12:03:12 INFO  AppInfoParser:83 - Kafka version : 0.11.0.0
2017-07-31 12:03:12 INFO  AppInfoParser:84 - Kafka commitId : cb8625948210849f
2017-07-31 12:03:13 INFO  KafkaProducer:972 - Closing the Kafka producer with timeoutMillis = 9223372036854775807 ms.
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 1.864 s
[INFO] Finished at: 2017-07-31T12:03:13+02:00
[INFO] Final Memory: 18M/296M
[INFO] ------------------------------------------------------------------------
```

In the connector log we should see something like this:

```bash
[2017-07-31 12:03:13,261] INFO Received 5 records (org.infinispan.kafka.InfinispanSinkTask:65)
[2017-07-31 12:03:13,261] INFO Record kafka coordinates:(test-1-{"name":"Andrea Cosentino"}). Writing it to Infinispan... (org.infinispan.kafka.InfinispanSinkTask:69)
[2017-07-31 12:03:13,290] INFO Record kafka coordinates:(test-2-{"name":"Jonathan Anstey"}). Writing it to Infinispan... (org.infinispan.kafka.InfinispanSinkTask:69)
[2017-07-31 12:03:13,292] INFO Record kafka coordinates:(test-3-{"name":"Claus Ibsen"}). Writing it to Infinispan... (org.infinispan.kafka.InfinispanSinkTask:69)
[2017-07-31 12:03:13,294] INFO Record kafka coordinates:(test-4-{"name":"Normam Maurer"}). Writing it to Infinispan... (org.infinispan.kafka.InfinispanSinkTask:69)
[2017-07-31 12:03:13,295] INFO Record kafka coordinates:(test-5-{"name":"Philip Roth"}). Writing it to Infinispan... (org.infinispan.kafka.InfinispanSinkTask:69)
[2017-07-31 12:03:20,015] INFO WorkerSinkTask{id=InfinispanSinkConnector-0} Committing offsets (org.apache.kafka.connect.runtime.WorkerSinkTask:278)
```

While in the Camel-Infinispan-Kafka demo log we should see a different result from querying:

```bash
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result size 0
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result content []
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result content [Author [name=Andrea Cosentino]]
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result size 1
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result content [Author [name=Andrea Cosentino]]
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result size 1
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result content [Author [name=Andrea Cosentino]]
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result size 1
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result content [Author [name=Andrea Cosentino]]
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result size 1
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result content [Author [name=Andrea Cosentino]]
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result size 1
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result content [Author [name=Andrea Cosentino]]
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result size 1
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result content [Author [name=Andrea Cosentino]]
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result size 1
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result content [Author [name=Andrea Cosentino]]
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result size 1
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result content [Author [name=Andrea Cosentino]]
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result size 1
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result content [Author [name=Andrea Cosentino]]
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result size 1
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result content [Author [name=Andrea Cosentino]]
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result size 1
[mel-1) thread #1 - timer://foo] CamelInfinispanRoute           INFO  Query Result content [Author [name=Andrea Cosentino]]
```

As you may see the Infinispan Cache has been populated with the data coming from Kafka topic test.

### Conclusion

This blog post introduce the new Infinispan-Kafka connector and show a little demo involving, Kafka, Infinispan and Camel. Obviously there is so much more work to do on the connector and contributions are more than welcome. Follow the developments on the [Github repo](https://github.com/infinispan/infinispan-kafka).
