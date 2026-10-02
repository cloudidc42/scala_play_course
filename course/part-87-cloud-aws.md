# Part 87: Cloud AWS — Steps 861-870

## บทนำ: AWS SDK for Scala

AWS SDK v2 สำหรับ Java/Scala ให้ async clients ที่ใช้ร่วมกับ Scala Future และ ZIO ได้ดี รองรับ S3, SQS, DynamoDB, Lambda และอีกหลายบริการ

---

## Step 861: AWS SDK Setup

```scala
// build.sbt
val awsSdkVersion = "2.21.46"

libraryDependencies ++= Seq(
  // AWS SDK v2 - เลือก services ที่ต้องการ
  "software.amazon.awssdk" % "s3"              % awsSdkVersion,
  "software.amazon.awssdk" % "sqs"             % awsSdkVersion,
  "software.amazon.awssdk" % "dynamodb"        % awsSdkVersion,
  "software.amazon.awssdk" % "lambda"          % awsSdkVersion,
  "software.amazon.awssdk" % "secretsmanager"  % awsSdkVersion,
  "software.amazon.awssdk" % "sns"             % awsSdkVersion,
  "software.amazon.awssdk" % "cloudwatch"      % awsSdkVersion,
  "software.amazon.awssdk" % "kinesis"         % awsSdkVersion,
  
  // Async HTTP client
  "software.amazon.awssdk" % "netty-nio-client" % awsSdkVersion,
  
  // Alpakka AWS (Akka Streams integration)
  "com.lightbend.akka" %% "akka-stream-alpakka-s3"          % "7.0.2",
  "com.lightbend.akka" %% "akka-stream-alpakka-sqs"         % "7.0.2",
  "com.lightbend.akka" %% "akka-stream-alpakka-dynamodb"    % "7.0.2",
  
  // ZIO AWS (ZIO integration)
  "dev.zio"            %% "zio-aws-s3"       % "7.21.46.0",
  "dev.zio"            %% "zio-aws-sqs"      % "7.21.46.0",
  "dev.zio"            %% "zio-aws-dynamodb" % "7.21.46.0"
)
```

---

## Step 862: S3 Operations

