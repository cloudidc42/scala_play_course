# Part 44: Play Templates (Twirl)

## Steps 431-440: Twirl Templates, @variables, @if @for, Layout Inheritance, Helpers, Reusable Components

---

## Step 431: Twirl Template Engine คืออะไร?

Twirl คือ type-safe HTML template engine ของ Play Framework ไฟล์มีนามสกุล `.scala.html` และ compile เป็น Scala function

### ข้อดีของ Twirl

```
Twirl Template Benefits
├── Type-safe: compile error ถ้า type ไม่ตรง
├── Auto-escaping: ป้องกัน XSS อัตโนมัติ
├── Composable: template ซ้อน template ได้
├── IDE support: syntax highlighting, autocomplete
└── Performance: compile เป็น Scala ล่วงหน้า
```

### Twirl Syntax Basics

```html
@* app/views/basics.scala.html *@
@* Parameter declaration - ต้องอยู่บรรทัดแรก *@
@(name: String, age: Int, items: List[String])

@* นี่คือ comment ใน Twirl *@

<!DOCTYPE html>
<html>
<body>
  @* แสดง variable ด้วย @ *@
  <h1>Hello, @name!</h1>
  <p>Age: @age</p>

  @* Expression ใน { } *@
  <p>Next year: @{age + 1}</p>

  @* Escape @ ด้วย @@ *@
  <p>Email: user@@example.com</p>

  @* Conditional *@
  @if(age >= 18) {
    <p>Adult</p>
  } else {
    <p>Minor</p>
  }

  @* Loop *@
  <ul>
    @for(item <- items) {
      <li>@item</li>
    }
  </ul>
</body>
</html>
```

---

## Step 432: Template Parameters และ Types

```html
@* app/views/product.scala.html *@
@* รับ case class เป็น parameter *@
@(product: models.Product, user: Option[models.User])(implicit request: Request[?])

@* import ใน template *@
@import java.time.format.DateTimeFormatter
@import models.ProductStatus

<!DOCTYPE html>
<html>
<head>
  <title>@product.name</title>
</head>
<body>
  <h1>@product.name</h1>

  @* Access fields ของ case class *@
  <p>Price: ฿@product.price</p>
  <p>Stock: @product.stock units</p>

  @* Format date *@
  <p>Created: @product.createdAt.format(DateTimeFormatter.ofPattern("dd/MM/yyyy"))</p>

  @* Pattern match on enum *@
  <p>Status:
    @product.status match {
      case ProductStatus.Active   => { <span class="active">Available</span> }
      case ProductStatus.Inactive => { <span class="inactive">Unavailable</span> }
      case ProductStatus.Draft    => { <span class="draft">Draft</span> }
    }
  </p>

  @* Option type *@
  @user.map { u =>
    <p>Viewed by: @u.name</p>
  }.getOrElse {
    <p>Guest view</p>
  }
</body>
</html>
```

---

## Step 433: Layout Inheritance

### Main Layout Template

```html
@* app/views/layouts/main.scala.html *@
@(title: String, extraMeta: Html = Html(""))(content: Html)(implicit request: Request[?])

<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>@title - My App</title>

  @* CSS *@
  <link rel="stylesheet" href="@routes.Assets.versioned("stylesheets/main.css")">
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css">

  @* Extra meta tags (injectable) *@
  @extraMeta
</head>
<body>
  @* Navigation *@
  @components.navbar()

  @* Flash messages *@
  @components.flashMessages()

  @* Main content *@
  <main class="container mt-4">
    @content
  </main>

  @* Footer *@
  @components.footer()

  @* JavaScript *@
  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
  <script src="@routes.Assets.versioned("javascripts/main.js")"></script>
</body>
</html>
```

### Dashboard Layout

```html
@* app/views/layouts/dashboard.scala.html *@
@(title: String)(sidebar: Html)(content: Html)(implicit request: Request[?])

@layouts.main(title) {
  <div class="row">
    @* Sidebar *@
    <div class="col-md-3">
      <nav class="sidebar">
        @sidebar
      </nav>
    </div>

    @* Content *@
    <div class="col-md-9">
      @content
    </div>
  </div>
}
```

