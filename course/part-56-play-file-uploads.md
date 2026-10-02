# Part 56: Play File Uploads

## Steps 551-560: File Upload Handling, Multipart Form, S3 Integration, File Validation

---

## Step 551: File Upload ใน Play

```scala
// build.sbt
libraryDependencies ++= Seq(
  guice,
  "software.amazon.awssdk" % "s3" % "2.21.29",  // AWS S3
  "commons-io"             % "commons-io" % "2.15.1"
)
```

---

## Step 552: Simple File Upload

```scala
// app/controllers/FileUploadController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.libs.Files.TemporaryFile
import java.nio.file.{Files, Paths}

@Singleton
class FileUploadController @Inject()(
  val controllerComponents: ControllerComponents
) extends BaseController {

  // Basic file upload
  def upload(): Action[MultipartFormData[TemporaryFile]] =
    Action(parse.multipartFormData) { implicit request =>

      request.body.file("file").map { filePart =>
        val filename    = filePart.filename
        val contentType = filePart.contentType.getOrElse("application/octet-stream")
        val fileSize    = filePart.fileSize

        // ตรวจสอบ file
        if (fileSize > 10 * 1024 * 1024) {  // 10MB limit
          return BadRequest("File too large (max 10MB)")
        }

        if (!isAllowedType(filename, contentType)) {
          return BadRequest("File type not allowed")
        }

        // บันทึกไฟล์
        val savedPath = Paths.get(s"/var/uploads/${sanitizeFilename(filename)}")
        filePart.ref.moveTo(savedPath, replace = true)

        Ok(play.api.libs.json.Json.obj(
          "filename"    -> filename,
          "size"        -> fileSize,
          "contentType" -> contentType,
          "path"        -> savedPath.toString
        ))
      }.getOrElse {
        BadRequest("No file uploaded")
      }
    }

  // Upload พร้อม form data
  def uploadWithData(): Action[MultipartFormData[TemporaryFile]] =
    Action(parse.multipartFormData) { implicit request =>
      val title = request.body.dataParts.getOrElse("title", Seq.empty).headOption.getOrElse("")
      val desc  = request.body.dataParts.getOrElse("description", Seq.empty).headOption.getOrElse("")

      request.body.file("image").map { image =>
        val filename = s"${java.util.UUID.randomUUID()}.${getExtension(image.filename)}"
        val savedPath = Paths.get(s"/var/uploads/images/$filename")
        Files.createDirectories(savedPath.getParent)
        image.ref.moveTo(savedPath, replace = true)

        Ok(play.api.libs.json.Json.obj(
          "title"       -> title,
          "description" -> desc,
          "imageUrl"    -> s"/images/$filename"
        ))
      }.getOrElse(BadRequest("No image uploaded"))
    }

  private def isAllowedType(filename: String, contentType: String): Boolean = {
    val allowedExtensions = Set("jpg", "jpeg", "png", "gif", "webp", "pdf", "doc", "docx")
    val allowedTypes = Set(
      "image/jpeg", "image/png", "image/gif", "image/webp",
      "application/pdf",
      "application/msword",
      "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
    )
    val ext = getExtension(filename).toLowerCase
    allowedExtensions.contains(ext) && allowedTypes.contains(contentType)
  }

  private def sanitizeFilename(filename: String): String =
    filename.replaceAll("[^a-zA-Z0-9._-]", "_").take(255)

  private def getExtension(filename: String): String =
    filename.split("\\.").lastOption.getOrElse("")
}
```

---

## Step 553: File Validation

