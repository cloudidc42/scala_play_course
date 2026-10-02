# Part 85: gRPC with Scala — Steps 841-850

## บทนำ: gRPC

gRPC เป็น high-performance RPC framework จาก Google ที่ใช้ Protocol Buffers สำหรับ serialization เร็วกว่า REST JSON 5-10x เหมาะสำหรับ internal microservice communication

---

## Step 841: ScalaPB Setup

```scala
// build.sbt
libraryDependencies ++= Seq(
  "io.grpc"               % "grpc-netty-shaded" % "1.59.0",
  "io.grpc"               % "grpc-stub"         % "1.59.0",
  "com.thesamet.scalapb" %% "scalapb-runtime-grpc" % scalapb.compiler.Version.scalapbVersion,
  "com.thesamet.scalapb" %% "scalapb-runtime"      % scalapb.compiler.Version.scalapbVersion % "protobuf"
)

// project/plugins.sbt
addSbtPlugin("com.thesamet" % "sbt-protoc" % "1.0.6")
libraryDependencies += "com.thesamet.scalapb" %% "compilerplugin" % "0.11.14"

// Protobuf compile settings
Compile / PB.targets := Seq(
  scalapb.gen(grpc = true) -> (Compile / sourceManaged).value / "scalapb"
)
```

---

## Step 842: Proto Definition

```protobuf
// src/main/protobuf/order_service.proto
syntax = "proto3";

package com.example.order;

option java_package = "com.example.order.proto";
option java_outer_classname = "OrderServiceProto";

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

// ===== Messages =====

message OrderItem {
  string product_id  = 1;
  int32  quantity    = 2;
  double unit_price  = 3;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  PENDING   = 1;
  CONFIRMED = 2;
  SHIPPED   = 3;
  DELIVERED = 4;
  CANCELLED = 5;
}

message Order {
  string                    id          = 1;
  string                    customer_id = 2;
  repeated OrderItem        items       = 3;
  OrderStatus               status      = 4;
  double                    total       = 5;
  string                    currency    = 6;
  google.protobuf.Timestamp created_at  = 7;
  google.protobuf.Timestamp updated_at  = 8;
}

// ===== Requests/Responses =====

message CreateOrderRequest {
  string             customer_id = 1;
  repeated OrderItem items       = 2;
  string             currency    = 3;
}

message CreateOrderResponse {
  Order order = 1;
}

message GetOrderRequest {
  string order_id = 1;
}

message GetOrderResponse {
  Order order = 1;
}

message ListOrdersRequest {
  string customer_id = 1;
  int32  page        = 2;
  int32  page_size   = 3;
  OrderStatus status_filter = 4;
}

message ListOrdersResponse {
  repeated Order orders     = 1;
  int32          total      = 2;
  int32          page       = 3;
  bool           has_more   = 4;
}

message UpdateOrderStatusRequest {
  string      order_id = 1;
  OrderStatus status   = 2;
}

message StreamOrderUpdatesRequest {
  string customer_id = 1;
}

// ===== Service Definition =====
service OrderService {
  // Unary: create order
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
  
  // Unary: get order
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
  
  // Unary: list orders
  rpc ListOrders(ListOrdersRequest) returns (ListOrdersResponse);
  
  // Unary: update status
  rpc UpdateOrderStatus(UpdateOrderStatusRequest) returns (Order);
  
  // Server streaming: stream order updates
  rpc StreamOrderUpdates(StreamOrderUpdatesRequest) returns (stream Order);
  
  // Client streaming: batch create
  rpc BatchCreateOrders(stream CreateOrderRequest) returns (ListOrdersResponse);
  
  // Bidirectional streaming: live order tracking
  rpc TrackOrders(stream GetOrderRequest) returns (stream Order);
}
```

---

## Step 843: gRPC Server Implementation

