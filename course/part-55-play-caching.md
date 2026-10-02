# Part 55: Play Caching

## Steps 541-550: Play Cache API, EhCache, Redis, HTTP Caching Headers, Cache Strategies

---

## Step 541: Play Cache API

```scala
// build.sbt
libraryDependencies ++= Seq(
  ehcache,  // EhCache - in-memory cache
  // หรือ
  "com.github.karelcemus" %% "play-redis" % "3.0.0",  // Redis cache
  guice
)
```

### Cache Abstraction

```
Play Cache API
├── AsyncCacheApi - non-blocking cache (recommended)
│   ├── get(key): Future[Option[T]]
│   ├── set(key, value, expiration): Future[Done]
│   ├── remove(key): Future[Done]
│   └── removeAll(): Future[Done]
└── SyncCacheApi - blocking cache
    ├── get(key): Option[T]
    ├── set(key, value, expiration)
    └── remove(key)
```

---

## Step 542: AsyncCacheApi

```scala
// app/services/CachedArticleService.scala
package services

import javax.inject.*
import play.api.cache.AsyncCacheApi
import scala.concurrent.*
import scala.concurrent.duration.*
import models.Article

@Singleton
class CachedArticleService @Inject()(
  articleRepo: repositories.ArticleRepository,
  cache: AsyncCacheApi,
  implicit val ec: ExecutionContext
) {

  // Cache key patterns
  private def articleKey(id: Long): String = s"article:$id"
  private def articleListKey(page: Int): String = s"articles:list:page:$page"
  private def articleCountKey: String = "articles:count"

  // get-or-set pattern (cache-aside)
  def findById(id: Long): Future[Option[Article]] = {
    cache.getOrElseUpdate[Option[Article]](articleKey(id), 10.minutes) {
      articleRepo.findById(id)
    }
  }

  // Manual get/set
  def findByIdManual(id: Long): Future[Option[Article]] = {
    cache.get[Option[Article]](articleKey(id)).flatMap {
      case Some(cached) =>
        Future.successful(cached)  // cache hit
      case None =>
        articleRepo.findById(id).flatMap { article =>
          cache.set(articleKey(id), article, 10.minutes)
               .map(_ => article)  // cache miss - save and return
        }
    }
  }

  // Cache list ด้วย shorter TTL
  def list(page: Int): Future[List[Article]] = {
    cache.getOrElseUpdate[List[Article]](articleListKey(page), 5.minutes) {
      articleRepo.findAll(page, 20, None, None, None).map(_._1)
    }
  }

  // Invalidate cache เมื่อ update
  def update(id: Long, title: String, content: String): Future[Article] = {
    articleRepo.update(id, title, content, List.empty, false, "").flatMap { article =>
      // invalidate article cache
      cache.remove(articleKey(id)).flatMap { _ =>
        // invalidate all list caches
        cache.removeAll().map(_ => article)
      }
    }
  }

  // Cache count แยกต่างหาก
  def getCount(): Future[Long] = {
    cache.getOrElseUpdate[Long](articleCountKey, 1.minute) {
      articleRepo.count()
    }
  }
}
```

---

## Step 543: EhCache Configuration

```hocon
# conf/application.conf - EhCache setup
play.cache.bindCaches = ["article-cache", "user-cache", "session-cache"]

# EhCache configuration
ehcache.jcache.provider = "org.ehcache.jsr107.EhcacheCachingProvider"
```

```xml
<!-- conf/ehcache.xml - EhCache configuration -->
<?xml version="1.0" encoding="UTF-8"?>
<ehcache xmlns="http://www.ehcache.org/v3">

  <!-- Default cache -->
  <cache alias="play">
    <expiry>
      <ttl unit="seconds">3600</ttl>
    </expiry>
    <heap unit="entries">1000</heap>
  </cache>

  <!-- Article cache: large, moderate TTL -->
  <cache alias="article-cache">
    <expiry>
      <ttl unit="minutes">10</ttl>
    </expiry>
    <resources>
      <heap unit="entries">5000</heap>
      <offheap unit="MB">50</offheap>  <!-- อย่าเก็บ Long-lived data ใน heap -->
    </resources>
    <eviction-advisor>com.myapp.ArticleCacheEvictionAdvisor</eviction-advisor>
  </cache>

  <!-- User cache: medium, short TTL -->
  <cache alias="user-cache">
    <expiry>
      <ttl unit="minutes">5</ttl>
    </expiry>
    <heap unit="entries">10000</heap>
  </cache>

  <!-- Session cache: small, very short TTL -->
  <cache alias="session-cache">
    <expiry>
      <ttl unit="minutes">30</ttl>
    </expiry>
    <heap unit="entries">500</heap>
  </cache>

</ehcache>
```