```scala
// S3Service.scala
import software.amazon.awssdk.auth.credentials.DefaultCredentialsProvider
import software.amazon.awssdk.regions.Region
import software.amazon.awssdk.services.s3.S3AsyncClient
import software.amazon.awssdk.services.s3.model._
import software.amazon.awssdk.core.async.{AsyncRequestBody, AsyncResponseTransformer}
import software.amazon.awssdk.transfer.s3.S3TransferManager
import software.amazon.awssdk.transfer.s3.model._
import java.nio.file.{Path, Paths}
import java.util.concurrent.CompletableFuture
import scala.jdk.FutureConverters._
import scala.concurrent.{ExecutionContext, Future}

class S3Service(bucket: String)(implicit ec: ExecutionContext) {
  
  private val s3Client = S3AsyncClient.builder()
    .credentialsProvider(DefaultCredentialsProvider.create())
    .region(Region.AP_SOUTHEAST_1)
    .build()
  
  private val transferManager = S3TransferManager.builder()
    .s3Client(s3Client)
    .build()
  
  // ===== Upload =====
  def uploadFile(key: String, content: String): Future[String] = {
    val request = PutObjectRequest.builder()
      .bucket(bucket)
      .key(key)
      .contentType("application/json")
      .serverSideEncryption(ServerSideEncryption.AES256)
      .build()
    
    s3Client.putObject(request, AsyncRequestBody.fromString(content))
      .asScala
      .map { response =>
        println(s"Uploaded $key, ETag: ${response.eTag()}")
        response.eTag()
      }
  }
  
  def uploadLargeFile(key: String, localPath: Path): Future[String] = {
    val upload = transferManager.uploadFile(
      UploadFileRequest.builder()
        .putObjectRequest(b => b.bucket(bucket).key(key))
        .source(localPath)
        .build()
    )
    
    upload.completionFuture().asScala.map { result =>
      result.response().eTag()
    }
  }
  
  // ===== Download =====
  def downloadAsString(key: String): Future[String] = {
    val request = GetObjectRequest.builder()
      .bucket(bucket)
      .key(key)
      .build()
    
    s3Client.getObject(request, AsyncResponseTransformer.toBytes())
      .asScala
      .map(_.asUtf8String())
  }
  
  def downloadToFile(key: String, localPath: Path): Future[Unit] = {
    val download = transferManager.downloadFile(
      DownloadFileRequest.builder()
        .getObjectRequest(b => b.bucket(bucket).key(key))
        .destination(localPath)
        .build()
    )
    
    download.completionFuture().asScala.map(_ => ())
  }
  
  // ===== List =====
  def listObjects(prefix: String): Future[List[String]] = {
    val request = ListObjectsV2Request.builder()
      .bucket(bucket)
      .prefix(prefix)
      .maxKeys(1000)
      .build()
    
    s3Client.listObjectsV2(request)
      .asScala
      .map { response =>
        response.contents().asScala.toList.map(_.key())
      }
  }
  
  // ===== Presigned URL =====
  def generatePresignedUrl(key: String, expirationMinutes: Int = 60): String = {
    import software.amazon.awssdk.services.s3.presigner.S3Presigner
    import software.amazon.awssdk.services.s3.presigner.model.GetObjectPresignRequest
    import java.time.Duration
    
    val presigner = S3Presigner.builder()
      .region(Region.AP_SOUTHEAST_1)
      .build()
    
    val presignedUrl = presigner.presignGetObject { b =>
      b.getObjectRequest(_.bucket(bucket).key(key))
        .signatureDuration(Duration.ofMinutes(expirationMinutes))
    }
    
    presigner.close()
    presignedUrl.url().toString
  }
  
  // ===== Multipart Upload (large files) =====
  def uploadMultipart(key: String, content: Array[Byte]): Future[String] = {
    // Create multipart upload
    val createRequest = CreateMultipartUploadRequest.builder()
      .bucket(bucket)
      .key(key)
      .build()
    
    s3Client.createMultipartUpload(createRequest).asScala.flatMap { createResponse =>
      val uploadId = createResponse.uploadId()
      val chunkSize = 5 * 1024 * 1024 // 5MB minimum
      val chunks = content.grouped(chunkSize).toList
      
      // Upload all parts
      val partFutures = chunks.zipWithIndex.map { case (chunk, idx) =>
        val partRequest = UploadPartRequest.builder()
          .bucket(bucket)
          .key(key)
          .uploadId(uploadId)
          .partNumber(idx + 1)
          .contentLength(chunk.length.toLong)
          .build()
        
        s3Client.uploadPart(partRequest, AsyncRequestBody.fromBytes(chunk))
          .asScala
          .map(r => CompletedPart.builder().partNumber(idx + 1).eTag(r.eTag()).build())
      }
      
      Future.sequence(partFutures).flatMap { completedParts =>
        // Complete multipart upload
        val completeRequest = CompleteMultipartUploadRequest.builder()
          .bucket(bucket)
          .key(key)
          .uploadId(uploadId)
          .multipartUpload(CompletedMultipartUpload.builder().parts(completedParts.asJava).build())
          .build()
        
        s3Client.completeMultipartUpload(completeRequest)
          .asScala
          .map(_.eTag())
      }
    }
  }
  
  // ===== Delete =====
  def deleteObject(key: String): Future[Unit] = {
    s3Client.deleteObject(DeleteObjectRequest.builder()
      .bucket(bucket)
      .key(key)
      .build()
    ).asScala.map(_ => ())
  }
  
  def close(): Unit = {
    transferManager.close()
    s3Client.close()
  }
}

object S3Demo {
  def main(args: Array[String]): Unit = {
    import scala.concurrent.{Await, ExecutionContext}
    import scala.concurrent.duration._
    import ExecutionContext.Implicits.global
    
    val s3 = new S3Service("my-app-bucket")
    
    try {
      // Upload JSON
      val eTag = Await.result(
        s3.uploadFile("orders/2024/01/orders.json", """[{"id":"O001","amount":999.99}]"""),
        30.seconds
      )
      println(s"Uploaded, ETag: $eTag")
      
      // Download
      val content = Await.result(s3.downloadAsString("orders/2024/01/orders.json"), 30.seconds)
      println(s"Downloaded: $content")
      
      // Presigned URL
      val url = s3.generatePresignedUrl("orders/2024/01/orders.json", 60)
      println(s"Presigned URL: $url")
      
      // List
      val keys = Await.result(s3.listObjects("orders/2024/"), 30.seconds)
      println(s"Files: ${keys.mkString(", ")}")
      
    } finally {
      s3.close()
    }
  }
}
```

---

## Step 863: SQS Operations

