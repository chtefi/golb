---
title: "Looking at Kafka's consumers' offsets"
date: "2017-08-07T02:08Z"
layout: post
path: "/2017/08/07/looking-at-kafka-s-consumers-offsets/"
language: "en"
tags: scala, kafka, consumers, offsets, lag
---

Kafka has quite evolved since some times. Its consuming model is very powerful, can greatly scale, is quite simple to understand.
It has never changed from a external point of view, but internally, it did since Kafka 0.9, and the appearance of the `__consumer_offsets`.
Before that, consumers offsets were stored in Zookeeper. Now, Kafka is eating its own dog food. Why is that?

It's definitely an implementation detail we should not care about nor rely on, because it can change anytime soon in the newer versions, but it's also very interesting to know it's there, what does it contains, to understand the Kafka consuming model and its constraints.

We'll start by looking at higher-level commands to check the offsets, then we'll go deeper with the `__consumer_offsets` topic, to finish by a Kafka Streams processor to convert it to a JSON-readable topic to finally be consumed by a timeseries database, for monitoring and alerting purpose.

---
Summary {.summary}

[[toc]]

---


# Eating its own dog food

The topic `__consumer_offsets` stores the offsets of the topics consumed by consumer applications (and even of itself, if someone consumes it!).

Let's resume the vocabulary around the Kafka's consuming model, to understand what it should store:

- A consumer consumes a `topic`.
- A consumer belongs to a `groupId`. (Note that the same `groupId` can be used to consume different topics)
- A consumer consumes topic's `partitions`. (and can consume only some of them)
- Each consumed partitions has its own `offset` for each couple `(topic, groupId)`.

`__consumer_offsets` is the storage of all these things through time.

If we consume it, we can be aware of all the changes and progression of each consumers.

# How to read the current offsets?

## Admin command: ConsumerGroupCommand

It's probably the simplest human-friendly way to do so. You don't even have to know it's coming from this topic. (this topic is an implementation detail after all). 

Kafka has severals commands (available through the generic `kafka-run-class` script), here we care about `ConsumerGroupCommand`:

```shell
$ kafka-run-class kafka.admin.ConsumerGroupCommand
List all consumer groups, describe a consumer group, or delete consumer group info.
--bootstrap-server # Only with --new-consumer
--command-config <command config property file>
--delete
--describe
--group <consumer group>
--list
--new-consumer
--topic <topic>
--zookeeper <urls> # Only without --new-consumer
```

```shell
$ kafka-run-class kafka.admin.ConsumerGroupCommand --bootstrap-server localhost:9092 \
  --group mygroup \
  --new-consumer \
  --describe
GROUP     TOPIC     PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG      OWNER
mygroup   mytopic   0          unknown         6971670         unknown  consumer-1_/137.74.23.1
mygroup   mytopic   1          6504514         6504514         0        consumer-1_/137.74.23.1
mygroup   mytopic   2          unknown         6507388         unknown  consumer-1_/137.74.23.1
mygroup   mytopic   3          6175879         6969711         793832   consumer-1_/172.16.10.5
mygroup   mytopic   4          unknown         6503476         unknown  consumer-1_/172.16.10.5
```
Note: some offsets are `unknown` (therefore the lag also) because the consumers did not consume all the partitions yet.

### Legacy: migrating from Zookeeper

Notice the `--new-consumer` and the Kafka's broker address, it does not need a Zookeeper address as before.
If you did migrated from a previous Kafka version, according to the brokers configuration, Kafka can dual-writes the offsets into Zookeeper and Kafka's `__consumer_offsets` (see `dual.commit.enabled=true` and `offsets.storage=kafka`).

### Trick: summing-up the lag

When we have several partitions, it's sometimes useful to just care about the sum of each partition's lag (0 meaning the group has catched up the latest messages):

