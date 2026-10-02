# Part 76: Apache Kafka Basics — Steps 751-760

## บทนำ: Apache Kafka

Apache Kafka คือ distributed event streaming platform ที่ใช้สำหรับ high-performance data pipelines, streaming analytics, data integration และ event-driven applications รองรับ millions of events per second

---

## Step 751: Kafka Architecture Concepts

```
Kafka Architecture:
┌─────────────────────────────────────────────────────┐
│                    Kafka Cluster                     │
│                                                      │
│  ┌─────────────────────────────────────────────┐    │
│  │              ZooKeeper / KRaft               │    │
│  │         (cluster coordination)              │    │
│  └─────────────────────────────────────────────┘    │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  │
│  │   Broker 1   │  │   Broker 2   │  │ Broker 3 │  │
│  │              │  │              │  │          │  │
│  │ Topic: orders│  │ Topic: orders│  │          │  │
│  │ Partition 0  │  │ Partition 1  │  │          │  │
│  │ [Leader]     │  │ [Leader]     │  │          │  │
│  │              │  │              │  │          │  │
│  │ Partition 1  │  │ Partition 0  │  │          │  │
│  │ [Replica]    │  │ [Replica]    │  │          │  │
│  └──────────────┘  └──────────────┘  └──────────┘  │
└─────────────────────────────────────────────────────┘

Producers ─────→ Topics ─────→ Consumers

Topic = ordered, partitioned log of records
Partition = append-only log
Offset = position in partition
Consumer Group = parallel consumption
```

### Key Concepts

```
Topic:
- Named stream of records
- Divided into partitions
- Retained for configurable time (default 7 days)
- Immutable (ไม่สามารถแก้ไข records เดิม)

Partition:
- Ordered sequence of records
- Each record has an offset (sequential ID)
- Records distributed by key hash (or round-robin)
- Replication เพื่อ fault tolerance

Consumer Group:
- Group of consumers ที่อ่าน topic เดียวกัน
- แต่ละ partition assigned ให้ consumer หนึ่งตัวในกลุ่ม
- Enables parallel consumption
- Offset per (group, partition) tracked automatically

Broker:
- Kafka server
- Holds subsets of partitions
- Leader/Follower model per partition
```

---

## Step 752: การติดตั้ง Kafka

### Docker Compose (แนะนำสำหรับ development)

```yaml
# docker-compose.yml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"
  
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "29092:29092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
  
  # Kafka UI
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    depends_on:
      - kafka
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
  
  # Schema Registry (สำหรับ Avro)
  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    depends_on:
      - kafka
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: 'kafka:29092'
```

### Kafka CLI Commands

```bash
# เริ่ม Kafka
docker-compose up -d

# สร้าง topic
kafka-topics --create \
  --bootstrap-server localhost:9092 \
  --replication-factor 1 \
  --partitions 3 \
  --topic orders

# List topics
kafka-topics --list --bootstrap-server localhost:9092

# Describe topic
kafka-topics --describe \
  --bootstrap-server localhost:9092 \
  --topic orders

# ส่งข้อความ
kafka-console-producer \
  --bootstrap-server localhost:9092 \
  --topic orders

# อ่านข้อความ (from beginning)
kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic orders \
  --from-beginning

# อ่านด้วย consumer group
kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic orders \
  --group my-consumer-group

# ดู consumer group status
kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --describe \
  --group my-consumer-group

# Reset offsets
kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --topic orders \
  --reset-offsets \
  --to-earliest \
  --execute

# Delete topic
kafka-topics --delete \
  --bootstrap-server localhost:9092 \
  --topic orders
```

---

## Step 753: Kafka Producer

