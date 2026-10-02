# Part 34: Akka Actors Basics

## Steps 331-340: Actor Model, ActorSystem, Messages, State, Lifecycle

---

## Step 331: Actor Model คืออะไร

```
Actor Model = model ของ concurrent computation ที่:
- แต่ละ Actor มี state ของตัวเอง (ไม่แชร์ state กัน)
- Communicate ผ่าน Messages เท่านั้น
- Process messages ทีละ 1 message (thread-safe by design)
- สามารถ create child actors, send messages, change behavior

ข้อดี:
- No shared mutable state → ไม่ต้องใช้ locks
- Location transparent → actor ที่ไหนก็ได้ (local/remote)
- Fault tolerant → supervision strategy
- High concurrency → millions of actors in JVM
```

```scala
// build.sbt dependencies:
// libraryDependencies += "com.typesafe.akka" %% "akka-actor-typed" % "2.7.0"
// libraryDependencies += "com.typesafe.akka" %% "akka-actor-testkit-typed" % "2.7.0" % Test

import akka.actor.typed._
import akka.actor.typed.scaladsl._
import scala.concurrent.duration._

// Typed Actor — message type is part of the actor type
object HelloActor {
  
  // Protocol — messages this actor accepts
  sealed trait Command
  case class Greet(name: String, replyTo: ActorRef[Greeting]) extends Command
  case class GreetMany(names: List[String], replyTo: ActorRef[List[Greeting]]) extends Command
  
  case class Greeting(message: String)
  
  // Behavior definition
  def apply(): Behavior[Command] = Behaviors.receive { (context, message) =>
    message match {
      case Greet(name, replyTo) =>
        context.log.info(s"Greeting $name")
        replyTo ! Greeting(s"Hello, $name! From ${context.self.path.name}")
        Behaviors.same
        
      case GreetMany(names, replyTo) =>
        val greetings = names.map(n => Greeting(s"Hello, $n!"))
        replyTo ! greetings
        Behaviors.same
    }
  }
}

object ActorBasicsDemo extends App {
  import akka.util.Timeout
  import scala.concurrent.Await
  
  val system = ActorSystem(
    Behaviors.setup[Nothing] { context =>
      val actor = context.spawn(HelloActor(), "hello-actor")
      
      // Ask pattern — request/response
      implicit val timeout: Timeout = 3.seconds
      implicit val ec = context.executionContext
      
      import akka.actor.typed.scaladsl.AskPattern._
      
      val future = actor.ask[HelloActor.Greeting](ref => HelloActor.Greet("World", ref))
      
      import scala.concurrent.ExecutionContext.Implicits.global
      future.foreach(g => println(s"Got: ${g.message}"))
      
      Behaviors.empty
    },
    "basics-system"
  )
  
  Thread.sleep(1000)
  system.terminate()
  Await.ready(system.whenTerminated, 5.seconds)
}
```

---

## Step 332: Stateful Actors

