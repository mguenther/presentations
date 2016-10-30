# Apache [Kafka]()
## for Fast Data Pipelines

**Markus Günther**

Freelance Software Engineer / Architect

[markus.guenther@gmail.com](mailto:markus.guenther@gmail.com) | [habitat47.de](http://www.habitat47.de) | [@mguenther](https://twitter.com/mguenther)

---

## Enter Apache [Kafka]()

----

### [Key Innovations]()<!-- .element: class="fragment" data-fragment-index="0" -->

* Messages are acknowledged in order<!-- .element: class="fragment" data-fragment-index="1" -->
* Messages are persisted for days / weeks<!-- .element: class="fragment" data-fragment-index="2" -->
* Consumers can manage their offsets<!-- .element: class="fragment" data-fragment-index="3" -->

----

### General Arrangement of a [Kafka]() Cluster

![Architectural-Overview](./kafka-architectural_overview.svg)

----

### What is the role of [ZooKeeper]()?

![ZooKeeper-Coordinates-Cluster](./kafka-zookeeper_for_coordination.svg)

----

### Relationships between [Producers](), [Consumers](), [Topics]()

![Relationships-Producers-Consumers-Topics](./kafka-relationships_between_producers_consumers_topics.svg)

----

### [Topics]() and [Partitions]() are replicated

![Topics-And-Partitions-Are-Replicated](./kafka-topics_and_partitions_are_distributed.svg)

----

### [Append-Only]() Logs Consumed [Sequentially]()

![Anatomy-Of-The-Kafka-Log](./kafka-anatomy_of_the_kafka_log.svg)

----

### So, how fast is this thing?

![Performance-Comparison](./kafka-producer_and_consumer_performance.png)

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

## [Kafka]() Gotchas and Best Practices

---

## No Inherent Serialization Mechanism

----

### [#1:]() Use a consistent [serialization mechanism]()

----

### The four stages of serializing data

1. Use the built-in serialization of your language
2. Use language-agnostic formats
3. Invent your own serialization on top of a language-agnostic one
4. Schema and documentation for the win!

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

---

## Potential of [Huge]() Data Loss at [Broker](), [Producer]() and [Consumer]()

----

### [#4:]() Disable unclean leader election

----

![Unclean-Leader-Election-1](./kafka-unclean_leader_election_1.svg)

----

![Unclean-Leader-Election-2](./kafka-unclean_leader_election_2.svg)

----

![Unclean-Leader-Election-3](./kafka-unclean_leader_election_3.svg)

----

![Unclean-Leader-Election-4](./kafka-unclean_leader_election_4.svg)

----

![Unclean-Leader-Election-5](./kafka-unclean_leader_election_5.svg)

----

### [#5:]() Monitor the size of in-sync-replica sets

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

### [#9:]() Limit the number of topics and partitions

----

### [#10:]() Disable automatic topic creation

----

### [#11:]() Use a consistent hashing scheme for keyed messages

---

### Takeaway

* Battle-proven technology for fast data pipelines
* Easy-to-use API
* Characteristics of a distributed commit log
  * ... not a traditional message broker
* No guarantee of message delivery
* No reliable solution for multi-master replication
* Monitoring? Enterprise? [Confluent](http://www.confluent.io/)!

---

# Thank you!

## Any Questions?

---

### Sources

#### Blogs & Articles

* [Kafka: A Distributed Messaging System for Log Processing](http://research.microsoft.com/en-us/um/people/srikanth/netdb11/netdb11papers/netdb11-final12.pdf) (Kreps et al.)
* [Jepsen: Kafka](https://aphyr.com/posts/293-jepsen-kafka) (Kyle Kingsbury)
* [Schema Evolution in Avro, Protocol Buffers and Thrift](https://martin.kleppmann.com/2012/12/05/schema-evolution-in-avro-protocol-buffers-thrift.html) (Martin Kleppmann)

#### Talks

* [No Data Loss Pipeline with Apache Kafka](http://www.slideshare.net/JiangjieQin/no-data-loss-pipeline-with-apache-kafka-49753844) (Jiangjie Qin)