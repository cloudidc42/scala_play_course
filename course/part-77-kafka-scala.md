# Part 77: Kafka with Scala (Alpakka Kafka) — Steps 761-770

## บทนำ: Alpakka Kafka

Alpakka Kafka (formerly Reactive Kafka) คือ Kafka connector สำหรับ Akka Streams ทำให้สามารถ integrate Kafka กับ reactive streams processing ได้อย่าง elegant

---

## Step 761: Setup และ build.sbt

```scala
// build.sbt
libraryDependencies ++= Seq(
  // Akka Streams
  "com.typesafe.akka" %% "akka-stream"       % "2.8.5",
  "com.typesafe.akka" %% "akka-actor-typed"  % "2.8.5",
  
  // Alpakka Kafka
  "com.typesafe.akka" %% "akka-stream-kafka" % "4.0.2",
  
  // Kafka clients
  "org.apache.kafka"  %  "kafka-clients"     % "3.6.0",
  
  // Serialization
  "io.spray"          %% "spray-json"        % "1.3.6",
  
  // Testing
  "com.typesafe.akka" %% "akka-stream-testkit" % "2.8.5" % Test,
  "org.scalatest"     %% "scalatest"           % "3.2.17" % Test
)
```

### application.conf

```hocon
# src/main/resources/application.conf
akka {
  loglevel = "INFO"
  
  kafka {
    producer {
      kafka-clients {
        bootstrap.servers = "localhost:9092"
        acks = all
        retries = 3
        compression.type = snappy
        batch.size = 16384
        linger.ms = 10
      }
    }
    
    consumer {
      kafka-clients {
        bootstrap.servers = "localhost:9092"
        group.id = "akka-consumer"
        auto.offset.reset = earliest
        enable.auto.commit = false
      }
      # Alpakka-specific settings
      stop-timeout = 30s
      close-timeout = 20s
      commit-timeout = 15s
      commit-time-warning = 10s
      wakeup-timeout = 10s
      max-wakeups = 10
      commit-refresh-interval = infinite
      
      poll-interval = 50ms
      poll-timeout = 50ms
      
      buffer-size = 100
    }
  }
}
```

---

## Step 762: Alpakka Producer

```scala
// AlpakkaProducer.scala
import akka.actor.ActorSystem
import akka.kafka.ProducerSettings
import akka.kafka.scaladsl.Producer
import akka.stream.scaladsl._
import org.apache.kafka.clients.producer.ProducerRecord
import org.apache.kafka.common.serialization.StringSerializer
import spray.json._
import DefaultJsonProtocol._
import scala.concurrent.{Await, ExecutionContext}
import scala.concurrent.duration._

case class OrderCreated(orderId: String, customerId: String, amount: Double, timestamp: Long)

object AlpakkaProducerExample {
  
  implicit val orderFormat: RootJsonFormat[OrderCreated] = jsonFormat4(OrderCreated)
  
  def main(args: Array[String]): Unit = {
    implicit val system: ActorSystem = ActorSystem("kafka-producer")
    implicit val ec: ExecutionContext = system.dispatcher
    
    // Producer settings
    val producerSettings = ProducerSettings(system, new StringSerializer, new StringSerializer)
      .withBootstrapServers("localhost:9092")
    
    // ===== Simple Producer =====
    println("=== Simple Producer ===")
    
    val orders = List(
      OrderCreated("O001", "C001", 999.99, System.currentTimeMillis()),
      OrderCreated("O002", "C002", 499.99, System.currentTimeMillis()),
      OrderCreated("O003", "C003", 1299.99, System.currentTimeMillis())
    )
    
    val done = Source(orders)
      .map { order =>
        new ProducerRecord[String, String](
          "orders",           // topic
          order.customerId,   // key (determines partition)
          order.toJson.compactPrint  // value
        )
      }
      .runWith(Producer.plainSink(producerSettings))
    
    Await.result(done, 10.seconds)
    println("Simple producer done")
    
    // ===== Producer with PassThrough =====
    // ส่งไปพร้อมกับ carry context (เช่น HTTP request context)
    case class RequestContext(requestId: String, order: OrderCreated)
    
    val ordersWithContext = List(
      RequestContext("REQ001", OrderCreated("O004", "C001", 200.0, System.currentTimeMillis())),
      RequestContext("REQ002", OrderCreated("O005", "C002", 300.0, System.currentTimeMillis()))
    )
    
    val producerWithContext = Source(ordersWithContext)
      .map { ctx =>
        val record = new ProducerRecord[String, String](
          "orders",
          ctx.order.customerId,
          ctx.order.toJson.compactPrint
        )
        (record, ctx.requestId) // carry context
      }
      .via(Producer.flexiFlow(producerSettings))
      .map { case (metadata, requestId) =>
        println(s"Request $requestId → partition=${metadata.partition()}, offset=${metadata.offset()}")
        requestId
      }
      .runWith(Sink.ignore)
    
    Await.result(producerWithContext, 10.seconds)
    
    // ===== Multi-topic Producer =====
    case class RoutedMessage(topic: String, key: String, value: String)
    
    val routedMessages = List(
      RoutedMessage("orders",       "O006", """{"type":"order","amount":100}"""),
      RoutedMessage("notifications","C001", """{"type":"email","msg":"Order placed"}"""),
      RoutedMessage("audit",        "O006", """{"type":"audit","action":"order_created"}""")
    )
    
    val multiTopicDone = Source(routedMessages)
      .map { msg =>
        new ProducerRecord[String, String](msg.topic, msg.key, msg.value)
      }
      .runWith(Producer.plainSink(producerSettings))
    
    Await.result(multiTopicDone, 10.seconds)
    println("Multi-topic producer done")
    
    system.terminate()
    Await.result(system.whenTerminated, 5.seconds)
  }
}
```