```scala
import akka.actor.typed._
import akka.actor.typed.scaladsl._
import scala.concurrent.duration._

object CounterActor {
  
  sealed trait Command
  case object Increment extends Command
  case object Decrement extends Command
  case class Add(n: Int) extends Command
  case class Reset(value: Int = 0) extends Command
  case class GetValue(replyTo: ActorRef[Int]) extends Command
  case class Subscribe(ref: ActorRef[CounterChanged]) extends Command
  
  case class CounterChanged(old: Int, current: Int)
  
  def apply(initial: Int = 0): Behavior[Command] = 
    counter(initial, Set.empty)
  
  private def counter(
    count: Int,
    subscribers: Set[ActorRef[CounterChanged]]
  ): Behavior[Command] = Behaviors.receive { (context, message) =>
    message match {
      case Increment =>
        val newCount = count + 1
        notifySubscribers(subscribers, count, newCount)
        counter(newCount, subscribers)
        
      case Decrement =>
        val newCount = count - 1
        notifySubscribers(subscribers, count, newCount)
        counter(newCount, subscribers)
        
      case Add(n) =>
        val newCount = count + n
        notifySubscribers(subscribers, count, newCount)
        counter(newCount, subscribers)
        
      case Reset(value) =>
        notifySubscribers(subscribers, count, value)
        counter(value, subscribers)
        
      case GetValue(replyTo) =>
        replyTo ! count
        Behaviors.same
        
      case Subscribe(ref) =>
        counter(count, subscribers + ref)
    }
  }
  
  private def notifySubscribers(
    subscribers: Set[ActorRef[CounterChanged]],
    old: Int,
    current: Int
  ): Unit = subscribers.foreach(_ ! CounterChanged(old, current))
}

// Shopping cart actor — more complex stateful actor
object ShoppingCartActor {
  
  case class Item(id: String, name: String, price: Double, quantity: Int)
  
  sealed trait Command
  case class AddItem(item: Item, replyTo: ActorRef[CartResult]) extends Command
  case class RemoveItem(itemId: String, replyTo: ActorRef[CartResult]) extends Command
  case class UpdateQuantity(itemId: String, qty: Int, replyTo: ActorRef[CartResult]) extends Command
  case class GetCart(replyTo: ActorRef[CartState]) extends Command
  case class Checkout(replyTo: ActorRef[CartResult]) extends Command
  case object Clear extends Command
  
  sealed trait CartResult
  case class ItemAdded(item: Item) extends CartResult
  case class ItemRemoved(itemId: String) extends CartResult
  case class QuantityUpdated(itemId: String, qty: Int) extends CartResult
  case class CheckoutStarted(total: Double, items: List[Item]) extends CartResult
  case class CartError(message: String) extends CartResult
  
  case class CartState(items: Map[String, Item]) {
    def total: Double = items.values.map(i => i.price * i.quantity).sum
    def itemCount: Int = items.values.map(_.quantity).sum
  }
  
  def apply(): Behavior[Command] = active(CartState(Map.empty))
  
  private def active(state: CartState): Behavior[Command] = 
    Behaviors.receive { (context, message) =>
      message match {
        case AddItem(item, replyTo) =>
          val existing = state.items.get(item.id)
          val updated = existing match {
            case Some(existing) => existing.copy(quantity = existing.quantity + item.quantity)
            case None => item
          }
          val newState = state.copy(items = state.items + (item.id -> updated))
          context.log.info(s"Added ${item.name} to cart. Total: ${newState.total}")
          replyTo ! ItemAdded(updated)
          active(newState)
          
        case RemoveItem(itemId, replyTo) =>
          if (!state.items.contains(itemId)) {
            replyTo ! CartError(s"Item $itemId not in cart")
            Behaviors.same
          } else {
            val newState = state.copy(items = state.items - itemId)
            replyTo ! ItemRemoved(itemId)
            active(newState)
          }
          
        case UpdateQuantity(itemId, qty, replyTo) =>
          state.items.get(itemId) match {
            case None => 
              replyTo ! CartError(s"Item $itemId not found")
              Behaviors.same
            case Some(item) if qty <= 0 =>
              val newState = state.copy(items = state.items - itemId)
              replyTo ! QuantityUpdated(itemId, 0)
              active(newState)
            case Some(item) =>
              val newState = state.copy(items = state.items + (itemId -> item.copy(quantity = qty)))
              replyTo ! QuantityUpdated(itemId, qty)
              active(newState)
          }
          
        case GetCart(replyTo) =>
          replyTo ! state
          Behaviors.same
          
        case Checkout(replyTo) =>
          if (state.items.isEmpty) {
            replyTo ! CartError("Cart is empty")
            Behaviors.same
          } else {
            val items = state.items.values.toList
            val total = state.total
            context.log.info(s"Checkout: ${items.size} items, total $$${total}")
            replyTo ! CheckoutStarted(total, items)
            checkedOut(state)  // Change behavior
          }
          
        case Clear =>
          active(CartState(Map.empty))
      }
    }
  
  // Different behavior after checkout
  private def checkedOut(state: CartState): Behavior[Command] = 
    Behaviors.receive { (context, message) =>
      message match {
        case GetCart(replyTo) =>
          replyTo ! state
          Behaviors.same
        case _ =>
          context.log.warn("Cart is checked out, cannot modify")
          Behaviors.same
      }
    }
}

// Note: Full runnable examples require akka dependencies
// These show the patterns — see akka documentation for full setup
```

