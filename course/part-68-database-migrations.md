# Part 68: Database Migrations with Flyway

## Steps 671-680: Flyway Setup, Versioned Migrations, Undo, CI Integration

---

## Step 671: Database Migration คืออะไร

Database Migration คือ process จัดการการเปลี่ยนแปลง database schema อย่างมีระเบียบ

```
ทำไมต้องใช้ Migration tool:
✗ แก้ SQL ตรง production → ไม่ track ว่าแก้อะไรไปแล้ว
✗ มี schema ต่างกันระหว่าง dev/staging/production
✗ ไม่รู้ว่า migration ไหนรัน/ไม่ได้รัน

✓ Flyway:
  ├── Track migration ที่รันแล้วใน flyway_schema_history
  ├── Versioned migrations (V1, V2, V3...)
  ├── Repeatable migrations (R__)
  ├── Undo migrations (U__)
  └── Validate checksums
```

### build.sbt

```scala
libraryDependencies ++= Seq(
  "org.flywaydb" % "flyway-core"              % "10.13.0",
  "org.flywaydb" % "flyway-database-postgresql" % "10.13.0",
  // SBT Plugin
)

// plugins.sbt
addSbtPlugin("io.github.davidmweber" % "flyway-sbt" % "7.4.0")
```

---

## Step 672: Migration Files Structure

```
db/migration/
├── V1__create_users_table.sql
├── V2__create_articles_table.sql
├── V3__create_comments_table.sql
├── V4__add_tags_to_articles.sql
├── V5__create_indexes.sql
├── V6__add_user_profiles.sql
├── V7__create_article_tags_table.sql
├── R__refresh_views.sql          ← Repeatable (re-run when changed)
└── V8__add_full_text_search.sql
```

---

## Step 673: Versioned Migrations

```sql
-- db/migration/V1__create_users_table.sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE TABLE users (
  id            TEXT        PRIMARY KEY DEFAULT gen_random_uuid()::text,
  name          VARCHAR(255) NOT NULL,
  email         VARCHAR(255) NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  role          VARCHAR(50)  NOT NULL DEFAULT 'user',
  is_active     BOOLEAN      NOT NULL DEFAULT true,
  created_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX users_email_idx ON users(email);

COMMENT ON TABLE users IS 'Application users';
COMMENT ON COLUMN users.role IS 'User role: admin, editor, user';
```

```sql
-- db/migration/V2__create_articles_table.sql
CREATE TABLE articles (
  id           TEXT         PRIMARY KEY DEFAULT gen_random_uuid()::text,
  title        VARCHAR(500) NOT NULL,
  slug         VARCHAR(500) NOT NULL UNIQUE,
  content      TEXT         NOT NULL,
  summary      TEXT,
  author_id    TEXT         NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
  status       VARCHAR(50)  NOT NULL DEFAULT 'draft',
  view_count   INTEGER      NOT NULL DEFAULT 0,
  published_at TIMESTAMPTZ,
  created_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX articles_author_idx       ON articles(author_id);
CREATE INDEX articles_status_idx       ON articles(status);
CREATE INDEX articles_slug_idx         ON articles(slug);
CREATE INDEX articles_published_at_idx ON articles(published_at DESC NULLS LAST)
  WHERE status = 'published';
```

```sql
-- db/migration/V3__create_comments_table.sql
CREATE TABLE comments (
  id         TEXT        PRIMARY KEY DEFAULT gen_random_uuid()::text,
  article_id TEXT        NOT NULL REFERENCES articles(id) ON DELETE CASCADE,
  author_id  TEXT        NOT NULL REFERENCES users(id)    ON DELETE RESTRICT,
  content    TEXT        NOT NULL,
  parent_id  TEXT        REFERENCES comments(id) ON DELETE CASCADE,
  is_deleted BOOLEAN     NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX comments_article_idx ON comments(article_id);
CREATE INDEX comments_author_idx  ON comments(author_id);
CREATE INDEX comments_parent_idx  ON comments(parent_id);
```

---

## Step 674: Additive Migrations

```sql
-- db/migration/V4__add_tags_support.sql
-- Migration ที่ ADD feature (Additive — safe)

CREATE TABLE tags (
  id         TEXT        PRIMARY KEY DEFAULT gen_random_uuid()::text,
  name       VARCHAR(100) NOT NULL,
  slug       VARCHAR(100) NOT NULL UNIQUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE article_tags (
  article_id TEXT NOT NULL REFERENCES articles(id) ON DELETE CASCADE,
  tag_id     TEXT NOT NULL REFERENCES tags(id)     ON DELETE CASCADE,
  PRIMARY KEY (article_id, tag_id)
);

CREATE INDEX article_tags_tag_idx ON article_tags(tag_id);

-- Seed some default tags
INSERT INTO tags (name, slug) VALUES
  ('Scala',       'scala'),
  ('Play',        'play'),
  ('Functional',  'functional'),
  ('Backend',     'backend'),
  ('Database',    'database');
```