```scala
// SQSService.scala
import software.amazon.awssdk.services.sqs.SqsAsyncClient
import software.amazon.awssdk.services.sqs.model._
import scala.jdk.CollectionConverters._
import scala.jdk.FutureConverters._
import scala.concurrent.{ExecutionContext, Future}

class SQSService(queueUrl: String)(implicit ec: ExecutionContext) {
  
  private val sqsClient = SqsAsyncClient.builder()
    .region(software.amazon.awssdk.regions.Region.AP_SOUTHEAST_1)
    .build()
  
  // ===== Send Message =====
  def sendMessage(body: String, attributes: Map[String, String] = Map.empty): Future[String] = {
    val msgAttributes = attributes.map { case (k, v) =>
      k -> MessageAttributeValue.builder()
             .dataType("String")
             .stringValue(v)
             .build()
    }.asJava
    
    sqsClient.sendMessage(
      SendMessageRequest.builder()
        .queueUrl(queueUrl)
        .messageBody(body)
        .messageAttributes(msgAttributes)
        .build()
    ).asScala.map(_.messageId())
  }
  
  // Send with deduplication (FIFO queues)
  def sendFIFOMessage(body: String, groupId: String, dedupId: String): Future[String] = {
    sqsClient.sendMessage(
      SendMessageRequest.builder()
        .queueUrl(queueUrl)
        .messageBody(body)
        .messageGroupId(groupId)
        .messageDeduplicationId(dedupId)
        .build()
    ).asScala.map(_.messageId())
  }
  
  // ===== Send Batch =====
  def sendBatch(messages: List[String]): Future[List[String]] = {
    val entries = messages.zipWithIndex.map { case (body, idx) =>
      SendMessageBatchRequestEntry.builder()
        .id(s"msg-$idx")
        .messageBody(body)
        .build()
    }.asJava
    
    sqsClient.sendMessageBatch(
      SendMessageBatchRequest.builder()
        .queueUrl(queueUrl)
        .entries(entries)
        .build()
    ).asScala.map { response =>
      response.successful().asScala.toList.map(_.messageId())
    }
  }
  
  // ===== Receive Messages =====
  def receiveMessages(maxMessages: Int = 10, waitSeconds: Int = 20): Future[List[Message]] = {
    sqsClient.receiveMessage(
      ReceiveMessageRequest.builder()
        .queueUrl(queueUrl)
        .maxNumberOfMessages(maxMessages)
        .waitTimeSeconds(waitSeconds)  // Long polling
        .messageAttributeNames("All")
        .attributeNames(QueueAttributeName.ALL)
        .build()
    ).asScala.map(_.messages().asScala.toList)
  }
  
  // ===== Delete Message (after processing) =====
  def deleteMessage(receiptHandle: String): Future[Unit] = {
    sqsClient.deleteMessage(
      DeleteMessageRequest.builder()
        .queueUrl(queueUrl)
        .receiptHandle(receiptHandle)
        .build()
    ).asScala.map(_ => ())
  }
  
  // ===== Change Visibility =====
  def extendVisibility(receiptHandle: String, timeoutSeconds: Int = 300): Future[Unit] = {
    sqsClient.changeMessageVisibility(
      ChangeMessageVisibilityRequest.builder()
        .queueUrl(queueUrl)
        .receiptHandle(receiptHandle)
        .visibilityTimeout(timeoutSeconds)
        .build()
    ).asScala.map(_ => ())
  }
  
  // ===== Process Loop =====
  def startProcessingLoop(handler: Message => Future[Unit]): Unit = {
    import scala.concurrent.duration._
    
    def loop(): Future[Unit] = {
      receiveMessages().flatMap { messages =>
        if (messages.isEmpty) {
          Future.successful(())
        } else {
          Future.sequence(messages.map { msg =>
            handler(msg)
              .flatMap(_ => deleteMessage(msg.receiptHandle()))
              .recover { case ex =>
                println(s"Failed to process ${msg.messageId()}: ${ex.getMessage}")
                // Message will become visible again after visibility timeout
              }
          }).map(_ => ())
        }
      }.flatMap(_ => loop())
    }
    
    loop()
  }
  
  def close(): Unit = sqsClient.close()
}
```

---

## Step 864: DynamoDB Operations