---

## Step 333: Actor Lifecycle and Supervision

```scala
import akka.actor.typed._
import akka.actor.typed.scaladsl._
import scala.concurrent.duration._

// Actor lifecycle hooks
object LifecycleActor {
  
  sealed trait Command
  case object Start extends Command
  case object Stop extends Command
  case class Process(data: String, replyTo: ActorRef[String]) extends Command
  
  def apply(): Behavior[Command] = Behaviors.setup { context =>
    context.log.info(s"${context.self.path.name} starting up")
    
    // PostStop signal
    Behaviors.receiveMessage[Command] {
      case Start =>
        context.log.info("Started processing")
        Behaviors.same
        
      case Stop =>
        context.log.info("Stopping...")
        Behaviors.stopped  // Signal to stop
        
      case Process(data, replyTo) =>
        context.log.info(s"Processing: $data")
        replyTo ! s"Processed: ${data.toUpperCase}"
        Behaviors.same
    }.receiveSignal {
      case (context, PostStop) =>
        context.log.info(s"${context.self.path.name} stopped cleanly")
        Behaviors.same
      case (context, PreRestart) =>
        context.log.info(s"${context.self.path.name} restarting")
        Behaviors.same
    }
  }
}

// Supervision — parent decides what happens when child fails
object SupervisionDemo {
  
  sealed trait WorkerCommand
  case class DoWork(data: String, replyTo: ActorRef[String]) extends WorkerCommand
  
  // Worker that can fail
  object FallibleWorker {
    def apply(): Behavior[WorkerCommand] = Behaviors.receive { (context, message) =>
      message match {
        case DoWork(data, replyTo) if data == "fail" =>
          throw new RuntimeException(s"Worker failed on: $data")
        case DoWork(data, replyTo) =>
          replyTo ! s"Processed: $data"
          Behaviors.same
      }
    }
  }
  
  // Supervisor strategies
  object SupervisorActor {
    sealed trait Command
    case class Submit(data: String, replyTo: ActorRef[String]) extends Command
    
    def apply(): Behavior[Command] = Behaviors.setup { context =>
      
      // Restart on RuntimeException, stop on others
      val worker = context.spawn(
        Behaviors.supervise(FallibleWorker())
          .onFailure[RuntimeException](SupervisorStrategy.restart.withLimit(3, 10.seconds)),
        "worker"
      )
      
      Behaviors.receiveMessage {
        case Submit(data, replyTo) =>
          worker ! DoWork(data, replyTo)
          Behaviors.same
      }
    }
  }
}

// Timers in actors
object TimerActor {
  
  sealed trait Command
  case object Tick extends Command
  case object Start extends Command
  case object Stop extends Command
  case class Schedule(interval: FiniteDuration) extends Command
  
  def apply(): Behavior[Command] = Behaviors.withTimers { timers =>
    Behaviors.receiveMessage {
      case Start =>
        timers.startTimerWithFixedDelay("tick-timer", Tick, 1.second)
        active(0, timers)
      case _ =>
        Behaviors.same
    }
  }
  
  private def active(count: Int, timers: TimerScheduler[Command]): Behavior[Command] =
    Behaviors.receiveMessage {
      case Tick =>
        val newCount = count + 1
        println(s"  [Timer] Tick $newCount")
        if (newCount >= 5) {
          timers.cancelAll()
          println("  [Timer] Stopped after 5 ticks")
          Behaviors.stopped
        } else {
          active(newCount, timers)
        }
      case Stop =>
        timers.cancelAll()
        Behaviors.stopped
      case _ =>
        Behaviors.same
    }
}

// Child actor creation pattern
object ParentChildDemo {
  
  object Parent {
    sealed trait Command
    case class CreateChild(name: String) extends Command
    case class BroadcastMessage(msg: String) extends Command
    
    def apply(): Behavior[Command] = managing(Map.empty)
    
    private def managing(children: Map[String, ActorRef[Child.Command]]): Behavior[Command] =
      Behaviors.receive { (context, message) =>
        message match {
          case CreateChild(name) =>
            val child = context.spawn(Child(name), name)
            context.watchWith(child, ChildTerminated(name))  // Watch for termination
            managing(children + (name -> child))
            
          case BroadcastMessage(msg) =>
            children.values.foreach(_ ! Child.Receive(msg))
            Behaviors.same
        }
      }
    
    case class ChildTerminated(name: String) extends Command
  }
  
  object Child {
    sealed trait Command
    case class Receive(msg: String) extends Command
    
    def apply(name: String): Behavior[Command] = Behaviors.receiveMessage {
      case Receive(msg) =>
        println(s"  Child '$name' received: $msg")
        Behaviors.same
    }
  }
}
```

