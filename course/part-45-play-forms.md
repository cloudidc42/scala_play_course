# Part 45: Play Forms

## Steps 441-450: Form Binding, Validation, Error Messages, CSRF Protection, Custom Constraints

---

## Step 441: Play Forms คืออะไร?

Play Forms คือ type-safe mechanism สำหรับ handling HTML form data, validation, และ error display

```
HTML Form Submit
       │
       ▼
  bindFromRequest()
       │
       ▼
  Form Validation
       │
    ┌──┴──┐
   fail  success
    │       │
BadRequest  process
(form w/   data
 errors)
```

### build.sbt

```scala
// build.sbt
libraryDependencies ++= Seq(
  guice,
  "org.scalatestplus.play" %% "scalatestplus-play" % "7.0.1" % Test
)
// Play Forms API เป็น built-in ไม่ต้องเพิ่ม dependency
```

---

## Step 442: Basic Form Definition

```scala
// app/controllers/UserController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.data.*
import play.api.data.Forms.*
import play.api.data.validation.Constraints.*

@Singleton
class UserController @Inject()(
  val controllerComponents: ControllerComponents,
  implicit val messagesProvider: play.api.i18n.MessagesApi
) extends BaseController with play.api.i18n.I18nSupport {

  // Form สำหรับ data type แบบง่าย
  val loginForm: Form[(String, String)] = Form(
    tuple(
      "email"    -> email,        // validates email format
      "password" -> nonEmptyText  // ต้องไม่ว่าง
    )
  )

  // Form สำหรับ case class
  case class RegisterData(
    name: String,
    email: String,
    password: String,
    confirmPassword: String,
    age: Int,
    acceptTerms: Boolean
  )

  val registerForm: Form[RegisterData] = Form(
    mapping(
      "name"            -> nonEmptyText(minLength = 2, maxLength = 50),
      "email"           -> email,
      "password"        -> nonEmptyText(minLength = 8),
      "confirmPassword" -> nonEmptyText,
      "age"             -> number(min = 18, max = 120),
      "acceptTerms"     -> boolean.verifying("ต้องยอมรับข้อตกลง", identity)
    )(RegisterData.apply)(r => Some((r.name, r.email, r.password, r.confirmPassword, r.age, r.acceptTerms)))
  )

  def showLogin(): Action[AnyContent] = Action { implicit request =>
    Ok(views.html.users.login(loginForm))
  }

  def processLogin(): Action[AnyContent] = Action { implicit request =>
    loginForm.bindFromRequest().fold(
      // กรณีมี validation error
      formWithErrors => {
        BadRequest(views.html.users.login(formWithErrors))
      },
      // กรณีผ่าน validation
      { case (email, password) =>
        // ตรวจสอบ credentials...
        if (email == "admin@example.com" && password == "password") {
          Redirect(routes.HomeController.index())
            .withSession("email" -> email)
            .flashing("success" -> "Login สำเร็จ!")
        } else {
          Redirect(routes.UserController.showLogin())
            .flashing("error" -> "Email หรือ Password ไม่ถูกต้อง")
        }
      }
    )
  }

  def showRegister(): Action[AnyContent] = Action { implicit request =>
    Ok(views.html.users.register(registerForm))
  }

  def processRegister(): Action[AnyContent] = Action { implicit request =>
    registerForm.bindFromRequest().fold(
      formWithErrors => BadRequest(views.html.users.register(formWithErrors)),
      data => {
        if (data.password != data.confirmPassword) {
          val formWithError = registerForm.fill(data).withError("confirmPassword", "รหัสผ่านไม่ตรงกัน")
          BadRequest(views.html.users.register(formWithError))
        } else {
          // สร้าง user...
          Redirect(routes.HomeController.index())
            .flashing("success" -> s"ลงทะเบียนสำเร็จ! ยินดีต้อนรับ ${data.name}")
        }
      }
    )
  }
}
```

---

## Step 443: Form Field Types