```scala
// KafkaProducer.scala
// build.sbt:
// "org.apache.kafka" % "kafka-clients" % "3.6.0"

import org.apache.kafka.clients.producer._
import org.apache.kafka.common.serialization.{StringSerializer, LongSerializer}
import java.util.Properties
import scala.concurrent.{Future, ExecutionContext}
import scala.util.{Success, Failure}

object KafkaProducerExample {
  
  // สร้าง producer properties
  def createProducerProps(bootstrapServers: String): Properties = {
    val props = new Properties()
    
    // Required settings
    props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers)
    props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,   classOf[StringSerializer].getName)
    props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, classOf[StringSerializer].getName)
    
    // Reliability settings
    props.put(ProducerConfig.ACKS_CONFIG, "all")           // wait for all replicas
    props.put(ProducerConfig.RETRIES_CONFIG, "3")          // retry 3 ครั้ง
    props.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, "1000")  // 1s between retries
    
    // Idempotence (exactly-once semantics)
    props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true")
    
    // Performance settings
    props.put(ProducerConfig.BATCH_SIZE_CONFIG, "16384")       // 16KB batch
    props.put(ProducerConfig.LINGER_MS_CONFIG, "10")           // รอ 10ms เพื่อ batch
    props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, "33554432") // 32MB buffer
    props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy")// compress
    
    // Message size
    props.put(ProducerConfig.MAX_REQUEST_SIZE_CONFIG, "1048576") // 1MB max message
    
    props
  }
  
  def main(args: Array[String]): Unit = {
    val bootstrapServers = "localhost:9092"
    val topic = "orders"
    
    val producer = new KafkaProducer[String, String](createProducerProps(bootstrapServers))
    
    try {
      // ===== Fire-and-forget =====
      println("=== Fire-and-forget ===")
      val record1 = new ProducerRecord[String, String](
        topic,
        "order-key-1",
        """{"order_id":"O001","amount":999.99,"status":"pending"}"""
      )
      producer.send(record1) // ส่งแล้วไม่รอ response
      
      // ===== Synchronous Send =====
      println("=== Synchronous Send ===")
      val record2 = new ProducerRecord[String, String](
        topic,
        "order-key-2",
        """{"order_id":"O002","amount":499.99,"status":"pending"}"""
      )
      
      try {
        val metadata = producer.send(record2).get() // รอจนกว่าจะ acknowledge
        println(s"Sent to: topic=${metadata.topic()}, " +
                s"partition=${metadata.partition()}, " +
                s"offset=${metadata.offset()}")
      } catch {
        case ex: Exception =>
          println(s"Error sending: ${ex.getMessage}")
      }
      
      // ===== Asynchronous Send with Callback =====
      println("=== Async Send with Callback ===")
      for (i <- 1 to 10) {
        val customerId = s"C${i % 5}"
        val orderId = s"O${100 + i}"
        val amount = 100.0 * i
        
        val record = new ProducerRecord[String, String](
          topic,
          customerId,  // key ใช้สำหรับ partition assignment
          s"""{"order_id":"$orderId","customer_id":"$customerId","amount":$amount}"""
        )
        
        producer.send(record, (metadata: RecordMetadata, exception: Exception) => {
          if (exception == null) {
            println(s"✓ $orderId → partition=${metadata.partition()}, offset=${metadata.offset()}")
          } else {
            println(s"✗ $orderId failed: ${exception.getMessage}")
          }
        })
      }
      
      // ===== Specific Partition =====
      println("=== Send to Specific Partition ===")
      val vipOrder = new ProducerRecord[String, String](
        topic,
        0,             // partition 0
        "VIP-customer",
        """{"order_id":"VIP001","priority":"high","amount":50000.0}"""
      )
      val vipMeta = producer.send(vipOrder).get()
      println(s"VIP order sent to partition ${vipMeta.partition()}")
      
      // ===== Flush และ close =====
      producer.flush() // รอให้ buffer ว่าง
      
    } finally {
      producer.close() // ต้อง close เสมอ
    }
  }
}
```

---

## Step 754: Kafka Consumer