### ใช้ Layout ใน Page

```html
@* app/views/articles/index.scala.html *@
@(articles: List[models.Article], page: Int, total: Int)(implicit request: Request[?])

@* ใช้ main layout *@
@layouts.main("บทความทั้งหมด") {
  <div class="articles-page">
    <h1>บทความทั้งหมด</h1>

    @* Article list *@
    <div class="article-list">
      @for(article <- articles) {
        @components.articleCard(article)
      }
    </div>

    @* Pagination *@
    @components.pagination(page, total, 10, routes.ArticleController.list(_))
  </div>
}
```

---

## Step 434: Reusable Components

### Navbar Component

```html
@* app/views/components/navbar.scala.html *@
@()(implicit request: Request[?])

<nav class="navbar navbar-expand-lg navbar-dark bg-dark">
  <div class="container">
    <a class="navbar-brand" href="@routes.HomeController.index()">
      My App
    </a>

    <button class="navbar-toggler" type="button" data-bs-toggle="collapse"
            data-bs-target="#navbarNav">
      <span class="navbar-toggler-icon"></span>
    </button>

    <div class="collapse navbar-collapse" id="navbarNav">
      <ul class="navbar-nav ms-auto">
        @* Active link detection *@
        <li class="nav-item">
          <a class="nav-link @if(request.path == "/"){active}"
             href="@routes.HomeController.index()">หน้าแรก</a>
        </li>
        <li class="nav-item">
          <a class="nav-link @if(request.path.startsWith("/articles")){active}"
             href="@routes.ArticleController.index()">บทความ</a>
        </li>

        @* แสดง User menu ถ้า login แล้ว *@
        @request.session.get("username").map { username =>
          <li class="nav-item dropdown">
            <a class="nav-link dropdown-toggle" href="#" role="button"
               data-bs-toggle="dropdown">
              @username
            </a>
            <ul class="dropdown-menu">
              <li><a class="dropdown-item" href="@routes.ProfileController.index()">
                โปรไฟล์</a></li>
              <li><hr class="dropdown-divider"></li>
              <li><a class="dropdown-item" href="@routes.AuthController.logout()">
                ออกจากระบบ</a></li>
            </ul>
          </li>
        }.getOrElse {
          <li class="nav-item">
            <a class="nav-link" href="@routes.AuthController.login()">เข้าสู่ระบบ</a>
          </li>
        }
      </ul>
    </div>
  </div>
</nav>
```

### Flash Messages Component

```html
@* app/views/components/flashMessages.scala.html *@
@()(implicit request: Request[?])

@request.flash.get("success").map { msg =>
  <div class="alert alert-success alert-dismissible fade show" role="alert">
    <i class="bi bi-check-circle"></i> @msg
    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
  </div>
}

@request.flash.get("error").map { msg =>
  <div class="alert alert-danger alert-dismissible fade show" role="alert">
    <i class="bi bi-exclamation-triangle"></i> @msg
    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
  </div>
}

@request.flash.get("warning").map { msg =>
  <div class="alert alert-warning alert-dismissible fade show" role="alert">
    <i class="bi bi-exclamation-circle"></i> @msg
    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
  </div>
}

@request.flash.get("info").map { msg =>
  <div class="alert alert-info alert-dismissible fade show" role="alert">
    <i class="bi bi-info-circle"></i> @msg
    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
  </div>
}
```

### Article Card Component

```html
@* app/views/components/articleCard.scala.html *@
@(article: models.Article)

<div class="card mb-3">
  @article.imageUrl.map { url =>
    <img src="@url" class="card-img-top" alt="@article.title">
  }
  <div class="card-body">
    <h5 class="card-title">
      <a href="@routes.ArticleController.show(article.id)">@article.title</a>
    </h5>
    <p class="card-text">@article.excerpt</p>
    <div class="d-flex justify-content-between align-items-center">
      <div>
        @for(tag <- article.tags) {
          <span class="badge bg-secondary me-1">@tag</span>
        }
      </div>
      <small class="text-muted">@article.publishedAt.toLocalDate</small>
    </div>
  </div>
</div>
```

