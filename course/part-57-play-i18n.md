# Part 57: Play i18n (Internationalization)

## Steps 561-570: Messages Files, Locale Detection, Pluralization, Date/Number Formatting

---

## Step 561: i18n ใน Play

Play มี built-in i18n support ผ่าน Messages API

```
i18n คือ Internationalization (18 ตัวอักษรระหว่าง i และ n)
l10n คือ Localization

Play i18n:
├── Messages files (conf/messages.*)
├── Locale detection (Accept-Language header, cookie, session)
├── Pluralization
├── Format: date, number, currency
└── Template integration
```

### Configuration

```hocon
# conf/application.conf
play.i18n {
  langs      = ["th", "en", "zh", "ja"]
  path       = "messages"
  langCookieName = "PLAY_LANG"
  langCookieSecure = false
  langCookieHttpOnly = false
  langCookieMaxAge = null
}
```

---

## Step 562: Messages Files

```
# conf/messages (default - English)
# Fallback เมื่อไม่มี localized message

# Application
app.name=My Application
app.tagline=Build great things

# Navigation
nav.home=Home
nav.articles=Articles
nav.about=About
nav.login=Login
nav.logout=Logout

# Authentication
auth.login.title=Login
auth.login.email=Email Address
auth.login.password=Password
auth.login.submit=Sign In
auth.login.forgotPassword=Forgot password?
auth.login.success=Welcome back, {0}!
auth.login.failed=Invalid email or password

auth.register.title=Create Account
auth.register.name=Full Name
auth.register.email=Email Address
auth.register.password=Password
auth.register.confirmPassword=Confirm Password
auth.register.submit=Create Account
auth.register.success=Account created! Welcome {0}!

# Articles
article.create=Create Article
article.edit=Edit Article
article.delete=Delete Article
article.by=By {0}
article.publishedAt=Published {0}
article.readTime={0} min read
article.views={0,choice,0#no views|1#1 view|1<{0} views}
article.comments={0,choice,0#No comments|1#1 comment|1<{0} comments}

# Form validation
error.required=This field is required
error.email=Please enter a valid email address
error.minLength=Minimum {0} characters required
error.maxLength=Maximum {0} characters allowed
error.number=Please enter a number
error.min=Must be at least {0}
error.max=Must be at most {0}
error.password.weak=Password is too weak

# Common
button.save=Save
button.cancel=Cancel
button.delete=Delete
button.confirm=Confirm
button.back=Back

common.loading=Loading...
common.noResults=No results found
common.error=An error occurred
common.success=Operation successful
```

```
# conf/messages.th (Thai)

# Application
app.name=แอปพลิเคชันของฉัน
app.tagline=สร้างสิ่งยิ่งใหญ่

# Navigation
nav.home=หน้าแรก
nav.articles=บทความ
nav.about=เกี่ยวกับเรา
nav.login=เข้าสู่ระบบ
nav.logout=ออกจากระบบ

# Authentication
auth.login.title=เข้าสู่ระบบ
auth.login.email=อีเมล
auth.login.password=รหัสผ่าน
auth.login.submit=เข้าสู่ระบบ
auth.login.forgotPassword=ลืมรหัสผ่าน?
auth.login.success=ยินดีต้อนรับกลับมา {0}!
auth.login.failed=อีเมลหรือรหัสผ่านไม่ถูกต้อง

auth.register.title=สร้างบัญชี
auth.register.name=ชื่อ-นามสกุล
auth.register.email=อีเมล
auth.register.password=รหัสผ่าน
auth.register.confirmPassword=ยืนยันรหัสผ่าน
auth.register.submit=สร้างบัญชี
auth.register.success=สร้างบัญชีสำเร็จ! ยินดีต้อนรับ {0}!

# Articles
article.create=เขียนบทความ
article.edit=แก้ไขบทความ
article.delete=ลบบทความ
article.by=โดย {0}
article.publishedAt=เผยแพร่เมื่อ {0}
article.readTime=ใช้เวลาอ่าน {0} นาที
article.views={0,choice,0#ยังไม่มีการดู|1#ดู 1 ครั้ง|1<ดู {0} ครั้ง}
article.comments={0,choice,0#ยังไม่มีความคิดเห็น|1#1 ความคิดเห็น|1<{0} ความคิดเห็น}

# Form validation
error.required=กรุณากรอกข้อมูลในช่องนี้
error.email=กรุณากรอกอีเมลที่ถูกต้อง
error.minLength=ต้องมีอย่างน้อย {0} ตัวอักษร
error.maxLength=ต้องมีไม่เกิน {0} ตัวอักษร
error.number=กรุณากรอกตัวเลข
error.min=ต้องมีค่าอย่างน้อย {0}
error.max=ต้องมีค่าไม่เกิน {0}
error.password.weak=รหัสผ่านไม่แข็งแรงพอ

# Common
button.save=บันทึก
button.cancel=ยกเลิก
button.delete=ลบ
button.confirm=ยืนยัน
button.back=กลับ

common.loading=กำลังโหลด...
common.noResults=ไม่พบผลลัพธ์
common.error=เกิดข้อผิดพลาด
common.success=ดำเนินการสำเร็จ
```

