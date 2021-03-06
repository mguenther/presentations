# Apache [Kafka]()
## for Fast Data Pipelines

**Markus Günther**

Freelance Software Engineer / Architect

<small>[markus.guenther@gmail.com](mailto:markus.guenther@gmail.com) | [habitat47.de](http://www.habitat47.de) | [@markus_guenther](https://twitter.com/markus_guenther)</small>

---

### How [data pipelines]() start off

![How-Data-Pipelines-Start-Off-1](./kafka-how_data_pipelines_start_1.svg)

Note: So, let's start off with an example of how data pipelines are typically constructed in the wild. We have some sort of backend, call it source, and some client that connects to the backend to fetch the data off of it that it requires to operate.

----

### How [data pipelines]() start off (cont'd)

![How-Data-Pipelines-Start-Off-2](./kafka-how_data_pipelines_start_2.svg)

Note: As the usefulness of the data grows, the number of clients will also increase. So let's just hook them up to the same source and re-use the data, shall we? Hopefully we had the time to implement client-specific interfaces instead of re-using existing ones.

----

### How [data pipelines]() start off (cont'd)

![How-Data-Pipelines-Start-Off-3](./kafka-how_data_pipelines_start_3.svg)

Note: At some point in time we need other data at the client-side as well. So we simply introduce other sources to the system and connect the existing clients to them. We typically end up with an entangled mess like the picture shows, where each and every client depends on at least one of the source systems. In the worst case, the source systems have data-dependencies in betweem them, creating even more mess to deal with in the long run.

----

### How [data pipelines]() start off (cont'd)

![How-Data-Pipelines-Start-Off-4](./kafka-how_data_pipelines_start_4.svg)

Note: It is no surprise that such a system is brittle and fragile. Depending on your design, clients may be coupled by interfaces that are too broad (rigidity). Data-dependencies hinder evolution of your data (schemas).

----

### Decoupling [data pipelines]()

![Decoupling-Data-Pipelines-1](./kafka-decoupling_data_pipeline_1.svg)

Note: So how should we go about designing data-intensive applications? Let's take a look at this simple illustration. Source systems push their raw data into a messaging system and a client that is interested in this data may read it from the messaging system.

----

### Decoupling [data pipelines]() (cont'd)

![Decoupling-Data-Pipelines-2](./kafka-decoupling_data_pipeline_2.svg)

Note: Integrating more clients in this scenario becomes easy, since there is no direct dependency on the producing systems. This simple design already shows that we can decouple the location from where data originates from the location where data is processed further. And perhaps the most important thing from the perspective of designer: This leads to clean interfaces that are client-oriented by default.

----

### Decoupling [data pipelines]() (cont'd)

![Decoupling-Data-Pipelines-3](./kafka-decoupling_data_pipeline_3.svg)

Note: Let's take this one step further and have a look at what might become of our simple data pipeline. We may end up with more than a single producing system that pushes data into the pipeline. In this case, we have two sources that emit raw data - in whatever form - and a couple of client-side systems that fetch data and perform different kinds of processing steps on them. For example, the data augmentation client reads data of the raw data queue and augments it using data it already has or is able to infer from the given data. And after it has done that, it persists the augmented data in a local database. The data aggregation on the other hand is a transformational client that collects and groups raw data and emits the aggregates into another queue from which another client reads the aggregates to generate reports. Last but not at least, we have a real-time analytics engine that also consumes data from the raw data queue, transforming stream-like data and pushing it to a results queue from which a dashboard client is able to read its data. We already have that all of these client-side systems have different requirements wrt. the data pipeline. An efficient and scalable messaging infrastructure is a cornerstone of this kind of architecture.

----

### What drives the [design]() of a [data pipeline]()?

* High throughput<!-- .element: class="fragment" data-fragment-index="1" -->
* Horizontal scalability<!-- .element: class="fragment" data-fragment-index="2" -->
* High availability<!-- .element: class="fragment" data-fragment-index="3" -->
* No data loss<!-- .element: class="fragment" data-fragment-index="4" -->
* Satisfies (soft) real-time constraints<!-- .element: class="fragment" data-fragment-index="5" -->
* Enforces structure of data<!-- .element: class="fragment" data-fragment-index="6" -->
* Single source of truth<!-- .element: class="fragment" data-fragment-index="7" -->

Note: So what actually drives the design of a data pipeline? For starters, data pipelines must be able to cope with huge volumes of data, so a high throughput is beneficial in that case. But note that there is an inherent trade-off between throughput and latency. Optimizing a data pipeline for high throughput will increase its latency, while optimizing for low latency (like in the real-time analytics use case) is likely to decrease throughput. A data pipeline should be able to scale horizontally without much effort. It should be highly available as well. It is crucial that the system is able to operate even if parts of it are unavailable due to network partitions or failure of individual systems. And of course we also want that the system does not lose any data - even in case of failures. To allow seamless data evolution (and thus client evolution as well), exchanged data should exhibit a well-defined structure using a schema that supports evolvability. And lastly, clients expect that the data they read off from a queue is a fact within the boundaries of the system.

---

## Enter Apache [Kafka]()

----

### [Overview]()

* Apache project, originated at LinkedIn<!-- .element: class="fragment" data-fragment-index="1" -->
* Distributed publish-subscribe messaging system<!-- .element: class="fragment" data-fragment-index="2" -->
* Supports both queue and topic semantics<!-- .element: class="fragment" data-fragment-index="3" -->

----

### [Point-to-Point]() Messaging

##### (Queue Semantics)

![Point-to-Point-Messaging](./kafka-messaging_point_to_point.svg)

Note: Let us quickly recap some fundamentals before diving into Kafka. I mentioned that Kafka supports both queue and topic semantics. But what does this mean? The illustration on this slide shows a simple point-to-point based messaging system. In this system, we have at least a single sender that emits data into a queue. The messaging systems routes the data to a specific receiver - hence the name point-to-point messaging.

----

### [Publish-Subscribe]() Messaging

##### (Topic Semantics)

![Publish-Subscribe-Messaging](./kafka-messaging_publish_subscribe.svg)

Note: In a topic-based system we have at least one publisher that emits data into a so-called topic. Clients subscribe to that topic and read the data off from it. In this scenario, a single message can be read from the all clients that subscribed to the particular topic.

----

### [Overview]()

* Apache project, originated at LinkedIn
* Distributed publish-subscribe messaging system
* Supports both queue and topic semantics
* Designed for real-time processing of events<!-- .element: class="fragment" data-fragment-index="1" -->
* Has at-least-once messaging semantics<!-- .element: class="fragment" data-fragment-index="2" -->
* No integration with JMS<!-- .element: class="fragment" data-fragment-index="3" -->
* Written in Scala<!-- .element: class="fragment" data-fragment-index="4" -->

----

### [Key Innovations]()

* Messages are acknowledged in order<!-- .element: class="fragment" data-fragment-index="1" -->
* Messages are persisted for days / weeks<!-- .element: class="fragment" data-fragment-index="2" -->
* Consumers can manage their offsets<!-- .element: class="fragment" data-fragment-index="3" -->

Note: Although Kafka has adopted many of the things that traditional approaches to messaging do, it differs in some areas. For instance, Kafka requires that consumed messages are acknowledged in order. Kafka consumers are not able to acknowledge messages out-of-order. This removes the need to track acknowledges per-message, so less overhead on the Kafka side. At the same time, consumers read from their subscribed topics like they would read from a file. Consumed messages are retained until a configurable retention policy says otherwise. Compare this to traditional messaging solutions with queue semantics: Once a message has been consumed and acknowledged, it becomes unavailable. Consumers are able to manage their current message offset themselves. It is even possible to manage these offsets outside of Kafka altogether. This isolates consumers from each other.

----

### General Arrangement of a [Kafka]() Cluster

![Architectural-Overview](./kafka-architectural_overview.svg)

Note: So, let's look at the system architecture of a Kafka cluster. Producers are applications that emit messages to topics managed by Kafka brokers. Typically, a Kafka cluster is comprised of a couple of Kafka brokers. The brokers manage messages on a per-topic and per-partition base - we will get to that in a couple of minutes - and write messages to disk. Consumers on the other hand are organized into consumer groups. Consumers subscribe to the topic of interest and read from that topic sequentially.

----

### What is the role of [ZooKeeper]()?

![ZooKeeper-Coordinates-Cluster](./kafka-zookeeper_for_coordination.svg)

Note: So you've seen the ZooKeeper node at the top of the last slide. What is the purpose of ZooKeeper in the system? Well, ZooKeeper handles a variety of things inside a Kafka cluster. For starters, topics are generally replicated and require a leader which manages the replication. A leader is simply a Kafka broker with that role. Electing a leader out of all the Kafka brokers is one of the things ZooKeeper does. ZooKeeper also knows about cluster membership, not only for brokers, but for consumers as well. ZooKeeper stores the configuration of topics, including the number of partitions for a particular topic or its replication factor. As of 0.9.0, some things were deprecated, some other things introduced. Up until version 0.9.0 of Kafka, ZooKeeper solely managed consumer offsets. This enforced a dependency between consumers and ZooKeeper, which is no longer present. Offsets are stored in a meta-topic at the brokers, but may be synchronized with consumer offsets in ZooKeeper if you still require them. Two interesting features made their way into the 0.9.0 release: The first is quotas. With Kafka 0.9.0 you can define read-quotas on a per-topic basis for consumers. The second is ACLs. Same as with quotas, you can define access restrictions on a per-topic basis for consumers.

----

### Relationships between [Producers](), [Consumers](), [Topics]()

![Relationships-Producers-Consumers-Topics](./kafka-relationships_between_producers_consumers_topics.svg)

Note: A topic is comprised of partitions. The number of partitions is configurable. There must be at least one partition for a particular topic. Producers emit messages to a topic; in this case, the assignment from message to topic-partition is done in a round-robin fashion. Producers may also emit so called keyed messages which route the message to a particular topic. Kafka guarantees strict ordering within a partition. This is a useful feature if consuming applications rely on the ordering of consumed messages. For example, think of event-sourced applications which typically have to maintain some form of causal-consistency between events for the same aggregate. Consumers are organized into a consumer group. If multiple consumers of a consumer group are subscribed to a topic, then the consumer group manages the assigned from messages per partition to a consumer to retain Kafka's ordering guarantees. A consumer may read from multiple partitions of a subscribed topic. A topic-partition is never shared between consumers, as this would break Kafka's ordering guarantees. Thus, the number of partitions is an important mechanism for the concurrent access of a topic, as it limits the number of consumers per consumer group that read data from the topic.

----

### [Topics]() and [Partitions]() are replicated

![Topics-And-Partitions-Are-Replicated](./kafka-topics_and_partitions_are_distributed.svg)

Note: Partitions are replicated between a configurable number of brokers of a Kafka cluster. Every topic-partition is assigned a leader. The leader is a Kafka broker which manages the replication for that particular topic-partition. The illustrations shows three brokers, partitions are for which the resp. broker is a leader are indicated using a red font. Partitions for which the resp. broker is a follower are indicated using a black font. When a producer emits a message to a leader, it replicates the same message to its followers and collects their acknowledgements. The number of acknowledgements necessary to indicate a success is configurable. The set of brokers that replicate a given message is known as the in-sync replica set. Consumers read from any of the brokers that hold the data.

----

### [Append-Only]() Logs Consumed [Sequentially]()

![Anatomy-Of-The-Kafka-Log](./kafka-anatomy_of_the_kafka_log.svg)

Note: Kafka brokers store messages per-partition in at least one segment file. A broker has an in-memory segment list that it keeps up-to-date. Messages are assigned a specific offset that increases linearly. The segment list keeps track of the offsets stored inside particular segments. Segment files have a configurable size. Messages are only ever appended to the segment. There is no random access involved at all when writing to a segment file. This, and the fact that Kafka uses zero-copy via the sendfile API, is one of the reasons why Kafka is able to achieve its incredible write performance (non-random I/O is much faster than random I/O).

----

### So, how fast is this thing?

![Performance-Comparison](./kafka-producer_and_consumer_performance.png)

Note: These charts are taken from the original paper written by Kreps et al. They showcase the performance on both the producer- and consumer-side in comparison to traditional messaging solutions like Apache ActiveMQ and RabbitMQ.

----

### The [KafkaProducer]() API

```scala
object SimpleProducer extends App {

  val props = new Properties
  props.put("bootstrap.servers", "127.0.0.1:9092")
  props.put("key.serializer",
    "org.apache.kafka.common.serialization.StringSerializer")
  props.put("value.serializer",
    "org.apache.kafka.common.serialization.StringSerializer")

  val producer = new KafkaProducer[String, String](props)

  (1 to 100).foreach(i => {
    val message = new ProducerRecord[String, String](
      "test", 
      i.toString, // key
      i.toString) // payload
    producer.send(message)
  })

  producer.close()
}

```

----

### The [KafkaConsumer]() API

```scala
object SimpleConsumer extends App {

  val props = new Properties
  props.put("bootstrap.servers", "localhost:9092")
  props.put("group.id", "scala-rhein-main-group")
  props.put("enable.auto.commit", "true")
  props.put("key.deserializer", 
    "org.apache.kafka.common.serialization.StringDeserializer")
  props.put("value.deserializer", 
    "org.apache.kafka.common.serialization.StringDeserializer")

  val consumer = new KafkaConsumer[String, String](props)
  consumer.subscribe(seqAsJavaList(List("test")))

  while (true) {
    val records = consumer.poll(100)
    JavaConversions
      .asScalaIterator(records.iterator)
      .foreach(record => 
        logger.info(s"offset=${record.offset}, key=${record.key}, value=${record.value}"))
  }
```

---

# Demo

---

## [Kafka]() Gotchas and Best Practices

---

## No Inherent Serialization Mechanism

Note: Kafka makes no assumptions on the used serialization mechanism. For a Kafka broker, messages are simply bytes written to disk. The KafkaProducer API comes with a minimal set of serializers and deserializers that are able to convert Strings to bytes and byte arrays to bytes.

----

### [#1:]() Use a consistent [serialization mechanism]()

----

### The [four stages]() of serializing data

1. Use the built-in serialization of your language<!-- .element: class="fragment" data-fragment-index="1" -->
2. Use language-agnostic formats<!-- .element: class="fragment" data-fragment-index="2" -->
3. Invent your own serialization on top of a language-agnostic one<!-- .element: class="fragment" data-fragment-index="3" -->
4. Schema and documentation for the win!<!-- .element: class="fragment" data-fragment-index="4" -->

----

#### Which [serialization framework]() should I use?

![Serialization-Comparison](./kafka-serialization_comparison.png)

----

#### [Apache Avro]() is suitable for streaming applications

* Schema representation in JSON or an IDL
* Supports the usual types
  * Primitive Types: boolean, int, long, string, etc.
  * Complex Types: Record, Enum, Array, Union, Map, Fixed
* Data is (de-)serialized using its schema
* Compact binary output

----

### How does a schema look like in [Apache Avro]()?

```json
{
  "namespace": "com.mgu.kafkaexamples.avro",
  "type": "record",
  "name": "Message",
  "fields": [
    {"name": "messageId", "type": "string"},
    {"name": "text", "type": "string"}
  ]
}
```

----

### ... or using an [IDL]()

```
@namespace("com.mgu.kafkaexamples.avro")
protocol SimpleExample {
  record Message {
    string messageId;
    string name;
  }
}
```

----

A JSON-based representation of ```Message```

```json
{
  "messageId": "f9ae42fc",
  "text": "Hello!"
}
```

is compiled to this using Apache Avro

![Avro-Encoded-Message](./kafka-avro_encoded_message.svg)

Note: We can see that there are no pointers to the data attribute in the schema serialized with the message. This means, that the Avro deserializer must read the binary using the same schema that it has been written with. So, how does Avro support schema evolution? The Avro deserializer optionally accepts two schemas: The one with which the data has been written and the one the current version of the deserializer understands. Using both schema and a number of resolution rules, Avro is able to cope with different schema versions, given the schemas are compatible to each other. Still, the schema of the producer must be known when deserializing data, which sets the requirements for a schema registry.

----

### How can I include [Avro]() in my Scala project?

```scala
libraryDependencies ++= Seq(
  ...
  "org.apache.avro" % "avro" % "1.6.3",
  "com.twitter" %% "bijection-avro" % "0.9.2")

Seq(sbtavro.SbtAvro.avroSettings : _*)

javaSource in sbtavro.SbtAvro.avroConfig <<= (sourceDirectory in Compile)(_ / "generated")

(stringType in avroConfig) := "String"
```

----

### Use [```bijection-avro```]() for bijective mappings

```scala
def serialize(payload: T): Option[Array[Byte]] = 
  try {
    Some(SpecificAvroCodecs.toBinary[T].apply(payload))
  } catch {
    case ex: Exception => None
  }
```

... and vice versa ...


```scala
def deserialize(payload: Array[Byte]): Option[T] =
  try {
    Some(SpecificAvroCodecs.toBinary[T].invert(payload).get)
  } catch {
    case ex: Exception => None
  }
```

---

## Kafka employs [at-least-once]() semantics wrt. messaging

----

### [#2:]() Use idempotent message handlers if possible

----

### [#3:]() Use a de-duplication filter if messages are non-idempotent

Note: This requires application-specific message IDs as part of the messages. Message offsets are no reliable IDs! Example: Suppose the consumer sends an e-mail on every read message. Sending the same e-mail twice degrades the customer-experience.

---

## Potential of [Huge]() Data Loss at [Broker](), [Producer]() and [Consumer]()

Note: Data loss at the broker might occur due to unclean leader election. Data loss at the producer might occur due to a full buffer or retry exhaustion. Data loss at the consumer might occur if the consumer auto-commits read message offsets and the consumer dies while actually processing the messages afterwards.

----

> "For a topic with replication factor N, Kafka can tolerate up to N-1 server failures without losing any messages committed to the log."

Note: This quote is directly taken from the Kafka documentation. It is a bold statement, don't you think? And it is utter nonsense. Most symptoms by the way give the guarantee that no data loss occurs if at most (N-1)/2 brokers crash. Even in this case, we deal with probabilities hear. But let us see why the above statement is problematic.

----

![Unclean-Leader-Election-1](./kafka-unclean_leader_election_1.svg)

Note: Assume we have an in-sync replica set with three Kafka brokers. The leader replicates messages to the followers, the followers acknowledge the replicated messages.

----

![Unclean-Leader-Election-2](./kafka-unclean_leader_election_2.svg)

Note: What happens if we have a network partition that isolates the right-most broker? Well, the in-sync replica set decreases by one broker, but it will continue working. So, messages accepted by the leader are replicated to the single follower, and as in the previous illustration, the follower sends back its acknowledgment.

----

![Unclean-Leader-Election-3](./kafka-unclean_leader_election_3.svg)

Note: Suppose now that the network partition isolates not only the right-most broker, but all of the followers of the in-sync-replica set. Per default, the in-sync replica is allowed to shrink to just the leader to keep the system available. So the leader will accept new messages and write them to its local log. But none of those messages are replicated any longer.

----

![Unclean-Leader-Election-4](./kafka-unclean_leader_election_4.svg)

Note: Suppose now that while ZooKeeper still maintains a connection to the followers, the leader dies and is no longer available. ZooKeeper detects a timeout for the leader and triggers a re-election. The broker in the middle is the new leader after the election concludes. Hence, from this point in time foreward, it manages the replication. Since the in-sync replica set went down to just a single broker, we call this kind of leader election an unclean leader election. But why?

----

![Unclean-Leader-Election-5](./kafka-unclean_leader_election_5.svg)

Note: As the old leader comes back online it rejoins the cluster and is now a follower for the same topic-partition. But its local log might have diverged significantly from the logs of the other brokers. Also note that the old leader is unable to re-emit the messages that the other brokers have not seen so far due to the network partition, since this would break the strict ordering per partition which Kafka guarantees. If you think is too academic: This experiment was conducted by Kyle Kingsbury as part of his Jepsen-series. He basically showed that an unclean leader election can cause an arbitrary amount of data loss. The link to the blog is in the references section at the end of the talk. It is a highly recommended read, as well as all the other installments of Kyle's Jepsen-series.

----

### [#4:]() Disable unclean leader election
	

----


### [#5:]() Monitor the size of in-sync replica sets

Note: If you notice that the in-sync replica sets goes down to a criticial amount, raise a warning so that your DevOps is able to act upon.

----

### [#6:]() Commit consumer offsets manually

---

## Mirroring considered dangerous

----


### [#7:]() Do not use mirroring for disaster recovery

----

### [Mirroring]() does not preserve offsets

![MirrorMaker](./kafka-mirrormaker.svg)

----

### [#8:]() Do not use mirroring for a chain-of-replication

---

## Going into production

----

### [#9:]() Limit the number of topics and partitions to < 1000

Note: As the number of topics and partitions grows, the number of random I/O increases as well. This employs some kind of soft-limit on the number of topics a Kafka cluster is able to handle without any performance degradation. Resources show that exceeding 1000 topics has an operational impact on the cluster. This constrains the design space of Kafka-powered applications: For instance, Kafka is not a good fit for a multi-tenant application that has to support up to 10000 or more tenants where each tenant has her own topic.

----

### [#10:]() Disable automatic topic creation

Note: By default automatic topic creation is set to true. This way, you have no chance to see if you misconfigured your applications and the producers write to the wrong, non-existent topic.

----

### [#11:]() Use a consistent hashing scheme for keyed messages

Note: If you do use keyed messages to select a particular partition when writing and reading to a Kafka cluster, be sure to use a consistent hashing scheme that equally distributes keys to partitions. Also bear in mind that re-partitioning your topic will cause that messages go into different partitions afterwards. So plan ahead and partition topics slightly bigger than you currently require it.

---

### [Advanced]() Topics and [Outlook]()

* Lossless data pipelines<!-- .element: class="fragment" data-fragment-index="1" -->
* Messaging backbone for<!-- .element: class="fragment" data-fragment-index="2" -->
  * ... Microservices?<!-- .element: class="fragment" data-fragment-index="2" -->
  * ... Event-sourced systems?<!-- .element: class="fragment" data-fragment-index="2" -->
* Ecosystem grows<!-- .element: class="fragment" data-fragment-index="3" -->
  * Kafka Connect<!-- .element: class="fragment" data-fragment-index="3" -->
  * Kafka Streams<!-- .element: class="fragment" data-fragment-index="3" -->
  * Integrations with Hadoop, Storm, Samza, Flume, ...<!-- .element: class="fragment" data-fragment-index="3" -->

----

### [Takeaway]()

* Battle-proven technology for fast data pipelines<!-- .element: class="fragment" data-fragment-index="1" -->
* Easy-to-use API<!-- .element: class="fragment" data-fragment-index="2" -->
* Characteristics of a distributed commit log<!-- .element: class="fragment" data-fragment-index="3" -->
  * ... not a traditional message broker<!-- .element: class="fragment" data-fragment-index="3" -->
* No guarantee of message delivery<!-- .element: class="fragment" data-fragment-index="4" -->
* No reliable solution for multi-master replication<!-- .element: class="fragment" data-fragment-index="5" -->
* Monitoring? Enterprise?<!-- .element: class="fragment" data-fragment-index="6" --> [Confluent](http://www.confluent.io/)!<!-- .element: class="fragment" data-fragment-index="6" -->

---

# [Thank]() you!

## Any [Questions?]()

---

### [Sources]()

#### Blogs & Articles

* [Kafka: A Distributed Messaging System [...]](http://research.microsoft.com/en-us/um/people/srikanth/netdb11/netdb11papers/netdb11-final12.pdf) (Kreps et al.)
* [Jepsen: Kafka](https://aphyr.com/posts/293-jepsen-kafka) (Kingsbury)
* [Schema Evolution in Avro, Protocol Buffers [...]](https://martin.kleppmann.com/2012/12/05/schema-evolution-in-avro-protocol-buffers-thrift.html) (Kleppmann)

#### Talks

* [Introduction to Kafka and Zookeeper](http://www.slideshare.net/rahuldausa/introduction-to-kafka-and-zookeeper) (Jain)
* [Introduction to Apache Kafka](http://www.slideshare.net/jhols1/kafka-atlmeetuppublicv2) (Holoman)
* [No Data Loss Pipeline with Apache Kafka](http://www.slideshare.net/JiangjieQin/no-data-loss-pipeline-with-apache-kafka-49753844) (Qin)
* [From a Kafkaesque Story to the Promised Land](http://www.slideshare.net/ransilberman/from-a-kafkaesque-story-to-the-promised-land-23992927) (Silberman)

#### Books

* [Streaming Architecture](https://www.mapr.com/streaming-architecture-using-apache-kafka-mapr-streams) (Dunning et al.)