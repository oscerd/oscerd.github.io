---
layout: post
title: Testing camel-cassandraql on a Dockerized Apache Cassandra cluster
---

From **Camel-2.15.0** we have a component for the integration with [Apache Cassandra](http://cassandra.apache.org/). With this post I'd like to show you what you can do with this component on a real Cassandra cluster. We will use the docker images from my [personal account on Docker Hub](https://hub.docker.com/r/oscerd/cassandra/). As first step we need to setup our Cluster. 

### Spinning up an Apache Cassandra cluster with Docker

For this post we will use [Docker](https://www.docker.com/) to spin up our Apache Cassandra Cluster. When everything will be up and running we will run a simple example to show how the Camel-cassandraql component can be used to interact with the cluster. 

As first step we will need to run a single node cluster:

```
docker run --name master_node -dt oscerd/cassandra
```

This command will run a single container with an Apache Cassandra instance (the Cassandra version is the one of latest tick-tock release, the 3.6). We could use a single-node cluster but in this way we won't exploit the Cassandra features. So let's add two other nodes!

```
docker run --name node1 -d -e SEED="$(docker inspect --format='{{ .NetworkSettings.IPAddress }}' master_node)" oscerd/cassandra
docker run --name node2 -d -e SEED="$(docker inspect --format='{{ .NetworkSettings.IPAddress }}' master_node)" oscerd/cassandra
```

Now we have two other nodes in our Cluster. The environment variable SEED will be used to make the node aware of, at least, one of the nodes in the Cluster. In the `cassandra.yaml` file there will be a `seeds` entry to track this information. More informations can be found in the [Cassandra documentation](http://docs.datastax.com/en/cassandra/3.x/cassandra/initialize/initSingleDS.html).
To be sure everything is up and running we need to use the [Nodetool utility](http://docs.datastax.com/en/cassandra/3.x/cassandra/tools/toolsNodetool.html), by running the following command:

```
docker exec -ti master_node /opt/cassandra/bin/nodetool status
```

and we should get the Cluster status as output:

```
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns (effective)  Host ID                               Rack
UN  172.17.0.3  102.67 KiB  256          65.9%             1a985c48-33a1-44aa-b7e9-f1a3620a6482  rack1
UN  172.17.0.2  107.64 KiB  256          68.2%             da54ce5e-6433-4ea0-b2c3-fbc6c63ea955  rack1
UN  172.17.0.4  15.42 KiB  256          65.8%             0f2ba25a-37b0-4f27-a10a-d9a44655396a  rack1
```

To run the example we need to create a keyspace and a table. Locally I have downloaded my [Apache Cassandra](http://cassandra.apache.org/) package with version 3.7. Run the `cqlsh` command:

```
<LOCAL_CASSANDRA_HOME>/bin/cqlsh $(docker inspect --format='{{ .NetworkSettings.IPAddress }}' master_node)
```

You should see the Cqlsh prompt

```
Connected to Test Cluster at 172.17.0.2:9042.
[cqlsh 5.0.1 | Cassandra 3.6 | CQL spec 3.4.2 | Native protocol v4]
Use HELP for help.
cqlsh>
```

Let's create a namespace `test` with a table `users`

```
create keyspace test with replication = {'class':'SimpleStrategy', 'replication_factor':3};
use test;
create table users ( id int primary key, name text );
insert into users (id,name) values (1, 'oscerd');
quit;
```

and check everything is fine by run a simple query:

```
cqlsh> use test;
cqlsh:test> select * from users;

 id | name
----+--------
  1 | oscerd

(1 rows)
cqlsh:test> 
```

You can do the same on the other two nodes, to check you cluster is working as you expect.
Following this steps we now have a three nodes cluster up and running. Let's use a simple camel-cassandraql example.

### Run the example from Apache Camel

In Apache Camel I added a little example showing how to query (a select and an insert operations) an Apache Cassandra cluster with the related component, you can find the example here [Camel-example-cdi-cassandraql](https://github.com/apache/camel/tree/master/examples/camel-example-cdi-cassandraql)
You can run the example with:

```
mvn clean compile
mvn camel:run
```

and you should have the following output:

```
2016-07-24 15:33:50,812 [cdi.Main.main()] INFO  Version                        - WELD-000900: 2.3.5 (Final)
Jul 24, 2016 3:33:50 PM org.apache.deltaspike.core.impl.config.EnvironmentPropertyConfigSourceProvider <init>
INFO: Custom config found by DeltaSpike. Name: 'META-INF/apache-deltaspike.properties', URL: 'file:/home/oscerd/workspace/apache-camel/camel/examples/camel-example-cdi-cassandraql/target/classes/META-INF/apache-deltaspike.properties'
Jul 24, 2016 3:33:50 PM org.apache.deltaspike.core.util.ProjectStageProducer initProjectStage
INFO: Computed the following DeltaSpike ProjectStage: Production
2016-07-24 15:33:51,064 [cdi.Main.main()] INFO  Bootstrap                      - WELD-000101: Transactional services not available. Injection of @Inject UserTransaction not available. Transactional observers will be invoked synchronously.
2016-07-24 15:33:51,170 [cdi.Main.main()] INFO  Event                          - WELD-000411: Observer method [BackedAnnotatedMethod] protected org.apache.deltaspike.core.impl.message.MessageBundleExtension.detectInterfaces(@Observes ProcessAnnotatedType) receives events for all annotated types. Consider restricting events using @WithAnnotations or a generic type with bounds.
2016-07-24 15:33:51,174 [cdi.Main.main()] INFO  Event                          - WELD-000411: Observer method [BackedAnnotatedMethod] protected org.apache.deltaspike.core.impl.interceptor.GlobalInterceptorExtension.promoteInterceptors(@Observes ProcessAnnotatedType, BeanManager) receives events for all annotated types. Consider restricting events using @WithAnnotations or a generic type with bounds.
2016-07-24 15:33:51,189 [cdi.Main.main()] INFO  Event                          - WELD-000411: Observer method [BackedAnnotatedMethod] private org.apache.camel.cdi.CdiCamelExtension.processAnnotatedType(@Observes ProcessAnnotatedType<?>) receives events for all annotated types. Consider restricting events using @WithAnnotations or a generic type with bounds.
2016-07-24 15:33:51,195 [cdi.Main.main()] INFO  Event                          - WELD-000411: Observer method [BackedAnnotatedMethod] protected org.apache.deltaspike.core.impl.exclude.extension.ExcludeExtension.vetoBeans(@Observes ProcessAnnotatedType, BeanManager) receives events for all annotated types. Consider restricting events using @WithAnnotations or a generic type with bounds.
2016-07-24 15:33:51,491 [cdi.Main.main()] WARN  Validator                      - WELD-001478: Interceptor class org.apache.deltaspike.core.impl.throttling.ThrottledInterceptor is enabled for the application and for the bean archive /home/oscerd/.m2/repository/org/apache/deltaspike/core/deltaspike-core-impl/1.7.1/deltaspike-core-impl-1.7.1.jar. It will only be invoked in the @Priority part of the chain.
2016-07-24 15:33:51,491 [cdi.Main.main()] WARN  Validator                      - WELD-001478: Interceptor class org.apache.deltaspike.core.impl.lock.LockedInterceptor is enabled for the application and for the bean archive /home/oscerd/.m2/repository/org/apache/deltaspike/core/deltaspike-core-impl/1.7.1/deltaspike-core-impl-1.7.1.jar. It will only be invoked in the @Priority part of the chain.
2016-07-24 15:33:51,491 [cdi.Main.main()] WARN  Validator                      - WELD-001478: Interceptor class org.apache.deltaspike.core.impl.future.FutureableInterceptor is enabled for the application and for the bean archive /home/oscerd/.m2/repository/org/apache/deltaspike/core/deltaspike-core-impl/1.7.1/deltaspike-core-impl-1.7.1.jar. It will only be invoked in the @Priority part of the chain.
2016-07-24 15:33:52,244 [cdi.Main.main()] INFO  CdiCamelExtension              - Camel CDI is starting Camel context [camel-example-cassandraql-cdi]
2016-07-24 15:33:52,245 [cdi.Main.main()] INFO  DefaultCamelContext            - Apache Camel 2.18-SNAPSHOT (CamelContext: camel-example-cassandraql-cdi) is starting
2016-07-24 15:33:52,246 [cdi.Main.main()] INFO  ManagedManagementStrategy      - JMX is enabled
2016-07-24 15:33:52,352 [cdi.Main.main()] INFO  DefaultTypeConverter           - Loaded 189 type converters
2016-07-24 15:33:52,367 [cdi.Main.main()] INFO  DefaultRuntimeEndpointRegistry - Runtime endpoint registry is in extended mode gathering usage statistics of all incoming and outgoing endpoints (cache limit: 1000)
2016-07-24 15:33:52,465 [cdi.Main.main()] INFO  DefaultCamelContext            - StreamCaching is not in use. If using streams then its recommended to enable stream caching. See more details at http://camel.apache.org/stream-caching.html
2016-07-24 15:33:52,547 [cdi.Main.main()] INFO  NettyUtil                      - Did not find Netty's native epoll transport in the classpath, defaulting to NIO.
2016-07-24 15:33:52,789 [cdi.Main.main()] INFO  DCAwareRoundRobinPolicy        - Using data-center name 'datacenter1' for DCAwareRoundRobinPolicy (if this is incorrect, please provide the correct datacenter name with DCAwareRoundRobinPolicy constructor)
2016-07-24 15:33:52,790 [cdi.Main.main()] INFO  Cluster                        - New Cassandra host /172.17.0.3:9042 added
2016-07-24 15:33:52,791 [cdi.Main.main()] INFO  Cluster                        - New Cassandra host /172.17.0.2:9042 added
2016-07-24 15:33:52,791 [cdi.Main.main()] INFO  Cluster                        - New Cassandra host /172.17.0.4:9042 added
2016-07-24 15:33:52,914 [cdi.Main.main()] INFO  DCAwareRoundRobinPolicy        - Using data-center name 'datacenter1' for DCAwareRoundRobinPolicy (if this is incorrect, please provide the correct datacenter name with DCAwareRoundRobinPolicy constructor)
2016-07-24 15:33:52,914 [cdi.Main.main()] INFO  Cluster                        - New Cassandra host /172.17.0.3:9042 added
2016-07-24 15:33:52,914 [cdi.Main.main()] INFO  Cluster                        - New Cassandra host /172.17.0.2:9042 added
2016-07-24 15:33:52,914 [cdi.Main.main()] INFO  Cluster                        - New Cassandra host /172.17.0.4:9042 added
2016-07-24 15:33:52,985 [cdi.Main.main()] INFO  DefaultCamelContext            - Route: route1 started and consuming from: timer://stream?repeatCount=1
2016-07-24 15:33:52,986 [cdi.Main.main()] INFO  DefaultCamelContext            - Total 1 routes, of which 1 are started.
2016-07-24 15:33:52,987 [cdi.Main.main()] INFO  DefaultCamelContext            - Apache Camel 2.18-SNAPSHOT (CamelContext: camel-example-cassandraql-cdi) started in 0.742 seconds
2016-07-24 15:33:53,018 [cdi.Main.main()] INFO  Bootstrap                      - WELD-ENV-002003: Weld SE container STATIC_INSTANCE initialized
2016-07-24 15:33:54,041 [ timer://stream] INFO  route1                         - Result from query [Row[1, oscerd]]
```

Running the query again you should see another entry in the `users` table:

```
cqlsh> use test;
cqlsh:test> select * from users;

 id | name
----+-----------
  1 |    oscerd
  2 | davsclaus

(2 rows)
cqlsh:test>
```

The route of the example is simple:

```java
    @ContextName("camel-example-cassandraql-cdi")
    static class KubernetesRoute extends RouteBuilder {

        @Override
        public void configure() {
            from("timer:stream?repeatCount=1")
                .to("cql://{{cassandra-master-ip}},{{cassandra-node1-ip}},{{cassandra-node2-ip}}/test?cql={{cql-select-query}}&consistencyLevel=quorum")
                .log("Result from query ${body}")
                .process(exchange -> {
                    exchange.getIn().setBody(Arrays.asList("davsclaus"));
                })
                .to("cql://{{cassandra-master-ip}},{{cassandra-node1-ip}},{{cassandra-node2-ip}}/test?cql={{cql-insert-query}}&consistencyLevel=quorum");
        }
    }
```

As you may see we are using a Quorum Consistency Level for both the select and insert operations. You can find more informations about it in the [Cassandra Consistency Level documentation](https://docs.datastax.com/en/cassandra/3.x/cassandra/dml/dmlConfigConsistency.html)

### Conclusions

Using Docker to spin up a Cassandra Cluster can be useful during integration and performance testing. In this post we saw a little example of the camel-cassandraql features in a real Apache Cassandra cluster.

Before the camel-cassandraql Pull Request submission, I was working on a Camel-Cassandra component too. This component is a bit different from the cassandraql one and you can find more information in the [Github Repository](https://github.com/oscerd/camel-cassandra). It is aligned with the latest Apache Camel released version 2.17.2.