---

## Step 763: Alpakka Consumer

```scala
// AlpakkaConsumer.scala
import akka.actor.ActorSystem
import akka.kafka.{ConsumerSettings, Subscriptions}
import akka.kafka.scaladsl.Consumer
import akka.stream.scaladsl._
import org.apache.kafka.common.serialization.StringDeserializer
import spray.json._
import DefaultJsonProtocol._
import scala.concurrent.{Await, ExecutionContext}
import scala.concurrent.duration._

object AlpakkaConsumerExample {
  
  implicit val orderFormat: RootJsonFormat[OrderCreated] = jsonFormat4(OrderCreated)
  
  def main(args: Array[String]): Unit = {
    implicit val system: ActorSystem = ActorSystem("kafka-consumer")
    implicit val ec: ExecutionContext = system.dispatcher
    
    // Consumer settings
    val consumerSettings = ConsumerSettings(system, new StringDeserializer, new StringDeserializer)
      .withBootstrapServers("localhost:9092")
      .withGroupId("alpakka-consumer")
      .withProperty("auto.offset.reset", "earliest")
    
    // ===== Plain Source (no commit control) =====
    println("=== Plain Source ===")
    
    val plainDone = Consumer
      .plainSource(consumerSettings, Subscriptions.topics("orders"))
      .map { record =>
        println(s"Plain: partition=${record.partition()}, offset=${record.offset()}, value=${record.value()}")
        record.value()
      }
      .take(5) // รับแค่ 5 messages แล้วหยุด
      .runWith(Sink.ignore)
    
    Await.result(plainDone, 15.seconds)
    
    // ===== Committable Source (manual commit) =====
    println("\n=== Committable Source ===")
    
    val committableDone = Consumer
      .committableSource(consumerSettings, Subscriptions.topics("orders"))
      .mapAsync(1) { msg =>
        // Process the message
        val value = msg.record.value()
        println(s"Processing: ${value.take(50)}")
        
        // Return commit offset
        scala.concurrent.Future.successful(msg.committableOffset)
      }
      .take(5)
      .runWith(akka.kafka.scaladsl.Committer.sink(
        akka.kafka.CommitterSettings(system)
      ))
    
    Await.result(committableDone, 15.seconds)
    
    // ===== Offset Batching for Performance =====
    println("\n=== Batched Commits ===")
    
    val batchDone = Consumer
      .committableSource(consumerSettings.withGroupId("batch-consumer"), Subscriptions.topics("orders"))
      .map { msg =>
        val value = msg.record.value()
        (value, msg.committableOffset)
      }
      .take(10)
      .batch(max = 20, first => akka.kafka.CommittableOffsetBatch(first._2)) {
        (batch, element) => batch.updated(element._2)
      }
      .mapAsync(3)(_.commitScaladsl())
      .runWith(Sink.ignore)
    
    Await.result(batchDone, 15.seconds)
    
    system.terminate()
    Await.result(system.whenTerminated, 5.seconds)
  }
}
```

---

## Step 764: Consumer-Producer Pipeline