```scala
// OrderServiceGrpcImpl.scala
package com.example.order.grpc

import com.example.order.proto._
import io.grpc.stub.StreamObserver
import io.grpc.{Status, StatusException}
import com.google.protobuf.timestamp.Timestamp
import scala.concurrent.{ExecutionContext, Future, Promise}
import scala.collection.concurrent.TrieMap

class OrderServiceGrpcImpl()(implicit ec: ExecutionContext)
    extends OrderServiceGrpc.OrderService {
  
  // In-memory store (production ใช้ database)
  private val orders = TrieMap[String, Order]()
  private val orderSubscribers = TrieMap[String, List[StreamObserver[Order]]]()
  
  // ===== Unary: Create Order =====
  override def createOrder(request: CreateOrderRequest): Future[CreateOrderResponse] = {
    // Validate
    if (request.customerId.isEmpty) {
      return Future.failed(new StatusException(
        Status.INVALID_ARGUMENT.withDescription("customer_id is required")
      ))
    }
    
    if (request.items.isEmpty) {
      return Future.failed(new StatusException(
        Status.INVALID_ARGUMENT.withDescription("At least one item required")
      ))
    }
    
    val orderId = java.util.UUID.randomUUID().toString
    val total   = request.items.map(i => i.unitPrice * i.quantity).sum
    val now     = Timestamp(seconds = java.time.Instant.now().getEpochSecond)
    
    val order = Order(
      id         = orderId,
      customerId = request.customerId,
      items      = request.items,
      status     = OrderStatus.PENDING,
      total      = total,
      currency   = request.currency,
      createdAt  = Some(now),
      updatedAt  = Some(now)
    )
    
    orders(orderId) = order
    
    // Notify subscribers
    notifySubscribers(request.customerId, order)
    
    Future.successful(CreateOrderResponse(order = Some(order)))
  }
  
  // ===== Unary: Get Order =====
  override def getOrder(request: GetOrderRequest): Future[GetOrderResponse] = {
    orders.get(request.orderId) match {
      case Some(order) => Future.successful(GetOrderResponse(order = Some(order)))
      case None        => Future.failed(new StatusException(
        Status.NOT_FOUND.withDescription(s"Order ${request.orderId} not found")
      ))
    }
  }
  
  // ===== Unary: List Orders =====
  override def listOrders(request: ListOrdersRequest): Future[ListOrdersResponse] = {
    val filtered = orders.values.toList
      .filter(_.customerId == request.customerId)
      .filter { o =>
        if (request.statusFilter == OrderStatus.ORDER_STATUS_UNSPECIFIED) true
        else o.status == request.statusFilter
      }
      .sortBy(_.createdAt.map(_.seconds).getOrElse(0L))(Ordering[Long].reverse)
    
    val page     = request.page.max(1)
    val pageSize = request.pageSize.max(1).min(100)
    val total    = filtered.size
    val slice    = filtered.slice((page - 1) * pageSize, page * pageSize)
    
    Future.successful(ListOrdersResponse(
      orders  = slice,
      total   = total,
      page    = page,
      hasMore = (page * pageSize) < total
    ))
  }
  
  // ===== Unary: Update Status =====
  override def updateOrderStatus(request: UpdateOrderStatusRequest): Future[Order] = {
    orders.get(request.orderId) match {
      case Some(order) =>
        val updated = order.copy(
          status    = request.status,
          updatedAt = Some(Timestamp(seconds = java.time.Instant.now().getEpochSecond))
        )
        orders(request.orderId) = updated
        notifySubscribers(updated.customerId, updated)
        Future.successful(updated)
      
      case None =>
        Future.failed(new StatusException(Status.NOT_FOUND))
    }
  }
  
  // ===== Server Streaming: Stream Order Updates =====
  override def streamOrderUpdates(
    request: StreamOrderUpdatesRequest,
    responseObserver: StreamObserver[Order]
  ): Unit = {
    // Register subscriber
    val customerId = request.customerId
    val existing = orderSubscribers.getOrElse(customerId, List.empty)
    orderSubscribers(customerId) = existing :+ responseObserver
    
    // Send existing orders
    orders.values.filter(_.customerId == customerId).foreach { order =>
      responseObserver.onNext(order)
    }
    
    // Note: keep connection open until client disconnects
    // In production, handle cancellation properly
  }
  
  // ===== Client Streaming: Batch Create =====
  override def batchCreateOrders(
    responseObserver: StreamObserver[ListOrdersResponse]
  ): StreamObserver[CreateOrderRequest] = {
    
    val createdOrders = scala.collection.mutable.ListBuffer[Order]()
    
    new StreamObserver[CreateOrderRequest] {
      override def onNext(request: CreateOrderRequest): Unit = {
        val orderId = java.util.UUID.randomUUID().toString
        val total   = request.items.map(i => i.unitPrice * i.quantity).sum
        val now     = Timestamp(seconds = java.time.Instant.now().getEpochSecond)
        
        val order = Order(
          id         = orderId,
          customerId = request.customerId,
          items      = request.items,
          status     = OrderStatus.PENDING,
          total      = total,
          currency   = request.currency,
          createdAt  = Some(now),
          updatedAt  = Some(now)
        )
        
        orders(orderId) = order
        createdOrders += order
      }
      
      override def onError(t: Throwable): Unit = {
        println(s"Client streaming error: ${t.getMessage}")
      }
      
      override def onCompleted(): Unit = {
        // Send response when all orders processed
        responseObserver.onNext(ListOrdersResponse(
          orders  = createdOrders.toList,
          total   = createdOrders.size,
          page    = 1,
          hasMore = false
        ))
        responseObserver.onCompleted()
      }
    }
  }
  
  // ===== Bidirectional Streaming: Track Orders =====
  override def trackOrders(
    responseObserver: StreamObserver[Order]
  ): StreamObserver[GetOrderRequest] = {
    
    new StreamObserver[GetOrderRequest] {
      override def onNext(request: GetOrderRequest): Unit = {
        orders.get(request.orderId) match {
          case Some(order) => responseObserver.onNext(order)
          case None        => // skip not found
        }
      }
      
      override def onError(t: Throwable): Unit = {
        println(s"Bidi streaming error: ${t.getMessage}")
      }
      
      override def onCompleted(): Unit = {
        responseObserver.onCompleted()
      }
    }
  }
  
  // Helper: notify streaming subscribers
  private def notifySubscribers(customerId: String, order: Order): Unit = {
    orderSubscribers.get(customerId).foreach { subscribers =>
      subscribers.foreach { observer =>
        try { observer.onNext(order) }
        catch { case _: Exception => /* subscriber disconnected */ }
      }
    }
  }
}
```

