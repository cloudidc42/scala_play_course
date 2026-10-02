# Part 46: Play JSON

## Steps 451-460: play-json, Reads/Writes/Format, OFormat, JSON Validation, Transformations, Error Handling

---

## Step 451: play-json Library

`play-json` คือ library สำหรับ JSON processing ใน Play Framework ออกแบบมาเพื่อความ type-safety และ performance

### build.sbt

```scala
// build.sbt
libraryDependencies ++= Seq(
  "org.playframework" %% "play-json" % "3.0.4",
  // หรือ ถ้าอยู่ใน Play project แล้ว play-json เป็น built-in
  guice
)
```

### JSON Model ใน Scala

```scala
// play.api.libs.json.JsValue hierarchy
JsValue
├── JsString("hello")
├── JsNumber(42)
├── JsBoolean(true)
├── JsNull
├── JsArray(Seq(JsValue, ...))
└── JsObject(Map("key" -> JsValue, ...))
```

---

## Step 452: สร้างและอ่าน JSON

```scala
// app/examples/JsonBasics.scala
package examples

import play.api.libs.json.*

object JsonBasics extends App {

  // สร้าง JSON object
  val json: JsValue = Json.obj(
    "name"    -> "สมชาย",
    "age"     -> 30,
    "email"   -> "somchai@example.com",
    "active"  -> true,
    "score"   -> 95.5,
    "tags"    -> Json.arr("scala", "play", "functional"),
    "address" -> Json.obj(
      "street"  -> "123 ถนนสุขุมวิท",
      "city"    -> "กรุงเทพ",
      "zipCode" -> "10110"
    ),
    "phone"   -> JsNull
  )

  println(Json.prettyPrint(json))
  // Output:
  // {
  //   "name" : "สมชาย",
  //   "age" : 30,
  //   ...
  // }

  // อ่านค่าจาก JSON
  val name = (json \ "name").as[String]         // "สมชาย"
  val age = (json \ "age").as[Int]              // 30
  val city = (json \ "address" \ "city").as[String]  // "กรุงเทพ"
  val firstTag = (json \ "tags" \ 0).as[String] // "scala"

  // Safe reading ด้วย asOpt
  val phone = (json \ "phone").asOpt[String]    // None (เป็น JsNull)
  val missing = (json \ "missing").asOpt[String] // None

  // Validate and get
  val nameResult: JsResult[String] = (json \ "name").validate[String]
  nameResult match {
    case JsSuccess(value, _) => println(s"Name: $value")
    case JsError(errors)     => println(s"Error: $errors")
  }

  // แปลง JSON เป็น String
  val jsonString: String = Json.stringify(json)
  val prettyString: String = Json.prettyPrint(json)

  // Parse JSON String
  val parsed: JsValue = Json.parse("""{"hello": "world"}""")

  // Array operations
  val tagsArray = (json \ "tags").as[JsArray]
  val tagsList = tagsArray.value.map(_.as[String]).toList
  println(s"Tags: $tagsList")  // List(scala, play, functional)
}
```

---

## Step 453: Reads - JSON to Scala

```scala
// app/models/Article.scala
package models

import play.api.libs.json.*
import play.api.libs.functional.syntax.*
import java.time.Instant

case class Article(
  id: Long,
  title: String,
  content: String,
  authorId: Long,
  tags: List[String],
  published: Boolean,
  publishedAt: Option[Instant],
  viewCount: Long
)

object Article {

  // Method 1: ใช้ Json.reads macro (แนะนำ)
  // implicit val reads: Reads[Article] = Json.reads[Article]

  // Method 2: Manual Reads (ยืดหยุ่นกว่า)
  implicit val reads: Reads[Article] = (
    (__ \ "id").read[Long] and
    (__ \ "title").read[String](Reads.minLength(1)) and
    (__ \ "content").read[String] and
    (__ \ "author_id").read[Long] and  // รับ snake_case แต่ map เป็น camelCase
    (__ \ "tags").read[List[String]].orElse(Reads.pure(List.empty)) and
    (__ \ "published").read[Boolean].orElse(Reads.pure(false)) and
    (__ \ "published_at").readNullable[Instant] and
    (__ \ "view_count").read[Long].orElse(Reads.pure(0L))
  )(Article.apply)

  // Method 3: Reads ด้วย custom logic
  val strictReads: Reads[Article] = Reads { json =>
    for {
      id        <- (json \ "id").validate[Long]
      title     <- (json \ "title").validate[String]
                      .filter(JsError("Title too short"))(_.length >= 3)
      content   <- (json \ "content").validate[String]
      authorId  <- (json \ "author_id").validate[Long]
    } yield Article(id, title, content, authorId, List.empty, false, None, 0)
  }
}
```