```scala
// app/utils/FileValidator.scala
package utils

import play.api.libs.Files.TemporaryFile
import play.api.mvc.MultipartFormData

case class FileValidationError(message: String)
case class FileValidationResult(
  isValid: Boolean,
  errors: List[String],
  mimeType: Option[String] = None
)

object FileValidator {

  val ImageTypes = Set("image/jpeg", "image/png", "image/gif", "image/webp")
  val DocumentTypes = Set(
    "application/pdf",
    "application/msword",
    "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
  )

  def validate(
    file: MultipartFormData.FilePart[TemporaryFile],
    maxSizeBytes: Long = 10 * 1024 * 1024,
    allowedTypes: Set[String] = ImageTypes ++ DocumentTypes
  ): FileValidationResult = {
    val errors = scala.collection.mutable.ListBuffer[String]()

    // 1. Check file size
    if (file.fileSize > maxSizeBytes) {
      errors += s"ไฟล์ใหญ่เกินไป (สูงสุด ${maxSizeBytes / 1024 / 1024}MB)"
    }

    // 2. Check MIME type จาก content type header
    val claimedType = file.contentType.getOrElse("application/octet-stream")
    if (!allowedTypes.contains(claimedType)) {
      errors += s"ประเภทไฟล์ไม่อนุญาต: $claimedType"
    }

    // 3. Check magic bytes (ตรวจ actual file content)
    val actualType = detectMimeType(file.ref.path.toFile)
    if (actualType != claimedType && actualType != "application/octet-stream") {
      errors += s"ไฟล์ไม่ตรงกับ Content-Type: อ้างว่าเป็น $claimedType แต่จริงๆ เป็น $actualType"
    }

    // 4. Check filename
    if (file.filename.isEmpty) {
      errors += "ชื่อไฟล์ต้องไม่ว่าง"
    }
    if (file.filename.length > 255) {
      errors += "ชื่อไฟล์ยาวเกินไป (สูงสุด 255 ตัวอักษร)"
    }

    FileValidationResult(errors.isEmpty, errors.toList, Some(actualType))
  }

  private def detectMimeType(file: java.io.File): String = {
    // อ่าน magic bytes เพื่อตรวจสอบ actual file type
    val bytes = new Array[Byte](12)
    val fis = new java.io.FileInputStream(file)
    try {
      fis.read(bytes)
    } finally {
      fis.close()
    }

    // Magic byte patterns
    if (bytes(0) == 0xFF.toByte && bytes(1) == 0xD8.toByte) "image/jpeg"
    else if (bytes(0) == 0x89.toByte && bytes(1) == 0x50.toByte && bytes(2) == 0x4E.toByte) "image/png"
    else if (bytes(0) == 0x47.toByte && bytes(1) == 0x49.toByte && bytes(2) == 0x46.toByte) "image/gif"
    else if (bytes(0) == 0x25.toByte && bytes(1) == 0x50.toByte && bytes(2) == 0x44.toByte) "application/pdf"
    else "application/octet-stream"
  }
}
```

---

## Step 554: S3 Integration

```scala
// app/services/S3Service.scala
package services

import javax.inject.*
import play.api.Configuration
import software.amazon.awssdk.auth.credentials.*
import software.amazon.awssdk.regions.Region
import software.amazon.awssdk.services.s3.S3Client
import software.amazon.awssdk.services.s3.model.*
import software.amazon.awssdk.services.s3.presigner.S3Presigner
import software.amazon.awssdk.services.s3.presigner.model.PutObjectPresignRequest
import java.nio.file.Path
import java.time.Duration
import scala.concurrent.*
import scala.util.*

@Singleton
class S3Service @Inject()(
  config: Configuration,
  implicit val ec: ExecutionContext
) {
  private val bucket    = config.get[String]("s3.bucket")
  private val region    = Region.of(config.get[String]("s3.region"))

  private val s3Client = S3Client.builder()
    .region(region)
    .credentialsProvider(DefaultCredentialsProvider.create())
    .build()

  private val presigner = S3Presigner.builder()
    .region(region)
    .credentialsProvider(DefaultCredentialsProvider.create())
    .build()

  // Upload file to S3
  def upload(
    key: String,
    filePath: Path,
    contentType: String,
    metadata: Map[String, String] = Map.empty
  ): Future[String] = Future {
    val request = PutObjectRequest.builder()
      .bucket(bucket)
      .key(key)
      .contentType(contentType)
      .metadata(scala.jdk.CollectionConverters.MapHasAsJava(metadata).asJava)
      .build()

    s3Client.putObject(request, filePath)

    // Return public URL
    s"https://$bucket.s3.$region.amazonaws.com/$key"
  }

  // Upload bytes
  def uploadBytes(
    key: String,
    bytes: Array[Byte],
    contentType: String
  ): Future[String] = Future {
    val request = PutObjectRequest.builder()
      .bucket(bucket)
      .key(key)
      .contentType(contentType)
      .contentLength(bytes.length.toLong)
      .build()

    s3Client.putObject(request, software.amazon.awssdk.core.sync.RequestBody.fromBytes(bytes))

    s"https://$bucket.s3.$region.amazonaws.com/$key"
  }

  // Generate presigned URL สำหรับ direct upload
  def generatePresignedUploadUrl(
    key: String,
    contentType: String,
    expirationMinutes: Int = 15
  ): Future[String] = Future {
    val putRequest = PutObjectRequest.builder()
      .bucket(bucket)
      .key(key)
      .contentType(contentType)
      .build()

    val presignRequest = PutObjectPresignRequest.builder()
      .signatureDuration(Duration.ofMinutes(expirationMinutes))
      .putObjectRequest(putRequest)
      .build()

    presigner.presignPutObject(presignRequest).url().toString
  }

  // Delete file
  def delete(key: String): Future[Unit] = Future {
    val request = DeleteObjectRequest.builder()
      .bucket(bucket)
      .key(key)
      .build()
    s3Client.deleteObject(request)
    ()
  }

  // Generate key สำหรับ file
  def generateKey(folder: String, filename: String): String = {
    val uuid = java.util.UUID.randomUUID().toString
    val ext  = filename.split("\\.").lastOption.getOrElse("")
    s"$folder/$uuid${if (ext.nonEmpty) s".$ext" else ""}"
  }
}
```

