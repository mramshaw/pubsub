# Adventures in Messaging

Noodling around with [pub/sub](http://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) systems.

In the examples below, the instructions will be for the _simplest possible configuration_. It is possible
to run both Redis and Kafka clusters, but for learning purposes it will suffice to use a single instance
of either.

* [Concepts](#concepts)
* [MQTT and Mosquitto](#mqtt-and-mosquitto)
   * [QoS](#qos)
   * [Message Retention](#message-retention)
* [Google Cloud Pub/Sub](#google-cloud-pubsub)
* [Amazon Simple Notification Service](#amazon-simple-notification-service)
* [RabbitMQ](#rabbitmq)
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
* [Reference](#reference)
* [To Do](#to-do)

## Concepts

The general concept is that __publishers__ create messages and ___publish___ them to a given
__channel__ or __topic__. And __Subscribers__ receive messages by ___subscribing___ to specific channel(s)
or topic(s). __Message Brokers__ may (or may not) handle the enqueueing and/or dispersal of the messages.

## MQTT and Mosquitto

I'm starting here because the ground is familiar:

    http://github.com/mramshaw/MQTT_and_mosquitto

__MQTT__ is the protocol and __Mosquitto__ is a message broker.

MQTT was specifically designed for unreliable networks, and so includes a __Last Will and Testament__
(LWT) feature. This is to handle the situation where clients unexpectedly drop or get disconnected - so
that closedown handling can be invoked (for a clean shutdown, for instance). The __Keep Alive__ feature
is of course important for determining the timing of when the LWT processing is invoked.

Depending on the specified [QoS](#qos) and [Message Retention](#message-retention), messages may or may
not be queued; as with [redis](#redis) listeners may need to be actively subscribed in order to receive
messages.

#### QoS

Valid values for Quality of Service are:

- 0 [at most once]
- 1 [at least once]
- 2 [exactly once]

QoS 0 is also referred to as "fire and forget" meaning these messages may or may not actually be delivered.

Messages published with QoS greater than zero will be queued, however the broker will respect the QoS
requested by the client - in which case the QoS may be downgraded.

QoS 2 is the most costly delivery mechanism. With QoS 1 the client must be capable of handling duplicated
messages.

#### Message Retention

Published messages may (or may not) be specified for retention.

If specified, the message is saved by the broker as the last known good value for the specific topic.

When a new client subscribes to a topic, they receive the last message that is retained on that topic.

[The broker stores only one retained message per topic.]

## Google Cloud Pub/Sub

Like MQTT QoS 0 or [redis](#redis), [Google Cloud Pub/Sub](http://cloud.google.com/pubsub/docs/) is also
apparently "fire and forget" meaning messages may or may not actually be delivered.

Supports large-scale messaging over HTTP (REST) or [gRPC](http://www.grpc.io/).

Note that, as with most messaging solutions, Cloud Pub/Sub doesn't guarantee the order of the messages.

After successfully pulling a message and processing it, you are supposed to __acknowledge__  the message
(this means notifying Google Cloud Pub/Sub that you successfully received the message). You may also
`--auto-ack` the message or messages, which automatically pulls and acknowledges the message or messages.

Failure to acknowledge the message within the __acknowledgement deadline__ period will result in the
message being re-sent.

## Amazon Simple Notification Service

Supported protocols:

1. [Amazon SQS](http://aws.amazon.com/sqs/)
2. HTTP/S
3. email
4. [SMS](http://en.wikipedia.org/wiki/SMS)
5. Lambda

[From: http://docs.aws.amazon.com/sns/latest/dg/welcome.html]

## RabbitMQ

Supported protocols:

1. AMQP 0-9-1
2. [STOMP](http://stomp.github.io/)
3. MQTT
4. AMQP 1.0
5. HTTP and WebSockets

[From: http://www.rabbitmq.com/protocols.html]

## ZeroMQ

Unusually, [ZeroMQ](http://zeromq.org/) is more of a protocol than middleware. Think sockets on steroids.

## Redis

[Perhaps not the ideal use case for [Redis](http://redis.io/), but as Redis is widely
 deployed perhaps worth considering. In any case, a useful tool for exploring how pub/sub works.]

Messages are not queued; as with [MQTT and Mosquitto](#mqtt-and-mosquitto) listeners
need to be actively subscribed in order to receive them.

Offers no persistence and no delivery guarantees.

Check out the following link for Salvatore Sanfilippo's thoughts on Redis and Pub/Sub:

    http://oldblog.antirez.com/post/redis-weekly-update-3-publish-submit.html

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

[Apache Kafka](http://kafka.apache.org/) describes itself as _a distributed
streaming platform_.

Like [redis](#redis), Kafka is a multi-use storage application that can be
used (among other things) for pub/sub messaging.

It is written in [Scala](http://www.scala-lang.org/), which requires a JVM
(Java Virtual Machine) to run.

Builds on [Apache Zookeeper](http://zookeeper.apache.org/), which is much like
[etcd](http://github.com/coreos/etcd).

There is also a pure Golang implementation, [Jocko](http://github.com/travisjeffery/jocko).

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

## Reference

Salvatore Sanfilippo (antirez) on Redis and Pub/Sub:

    http://oldblog.antirez.com/post/redis-weekly-update-3-publish-submit.html

[For anyone interested in Redis, the antirez weblog is a great resource.]

## To Do

- [ ] Explore ZeroMQ
- [x] Add QoS and retention notes for MQTT 
- [x] Flesh out the Kafka messaging details
- [x] Investigate [Google Cloud Pub/Sub](http://cloud.google.com/pubsub/docs/)
- [ ] Investigate [Amazon SNS](http://docs.aws.amazon.com/sns/latest/dg/welcome.html)
- [ ] Investigate [RabbitMQ](http://www.rabbitmq.com/) and [RabbitMQ as a Service](http://www.cloudamqp.com/)