```scala
// DynamoDBService.scala
import software.amazon.awssdk.services.dynamodb.DynamoDbAsyncClient
import software.amazon.awssdk.services.dynamodb.model._
import spray.json._
import spray.json.DefaultJsonProtocol._
import scala.jdk.CollectionConverters._
import scala.jdk.FutureConverters._
import scala.concurrent.{ExecutionContext, Future}

// Table schema
case class OrderRecord(
  orderId: String,
  customerId: String,
  status: String,
  amount: Double,
  createdAt: String,
  ttl: Option[Long] = None
)

class DynamoDBOrderRepository(tableName: String)(implicit ec: ExecutionContext) {
  
  private val dynamoDB = DynamoDbAsyncClient.builder()
    .region(software.amazon.awssdk.regions.Region.AP_SOUTHEAST_1)
    .build()
  
  // Convert to AttributeValue map
  private def toItem(order: OrderRecord): Map[String, AttributeValue] = {
    val base = Map(
      "orderId"    -> AttributeValue.builder().s(order.orderId).build(),
      "customerId" -> AttributeValue.builder().s(order.customerId).build(),
      "status"     -> AttributeValue.builder().s(order.status).build(),
      "amount"     -> AttributeValue.builder().n(order.amount.toString).build(),
      "createdAt"  -> AttributeValue.builder().s(order.createdAt).build()
    )
    
    order.ttl.map { t =>
      base + ("ttl" -> AttributeValue.builder().n(t.toString).build())
    }.getOrElse(base)
  }
  
  // Convert from AttributeValue map
  private def fromItem(item: Map[String, AttributeValue]): OrderRecord = {
    OrderRecord(
      orderId    = item("orderId").s(),
      customerId = item("customerId").s(),
      status     = item("status").s(),
      amount     = item("amount").n().toDouble,
      createdAt  = item("createdAt").s(),
      ttl        = item.get("ttl").map(_.n().toLong)
    )
  }
  
  // ===== Put (Create/Update) =====
  def putOrder(order: OrderRecord): Future[Unit] = {
    dynamoDB.putItem(
      PutItemRequest.builder()
        .tableName(tableName)
        .item(toItem(order).asJava)
        .build()
    ).asScala.map(_ => ())
  }
  
  // ===== Get =====
  def getOrder(orderId: String): Future[Option[OrderRecord]] = {
    dynamoDB.getItem(
      GetItemRequest.builder()
        .tableName(tableName)
        .key(Map("orderId" -> AttributeValue.builder().s(orderId).build()).asJava)
        .consistentRead(true)
        .build()
    ).asScala.map { response =>
      if (response.hasItem) Some(fromItem(response.item().asScala.toMap))
      else None
    }
  }
  
  // ===== Update =====
  def updateOrderStatus(orderId: String, newStatus: String): Future[Unit] = {
    dynamoDB.updateItem(
      UpdateItemRequest.builder()
        .tableName(tableName)
        .key(Map("orderId" -> AttributeValue.builder().s(orderId).build()).asJava)
        .updateExpression("SET #status = :newStatus, updatedAt = :updatedAt")
        .expressionAttributeNames(Map("#status" -> "status").asJava)
        .expressionAttributeValues(Map(
          ":newStatus" -> AttributeValue.builder().s(newStatus).build(),
          ":updatedAt" -> AttributeValue.builder().s(java.time.Instant.now().toString).build()
        ).asJava)
        .conditionExpression("attribute_exists(orderId)")  // fail if not exists
        .build()
    ).asScala.map(_ => ())
  }
  
  // ===== Query (by GSI) =====
  def getOrdersByCustomer(customerId: String, status: Option[String] = None): Future[List[OrderRecord]] = {
    val queryBuilder = QueryRequest.builder()
      .tableName(tableName)
      .indexName("customerId-createdAt-index") // GSI
      .keyConditionExpression("customerId = :customerId")
      .expressionAttributeValues(scala.collection.mutable.Map(
        ":customerId" -> AttributeValue.builder().s(customerId).build()
      ).asJava)
    
    val request = status match {
      case Some(s) =>
        queryBuilder
          .filterExpression("#status = :status")
          .expressionAttributeNames(Map("#status" -> "status").asJava)
          .expressionAttributeValues(Map(
            ":customerId" -> AttributeValue.builder().s(customerId).build(),
            ":status" -> AttributeValue.builder().s(s).build()
          ).asJava)
          .build()
      case None => queryBuilder.build()
    }
    
    dynamoDB.query(request).asScala.map { response =>
      response.items().asScala.toList.map(item => fromItem(item.asScala.toMap))
    }
  }
  
  // ===== Batch Write =====
  def batchWriteOrders(orders: List[OrderRecord]): Future[Unit] = {
    // DynamoDB บatch max 25 items
    val batches = orders.grouped(25)
    
    Future.sequence(batches.map { batch =>
      val writeRequests = batch.map { order =>
        WriteRequest.builder()
          .putRequest(PutRequest.builder().item(toItem(order).asJava).build())
          .build()
      }.asJava
      
      dynamoDB.batchWriteItem(
        BatchWriteItemRequest.builder()
          .requestItems(Map(tableName -> writeRequests).asJava)
          .build()
      ).asScala
    }.toList).map(_ => ())
  }
  
  // ===== Scan (expensive - use sparingly) =====
  def scanByStatus(status: String): Future[List[OrderRecord]] = {
    dynamoDB.scan(
      ScanRequest.builder()
        .tableName(tableName)
        .filterExpression("#status = :status")
        .expressionAttributeNames(Map("#status" -> "status").asJava)
        .expressionAttributeValues(Map(
          ":status" -> AttributeValue.builder().s(status).build()
        ).asJava)
        .build()
    ).asScala.map { response =>
      response.items().asScala.toList.map(item => fromItem(item.asScala.toMap))
    }
  }
  
  // ===== Transaction =====
  def createOrderWithInventory(
    order: OrderRecord,
    productId: String,
    quantity: Int
  ): Future[Unit] = {
    dynamoDB.transactWriteItems(
      TransactWriteItemsRequest.builder()
        .transactItems(
          // Create order
          TransactWriteItem.builder()
            .put(Put.builder()
              .tableName(tableName)
              .item(toItem(order).asJava)
              .conditionExpression("attribute_not_exists(orderId)") // prevent duplicate
              .build())
            .build(),
          
          // Update inventory
          TransactWriteItem.builder()
            .update(Update.builder()
              .tableName("products")
              .key(Map("productId" -> AttributeValue.builder().s(productId).build()).asJava)
              .updateExpression("SET stock = stock - :qty")
              .conditionExpression("stock >= :qty")  // ensure sufficient stock
              .expressionAttributeValues(Map(
                ":qty" -> AttributeValue.builder().n(quantity.toString).build()
              ).asJava)
              .build())
            .build()
        ).build()
    ).asScala.map(_ => ())
  }
  
  def close(): Unit = dynamoDB.close()
}
```