```shell
$ kafka-run-class kafka.admin.ConsumerGroupCommand --bootstrap-server localhost:9092 \
--new-consumer \
--group mygroup \
--describe 2>/dev/null | awk 'NR>1 { print $6 }' | paste -sd+ - | bc
98
```
You know the whole group has _only_ 98 events still not consumed. If this is a topic with tons of real-time events, that's not bad!

### Trick: listing all the active groups

This command is very useful to discover all the active groups on the cluster:
```shell
$ kafka-run-class kafka.admin.ConsumerGroupCommand --bootstrap-server localhost:9092 --new-consumer --list
money-streamers
monitoring
weather
```

Note that during a partition rebalancing, the affected group temporary disappears, because is not active anymore.

## Consuming __consumer_offsets

Because it's a topic, it's possible to just consume it as any other topic.

First of all, because it's an _internal_ Kafka topic, by default, the consumers can't see it, therefore they can't consume it.
We must ask them to not exclude it (default is true). We must add to the consumer's props:
```shell
$ echo "exclude.internal.topics=false" > /tmp/consumer.config`
```
Then use it to consume the topic:
```shell
$ kafka-console-consumer --consumer.config /tmp/consumer.config \
  --zookeeper localhost:2181 \
  --topic __consumer_offsets
```

Output:
```
     ▒k    ]▒▒▒▒  ]▒▒▒▒
     ▒kg    ]▒▒▒▒  ]▒▒▒▒
     ▒▒▒    ]▒▒▒▒  ]▒▒▒▒
```
WHAT KIND OF SORCERY IS THIS?

Because it's saved as binary data, we need some kind of formatter to help us out:

```shell
$ kafka-console-consumer --consumer.config /tmp/consumer.config \
  --formatter "kafka.coordinator.GroupMetadataManager\$OffsetsMessageFormatter" \
  --zookeeper localhost:2181 \
  --topic __consumer_offsets
```

Here it is:
```
[mygroup1,mytopic1,11]::[OffsetMetadata[55166421,NO_METADATA],CommitTime 1502060076305,ExpirationTime 1502146476305]
[mygroup1,mytopic1,13]::[OffsetMetadata[55037927,NO_METADATA],CommitTime 1502060076305,ExpirationTime 1502146476305]
[mygroup2,mytopic2,0]::[OffsetMetadata[126,NO_METADATA],CommitTime 1502060076343,ExpirationTime 1502146476343]
```

We can't really make any use of it, except playing with some regexes. Would it be better to get some nice JSON instead? (yes!)

Note that each message in this topic has a key and a value. It's very important, as we'll see next in the following Compaction section.

## Kafka Streams: convert it to JSON

I've written a [Kafka's Streams app](https://github.com/chtefi/kafka-streams-consumer-offsets-to-json) that reads this topic and convert its `(key, val)` to another topic, JSON-readable.


A typical converted message is:

```json
{
    "topic":"WordsWithCountsTopic",
    "partition":0,
    "group":"console-consumer-26549",
    "version":1,
    "offset":95,
    "metadata":"",
    "commitTimestamp":1501542796444,
    "expireTimestamp":1501629196444
}
```

It's a recording that the group `console-consumer-26549` which consumes the topic `WordsWithCountsTopic` has read until the offset `95` of its partition `0`.


# Who sends messages inside ?

Who and when actually?


...


# Compaction

`__consumer_offsets` is a compacted topic. It's useful to not consume too many disk space for no reason, because we don't care of the past state. The compaction is only possible because this topic has a fixed key for the same event: the combinaison `(group, topic, partition number)`.

The purpose of the `__consumer_offsets` topic is to keep the latest consumed offset per group/topic/partition, which is why the key is the combinaison of them. Through the compaction, only the latest value will be saved into Kafka's data, the past offsets are useless.


# Usage of the JSON version

It's another mean to monitor the evolution of the consumed offsets, and can easily be processed by any third-party program or database (such as timeseries database), because it's plain JSON.


Draw a dRuid dashboard:
----