```scala
// Field types ที่ Play Forms รองรับ

val formWithAllTypes = Form(
  mapping(
    // Text fields
    "text"      -> text,                           // String (อาจว่างได้)
    "nonEmpty"  -> nonEmptyText,                   // String ที่ต้องไม่ว่าง
    "withLen"   -> nonEmptyText(minLength = 3, maxLength = 100),

    // Number fields
    "intField"  -> number,                         // Int
    "longField" -> longNumber,                     // Long
    "dblField"  -> of[Double],                     // Double (ด้วย Formatter)

    // Number with range
    "age"       -> number(min = 1, max = 150),
    "price"     -> bigDecimal(10, 2),              // BigDecimal

    // Boolean
    "check"     -> boolean,                        // Boolean (checkbox)

    // Date/Time
    "date"      -> localDate("yyyy-MM-dd"),        // LocalDate
    "datetime"  -> localDateTime("yyyy-MM-dd'T'HH:mm"),  // LocalDateTime

    // Email
    "email"     -> email,

    // Optional fields
    "optText"   -> optional(text),                 // Option[String]
    "optInt"    -> optional(number),               // Option[Int]

    // List fields
    "tags"      -> list(text),                     // List[String]
    "ids"       -> list(longNumber),               // List[Long]

    // Ignored field (ใช้ fixed value)
    "role"      -> ignored("user")                 // String = "user" เสมอ

  )(MyData.apply)(d => Some(d.text, d.nonEmpty, /* ... */))
)
```

---

## Step 444: Custom Validation Constraints

```scala
// app/controllers/ValidationController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.data.*
import play.api.data.Forms.*
import play.api.data.validation.*
import play.api.data.validation.Constraints.*

@Singleton
class ValidationController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController with play.api.i18n.I18nSupport {

  // Custom constraint: Thai phone number
  val thaiPhone: Constraint[String] = Constraint("thaiPhone") { phone =>
    val cleaned = phone.replaceAll("[\\s\\-]", "")
    if (cleaned.matches("^(\\+66|0)[0-9]{8,9}$"))
      Valid
    else
      Invalid(ValidationError("เบอร์โทรศัพท์ไทยไม่ถูกต้อง (ตัวอย่าง: 081-234-5678)"))
  }

  // Custom constraint: Strong password
  val strongPassword: Constraint[String] = Constraint("strongPassword") { password =>
    val errors = scala.collection.mutable.ListBuffer[ValidationError]()

    if (password.length < 8)
      errors += ValidationError("รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร")
    if (!password.exists(_.isUpper))
      errors += ValidationError("รหัสผ่านต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว")
    if (!password.exists(_.isLower))
      errors += ValidationError("รหัสผ่านต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว")
    if (!password.exists(_.isDigit))
      errors += ValidationError("รหัสผ่านต้องมีตัวเลขอย่างน้อย 1 ตัว")
    if (!password.exists("!@#$%^&*".contains(_)))
      errors += ValidationError("รหัสผ่านต้องมีอักขระพิเศษ (!@#$%^&*) อย่างน้อย 1 ตัว")

    if (errors.isEmpty) Valid
    else Invalid(errors.toList)
  }

  // Custom constraint: Thai national ID
  val thaiNationalId: Constraint[String] = Constraint("thaiNationalId") { id =>
    val digits = id.replaceAll("[\\s\\-]", "")
    if (digits.length != 13 || !digits.forall(_.isDigit)) {
      Invalid(ValidationError("เลขบัตรประชาชนต้องมี 13 หลัก"))
    } else {
      // Luhn algorithm for Thai ID
      val weights = List(13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2)
      val sum = digits.take(12).zip(weights).map { case (d, w) => d.asDigit * w }.sum
      val checkDigit = (11 - (sum % 11)) % 10
      if (checkDigit == digits(12).asDigit) Valid
      else Invalid(ValidationError("เลขบัตรประชาชนไม่ถูกต้อง"))
    }
  }

  // ใช้ custom constraints ใน form
  case class ProfileData(
    name: String,
    phone: String,
    nationalId: String,
    password: String
  )

  val profileForm: Form[ProfileData] = Form(
    mapping(
      "name"       -> nonEmptyText(minLength = 2, maxLength = 100),
      "phone"      -> text.verifying(thaiPhone),
      "nationalId" -> text.verifying(thaiNationalId),
      "password"   -> text.verifying(strongPassword)
    )(ProfileData.apply)(d => Some((d.name, d.phone, d.nationalId, d.password)))
  )
}
```

---

## Step 445: Cross-Field Validation

