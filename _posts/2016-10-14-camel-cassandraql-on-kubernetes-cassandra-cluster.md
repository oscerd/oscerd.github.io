---
layout: post
title: Using Camel-cassandraql on a Kubernetes Cassandra cluster with Fabric8
---

In the last months I blogged about [Testing Camel-cassandraql](http://oscerd.github.io/2016/07/25/camel-cassandraql-on-dockerized-cluster/) on a Dockerized cluster. This time I'd like to show you a similar example running on [Kubernetes](http://kubernetes.io/). I've worked on a fabric8 quickstart splitted in two different projects: [Cassandra server](https://github.com/fabric8io/fabric8-ipaas/tree/master/cassandra) and [Cassandra client](https://github.com/fabric8-quickstarts/cassandra-client). The server will spin up a Cassandra cluster with three nodes and the client will run a camel-cassandraql route querying the keyspace in the Cluster. You'll need a running Kubernetes cluster to follow this example. I decided to use [Fabric8 on Minikube](http://fabric8.io/guide/getStarted/minikube.html), but you can also try to run the example using [Fabric8 on Minishift](http://fabric8.io/guide/getStarted/minishift.html). Once you have your environment is up and running you can start to execute the following sections commands.

### Spinning up an Apache Cassandra cluster on Kubernetes

For this post we will use [Kubernetes](http://kubernetes.io/) to spin up our Apache Cassandra Cluster. To start the cluster you can use the [Fabric8 IPaas Cassandra app](https://github.com/fabric8io/fabric8-ipaas/tree/master/cassandra). Since we are using Minikube, you'll need to follow the [related instructions](http://fabric8.io/guide/getStarted/minikube.html).

Don't forget to setup your local Docker to interact with Minikube.

```
eval $(minikube docker-env)
```

This project will spin up a Cassandra Cluster with three nodes. 

As first step, in the cassandra folder, we will need to run

```
mvn clean install
```

and 

```
mvn fabric8:deploy
```

Once we're done with this commands we can watch the Kubernetes pods state for our specific app and after a while we should be able to see our pods running:

```
oscerd@localhost:~/workspace/fabric8-universe/fabric8-ipaas/cassandra$ kubectl get pods -l app=cassandra --watch
NAME                         READY     STATUS    RESTARTS   AGE
cassandra-2459930792-lfh93   1/1       Running   0          15s
cassandra-2459930792-m8c9p   1/1       Running   0          15s
cassandra-2459930792-zxiv3   1/1       Running   0          15s
```

I usually prefer the command line, but you can also watch the status of your pods on the wonderful [Fabric8 Console](https://fabric8.io/guide/console.html) and with Minikube/Minishift is super easy to access the console, simply run the following command:

```
minikube service fabric8
```

or on Minishift

```
minishift service fabric8
```

In Apache Cassandra nodetool is one of the most important command to manage a Cluster and executing command on single node. 

It is interesting to look at the way the Kubernetes team developed the Seeds discovery into them Cassandra example. The class to look at is [KubernetesSeedProvider](https://github.com/kubernetes/kubernetes/blob/master/examples/storage/cassandra/java/src/main/java/io/k8s/cassandra/KubernetesSeedProvider.java) and it implements the [SeedProvider interface](https://github.com/apache/cassandra/blob/trunk/src/java/org/apache/cassandra/locator/SeedProvider.java) from Apache Cassandra project. The getSeeds method is implemented [here](https://github.com/kubernetes/kubernetes/blob/master/examples/storage/cassandra/java/src/main/java/io/k8s/cassandra/KubernetesSeedProvider.java#L98-L168): if the KubernetesSeedProvider is unable to find seeds it will fall back to [createDefaultSeeds method](https://github.com/kubernetes/kubernetes/blob/master/examples/storage/cassandra/java/src/main/java/io/k8s/cassandra/KubernetesSeedProvider.java#L170-L204) that is exactly the [simpleSeedProvider implementation](https://github.com/apache/cassandra/blob/trunk/src/java/org/apache/cassandra/locator/SimpleSeedProvider.java) from Apache Cassandra Project.

The Kubenertes-cassandra jar is used in the [Cassandra DockerFile](https://github.com/kubernetes/kubernetes/blob/master/examples/storage/cassandra/image/Dockerfile) from Google example and it is added to the classpath in the [run.sh script](https://github.com/kubernetes/kubernetes/blob/master/examples/storage/cassandra/image/files/run.sh).

In the Cassandra fabric8-ipaas project we used the deployment.yml and service.yml configuration. You can find the details in the [repository](https://github.com/fabric8io/fabric8-ipaas/tree/master/cassandra/src/main/fabric8). You can also scale up/down your cluster with this simple command:

```
kubectl scale --replicas=2 deployment/cassandra
```

and after a while you'll see the scaled deployment:

```
oscerd@localhost:~/workspace/fabric8-universe/fabric8-ipaas/cassandrakubectl get pods -l app=cassandra --watch
NAME                         READY     STATUS    RESTARTS   AGE
cassandra-2459930792-lfh93   1/1       Running   0          28m
cassandra-2459930792-m8c9p   1/1       Running   0          28m
```

Our cluster is up and running now and we can run our camel-cassandraql route from [Cassandra client](https://github.com/fabric8-quickstarts/cassandra-client).

### Running the Cassandra-client Fabric8 Quickstart

In Fabric8 Quickstarts organization I've added a little [Cassandra-client example](https://github.com/fabric8-quickstarts/cassandra-client). The route will do a simple query on the keyspace we've just created on our Cluster. This quickstart requires Camel 2.18.0 release to work fine, now that 2.18.0 is out we can cover the example. The quickstart will first execute a bean to pre-populate the Cassandra keyspace as [init-method](https://github.com/fabric8-quickstarts/cassandra-client/blob/master/src/main/resources/META-INF/spring/camel-context.xml#L28) and it defines a [main route](https://github.com/fabric8-quickstarts/cassandra-client/blob/master/src/main/resources/META-INF/spring/camel-context.xml#L31-L35). It's a very simple quickstart (single query/print result set), but it shows how it is easy to interact with the Cassandra Cluster. You don't have to add contact points of single nodes to your Cassandra endpoint URI or use a complex configuration, you can simply use the service name ("cassandra") and Kubernetes will do the work for you.

You can run the example with:

```
mvn clean install
mvn fabric8:deploy
```

You should be able to see your cassandra-client running:

```
oscerd@localhost:~/workspace/fabric8-universe/fabric8-quickstarts/cassandra-client$ kubectl get pods
NAME                                      READY     STATUS    RESTARTS   AGE
cassandra-2459930792-lfh93                1/1       Running   0          45m
cassandra-2459930792-m8c9p                1/1       Running   0          45m
cassandra-2459930792-ms265                1/1       Running   0          15m
cassandra-client-3789635050-e1f2g         1/1       Running   0          1m
```

Lets take a look at the logs from the Cassandra-client pod

```
oscerd@localhost:~/workspace/fabric8-universe/fabric8-quickstarts/cassandra-client$ kubectl logs cassandra-client-3789635050-e1f2g
2016-10-14 09:06:15,786 [main           ] INFO  DCAwareRoundRobinPolicy        - Using data-center name 'datacenter1' for DCAwareRoundRobinPolicy (if this is incorrect, please provide the correct datacenter name with DCAwareRoundRobinPolicy constructor)
2016-10-14 09:06:15,789 [main           ] INFO  Cluster                        - New Cassandra host cassandra/10.0.0.67:9042 added
2016-10-14 09:06:15,790 [main           ] INFO  Cluster                        - New Cassandra host /172.17.0.10:9042 added
2016-10-14 09:06:15,790 [main           ] INFO  Cluster                        - New Cassandra host /172.17.0.12:9042 added
2016-10-14 09:06:20,701 [main           ] INFO  SpringCamelContext             - Apache Camel 2.18.0 (CamelContext: camel-1) is starting
2016-10-14 09:06:20,703 [main           ] INFO  ManagedManagementStrategy      - JMX is enabled
2016-10-14 09:06:20,928 [main           ] INFO  DefaultTypeConverter           - Loaded 190 type converters
2016-10-14 09:06:20,977 [main           ] INFO  DefaultRuntimeEndpointRegistry - Runtime endpoint registry is in extended mode gathering usage statistics of all incoming and outgoing endpoints (cache limit: 1000)
2016-10-14 09:06:21,228 [main           ] INFO  SpringCamelContext             - StreamCaching is not in use. If using streams then its recommended to enable stream caching. See more details at http://camel.apache.org/stream-caching.html
2016-10-14 09:06:21,231 [main           ] INFO  ClockFactory                   - Using native clock to generate timestamps.
2016-10-14 09:06:21,503 [main           ] INFO  DCAwareRoundRobinPolicy        - Using data-center name 'datacenter1' for DCAwareRoundRobinPolicy (if this is incorrect, please provide the correct datacenter name with DCAwareRoundRobinPolicy constructor)
2016-10-14 09:06:21,548 [main           ] INFO  Cluster                        - New Cassandra host cassandra/10.0.0.67:9042 added
2016-10-14 09:06:21,548 [main           ] INFO  Cluster                        - New Cassandra host /172.17.0.11:9042 added
2016-10-14 09:06:21,548 [main           ] INFO  Cluster                        - New Cassandra host /172.17.0.10:9042 added
2016-10-14 09:06:22,047 [main           ] INFO  SpringCamelContext             - Route: cassandra-route started and consuming from: timer://foo?period=5000
2016-10-14 09:06:22,075 [main           ] INFO  SpringCamelContext             - Total 1 routes, of which 1 are started.
2016-10-14 09:06:22,083 [main           ] INFO  SpringCamelContext             - Apache Camel 2.18.0 (CamelContext: camel-1) started in 1.374 seconds
2016-10-14 09:06:23,133 [0 - timer://foo] INFO  cassandra-route                - Query result set [Row[1, oscerd]]
2016-10-14 09:06:28,092 [0 - timer://foo] INFO  cassandra-route                - Query result set [Row[1, oscerd]]
```

As you can see everything is straightforward and you can think about more complex architectures. For example: an Infinispan cache with a persisted cache store based on Apache Cassandra Cluster with Java application, Camel routes or Spring-boot applications interacting with the Infinispan platform.

The route of the example is super simple:

{% raw %}
```xml
  <camelContext xmlns="http://camel.apache.org/schema/spring" depends-on="populate">
    <route id="cassandra-route">
      <from uri="timer:foo?period=5000"/>
      <to uri="cql://cassandra/test?cql=select * from users;&amp;consistencyLevel=quorum" />
      <log message="Query result set ${body}"/>
    </route>

  </camelContext>
```
{% endraw %}

and it abstracts completely the details of the Cassandra cluster (how many nodes? Where are the nodes? What are the addresses of them? and so on). 


### Conclusions

This example shows only a bit of the power of Kubernetes/Openshift/Fabric8/Microservices world and in the future I would like to create a complex architecture quickstart. Stay tuned!