---

## Step 454: Writes - Scala to JSON

```scala
// app/models/Article.scala (ต่อ)

object Article {
  // ...

  // Method 1: Json.writes macro
  // implicit val writes: Writes[Article] = Json.writes[Article]

  // Method 2: Manual Writes
  implicit val writes: Writes[Article] = (article: Article) => Json.obj(
    "id"           -> article.id,
    "title"        -> article.title,
    "content"      -> article.content,
    "author_id"    -> article.authorId,
    "tags"         -> article.tags,
    "published"    -> article.published,
    "published_at" -> article.publishedAt,
    "view_count"   -> article.viewCount,
    // computed fields
    "word_count"   -> article.content.split("\\s+").length,
    "is_long"      -> (article.content.length > 1000)
  )

  // Method 3: Writes ด้วย OWrites (สำหรับ JsObject)
  val apiWrites: OWrites[Article] = OWrites { article =>
    Json.obj(
      "id"      -> article.id,
      "title"   -> article.title,
      "summary" -> article.content.take(200) + "..."  // truncate content
    )
  }

  // Writes ที่ exclude บาง fields
  val publicWrites: Writes[Article] = Writes { article =>
    Json.obj(
      "id"        -> article.id,
      "title"     -> article.title,
      "published" -> article.published
      // ไม่ include content (อาจมี sensitive data)
    )
  }
}
```

---

## Step 455: Format - ทั้ง Reads และ Writes

```scala
// app/models/User.scala
package models

import play.api.libs.json.*
import java.time.LocalDateTime

case class User(
  id: Long,
  username: String,
  email: String,
  role: UserRole,
  createdAt: LocalDateTime
)

enum UserRole:
  case Admin, Editor, Viewer

object UserRole {
  implicit val format: Format[UserRole] = Format(
    Reads { json =>
      json.validate[String].flatMap {
        case "admin"  => JsSuccess(UserRole.Admin)
        case "editor" => JsSuccess(UserRole.Editor)
        case "viewer" => JsSuccess(UserRole.Viewer)
        case other    => JsError(s"Unknown role: $other")
      }
    },
    Writes(role => JsString(role.toString.toLowerCase))
  )
}

object User {
  // Format = Reads + Writes ในตัวเดียว
  implicit val format: Format[User] = Json.format[User]

  // OFormat = OReads + OWrites (return JsObject)
  implicit val oformat: OFormat[User] = Json.format[User]
}
```

### LocalDateTime Format