---

## Step 865: Lambda Functions

```scala
// LambdaHandler.scala
// สำหรับ AWS Lambda ใช้ aws-lambda-java-core library

// build.sbt
// libraryDependencies += "com.amazonaws" % "aws-lambda-java-core" % "1.2.3"
// libraryDependencies += "com.amazonaws" % "aws-lambda-java-events" % "3.11.3"

import com.amazonaws.services.lambda.runtime.{Context, RequestHandler}
import com.amazonaws.services.lambda.runtime.events.{APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent}
import spray.json._
import spray.json.DefaultJsonProtocol._

// ===== Request/Response types =====
case class OrderRequest(customerId: String, amount: Double)
case class OrderResponse(orderId: String, status: String)

object OrderJson {
  implicit val requestFormat: RootJsonFormat[OrderRequest] = jsonFormat2(OrderRequest)
  implicit val responseFormat: RootJsonFormat[OrderResponse] = jsonFormat2(OrderResponse)
}

// ===== Lambda Handler =====
class CreateOrderHandler extends RequestHandler[APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent] {
  import OrderJson._
  
  override def handleRequest(
    input: APIGatewayProxyRequestEvent,
    context: Context
  ): APIGatewayProxyResponseEvent = {
    
    val logger = context.getLogger
    
    try {
      // Parse request
      val body = input.getBody
      logger.log(s"Request: $body")
      
      val request = body.parseJson.convertTo[OrderRequest]
      
      // Process
      val orderId = java.util.UUID.randomUUID().toString
      val response = OrderResponse(orderId, "pending")
      
      logger.log(s"Created order: $orderId")
      
      // Return response
      new APIGatewayProxyResponseEvent()
        .withStatusCode(201)
        .withBody(response.toJson.toString)
        .withHeaders(Map("Content-Type" -> "application/json").asJava)
        
    } catch {
      case ex: Exception =>
        context.getLogger.log(s"Error: ${ex.getMessage}")
        new APIGatewayProxyResponseEvent()
          .withStatusCode(500)
          .withBody(s"""{"error":"${ex.getMessage}"}""")
    }
  }
}

// ===== Lambda with SQS Trigger =====
import com.amazonaws.services.lambda.runtime.events.SQSEvent

class SQSOrderProcessor extends RequestHandler[SQSEvent, Void] {
  
  override def handleRequest(event: SQSEvent, context: Context): Void = {
    import scala.jdk.CollectionConverters._
    
    event.getRecords.asScala.foreach { record =>
      try {
        context.getLogger.log(s"Processing: ${record.getBody}")
        // Process message
        Thread.sleep(100) // simulate processing
        context.getLogger.log(s"Processed: ${record.getMessageId}")
      } catch {
        case ex: Exception =>
          context.getLogger.log(s"Failed: ${ex.getMessage}")
          throw ex // re-throw to trigger retry/DLQ
      }
    }
    
    null
  }
}
```