### Pagination Component

```html
@* app/views/components/pagination.scala.html *@
@(currentPage: Int, totalItems: Int, itemsPerPage: Int, urlBuilder: Int => Call)

@defining(math.ceil(totalItems.toDouble / itemsPerPage).toInt) { totalPages =>
  @if(totalPages > 1) {
    <nav aria-label="Page navigation">
      <ul class="pagination justify-content-center">
        @* Previous button *@
        @if(currentPage > 1) {
          <li class="page-item">
            <a class="page-link" href="@urlBuilder(currentPage - 1)">
              &laquo; ก่อนหน้า
            </a>
          </li>
        } else {
          <li class="page-item disabled">
            <span class="page-link">&laquo; ก่อนหน้า</span>
          </li>
        }

        @* Page numbers *@
        @for(page <- Math.max(1, currentPage - 2) to Math.min(totalPages, currentPage + 2)) {
          <li class="page-item @if(page == currentPage){active}">
            <a class="page-link" href="@urlBuilder(page)">@page</a>
          </li>
        }

        @* Next button *@
        @if(currentPage < totalPages) {
          <li class="page-item">
            <a class="page-link" href="@urlBuilder(currentPage + 1)">
              ถัดไป &raquo;
            </a>
          </li>
        } else {
          <li class="page-item disabled">
            <span class="page-link">ถัดไป &raquo;</span>
          </li>
        }
      </ul>
    </nav>
    <p class="text-center text-muted">
      แสดง @((currentPage - 1) * itemsPerPage + 1)-@(Math.min(currentPage * itemsPerPage, totalItems))
      จาก @totalItems รายการ
    </p>
  }
}
```

---

## Step 435: Helper Functions ใน Templates

### Template Helpers

```html
@* app/views/helpers/package.scala.html *@
@* (ไฟล์นี้ไม่มีจริง แต่แสดงวิธีสร้าง helper ใน object) *@
```

```scala
// app/views/helpers/ViewHelpers.scala
package views.helpers

import play.twirl.api.Html
import java.time.*
import java.time.format.DateTimeFormatter
import java.time.temporal.ChronoUnit

object ViewHelpers {

  // Format date เป็น Thai format
  def thaiDate(date: LocalDate): String = {
    val formatter = DateTimeFormatter.ofPattern("d MMMM")
    val thaiYear = date.getYear + 543  // พ.ศ.
    s"${date.format(formatter)} $thaiYear"
  }

  // Format datetime เป็น relative time (x minutes ago)
  def timeAgo(dateTime: LocalDateTime): String = {
    val now = LocalDateTime.now()
    val minutes = ChronoUnit.MINUTES.between(dateTime, now)
    val hours = ChronoUnit.HOURS.between(dateTime, now)
    val days = ChronoUnit.DAYS.between(dateTime, now)

    if (minutes < 1) "เมื่อสักครู่"
    else if (minutes < 60) s"$minutes นาทีที่แล้ว"
    else if (hours < 24) s"$hours ชั่วโมงที่แล้ว"
    else if (days < 7) s"$days วันที่แล้ว"
    else thaiDate(dateTime.toLocalDate)
  }

  // Format number ด้วย commas
  def formatNumber(n: Long): String =
    f"$n%,d"

  def formatNumber(n: Double, decimals: Int = 2): String =
    f"$n%,.${decimals}f"

  // Truncate text
  def truncate(text: String, maxLength: Int = 100): String =
    if (text.length <= maxLength) text
    else text.take(maxLength - 3) + "..."

  // Generate avatar initials
  def avatarInitials(name: String): String =
    name.split(" ").take(2).map(_.headOption.getOrElse(' ')).mkString.toUpperCase

  // Safe HTML (bypass auto-escaping - ใช้ระวัง!)
  def safeHtml(html: String): Html = Html(html)

  // Markdown to HTML (ต้องมี dependency)
  // def markdown(text: String): Html = Html(commonmark.render(text))
}
```