```scala
// app/utils/JsonFormats.scala
package utils

import play.api.libs.json.*
import java.time.*
import java.time.format.DateTimeFormatter

object JsonFormats {

  // LocalDateTime format
  implicit val localDateTimeFormat: Format[LocalDateTime] = Format(
    Reads { json =>
      json.validate[String].flatMap { str =>
        try JsSuccess(LocalDateTime.parse(str, DateTimeFormatter.ISO_LOCAL_DATE_TIME))
        catch case _: Exception => JsError(s"Invalid datetime: $str")
      }
    },
    Writes(dt => JsString(dt.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME)))
  )

  // LocalDate format
  implicit val localDateFormat: Format[LocalDate] = Format(
    Reads { json =>
      json.validate[String].flatMap { str =>
        try JsSuccess(LocalDate.parse(str))
        catch case _: Exception => JsError(s"Invalid date: $str")
      }
    },
    Writes(d => JsString(d.toString))
  )

  // Instant format (ISO 8601)
  implicit val instantFormat: Format[Instant] = Format(
    Reads(_.validate[String].flatMap(s =>
      try JsSuccess(Instant.parse(s))
      catch case _: Exception => JsError(s"Invalid instant")
    )),
    Writes(i => JsString(i.toString))
  )

  // BigDecimal format
  implicit val bigDecimalFormat: Format[BigDecimal] = Format(
    Reads(_.validate[Double].map(BigDecimal(_))),
    Writes(bd => JsNumber(bd))
  )
}
```

---

## Step 456: JSON Validation

```scala
// app/controllers/api/UserApiController.scala
package controllers.api

import javax.inject.*
import play.api.mvc.*
import play.api.libs.json.*
import play.api.libs.functional.syntax.*

@Singleton
class UserApiController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  // Reads พร้อม validation rules
  case class CreateUserRequest(
    username: String,
    email: String,
    password: String,
    age: Int
  )

  implicit val createUserReads: Reads[CreateUserRequest] = (
    (__ \ "username").read[String](
      Reads.minLength(3) andKeep Reads.maxLength(50)
    ) and
    (__ \ "email").read[String](Reads.email) and
    (__ \ "password").read[String](Reads.minLength(8)) and
    (__ \ "age").read[Int](Reads.min(18) andKeep Reads.max(120))
  )(CreateUserRequest.apply)

  def create(): Action[JsValue] = Action(parse.json) { implicit request =>
    request.body.validate[CreateUserRequest] match {
      case JsSuccess(createReq, _) =>
        // Process the valid request
        val userId = java.util.UUID.randomUUID().toString
        Created(Json.obj(
          "id"       -> userId,
          "username" -> createReq.username,
          "email"    -> createReq.email,
          "message"  -> "User created successfully"
        ))

      case JsError(errors) =>
        // Format errors for client
        val errorResponse = Json.obj(
          "message" -> "Validation failed",
          "errors"  -> JsError.toJson(errors)
        )
        BadRequest(errorResponse)
    }
  }

  // Validate nested structures
  case class Address(street: String, city: String, zipCode: String)
  case class UpdateProfileRequest(
    name: Option[String],
    bio: Option[String],
    address: Option[Address]
  )

  implicit val addressReads: Reads[Address] = Json.reads[Address]
  implicit val updateProfileReads: Reads[UpdateProfileRequest] = Json.reads[UpdateProfileRequest]

  def updateProfile(id: Long): Action[JsValue] = Action(parse.json) { implicit request =>
    request.body.validate[UpdateProfileRequest].fold(
      errors => BadRequest(Json.obj("errors" -> JsError.toJson(errors))),
      req    => Ok(Json.obj("message" -> "Profile updated", "id" -> id))
    )
  }
}
```

---

## Step 457: JSON Transformations

```scala
// app/utils/JsonTransformations.scala
package utils

import play.api.libs.json.*
import play.api.libs.json.Reads.*

object JsonTransformations {

  // ตัดบาง fields ออก
  val removePassword: Reads[JsObject] =
    (__ \ "password").json.prune

  // เพิ่ม field
  val addTimestamp: Reads[JsObject] =
    __.json.update(
      (__ \ "timestamp").json.put(JsString(java.time.Instant.now().toString))
    )

  // เปลี่ยนชื่อ field
  val renameFields: Reads[JsObject] = (
    (__ \ "user_name").json.copyFrom((__ \ "username").json.pick) and
    (__ \ "username").json.prune
  ).reduce

  // Transform nested object
  val transformNested: Reads[JsObject] =
    (__ \ "address" \ "zip_code").json.copyFrom(
      (__ \ "address" \ "postal_code").json.pick
    )

  // Convert array elements
  def transformArray(elementTransform: Reads[JsObject]): Reads[JsArray] =
    Reads.seq(elementTransform).map(JsArray(_))

  // Combine transformations
  val userTransform: Reads[JsObject] =
    removePassword andThen addTimestamp

  // Example usage
  def transformUser(userJson: JsObject): JsResult[JsObject] =
    userJson.transform(userTransform)
}
```

