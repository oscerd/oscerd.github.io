---
layout: post
title: Introducing Camel-Nats
---

In the latest version of Apache Camel (2.17.0) we released camel-nats. [NATS](https://nats.io), is a cloud-native messaging system from Apcera.
The first version of the component was based on [java_nats](https://github.com/tyagihas/java_nats), this library has been deprecated from a while and we decide to switch 
to the brand new client [JNats](https://github.com/nats-io/jnats) in the next major release (Camel 2.18.0)

### The component

[Camel-Nats](https://github.com/apache/camel/tree/master/components/camel-nats) provides both producing/consuming endpoints. These are the options you can define:

- servers - a list of comma separated gnatsd servers
- topic - The topic name you want to use
- reconnect - Whether or not using reconnection feature (default true)
- pedantic - Whether or not running in pedantic mode (default false)
- verbose - Whether or not running in verbose mode (default false)
- ssl - Whether or not using SSL (default false)
- reconnectTimeWait - Waiting time before attempts reconnection (in milliseconds, default 3000)
- maxReconnectAttempts - Max reconnection attempts (default 3)
- pingInterval - Ping interval to be aware if connection is still alive (in milliseconds, default 4000)
- noRandomizeServers - Whether or not randomizing the order of servers for the connection attempts (default false)
- queueName - The Queue name if we are using nats for a queue configuration (consumer only)
- maxMessages - Stop receiving messages from a topic we are subscribing to after maxMessages (default unlimited, consumer only)
- poolSize - Consumer pool size (default 10, consumer only)

For more information about Nats configuration take a look at the [docs](http://nats.io/documentation/)

### Examples 

From the producer perspective:

```java
from("direct:send").to("nats://localhost:4222?topic=test");
```

while from the consumer perspective you can have:

```java
from("nats://localhost:4222?topic=test&maxMessages=5").to("mock:result")
```

In this case you will consume messages from the topic test until you received 5 messages.

The message from the consumer will have two headers:

- CamelNatsMessageTimestamp, the timestamp of the consumed message
- CamelNatsSubscriptionId, the Subscription Id of the consumer
