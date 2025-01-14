---
id: concepts-clients
title: Pulsar Clients
sidebar_label: "Clients"
original_id: concepts-clients
---

Pulsar exposes a client API with language bindings for [Java](client-libraries-java.md) and [C++](client-libraries-cpp). The client API optimizes and encapsulates Pulsar's client-broker communication protocol and exposes a simple and intuitive API for use by applications.

Under the hood, the current official Pulsar client libraries support transparent reconnection and/or connection failover to brokers, queuing of messages until acknowledged by the broker, and heuristics such as connection retries with backoff.

> #### Custom client libraries
> If you'd like to create your own client library, we recommend consulting the documentation on Pulsar's custom [binary protocol](developing-binary-protocol)


## Client setup phase

When an application wants to create a producer/consumer, the Pulsar client library will initiate a setup phase that is composed of two steps:

1. The client will attempt to determine the owner of the topic by sending an HTTP lookup request to the broker. The request could reach one of the active brokers which, by looking at the (cached) zookeeper metadata will know who is serving the topic or, in case nobody is serving it, will try to assign it to the least loaded broker.
1. Once the client library has the broker address, it will create a TCP connection (or reuse an existing connection from the pool) and authenticate it. Within this connection, client and broker exchange binary commands from a custom protocol. At this point the client will send a command to create producer/consumer to the broker, which will comply after having validated the authorization policy.

Whenever the TCP connection breaks, the client will immediately re-initiate this setup phase and will keep trying with exponential backoff to re-establish the producer or consumer until the operation succeeds.

## Reader interface

In Pulsar, the "standard" [consumer interface](concepts-messaging.md#consumers) involves using consumers to listen on [topics](reference-terminology.md#topic), process incoming messages, and finally acknowledge those messages when they've been processed. Whenever a consumer connects to a topic, it automatically begins reading from the earliest un-acked message onward because the topic's cursor is automatically managed by Pulsar.

The **reader interface** for Pulsar enables applications to manually manage cursors. When you use a reader to connect to a topic---rather than a consumer---you need to specify *which* message the reader begins reading from when it connects to a topic. When connecting to a topic, the reader interface enables you to begin with:

* The **earliest** available message in the topic
* The **latest** available message in the topic
* Some other message between the earliest and the latest. If you select this option, you'll need to explicitly provide a message ID. Your application will be responsible for "knowing" this message ID in advance, perhaps fetching it from a persistent data store or cache.

The reader interface is helpful for use cases like using Pulsar to provide [effectively-once](https://streaml.io/blog/exactly-once/) processing semantics for a stream processing system. For this use case, it's essential that the stream processing system be able to "rewind" topics to a specific message and begin reading there. The reader interface provides Pulsar clients with the low-level abstraction necessary to "manually position" themselves within a topic.

![The Pulsar consumer and reader interfaces](/assets/pulsar-reader-consumer-interfaces.png)

> ### Non-partitioned topics only
> The reader interface for Pulsar cannot currently be used with [partitioned topics](concepts-messaging.md#partitioned-topics).

Here's a Java example that begins reading from the earliest available message on a topic:

```java

import org.apache.pulsar.client.api.Message;
import org.apache.pulsar.client.api.MessageId;
import org.apache.pulsar.client.api.Reader;

// Create a reader on a topic and for a specific message (and onward)
Reader<byte[]> reader = pulsarClient.newReader()
    .topic("reader-api-test")
    .startMessageId(MessageId.earliest)
    .create();

while (true) {
    Message message = reader.readNext();

    // Process the message
}

```

To create a reader that will read from the latest available message:

```java

Reader<byte[]> reader = pulsarClient.newReader()
    .topic(topic)
    .startMessageId(MessageId.latest)
    .create();

```

To create a reader that will read from some message between earliest and latest:

```java

byte[] msgIdBytes = // Some byte array
MessageId id = MessageId.fromByteArray(msgIdBytes);
Reader<byte[]> reader = pulsarClient.newReader()
    .topic(topic)
    .startMessageId(id)
    .create();

```