```scala
// KafkaConsumer.scala
import org.apache.kafka.clients.consumer._
import org.apache.kafka.common.serialization.StringDeserializer
import org.apache.kafka.common.TopicPartition
import java.util.Properties
import java.time.Duration
import scala.collection.JavaConverters._

object KafkaConsumerExample {
  
  def createConsumerProps(
    bootstrapServers: String, 
    groupId: String
  ): Properties = {
    val props = new Properties()
    
    // Required
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers)
    props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId)
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,   classOf[StringDeserializer].getName)
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, classOf[StringDeserializer].getName)
    
    // Offset management
    props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest") // earliest | latest | none
    props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false")   // manual commit
    
    // Performance
    props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, "500")       // max records per poll
    props.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, "1024")       // min 1KB per fetch
    props.put(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, "500")      // max 500ms wait
    
    // Session timeout
    props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, "30000")   // 30s timeout
    props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, "10000") // 10s heartbeat
    
    props
  }
  
  def main(args: Array[String]): Unit = {
    val bootstrapServers = "localhost:9092"
    val topic = "orders"
    val groupId = "order-processor-group"
    
    // ===== Basic Consumer =====
    println("=== Basic Consumer ===")
    
    val consumer = new KafkaConsumer[String, String](
      createConsumerProps(bootstrapServers, groupId)
    )
    
    try {
      // Subscribe to topic
      consumer.subscribe(java.util.Arrays.asList(topic))
      
      var running = true
      var processedCount = 0
      
      while (running && processedCount < 100) {
        // Poll for records (timeout 1 second)
        val records = consumer.poll(Duration.ofSeconds(1))
        
        if (!records.isEmpty) {
          println(s"Received ${records.count()} records")
          
          for (record <- records.asScala) {
            println(s"""
              |Topic: ${record.topic()}
              |Partition: ${record.partition()}
              |Offset: ${record.offset()}
              |Key: ${record.key()}
              |Value: ${record.value()}
              |Timestamp: ${record.timestamp()}
            """.stripMargin)
            
            processedCount += 1
            
            if (processedCount >= 10) running = false
          }
          
          // Manual commit after processing
          consumer.commitSync() // commit current offsets
        }
      }
      
    } finally {
      consumer.close()
    }
  }
}
```

---

## Step 755: Manual Offset Management

```scala
// OffsetManagement.scala
import org.apache.kafka.clients.consumer._
import org.apache.kafka.common.TopicPartition
import java.util.Properties
import java.time.Duration
import scala.collection.JavaConverters._

object OffsetManagement {
  def main(args: Array[String]): Unit = {
    val props = new Properties()
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092")
    props.put(ConsumerConfig.GROUP_ID_CONFIG, "offset-demo")
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer")
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer")
    props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest")
    props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false") // manual commit
    
    val consumer = new KafkaConsumer[String, String](props)
    val topic = "orders"
    
    try {
      consumer.subscribe(java.util.Arrays.asList(topic))
      
      // ===== CommitSync: รอจนกว่าจะ commit สำเร็จ =====
      val records1 = consumer.poll(Duration.ofSeconds(1))
      records1.asScala.foreach { record =>
        processRecord(record)
      }
      consumer.commitSync() // commit ทุก partition ที่ poll มา
      
      // ===== CommitAsync: ไม่รอ result =====
      val records2 = consumer.poll(Duration.ofSeconds(1))
      records2.asScala.foreach { record =>
        processRecord(record)
      }
      consumer.commitAsync { (offsets, exception) =>
        if (exception != null) {
          println(s"Commit failed: ${exception.getMessage}")
          // อาจ retry หรือ log error
        } else {
          println(s"Committed: $offsets")
        }
      }
      
      // ===== Commit Specific Offsets =====
      val records3 = consumer.poll(Duration.ofSeconds(1))
      val offsetsToCommit = scala.collection.mutable.Map[TopicPartition, OffsetAndMetadata]()
      
      records3.asScala.foreach { record =>
        if (processRecord(record)) {
          // เฉพาะ commit records ที่ process สำเร็จ
          val tp = new TopicPartition(record.topic(), record.partition())
          offsetsToCommit(tp) = new OffsetAndMetadata(record.offset() + 1)
        }
      }
      
      consumer.commitSync(offsetsToCommit.asJava)
      
      // ===== Seek to Specific Offset =====
      // มีประโยชน์เมื่อต้องการ replay หรือ skip messages
      
      // Seek to beginning
      val partitions = consumer.assignment()
      consumer.seekToBeginning(partitions)
      
      // Seek to end
      consumer.seekToEnd(partitions)
      
      // Seek to specific offset
      val tp = new TopicPartition(topic, 0)
      consumer.seek(tp, 100L) // เริ่มอ่านจาก offset 100
      
      // Seek to timestamp
      val timestampsToSearch = Map(
        new TopicPartition(topic, 0) -> Long.box(System.currentTimeMillis() - 3600000L) // 1 hour ago
      )
      val offsetsForTimes = consumer.offsetsForTimes(timestampsToSearch.asJava)
      offsetsForTimes.asScala.foreach { case (tp, offsetAndTimestamp) =>
        if (offsetAndTimestamp != null) {
          consumer.seek(tp, offsetAndTimestamp.offset())
          println(s"Seeked $tp to offset ${offsetAndTimestamp.offset()}")
        }
      }
      
    } finally {
      consumer.close()
    }
  }
  
  def processRecord(record: ConsumerRecord[String, String]): Boolean = {
    // process logic here
    println(s"Processing: offset=${record.offset()}, key=${record.key()}")
    true
  }
}
```