```scala
// ConsumerProducerPipeline.scala
import akka.actor.ActorSystem
import akka.kafka._
import akka.kafka.scaladsl._
import akka.stream.scaladsl._
import org.apache.kafka.clients.producer.ProducerRecord
import org.apache.kafka.common.serialization._
import spray.json._
import DefaultJsonProtocol._
import scala.concurrent.{Await, ExecutionContext}
import scala.concurrent.duration._

case class RawEvent(eventId: String, userId: String, action: String, timestamp: Long)
case class EnrichedEvent(
  eventId: String,
  userId: String,
  action: String,
  timestamp: Long,
  actionCategory: String,
  isHighValue: Boolean
)

object ConsumerProducerPipeline {
  
  implicit val rawEventFormat:      RootJsonFormat[RawEvent]      = jsonFormat4(RawEvent)
  implicit val enrichedEventFormat: RootJsonFormat[EnrichedEvent] = jsonFormat6(EnrichedEvent)
  
  def categorize(action: String): String = action match {
    case "purchase" | "add_to_cart" => "conversion"
    case "view" | "search"          => "discovery"
    case "login" | "logout"         => "auth"
    case _                          => "other"
  }
  
  def isHighValue(action: String): Boolean = action == "purchase"
  
  def main(args: Array[String]): Unit = {
    implicit val system: ActorSystem = ActorSystem("pipeline")
    implicit val ec: ExecutionContext = system.dispatcher
    
    val consumerSettings = ConsumerSettings(system, new StringDeserializer, new StringDeserializer)
      .withBootstrapServers("localhost:9092")
      .withGroupId("event-enricher")
      .withProperty("auto.offset.reset", "earliest")
    
    val producerSettings = ProducerSettings(system, new StringSerializer, new StringSerializer)
      .withBootstrapServers("localhost:9092")
    
    // ===== Pipeline: consume → enrich → produce =====
    val pipelineDone = Consumer
      .committableSource(consumerSettings, Subscriptions.topics("raw-events"))
      .map { msg =>
        val raw = msg.record.value().parseJson.convertTo[RawEvent]
        val enriched = EnrichedEvent(
          eventId       = raw.eventId,
          userId        = raw.userId,
          action        = raw.action,
          timestamp     = raw.timestamp,
          actionCategory = categorize(raw.action),
          isHighValue    = isHighValue(raw.action)
        )
        
        // ProducerMessage.Message ส่งพร้อมกับ committable offset
        ProducerMessage.single(
          new ProducerRecord[String, String](
            "enriched-events",
            enriched.userId,
            enriched.toJson.compactPrint
          ),
          msg.committableOffset // pass through offset
        )
      }
      .via(Producer.flexiFlow(producerSettings))
      .map(_.passThrough) // extract committable offsets
      .take(20)
      .runWith(Committer.sink(CommitterSettings(system)))
    
    Await.result(pipelineDone, 30.seconds)
    println("Pipeline completed")
    
    // ===== Fan-out Pipeline =====
    // หนึ่ง consumer, หลาย sinks
    val (control, fanoutDone) = Consumer
      .plainSource(consumerSettings.withGroupId("fanout-consumer"), Subscriptions.topics("orders"))
      .map { record =>
        val json = record.value().parseJson.asJsObject
        json
      }
      .alsoTo(
        // Sink 1: high value orders
        Flow[JsObject]
          .filter { json =>
            json.fields.get("amount").exists(v => v.toString.toDouble > 500)
          }
          .map { json =>
            new ProducerRecord[String, String]("high-value-orders", json.compactPrint)
          }
          .to(Producer.plainSink(producerSettings))
      )
      .to(
        // Sink 2: all orders for analytics
        Flow[JsObject]
          .map { json =>
            new ProducerRecord[String, String]("analytics-orders", json.compactPrint)
          }
          .to(Producer.plainSink(producerSettings))
      )
      .run()
    
    Thread.sleep(5000)
    control.shutdown()
    
    system.terminate()
    Await.result(system.whenTerminated, 5.seconds)
  }
}
```

---

## Step 765: Error Handling