### Named Cache Injection

```scala
// ใช้ specific named cache
@Singleton
class UserController @Inject()(
  @NamedCache("user-cache") userCache: AsyncCacheApi,
  @NamedCache("article-cache") articleCache: AsyncCacheApi,
  val controllerComponents: ControllerComponents
) extends BaseController {

  def getUser(id: Long): Action[AnyContent] = Action.async { _ =>
    userCache.getOrElseUpdate[models.User](s"user:$id", 5.minutes) {
      // fetch from DB
      Future.successful(models.User(id, "username", "email@example.com", models.UserRole.Viewer, java.time.LocalDateTime.now()))
    }.map(user => Ok(play.api.libs.json.Json.toJson(user)))
  }
}
```

---

## Step 544: Redis Cache

```scala
// build.sbt
libraryDependencies += "com.github.karelcemus" %% "play-redis" % "3.0.0"
```

```hocon
# conf/application.conf - Redis configuration
play.cache.redis {
  host = "localhost"
  host = ${?REDIS_HOST}
  port = 6379
  port = ${?REDIS_PORT}
  password = ${?REDIS_PASSWORD}
  database = 0
  timeout = 1s
  maxConnections = 10

  # กำหนดค่า serialization
  serializer = "org.apache.commons.lang3.SerializationUtils"
}

# ใช้ Redis เป็น default cache
play.cache.bindCaches = ["play-redis"]
```

```scala
// app/services/RedisService.scala
package services

import javax.inject.*
import play.api.cache.AsyncCacheApi
import play.api.libs.json.*
import scala.concurrent.*
import scala.concurrent.duration.*

@Singleton
class RedisService @Inject()(
  cache: AsyncCacheApi,
  implicit val ec: ExecutionContext
) {

  // เก็บ JSON data ใน Redis
  def setJson[T: Writes](key: String, value: T, ttl: Duration): Future[Unit] =
    cache.set(key, Json.stringify(Json.toJson(value)), ttl).map(_ => ())

  // อ่าน JSON data จาก Redis
  def getJson[T: Reads](key: String): Future[Option[T]] =
    cache.get[String](key).map { optStr =>
      optStr.flatMap { str =>
        Json.parse(str).asOpt[T]
      }
    }

  // Increment counter (atomic)
  def increment(key: String): Future[Long] = {
    cache.get[Long](key).flatMap {
      case Some(n) =>
        cache.set(key, n + 1).map(_ => n + 1)
      case None =>
        cache.set(key, 1L).map(_ => 1L)
    }
  }

  // Rate limit check
  def isRateLimited(key: String, limit: Int, window: Duration): Future[Boolean] = {
    increment(s"ratelimit:$key").map(_ > limit)
  }

  // Store session data
  def setSession(sessionId: String, data: Map[String, String]): Future[Unit] =
    cache.set(s"session:$sessionId", Json.stringify(Json.toJson(data)), 30.minutes)
         .map(_ => ())

  def getSession(sessionId: String): Future[Option[Map[String, String]]] =
    cache.get[String](s"session:$sessionId").map { optStr =>
      optStr.flatMap(s => Json.parse(s).asOpt[Map[String, String]])
    }
}
```

---

## Step 545: HTTP Caching Headers