---

## Step 756: Topics และ Partitions

```scala
// TopicManagement.scala
import org.apache.kafka.clients.admin._
import org.apache.kafka.common.config.TopicConfig
import java.util.Properties
import scala.collection.JavaConverters._

object TopicManagement {
  
  def createAdminClient(bootstrapServers: String): AdminClient = {
    val props = new Properties()
    props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers)
    props.put(AdminClientConfig.REQUEST_TIMEOUT_MS_CONFIG, "5000")
    AdminClient.create(props)
  }
  
  def main(args: Array[String]): Unit = {
    val admin = createAdminClient("localhost:9092")
    
    try {
      // ===== สร้าง Topic =====
      println("=== Create Topics ===")
      
      val topicConfigs = Map(
        TopicConfig.RETENTION_MS_CONFIG         -> "604800000",  // 7 days
        TopicConfig.CLEANUP_POLICY_CONFIG       -> "delete",
        TopicConfig.COMPRESSION_TYPE_CONFIG     -> "snappy",
        TopicConfig.MIN_IN_SYNC_REPLICAS_CONFIG -> "1"
      )
      
      val newTopics = List(
        new NewTopic("orders", 3, 1.toShort)
          .configs(topicConfigs.asJava),
        new NewTopic("order-events", 6, 1.toShort)
          .configs(topicConfigs.asJava),
        new NewTopic("dlq-orders", 1, 1.toShort) // Dead Letter Queue
      )
      
      val createResult = admin.createTopics(newTopics.asJava)
      newTopics.foreach { topic =>
        try {
          createResult.values().get(topic.name()).get()
          println(s"Created: ${topic.name()}")
        } catch {
          case ex: Exception =>
            println(s"Failed to create ${topic.name()}: ${ex.getCause.getMessage}")
        }
      }
      
      // ===== List Topics =====
      println("\n=== List Topics ===")
      val topics = admin.listTopics().names().get()
      topics.asScala.filter(!_.startsWith("__")).toList.sorted.foreach(println)
      
      // ===== Describe Topics =====
      println("\n=== Describe Topics ===")
      val described = admin.describeTopics(List("orders").asJava).all().get()
      described.asScala.foreach { case (name, desc) =>
        println(s"Topic: $name")
        desc.partitions().asScala.foreach { partition =>
          println(s"  Partition ${partition.partition()}: " +
                  s"leader=${partition.leader().id()}, " +
                  s"replicas=${partition.replicas().asScala.map(_.id()).mkString(",")}")
        }
      }
      
      // ===== Increase Partitions =====
      // ระวัง: ลด partitions ไม่ได้!
      val newPartitions = Map("orders" -> NewPartitions.increaseTo(6))
      admin.createPartitions(newPartitions.asJava).all().get()
      println("\nIncreased orders partitions to 6")
      
      // ===== Consumer Group Info =====
      println("\n=== Consumer Groups ===")
      val groups = admin.listConsumerGroups().all().get()
      groups.asScala.foreach { group =>
        println(s"  Group: ${group.groupId()}, State: ${group.state().orElse(null)}")
      }
      
      // ===== Delete Topics =====
      println("\n=== Cleanup ===")
      admin.deleteTopics(List("dlq-orders").asJava).all().get()
      println("Deleted dlq-orders")
      
    } finally {
      admin.close()
    }
  }
}
```