```scala
// KafkaErrorHandling.scala
import akka.actor.ActorSystem
import akka.kafka._
import akka.kafka.scaladsl._
import akka.stream.scaladsl._
import akka.stream.ActorAttributes
import akka.stream.Supervision
import org.apache.kafka.clients.producer.ProducerRecord
import org.apache.kafka.common.serialization._
import scala.concurrent.{Await, ExecutionContext}
import scala.concurrent.duration._

object KafkaErrorHandling {
  
  case class ProcessingError(record: String, error: String)
  
  def main(args: Array[String]): Unit = {
    implicit val system: ActorSystem = ActorSystem("error-handling")
    implicit val ec: ExecutionContext = system.dispatcher
    
    val consumerSettings = ConsumerSettings(system, new StringDeserializer, new StringDeserializer)
      .withBootstrapServers("localhost:9092")
      .withGroupId("error-handler")
    
    val producerSettings = ProducerSettings(system, new StringSerializer, new StringSerializer)
      .withBootstrapServers("localhost:9092")
    
    // ===== Supervision Strategy =====
    val decider: Supervision.Decider = {
      case ex: spray.json.JsonParser.ParsingException =>
        println(s"JSON parsing error: ${ex.getMessage} - RESUMING")
        Supervision.Resume // skip bad message, continue
      
      case ex: IllegalArgumentException =>
        println(s"Validation error: ${ex.getMessage} - RESUMING")
        Supervision.Resume
      
      case ex: Exception =>
        println(s"Unexpected error: ${ex.getMessage} - STOPPING")
        Supervision.Stop
    }
    
    // ===== Dead Letter Queue Pattern =====
    val (dlqSuccess, dlqFailure) =
      Flow[String]
        .map { value =>
          if (value.contains("INVALID")) {
            throw new IllegalArgumentException(s"Invalid message: $value")
          }
          value.toUpperCase
        }
        .divertTo(
          // DLQ sink: failed messages
          Flow[String]
            .map { value =>
              new ProducerRecord[String, String]("dead-letter-queue", value)
            }
            .to(Producer.plainSink(producerSettings)),
          _ => false // ไม่ divert ที่นี่ จัดการใน recover
        )
        .run()
    
    // ===== Recover Pattern =====
    Consumer
      .plainSource(consumerSettings, Subscriptions.topics("orders"))
      .map(_.value())
      .mapConcat { value =>
        try {
          // Try to process
          List(value.toUpperCase)
        } catch {
          case ex: Exception =>
            println(s"Error processing, sending to DLQ: ${ex.getMessage}")
            // ส่งไป DLQ
            List.empty
        }
      }
      .take(10)
      .withAttributes(ActorAttributes.supervisionStrategy(decider))
      .runWith(Sink.foreach(println))
      .onComplete { result =>
        println(s"Stream completed: $result")
      }
    
    // ===== Retry Pattern =====
    def processWithRetry(value: String, retries: Int = 3): scala.concurrent.Future[String] = {
      scala.concurrent.Future {
        if (scala.util.Random.nextDouble() < 0.3 && retries > 0) {
          throw new RuntimeException("Transient error")
        }
        value.toUpperCase
      }.recoverWith {
        case ex: RuntimeException if retries > 0 =>
          println(s"Retrying... attempts left: ${retries - 1}")
          processWithRetry(value, retries - 1)
      }
    }
    
    Consumer
      .plainSource(consumerSettings.withGroupId("retry-consumer"), Subscriptions.topics("orders"))
      .mapAsync(3) { record =>
        processWithRetry(record.value())
          .recover { case ex =>
            s"FAILED: ${ex.getMessage}"
          }
      }
      .take(5)
      .runWith(Sink.foreach(println))
    
    Thread.sleep(10000)
    
    system.terminate()
    Await.result(system.whenTerminated, 5.seconds)
  }
}
```

---

## Step 766: Kafka Streams Processing