```yaml
# lambda-function.yaml (SAM template)
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: java17
    MemorySize: 512
    Timeout: 30
    Environment:
      Variables:
        APP_ENV: production
        DYNAMODB_TABLE: !Ref OrdersTable

Resources:
  CreateOrderFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: com.example.CreateOrderHandler::handleRequest
      CodeUri: target/scala-2.13/lambda-assembly-1.0.0.jar
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /orders
            Method: POST
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref OrdersTable
        - SQSSendMessagePolicy:
            QueueName: !GetAtt OrderQueue.QueueName

  OrdersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      AttributeDefinitions:
        - AttributeName: orderId
          AttributeType: S
      KeySchema:
        - AttributeName: orderId
          KeyType: HASH
      BillingMode: PAY_PER_REQUEST

  OrderQueue:
    Type: AWS::SQS::Queue
    Properties:
      VisibilityTimeout: 300
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt OrderDLQ.Arn
        maxReceiveCount: 3

  OrderDLQ:
    Type: AWS::SQS::Queue
```

---

## Step 866: SNS Events

```scala
// SNSService.scala
import software.amazon.awssdk.services.sns.SnsAsyncClient
import software.amazon.awssdk.services.sns.model._
import spray.json._
import scala.jdk.FutureConverters._
import scala.concurrent.{ExecutionContext, Future}

class SNSEventPublisher(topicArn: String)(implicit ec: ExecutionContext) {
  
  private val snsClient = SnsAsyncClient.builder()
    .region(software.amazon.awssdk.regions.Region.AP_SOUTHEAST_1)
    .build()
  
  def publishEvent(event: String, subject: String = "", attributes: Map[String, String] = Map.empty): Future[String] = {
    val msgAttributes = attributes.map { case (k, v) =>
      k -> MessageAttributeValue.builder()
             .dataType("String")
             .stringValue(v)
             .build()
    }.asJava
    
    snsClient.publish(
      PublishRequest.builder()
        .topicArn(topicArn)
        .message(event)
        .subject(subject)
        .messageAttributes(msgAttributes)
        .build()
    ).asScala.map(_.messageId())
  }
  
  // Fanout: publish ไปยัง multiple SQS queues พร้อมกัน
  def publishFanout(event: String, messageType: String): Future[String] = {
    publishEvent(event, attributes = Map("MessageType" -> messageType))
  }
  
  def close(): Unit = snsClient.close()
}
```

---

## Step 867: CloudWatch Metrics

```scala
// CloudWatchMetrics.scala
import software.amazon.awssdk.services.cloudwatch.CloudWatchAsyncClient
import software.amazon.awssdk.services.cloudwatch.model._
import scala.jdk.FutureConverters._
import scala.concurrent.{ExecutionContext, Future}
import java.time.Instant

class CloudWatchMetricsReporter(namespace: String)(implicit ec: ExecutionContext) {
  
  private val cwClient = CloudWatchAsyncClient.builder()
    .region(software.amazon.awssdk.regions.Region.AP_SOUTHEAST_1)
    .build()
  
  def putMetric(
    metricName: String,
    value: Double,
    unit: StandardUnit = StandardUnit.COUNT,
    dimensions: Map[String, String] = Map.empty
  ): Future[Unit] = {
    
    val dims = dimensions.map { case (k, v) =>
      Dimension.builder().name(k).value(v).build()
    }.toList.asJava
    
    val datum = MetricDatum.builder()
      .metricName(metricName)
      .value(value)
      .unit(unit)
      .timestamp(Instant.now())
      .dimensions(dims)
      .build()
    
    cwClient.putMetricData(
      PutMetricDataRequest.builder()
        .namespace(namespace)
        .metricData(datum)
        .build()
    ).asScala.map(_ => ())
  }
  
  def putBatchMetrics(metrics: List[(String, Double, Map[String, String])]): Future[Unit] = {
    // CloudWatch accepts up to 20 metrics per call
    val batches = metrics.grouped(20)
    
    Future.sequence(batches.map { batch =>
      val datums = batch.map { case (name, value, dims) =>
        MetricDatum.builder()
          .metricName(name)
          .value(value)
          .unit(StandardUnit.COUNT)
          .timestamp(Instant.now())
          .dimensions(dims.map { case (k, v) =>
            Dimension.builder().name(k).value(v).build()
          }.toList.asJava)
          .build()
      }.asJava
      
      cwClient.putMetricData(
        PutMetricDataRequest.builder()
          .namespace(namespace)
          .metricData(datums)
          .build()
      ).asScala
    }.toList).map(_ => ())
  }
  
  // Record business metrics
  def recordOrderCreated(customerId: String, amount: Double): Future[Unit] = {
    putBatchMetrics(List(
      ("OrdersCreated", 1.0, Map("Environment" -> "production")),
      ("OrderRevenue", amount, Map("Environment" -> "production", "Currency" -> "USD"))
    ))
  }
  
  def close(): Unit = cwClient.close()
}
```