---

## Step 757: Serialization

```scala
// KafkaSerialization.scala
import org.apache.kafka.clients.producer._
import org.apache.kafka.clients.consumer._
import org.apache.kafka.common.serialization._
import spray.json._
import java.util.Properties

// Domain model
case class OrderEvent(
  orderId: String,
  customerId: String,
  amount: Double,
  status: String,
  timestamp: Long = System.currentTimeMillis()
)

// JSON Serialization ด้วย spray-json
// build.sbt: "io.spray" %% "spray-json" % "1.3.6"
import DefaultJsonProtocol._
object OrderEventJsonProtocol extends DefaultJsonProtocol {
  implicit val orderEventFormat: RootJsonFormat[OrderEvent] = jsonFormat5(OrderEvent)
}

// Custom Kafka Serializer
class OrderEventSerializer extends Serializer[OrderEvent] {
  import OrderEventJsonProtocol._
  
  override def serialize(topic: String, data: OrderEvent): Array[Byte] = {
    if (data == null) null
    else data.toJson.compactPrint.getBytes("UTF-8")
  }
  
  override def configure(configs: java.util.Map[String, _], isKey: Boolean): Unit = {}
  override def close(): Unit = {}
}

// Custom Kafka Deserializer
class OrderEventDeserializer extends Deserializer[OrderEvent] {
  import OrderEventJsonProtocol._
  
  override def deserialize(topic: String, data: Array[Byte]): OrderEvent = {
    if (data == null) null
    else new String(data, "UTF-8").parseJson.convertTo[OrderEvent]
  }
  
  override def configure(configs: java.util.Map[String, _], isKey: Boolean): Unit = {}
  override def close(): Unit = {}
}

object KafkaSerializationExample {
  def main(args: Array[String]): Unit = {
    
    // ===== Producer with Custom Serializer =====
    val producerProps = new Properties()
    producerProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092")
    producerProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, classOf[StringSerializer].getName)
    producerProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, classOf[OrderEventSerializer].getName)
    
    val producer = new KafkaProducer[String, OrderEvent](producerProps)
    
    val event = OrderEvent("O001", "C001", 999.99, "pending")
    val record = new ProducerRecord[String, OrderEvent]("orders", event.orderId, event)
    
    producer.send(record, (meta, ex) => {
      if (ex == null) println(s"Sent to offset ${meta.offset()}")
      else println(s"Failed: ${ex.getMessage}")
    })
    
    producer.flush()
    producer.close()
    
    // ===== Consumer with Custom Deserializer =====
    val consumerProps = new Properties()
    consumerProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092")
    consumerProps.put(ConsumerConfig.GROUP_ID_CONFIG, "order-consumer")
    consumerProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, classOf[StringDeserializer].getName)
    consumerProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, classOf[OrderEventDeserializer].getName)
    consumerProps.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest")
    
    val consumer = new KafkaConsumer[String, OrderEvent](consumerProps)
    consumer.subscribe(java.util.Arrays.asList("orders"))
    
    import java.time.Duration
    import scala.collection.JavaConverters._
    
    val records = consumer.poll(Duration.ofSeconds(2))
    records.asScala.foreach { record =>
      val order = record.value()
      println(s"Received: OrderId=${order.orderId}, Amount=${order.amount}, Status=${order.status}")
    }
    
    consumer.close()
  }
}
```

