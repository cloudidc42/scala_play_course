# Part 66: Redis with Scala

## Steps 651-660: Redis Setup, Caching Patterns, Pub/Sub, Sorted Sets, Sessions

---

## Step 651: Redis คืออะไร

Redis คือ in-memory data structure store ที่ใช้เป็น cache, message broker, และ session store

```
Redis use cases:
├── Caching (Cache-Aside, Read-Through)
├── Session storage
├── Rate limiting
├── Pub/Sub messaging
├── Sorted sets (leaderboards, rankings)
├── Distributed locks
├── Job queues
└── Real-time analytics
```

### build.sbt

```scala
libraryDependencies ++= Seq(
  // Lettuce (async/reactive client — recommended)
  "io.lettuce"    % "lettuce-core"  % "6.3.2.RELEASE",

  // หรือ Jedis (synchronous)
  "redis.clients" % "jedis"         % "5.1.2",

  // สำหรับ Play Cache
  "com.github.karelcemus" %% "play-redis" % "3.0.0"
)
```

---

## Step 652: Redis Connection Setup

```scala
// app/db/RedisClient.scala
package db

import io.lettuce.core.*
import io.lettuce.core.api.*
import io.lettuce.core.api.async.*
import io.lettuce.core.api.reactive.*
import javax.inject.*
import play.api.Configuration
import play.api.inject.ApplicationLifecycle
import scala.concurrent.Future

@Singleton
class RedisClient @Inject()(
  config: Configuration,
  lifecycle: ApplicationLifecycle
) {
  private val redisUrl  = config.getOrElse("redis.url", "redis://localhost:6379")
  private val client    = RedisClient.create(redisUrl)
  private val connection = client.connect()

  // Async API
  val async: RedisAsyncCommands[String, String] = connection.async()

  // Reactive API (for streaming)
  val reactive: RedisReactiveCommands[String, String] = connection.reactive()

  // Sync API (for simple operations)
  val sync: RedisCommands[String, String] = connection.sync()

  // Cleanup
  lifecycle.addStopHook(() => Future.successful {
    connection.close()
    client.shutdown()
  })
}
```

---

## Step 653: Basic Cache Operations

```scala
// app/services/RedisCacheService.scala
package services

import db.RedisClient
import io.lettuce.core.SetArgs
import javax.inject.*
import play.api.libs.json.*
import scala.concurrent.*
import scala.concurrent.duration.*
import java.util.concurrent.TimeUnit

@Singleton
class RedisCacheService @Inject()(redisClient: RedisClient)(implicit ec: ExecutionContext) {

  private val redis = redisClient.async

  // SET with TTL
  def set(key: String, value: String, ttl: Duration): Future[Unit] =
    redis.setex(key, ttl.toSeconds, value).toCompletableFuture.toScala.map(_ => ())

  // SET JSON
  def setJson[T: Writes](key: String, value: T, ttl: Duration): Future[Unit] =
    set(key, Json.stringify(Json.toJson(value)), ttl)

  // GET
  def get(key: String): Future[Option[String]] =
    redis.get(key).toCompletableFuture.toScala.map(Option(_))

  // GET JSON
  def getJson[T: Reads](key: String): Future[Option[T]] =
    get(key).map(_.flatMap(s =>
      Json.parse(s).asOpt[T]
    ))

  // GET or SET (Cache-Aside)
  def getOrSet[T: Reads: Writes](
    key: String,
    ttl: Duration
  )(compute: => Future[T]): Future[T] = {
    getJson[T](key).flatMap {
      case Some(cached) => Future.successful(cached)
      case None =>
        compute.flatMap { value =>
          setJson(key, value, ttl).map(_ => value)
        }
    }
  }

  // DELETE
  def delete(key: String): Future[Long] =
    redis.del(key).toCompletableFuture.toScala.map(_.toLong)

  // DELETE pattern (use carefully)
  def deletePattern(pattern: String): Future[Long] =
    redis.keys(pattern).toCompletableFuture.toScala.flatMap { keys =>
      if (keys.isEmpty) Future.successful(0L)
      else redis.del(keys.toSeq: _*).toCompletableFuture.toScala.map(_.toLong)
    }

  // EXISTS
  def exists(key: String): Future[Boolean] =
    redis.exists(key).toCompletableFuture.toScala.map(_ > 0)

  // TTL
  def ttl(key: String): Future[Long] =
    redis.ttl(key).toCompletableFuture.toScala.map(_.toLong)

  // INCR (counter)
  def incr(key: String): Future[Long] =
    redis.incr(key).toCompletableFuture.toScala.map(_.toLong)

  def incrBy(key: String, amount: Long): Future[Long] =
    redis.incrby(key, amount).toCompletableFuture.toScala.map(_.toLong)

  // Scala extension
  implicit class CompletionStageOps[T](cs: java.util.concurrent.CompletionStage[T]) {
    def toScala: Future[T] = {
      val promise = Promise[T]()
      cs.whenComplete { (value, ex) =>
        if (ex != null) promise.failure(ex)
        else promise.success(value)
      }
      promise.future
    }
  }
}
```

