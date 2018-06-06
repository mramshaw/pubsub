# Adventures in Messaging

Noodling around with [pub/sub](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) systems.

In the examples below, the instructions will be for the _simplest possible configuration_. It is possible
to run both Redis and Kafka clusters, but for learning purposes it will suffice to use a single instance
of either.

* [Concepts](#concepts)
* [MQTT and Mosquitto](#mqtt-and-mosquitto)
* [ZeroMQ](#zeromq)
* [Redis](#redis)
    * [Start Redis](#start-redis)
    * [Register a subscriber](#register-a-subscriber)
    * [Publish a message](#publish-a-message)
    * [Useful commands](#useful-commands)
    * [Close down the publisher and subscriber](#close-down-the-publisher-and-subscriber)
    * [Close down the broker](#close-down-the-broker)
* [Kafka](#kafka)
    * [Java](#java)
    * [Zookeeper](#zookeeper)
    * [Start Kafka](#start-kafka)
    * [Create a topic](#create-a-topic)
    * [Pass messages](#pass-messages)
    * [Cleanup](#cleanup)
* [To Do](#to-do)

## Concepts

The general concept seems to be that __publishers__ create messages and _publish_ them to a given
__channel__ or __topic__. And __Subscribers__ receive messages by _subscribing_ to specific channel(s)
or topic(s). __Message Brokers__ may (or may not) handle the enqueueing and/or dispersal of the messages.

## MQTT and Mosquitto

I'm starting here because the ground is familiar:

    https://github.com/mramshaw/MQTT_and_mosquitto

__MQTT__ is the protocol and __Mosquitto__ is a message broker.

Messages are not queued; as with [redis](#redis) listeners need to be actively
subscribed in order to receive them.

## ZeroMQ

Unusually, [ZeroMQ](http://zeromq.org/) is more of a protocol than middleware. Think sockets on steroids.

## Redis

[Perhaps not the ideal use case for [Redis](https://redis.io/), but as Redis is widely
 deployed perhaps worth considering. In any case, a useful tool for exploring how pub/sub works.]

Messages are not queued; as with [MQTT and Mosquitto](#mqtt-and-mosquitto) listeners
need to be actively subscribed in order to receive them.

Offers no persistence and no delivery guarantees.

#### Start Redis

In a console, execute the following command to start Redis:

```
$ docker-compose up
```

#### Register a subscriber

In another console, use the `redis-cli` command to subscribe to a topic, as follows:

```
$ redis-cli
127.0.0.1:6379> subscribe news
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "news"
3) (integer) 1
```

[In Redis, a return code of 1 usually indicates success while 0 usually indicates failure.]

#### Publish a message

In a third console, use the `redis-cli` command to publish a message to the news topic, as follows:

```
$ redis-cli
127.0.0.1:6379> publish news "redis is up!"
(integer) 1
127.0.0.1:6379>
```

In the second console, the subscriber should echo the message sent:

```
$ redis-cli
127.0.0.1:6379> subscribe news
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "news"
3) (integer) 1
1) "message"
2) "news"
3) "redis is up!"
```

#### Useful commands

In the second console, the following commands may be interesting:

```
127.0.0.1:6379> pubsub channels
1) "news"
127.0.0.1:6379> pubsub numsub news
1) "news"
2) (integer) 1
127.0.0.1:6379>
```

[List the channels & List the number of subscribers to the news channel.]

#### Close down the publisher and subscriber

Either type `exit` or Ctrl-C in the second and third console to terminate.
  
#### Close down the broker

Shut down Redis (first console) with Ctrl-C.

And:

```
docker-compose down
```

## Kafka

[Apache Kafka](https://kafka.apache.org/) describes itself as _a distributed
streaming platform_.

Like [redis](#redis), Kafka is a multi-use storage application that can be
used (among other things) for pub/sub messaging.

It is written in [Scala](https://www.scala-lang.org/), which requires a JVM
(Java Virtual Machine) to run.

Builds on [Apache Zookeeper](https://zookeeper.apache.org/), which is much like
[etcd](https://github.com/coreos/etcd).

There is also a pure Golang implementation, [Jocko](https://github.com/travisjeffery/jocko).

#### Java

Verify `java` is installed:

```
$ java -version
openjdk version "1.8.0_171"
OpenJDK Runtime Environment (build 1.8.0_171-8u171-b11-0ubuntu0.16.04.1-b11)
OpenJDK 64-Bit Server VM (build 25.171-b11, mixed mode)
$
```

#### Zookeeper

Start `zookeeper` (bundled with Kafka) as follows:

```
$ bin/zookeeper-server-start.sh config/zookeeper.properties
<...>
```

#### Start Kafka

In a new console, start `kafka` as follows:

```
$ bin/kafka-server-start.sh config/server.properties
<...>
[2018-06-05 15:12:16,412] INFO [Kafka Server 0], started (kafka.server.KafkaServer)
```

#### Create a topic

In a third console, create a topic as follows:

```
$ bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic news
Created topic "news".
$
```

And verify the topic was created, as follows:

```
$ bin/kafka-topics.sh --list --zookeeper localhost:2181
news
$
```

#### Pass messages

In the third console, open a publisher as follows:

```
$ bin/kafka-console-producer.sh --broker-list localhost:9092 --topic news
>Kafka is UP!
>
```

[And type a message for broadcasting.]

In a fourth console, open a subscriber as follows:

```
$ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic news

```

Note that the recently published message is not received. Let's fix that:

```
$ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic news
^CProcessed a total of 0 messages
$ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic news --from-beginning
Kafka is UP!

```

[First we entered Ctrl-C to kill the subscriber. Then we restarted the subscriber
 with the `--from-beginning` option so as to get all previously published messages.]

And now our publisher (third console) operates as expected:

```
$ bin/kafka-console-producer.sh --broker-list localhost:9092 --topic news
>Kafka is UP!
>Hello?
>
```

And our subscriber (fourth console) receives the latest message:

```
$ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic news --from-beginning
Kafka is UP!
Hello?
```

[From the above it should be apparent that Kafka caches (stores) messages. Also, it does not
 seem to track which messages have been _consumed_ either - this seems to be a _subscriber_
 responsibility. For instance, specifying `from-beginning` (or any other constant offset)
 will mean that the subscriber messages will always start with the same message.]

#### Cleanup

Now kill everything with Ctrl-C.

Subscriber:

```
$ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic news --from-beginning
Kafka is UP!
Hello?
^CProcessed a total of 2 messages
$
```

Publisher:

```
$ bin/kafka-console-producer.sh --broker-list localhost:9092 --topic news
>Kafka is UP!
>Hello?
>^C$
```

Kafka:

```
<...>
[2018-06-05 15:51:45,855] INFO [Kafka Server 0], shut down completed (kafka.server.KafkaServer)
$
```

Zookeeper:

```
<...>
[2018-06-05 15:51:45,854] INFO Closed socket connection for client /127.0.0.1:45648 which had sessionid 0x163d20186a50000 (org.apache.zookeeper.server.NIOServerCnxn)
^C$
```

## To Do

- [ ] Explore ZeroMQ
- [x] Flesh out the Kafka messaging details
- [ ] Investigate [Amazon SNS](https://docs.aws.amazon.com/sns/latest/dg/SMSMessages.html)
- [ ] Investigate [Google Cloud Pub/Sub](https://cloud.google.com/pubsub/docs/)