---

## Step 758: Consumer Groups และ Rebalancing

```scala
// ConsumerGroupDemo.scala
import org.apache.kafka.clients.consumer._
import java.util.Properties
import java.time.Duration
import scala.collection.JavaConverters._
import java.util.concurrent.{Executors, TimeUnit}

object ConsumerGroupDemo {
  
  def createConsumer(
    bootstrapServers: String,
    groupId: String,
    instanceId: Int
  ): KafkaConsumer[String, String] = {
    val props = new Properties()
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers)
    props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId)
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer")
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer")
    props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest")
    props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false")
    props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, "10")
    
    val consumer = new KafkaConsumer[String, String](props)
    
    // ConsumerRebalanceListener
    consumer.subscribe(
      java.util.Arrays.asList("orders"),
      new ConsumerRebalanceListener {
        override def onPartitionsRevoked(partitions: java.util.Collection[org.apache.kafka.common.TopicPartition]): Unit = {
          println(s"Consumer$instanceId: Partitions REVOKED: ${partitions.asScala.map(_.partition()).mkString(", ")}")
          // commit offsets ก่อนสูญเสีย partition ownership
          consumer.commitSync()
        }
        
        override def onPartitionsAssigned(partitions: java.util.Collection[org.apache.kafka.common.TopicPartition]): Unit = {
          println(s"Consumer$instanceId: Partitions ASSIGNED: ${partitions.asScala.map(_.partition()).mkString(", ")}")
        }
      }
    )
    
    consumer
  }
  
  def main(args: Array[String]): Unit = {
    val groupId = "parallel-processor"
    val numConsumers = 3
    
    // ===== สร้าง parallel consumers =====
    val executor = Executors.newFixedThreadPool(numConsumers)
    
    (1 to numConsumers).foreach { i =>
      executor.submit(new Runnable {
        override def run(): Unit = {
          val consumer = createConsumer("localhost:9092", groupId, i)
          
          try {
            var count = 0
            while (count < 20) {
              val records = consumer.poll(Duration.ofSeconds(1))
              
              for (record <- records.asScala) {
                println(s"Consumer$i | Part=${record.partition()} | Off=${record.offset()} | Key=${record.key()}")
                count += 1
              }
              
              if (!records.isEmpty) consumer.commitSync()
            }
          } finally {
            consumer.close()
            println(s"Consumer$i closed")
          }
        }
      })
    }
    
    executor.shutdown()
    executor.awaitTermination(30, TimeUnit.SECONDS)
    
    // Partition Assignment:
    // 3 consumers, 3 partitions → 1 partition per consumer
    // 3 consumers, 6 partitions → 2 partitions per consumer
    // 3 consumers, 2 partitions → 1 consumer gets 0 partitions (idle)
    
    println("""
      Consumer Group Rules:
      - Max parallel consumers = number of partitions
      - Extra consumers = idle (backup)
      - One consumer can own multiple partitions
      - One partition assigned to only one consumer per group
      - Different groups can read same partition independently
    """)
  }
}
```

---

## Step 759: Kafka Transactions และ Exactly-Once