---

## Step 675: Destructive Migrations (Careful!)

```sql
-- db/migration/V5__rename_column.sql
-- การ rename column ต้องระวัง! 
-- Pattern: Add new → Migrate data → Drop old

-- Step 1: Add new column
ALTER TABLE articles ADD COLUMN content_html TEXT;

-- Step 2: Migrate data (convert markdown to HTML — ทำใน code แล้วค่อย update)
-- UPDATE articles SET content_html = content;  -- ถ้า simple copy

-- Step 3: Add NOT NULL constraint (ทำหลังจาก data migrate ครบ)
-- ALTER TABLE articles ALTER COLUMN content_html SET NOT NULL;

-- Step 4: Drop old column (migration ต่อไป หลังจาก deploy ใหม่แล้ว)
-- ALTER TABLE articles DROP COLUMN content;
```

```sql
-- db/migration/V6__add_user_preferences.sql
-- เพิ่ม JSONB column สำหรับ flexible data

ALTER TABLE users
  ADD COLUMN preferences  JSONB NOT NULL DEFAULT '{}',
  ADD COLUMN last_login_at TIMESTAMPTZ,
  ADD COLUMN login_count  INTEGER NOT NULL DEFAULT 0;

-- GIN index สำหรับ JSONB queries
CREATE INDEX users_preferences_gin ON users USING gin(preferences);

-- Update timestamp trigger
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER articles_updated_at
  BEFORE UPDATE ON articles
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

---

## Step 676: Full-Text Search Migration

```sql
-- db/migration/V7__add_full_text_search.sql

-- Add tsvector column สำหรับ full-text search
ALTER TABLE articles ADD COLUMN search_vector TSVECTOR;

-- Generate search vector from title + content
UPDATE articles SET search_vector =
  setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
  setweight(to_tsvector('english', COALESCE(summary, '')), 'B') ||
  setweight(to_tsvector('english', COALESCE(content, '')), 'C');

-- GIN index สำหรับ fast full-text search
CREATE INDEX articles_search_vector_idx ON articles USING gin(search_vector);

-- Auto-update trigger
CREATE OR REPLACE FUNCTION articles_search_vector_trigger()
RETURNS TRIGGER AS $$
BEGIN
  NEW.search_vector :=
    setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(NEW.summary, '')), 'B') ||
    setweight(to_tsvector('english', COALESCE(NEW.content, '')), 'C');
  RETURN NEW;
END
$$ LANGUAGE plpgsql;

CREATE TRIGGER articles_search_vector_update
  BEFORE INSERT OR UPDATE ON articles
  FOR EACH ROW EXECUTE FUNCTION articles_search_vector_trigger();
```

---

## Step 677: Repeatable Migrations

```sql
-- db/migration/R__refresh_materialized_views.sql
-- Repeatable: รันทุกครั้งที่ checksum เปลี่ยน

-- Article statistics materialized view
CREATE MATERIALIZED VIEW IF NOT EXISTS article_stats AS
SELECT
  a.id,
  a.title,
  a.author_id,
  u.name           AS author_name,
  a.status,
  a.view_count,
  a.published_at,
  COUNT(c.id)      AS comment_count,
  COUNT(DISTINCT at.tag_id) AS tag_count
FROM articles a
JOIN users u ON a.author_id = u.id
LEFT JOIN comments c ON a.id = c.article_id AND NOT c.is_deleted
LEFT JOIN article_tags at ON a.id = at.article_id
GROUP BY a.id, u.name;

CREATE UNIQUE INDEX IF NOT EXISTS article_stats_id_idx ON article_stats(id);

-- Refresh materialized view (schedule นี้ด้วย cron/pg_cron)
-- SELECT cron.schedule('refresh_article_stats', '*/15 * * * *', 'REFRESH MATERIALIZED VIEW CONCURRENTLY article_stats');
```

---

## Step 678: Flyway Configuration

```hocon
# conf/application.conf
db.default {
  driver   = org.postgresql.Driver
  url      = "jdbc:postgresql://localhost:5432/myapp"
  username = "myapp"
  password = "secret"
}

# Flyway config
flyway {
  locations          = ["classpath:db/migration"]
  baselineOnMigrate  = true
  validateOnMigrate  = true
  outOfOrder         = false
  cleanDisabled      = true    # Never clean production!
  
  # Placeholders
  placeholders {
    schema = "public"
    appUser = "myapp"
  }
}
```

```scala
// app/db/FlywayModule.scala — Run migrations on startup
package db