---

## Step 654: Cache Patterns

```scala
// app/services/ArticleCacheService.scala
package services

import javax.inject.*
import models.*
import scala.concurrent.*
import scala.concurrent.duration.*

@Singleton
class ArticleCacheService @Inject()(
  cache: RedisCacheService,
  articleRepo: ArticleRepository
)(implicit ec: ExecutionContext) {

  private val ArticleTtl    = 30.minutes
  private val ListTtl       = 5.minutes
  private val TagsTtl       = 1.hour

  // Cache key patterns
  private def articleKey(id: String)    = s"article:$id"
  private def articleSlugKey(slug: String) = s"article:slug:$slug"
  private def listKey(page: Int)        = s"articles:list:$page"
  private def tagKey(tag: String)       = s"articles:tag:$tag"
  private def userArticlesKey(userId: String) = s"user:$userId:articles"

  // Cache-Aside: find article
  def findById(id: String): Future[Option[Article]] =
    cache.getOrSet[Option[Article]](articleKey(id), ArticleTtl) {
      articleRepo.findById(id)
    }

  // Invalidate cache เมื่อ article update
  def invalidateArticle(id: String, slug: Option[String] = None): Future[Unit] = {
    val keys = Seq(articleKey(id)) ++
               slug.map(articleSlugKey).toSeq ++
               (1 to 10).map(p => listKey(p))  // invalidate first 10 pages

    Future.sequence(keys.map(cache.delete)).map(_ => ())
  }

  // Write-Through: update ทั้ง DB และ cache พร้อมกัน
  def updateArticle(article: Article): Future[Article] = {
    for {
      updated <- articleRepo.update(article)
      _       <- cache.setJson(articleKey(article.id), updated, ArticleTtl)
      // Invalidate list caches
      _       <- cache.deletePattern("articles:list:*")
    } yield updated
  }

  // Cache list with pagination
  def findPublished(page: Int = 1, pageSize: Int = 20): Future[Seq[Article]] = {
    val offset = (page - 1) * pageSize
    cache.getOrSet[Seq[Article]](listKey(page), ListTtl) {
      articleRepo.findPublished(pageSize, offset)
    }
  }
}
```

---

## Step 655: Rate Limiting with Redis

```scala
// app/services/RateLimitService.scala
package services

import db.RedisClient
import javax.inject.*
import scala.concurrent.*
import scala.concurrent.duration.*

@Singleton
class RateLimitService @Inject()(
  redisClient: RedisClient
)(implicit ec: ExecutionContext) {

  private val redis = redisClient.async

  // Fixed Window Rate Limiting
  // ตรวจว่า key เกิน limit ใน time window หรือไม่
  def isRateLimited(
    key: String,
    limit: Int,
    window: Duration
  ): Future[Boolean] = {
    val redisKey = s"ratelimit:$key:${System.currentTimeMillis() / window.toMillis}"

    import scala.jdk.FutureConverters.*

    for {
      count <- redis.incr(redisKey).toCompletableFuture.asScala
      _     <- if (count == 1)
                 redis.expire(redisKey, window.toSeconds).toCompletableFuture.asScala
               else
                 Future.successful(false)
    } yield count > limit
  }

  // Sliding Window Rate Limiting (more accurate)
  def checkSlidingWindow(
    key: String,
    limit: Int,
    windowMs: Long
  ): Future[Boolean] = {
    val now     = System.currentTimeMillis()
    val window  = now - windowMs
    val redisKey = s"ratelimit:sliding:$key"

    import scala.jdk.FutureConverters.*

    for {
      // Remove old entries
      _     <- redis.zremrangebyscore(
                 redisKey,
                 io.lettuce.core.Range.create(Double.NegativeInfinity, window.toDouble)
               ).toCompletableFuture.asScala

      // Count current window
      count <- redis.zcard(redisKey).toCompletableFuture.asScala

      // Add current request
      _     <- redis.zadd(redisKey, now.toDouble, s"$now-${scala.util.Random.nextInt()}")
                 .toCompletableFuture.asScala

      // Set expiry
      _     <- redis.expire(redisKey, (windowMs / 1000) + 1)
                 .toCompletableFuture.asScala
    } yield count >= limit
  }

  // Token Bucket
  def consumeToken(
    key: String,
    capacity: Int,
    refillRate: Int,  // tokens per second
  ): Future[Boolean] = {
    // Implement token bucket algorithm with Lua script
    val script = """
      local key = KEYS[1]
      local capacity = tonumber(ARGV[1])
      local refill_rate = tonumber(ARGV[2])
      local now = tonumber(ARGV[3])

      local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
      local tokens = tonumber(bucket[1]) or capacity
      local last_refill = tonumber(bucket[2]) or now

      -- Refill tokens
      local elapsed = (now - last_refill) / 1000
      tokens = math.min(capacity, tokens + elapsed * refill_rate)

      if tokens >= 1 then
        tokens = tokens - 1
        redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
        redis.call('EXPIRE', key, 3600)
        return 1
      else
        return 0
      end
    """

    import scala.jdk.FutureConverters.*

    redis.eval[java.lang.Long](
      script,
      io.lettuce.core.ScriptOutputType.INTEGER,
      Array(key),
      capacity.toString, refillRate.toString, System.currentTimeMillis().toString
    ).toCompletableFuture.asScala.map(_ == 1L)
  }
}
```