```scala
// app/controllers/CachedResponseController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import scala.concurrent.*

@Singleton
class CachedResponseController @Inject()(
  val controllerComponents: ControllerComponents,
  implicit val ec: ExecutionContext
) extends BaseController {

  // Cache-Control headers
  def publicPage(): Action[AnyContent] = Action { implicit request =>
    Ok("Public content that can be cached by CDN")
      .withHeaders(
        "Cache-Control" -> "public, max-age=3600",       // Cache 1 hour
        "Vary"          -> "Accept-Encoding",              // Vary on encoding
        "ETag"          -> "\"abc123\""
      )
  }

  def privateData(): Action[AnyContent] = Action { implicit request =>
    Ok("Private user data")
      .withHeaders(
        "Cache-Control" -> "private, no-cache, must-revalidate",
        "Pragma"        -> "no-cache"
      )
  }

  // ETag-based conditional caching
  def article(id: Long): Action[AnyContent] = Action.async { implicit request =>
    // ดึงข้อมูลและสร้าง ETag จาก content hash
    fetchArticle(id).map {
      case None => NotFound("Article not found")
      case Some(article) =>
        val etag = s""""${computeETag(article)}""""
        val lastModified = article.updatedAt.toString

        // Check conditional request headers
        val ifNoneMatch = request.headers.get("If-None-Match")
        val ifModifiedSince = request.headers.get("If-Modified-Since")

        if (ifNoneMatch.contains(etag) || ifModifiedSince.contains(lastModified)) {
          NotModified.withHeaders(
            "ETag"          -> etag,
            "Last-Modified" -> lastModified
          )
        } else {
          Ok(play.api.libs.json.Json.toJson(article))
            .withHeaders(
              "ETag"          -> etag,
              "Last-Modified" -> lastModified,
              "Cache-Control" -> "public, max-age=300, must-revalidate",
              "Vary"          -> "Accept, Accept-Encoding"
            )
        }
    }
  }

  // Stale-while-revalidate pattern
  def feedWithSwr(): Action[AnyContent] = Action { implicit request =>
    Ok("Fresh content")
      .withHeaders(
        // Cache 10 minutes, แต่ยังเสิร์ฟ stale content ได้อีก 1 ชม. ขณะ revalidate
        "Cache-Control" -> "public, max-age=600, stale-while-revalidate=3600"
      )
  }

  private def fetchArticle(id: Long): Future[Option[Article]] =
    Future.successful(Some(Article(id, "Title", "Content", java.time.LocalDateTime.now())))

  private def computeETag(article: Article): String =
    Integer.toHexString(article.hashCode())

  case class Article(id: Long, title: String, content: String, updatedAt: java.time.LocalDateTime)
}
```

---

## Step 546: Cache Patterns

```scala
// app/services/CachePatterns.scala
package services

import javax.inject.*
import play.api.cache.AsyncCacheApi
import scala.concurrent.*
import scala.concurrent.duration.*

@Singleton
class CachePatterns @Inject()(
  cache: AsyncCacheApi,
  implicit val ec: ExecutionContext
) {

  // Pattern 1: Cache-Aside (Lazy Loading)
  def cacheAside[T](key: String, ttl: Duration)(fetchFromDb: => Future[T]): Future[T] = {
    cache.get[T](key).flatMap {
      case Some(value) => Future.successful(value)  // Cache hit
      case None =>
        fetchFromDb.flatMap { value =>
          cache.set(key, value, ttl).map(_ => value)  // Cache miss: load and cache
        }
    }
  }

  // Pattern 2: Write-Through (update cache เมื่อ write)
  def writeThrough[T](key: String, value: T, ttl: Duration)(writeToDb: T => Future[T]): Future[T] = {
    writeToDb(value).flatMap { saved =>
      cache.set(key, saved, ttl).map(_ => saved)  // Write to both DB and cache
    }
  }

  // Pattern 3: Write-Behind (write to cache, async flush to DB)
  def writeBehind[T](key: String, value: T, ttl: Duration)(flushToDb: T => Future[Unit]): Future[Unit] = {
    cache.set(key, value, ttl).map { _ =>
      // Async flush to DB (fire and forget)
      flushToDb(value).recover { case ex =>
        play.api.Logger("cache").error(s"Write-behind failed: ${ex.getMessage}")
      }
    }
  }

  // Pattern 4: Refresh-Ahead (proactively refresh)
  def refreshAhead[T](
    key: String,
    ttl: Duration,
    refreshThreshold: Duration
  )(fetchFromDb: => Future[T]): Future[T] = {
    cache.get[T](key).flatMap {
      case Some(value) =>
        // ตรวจสอบว่าใกล้หมดอายุหรือยัง ถ้าใกล้ หรือ refresh async
        Future.successful(value)
      case None =>
        fetchFromDb.flatMap { value =>
          cache.set(key, value, ttl).map(_ => value)
        }
    }
  }

  // Pattern 5: Cache Stampede Prevention (ป้องกัน thundering herd)
  private val locks = new java.util.concurrent.ConcurrentHashMap[String, Future[?]]()

  def stampedePrevented[T](key: String, ttl: Duration)(fetchFromDb: => Future[T]): Future[T] = {
    cache.get[T](key).flatMap {
      case Some(value) => Future.successful(value)
      case None =>
        // ตรวจสอบว่ามี request เดียวกันอยู่แล้วหรือไม่
        val existing = locks.get(key).asInstanceOf[Future[T]]
        if (existing != null) {
          existing  // ใช้ future เดิม
        } else {
          val future = fetchFromDb.flatMap { value =>
            cache.set(key, value, ttl).map(_ => value)
          }.andThen { _ =>
            locks.remove(key)
          }
          locks.put(key, future)
          future
        }
    }
  }
}
```