---

## Step 844: gRPC Server Startup

```scala
// GrpcServer.scala
package com.example.order.grpc

import io.grpc.{Server, ServerBuilder}
import io.grpc.netty.shaded.io.grpc.netty.NettyServerBuilder
import java.util.concurrent.{Executors, TimeUnit}
import scala.concurrent.ExecutionContext

object GrpcServer {
  
  def main(args: Array[String]): Unit = {
    val port = sys.env.getOrElse("GRPC_PORT", "9090").toInt
    
    implicit val ec: ExecutionContext = ExecutionContext.fromExecutorService(
      Executors.newFixedThreadPool(Runtime.getRuntime.availableProcessors() * 2)
    )
    
    val serviceImpl = new OrderServiceGrpcImpl()
    
    val server: Server = NettyServerBuilder
      .forPort(port)
      .addService(OrderServiceGrpc.bindService(serviceImpl, ec))
      // TLS (production)
      // .useTransportSecurity(certChainFile, privateKeyFile)
      // Interceptors
      .intercept(new AuthInterceptor())
      .intercept(new LoggingInterceptor())
      .intercept(new MetricsInterceptor())
      // Resource limits
      .maxInboundMessageSize(10 * 1024 * 1024) // 10MB
      .maxInboundMetadataSize(1024 * 1024)       // 1MB
      .keepAliveTime(30, TimeUnit.SECONDS)
      .keepAliveTimeout(10, TimeUnit.SECONDS)
      .maxConnectionAge(5, TimeUnit.MINUTES)
      .maxConnectionAgeGrace(30, TimeUnit.SECONDS)
      .build()
      .start()
    
    println(s"gRPC Server started on port $port")
    
    // Shutdown hook
    sys.addShutdownHook {
      println("Shutting down gRPC server...")
      server.shutdown()
      server.awaitTermination(30, TimeUnit.SECONDS)
      println("gRPC server shut down")
    }
    
    server.awaitTermination()
  }
}

// ===== Interceptors =====
import io.grpc._

class AuthInterceptor extends ServerInterceptor {
  override def interceptCall[ReqT, RespT](
    call: ServerCall[ReqT, RespT],
    headers: Metadata,
    next: ServerCallHandler[ReqT, RespT]
  ): ServerCall.Listener[ReqT] = {
    
    val authToken = headers.get(Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER))
    
    if (authToken == null || !authToken.startsWith("Bearer ")) {
      call.close(Status.UNAUTHENTICATED.withDescription("Missing or invalid auth token"), new Metadata())
      return new ServerCall.Listener[ReqT] {}
    }
    
    // Validate JWT token here
    next.startCall(call, headers)
  }
}

class LoggingInterceptor extends ServerInterceptor {
  override def interceptCall[ReqT, RespT](
    call: ServerCall[ReqT, RespT],
    headers: Metadata,
    next: ServerCallHandler[ReqT, RespT]
  ): ServerCall.Listener[ReqT] = {
    val methodName = call.getMethodDescriptor.getFullMethodName
    val start = System.currentTimeMillis()
    
    val delegate = next.startCall(
      new ForwardingServerCall.SimpleForwardingServerCall[ReqT, RespT](call) {
        override def close(status: Status, trailers: Metadata): Unit = {
          val elapsed = System.currentTimeMillis() - start
          println(s"gRPC $methodName: ${status.getCode} (${elapsed}ms)")
          super.close(status, trailers)
        }
      },
      headers
    )
    
    delegate
  }
}

class MetricsInterceptor extends ServerInterceptor {
  override def interceptCall[ReqT, RespT](
    call: ServerCall[ReqT, RespT],
    headers: Metadata,
    next: ServerCallHandler[ReqT, RespT]
  ): ServerCall.Listener[ReqT] = {
    // Record metrics (Prometheus)
    next.startCall(call, headers)
  }
}
```