---

## Step 555: S3 Upload Controller

```scala
// app/controllers/S3UploadController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.libs.Files.TemporaryFile
import play.api.libs.json.*
import services.S3Service
import utils.FileValidator
import scala.concurrent.*

@Singleton
class S3UploadController @Inject()(
  val controllerComponents: ControllerComponents,
  s3Service: S3Service,
  implicit val ec: ExecutionContext
) extends BaseController {

  // Upload file โดยตรง (server → S3)
  def uploadImage(): Action[MultipartFormData[TemporaryFile]] =
    Action.async(parse.multipartFormData) { implicit request =>

      request.body.file("image") match {
        case None =>
          Future.successful(BadRequest(Json.obj("error" -> "No file uploaded")))

        case Some(file) =>
          // Validate file
          val validation = FileValidator.validate(
            file,
            maxSizeBytes = 5 * 1024 * 1024,  // 5MB
            allowedTypes = FileValidator.ImageTypes
          )

          if (!validation.isValid) {
            Future.successful(UnprocessableEntity(Json.obj(
              "error"  -> "Validation failed",
              "errors" -> validation.errors
            )))
          } else {
            // Upload to S3
            val key = s3Service.generateKey("images", file.filename)
            val contentType = file.contentType.getOrElse("image/jpeg")

            s3Service.upload(key, file.ref.path, contentType).map { url =>
              Created(Json.obj(
                "url"         -> url,
                "key"         -> key,
                "filename"    -> file.filename,
                "size"        -> file.fileSize,
                "contentType" -> contentType
              ))
            }.recover {
              case ex => InternalServerError(Json.obj("error" -> s"Upload failed: ${ex.getMessage}"))
            }
          }
      }
    }

  // Generate presigned URL (client uploads directly to S3)
  def getPresignedUrl(): Action[JsValue] = Action.async(parse.json) { implicit request =>
    val filename    = (request.body \ "filename").asOpt[String].getOrElse("file")
    val contentType = (request.body \ "contentType").asOpt[String].getOrElse("application/octet-stream")

    // Validate content type
    if (!FileValidator.ImageTypes.contains(contentType) &&
        !FileValidator.DocumentTypes.contains(contentType)) {
      Future.successful(BadRequest(Json.obj("error" -> "Content type not allowed")))
    } else {
      val key = s3Service.generateKey("uploads", filename)

      s3Service.generatePresignedUploadUrl(key, contentType).map { presignedUrl =>
        Ok(Json.obj(
          "uploadUrl"  -> presignedUrl,
          "key"        -> key,
          "expiresIn"  -> 900  // 15 minutes
        ))
      }
    }
  }
}
```

---

## Step 556: Image Processing