```scala
// app/controllers/RegistrationController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.data.*
import play.api.data.Forms.*
import play.api.data.validation.*

@Singleton
class RegistrationController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController with play.api.i18n.I18nSupport {

  case class RegistrationData(
    startDate: java.time.LocalDate,
    endDate: java.time.LocalDate,
    minBudget: BigDecimal,
    maxBudget: BigDecimal,
    password: String,
    confirmPassword: String
  )

  val registrationForm: Form[RegistrationData] = Form(
    mapping(
      "startDate"       -> localDate,
      "endDate"         -> localDate,
      "minBudget"       -> bigDecimal,
      "maxBudget"       -> bigDecimal,
      "password"        -> nonEmptyText(minLength = 8),
      "confirmPassword" -> nonEmptyText
    )(RegistrationData.apply)(d =>
      Some((d.startDate, d.endDate, d.minBudget, d.maxBudget, d.password, d.confirmPassword))
    ).verifying(
      // Cross-field validation: endDate ต้องมาหลัง startDate
      Constraint { data: RegistrationData =>
        if (data.endDate.isAfter(data.startDate)) Valid
        else Invalid(ValidationError("วันสิ้นสุดต้องมาหลังวันเริ่มต้น"))
      }
    ).verifying(
      // Cross-field validation: maxBudget ต้องมากกว่า minBudget
      Constraint { data: RegistrationData =>
        if (data.maxBudget > data.minBudget) Valid
        else Invalid(ValidationError("งบประมาณสูงสุดต้องมากกว่าต่ำสุด"))
      }
    ).verifying(
      // Cross-field validation: passwords ต้องตรงกัน
      Constraint { data: RegistrationData =>
        if (data.password == data.confirmPassword) Valid
        else Invalid(ValidationError("รหัสผ่านไม่ตรงกัน"))
      }
    )
  )

  def showForm(): Action[AnyContent] = Action { implicit request =>
    Ok(views.html.registration(registrationForm))
  }

  def process(): Action[AnyContent] = Action { implicit request =>
    registrationForm.bindFromRequest().fold(
      formWithErrors => BadRequest(views.html.registration(formWithErrors)),
      data => {
        // Process valid data
        Ok(s"Registered! Budget: ${data.minBudget} - ${data.maxBudget}")
      }
    )
  }
}
```

---

## Step 446: CSRF Protection

```hocon
# conf/application.conf
# CSRF protection - enabled by default ใน Play
play.filters.enabled += "play.filters.csrf.CSRFFilter"

# CSRF configuration
play.filters.csrf {
  token {
    name = "csrfToken"  # ชื่อ token field ใน form
    sign = true
  }
  cookie {
    name = "PLAY_CSRF_TOKEN"
    secure = false  # true ใน production
    httpOnly = false  # ต้อง false ถ้า JavaScript ต้องการอ่าน
  }
  # methods ที่ต้องการ CSRF check
  method.whiteList = ["GET", "HEAD", "OPTIONS"]
}
```

### CSRF ใน Forms

```html
@* app/views/users/register.scala.html *@
@(form: Form[?])(implicit request: Request[?], messages: MessagesProvider)
@import helper.CSRF

<form action="@routes.UserController.processRegister()" method="POST">
  @* CSRF token - ต้องมีใน every POST form *@
  @CSRF.formField

  <div class="mb-3">
    <label for="name">ชื่อ</label>
    <input type="text" name="name" id="name"
           value="@form("name").value.getOrElse("")"
           class="form-control @if(form("name").hasErrors){is-invalid}">
    @for(error <- form("name").errors) {
      <div class="invalid-feedback">@messages(error.message)</div>
    }
  </div>

  <button type="submit" class="btn btn-primary">สมัครสมาชิก</button>
</form>
```

### CSRF ใน AJAX Requests

```javascript
// JavaScript - อ่าน CSRF token จาก cookie
function getCsrfToken() {
  const match = document.cookie.match(/PLAY_CSRF_TOKEN=([^;]+)/);
  return match ? decodeURIComponent(match[1]) : null;
}

// เพิ่ม CSRF header ใน AJAX request
async function createArticle(data) {
  const response = await fetch('/api/articles', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Csrf-Token': getCsrfToken()  // Play ต้องการ header นี้
    },
    body: JSON.stringify(data)
  });
  return response.json();
}
```

### ปิด CSRF สำหรับ API Endpoints