---

## Step 868: Kinesis Streams

```scala
// KinesisService.scala
import software.amazon.awssdk.services.kinesis.KinesisAsyncClient
import software.amazon.awssdk.services.kinesis.model._
import software.amazon.awssdk.core.SdkBytes
import scala.jdk.FutureConverters._
import scala.concurrent.{ExecutionContext, Future}

class KinesisProducer(streamName: String)(implicit ec: ExecutionContext) {
  
  private val kinesisClient = KinesisAsyncClient.builder()
    .region(software.amazon.awssdk.regions.Region.AP_SOUTHEAST_1)
    .build()
  
  def putRecord(data: String, partitionKey: String): Future[String] = {
    kinesisClient.putRecord(
      PutRecordRequest.builder()
        .streamName(streamName)
        .data(SdkBytes.fromUtf8String(data))
        .partitionKey(partitionKey)
        .build()
    ).asScala.map(_.sequenceNumber())
  }
  
  def putRecords(records: List[(String, String)]): Future[Int] = {
    val entries = records.map { case (data, partitionKey) =>
      PutRecordsRequestEntry.builder()
        .data(SdkBytes.fromUtf8String(data))
        .partitionKey(partitionKey)
        .build()
    }.asJava
    
    kinesisClient.putRecords(
      PutRecordsRequest.builder()
        .streamName(streamName)
        .records(entries)
        .build()
    ).asScala.map { response =>
      response.records().size() - response.failedRecordCount()
    }
  }
  
  def close(): Unit = kinesisClient.close()
}
```

---

## Step 869: Secrets Manager

```scala
// SecretsManagerService.scala
import software.amazon.awssdk.services.secretsmanager.SecretsManagerAsyncClient
import software.amazon.awssdk.services.secretsmanager.model._
import spray.json._
import spray.json.DefaultJsonProtocol._
import scala.jdk.FutureConverters._
import scala.concurrent.{ExecutionContext, Future}

class SecretsService(implicit ec: ExecutionContext) {
  
  private val client = SecretsManagerAsyncClient.builder()
    .region(software.amazon.awssdk.regions.Region.AP_SOUTHEAST_1)
    .build()
  
  // Cache secrets (rotate every hour)
  private val cache = scala.collection.concurrent.TrieMap[String, (String, Long)]()
  private val CACHE_TTL_MS = 3600 * 1000L
  
  def getSecret(secretName: String): Future[String] = {
    val now = System.currentTimeMillis()
    
    // Check cache
    cache.get(secretName) match {
      case Some((value, timestamp)) if now - timestamp < CACHE_TTL_MS =>
        Future.successful(value)
      case _ =>
        // Fetch from Secrets Manager
        client.getSecretValue(
          GetSecretValueRequest.builder()
            .secretId(secretName)
            .build()
        ).asScala.map { response =>
          val secret = response.secretString()
          cache(secretName) = (secret, now)
          secret
        }
    }
  }
  
  def getSecretAsJson(secretName: String): Future[Map[String, String]] = {
    getSecret(secretName).map { secret =>
      implicit val mapFormat: JsonFormat[Map[String, String]] = mapFormat[String, String]
      secret.parseJson.convertTo[Map[String, String]]
    }
  }
  
  // Database credentials
  def getDatabaseCredentials(secretName: String = "myapp/database"): Future[(String, String)] = {
    getSecretAsJson(secretName).map { json =>
      (json("username"), json("password"))
    }
  }
  
  def close(): Unit = client.close()
}
```

---

## Step 870: AWS Best Practices

