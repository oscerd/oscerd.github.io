---
layout: post
title: An introduction to Camel Kubernetes component
---

Since **Camel 2.17.0** we have a Kubernetes component to interact with a Kubernetes cluster. The aim of this post is providing a first introduction to the component: how it works, what you can do with it and a little example of producer endpoint. The article is based on the Camel-Kubernetes **2.18-SNAPSHOT** version.

### Main Features

Camel-Kubernetes is based on the popular [Kubernetes/Openshift client](https://github.com/fabric8io/kubernetes-client) from the [Fabric8](http://fabric8.io/) team. The component provides both Producer and Consumer endpoints and it has the following options (directly from the brand new automatic generated documentation of Apache Camel 2.18). To better understand the component you may need to take a look at [Kubernetes site](http://kubernetes.io)


| Name      | Group   | Default | Java Type | Description           |
|-----------|---------|---------|-----------|------------------     |
| masterUrl | common |  | String | *Required* Kubernetes Master url |
| apiVersion | common |  | String | The Kubernetes API Version to use |
| category | common |  | String | *Required* Kubernetes Producer and Consumer category |
| dnsDomain | common |  | String | The dns domain used for ServiceCall EIP |
| kubernetesClient | common |  | DefaultKubernetesClient | Default KubernetesClient to use if provided |
| portName | common |  | String | The port name used for ServiceCall EIP |
| bridgeErrorHandler | consumer | false | boolean | Allows for bridging the consumer to the Camel routing Error Handler which mean any exceptions occurred while the consumer is trying to pickup incoming messages or the likes will now be processed as a message and handled by the routing Error Handler. By default the consumer will use the org.apache.camel.spi.ExceptionHandler to deal with exceptions that will be logged at WARN/ERROR level and ignored. |
| labelKey | consumer |  | String | The Consumer Label key when watching at some resources |
| labelValue | consumer |  | String | The Consumer Label value when watching at some resources |
| namespace | consumer |  | String | The namespace |
| poolSize | consumer | 1 | int | The Consumer pool size |
| resourceName | consumer |  | String | The Consumer Resource Name we would like to watch |
| exceptionHandler | consumer (advanced) |  | ExceptionHandler | To let the consumer use a custom ExceptionHandler. Notice if the option bridgeErrorHandler is enabled then this options is not in use. By default the consumer will deal with exceptions that will be logged at WARN/ERROR level and ignored. |
| operation | producer |  | String | Producer operation to do on Kubernetes |
| exchangePattern | advanced | InOnly | ExchangePattern | Sets the default exchange pattern when creating an exchange |
| synchronous | advanced | false | boolean | Sets whether synchronous processing should be strictly used or Camel is allowed to use asynchronous processing (if supported). |
| caCertData | security |  | String | The CA Cert Data |
| caCertFile | security |  | String | The CA Cert File |
| clientCertData | security |  | String | The Client Cert Data |
| clientCertFile | security |  | String | The Client Cert File |
| clientKeyAlgo | security |  | String | The Key Algorithm used by the client |
| clientKeyData | security |  | String | The Client Key data |
| clientKeyFile | security |  | String | The Client Key file |
| clientKeyPassphrase | security |  | String | The Client Key Passphrase |
| oauthToken | security |  | String | The Auth Token |
| password | security |  | String | Password to connect to Kubernetes |
| trustCerts | security |  | Boolean | Define if the certs we used are trusted anyway or not |
| username | security |  | String | Username to connect to Kubernetes |

The headers used by the component are the following:

|Name |Type |Description |
|-----|---- |----------- |
|CamelKubernetesOperation |String |The Producer operation |
|CamelKubernetesNamespaceName |String |The Namespace name |
|CamelKubernetesNamespaceLabels |Map |The Namespace Labels |
|CamelKubernetesServiceLabels |Map |The Service labels |
|CamelKubernetesServiceName |String |The Service name |
|CamelKubernetesServiceSpec |io.fabric8.kubernetes.api.model.ServiceSpec |The Spec for a Service |
|CamelKubernetesReplicationControllersLabels |Map |Replication controller labels |
|CamelKubernetesReplicationControllerName |String |Replication controller name |
|CamelKubernetesReplicationControllerSpec |io.fabric8.kubernetes.api.model.ReplicationControllerSpec |The Spec for a Replication Controller |
|CamelKubernetesReplicationControllerReplicas |Integer |The number of replicas for a Replication Controller during the Scale operation |
|CamelKubernetesPodsLabels |Map |Pod labels |
|CamelKubernetesPodName |String |Pod name |
|CamelKubernetesPodSpec |io.fabric8.kubernetes.api.model.PodSpec |The Spec for a Pod |
|CamelKubernetesPersistentVolumesLabels |Map |Persistent Volume labels |
|CamelKubernetesPersistentVolumesName |String |Persistent Volume name |
|CamelKubernetesPersistentVolumesClaimsLabels |Map |Persistent Volume Claim labels |
|CamelKubernetesPersistentVolumesClaimsName |String |Persistent Volume Claim name | 
|CamelKubernetesPersistentVolumesClaimsSpec |io.fabric8.kubernetes.api.model.PersistentVolumeClaimSpec |The Spec for a Persistent Volume claim |
|CamelKubernetesSecretsLabels |Map |Secret labels |
|CamelKubernetesSecretsName |String |Secret name |
|CamelKubernetesSecret |io.fabric8.kubernetes.api.model.Secret |A Secret Object |
|CamelKubernetesResourcesQuotaLabels |Map |Resource Quota labels |
|CamelKubernetesResourcesQuotaName |String |Resource Quota name |
|CamelKubernetesResourceQuotaSpec |io.fabric8.kubernetes.api.model.ResourceQuotaSpec |The Spec for a Resource Quota |
|CamelKubernetesServiceAccountsLabels |Map |Service Account labels |
|CamelKubernetesServiceAccountName |String |Service Account name | 
|CamelKubernetesServiceAccount |io.fabric8.kubernetes.api.model.ServiceAccount |A Service Account object | 
|CamelKubernetesNodesLabels |Map |Node labels |
|CamelKubernetesNodeName |String |Node name |
|CamelKubernetesBuildsLabels |Map |Openshift Build labels |
|CamelKubernetesBuildName |String |Openshift Build name |
|CamelKubernetesBuildConfigsLabels |Map |Openshift Build Config labels |
|CamelKubernetesBuildConfigName |String |Openshift Build Config name |
|CamelKubernetesEventAction |io.fabric8.kubernetes.client.Watcher.Action |Action watched by the consumer |
|CamelKubernetesEventTimestamp |String |Timestamp of the action watched by the consumer |
|CamelKubernetesConfigMapName |String |ConfigMap name |
|CamelKubernetesConfigMapsLabels |Map |ConfigMap labels |
|CamelKubernetesConfigData |Map |ConfigMap Data |

The whole component divides his features in a set of categories:

 - namespaces
 - services
 - replicationControllers
 - pods
 - persistentVolumes
 - persistentVolumesClaims
 - secrets
 - resourcesQuota
 - serviceAccounts
 - nodes
 - configMaps
 - builds
 - buildConfigs

The producer endpoints are based on a set of Operations:

- Namespaces
   - listNamespaces
   - listNamespacesByLabels
   - getNamespace
   - createNamespace
   - deleteNamespace
    
- Services 
   - listServices
   - listServicesByLabels
   - getService
   - createService
   - deleteService
    
- Replication Controllers
   - listReplicationControllers
   - listReplicationControllersByLabels
   - getReplicationController
   - createReplicationController
   - deleteReplicationController
   - scaleReplicationController
    
- Pods
   - listPods
   - listPodsByLabels
   - getPod
   - createPod
   - deletePod
    
- Persistent Volumes
   - listPersistentVolumes
   - listPersistentVolumesByLabels
   - getPersistentVolume
    
- Persistent Volumes Claims
   - listPersistentVolumesClaims
   - listPersistentVolumesClaimsByLabels
   - getPersistentVolumeClaim
   - createPersistentVolumeClaim
   - deletePersistentVolumeClaim
    
- Secrets
   - listSecrets
   - listSecretsByLabels
   - getSecret
   - createSecret
   - deleteSecret
    
- Resources quota
   - listResourcesQuota
   - listResourcesQuotaByLabels
   - getResourceQuota
   - createResourceQuota
   - deleteResourceQuota
    
- Service Accounts
   - listServiceAccounts
   - listServiceAccountsByLabels
   - getServiceAccount
   - createServiceAccount
   - deleteServiceAccount
    
- Nodes
   - listNodes
   - listNodesByLabels
   - getNode
    
- Config Maps
   - listConfigMaps
   - listConfigMapsByLabels
   - getConfigMap
   - createConfigMap
   - deleteConfigMap
    
- Builds
   - listBuilds
   - listBuildsByLabels
   - getBuild
    
- Build Configs
   - listBuildConfigs
   - listBuildConfigsByLabels
   - getBuildConfig

The documentation part is usually so boring, so let's see how you can declare a producer or a consumer to interact with a Kubernetes cluster:

```java
    from("direct:list")
        .to("kubernetes://https://localhost:8443?oauthToken=xxxxxxxx&category=pods&operation=listPods")
        .to("mock:result")
```

In this Producer snippet you will ask for the list of Pods in any namespace of your Kubernetes cluster. The operation will return a list of Pod (io.fabric8.kubernetes.api.model.Pod).

```java
    from("kubernetes://https://localhost:8443?oauthToken=xxxxxxxx&category=pods&namespace=default&labelKey=kind&labelValue=http")
        .to("mock:result")
```

In this Consumer snippet you will consume events from the Kubernetes cluster related to Pods, in the `default` namespace, labeled with key `kind` and value `http`.
The exchange will contain the Pod object (in the body) with all the related metadata, a timestamp (in the `CamelKubernetesEventTimestamp` header) and an action (in the `CamelKubernetesEventAction` header).

### A Little example

In the last days I worked on a little example that lists pods from a Kubernetes Cluster. I added the example to the Apache Camel examples folder.
The [Camel CDI Kubernetes example](https://github.com/apache/camel/tree/master/examples/camel-example-cdi-kubernetes) assumes you have a Kubernetes Cluster running in your environment.
Personally, for testing I prefer to use the [Vagrant Openshift Image](https://github.com/fabric8io/fabric8-installer/tree/master/vagrant/openshift) from the [Fabric8](http://fabric8.io/) team: 
to run the image and getting started you can follow the [guide](http://fabric8.io/guide/getStartedVagrant.html) from [Fabric8 documentation](http://fabric8.io/guide/index.html).

After the `vagrant up` command has completed remember to run the following command:

```
fabric8-installer/vagrant/openshift$ export KUBERNETES_DOMAIN=vagrant.f8
fabric8-installer/vagrant/openshift$ export DOCKER_HOST=tcp://vagrant.f8:2375
```

and login in Openshift

```
fabric8-installer/vagrant/openshift$ oc login https://172.28.128.4:8443
```

you will be asked from user/password (admin/admin). After the login will be successful you will be able to get your OAuth token (this will be useful for the Camel-Kubernetes example):

```
fabric8-installer/vagrant/openshift$ oc whoami --token
XXxnZi2YghQyke-eARXBZ38K-p5zIpco0ShdzO8gK6U
```

Now we have all the elements to run our example. Edit the [apache-deltaspike.properties](https://github.com/apache/camel/blob/master/examples/camel-example-cdi-kubernetes/src/main/resources/META-INF/apache-deltaspike.properties) with the correct OAuth Token from Openshift.

```
oscerd@localhost:~/workspace/apache-camel/camel/examples/camel-example-cdi-kubernetes$ mvn compile camel:run
```

The route is triggered with a timer and it has a repeatCount option equals to 3. The output of the examples will be:

```
2016-07-12 12:00:42,220 [cdi.Main.main()] INFO  CdiCamelExtension              - Camel CDI is starting Camel context [camel-example-kubernetes-cdi]
2016-07-12 12:00:42,220 [cdi.Main.main()] INFO  DefaultCamelContext            - Apache Camel 2.18-SNAPSHOT (CamelContext: camel-example-kubernetes-cdi) is starting
2016-07-12 12:00:42,222 [cdi.Main.main()] INFO  ManagedManagementStrategy      - JMX is enabled
2016-07-12 12:00:42,355 [cdi.Main.main()] INFO  DefaultTypeConverter           - Loaded 188 type converters
2016-07-12 12:00:42,376 [cdi.Main.main()] INFO  DefaultRuntimeEndpointRegistry - Runtime endpoint registry is in extended mode gathering usage statistics of all incoming and outgoing endpoints (cache limit: 1000)
2016-07-12 12:00:42,468 [cdi.Main.main()] INFO  DefaultCamelContext            - StreamCaching is not in use. If using streams then its recommended to enable stream caching. See more details at http://camel.apache.org/stream-caching.html
2016-07-12 12:00:42,828 [cdi.Main.main()] INFO  DefaultCamelContext            - Route: route1 started and consuming from: timer://stream?repeatCount=3
2016-07-12 12:00:42,831 [cdi.Main.main()] INFO  DefaultCamelContext            - Total 1 routes, of which 1 are started.
2016-07-12 12:00:42,832 [cdi.Main.main()] INFO  DefaultCamelContext            - Apache Camel 2.18-SNAPSHOT (CamelContext: camel-example-kubernetes-cdi) started in 0.611 seconds
2016-07-12 12:00:42,878 [cdi.Main.main()] INFO  Bootstrap                      - WELD-ENV-002003: Weld SE container STATIC_INSTANCE initialized
We currently have 13 pods
Pod name docker-registry-1-c6ie5 with status Running
Pod name fabric8-docker-registry-pgo5y with status Running
Pod name fabric8-forge-wvyw7 with status Running
Pod name fabric8-tr0b9 with status Running
Pod name gogs-2p4mn with status Running
Pod name grafana-754y7 with status Running
Pod name infinispan-client-a7z3k with status Running
Pod name infinispan-client-iubag with status Running
Pod name infinispan-server-wl0in with status Running
Pod name jenkins-cr2ez with status Running
Pod name nexus-aarks with status Running
Pod name prometheus-mp0kr with status Running
Pod name router-1-dkjsb with status Running
We currently have 13 pods
Pod name docker-registry-1-c6ie5 with status Running
Pod name fabric8-docker-registry-pgo5y with status Running
Pod name fabric8-forge-wvyw7 with status Running
Pod name fabric8-tr0b9 with status Running
Pod name gogs-2p4mn with status Running
Pod name grafana-754y7 with status Running
Pod name infinispan-client-a7z3k with status Running
Pod name infinispan-client-iubag with status Running
Pod name infinispan-server-wl0in with status Running
Pod name jenkins-cr2ez with status Running
Pod name nexus-aarks with status Running
Pod name prometheus-mp0kr with status Running
Pod name router-1-dkjsb with status Running
We currently have 13 pods
Pod name docker-registry-1-c6ie5 with status Running
Pod name fabric8-docker-registry-pgo5y with status Running
Pod name fabric8-forge-wvyw7 with status Running
Pod name fabric8-tr0b9 with status Running
Pod name gogs-2p4mn with status Running
Pod name grafana-754y7 with status Running
Pod name infinispan-client-a7z3k with status Running
Pod name infinispan-client-iubag with status Running
Pod name infinispan-server-wl0in with status Running
Pod name jenkins-cr2ez with status Running
Pod name nexus-aarks with status Running
Pod name prometheus-mp0kr with status Running
Pod name router-1-dkjsb with status Running
^C2016-07-12 12:00:50,946 [Thread-1       ] INFO  MainSupport$HangupInterceptor  - Received hang up - stopping the main instance.
2016-07-12 12:00:50,950 [Thread-1       ] INFO  CamelContextProducer           - Camel CDI is stopping Camel context [camel-example-kubernetes-cdi]
2016-07-12 12:00:50,950 [Thread-1       ] INFO  DefaultCamelContext            - Apache Camel 2.18-SNAPSHOT (CamelContext: camel-example-kubernetes-cdi) is shutting down
2016-07-12 12:00:50,951 [Thread-1       ] INFO  DefaultShutdownStrategy        - Starting to graceful shutdown 1 routes (timeout 300 seconds)
2016-07-12 12:00:50,953 [ - ShutdownTask] INFO  DefaultShutdownStrategy        - Route: route1 shutdown complete, was consuming from: timer://stream?repeatCount=3
2016-07-12 12:00:50,953 [Thread-1       ] INFO  DefaultShutdownStrategy        - Graceful shutdown of 1 routes completed in 0 seconds
2016-07-12 12:00:50,965 [Thread-1       ] INFO  MainLifecycleStrategy          - CamelContext: camel-example-kubernetes-cdi has been shutdown, triggering shutdown of the JVM.
2016-07-12 12:00:50,968 [Thread-1       ] INFO  DefaultCamelContext            - Apache Camel 2.18-SNAPSHOT (CamelContext: camel-example-kubernetes-cdi) uptime 8.748 seconds
2016-07-12 12:00:50,968 [Thread-1       ] INFO  DefaultCamelContext            - Apache Camel 2.18-SNAPSHOT (CamelContext: camel-example-kubernetes-cdi) is shutdown in 0.018 seconds
2016-07-12 12:00:50,977 [Thread-1       ] INFO  Bootstrap                      - WELD-ENV-002001: Weld SE container STATIC_INSTANCE shut down
```

As you can see the returned pods are 13 in my environment and all of them are in Running status.

### Conclusion

This post was just a little introduction on what Camel-Kubernetes can do. I think the most interesting part is the consumer side anyway. I will focus on the different types of consumer in one of next Camel related post. Meanwhile please try the component yourself and if you find something wrong, you think something can be improved or you think there is something else we can focus on, don't hesitate and post on [Camel Dev mailing list](http://camel.465427.n5.nabble.com/Camel-Development-f479097.html) or on the [Camel Users mailing list](http://camel.465427.n5.nabble.com/Camel-Users-f465428.html), or, if you are sure there is a bug or you think an improvement is truly needed, raise a JIRA on [Apache Camel JIRA](https://issues.apache.org/jira/browse/CAMEL/). 
You can find examples of different operations you can do with Producer Endpoints and Consumer Endpoints in the [Component unit test](https://github.com/apache/camel/tree/master/components/camel-kubernetes/src/test).

Stay tuned for the next Camel-Kubernetes consumers post.