```scala
// KafkaTransactions.scala
import org.apache.kafka.clients.producer._
import org.apache.kafka.clients.consumer._
import java.util.Properties
import java.time.Duration
import scala.collection.JavaConverters._

object KafkaTransactions {
  
  def createTransactionalProducer(transactionalId: String): KafkaProducer[String, String] = {
    val props = new Properties()
    props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092")
    props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer")
    props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer")
    props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, transactionalId) // unique per producer
    props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true")
    props.put(ProducerConfig.ACKS_CONFIG, "all")
    
    new KafkaProducer[String, String](props)
  }
  
  def createTransactionalConsumer(groupId: String): KafkaConsumer[String, String] = {
    val props = new Properties()
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092")
    props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId)
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer")
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer")
    props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed") // อ่านแค่ committed transactions
    props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false")
    
    new KafkaConsumer[String, String](props)
  }
  
  def main(args: Array[String]): Unit = {
    val producer = createTransactionalProducer("order-processor-1")
    
    // ต้อง init transaction ก่อน
    producer.initTransactions()
    
    try {
      // ===== Transaction 1: สำเร็จ =====
      producer.beginTransaction()
      try {
        producer.send(new ProducerRecord("orders", "O001", """{"status":"confirmed"}"""))
        producer.send(new ProducerRecord("inventory", "P001", """{"reserved":1}"""))
        producer.send(new ProducerRecord("notifications", "C001", """{"msg":"Order confirmed"}"""))
        
        // commit transaction - ทุก messages committed atomically
        producer.commitTransaction()
        println("Transaction 1 committed")
        
      } catch {
        case ex: Exception =>
          producer.abortTransaction() // rollback ทุก messages
          println(s"Transaction 1 aborted: ${ex.getMessage}")
      }
      
      // ===== Transaction 2: ล้มเหลว =====
      producer.beginTransaction()
      try {
        producer.send(new ProducerRecord("orders", "O002", """{"status":"confirmed"}"""))
        
        // Simulate failure
        throw new RuntimeException("Payment service unavailable")
        
        producer.send(new ProducerRecord("payments", "O002", """{"charged":true}"""))
        producer.commitTransaction()
        
      } catch {
        case ex: Exception =>
          producer.abortTransaction()
          println(s"Transaction 2 aborted: ${ex.getMessage}")
          // Message ใน orders topic จะถูก abort ด้วย
      }
      
    } finally {
      producer.close()
    }
    
    // ===== Read Committed =====
    val consumer = createTransactionalConsumer("transactional-reader")
    consumer.subscribe(java.util.Arrays.asList("orders"))
    
    val records = consumer.poll(Duration.ofSeconds(2))
    println(s"\nCommitted records only: ${records.count()}")
    records.asScala.foreach { r =>
      println(s"  Key=${r.key()}, Value=${r.value()}")
    }
    // จะเห็นแค่ O001 (committed) ไม่เห็น O002 (aborted)
    
    consumer.close()
  }
}
```

---

## Step 760: Kafka Monitoring และ Metrics

```scala
// KafkaMonitoring.scala
import org.apache.kafka.clients.admin._
import org.apache.kafka.clients.consumer.OffsetAndMetadata
import org.apache.kafka.common.TopicPartition
import java.util.Properties
import scala.collection.JavaConverters._

object KafkaMonitoring {
  def main(args: Array[String]): Unit = {
    val props = new Properties()
    props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092")
    
    val admin = AdminClient.create(props)
    
    try {
      // ===== Consumer Lag =====
      println("=== Consumer Lag ===")
      
      val groupId = "order-processor"
      val topic = "orders"
      
      // ดู current offsets ของ consumer group
      val groupOffsets = admin.listConsumerGroupOffsets(groupId)
        .partitionsToOffsetAndMetadata().get()
      
      // ดู end offsets (latest)
      val partitions = groupOffsets.keySet()
      val endOffsets = admin.listOffsets(
        partitions.asScala.map(tp => tp -> OffsetSpec.latest()).toMap.asJava
      ).all().get()
      
      println(f"${"Topic"}%-15s ${"Partition"}%-10s ${"Current"}%-10s ${"End"}%-10s ${"Lag"}%-10s")
      println("-" * 60)
      
      partitions.asScala.foreach { tp =>
        val currentOffset = groupOffsets.get(tp).offset()
        val endOffset = endOffsets.get(tp).offset()
        val lag = endOffset - currentOffset
        
        println(f"${tp.topic()}%-15s ${tp.partition()}%-10d $currentOffset%-10d $endOffset%-10d $lag%-10d")
      }
      
      // ===== Topic Stats =====
      println("\n=== Topic Stats ===")
      
      val topicDesc = admin.describeTopics(List(topic).asJava).all().get()
      topicDesc.asScala.foreach { case (name, desc) =>
        println(s"Topic: $name")
        println(s"  Partitions: ${desc.partitions().size()}")
        
        val startOffsets = admin.listOffsets(
          desc.partitions().asScala.map(p => 
            new TopicPartition(name, p.partition()) -> OffsetSpec.earliest()
          ).toMap.asJava
        ).all().get()
        
        val endOffsets2 = admin.listOffsets(
          desc.partitions().asScala.map(p =>
            new TopicPartition(name, p.partition()) -> OffsetSpec.latest()
          ).toMap.asJava
        ).all().get()
        
        val totalMessages = desc.partitions().asScala.map { p =>
          val tp = new TopicPartition(name, p.partition())
          val start = startOffsets.get(tp).offset()
          val end = endOffsets2.get(tp).offset()
          end - start
        }.sum
        
        println(s"  Total messages: $totalMessages")
      }
      
      // ===== Cluster Info =====
      println("\n=== Cluster Info ===")
      val clusterDesc = admin.describeCluster()
      val nodes = clusterDesc.nodes().get()
      val controller = clusterDesc.controller().get()
      
      println(s"Cluster ID: ${clusterDesc.clusterId().get()}")
      println(s"Controller: ${controller.id()}")
      println(s"Brokers (${nodes.size()}):")
      nodes.asScala.foreach { node =>
        println(s"  Broker ${node.id()}: ${node.host()}:${node.port()} [${node.rack()}]")
      }
      
    } finally {
      admin.close()
    }
  }
}
```