```scala
// AWSBestPractices.scala

object AWSBestPractices {
  
  /*
  ===== IAM Best Practices =====
  
  1. Least privilege: ให้ permission เฉพาะที่จำเป็น
  
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "s3:GetObject",
          "s3:PutObject"
        ],
        "Resource": "arn:aws:s3:::my-app-bucket/orders/*"
      },
      {
        "Effect": "Allow",
        "Action": [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:UpdateItem",
          "dynamodb:Query"
        ],
        "Resource": "arn:aws:dynamodb:ap-southeast-1:123456789:table/Orders"
      }
    ]
  }
  
  2. IRSA (IAM Roles for Service Accounts) สำหรับ Kubernetes:
     - ไม่ใช้ AWS credentials ใน environment variables
     - ใช้ IAM role gated by OIDC
  
  3. Secrets rotation:
     - Enable automatic rotation ใน Secrets Manager
     - Application ต้อง handle secrets refresh
  
  ===== Cost Optimization =====
  
  S3:
  - ใช้ S3 Intelligent-Tiering สำหรับ data ที่ access pattern ไม่แน่นอน
  - Enable S3 Lifecycle rules ลบ old objects
  - ใช้ S3 Select แทน download ทั้ง file
  
  DynamoDB:
  - On-demand billing สำหรับ unpredictable workloads
  - Provisioned + Auto Scaling สำหรับ predictable
  - DAX (DynamoDB Accelerator) สำหรับ microsecond reads
  - TTL สำหรับ auto-delete expired items
  
  SQS:
  - Long polling (waitTimeSeconds = 20) ลด API calls
  - Batch operations (sendMessageBatch, deleteMessageBatch)
  - FIFO queues เฉพาะเมื่อ order matters (expensive)
  
  Lambda:
  - ARM64 (Graviton2): 20% cheaper, 20% faster
  - Provisioned Concurrency สำหรับ cold start sensitive
  - Right-size memory (test with Lambda Power Tuning)
  
  ===== Resilience =====
  
  Multi-AZ:
  - DynamoDB: automatically multi-AZ
  - SQS: automatically multi-AZ
  - S3: automatically multi-AZ
  
  Retry Strategy:
  - Use AWS SDK built-in retry with exponential backoff
  - AWS SDK v2 default: 3 retries with exponential backoff
  
  Timeouts:
  - Always set connection and read timeouts
  - Circuit breaker for downstream services
  */
  
  def main(args: Array[String]): Unit = {
    println("AWS Best Practices for Scala Applications:")
    println("1. Use IAM roles, not access keys")
    println("2. Encrypt at rest (S3 SSE, DynamoDB encryption)")
    println("3. Encrypt in transit (HTTPS only)")
    println("4. Use VPC endpoints for private AWS access")
    println("5. Enable CloudTrail for audit logging")
    println("6. Set resource-based policies on S3/SQS")
    println("7. Use Secrets Manager for credentials")
    println("8. Right-size Lambda memory for cost optimization")
  }
}
```

---

## สรุป Part 87: AWS SDK สำหรับ Scala

| Service | Use Case | SDK Pattern |
|---------|----------|-------------|
| S3 | Object storage, file sharing | Async client + Transfer Manager |
| SQS | Message queue, decoupling | Long polling + Batch |
| DynamoDB | NoSQL, key-value, time-series | Query/Scan + Transactions |
| Lambda | Serverless functions, event processing | RequestHandler |
| SNS | Event fanout, pub/sub | Publish + Subscriptions |
| CloudWatch | Metrics, logs, alarms | PutMetricData |
| Kinesis | Real-time streaming | PutRecords |
| Secrets Manager | Secure credentials | GetSecretValue + Cache |

### Performance Tips
1. Reuse AWS clients (expensive to create)
2. Use async clients + Future.sequence for parallel
3. Batch operations when possible
4. Cache secrets (rotate hourly)
5. Use Transfer Manager for large S3 files

---

## แบบฝึกหัด Part 87

1. **S3 Data Pipeline**: สร้าง pipeline ที่ read files จาก S3, process ด้วย Spark, และ write results กลับ S3 ด้วย multipart upload

2. **DynamoDB Single Table Design**: design single table schema สำหรับ e-commerce ที่มี Orders, Customers, Products โดยใช้ composite keys และ GSIs

3. **Event-Driven Architecture**: implement order processing ด้วย SNS fanout → multiple SQS queues → Lambda processors

4. **Lambda Cold Start**: optimize Lambda cold start สำหรับ Scala JVM โดยใช้ SnapStart, Provisioned Concurrency, และ GraalVM native image

5. **Cost Monitor**: สร้าง CloudWatch dashboard + alerts ที่ monitor AWS costs per service และ alert เมื่อ cost เกิน budget

---

## ไปต่อ: Part 88 — Observability
ใน Part ถัดไปจะเรียน Structured logging, Prometheus metrics, OpenTelemetry, Jaeger tracing

[→ Part 88: Observability](./part-88-observability.md)