```scala
// app/controllers/api/ArticleApiController.scala
package controllers.api

import javax.inject.*
import play.api.mvc.*
import play.filters.csrf.CSRFAddToken

@Singleton
class ArticleApiController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  // ปิด CSRF สำหรับ API (เพราะใช้ JWT แทน)
  def create(): Action[play.api.libs.json.JsValue] =
    Action(parse.json) { implicit request =>
      // API endpoints ที่รับ Authorization header ไม่ต้องการ CSRF
      Ok(play.api.libs.json.Json.obj("status" -> "created"))
    }
}
```

```hocon
# ปิด CSRF สำหรับ /api/* routes
play.filters.csrf.routeModifiers.noCheck = [nocsrf]
```

```
# conf/routes - เพิ่ม nocsrf modifier
+ nocsrf
POST    /api/articles     controllers.api.ArticleApiController.create()
```

---

## Step 447: Form Error Display

```html
@* app/views/components/formErrors.scala.html *@
@(form: Form[?])(implicit messages: MessagesProvider)

@* Global form errors (cross-field validation errors) *@
@if(form.hasGlobalErrors) {
  <div class="alert alert-danger">
    <ul class="mb-0">
      @for(error <- form.globalErrors) {
        <li>@messages(error.message, error.args: _*)</li>
      }
    </ul>
  </div>
}
```

```html
@* app/views/components/formField.scala.html *@
@(form: Form[?], field: String, label: String, inputType: String = "text")(implicit messages: MessagesProvider)

<div class="mb-3">
  <label for="@field" class="form-label">@label</label>
  <input
    type="@inputType"
    class="form-control @if(form(field).hasErrors){is-invalid}"
    id="@field"
    name="@field"
    value="@form(field).value.getOrElse("")"
  >
  @for(error <- form(field).errors) {
    <div class="invalid-feedback">
      @messages(error.message, error.args: _*)
    </div>
  }
</div>
```

---

## Step 448: Pre-filled Forms

```scala
// app/controllers/ArticleController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.data.*
import play.api.data.Forms.*

@Singleton
class ArticleController @Inject()(
  val controllerComponents: ControllerComponents,
  articleRepo: models.ArticleRepository
) extends BaseController with play.api.i18n.I18nSupport {

  case class ArticleData(
    title: String,
    content: String,
    tags: List[String],
    published: Boolean
  )

  val articleForm: Form[ArticleData] = Form(
    mapping(
      "title"     -> nonEmptyText(maxLength = 200),
      "content"   -> nonEmptyText,
      "tags"      -> list(text),
      "published" -> boolean
    )(ArticleData.apply)(d => Some((d.title, d.content, d.tags, d.published)))
  )

  // Edit form - pre-fill ด้วยข้อมูลที่มีอยู่
  def edit(id: Long): Action[AnyContent] = Action { implicit request =>
    articleRepo.findById(id) match {
      case Some(article) =>
        // pre-fill form ด้วยข้อมูล article ที่มีอยู่
        val prefilledForm = articleForm.fill(ArticleData(
          title     = article.title,
          content   = article.content,
          tags      = article.tags,
          published = article.published
        ))
        Ok(views.html.articles.edit(id, prefilledForm))

      case None =>
        NotFound("ไม่พบบทความ")
    }
  }

  def update(id: Long): Action[AnyContent] = Action { implicit request =>
    articleForm.bindFromRequest().fold(
      formWithErrors => BadRequest(views.html.articles.edit(id, formWithErrors)),
      data => {
        articleRepo.update(id, data.title, data.content, data.tags, data.published)
        Redirect(routes.ArticleController.show(id))
          .flashing("success" -> "อัพเดตบทความสำเร็จ!")
      }
    )
  }
}
```

---

## Step 449: Advanced Form Patterns

### Nested Form Mapping

```scala
// Nested case classes
case class Address(
  street: String,
  city: String,
  zipCode: String,
  country: String
)

case class PersonData(
  name: String,
  email: String,
  address: Address
)

val addressMapping = mapping(
  "street"  -> nonEmptyText,
  "city"    -> nonEmptyText,
  "zipCode" -> text(minLength = 5, maxLength = 5),
  "country" -> nonEmptyText
)(Address.apply)(d => Some((d.street, d.city, d.zipCode, d.country)))

val personForm: Form[PersonData] = Form(
  mapping(
    "name"    -> nonEmptyText,
    "email"   -> email,
    "address" -> addressMapping  // nested mapping
  )(PersonData.apply)(d => Some((d.name, d.email, d.address)))
)
```