```scala
// KafkaStreamsProcessing.scala
import akka.actor.ActorSystem
import akka.kafka._
import akka.kafka.scaladsl._
import akka.stream.scaladsl._
import org.apache.kafka.clients.producer.ProducerRecord
import org.apache.kafka.common.serialization._
import scala.concurrent.{Await, ExecutionContext}
import scala.concurrent.duration._

object KafkaStreamsProcessing {
  
  def main(args: Array[String]): Unit = {
    implicit val system: ActorSystem = ActorSystem("streams-processing")
    implicit val ec: ExecutionContext = system.dispatcher
    
    val consumerSettings = ConsumerSettings(system, new StringDeserializer, new StringDeserializer)
      .withBootstrapServers("localhost:9092")
      .withGroupId("streams-processor")
    
    val producerSettings = ProducerSettings(system, new StringSerializer, new StringSerializer)
      .withBootstrapServers("localhost:9092")
    
    // ===== Aggregation Window =====
    // Group messages in time windows
    
    import scala.collection.mutable
    
    // รวบรวม events ทุก 5 วินาที แล้วส่ง aggregate
    val windowStream = Consumer
      .plainSource(consumerSettings, Subscriptions.topics("click-events"))
      .map(_.value())
      .groupedWithin(100, 5.seconds) // รวบ max 100 records หรือทุก 5 วินาที
      .map { batch =>
        val count = batch.length
        val eventTypes = batch.groupBy(identity).map { case (k, v) => s"$k:${v.size}" }.mkString(",")
        s"""{"window":"5s","count":$count,"breakdown":"$eventTypes"}"""
      }
      .map { aggregate =>
        new ProducerRecord[String, String]("click-aggregates", aggregate)
      }
      .runWith(Producer.plainSink(producerSettings))
    
    // ===== Stateful Processing =====
    // สร้าง session windows ด้วย state
    
    case class UserSession(
      userId: String,
      events: List[String],
      startTime: Long,
      lastEventTime: Long
    )
    
    val sessionTimeout = 30000L // 30 seconds
    val sessions = mutable.Map[String, UserSession]()
    
    Consumer
      .plainSource(consumerSettings.withGroupId("session-processor"), Subscriptions.topics("user-events"))
      .map { record =>
        val parts = record.value().split(":")
        val userId = parts(0)
        val event = parts(1)
        (userId, event, System.currentTimeMillis())
      }
      .statefulMapConcat { () =>
        // State ต่อ stream
        val activeSessions = mutable.Map[String, UserSession]()
        
        { case (userId, event, now) =>
          // Close expired sessions
          val expired = activeSessions.filter { case (uid, s) =>
            now - s.lastEventTime > sessionTimeout
          }
          
          expired.foreach { case (uid, session) =>
            activeSessions.remove(uid)
            // emit session end event
            println(s"Session ended for $uid: ${session.events.size} events")
          }
          
          // Update or create session
          val session = activeSessions.getOrElse(userId, 
            UserSession(userId, Nil, now, now))
          
          activeSessions(userId) = session.copy(
            events = event :: session.events,
            lastEventTime = now
          )
          
          // Emit nothing (stateful accumulation)
          List.empty[String]
        }
      }
      .take(0) // ไม่เอา output (แค่ state updates)
      .runWith(Sink.ignore)
    
    // ===== Parallel Processing Partitions =====
    // Process each partition separately for ordering guarantees
    val partitionParallelism = 3
    
    Consumer
      .plainPartitionedSource(
        consumerSettings.withGroupId("partition-processor"),
        Subscriptions.topics("orders")
      )
      .flatMapMerge(partitionParallelism, { case (topicPartition, source) =>
        println(s"Starting to process partition: ${topicPartition.partition()}")
        source
          .map { record =>
            s"Partition ${record.partition()}: ${record.value()}"
          }
          .alsoTo(Sink.foreach(msg => println(msg)))
      })
      .take(30)
      .runWith(Sink.ignore)
    
    Thread.sleep(15000)
    
    system.terminate()
    Await.result(system.whenTerminated, 5.seconds)
  }
}
```

---

## Step 767: Kafka Avro Serialization