---

## Step 547: Distributed Cache (Redis Cluster)

```hocon
# conf/application.conf - Redis Cluster

play.cache.redis {
  # Single node
  host = "localhost"
  port = 6379

  # หรือ Sentinel mode
  sentinel {
    master = "mymaster"
    nodes = [
      { host = "sentinel1", port = 26379 },
      { host = "sentinel2", port = 26380 },
      { host = "sentinel3", port = 26381 }
    ]
  }

  # หรือ Cluster mode
  cluster {
    nodes = [
      { host = "redis1", port = 6379 },
      { host = "redis2", port = 6379 },
      { host = "redis3", port = 6379 }
    ]
  }
}
```

---

## Step 548: Caffeine Cache (High-performance in-memory)

```scala
// build.sbt
libraryDependencies += "com.github.ben-manes.caffeine" % "caffeine" % "3.1.8"
```

```scala
// app/utils/LocalCache.scala
package utils

import com.github.benmanes.caffeine.cache.{Caffeine, Cache as CaffeineCache}
import java.util.concurrent.TimeUnit
import scala.concurrent.*

// Type-safe, high-performance local cache
class LocalCache[K, V](
  maxSize: Long = 10000,
  ttlSeconds: Long = 300,
  refreshAfterWriteSeconds: Long = 0
) {

  private val caffeineCache: CaffeineCache[K, V] = {
    val builder = Caffeine.newBuilder()
      .maximumSize(maxSize)
      .expireAfterWrite(ttlSeconds, TimeUnit.SECONDS)

    if (refreshAfterWriteSeconds > 0)
      builder.refreshAfterWrite(refreshAfterWriteSeconds, TimeUnit.SECONDS)

    builder.build[K, V]()
  }

  def get(key: K): Option[V] = Option(caffeineCache.getIfPresent(key))

  def set(key: K, value: V): Unit = caffeineCache.put(key, value)

  def remove(key: K): Unit = caffeineCache.invalidate(key)

  def getOrLoad(key: K)(loader: K => V): V =
    caffeineCache.get(key, k => loader(k))

  def invalidateAll(): Unit = caffeineCache.invalidateAll()

  def size: Long = caffeineCache.estimatedSize()
}

// Pre-built caches
object Caches {
  val articleCache = new LocalCache[Long, models.Article](maxSize = 5000, ttlSeconds = 600)
  val userCache = new LocalCache[Long, models.User](maxSize = 10000, ttlSeconds = 300)
  val configCache = new LocalCache[String, String](maxSize = 100, ttlSeconds = 60)
}
```

---

## Step 549: Cache Monitoring