---

## Step 656: Pub/Sub

```scala
// app/services/RedisPubSubService.scala
package services

import io.lettuce.core.pubsub.*
import io.lettuce.core.pubsub.api.async.*
import db.RedisClient
import javax.inject.*
import org.apache.pekko.actor.ActorSystem
import play.api.libs.json.*
import scala.concurrent.*

@Singleton
class RedisPubSubService @Inject()(
  redisClient: RedisClient,
  system: ActorSystem
)(implicit ec: ExecutionContext) {

  // Publisher
  def publish(channel: String, message: String): Future[Long] = {
    import scala.jdk.FutureConverters.*
    redisClient.async.publish(channel, message).toCompletableFuture.asScala.map(_.toLong)
  }

  def publishJson[T: Writes](channel: String, message: T): Future[Long] =
    publish(channel, Json.stringify(Json.toJson(message)))

  // Subscriber (รับข้อความผ่าน Pekko Source)
  def subscribe(channel: String): org.apache.pekko.stream.scaladsl.Source[String, ?] = {
    import org.apache.pekko.stream.scaladsl.*
    import org.apache.pekko.NotUsed

    val (queue, source) = Source.queue[String](100, org.apache.pekko.stream.OverflowStrategy.dropHead)
      .preMaterialize()(org.apache.pekko.stream.Materializer(system))

    val listener = new RedisPubSubListener[String, String] {
      override def message(channel: String, message: String): Unit =
        queue.offer(message)

      override def message(pattern: String, channel: String, message: String): Unit =
        queue.offer(message)

      override def subscribed(channel: String, count: Long): Unit = {}
      override def unsubscribed(channel: String, count: Long): Unit = {}
      override def psubscribed(pattern: String, count: Long): Unit = {}
      override def punsubscribed(pattern: String, count: Long): Unit = {}
    }

    // Create pubsub connection
    val pubSubConn = redisClient.async.getStatefulConnection.asInstanceOf[StatefulRedisPubSubConnection[String, String]]
    pubSubConn.addListener(listener)
    pubSubConn.async().subscribe(channel)

    source
  }
}
```

---

## Step 657: Sorted Sets (Leaderboard)