---

## Step 845: gRPC Client

```scala
// OrderServiceClient.scala
package com.example.order.grpc.client

import com.example.order.proto._
import io.grpc.{ManagedChannel, ManagedChannelBuilder}
import io.grpc.stub.StreamObserver
import scala.concurrent.{Future, ExecutionContext, Promise}
import java.util.concurrent.TimeUnit

class OrderServiceClient(host: String, port: Int)(implicit ec: ExecutionContext) {
  
  private val channel: ManagedChannel = ManagedChannelBuilder
    .forAddress(host, port)
    .usePlaintext()  // ไม่ใช้ TLS (dev)
    // production: .useTransportSecurity()
    .keepAliveTime(30, TimeUnit.SECONDS)
    .keepAliveWithoutCalls(true)
    .build()
  
  // Async stub (Future-based)
  private val asyncStub = OrderServiceGrpc.stub(channel)
  
  // Blocking stub (สำหรับ simple use cases)
  private val blockingStub = OrderServiceGrpc.blockingStub(channel)
  
  // ===== Unary calls =====
  def createOrder(customerId: String, items: List[(String, Int, Double)]): Future[Order] = {
    val request = CreateOrderRequest(
      customerId = customerId,
      items = items.map { case (productId, qty, price) =>
        OrderItem(productId = productId, quantity = qty, unitPrice = price)
      },
      currency = "USD"
    )
    
    asyncStub.createOrder(request).map(_.getOrder)
  }
  
  def getOrder(orderId: String): Future[Order] = {
    asyncStub.getOrder(GetOrderRequest(orderId = orderId)).map(_.getOrder)
  }
  
  def listOrders(customerId: String, page: Int = 1, pageSize: Int = 20): Future[ListOrdersResponse] = {
    asyncStub.listOrders(ListOrdersRequest(
      customerId = customerId,
      page       = page,
      pageSize   = pageSize
    ))
  }
  
  // ===== Server Streaming =====
  def streamOrderUpdates(customerId: String, onUpdate: Order => Unit): Unit = {
    asyncStub.streamOrderUpdates(
      StreamOrderUpdatesRequest(customerId = customerId),
      new StreamObserver[Order] {
        override def onNext(order: Order): Unit    = onUpdate(order)
        override def onError(t: Throwable): Unit   = println(s"Stream error: ${t.getMessage}")
        override def onCompleted(): Unit            = println("Stream completed")
      }
    )
  }
  
  // ===== Client Streaming =====
  def batchCreateOrders(orders: List[(String, List[(String, Int, Double)])]): Future[ListOrdersResponse] = {
    val promise = Promise[ListOrdersResponse]()
    
    val responseObserver = new StreamObserver[ListOrdersResponse] {
      override def onNext(response: ListOrdersResponse): Unit = promise.success(response)
      override def onError(t: Throwable): Unit = promise.failure(t)
      override def onCompleted(): Unit = ()
    }
    
    val requestObserver = asyncStub.batchCreateOrders(responseObserver)
    
    // Send all orders
    orders.foreach { case (customerId, items) =>
      requestObserver.onNext(CreateOrderRequest(
        customerId = customerId,
        items = items.map { case (pid, qty, price) =>
          OrderItem(productId = pid, quantity = qty, unitPrice = price)
        }
      ))
    }
    
    requestObserver.onCompleted()
    
    promise.future
  }
  
  // ===== Bidirectional Streaming =====
  def trackOrdersLive(orderIds: List[String], onUpdate: Order => Unit): Unit = {
    val requestObserver = asyncStub.trackOrders(
      new StreamObserver[Order] {
        override def onNext(order: Order): Unit = onUpdate(order)
        override def onError(t: Throwable): Unit = println(s"Track error: ${t.getMessage}")
        override def onCompleted(): Unit = println("Tracking completed")
      }
    )
    
    // Send requests
    orderIds.foreach { orderId =>
      requestObserver.onNext(GetOrderRequest(orderId = orderId))
    }
    
    requestObserver.onCompleted()
  }
  
  def shutdown(): Unit = {
    channel.shutdown().awaitTermination(5, TimeUnit.SECONDS)
  }
}

// ===== Client Demo =====
object ClientDemo {
  def main(args: Array[String]): Unit = {
    import scala.concurrent.{Await, ExecutionContext}
    import scala.concurrent.duration._
    import ExecutionContext.Implicits.global
    
    val client = new OrderServiceClient("localhost", 9090)
    
    try {
      println("=== Creating order ===")
      val order = Await.result(
        client.createOrder("C001", List(
          ("P001", 2, 999.99),
          ("P002", 1, 49.99)
        )),
        10.seconds
      )
      println(s"Created: ${order.id}, total: ${order.total}")
      
      println("\n=== Getting order ===")
      val fetched = Await.result(client.getOrder(order.id), 10.seconds)
      println(s"Got: ${fetched.id}, status: ${fetched.status}")
      
      println("\n=== Listing orders ===")
      val list = Await.result(client.listOrders("C001"), 10.seconds)
      println(s"Total orders: ${list.total}")
      
      println("\n=== Server streaming ===")
      client.streamOrderUpdates("C001", order => println(s"Update: ${order.id} -> ${order.status}"))
      Thread.sleep(1000)
      
      println("\n=== Batch create ===")
      val batch = Await.result(
        client.batchCreateOrders(List(
          ("C001", List(("P003", 3, 10.0))),
          ("C002", List(("P004", 1, 500.0))),
          ("C003", List(("P005", 2, 75.0)))
        )),
        10.seconds
      )
      println(s"Batch created: ${batch.total} orders")
      
    } finally {
      client.shutdown()
    }
  }
}
```