### ใช้ Helper ใน Template

```html
@* app/views/articles/show.scala.html *@
@(article: models.Article)(implicit request: Request[?])
@import views.helpers.ViewHelpers.*

@layouts.main(article.title) {
  <article>
    <header>
      <h1>@article.title</h1>
      <div class="meta">
        <span>โดย @article.authorName</span>
        <span>เมื่อ @timeAgo(article.publishedAt)</span>
        <span>@formatNumber(article.viewCount) ครั้ง</span>
      </div>
    </header>

    <div class="content">
      @* แสดง HTML content โดยไม่ escape *@
      @safeHtml(article.htmlContent)
    </div>

    <footer>
      <p>อัพเดตล่าสุด: @thaiDate(article.updatedAt.toLocalDate)</p>
    </footer>
  </article>
}
```

---

## Step 436: Twirl @defining และ @let

```html
@* app/views/calculations.scala.html *@
@(products: List[models.Product])

@* @defining สร้าง local variable ใน template *@
@defining(products.map(_.price).sum) { total =>
  @defining(products.length) { count =>
    <p>รวม @count รายการ ราคา ฿@total</p>
    <p>เฉลี่ย ฿@{if (count > 0) total / count else 0}</p>
  }
}

@* ใช้ @defining สำหรับ expensive computation *@
@defining(products.sortBy(_.price).take(3)) { cheapest =>
  <h3>3 สินค้าราคาถูกที่สุด:</h3>
  <ul>
    @for(p <- cheapest) {
      <li>@p.name - ฿@p.price</li>
    }
  </ul>
}
```

---

## Step 437: Form Templates

```html
@* app/views/articles/create.scala.html *@
@(form: Form[(String, String, List[String])])(implicit request: Request[?], messages: MessagesProvider)
@import helper.*
@import helper.CSRF

@layouts.main("สร้างบทความใหม่") {
  <div class="row justify-content-center">
    <div class="col-md-8">
      <h1>สร้างบทความใหม่</h1>

      @* CSRF token จาก Play *@
      @helper.form(action = routes.ArticleController.create()) {
        @CSRF.formField

        @* Text field *@
        <div class="mb-3">
          <label for="title" class="form-label">ชื่อบทความ</label>
          <input type="text"
                 class="form-control @if(form("title").hasErrors){is-invalid}"
                 id="title"
                 name="title"
                 value="@form("title").value.getOrElse("")">
          @for(error <- form("title").errors) {
            <div class="invalid-feedback">@error.message</div>
          }
        </div>

        @* Textarea *@
        <div class="mb-3">
          <label for="content" class="form-label">เนื้อหา</label>
          <textarea class="form-control @if(form("content").hasErrors){is-invalid}"
                    id="content"
                    name="content"
                    rows="10">@form("content").value.getOrElse("")</textarea>
          @for(error <- form("content").errors) {
            <div class="invalid-feedback">@error.message</div>
          }
        </div>

        @* Submit button *@
        <button type="submit" class="btn btn-primary">สร้างบทความ</button>
        <a href="@routes.ArticleController.index()" class="btn btn-secondary">ยกเลิก</a>
      }
    </div>
  </div>
}
```

---

## Step 438: Implicit Parameters ใน Templates

```html
@* app/views/layouts/main.scala.html *@
@* implicit request ทำให้ routes, session, flash ใช้งานได้ *@
@(title: String)(content: Html)(implicit request: Request[?], messages: MessagesProvider, lang: Lang)
```

```scala
// Controller ต้อง implicit request เพื่อใน template ใช้ได้
def show(id: Long): Action[AnyContent] = Action { implicit request =>
  // implicit request ถูกส่งต่อให้ template อัตโนมัติ
  Ok(views.html.articles.show(article))
}
```

### MessagesProvider สำหรับ i18n