import javax.inject.*
import org.flywaydb.core.Flyway
import play.api.db.Database
import play.api.inject.ApplicationLifecycle
import play.api.{Configuration, Logger}
import scala.concurrent.Future

@Singleton
class FlywayMigration @Inject()(
  db: Database,
  config: Configuration,
  lifecycle: ApplicationLifecycle
) {
  private val logger = Logger(getClass)

  // Run migrations on startup
  run()

  private def run(): Unit = {
    logger.info("Running Flyway migrations...")

    val flyway = Flyway.configure()
      .dataSource(db.url, db.driverClass, "")
      .locations("classpath:db/migration")
      .baselineOnMigrate(true)
      .validateOnMigrate(true)
      .load()

    val result = flyway.migrate()
    logger.info(s"Applied ${result.migrationsExecuted} migration(s)")
  }
}
```

---

## Step 679: CI/CD Integration

```yaml
# .github/workflows/migration.yml
name: Database Migration

on:
  push:
    paths:
      - 'db/migration/**'
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: myapp_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
    - uses: actions/checkout@v4

    - name: Validate migrations
      uses: joshuaavalon/flyway-action@v3.0.0
      with:
        url: jdbc:postgresql://localhost:5432/myapp_test
        user: test
        password: test
        locations: filesystem:db/migration
        command: validate

    - name: Run migrations
      uses: joshuaavalon/flyway-action@v3.0.0
      with:
        url: jdbc:postgresql://localhost:5432/myapp_test
        user: test
        password: test
        locations: filesystem:db/migration
        command: migrate

    - name: Check migration info
      uses: joshuaavalon/flyway-action@v3.0.0
      with:
        url: jdbc:postgresql://localhost:5432/myapp_test
        user: test
        password: test
        command: info
```

---

## Step 680: Best Practices

```sql
-- ✅ Migration Best Practices

-- 1. Always additive in first deploy
-- Add column (nullable or with default)
ALTER TABLE articles ADD COLUMN featured BOOLEAN NOT NULL DEFAULT false;
-- ❌ ไม่ add column NOT NULL ที่ไม่มี default ตรง production (จะ error!)

-- 2. Multi-step destructive changes
-- Step A (Deploy 1): เพิ่ม column ใหม่
ALTER TABLE articles ADD COLUMN content_v2 TEXT;

-- Step B (Deploy 2): Application ใช้ column ใหม่แล้ว
-- ลบ column เก่า
ALTER TABLE articles DROP COLUMN content;

-- 3. Idempotent migrations
CREATE TABLE IF NOT EXISTS my_table (...);
CREATE INDEX IF NOT EXISTS my_idx ON my_table(col);

-- 4. Data migrations แยกจาก schema migrations
-- V8__add_featured_column.sql (schema)
-- V9__migrate_featured_data.sql (data)
UPDATE articles SET featured = true WHERE view_count > 10000;

-- 5. ไม่แก้ migrations ที่ deploy ไปแล้ว!
-- Flyway จะ detect checksum mismatch และ fail
-- ให้สร้าง migration ใหม่แทน
```

---

## สรุป Part 68

| Concept | Flyway Feature | ตัวอย่าง |
|---------|---------------|---------|
| Version migrations | `V{n}__{desc}.sql` | `V1__create_users.sql` |
| Repeatable | `R__{desc}.sql` | Refresh views |
| Baseline | `baselineOnMigrate` | Existing DB |
| Validate | `validateOnMigrate` | Checksum check |
| Out of order | `outOfOrder` | Team parallel work |
| Placeholders | `${placeholder}` | Environment values |
| History table | `flyway_schema_history` | Auto-created |
| Info | `flyway info` | Migration status |
| Repair | `flyway repair` | Fix failed migration |
| Clean | `flyway clean` | ⚠️ Dev only! |

---

## แบบฝึกหัด Part 68

1. **Migration Strategy**: ออกแบบ migration plan สำหรับ rename 3 columns ใน production table ที่มี data อยู่ โดยไม่ downtime

2. **Multi-Environment**: Setup Flyway ที่ใช้ migration set เดียวกันสำหรับ dev/staging/production แต่มี environment-specific seed data

3. **Rollback Plan**: สร้าง migration strategy ที่ include rollback plan สำหรับทุก migration ด้วย undo scripts

4. **CI Validation**: Setup CI pipeline ที่ validate migrations ด้วย test database ก่อน merge แล้ว auto-apply ใน staging หลัง merge

5. **Large Table Migration**: ออกแบบ zero-downtime migration สำหรับ table ที่มี 100 ล้าน rows ด้วย batched updates และ background processing

---

[→ ไปยัง Part 69: CQRS and Event Sourcing](part-69-cqrs-and-event-sourcing.md)