---

## Step 846: gRPC Error Handling

```scala
// GrpcErrorHandling.scala
import io.grpc.{Status, StatusException, StatusRuntimeException, Metadata}
import scala.concurrent.Future
import scala.util.{Try, Success, Failure}
import scala.concurrent.ExecutionContext.Implicits.global

object GrpcErrorHandling {
  
  // ===== Server-side Error Mapping =====
  def toGrpcStatus(ex: Exception): Status = ex match {
    case _: NoSuchElementException    => Status.NOT_FOUND
    case _: IllegalArgumentException  => Status.INVALID_ARGUMENT
    case _: SecurityException         => Status.PERMISSION_DENIED
    case _: UnsupportedOperationException => Status.UNIMPLEMENTED
    case _                            => Status.INTERNAL
  }
  
  def handleError[T](f: => Future[T]): Future[T] = {
    f.recoverWith {
      case ex: StatusException         => Future.failed(ex)
      case ex: StatusRuntimeException  => Future.failed(ex)
      case ex: IllegalArgumentException =>
        Future.failed(new StatusException(
          Status.INVALID_ARGUMENT.withDescription(ex.getMessage).withCause(ex)
        ))
      case ex: Exception =>
        Future.failed(new StatusException(
          Status.INTERNAL.withDescription("Internal server error").withCause(ex)
        ))
    }
  }
  
  // ===== Client-side Error Handling =====
  def handleClientError[T](f: Future[T]): Future[Either[String, T]] = {
    f.map(Right(_)).recover {
      case ex: StatusRuntimeException =>
        val status = ex.getStatus
        status.getCode match {
          case Status.Code.NOT_FOUND         => Left(s"Not found: ${status.getDescription}")
          case Status.Code.INVALID_ARGUMENT  => Left(s"Invalid: ${status.getDescription}")
          case Status.Code.UNAUTHENTICATED   => Left("Authentication required")
          case Status.Code.PERMISSION_DENIED => Left("Access denied")
          case Status.Code.UNAVAILABLE       => Left("Service unavailable")
          case _                             => Left(s"Error: ${status.getDescription}")
        }
      case ex =>
        Left(s"Unexpected error: ${ex.getMessage}")
    }
  }
  
  // ===== Retry with backoff =====
  def withRetry[T](maxAttempts: Int, delayMs: Long = 1000)(f: => Future[T]): Future[T] = {
    f.recoverWith {
      case ex: StatusRuntimeException if isRetryable(ex.getStatus.getCode) && maxAttempts > 1 =>
        println(s"Retrying gRPC call, attempts left: ${maxAttempts - 1}")
        Thread.sleep(delayMs)
        withRetry(maxAttempts - 1, delayMs * 2)(f)
    }
  }
  
  def isRetryable(code: Status.Code): Boolean = code match {
    case Status.Code.UNAVAILABLE   => true
    case Status.Code.RESOURCE_EXHAUSTED => true
    case Status.Code.DEADLINE_EXCEEDED  => true
    case _                          => false
  }
  
  def main(args: Array[String]): Unit = {
    val result = handleClientError(
      withRetry(maxAttempts = 3) {
        // simulate a gRPC call
        Future.failed(new StatusRuntimeException(Status.UNAVAILABLE.withDescription("Server is starting")))
      }
    )
    
    result.foreach {
      case Right(v)  => println(s"Success: $v")
      case Left(err) => println(s"Error: $err")
    }
    
    Thread.sleep(5000)
  }
}
```