```scala
// AvroKafka.scala
// build.sbt เพิ่ม:
// "io.confluent" % "kafka-avro-serializer" % "7.5.0"
// "org.apache.avro" % "avro" % "1.11.3"

// Schema definition (สร้าง .avsc file)
/*
{
  "type": "record",
  "name": "Order",
  "namespace": "com.example",
  "fields": [
    {"name": "order_id",    "type": "string"},
    {"name": "customer_id", "type": "string"},
    {"name": "amount",      "type": "double"},
    {"name": "status",      "type": "string"},
    {"name": "timestamp",   "type": "long"}
  ]
}
*/

import org.apache.avro.{Schema, GenericData}
import org.apache.avro.generic.{GenericRecord, GenericRecordBuilder}
import org.apache.avro.io.{BinaryEncoder, EncoderFactory, DecoderFactory}
import org.apache.avro.specific.{SpecificDatumWriter, SpecificDatumReader}
import java.io.ByteArrayOutputStream

object AvroKafka {
  
  // Avro Schema
  val orderSchemaStr = """
    {
      "type": "record",
      "name": "Order",
      "namespace": "com.example",
      "fields": [
        {"name": "order_id",    "type": "string"},
        {"name": "customer_id", "type": "string"},
        {"name": "amount",      "type": "double"},
        {"name": "status",      "type": "string"},
        {"name": "timestamp",   "type": "long"}
      ]
    }
  """
  
  val schema: Schema = new Schema.Parser().parse(orderSchemaStr)
  
  // Serialize to Avro bytes
  def serialize(orderId: String, customerId: String, amount: Double, status: String): Array[Byte] = {
    val record = new GenericData.Record(schema)
    record.put("order_id",    orderId)
    record.put("customer_id", customerId)
    record.put("amount",      amount)
    record.put("status",      status)
    record.put("timestamp",   System.currentTimeMillis())
    
    val out = new ByteArrayOutputStream()
    val encoder = EncoderFactory.get().binaryEncoder(out, null)
    val writer = new org.apache.avro.generic.GenericDatumWriter[GenericRecord](schema)
    writer.write(record, encoder)
    encoder.flush()
    out.close()
    out.toByteArray
  }
  
  // Deserialize from Avro bytes
  def deserialize(bytes: Array[Byte]): GenericRecord = {
    val decoder = DecoderFactory.get().binaryDecoder(bytes, null)
    val reader = new org.apache.avro.generic.GenericDatumReader[GenericRecord](schema)
    reader.read(null, decoder)
  }
  
  def main(args: Array[String]): Unit = {
    // Test serialization
    val bytes = serialize("O001", "C001", 999.99, "pending")
    println(s"Avro bytes size: ${bytes.length}")
    
    val record = deserialize(bytes)
    println(s"Deserialized: ${record.get("order_id")} - ${record.get("amount")}")
    
    // ถ้าใช้กับ Kafka + Schema Registry:
    // val producerProps = new Properties()
    // producerProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
    //   "io.confluent.kafka.serializers.KafkaAvroSerializer")
    // producerProps.put("schema.registry.url", "http://localhost:8081")
    
    println("""
      Avro Benefits:
      - Schema evolution (add/remove nullable fields)
      - Binary format (compact)
      - Schema Registry for centralized schema management
      - Compatible with Spark, Hive, Flink
      
      Schema Registry:
      - Stores schemas centrally
      - Validates compatibility (backward/forward/full)
      - Producer registers schema before sending
      - Consumer retrieves schema by ID from header
    """)
  }
}
```

---

## Step 768: Kafka Exactly-Once Semantics

```scala
// ExactlyOnceSemantics.scala
import akka.actor.ActorSystem
import akka.kafka._
import akka.kafka.scaladsl._
import akka.stream.scaladsl._
import org.apache.kafka.clients.producer.ProducerRecord
import org.apache.kafka.common.serialization._
import scala.concurrent.{Await, ExecutionContext}
import scala.concurrent.duration._

object ExactlyOnceSemantics {
  
  def main(args: Array[String]): Unit = {
    implicit val system: ActorSystem = ActorSystem("exactly-once")
    implicit val ec: ExecutionContext = system.dispatcher
    
    val consumerSettings = ConsumerSettings(system, new StringDeserializer, new StringDeserializer)
      .withBootstrapServers("localhost:9092")
      .withGroupId("eos-consumer")
      .withProperty("isolation.level", "read_committed")
      .withProperty("auto.offset.reset", "earliest")
    
    val producerSettings = ProducerSettings(system, new StringSerializer, new StringSerializer)
      .withBootstrapServers("localhost:9092")
      .withProperty("enable.idempotence", "true")
      .withProperty("transactional.id", "eos-producer-1")
      .withProperty("acks", "all")
    
    // ===== Transactional Producer-Consumer =====
    // Exactly-once: consume from input, produce to output atomically
    
    val transactionalDone = Transactional
      .source(consumerSettings, Subscriptions.topics("input-orders"))
      .map { msg =>
        val processedValue = s"PROCESSED:${msg.record.value()}"
        ProducerMessage.single(
          new ProducerRecord[String, String]("processed-orders", msg.record.key(), processedValue),
          msg.partitionOffset
        )
      }
      .take(10)
      .runWith(Transactional.sink(producerSettings, "eos-transactional-id"))
    
    Await.result(transactionalDone, 30.seconds)
    println("Exactly-once pipeline completed")
    
    system.terminate()
    Await.result(system.whenTerminated, 5.seconds)
  }
}
```

---

## Step 769: Performance Tuning