---

## Step 334-340: Actor Patterns

```scala
import akka.actor.typed._
import akka.actor.typed.scaladsl._
import scala.concurrent.duration._

// Request-Response pattern (Ask pattern)
object AskPatternDemo {
  
  object DataService {
    sealed trait Command
    case class GetData(key: String, replyTo: ActorRef[Response]) extends Command
    
    sealed trait Response
    case class DataFound(key: String, value: String) extends Response
    case class DataNotFound(key: String) extends Response
    
    val data = Map("k1" -> "value1", "k2" -> "value2", "k3" -> "value3")
    
    def apply(): Behavior[Command] = Behaviors.receiveMessage {
      case GetData(key, replyTo) =>
        data.get(key) match {
          case Some(v) => replyTo ! DataFound(key, v)
          case None    => replyTo ! DataNotFound(key)
        }
        Behaviors.same
    }
  }
  
  object Aggregator {
    sealed trait Command
    case class FetchAll(keys: List[String], replyTo: ActorRef[Map[String, String]]) extends Command
    
    def apply(): Behavior[Command] = Behaviors.setup { context =>
      val service = context.spawn(DataService(), "data-service")
      
      Behaviors.receiveMessage {
        case FetchAll(keys, replyTo) =>
          implicit val timeout: akka.util.Timeout = 3.seconds
          implicit val ec = context.executionContext
          import akka.actor.typed.scaladsl.AskPattern._
          implicit val scheduler = context.system.scheduler
          
          val futures = keys.map { key =>
            service.ask[DataService.Response](ref => DataService.GetData(key, ref)).map {
              case DataService.DataFound(k, v) => k -> v
              case DataService.DataNotFound(k) => k -> ""
            }
          }
          
          import scala.concurrent.Future
          Future.sequence(futures).foreach { results =>
            replyTo ! results.filter(_._2.nonEmpty).toMap
          }
          
          Behaviors.same
      }
    }
  }
}

// Router pattern — distribute work across actor pool
object RouterPattern {
  
  sealed trait WorkerMessage
  case class Work(id: Int, data: String, replyTo: ActorRef[String]) extends WorkerMessage
  
  object Worker {
    def apply(workerId: Int): Behavior[WorkerMessage] = Behaviors.receiveMessage {
      case Work(id, data, replyTo) =>
        println(s"  Worker $workerId processing job $id: $data")
        Thread.sleep(100)  // Simulate work
        replyTo ! s"Job $id done by worker $workerId"
        Behaviors.same
    }
  }
  
  object WorkerPool {
    sealed trait Command
    case class Submit(id: Int, data: String, replyTo: ActorRef[String]) extends Command
    
    def apply(poolSize: Int): Behavior[Command] = Behaviors.setup { context =>
      val workers = (1 to poolSize).map { i =>
        context.spawn(Worker(i), s"worker-$i")
      }.toVector
      
      var nextWorker = 0
      
      Behaviors.receiveMessage {
        case Submit(id, data, replyTo) =>
          val worker = workers(nextWorker % workers.size)
          worker ! Work(id, data, replyTo)
          nextWorker += 1
          Behaviors.same
      }
    }
  }
}

// Actor as a cache
object CachingActor {
  
  sealed trait Command[A]
  case class Get[A](key: String, loader: String => scala.concurrent.Future[A], 
                    replyTo: ActorRef[Either[Throwable, A]]) extends Command[A]
  case class Invalidate(key: String) extends Command[Nothing]
  case class InvalidateAll() extends Command[Nothing]
  
  // Type-erased version for mixed cache
  sealed trait CacheCommand
  case class CacheGet(key: String, loader: String => scala.concurrent.Future[Any],
                      replyTo: ActorRef[Either[Throwable, Any]]) extends CacheCommand
  case class CacheInvalidate(key: String) extends CacheCommand
  case class CacheInvalidateAll() extends CacheCommand
  
  def apply(): Behavior[CacheCommand] = withCache(Map.empty)
  
  private def withCache(cache: Map[String, Any]): Behavior[CacheCommand] =
    Behaviors.receive { (context, message) =>
      message match {
        case CacheGet(key, loader, replyTo) =>
          cache.get(key) match {
            case Some(value) =>
              context.log.debug(s"Cache HIT: $key")
              replyTo ! Right(value)
              Behaviors.same
              
            case None =>
              context.log.debug(s"Cache MISS: $key, loading...")
              import scala.concurrent.ExecutionContext.Implicits.global
              loader(key).onComplete {
                case scala.util.Success(value) => 
                  replyTo ! Right(value)
                  // Note: would need to send message back to self to update cache
                case scala.util.Failure(e) =>
                  replyTo ! Left(e)
              }
              Behaviors.same  // Real implementation would update cache
              
          }
        case CacheInvalidate(key) =>
          withCache(cache - key)
        case CacheInvalidateAll() =>
          withCache(Map.empty)
      }
    }
}
```