```scala
// app/controllers/CacheMonitorController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.cache.AsyncCacheApi
import play.api.libs.json.*
import scala.concurrent.*

@Singleton
class CacheMonitorController @Inject()(
  val controllerComponents: ControllerComponents,
  cache: AsyncCacheApi,
  implicit val ec: ExecutionContext
) extends BaseController {

  // Cache stats endpoint (admin only)
  def stats(): Action[AnyContent] = Action { implicit request =>
    // EhCache statistics (ถ้าใช้ JMX)
    Ok(Json.obj(
      "cacheType"   -> "EhCache",
      "status"      -> "running",
      "localCaches" -> Json.obj(
        "articleCache" -> Json.obj(
          "size"    -> utils.Caches.articleCache.size,
          "maxSize" -> 5000
        ),
        "userCache" -> Json.obj(
          "size"    -> utils.Caches.userCache.size,
          "maxSize" -> 10000
        )
      )
    ))
  }

  // Clear all caches (admin operation)
  def clearAll(): Action[AnyContent] = Action.async { implicit request =>
    cache.removeAll().map { _ =>
      utils.Caches.articleCache.invalidateAll()
      utils.Caches.userCache.invalidateAll()
      Ok(Json.obj("message" -> "All caches cleared"))
    }
  }
}
```

---

## Step 550: Cache Strategies สำหรับ Production

```
Production Cache Layers:
1. L1 Cache: Caffeine (in-process, microseconds)
   └── สำหรับ hot data ที่ process บ่อยมาก

2. L2 Cache: EhCache (in-memory, single node)
   └── สำหรับ data ที่ query บ่อยแต่ไม่ shared ระหว่าง instances

3. L3 Cache: Redis (distributed, milliseconds)
   └── สำหรับ data ที่ shared ระหว่าง multiple instances

4. L4 Cache: CDN (edge caching)
   └── สำหรับ static assets และ public pages
```

```scala
// app/services/TieredCacheService.scala
package services

import javax.inject.*
import play.api.cache.AsyncCacheApi
import utils.LocalCache
import scala.concurrent.*
import scala.concurrent.duration.*

@Singleton
class TieredCacheService @Inject()(
  redisCache: AsyncCacheApi,  // L3: Redis
  implicit val ec: ExecutionContext
) {

  // L1: Caffeine (5 minutes)
  private val l1 = new LocalCache[String, String](maxSize = 1000, ttlSeconds = 300)

  def get(key: String)(fetchFromDb: => Future[String]): Future[String] = {
    // 1. Check L1 (fastest)
    l1.get(key) match {
      case Some(value) => Future.successful(value)
      case None =>
        // 2. Check L3 (Redis)
        redisCache.get[String](key).flatMap {
          case Some(value) =>
            l1.set(key, value)  // Promote to L1
            Future.successful(value)
          case None =>
            // 3. Fetch from DB
            fetchFromDb.flatMap { value =>
              l1.set(key, value)
              redisCache.set(key, value, 10.minutes).map(_ => value)
            }
        }
    }
  }

  def invalidate(key: String): Future[Unit] = {
    l1.remove(key)
    redisCache.remove(key).map(_ => ())
  }
}
```

---

## สรุป Part 55

| Cache Type | Library | Scope | Speed |
|-----------|---------|-------|-------|
| EhCache | Built-in | Process-local | ~μs |
| Caffeine | `caffeine` | Process-local | ~μs |
| Redis | `play-redis` | Distributed | ~ms |
| HTTP Cache | Browser/CDN | Client-side | varies |

| Cache Pattern | Use Case |
|--------------|---------|
| Cache-Aside | Read-heavy data |
| Write-Through | Consistency important |
| Write-Behind | Write-heavy with eventual consistency |
| Refresh-Ahead | Predictable access patterns |

---

## แบบฝึกหัด Part 55

1. **Cache-Aside Pattern**: Implement cache-aside pattern สำหรับ user profile ที่ expire ใน 5 นาที และ invalidate เมื่อ profile อัพเดต

2. **HTTP Caching**: เพิ่ม ETag และ Last-Modified headers ใน article API และ handle conditional requests (If-None-Match, If-Modified-Since)

3. **Tiered Cache**: Implement 2-tier cache (Caffeine + Redis) สำหรับ product catalog ที่ update ไม่บ่อย

4. **Cache Warming**: สร้าง startup job ที่ pre-warm cache ด้วย most-popular articles เมื่อ application start

5. **Cache Monitoring**: สร้าง dashboard endpoint ที่แสดง cache hit rate, miss rate, และ eviction rate สำหรับ monitoring

---

[→ ไปยัง Part 56: Play File Uploads](part-56-play-file-uploads.md)