```scala
// KafkaPerformanceTuning.scala
import akka.actor.ActorSystem
import akka.kafka._
import akka.kafka.scaladsl._
import akka.stream.scaladsl._
import org.apache.kafka.clients.producer.ProducerRecord
import org.apache.kafka.common.serialization._
import scala.concurrent.{Await, ExecutionContext}
import scala.concurrent.duration._

object KafkaPerformanceTuning {
  
  def main(args: Array[String]): Unit = {
    implicit val system: ActorSystem = ActorSystem("performance")
    implicit val ec: ExecutionContext = system.dispatcher
    
    // ===== High-Throughput Producer =====
    val highThroughputProducer = ProducerSettings(system, new StringSerializer, new StringSerializer)
      .withBootstrapServers("localhost:9092")
      .withProperty("batch.size",             "65536")   // 64KB batch
      .withProperty("linger.ms",              "50")      // รอ 50ms เพื่อ batch
      .withProperty("compression.type",       "snappy")  // compress
      .withProperty("buffer.memory",          "67108864") // 64MB buffer
      .withProperty("max.in.flight.requests.per.connection", "5") // pipeline
    
    // ===== High-Throughput Consumer =====
    val highThroughputConsumer = ConsumerSettings(system, new StringDeserializer, new StringDeserializer)
      .withBootstrapServers("localhost:9092")
      .withGroupId("high-throughput")
      .withProperty("fetch.min.bytes",        "65536")   // 64KB min fetch
      .withProperty("fetch.max.wait.ms",      "500")     // max wait
      .withProperty("max.poll.records",       "500")     // 500 records per poll
      .withProperty("max.partition.fetch.bytes", "1048576") // 1MB per partition
    
    // ===== Parallel Processing =====
    val parallelism = 10 // process 10 messages concurrently
    
    val throughputPipeline = Consumer
      .plainSource(highThroughputConsumer, Subscriptions.topics("orders"))
      .mapAsync(parallelism) { record =>
        // Async processing
        scala.concurrent.Future {
          Thread.sleep(10) // simulate work
          record.value().toUpperCase
        }
      }
      .groupedWithin(100, 1.second) // batch outputs
      .map { batch =>
        println(s"Processed batch of ${batch.size}")
        batch
      }
      .runWith(Sink.ignore)
    
    // ===== Benchmarking =====
    var totalMessages = 0
    val startTime = System.currentTimeMillis()
    
    Consumer
      .plainSource(highThroughputConsumer.withGroupId("bench"), Subscriptions.topics("orders"))
      .take(1000)
      .map { _ =>
        totalMessages += 1
        if (totalMessages % 100 == 0) {
          val elapsed = (System.currentTimeMillis() - startTime) / 1000.0
          val throughput = totalMessages / elapsed
          println(f"Throughput: $throughput%.1f msg/s ($totalMessages total)")
        }
      }
      .runWith(Sink.ignore)
    
    println("""
      Performance Guidelines:
      Producer:
        - batch.size=65536 + linger.ms=50 → 5-10x throughput
        - compression.type=snappy → 3-4x bandwidth reduction
        - max.in.flight.requests.per.connection=5 → pipelining
      
      Consumer:
        - max.poll.records=500 → reduce poll overhead
        - fetch.min.bytes=65536 → larger fetches
        - Use mapAsync for IO-bound processing
        - Use grouped batching for DB writes
      
      Partitions:
        - More partitions = higher parallelism
        - But too many = overhead
        - Rule: partitions = 2-4 × consumer count
    """)
    
    Thread.sleep(10000)
    system.terminate()
    Await.result(system.whenTerminated, 5.seconds)
  }
}
```

---

## Step 770: Production Kafka Application