---

## Step 563: ใช้ Messages ใน Controller

```scala
// app/controllers/I18nController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.i18n.*

@Singleton
class I18nController @Inject()(
  val controllerComponents: ControllerComponents,
  override val messagesApi: MessagesApi
) extends BaseController with I18nSupport {

  def index(): Action[AnyContent] = Action { implicit request =>
    // implicit request ทำให้ messages ทำงานได้
    val welcomeMsg = Messages("app.name")
    val loginMsg   = Messages("nav.login")

    // Messages ที่มี arguments
    val loginSuccess = Messages("auth.login.success", "John")

    Ok(s"$welcomeMsg - $loginMsg - $loginSuccess")
  }

  // เปลี่ยนภาษา
  def setLanguage(lang: String): Action[AnyContent] = Action { implicit request =>
    val newLang = Lang(lang)
    if (messagesApi.isDefinedAt(newLang)) {
      Redirect(request.headers.get("Referer").getOrElse("/"))
        .withLang(newLang)  // บันทึกภาษาใน cookie
    } else {
      BadRequest(s"Unsupported language: $lang")
    }
  }

  // ดึงภาษาปัจจุบัน
  def currentLang(): Action[AnyContent] = Action { implicit request =>
    val lang = request.lang
    Ok(play.api.libs.json.Json.obj(
      "lang"    -> lang.language,
      "country" -> lang.country,
      "code"    -> lang.code
    ))
  }
}
```

---

## Step 564: Messages ใน Templates

```html
@* app/views/layouts/main.scala.html *@
@(title: String)(content: Html)(implicit request: Request[?], messages: Messages)

<!DOCTYPE html>
<html lang="@messages.lang.language">
<head>
  <title>@title - @messages("app.name")</title>
</head>
<body>
  <nav>
    <a href="@routes.HomeController.index()">@messages("nav.home")</a>
    <a href="@routes.ArticleController.index()">@messages("nav.articles")</a>

    @* Language switcher *@
    <div class="language-switcher">
      @for(lang <- Seq("th", "en", "zh")) {
        <a href="@routes.I18nController.setLanguage(lang)"
           class="@if(messages.lang.language == lang){active}">
          @lang.toUpperCase
        </a>
      }
    </div>

    @* User menu *@
    @request.session.get("username").map { username =>
      <span>@messages("auth.login.success", username)</span>
      <a href="@routes.AuthController.logout()">@messages("nav.logout")</a>
    }.getOrElse {
      <a href="@routes.AuthController.login()">@messages("nav.login")</a>
    }
  </nav>

  @content
</body>
</html>
```

```html
@* app/views/auth/login.scala.html *@
@(form: Form[?])(implicit request: Request[?], messages: Messages)

@layouts.main(messages("auth.login.title")) {
  <div class="row justify-content-center">
    <div class="col-md-5">
      <h1>@messages("auth.login.title")</h1>

      @helper.form(action = routes.AuthController.processLogin()) {
        @helper.CSRF.formField

        <div class="mb-3">
          <label>@messages("auth.login.email")</label>
          <input type="email" name="email" class="form-control"
                 value="@form("email").value.getOrElse("")">
          @for(error <- form("email").errors) {
            <small class="text-danger">@messages(error.message, error.args: _*)</small>
          }
        </div>

        <div class="mb-3">
          <label>@messages("auth.login.password")</label>
          <input type="password" name="password" class="form-control">
        </div>

        <button type="submit" class="btn btn-primary">
          @messages("auth.login.submit")
        </button>

        <a href="@routes.AuthController.forgotPassword()" class="ms-3">
          @messages("auth.login.forgotPassword")
        </a>
      }
    </div>
  </div>
}
```