```scala
// app/services/LeaderboardService.scala
package services

import db.RedisClient
import javax.inject.*
import scala.concurrent.*
import scala.jdk.FutureConverters.*

@Singleton
class LeaderboardService @Inject()(redisClient: RedisClient)(implicit ec: ExecutionContext) {

  private val redis = redisClient.async

  // Key patterns
  private def globalKey()            = "leaderboard:global"
  private def dailyKey(date: String) = s"leaderboard:daily:$date"
  private def weeklyKey(week: String) = s"leaderboard:weekly:$week"

  // Add/Update score
  def addScore(userId: String, points: Double): Future[Unit] = {
    val date = java.time.LocalDate.now().toString
    val week = java.time.LocalDate.now().format(
      java.time.format.DateTimeFormatter.ofPattern("yyyy-'W'ww")
    )

    Future.sequence(Seq(
      redis.zincrby(globalKey(), points, userId).toCompletableFuture.asScala,
      redis.zincrby(dailyKey(date), points, userId).toCompletableFuture.asScala,
      redis.zincrby(weeklyKey(week), points, userId).toCompletableFuture.asScala
    )).map(_ => ())
  }

  // Top N players
  def topPlayers(n: Int, period: String = "global"): Future[Seq[(String, Double)]] = {
    val key = period match {
      case "daily"  => dailyKey(java.time.LocalDate.now().toString)
      case "weekly" => weeklyKey("this-week")
      case _        => globalKey()
    }

    redis.zrevrangeWithScores(key, 0, n - 1).toCompletableFuture.asScala
      .map(_.asScala.map(sv => (sv.getValue, sv.getScore)).toSeq)
  }

  // Player rank
  def getRank(userId: String): Future[Option[Long]] =
    redis.zrevrank(globalKey(), userId).toCompletableFuture.asScala
      .map(rank => Option(rank).map(_.toLong + 1))

  // Player score
  def getScore(userId: String): Future[Option[Double]] =
    redis.zscore(globalKey(), userId).toCompletableFuture.asScala
      .map(score => Option(score).map(_.toDouble))

  // Players around a user (context rank)
  def getSurroundingPlayers(userId: String, range: Int = 5): Future[Seq[(String, Double)]] = {
    redis.zrevrank(globalKey(), userId).toCompletableFuture.asScala.flatMap {
      case null => Future.successful(Seq.empty)
      case rank =>
        val start = Math.max(0, rank - range)
        val stop  = rank + range
        redis.zrevrangeWithScores(globalKey(), start, stop).toCompletableFuture.asScala
          .map(_.asScala.map(sv => (sv.getValue, sv.getScore)).toSeq)
    }
  }

  // Scala collection extension
  implicit class JavaListOps[T](list: java.util.List[T]) {
    def asScala: Seq[T] = scala.jdk.CollectionConverters.ListHasAsScala(list).asScala.toSeq
  }
}
```

---

## Step 658: Session Store

```scala
// app/services/RedisSessionService.scala
package services

import db.RedisClient
import javax.inject.*
import play.api.libs.json.*
import play.api.mvc.{Cookie, RequestHeader, Result}
import scala.concurrent.*
import scala.concurrent.duration.*
import java.util.UUID

@Singleton
class RedisSessionService @Inject()(
  redisClient: RedisClient
)(implicit ec: ExecutionContext) {

  private val SessionTtl    = 24.hours
  private val SessionCookie = "SESSION_ID"

  case class SessionData(
    userId: String,
    email: String,
    role: String,
    createdAt: Long = System.currentTimeMillis()
  )

  implicit val sessionFormat: OFormat[SessionData] = Json.format[SessionData]

  private def sessionKey(id: String) = s"session:$id"

  // Create session
  def create(userId: String, email: String, role: String): Future[String] = {
    val sessionId = UUID.randomUUID().toString
    val data = SessionData(userId, email, role)
    val key  = sessionKey(sessionId)

    import scala.jdk.FutureConverters.*

    redisClient.async
      .setex(key, SessionTtl.toSeconds, Json.stringify(Json.toJson(data)))
      .toCompletableFuture.asScala
      .map(_ => sessionId)
  }

  // Get session
  def get(sessionId: String): Future[Option[SessionData]] = {
    import scala.jdk.FutureConverters.*

    redisClient.async.get(sessionKey(sessionId)).toCompletableFuture.asScala
      .map(value => Option(value).flatMap(s => Json.parse(s).asOpt[SessionData]))
  }

  // Refresh TTL
  def refresh(sessionId: String): Future[Boolean] = {
    import scala.jdk.FutureConverters.*
    redisClient.async.expire(sessionKey(sessionId), SessionTtl.toSeconds)
      .toCompletableFuture.asScala.map(b => b)
  }

  // Destroy session
  def destroy(sessionId: String): Future[Long] = {
    import scala.jdk.FutureConverters.*
    redisClient.async.del(sessionKey(sessionId)).toCompletableFuture.asScala.map(_.toLong)
  }

  // Cookie helpers
  def addSessionCookie(result: Result, sessionId: String): Result =
    result.withCookies(Cookie(
      name     = SessionCookie,
      value    = sessionId,
      maxAge   = Some(SessionTtl.toSeconds.toInt),
      httpOnly = true,
      secure   = true,
      sameSite = Some(Cookie.SameSite.Lax)
    ))

  def getSessionId(request: RequestHeader): Option[String] =
    request.cookies.get(SessionCookie).map(_.value)
}
```

---

## Step 659: Distributed Lock