---

## Step 847: gRPC with TLS

```scala
// GrpcTLS.scala
import io.grpc.netty.shaded.io.grpc.netty.{GrpcSslContexts, NettyChannelBuilder, NettyServerBuilder}
import io.netty.handler.ssl.{ClientAuth, SslContextBuilder}
import java.io.File

object GrpcTLS {
  
  // ===== Server with TLS =====
  def buildServerWithTLS(port: Int, certFile: File, keyFile: File): io.grpc.Server = {
    val sslContext = GrpcSslContexts
      .configure(SslContextBuilder.forServer(certFile, keyFile))
      .clientAuth(ClientAuth.OPTIONAL)  // or REQUIRE for mTLS
      .build()
    
    NettyServerBuilder
      .forPort(port)
      .sslContext(sslContext)
      // Add services...
      .build()
  }
  
  // ===== Client with TLS =====
  def buildClientWithTLS(host: String, port: Int, trustCert: File): io.grpc.ManagedChannel = {
    val sslContext = GrpcSslContexts
      .configure(SslContextBuilder.forClient().trustManager(trustCert))
      .build()
    
    NettyChannelBuilder
      .forAddress(host, port)
      .sslContext(sslContext)
      .build()
  }
  
  // ===== mTLS (mutual TLS) =====
  def buildClientWithMTLS(
    host: String, port: Int,
    certFile: File, keyFile: File, trustCert: File
  ): io.grpc.ManagedChannel = {
    val sslContext = GrpcSslContexts
      .configure(SslContextBuilder.forClient()
        .keyManager(certFile, keyFile)
        .trustManager(trustCert))
      .build()
    
    NettyChannelBuilder
      .forAddress(host, port)
      .sslContext(sslContext)
      .build()
  }
}
```