```html
@* app/views/form.scala.html *@
@(form: Form[?])(implicit messages: MessagesProvider)

@* ใช้ messages("key") สำหรับ i18n *@
<label>@messages("article.title")</label>
<button>@messages("button.submit")</button>
```

```
# conf/messages.th
article.title=ชื่อบทความ
button.submit=บันทึก
error.required=กรุณากรอกข้อมูลนี้
```

---

## Step 439: JavaScript และ CSS ใน Templates

```html
@* app/views/articles/show.scala.html *@
@(article: models.Article)(implicit request: Request[?])

@layouts.main(article.title) {
  @* Content *@
  <article id="article-@article.id">
    <h1>@article.title</h1>
    <div class="content">@Html(article.htmlContent)</div>
  </article>

  @* Inline JavaScript ด้วย @defining เพื่อส่ง Scala data *@
  <script>
    // ส่ง data จาก Scala ไป JavaScript
    const articleData = {
      id: @article.id,
      title: "@article.title.replace("\"", "\\\""))",
      viewCount: @article.viewCount,
      tags: @Html(play.api.libs.json.Json.toJson(article.tags).toString())
    };

    // Track view
    fetch('/api/articles/@article.id/view', { method: 'POST' });
  </script>
}
```

---

## Step 440: Template Testing

```scala
// test/views/ArticleViewSpec.scala
package views

import org.scalatestplus.play.*
import play.api.test.*
import play.api.test.Helpers.*
import play.twirl.api.Html

class ArticleViewSpec extends PlaySpec {

  "Article index view" should {
    "display article list" in {
      val articles = List(
        models.Article(1L, "Test Article 1", "Content 1"),
        models.Article(2L, "Test Article 2", "Content 2")
      )

      val html = views.html.articles.index(articles, 1, 2)
      val content = contentAsString(html)

      content must include("Test Article 1")
      content must include("Test Article 2")
    }

    "display pagination when there are multiple pages" in {
      val articles = List(models.Article(1L, "Test", "Content"))
      val html = views.html.articles.index(articles, 1, 100)
      val content = contentAsString(html)

      content must include("pagination")
    }
  }

  "Article show view" should {
    "display article details" in {
      val article = models.Article(1L, "My Article", "My Content",
                                    tags = List("scala", "play"))

      implicit val request: play.api.mvc.RequestHeader = FakeRequest()
      val html = views.html.articles.show(article)
      val content = contentAsString(html)

      content must include("My Article")
      content must include("scala")
      content must include("play")
    }
  }
}
```

---

## สรุป Part 44

| Concept | คำอธิบาย |
|---------|---------|
| `.scala.html` | Twirl template file extension |
| `@variable` | แสดงค่า variable (auto-escaped) |
| `@{expression}` | Scala expression ที่ซับซ้อน |
| `@if(cond) { } else { }` | Conditional rendering |
| `@for(x <- list) { }` | Loop rendering |
| `@defining(expr) { val => }` | Local variable |
| `@Html(str)` | Render unescaped HTML |
| Layout template | Template ที่รับ `(content: Html)` parameter |
| Component | Template file ที่เรียกใช้ซ้ำได้ |
| Implicit request | ส่ง request context ไปยัง template |

---

## แบบฝึกหัด Part 44

1. **Main Layout**: สร้าง main layout template พร้อม navbar, footer, flash messages และ CSS framework ที่เลือกเอง

2. **Components Library**: สร้าง component library ประกอบด้วย: `card`, `badge`, `avatar`, `loading-spinner`, `empty-state` components ที่ reusable

3. **Product Listing Page**: สร้างหน้า product listing ที่มี: grid layout, search bar, filter sidebar, pagination, และ product card components

4. **Form Template**: สร้าง form template สำหรับ user registration ที่มี validation error display, field highlighting, และ success message

5. **Dashboard Template**: สร้าง 2-column dashboard layout ที่มี sidebar navigation, stat cards (ตัวเลขสำคัญ), และ data table

---

[→ ไปยัง Part 45: Play Forms](part-45-play-forms.md)