---

## Step 458: JSON Error Handling

```scala
// app/controllers/api/BaseApiController.scala
package controllers.api

import play.api.mvc.*
import play.api.libs.json.*

trait BaseApiController extends BaseController {

  // Helper: return JSON error
  def jsonError(message: String, status: Int = 400): Result =
    Status(status)(Json.obj(
      "error"   -> true,
      "message" -> message
    ))

  // Helper: handle JSON parse errors
  def withJson[T: Reads](block: T => Result)(implicit request: Request[JsValue]): Result =
    request.body.validate[T].fold(
      errors => BadRequest(Json.obj(
        "error"   -> true,
        "message" -> "Invalid request body",
        "details" -> formatErrors(errors)
      )),
      block
    )

  // Format JsError เป็น readable format
  def formatErrors(errors: collection.Seq[(JsPath, collection.Seq[JsonValidationError])]): JsValue =
    JsObject(
      errors.map { case (path, validationErrors) =>
        path.toString() -> JsArray(validationErrors.map(e =>
          Json.obj(
            "message" -> e.message,
            "args"    -> JsArray(e.args.map(arg => JsString(arg.toString)))
          )
        ))
      }
    )
}
```

```scala
// app/controllers/api/ArticleApiController.scala
package controllers.api

import javax.inject.*
import play.api.mvc.*
import play.api.libs.json.*

@Singleton
class ArticleApiController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseApiController {

  case class CreateArticleRequest(
    title: String,
    content: String,
    tags: List[String]
  )

  implicit val reads: Reads[CreateArticleRequest] = Json.reads[CreateArticleRequest]

  def create(): Action[JsValue] = Action(parse.json) { implicit request =>
    withJson[CreateArticleRequest] { req =>
      // Process request
      val id = scala.util.Random.nextLong()
      Created(Json.obj(
        "id"      -> id,
        "title"   -> req.title,
        "message" -> "Article created"
      ))
    }
  }
}
```

---

## Step 459: Streaming JSON

```scala
// app/controllers/api/StreamApiController.scala
package controllers.api

import javax.inject.*
import play.api.mvc.*
import play.api.libs.json.*
import play.api.http.ContentTypes
import org.apache.pekko.stream.scaladsl.*
import org.apache.pekko.util.ByteString
import scala.concurrent.ExecutionContext

@Singleton
class StreamApiController @Inject()(
  val controllerComponents: ControllerComponents,
  implicit val ec: ExecutionContext
) extends BaseController {

  // Stream large JSON response ด้วย chunked transfer
  def streamArticles(): Action[AnyContent] = Action { implicit request =>
    // สมมติมี articles จำนวนมาก
    val articles = (1 to 1000).map(i =>
      Json.obj("id" -> i, "title" -> s"Article $i")
    )

    // สร้าง JSON array stream
    val source: Source[ByteString, ?] = Source(articles)
      .map(article => ByteString(Json.stringify(article)))
      .intersperse(ByteString("["), ByteString(","), ByteString("]"))

    Ok.chunked(source).as(JSON)
  }

  // Parse streaming JSON input
  def processLargeUpload(): Action[JsValue] =
    Action(parse.json(maxLength = 10 * 1024 * 1024)) { implicit request =>
      // Process up to 10MB JSON
      val items = (request.body \ "items").as[JsArray].value
      Ok(Json.obj("received" -> items.length, "processed" -> true))
    }
}
```

---

## Step 460: play-json Best Practices