```scala
// ProductionKafkaApp.scala
import akka.actor.ActorSystem
import akka.kafka._
import akka.kafka.scaladsl._
import akka.stream.scaladsl._
import org.apache.kafka.clients.producer.ProducerRecord
import org.apache.kafka.common.serialization._
import spray.json._
import DefaultJsonProtocol._
import scala.concurrent.{Await, ExecutionContext}
import scala.concurrent.duration._

// Domain events
sealed trait OrderEvent
case class OrderPlaced(orderId: String, customerId: String, amount: Double, items: List[String]) extends OrderEvent
case class PaymentProcessed(orderId: String, paymentId: String, amount: Double) extends OrderEvent
case class OrderShipped(orderId: String, trackingNumber: String) extends OrderEvent

object OrderEventProtocol extends DefaultJsonProtocol {
  implicit val orderPlacedFormat:      RootJsonFormat[OrderPlaced]      = jsonFormat4(OrderPlaced)
  implicit val paymentProcessedFormat: RootJsonFormat[PaymentProcessed] = jsonFormat3(PaymentProcessed)
  implicit val orderShippedFormat:     RootJsonFormat[OrderShipped]     = jsonFormat2(OrderShipped)
}

object ProductionKafkaApp {
  import OrderEventProtocol._
  
  // Command processing
  def processOrderEvent(json: String): Either[String, String] = {
    try {
      val jsObj = json.parseJson.asJsObject
      val eventType = jsObj.fields.get("type").map(_.convertTo[String])
      
      eventType match {
        case Some("OrderPlaced") =>
          val event = json.parseJson.convertTo[OrderPlaced]
          val notification = s"""{"orderId":"${event.orderId}","msg":"Your order has been placed","amount":${event.amount}}"""
          Right(notification)
        
        case Some("PaymentProcessed") =>
          val event = json.parseJson.convertTo[PaymentProcessed]
          val notification = s"""{"orderId":"${event.orderId}","msg":"Payment confirmed","paymentId":"${event.paymentId}"}"""
          Right(notification)
        
        case Some("OrderShipped") =>
          val event = json.parseJson.convertTo[OrderShipped]
          val notification = s"""{"orderId":"${event.orderId}","msg":"Your order is on its way","tracking":"${event.trackingNumber}"}"""
          Right(notification)
        
        case _ => Left(s"Unknown event type: $eventType")
      }
    } catch {
      case ex: Exception => Left(s"Parse error: ${ex.getMessage}")
    }
  }
  
  def main(args: Array[String]): Unit = {
    implicit val system: ActorSystem = ActorSystem("production-kafka")
    implicit val ec: ExecutionContext = system.dispatcher
    
    val consumerSettings = ConsumerSettings(system, new StringDeserializer, new StringDeserializer)
      .withBootstrapServers("localhost:9092")
      .withGroupId("notification-service")
      .withProperty("auto.offset.reset", "latest")
    
    val producerSettings = ProducerSettings(system, new StringSerializer, new StringSerializer)
      .withBootstrapServers("localhost:9092")
    
    // ===== Notification Pipeline =====
    val (control, done) = Consumer
      .committableSource(consumerSettings, Subscriptions.topics("order-events"))
      .mapAsync(5) { msg =>
        scala.concurrent.Future {
          val result = processOrderEvent(msg.record.value())
          (result, msg.committableOffset)
        }
      }
      .mapConcat { case (result, offset) =>
        result match {
          case Right(notification) =>
            List(
              ProducerMessage.single(
                new ProducerRecord[String, String]("notifications", notification),
                offset
              )
            )
          case Left(error) =>
            println(s"Error: $error, sending to DLQ")
            List(
              ProducerMessage.single(
                new ProducerRecord[String, String]("dead-letter-orders", error),
                offset
              )
            )
        }
      }
      .via(Producer.flexiFlow(producerSettings))
      .map(_.passThrough)
      .toMat(Committer.sink(CommitterSettings(system)))(Keep.both)
      .run()
    
    // Graceful shutdown
    sys.addShutdownHook {
      println("Shutting down Kafka consumer...")
      val shutdown = control.drainAndShutdown()
      Await.result(shutdown, 30.seconds)
    }
    
    println("Production Kafka app started")
    Await.result(done, Duration.Inf)
    
    system.terminate()
  }
}
```

---

## สรุป Part 77

| Component | Class | Use Case |
|-----------|-------|----------|
| Plain Source | Consumer.plainSource | No commit needed |
| Committable Source | Consumer.committableSource | Manual commit |
| Partitioned Source | Consumer.plainPartitionedSource | Per-partition ordering |
| Plain Sink | Producer.plainSink | Simple produce |
| Transactional | Transactional.source/sink | Exactly-once |
| Committer | Committer.sink | Batch commit |

### Delivery Semantics

| Pattern | Consumer | Producer | Guarantee |
|---------|----------|----------|-----------|
| At-most-once | auto commit | fire-forget | May lose |
| At-least-once | manual commit after process | wait ack | May duplicate |
| Exactly-once | read_committed + manual | transactional | No dup/loss |

---

## แบบฝึกหัด Part 77

1. **Event Sourcing**: implement event sourcing ที่ produce events ไป Kafka แล้ว consumer replay events เพื่อ rebuild state

2. **Error Recovery**: implement production-ready error handling ที่มี: retry with backoff, dead letter queue, และ circuit breaker

3. **Parallel Processing**: สร้าง consumer ที่ process แต่ละ partition ใน parallel thread โดยรักษา ordering ภายใน partition

4. **Exactly-Once Pipeline**: implement transactional consume-transform-produce pipeline ที่ guarantee exactly-once แม้มี failure

5. **Throughput Benchmark**: สร้าง producer ที่ส่ง 1 million messages แล้ว benchmark throughput ด้วย configuration ต่างๆ

---

## ไปต่อ: Part 78 — Spark + Kafka Integration
ใน Part ถัดไปจะเรียนการ integrate Spark Structured Streaming กับ Kafka

[→ Part 78: Spark-Kafka Integration](./part-78-spark-kafka-integration.md)