```scala
// app/services/ImageService.scala
package services

import javax.inject.*
import java.awt.image.BufferedImage
import javax.imageio.ImageIO
import java.io.{ByteArrayInputStream, ByteArrayOutputStream}
import scala.concurrent.*

@Singleton
class ImageService @Inject()(
  s3Service: S3Service,
  implicit val ec: ExecutionContext
) {

  case class ImageDimensions(width: Int, height: Int)

  // Resize image
  def resize(
    imageBytes: Array[Byte],
    maxWidth: Int,
    maxHeight: Int,
    format: String = "jpg"
  ): Future[Array[Byte]] = Future {
    val original = ImageIO.read(new ByteArrayInputStream(imageBytes))
    val dims = calculateDimensions(
      original.getWidth, original.getHeight,
      maxWidth, maxHeight
    )

    val scaled = original.getScaledInstance(dims.width, dims.height, java.awt.Image.SCALE_SMOOTH)
    val output = new BufferedImage(dims.width, dims.height, BufferedImage.TYPE_INT_RGB)
    output.getGraphics.drawImage(scaled, 0, 0, null)

    val baos = new ByteArrayOutputStream()
    ImageIO.write(output, format, baos)
    baos.toByteArray
  }

  // Generate thumbnails
  def generateThumbnail(
    imageBytes: Array[Byte],
    size: Int = 200
  ): Future[Array[Byte]] = resize(imageBytes, size, size)

  // Upload with multiple sizes
  def uploadWithSizes(
    originalBytes: Array[Byte],
    baseKey: String,
    contentType: String
  ): Future[Map[String, String]] = {
    val sizes = Map(
      "original"  -> (3000, 3000),
      "large"     -> (1200, 800),
      "medium"    -> (600, 400),
      "thumbnail" -> (200, 200)
    )

    val uploadFutures = sizes.map { case (name, (w, h)) =>
      val key = s"${baseKey}_$name"
      resize(originalBytes, w, h).flatMap { resized =>
        s3Service.uploadBytes(key, resized, contentType).map(name -> _)
      }
    }

    Future.sequence(uploadFutures.map { case (name, future) =>
      future.map(url => name -> url)
    }).map(_.toMap)
  }

  private def calculateDimensions(
    originalW: Int, originalH: Int,
    maxW: Int, maxH: Int
  ): ImageDimensions = {
    val ratio = Math.min(maxW.toDouble / originalW, maxH.toDouble / originalH)
    if (ratio >= 1) ImageDimensions(originalW, originalH)
    else ImageDimensions((originalW * ratio).toInt, (originalH * ratio).toInt)
  }
}
```

---

## Step 557: Frontend Upload UI

```html
<!-- app/views/upload.scala.html -->
@()(implicit request: Request[?])
@import helper.CSRF

@layouts.main("Upload File") {
  <div class="container mt-4">
    <h1>อัพโหลดไฟล์</h1>

    <!-- Direct upload form -->
    <div class="card mb-4">
      <div class="card-header">อัพโหลดรูปภาพ</div>
      <div class="card-body">
        <form id="uploadForm" enctype="multipart/form-data">
          @CSRF.formField
          <div class="mb-3">
            <label class="form-label">เลือกรูปภาพ (สูงสุด 5MB)</label>
            <input type="file" class="form-control" id="imageInput" name="image"
                   accept="image/*">
          </div>
          <div id="preview" class="mb-3" style="display:none">
            <img id="previewImg" src="" style="max-width: 300px; max-height: 200px;">
          </div>
          <div class="progress mb-3" style="display:none" id="progressBar">
            <div class="progress-bar" role="progressbar" style="width: 0%"></div>
          </div>
          <button type="submit" class="btn btn-primary">อัพโหลด</button>
        </form>
        <div id="result" class="mt-3"></div>
      </div>
    </div>
  </div>

  <script>
    // Preview image before upload
    document.getElementById('imageInput').addEventListener('change', function(e) {
      const file = e.target.files[0];
      if (file) {
        const reader = new FileReader();
        reader.onload = function(e) {
          document.getElementById('previewImg').src = e.target.result;
          document.getElementById('preview').style.display = 'block';
        };
        reader.readAsDataURL(file);
      }
    });

    // Upload with progress
    document.getElementById('uploadForm').addEventListener('submit', async function(e) {
      e.preventDefault();

      const formData = new FormData(this);
      const progressBar = document.getElementById('progressBar');
      const progress = progressBar.querySelector('.progress-bar');

      progressBar.style.display = 'block';

      try {
        const xhr = new XMLHttpRequest();

        xhr.upload.addEventListener('progress', function(e) {
          if (e.lengthComputable) {
            const percent = Math.round(e.loaded / e.total * 100);
            progress.style.width = percent + '%';
            progress.textContent = percent + '%';
          }
        });

        xhr.addEventListener('load', function() {
          const response = JSON.parse(xhr.responseText);
          if (xhr.status === 201) {
            document.getElementById('result').innerHTML = `
              <div class="alert alert-success">
                อัพโหลดสำเร็จ!<br>
                URL: <a href="${response.url}" target="_blank">${response.url}</a>
              </div>
              <img src="${response.url}" class="img-thumbnail" style="max-width: 300px;">
            `;
          } else {
            document.getElementById('result').innerHTML = `
              <div class="alert alert-danger">
                เกิดข้อผิดพลาด: ${response.error}
              </div>
            `;
          }
          progressBar.style.display = 'none';
        });

        xhr.open('POST', '@routes.S3UploadController.uploadImage()');
        xhr.send(formData);
      } catch (error) {
        document.getElementById('result').innerHTML = `
          <div class="alert alert-danger">Error: ${error.message}</div>
        `;
      }
    });
  </script>
}
```