```scala
// app/models/package.scala
package models

import play.api.libs.json.*
import java.time.*

// Centralized implicit formats
package object formats {

  // Reusable date/time formats
  implicit val instantFormat: Format[Instant] = Format(
    Reads(_.validate[Long].map(Instant.ofEpochMilli)),
    Writes(i => JsNumber(i.toEpochMilli))
  )

  // Enum formats สำหรับ Scala 3 enums
  def enumFormat[E](
    fromString: String => Option[E],
    toString: E => String
  ): Format[E] = Format(
    Reads { json =>
      json.validate[String].flatMap { s =>
        fromString(s).fold[JsResult[E]](JsError(s"Unknown value: $s"))(JsSuccess(_))
      }
    },
    Writes(e => JsString(toString(e)))
  )
}

// Best practice: เก็บ format ใน companion object
case class Product(
  id: Long,
  name: String,
  price: BigDecimal,
  category: String,
  inStock: Boolean
)

object Product {
  // ใช้ Json.format macro - สะดวกที่สุด
  implicit val format: Format[Product] = Json.format[Product]

  // หรือ OFormat ถ้าต้องการ JsObject
  // implicit val oformat: OFormat[Product] = Json.format[Product]
}

// API Response wrapper
case class ApiResponse[T](
  data: T,
  meta: Map[String, String] = Map.empty
)

object ApiResponse {
  def success[T: Writes](data: T): JsValue = Json.obj(
    "success" -> true,
    "data"    -> Json.toJson(data)
  )

  def error(message: String, code: String = "ERROR"): JsValue = Json.obj(
    "success" -> false,
    "error"   -> Json.obj("code" -> code, "message" -> message)
  )

  def paginated[T: Writes](
    items: List[T],
    total: Long,
    page: Int,
    limit: Int
  ): JsValue = Json.obj(
    "success" -> true,
    "data"    -> Json.toJson(items),
    "meta"    -> Json.obj(
      "total"       -> total,
      "page"        -> page,
      "limit"       -> limit,
      "totalPages"  -> Math.ceil(total.toDouble / limit).toInt,
      "hasNextPage" -> (page * limit < total)
    )
  )
}
```

---

## สรุป Part 46

| Concept | Type | ตัวอย่าง |
|---------|------|---------|
| JsValue | Base type | `JsString`, `JsNumber`, `JsArray`, `JsObject` |
| Reads[T] | JSON → T | `Json.reads[T]` หรือ manual |
| Writes[T] | T → JSON | `Json.writes[T]` หรือ manual |
| Format[T] | Reads + Writes | `Json.format[T]` |
| OFormat[T] | Reads + OWrites | `Json.format[T]` |
| `validate[T]` | JsResult[T] | `json.validate[String]` |
| `as[T]` | T (throws) | `json.as[String]` |
| `asOpt[T]` | Option[T] | `json.asOpt[String]` |
| `Json.obj(...)` | JsObject | `Json.obj("key" -> value)` |
| `Json.arr(...)` | JsArray | `Json.arr(1, 2, 3)` |

---

## แบบฝึกหัด Part 46

1. **Custom Format**: สร้าง `Format` สำหรับ custom `Money` case class ที่มี `amount: BigDecimal` และ `currency: String` โดย serialize เป็น `"100.50 THB"`

2. **Nested JSON**: สร้าง `Reads[Order]` ที่ parse JSON ที่มี nested `OrderItem` list และ `Customer` object โดย map จาก snake_case JSON เป็น camelCase Scala

3. **JSON Validation**: สร้าง API endpoint ที่ validate JSON request body อย่างเข้มงวดและ return detailed error messages เมื่อ validation fail

4. **JSON Transformation**: สร้าง function ที่รับ user JSON จาก external API (snake_case) และแปลงเป็น internal format (camelCase) พร้อมเพิ่ม computed fields

5. **Pagination Response**: สร้าง generic `PaginatedResponse[T]` ที่ serialize เป็น JSON พร้อม metadata (total, page, hasNext) และใช้กับ Article และ Product endpoints

---

[→ ไปยัง Part 47: Play REST API](part-47-play-rest-api.md)
