# Adventures in Messaging

Noodling around with [pub/sub](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) systems.

The general concept seems to be that __publishers__ create messages and _publish_ them to a given topic.
__Subscribers__ receive messages by _subscribing_ to a specific topic (or topics). __Message Brokers__
may (or may not) handle the enqueueing and/or dispersal of the messages.

## MQTT and Mosquitto

I'm starting here because the ground is familiar:

    https://github.com/mramshaw/MQTT_and_mosquitto

__MQTT__ is the protocol and __Mosquitto__ is a message broker.

Messages are not queued; as with [redis](#redis) listeners need to be actively
subscribed in order to receive them.

## ZeroMQ

Unusually, this is more of a protocol than middleware. Think sockets on steroids.

## Redis

[Perhaps not the ideal use case for Redis, but as Redis is widely deployed perhaps
 worth considering. In any case, a useful tool for exploring how pub/sub works.]

Messages are not queued; as with [MQTT and Mosquitto](#mqtt-and-mosquitto) listeners
need to be actively subscribed in order to receive them.

#### Start Redis

In a terminal, execute the following command to start Redis:

```
docker-compose up
```

#### Register a subscriber

In another terminal, use the `redis-cli` command to subscribe to a topic, as follows:

```
$ redis-cli
127.0.0.1:6379> subscribe news
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "news"
3) (integer) 1
```

#### Publish a message to the news topic

In a third terminal, use the `redis-cli` command to publish a message, as follows:

```
$ redis-cli
127.0.0.1:6379> publish news "redis is up!"
(integer) 1
127.0.0.1:6379>
```

In the second terminal, the subscriber should echo the message sent:

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

In the second terminal, the following commands may be interesting:

```
127.0.0.1:6379> pubsub channels
1) "news"
127.0.0.1:6379> pubsub numsub news
1) "news"
2) (integer) 1
127.0.0.1:6379>
```

#### Close down the publisher and subscriber

Either type `exit` or Ctrl-C in the second and third terminals to terminate.
  
#### Close down the broker

Shut down Redis (first terminal) with Ctrl-C.

And:

```
docker-compose down
```

## Kafka

Built on top of Apache Zookeeper.

## To Do

- [ ] Explore ZeroMQ
- [ ] Flesh out the Kafka messaging details
- [ ] Investigate [Amazon SNS](https://docs.aws.amazon.com/sns/latest/dg/SMSMessages.html)
- [ ] Investigate [Google Cloud Pub/Sub](https://cloud.google.com/pubsub/docs/)