---

## Step 848: gRPC Reflection

```scala
// GrpcReflection.scala
import io.grpc.protobuf.services.ProtoReflectionService
import io.grpc.ServerBuilder

// Enable gRPC reflection สำหรับ development tools (grpcurl, grpc-ui)
object GrpcReflectionServer {
  
  def buildServerWithReflection(port: Int)(implicit ec: scala.concurrent.ExecutionContext): io.grpc.Server = {
    val serviceImpl = new OrderServiceGrpcImpl()
    
    ServerBuilder
      .forPort(port)
      .addService(OrderServiceGrpc.bindService(serviceImpl, ec))
      .addService(ProtoReflectionService.newInstance()) // enable reflection
      .build()
  }
}

// ===== grpcurl commands =====
/*
# List all services
grpcurl -plaintext localhost:9090 list

# List methods
grpcurl -plaintext localhost:9090 list com.example.order.OrderService

# Call a method
grpcurl -plaintext -d '{"customer_id":"C001","items":[{"product_id":"P001","quantity":2,"unit_price":99.99}],"currency":"USD"}' \
  localhost:9090 com.example.order.OrderService/CreateOrder

# Stream
grpcurl -plaintext -d '{"customer_id":"C001"}' \
  localhost:9090 com.example.order.OrderService/StreamOrderUpdates
*/
```

---

## Step 849: gRPC Performance Tuning