---

## Step 565: Locale Detection

```scala
// app/filters/LocaleFilter.scala
package filters

import javax.inject.*
import org.apache.pekko.stream.Materializer
import play.api.mvc.*
import play.api.i18n.*
import scala.concurrent.*

@Singleton
class LocaleFilter @Inject()(implicit
  val mat: Materializer,
  messagesApi: MessagesApi,
  ec: ExecutionContext
) extends Filter {

  override def apply(
    next: RequestHeader => Future[Result]
  )(request: RequestHeader): Future[Result] = {
    // Priority: session > cookie > Accept-Language header
    val lang = detectLanguage(request)

    next(request).map { result =>
      // เก็บ language preference
      result.withLang(lang)(messagesApi)
    }
  }

  private def detectLanguage(request: RequestHeader): Lang = {
    // 1. Session preference
    request.session.get("lang").flatMap(code =>
      messagesApi.availables.find(_.code == code)
    ).orElse(
      // 2. Cookie preference
      request.cookies.get("PLAY_LANG").flatMap(cookie =>
        messagesApi.availables.find(_.code == cookie.value)
      )
    ).orElse(
      // 3. Accept-Language header
      request.headers.get("Accept-Language").flatMap { acceptLanguage =>
        parseAcceptLanguage(acceptLanguage)
          .flatMap(lang => messagesApi.availables.find(_.language == lang))
      }
    ).getOrElse(
      // 4. Default language
      messagesApi.availables.headOption.getOrElse(Lang("th"))
    )
  }

  private def parseAcceptLanguage(header: String): Option[String] = {
    // Parse "th-TH,th;q=0.9,en;q=0.8"
    header.split(",")
      .map(_.split(";").head.trim.split("-").head.trim.toLowerCase)
      .headOption
  }
}
```

---

## Step 566: Pluralization

```scala
// Play ใช้ Java MessageFormat สำหรับ pluralization

// conf/messages.th
article.count={0,choice,0#ไม่มีบทความ|1#มี 1 บทความ|1<มี {0} บทความ}
user.count={0,choice,1#1 ผู้ใช้|1<{0} ผู้ใช้}
item.added={0,choice,1#เพิ่ม {0} รายการ|1<เพิ่ม {0} รายการ} ในตะกร้า
```

```scala
// ใช้ใน controller/template
val count = 5
val msg = Messages("article.count", count)
// ผลลัพธ์: "มี 5 บทความ"

val oneMsg = Messages("article.count", 1)
// ผลลัพธ์: "มี 1 บทความ"

val zeroMsg = Messages("article.count", 0)
// ผลลัพธ์: "ไม่มีบทความ"
```

---

## Step 567: Date และ Number Formatting