---

## Step 558-560: File Download และ Management

```scala
// app/controllers/FileManagementController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.libs.json.*
import services.S3Service
import scala.concurrent.*

@Singleton
class FileManagementController @Inject()(
  val controllerComponents: ControllerComponents,
  s3Service: S3Service,
  implicit val ec: ExecutionContext
) extends BaseController {

  // List user files
  def listFiles(userId: Long): Action[AnyContent] = Action.async { implicit request =>
    // Fetch file metadata from DB
    Future.successful(Ok(Json.arr(
      Json.obj("key" -> "images/abc.jpg", "url" -> "https://...", "size" -> 102400)
    )))
  }

  // Delete file
  def deleteFile(): Action[JsValue] = Action.async(parse.json) { implicit request =>
    val key = (request.body \ "key").asOpt[String]

    key match {
      case None =>
        Future.successful(BadRequest(Json.obj("error" -> "Key required")))
      case Some(k) =>
        // ตรวจสอบ ownership ก่อน delete
        s3Service.delete(k).map { _ =>
          Ok(Json.obj("message" -> "File deleted", "key" -> k))
        }.recover {
          case ex => InternalServerError(Json.obj("error" -> ex.getMessage))
        }
    }
  }
}
```

---

## สรุป Part 56

| Aspect | Play Feature | Best Practice |
|--------|-------------|--------------|
| Upload | `parse.multipartFormData` | Validate ทุก file |
| Validation | Magic bytes check | ไม่เชื่อ Content-Type เท่านั้น |
| Storage | S3 / local | ใช้ S3 ใน production |
| Size limit | `maxLength` | ป้องกัน large uploads |
| Security | Filename sanitize | ป้องกัน path traversal |
| Presigned URL | S3 presigner | Client uploads โดยตรง |
| Images | Image resizing | สร้าง thumbnails |

---

## แบบฝึกหัด Part 56

1. **Image Gallery**: สร้าง image gallery ที่ upload รูปภาพไปยัง S3, สร้าง thumbnails, และ แสดงใน paginated grid

2. **Document Upload**: สร้าง document upload system ที่ validate PDF/DOC files, scan สำหรับ malware (mock), และเก็บ metadata ใน database

3. **Presigned Upload**: Implement presigned URL flow ที่ frontend upload โดยตรงไปยัง S3 และ confirm กับ server หลัง upload เสร็จ

4. **Chunked Upload**: Implement resumable upload สำหรับ large files โดยแบ่งเป็น chunks และ reassemble ใน server

5. **Avatar Upload**: สร้าง avatar upload system ที่ crop รูปเป็น square, resize หลาย sizes (32px, 64px, 128px), และ update user profile

---

[→ ไปยัง Part 57: Play i18n](part-57-play-i18n.md)