```scala
// GrpcPerformanceTuning.scala
import io.grpc.netty.shaded.io.grpc.netty.NettyChannelBuilder
import java.util.concurrent.{Executors, TimeUnit}

object GrpcPerformanceTuning {
  
  // ===== Channel Pool =====
  // gRPC channel สามารถ multiplex หลาย requests บน single connection
  // แต่ถ้า throughput สูงมาก ใช้หลาย channels
  
  class ChannelPool(host: String, port: Int, poolSize: Int = 4) {
    private val channels = (1 to poolSize).map { _ =>
      NettyChannelBuilder
        .forAddress(host, port)
        .usePlaintext()
        .keepAliveTime(30, TimeUnit.SECONDS)
        .keepAliveWithoutCalls(true)
        .build()
    }.toArray
    
    private var index = 0
    
    def getChannel(): io.grpc.ManagedChannel = {
      val ch = channels(index % poolSize)
      index += 1
      ch
    }
    
    def shutdown(): Unit = channels.foreach(_.shutdown())
  }
  
  // ===== Compression =====
  // Enable compression for large payloads
  def createCompressedStub(channel: io.grpc.ManagedChannel) = {
    OrderServiceGrpc.stub(channel)
      .withCompression("gzip")
  }
  
  // ===== Deadline =====
  // Set timeout per call
  def callWithDeadline[T](stub: OrderServiceGrpc.OrderServiceStub, timeout: Long, unit: TimeUnit) = {
    stub.withDeadlineAfter(timeout, unit)
  }
  
  // ===== FlowControl =====
  /*
  gRPC HTTP/2 flow control:
  - Server ควร buffer ไม่เกิน maxInboundMessageSize
  - Client ควร set reaasonable deadline
  - Streaming: onNext() calls are async — ระวัง backpressure
  */
  
  def main(args: Array[String]): Unit = {
    println("""
      gRPC Performance Tips:
      1. Reuse channels (expensive to create)
      2. Use channel pool for high throughput
      3. Set deadlines on every call
      4. Enable compression for large payloads
      5. Use server streaming for large result sets
      6. Prefer async stubs over blocking stubs
      7. Use connection multiplexing (HTTP/2 default)
    """)
  }
}
```

---

## Step 850: gRPC Gateway (REST Bridge)

```yaml
# grpc-gateway.yaml (สำหรับ HTTP REST ↔ gRPC bridging)
# ใช้ grpc-gateway หรือ Envoy proxy

---
# envoy.yaml
static_resources:
  listeners:
    - name: listener_0
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 8080
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                codec_type: AUTO
                stat_prefix: ingress_http
                route_config:
                  virtual_hosts:
                    - name: order_service
                      domains: ["*"]
                      routes:
                        - match:
                            prefix: "/api/v1/orders"
                          route:
                            cluster: order_grpc_service
                            timeout: 30s
                http_filters:
                  - name: envoy.filters.http.grpc_json_transcoder
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.grpc_json_transcoder.v3.GrpcJsonTranscoder
                      proto_descriptor: "order_service.pb"
                      services: ["com.example.order.OrderService"]
                      print_options:
                        add_whitespace: true
                  - name: envoy.filters.http.router

  clusters:
    - name: order_grpc_service
      connect_timeout: 0.25s
      type: LOGICAL_DNS
      lb_policy: ROUND_ROBIN
      http2_protocol_options: {}
      load_assignment:
        cluster_name: order_grpc_service
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: order-service
                      port_value: 9090
```

---

## สรุป Part 85: gRPC with Scala

| RPC Type | Pattern | Use Case |
|---------|---------|----------|
| Unary | Request → Response | CRUD operations |
| Server Streaming | Request → Stream | Real-time updates |
| Client Streaming | Stream → Response | Batch upload |
| Bidirectional | Stream ↔ Stream | Chat, live tracking |

### gRPC vs REST

| Aspect | gRPC | REST |
|--------|------|------|
| Protocol | HTTP/2 | HTTP/1.1 |
| Serialization | Protobuf | JSON |
| Speed | ~10x faster | Baseline |
| Schema | Required | Optional |
| Streaming | Native | WebSocket/SSE |
| Browser | Limited | Native |

---

## แบบฝึกหัด Part 85

1. **Full gRPC Service**: สร้าง Product Catalog gRPC service ที่มี CRUD + server streaming สำหรับ price updates

2. **Error Handling**: implement comprehensive error handling ที่ map domain errors ไปยัง gRPC status codes และ retry logic

3. **mTLS**: configure mutual TLS สำหรับ secure service-to-service communication

4. **Interceptors**: สร้าง server interceptors สำหรับ authentication, logging, rate limiting, และ metrics collection

5. **REST Bridge**: setup Envoy proxy เพื่อ expose gRPC service เป็น REST API โดยไม่ต้องแก้ Scala code

---

## ไปต่อ: Part 86 — Service Mesh
ใน Part ถัดไปจะเรียน Istio, traffic management, mTLS, observability

[→ Part 86: Service Mesh](./part-86-service-mesh.md)