```scala
// app/utils/I18nFormatters.scala
package utils

import java.text.{NumberFormat, DateFormat}
import java.time.*
import java.time.format.DateTimeFormatter
import java.util.Locale
import play.api.i18n.Lang

object I18nFormatters {

  // Format number ตาม locale
  def formatNumber(n: Long, lang: Lang): String = {
    val locale = lang.toLocale
    NumberFormat.getNumberInstance(locale).format(n)
  }

  def formatDecimal(n: Double, decimals: Int = 2, lang: Lang): String = {
    val locale = lang.toLocale
    val fmt = NumberFormat.getNumberInstance(locale)
    fmt.setMinimumFractionDigits(decimals)
    fmt.setMaximumFractionDigits(decimals)
    fmt.format(n)
  }

  // Format currency
  def formatCurrency(amount: BigDecimal, currencyCode: String, lang: Lang): String = {
    val locale = lang.toLocale
    val fmt = NumberFormat.getCurrencyInstance(locale)
    val currency = java.util.Currency.getInstance(currencyCode)
    fmt.setCurrency(currency)
    fmt.format(amount.toDouble)
  }

  // Format date ตาม locale
  def formatDate(date: LocalDate, lang: Lang): String = {
    val locale = lang.toLocale
    val formatter = DateTimeFormatter.ofLocalizedDate(java.time.format.FormatStyle.LONG)
                                     .withLocale(locale)
    date.format(formatter)
  }

  def formatDateTime(dt: LocalDateTime, lang: Lang): String = {
    val locale = lang.toLocale
    val formatter = DateTimeFormatter.ofLocalizedDateTime(
      java.time.format.FormatStyle.MEDIUM,
      java.time.format.FormatStyle.SHORT
    ).withLocale(locale)
    dt.format(formatter)
  }

  // Thai Buddhist Calendar
  def toThaiDate(date: LocalDate): String = {
    val thaiYear = date.getYear + 543
    val thaiMonths = Array(
      "", "มกราคม", "กุมภาพันธ์", "มีนาคม", "เมษายน",
      "พฤษภาคม", "มิถุนายน", "กรกฎาคม", "สิงหาคม",
      "กันยายน", "ตุลาคม", "พฤศจิกายน", "ธันวาคม"
    )
    s"${date.getDayOfMonth} ${thaiMonths(date.getMonthValue)} พ.ศ. $thaiYear"
  }

  // Format เวลาเป็น relative (5 นาทีที่แล้ว)
  def timeAgo(dt: LocalDateTime, lang: Lang): String = {
    val now = LocalDateTime.now()
    val seconds = java.time.Duration.between(dt, now).getSeconds

    if (lang.language == "th") {
      if (seconds < 60) "เมื่อสักครู่"
      else if (seconds < 3600) s"${seconds / 60} นาทีที่แล้ว"
      else if (seconds < 86400) s"${seconds / 3600} ชั่วโมงที่แล้ว"
      else if (seconds < 604800) s"${seconds / 86400} วันที่แล้ว"
      else toThaiDate(dt.toLocalDate)
    } else {
      if (seconds < 60) "just now"
      else if (seconds < 3600) s"${seconds / 60} minutes ago"
      else if (seconds < 86400) s"${seconds / 3600} hours ago"
      else if (seconds < 604800) s"${seconds / 86400} days ago"
      else formatDate(dt.toLocalDate, lang)
    }
  }
}
```

---

## Step 568: Messages ใน JSON API

```scala
// app/controllers/api/I18nApiController.scala
package controllers.api

import javax.inject.*
import play.api.mvc.*
import play.api.i18n.*
import play.api.libs.json.*
import scala.concurrent.*

@Singleton
class I18nApiController @Inject()(
  val controllerComponents: ControllerComponents,
  override val messagesApi: MessagesApi,
  implicit val ec: ExecutionContext
) extends BaseController with I18nSupport {

  // Return localized messages สำหรับ frontend
  def messages(): Action[AnyContent] = Action { implicit request =>
    val lang = request.lang

    // ส่ง messages ที่ frontend ต้องการ
    val messages = Map(
      "nav.home"      -> messagesApi("nav.home")(lang),
      "nav.articles"  -> messagesApi("nav.articles")(lang),
      "button.save"   -> messagesApi("button.save")(lang),
      "button.cancel" -> messagesApi("button.cancel")(lang),
      "common.loading"-> messagesApi("common.loading")(lang),
      "common.error"  -> messagesApi("common.error")(lang)
    )

    Ok(Json.toJson(messages))
      .withHeaders("Content-Language" -> lang.code)
  }

  // Localized error responses
  def createArticle(): Action[JsValue] = Action(parse.json) { implicit request =>
    val title = (request.body \ "title").asOpt[String]

    title match {
      case None =>
        UnprocessableEntity(Json.obj(
          "error" -> Messages("error.required")
        ))
      case Some(t) if t.length < 3 =>
        UnprocessableEntity(Json.obj(
          "error" -> Messages("error.minLength", 3)
        ))
      case Some(t) =>
        Created(Json.obj("title" -> t, "message" -> Messages("common.success")))
    }
  }
}
```

---

## Step 569: Custom Message Formats