---

## สรุป Part 76

| Concept | Description | Key Setting |
|---------|-------------|-------------|
| Topic | Named log of records | partitions, retention |
| Partition | Ordered sub-log | replication-factor |
| Offset | Record position | consumer tracks per partition |
| Consumer Group | Parallel consumers | group.id |
| Producer Acks | Reliability guarantee | acks=all |
| Auto Commit | Automatic offset commit | enable.auto.commit |
| Transaction | Atomic multi-topic writes | transactional.id |
| Isolation Level | Read committed vs uncommitted | isolation.level |

### Kafka Configuration Quick Reference

```
Producer:
  acks=all + retries=INT_MAX + enable.idempotence=true → exactly-once

Consumer:
  enable.auto.commit=false + commitSync after processing → at-least-once
  read_committed + transactional producer → exactly-once

Performance:
  linger.ms=10-100 + batch.size=65536 + compression.type=snappy → high throughput
  fetch.min.bytes=1024 + fetch.max.wait.ms=500 → batched reads
```

---

## แบบฝึกหัด Part 76

1. **Producer Configuration**: สร้าง producer ที่ configure สำหรับ maximum throughput แล้ว benchmark TPS (transactions per second) เปรียบเทียบกับ default settings

2. **Consumer Groups**: สร้าง topic มี 6 partitions แล้วรัน 3 consumer instances ในกลุ่มเดียวกัน ดูว่า partitions ถูก assign อย่างไร จากนั้นหยุด 1 consumer ดู rebalancing

3. **Offset Management**: implement exactly-once processing ด้วย manual offset commit + transactional producer เขียน test ที่แสดงว่า crash กลางคัน ไม่ทำให้ process ข้อมูลซ้ำ

4. **Dead Letter Queue**: implement DLQ pattern ที่เมื่อ process record fail หลายครั้ง จะส่งไปยัง dead-letter-queue topic แทน

5. **Lag Monitoring**: สร้าง monitoring service ที่ poll consumer lag ทุก 30 วินาที และ alert เมื่อ lag เกิน threshold

---

## ไปต่อ: Part 77 — Kafka with Scala (Alpakka)
ใน Part ถัดไปจะเรียนการใช้ Kafka กับ Akka Streams ผ่าน Alpakka connector

[→ Part 77: Kafka with Scala](./part-77-kafka-scala.md)