---

## สรุป Part 34

| Concept | Actor Typed API | ความหมาย |
|---------|----------------|---------|
| Behavior | `Behavior[T]` | How actor handles messages |
| Behaviors.receive | Process messages with context | Access to context + message |
| Behaviors.receiveMessage | Process messages only | Simpler, no context |
| Behaviors.setup | Initialize actor | Run once at startup |
| Behaviors.withTimers | Timer-based behavior | Periodic tasks |
| Behaviors.same | Return same behavior | No state change |
| Behaviors.stopped | Signal stop | Actor terminates |
| ActorRef[T] | Reference to actor | Send messages with `!` |
| Ask pattern | `actor.ask(ref => Msg(ref))` | Request-response |
| Supervision | `Behaviors.supervise(b)` | Fault tolerance |

---

## แบบฝึกหัด Part 34

**ข้อ 1:** Implement `OrderProcessorActor` ที่ orchestrate validation, payment, fulfillment ผ่าน child actors

**ข้อ 2:** สร้าง `GameSessionActor` ที่ manage state ของ multiplayer game session

**ข้อ 3:** Implement `BroadcastActor` ที่ forward messages ไปยัง registered subscribers

**ข้อ 4:** สร้าง `SchedulerActor` ที่ run jobs ตาม schedule ด้วย timers

**ข้อ 5:** Implement `RateLimiterActor` ที่ queue requests และ release ตาม configured rate

---

➡️ ต่อไป: [Part 35 — Akka Actor Communication](part-35-akka-actor-communication.md)