```scala
// app/utils/MessageFormatter.scala
package utils

import play.api.i18n.{Lang, MessagesApi}

class MessageFormatter(messagesApi: MessagesApi, lang: Lang) {

  def apply(key: String, args: Any*): String =
    messagesApi(key, args: _*)(lang)

  // Helper สำหรับ common patterns
  def articleCount(n: Int): String = apply("article.count", n)

  def timeAgo(dt: java.time.LocalDateTime): String =
    I18nFormatters.timeAgo(dt, lang)

  def formatDate(date: java.time.LocalDate): String =
    if (lang.language == "th") I18nFormatters.toThaiDate(date)
    else I18nFormatters.formatDate(date, lang)

  def currency(amount: BigDecimal): String = lang.language match {
    case "th" => I18nFormatters.formatCurrency(amount, "THB", lang)
    case _    => I18nFormatters.formatCurrency(amount, "USD", lang)
  }
}
```

---

## Step 570: Testing i18n

```scala
// test/i18n/I18nSpec.scala
package i18n

import org.scalatestplus.play.*
import org.scalatestplus.play.guice.GuiceOneAppPerTest
import play.api.i18n.*
import play.api.test.*
import play.api.test.Helpers.*

class I18nSpec extends PlaySpec with GuiceOneAppPerTest {

  "Messages" should {
    "return Thai messages for Thai locale" in {
      val messagesApi = app.injector.instanceOf[MessagesApi]
      implicit val lang = Lang("th")

      messagesApi("nav.home") mustBe "หน้าแรก"
      messagesApi("button.save") mustBe "บันทึก"
    }

    "return English messages for English locale" in {
      val messagesApi = app.injector.instanceOf[MessagesApi]
      implicit val lang = Lang("en")

      messagesApi("nav.home") mustBe "Home"
      messagesApi("button.save") mustBe "Save"
    }

    "format messages with arguments" in {
      val messagesApi = app.injector.instanceOf[MessagesApi]
      implicit val lang = Lang("th")

      messagesApi("auth.login.success", "John") mustBe "ยินดีต้อนรับกลับมา John!"
    }

    "handle pluralization" in {
      val messagesApi = app.injector.instanceOf[MessagesApi]
      implicit val lang = Lang("th")

      messagesApi("article.count", 0) mustBe "ไม่มีบทความ"
      messagesApi("article.count", 1) mustBe "มี 1 บทความ"
      messagesApi("article.count", 5) mustBe "มี 5 บทความ"
    }
  }

  "Language detection" should {
    "use Accept-Language header" in {
      val request = FakeRequest().withHeaders("Accept-Language" -> "th-TH,th;q=0.9")
      val result = route(app, request.withTarget(request.target.withPath("/"))).get

      // ตรวจสอบว่าใช้ภาษาไทย
      header("Content-Language", result) mustBe Some("th")
    }
  }
}
```

---

## สรุป Part 57

| Concept | Implementation | ตัวอย่าง |
|---------|---------------|---------|
| Messages files | `conf/messages.{lang}` | `conf/messages.th` |
| Message lookup | `Messages("key")` | `Messages("nav.home")` |
| Message with args | `Messages("key", arg1, arg2)` | `Messages("login.success", "John")` |
| Pluralization | `{0,choice,...}` | `{0,choice,1#1 item|1<{0} items}` |
| Lang detection | Accept-Language, cookie, session | Auto-detected |
| Lang switching | `Redirect(...).withLang(lang)` | Language selector |
| Date format | `DateTimeFormatter.withLocale` | Thai Buddhist date |
| Template | `(implicit messages: Messages)` | Template parameter |

---

## แบบฝึกหัด Part 57

1. **Multi-language Site**: แปล web application เป็น 3 ภาษา (Thai, English, Japanese) พร้อม language switcher ใน navigation

2. **Date Formatting**: สร้าง helper ที่ format dates เป็น Thai Buddhist Calendar (พ.ศ.) สำหรับ language=th และ Gregorian สำหรับ others

3. **Currency Localization**: สร้าง price display ที่แสดง ฿ สำหรับ Thai, $ สำหรับ English ด้วย proper formatting

4. **RTL Support**: เพิ่ม Arabic (ar) language support พร้อม RTL (right-to-left) CSS ที่ auto-switch ตาม language

5. **SEO i18n**: Implement SEO-friendly i18n ด้วย hreflang tags, canonical URLs, และ language-specific sitemaps

---

[→ ไปยัง Part 58: Play Modules](part-58-play-modules.md)