### Form with Repeated Fields

```scala
// Form ที่มี list of objects
case class OrderItem(productId: Long, quantity: Int)
case class OrderData(
  customerId: Long,
  items: List[OrderItem]
)

val orderItemMapping = mapping(
  "productId" -> longNumber,
  "quantity"  -> number(min = 1)
)(OrderItem.apply)(i => Some((i.productId, i.quantity)))

val orderForm: Form[OrderData] = Form(
  mapping(
    "customerId" -> longNumber,
    "items"      -> list(orderItemMapping)  // list of nested objects
  )(OrderData.apply)(d => Some((d.customerId, d.items)))
)
```

---

## Step 450: Form Internationalization

```
# conf/messages (default - English)
# Form validation messages
error.required=This field is required
error.minLength=Minimum length is {0} characters
error.maxLength=Maximum length is {0} characters
error.email=Valid email address required
error.number=Must be a number
error.min=Must be greater than or equal to {0}
error.max=Must be less than or equal to {0}

# Custom messages
thaiPhone=Invalid Thai phone number
strongPassword=Password must contain uppercase, lowercase, digit and special character
```

```
# conf/messages.th (Thai)
# Form validation messages
error.required=กรุณากรอกข้อมูลในช่องนี้
error.minLength=ต้องมีอย่างน้อย {0} ตัวอักษร
error.maxLength=ต้องมีไม่เกิน {0} ตัวอักษร
error.email=กรุณากรอกอีเมลที่ถูกต้อง
error.number=กรุณากรอกตัวเลข
error.min=ต้องมีค่าอย่างน้อย {0}
error.max=ต้องมีค่าไม่เกิน {0}

# Custom messages
thaiPhone=เบอร์โทรศัพท์ไทยไม่ถูกต้อง
strongPassword=รหัสผ่านต้องมีตัวพิมพ์ใหญ่ เล็ก ตัวเลข และอักขระพิเศษ
```

```scala
// Controller ต้อง extend I18nSupport
@Singleton
class FormController @Inject()(
  val controllerComponents: ControllerComponents,
  override val messagesApi: play.api.i18n.MessagesApi
) extends BaseController with play.api.i18n.I18nSupport {

  def showForm(): Action[AnyContent] = Action { implicit request =>
    // implicit MessagesRequest ทำให้ template ใช้ messages() ได้
    Ok(views.html.myForm(myForm))
  }
}
```

---

## สรุป Part 45

| Concept | API | ตัวอย่าง |
|---------|-----|---------|
| Form definition | `Form(mapping(...))` | `Form(mapping("name" -> nonEmptyText))` |
| Bind request | `form.bindFromRequest()` | อ่านค่าจาก POST body |
| Handle result | `.fold(invalid, valid)` | Error handling |
| Pre-fill form | `form.fill(data)` | แสดง edit form |
| Custom constraint | `Constraint { ... }` | Validate complex rules |
| CSRF token | `@CSRF.formField` | ใส่ใน every POST form |
| Error messages | `form("field").errors` | แสดง validation errors |
| i18n | `messages("error.key")` | Localized messages |

---

## แบบฝึกหัด Part 45

1. **Registration Form**: สร้าง registration form ที่ validate: email (unique ตรวจจาก mock DB), password strength, Thai phone number, และ birthdate (ต้องอายุ 18+)

2. **Multi-step Form**: สร้าง 3-step form (Personal Info → Address → Confirmation) ที่เก็บ intermediate data ใน session

3. **Dynamic Fields**: สร้าง order form ที่ user เพิ่ม/ลบ order items ได้แบบ dynamic (ใช้ JavaScript + Twirl)

4. **File Upload Form**: สร้าง form ที่ upload รูปโปรไฟล์ พร้อม validate: ขนาดไม่เกิน 5MB, นามสกุล .jpg/.png/.gif เท่านั้น

5. **Search Form**: สร้าง advanced search form ที่มี full-text search, date range picker, multiple checkboxes (tags), dropdown (category), และ price range slider

---

[→ ไปยัง Part 46: Play JSON](part-46-play-json.md)
