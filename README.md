Kafka Plugins for Akka Persistence
==================================

This project features [storage plugins](http://doc.akka.io/docs/akka/2.3.4/scala/persistence.html#storage-plugins) for [Akka Persistence](http://doc.akka.io/docs/akka/2.3.4/scala/persistence.html) that are backed by [Apache Kafka](http://kafka.apache.org/).

- The [journal plugin](#journal-plugin) is implemented and passes the [TCK](https://github.com/krasserm/akka-persistence-testkit).
- The [snapshot store plugin](#snapshot-store-plugin) is not implemented yet (but coming soon).

Dependency
----------

To include the Kafka storage plugins into your `sbt` project, add the following lines to your `build.sbt` file:

    resolvers += "krasserm at bintray" at "http://dl.bintray.com/krasserm/maven"

    libraryDependencies += "com.github.krasserm" %% "akka-persistence-kafka" % "0.1"

Version `0.1` depends on Kafka 0.8.1.1, Akka 2.3.4 and Scala 2.10.

Journal plugin
--------------

### Activation 

To activate the journal plugin, add the following line to `application.conf`:

    akka.persistence.journal.plugin = "kafka-journal"

This will run the journal with default settings and connect to a Zookeeper instance running on `localhost:2181`. The Zookeeper connect string can be customized with the `kafka-journal.zookeeper.connect` configuration key (see also [Kafka cluster](#kafka-cluster) section).

### Use cases 

- Akka Persistence [journal plugin](http://doc.akka.io/docs/akka/2.3.4/scala/persistence.html#journal-plugin-api) (obvious).
- Event publishing to [user-defined topics](#user-defined-topics).
- Event consumption from user-defined topics by [external consumers](#external-consumers).  

### Usage hints

Kafka does not permanently store log entries but rather deletes them after a configurable _retention time_ which defaults to 7 days in Kafka 0.8.x. Therefore, applications need to take snapshots of their persistent actors at intervals that are smaller than the configured retention time (for example, every 3 days). This ensures that persistent actors can always be recovered successfully. 

Alternatively, the retention time can be set to a maximum value so that Kafka will never delete old entries. In this case, all events written by a single persistent actor must fit on a single node. This is a limitation of the current implementation which may be removed in later versions. However, this limitation is likely not relevant when running Kafka with default (or comparable) retention times and taking snapshots.

### Journal topics

For each persistent actor, the plugin creates a Kafka topic where the topic name equals the actor's `persistenceId`. Events published to these topics are serialized `akka.persistence.PersistentRepr` objects (see [journal plugin API](http://doc.akka.io/docs/akka/2.3.4/scala/persistence.html#journal-plugin-api)). Serialization of `PersistentRepr` objects can be [customized](http://doc.akka.io/docs/akka/2.3.4/scala/persistence.html#custom-serialization). Journal topics are mainly intended for internal use (for recovery of persistent actors) but can also be [consumed externally](#external-consumers). 

### User-defined topics

The journal plugin can also publish events to user-defined topics. By default, all events generated by all persistent actors are published to a single `events` topic. This topic is intended for [external consumption](#external-consumers) only. Events published to user-defined topics are serialized `Event` objects

```scala
package akka.persistence.kafka

/**
 * Event published to user-defined topics.
 *
 * @param persistenceId Id of the persistent actor that generates event `data`.
 * @param sequenceNr Sequence number of the event.
 * @param data Event data generated by a persistent actor.
 */
case class Event(persistenceId: String, sequenceNr: Long, data: Any)
```

where `data` is the actual event written by a persistent actor (by calling `persist` or `persistAsync`), `sequenceNr` is the event's sequence number and `persistenceId` the id of the persistent actor. By default, `Event` objects are serialized with `DefaultEventEncoder` (using Java serialization) which is configured in the [reference configuration](#reference-configuration) as follows:
                                                                                                                                                                                                                
    kafka-journal.event.producer.serializer.class = "akka.persistence.kafka.DefaultEventEncoder"

Applications can implement and configure their own `kafka.serializer.Encoder` to customize `Event` serialization. 

For publishing events to user-defined topics the journal plugin uses an `EventTopicMapper`: 

```scala
package akka.persistence.kafka

/**
 * Defines a mapping of events to user-defined topics.
 */
trait EventTopicMapper {
  /**
   * Maps an event to zero or more topics.
   *
   * @param event event to be mapped.
   * @return a sequence of topic names.
   */
  def topicsFor(event: Event): immutable.Seq[String]
}
```

The default mapper is `DefaultEventTopicMapper` which maps all events to the `events` topic. It is configured in the [reference configuration](#reference-configuration) as follows: 

    kafka-journal.event.producer.topic.mapper.class = "akka.persistence.kafka.DefaultEventTopicMapper"

To customize the mapping of events to user-defined topics, applications can implement and configure a custom `EventTopicMapper`. For example, in order to publish

- events from persistent actor `a` to topics `topic-a-1` and `topic-a-2` and
- events from persistent actor `b` to topic `topic-b` 

and to turn of publishing of events from all other actors, one would implement the following `ExampleEventTopicMapper`

```scala
package akka.persistence.kafka.example

class ExampleEventTopicMapper extends EventTopicMapper {
  def topicsFor(event: Event): Seq[String] = event.persistenceId match {
    case "a" => List("topic-a-1", "topic-a-2")
    case "b" => List("topic-b")
    case _   => Nil
  }
```

and configure it in `application.conf`:

    kafka-journal.event.producer.topic.mapper.class = "akka.persistence.kafka.example.ExampleEventTopicMapper"

To turn off publishing events to user-defined topics, the `EmptyEventTopicMapper` should be configured.

    kafka-journal.event.producer.topic.mapper.class = "akka.persistence.kafka.EmptyEventTopicMapper"

### External consumers

The following example shows how to consume `Event`s from a user-defined topic with name `topic-a-2` (see [previous](#user-defined-topics) example) using Kafka's [high-level consumer API](http://kafka.apache.org/documentation.html#highlevelconsumerapi):
  
```scala
import java.util.Properties

import akka.persistence.kafka.{DefaultEventDecoder, Event}

import kafka.consumer.{Consumer, ConsumerConfig}
import kafka.serializer.StringDecoder

val props = new Properties()
props.put("group.id", "consumer-1")
props.put("zookeeper.connect", "localhost:2181")
// ...

val consConn = Consumer.create(new ConsumerConfig(props))
val streams = consConn.createMessageStreams(Map("topic-a-2" -> 1),
  keyDecoder = new StringDecoder, valueDecoder = new DefaultEventDecoder)

streams("topic-a-2")(0).foreach { mm =>
  val event: Event = mm.message
  println(s"consumed ${event}")
}
```  

Applications may also consume serialized `PersistentRepr` objects from journal topics and deserialize them with Akka's serialization extension: 

```scala
import java.util.Properties

import akka.actor._
import akka.persistence.PersistentRepr
import akka.serialization.SerializationExtension

import com.typesafe.config.ConfigFactory

import kafka.consumer.{Consumer, ConsumerConfig}
import kafka.serializer.{DefaultDecoder, StringDecoder}

val props = new Properties()
props.put("group.id", "consumer-2")
props.put("zookeeper.connect", "localhost:2181")
// ...

val system = ActorSystem("example")
val extension = SerializationExtension(system)

val consConn = Consumer.create(new ConsumerConfig(props))
val streams = consConn.createMessageStreams(Map("a" -> 1),
  keyDecoder = new StringDecoder, valueDecoder = new DefaultDecoder)

streams("a")(0).foreach { mm =>
  val persistent: PersistentRepr = extension.deserialize(mm.message, classOf[PersistentRepr]).get
  println(s"consumed ${persistent}")
}
```

There are many other libraries that can be used to consume (event) streams from Kafka topics, such as [Spark Streaming](http://spark.apache.org/docs/latest/streaming-programming-guide.html), to mention only one example.    

### Implementation notes

- During initialization, the journal plugin fetches cluster metadata from Zookeeper which may take a few seconds.
- The journal plugin always writes `PersistentRepr` entries to partition 0 of journal topics. This ensures that all events written by a single persistent actor are stored in correct order. Later versions of the plugin may switch to a higher partition after having written a configurable number of events to the current partition. 
- The journal plugin distributes `Event` entries to all available partitions of user-defined topics. The partition key is the event's `persistenceId` so that a partial ordering of events is preserved when consuming events from user-defined topics. In other words, events written by a single persistent actor are always consumed in correct order but the relative ordering of events from different persistent actors is not defined.  

### Current limitations

- The journal plugin does not support features that have been deprecated in Akka 2.3.4 (channels and single event deletions).
- Range deletions are not persistent (which may not be relevant for applications that configure Kafka with reasonably small retention times).

### Example source code

The complete source code of all examples from previous sections is in [Example.scala](https://github.com/krasserm/akka-persistence-kafka/blob/master/src/test/scala/akka/persistence/kafka/example/Example.scala), the corresponding configuration in [example.conf](https://github.com/krasserm/akka-persistence-kafka/blob/master/src/test/resources/example.conf).

Snapshot store plugin
---------------------

Not implemented yet.

Kafka
-----

### Kafka cluster

To connect to an existing Kafka cluster, an application must set a value for the `kafka-journal.zookeeper.connect` key in its `application.conf`:  

    kafka-journal.zookeeper.connect = "<host1>:<port1>,<host2>:<port2>,..."

If you want to run a Kafka cluster on a single node, you may find [this article](http://www.michael-noll.com/blog/2013/03/13/running-a-multi-broker-apache-kafka-cluster-on-a-single-node/) useful.

### Test server

Applications may also start a single Kafka and Zookeeper instance with the `TestServer` class.

```scala
import akka.persistence.kafka.server.TestServer

// start a local Kafka and Zookeeper instance
val server = new TestServer() 

// use the local instance
// ...

// and stop it
server.stop()
```

The `TestServer` configuration can be customized with the `test-server.*` configuration keys (see [reference configuration](#reference-configuration) for details).

Reference configuration
-----------------------

    kafka-journal {
    
      # FQCN of the Kafka journal plugin
      class = "akka.persistence.kafka.journal.KafkaJournal"
    
      # Dispatcher for the plugin actor
      plugin-dispatcher = "akka.persistence.dispatchers.default-plugin-dispatcher"
    
      # Dispatcher for message replay
      replay-dispatcher = "akka.persistence.dispatchers.default-replay-dispatcher"
    
      # The partition to use when publishing to and consuming from journal topics.
      partition = 0
    
      consumer {
        # -------------------------------------------------------------------
        # Simple consumer configuration (used for recovery of persistent
        # actors).
        #
        # See http://kafka.apache.org/documentation.html#consumerconfigs
        # See http://kafka.apache.org/documentation.html#simpleconsumerapi
        # -------------------------------------------------------------------
    
        socket.timeout.ms = 30000
    
        socket.receive.buffer.bytes = 65536
    
        fetch.message.max.bytes = 1048576
      }
    
      producer {
        # -------------------------------------------------------------------
        # PersistentRepr producer (to journal topics) configuration.
        #
        # See http://kafka.apache.org/documentation.html#producerconfigs
        #
        # The metadata.broker.list property is set dynamically by the journal.
        # No need to set it here.
        # -------------------------------------------------------------------
    
        # DO NOT CHANGE!
        producer.type = "sync"
    
        # DO NOT CHANGE!
        request.required.acks = 1
    
        # DO NOT CHANGE!
        partitioner.class = "akka.persistence.kafka.StickyPartitioner"
    
        # DO NOT CHANGE!
        key.serializer.class = "kafka.serializer.StringEncoder"
    
        # Add further Kafka producer settings here, if needed.
        # ...
      }
    
      event.producer {
        # -------------------------------------------------------------------
        # Event producer (to user-defined topics) configuration.
        #
        # See http://kafka.apache.org/documentation.html#producerconfigs
        # -------------------------------------------------------------------
    
        producer.type = "sync"
    
        request.required.acks = 0
    
        topic.mapper.class = "akka.persistence.kafka.DefaultEventTopicMapper"
    
        serializer.class = "akka.persistence.kafka.DefaultEventEncoder"
    
        key.serializer.class = "kafka.serializer.StringEncoder"
    
        # Add further Kafka producer settings here, if needed.
        # ...
      }
    
      zookeeper {
        # -------------------------------------------------------------------
        # Zookeeper client configuration
        # -------------------------------------------------------------------
    
        connect = "localhost:2181"
    
        session.timeout.ms = 6000
    
        connection.timeout.ms = 6000
    
        sync.time.ms = 2000
      }
    }
    
    test-server {
      # -------------------------------------------------------------------
      # Test Kafka and Zookeeper server configuration.
      #
      # See http://kafka.apache.org/documentation.html#brokerconfigs
      # -------------------------------------------------------------------
    
      zookeeper {
    
        port = 2181
    
        dir = "data/zookeeper"
      }
    
      kafka {
    
        broker.id = 1
    
        num.partitions = 2
    
        port = 6667
    
        log.dirs = data/kafka
    
        log.index.size.max.bytes = 1024
      }
    }