```scala
// app/services/RedisLockService.scala
package services

import db.RedisClient
import io.lettuce.core.SetArgs
import javax.inject.*
import scala.concurrent.*
import scala.concurrent.duration.*
import java.util.UUID

@Singleton
class RedisLockService @Inject()(redisClient: RedisClient)(implicit ec: ExecutionContext) {

  private val redis = redisClient.sync  // ใช้ sync สำหรับ atomic operations

  // Acquire lock (SET NX EX)
  def acquire(key: String, ttl: Duration): Option[String] = {
    val lockValue = UUID.randomUUID().toString
    val lockKey   = s"lock:$key"

    val result = redis.set(
      lockKey,
      lockValue,
      SetArgs.Builder.nx().ex(ttl.toSeconds)
    )

    if (result == "OK") Some(lockValue) else None
  }

  // Release lock (Lua script for atomic check-and-delete)
  def release(key: String, lockValue: String): Boolean = {
    val script = """
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
      else
        return 0
      end
    """

    val lockKey = s"lock:$key"
    val result = redis.eval[java.lang.Long](
      script,
      io.lettuce.core.ScriptOutputType.INTEGER,
      Array(lockKey),
      lockValue
    )

    result == 1L
  }

  // Helper: execute with lock
  def withLock[T](key: String, ttl: Duration)(
    onAcquired: => Future[T],
    onFailed: => Future[T] = Future.failed(new Exception(s"Could not acquire lock: $key"))
  ): Future[T] = {
    acquire(key, ttl) match {
      case None => onFailed
      case Some(lockValue) =>
        onAcquired.andThen { case _ => release(key, lockValue) }
    }
  }
}

// ใช้งาน
def processPayment(orderId: String): Future[PaymentResult] =
  lockService.withLock(s"payment:$orderId", 30.seconds) {
    // ทำงานที่ต้องการ exclusive access
    paymentProcessor.process(orderId)
  }
```

---

## Step 660: Testing Redis

```scala
// test/services/RedisServiceSpec.scala (ต้องมี Redis running)
class RedisCacheServiceSpec extends PlaySpec with GuiceOneAppPerTest {

  "RedisCacheService" should {
    "set and get values" in {
      val cache = app.injector.instanceOf[RedisCacheService]
      import scala.concurrent.duration.*

      val result = for {
        _   <- cache.set("test:key", "test-value", 60.seconds)
        got <- cache.get("test:key")
        _   <- cache.delete("test:key")
      } yield got

      val got = Await.result(result, 5.seconds)
      got mustBe Some("test-value")
    }

    "expire keys" in {
      val cache = app.injector.instanceOf[RedisCacheService]
      import scala.concurrent.duration.*

      val result = for {
        _ <- cache.set("test:expire", "value", 1.second)
        _ <- Future { Thread.sleep(1100) }
        v <- cache.get("test:expire")
      } yield v

      Await.result(result, 5.seconds) mustBe None
    }
  }
}
```

---

## สรุป Part 66

| Use Case | Redis Data Structure | Operation |
|----------|--------------------|-----------| 
| Caching | String | SET/GET/SETEX |
| Counter | String | INCR/INCRBY |
| Session | Hash/String | HMSET/HGETALL |
| Rate limit | String + INCR | INCR + EXPIRE |
| Pub/Sub | Channel | PUBLISH/SUBSCRIBE |
| Leaderboard | Sorted Set | ZADD/ZREVRANGE |
| Queue | List | LPUSH/RPOP |
| Distributed lock | String NX | SET NX EX |
| Tags/Sets | Set | SADD/SMEMBERS |
| Sliding window | Sorted Set | ZADD/ZRANGEBYSCORE |

---

## แบบฝึกหัด Part 66

1. **Multi-level Cache**: Implement L1 (Caffeine in-memory) + L2 (Redis) cache hierarchy ที่ check L1 ก่อน แล้ว L2 แล้วค่อย load จาก DB

2. **Real-time Leaderboard**: สร้าง gaming leaderboard ที่ update real-time ด้วย Redis Sorted Sets และ push rankings ไปยัง frontend ผ่าน SSE

3. **Distributed Job Queue**: Implement job queue ด้วย Redis Lists ที่ support priority, retry logic, และ dead letter queue

4. **Session Security**: สร้าง secure session management ด้วย Redis ที่ support multiple devices, session revocation, และ suspicious activity detection

5. **Rate Limiter Middleware**: Implement sliding window rate limiter เป็น Play Filter ที่ rate limit ทั้ง IP-based และ API key-based

---

[→ ไปยัง Part 67: Elasticsearch](part-67-elasticsearch.md)
