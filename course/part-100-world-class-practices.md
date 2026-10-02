# Part 100: World-Class Practices — Steps 991-1000

## บทนำ: World-Class Engineering

Part สุดท้ายของ course รวม practices ที่แยกแยะ engineer ระดับ senior/staff จาก engineer ทั่วไป — code review, ADRs, runbooks, on-call, technical leadership

---

## Step 991: Code Review Excellence

```markdown
# Code Review Guidelines

## As Author

### Before Requesting Review
- [ ] Self-review: read your own diff
- [ ] Tests pass locally
- [ ] No debug code / commented-out code
- [ ] No unintended whitespace changes
- [ ] PR description explains WHAT and WHY (not HOW)
- [ ] Linked to issue/ticket
- [ ] Size: ideally < 400 lines changed

### PR Description Template
```
## What
Brief description of changes.

## Why
The problem this solves / the requirement it fulfills.

## How
Key technical decisions made. Alternatives considered.

## Testing
How you tested this. Describe edge cases covered.

## Screenshots/Benchmarks (if applicable)
[Before/After metrics]
```

## As Reviewer

### What to Look For
1. **Correctness**: Does it do what it claims?
2. **Edge cases**: Empty list? Null? Concurrent access?
3. **Error handling**: All errors handled? Correct status codes?
4. **Performance**: Any O(n²) hiding in loops?
5. **Security**: Any injection vulnerabilities? Sensitive data logged?
6. **Tests**: Good coverage? Tests meaningful behavior?
7. **Naming**: Clear, consistent with codebase?
8. **Complexity**: Can this be simpler?

### Comment Levels
- **nit:** Minor style, take or leave
- **suggestion:** Would be better this way (non-blocking)
- **concern:** Please address before merging (blocking)
- **question:** I don't understand — explain or change

### Golden Rule
"Critique the code, not the person."
Say: "This function could cause issues when X"
Not: "You made a mistake here"
```

```scala
// ReviewExamples.scala — Common review points

// ===== Review Point 1: Missing error case =====
// ❌ Before (will throw on empty list)
def getFirstItem(items: List[String]): String = items.head

// ✅ After (explicit handling)
def getFirstItem(items: List[String]): Option[String] = items.headOption

// ===== Review Point 2: N+1 query =====
// ❌ Before (N queries in a loop)
def loadOrdersWithUsers(orderIds: List[Long]): List[(Order, User)] = {
  orderIds.map { id =>
    val order = db.findOrder(id)        // 1 query
    val user  = db.findUser(order.userId)  // N queries!
    (order, user)
  }
}

// ✅ After (2 queries total)
def loadOrdersWithUsersBatch(orderIds: List[Long]): List[(Order, User)] = {
  val orders  = db.findOrders(orderIds)                    // 1 query
  val userIds = orders.map(_.userId).distinct
  val users   = db.findUsers(userIds).map(u => u.id -> u).toMap  // 1 query
  orders.map(o => (o, users(o.userId)))
}

case class Order(id: Long, userId: Long, total: Double)
case class User(id: Long, name: String)
object db {
  def findOrder(id: Long): Order = Order(id, 1L, 99.99)
  def findOrders(ids: List[Long]): List[Order] = ids.map(id => Order(id, 1L, 99.99))
  def findUser(id: Long): User = User(id, "Alice")
  def findUsers(ids: List[Long]): List[User] = ids.map(id => User(id, "Alice"))
}
```

---

## Step 992: Architecture Decision Records

```markdown
# ADR-0042: Use Delta Lake for Data Pipeline Storage

**Date**: 2024-03-15
**Status**: Accepted
**Deciders**: @alice, @bob, @charlie

## Context
Our data pipeline needs a storage format that supports:
- ACID transactions for concurrent writes
- Schema evolution as events change
- Time travel for debugging and reprocessing
- Efficient upsert operations

We evaluated: Parquet (raw), Hudi, Iceberg, Delta Lake.

## Decision
Use **Delta Lake** for all pipeline storage layers.

## Rationale

| Criterion | Parquet | Hudi | Iceberg | Delta Lake |
|-----------|---------|------|---------|------------|
| Spark integration | Good | Good | Good | Excellent |
| ACID transactions | No | Yes | Yes | Yes |
| Time travel | No | Yes | Yes | Yes |
| Schema evolution | Limited | Yes | Yes | Yes |
| Community (2024) | Large | Medium | Growing | Large |
| Databricks support | N/A | N/A | N/A | Native |
| Team familiarity | High | None | Low | Medium |

Delta Lake wins on Spark integration depth and team learning curve.

## Consequences

**Positive:**
- Full ACID on S3 — safe concurrent writes
- Easy schema evolution without reprocessing
- Time travel simplifies debugging

**Negative:**
- Delta log files add small storage overhead (~1%)
- Requires `OPTIMIZE` / `VACUUM` jobs
- Slightly more complex than raw Parquet

## Implementation Notes
- `delta.autoOptimize.optimizeWrite = true` for all tables
- VACUUM retention: 7 days (for time travel debugging)
- Z-ORDER by most common filter columns

## References
- [Delta Lake vs Iceberg vs Hudi (2024)](https://...)
- [Our pipeline design doc](https://...)
```

---

## Step 993: Runbook Template

```markdown
# Runbook: Order Service — High Error Rate

**Service**: order-service
**Severity**: P1 (> 5% error rate), P2 (1-5% error rate)
**Owner**: Platform Team
**Last Updated**: 2024-03-15

## Quick Reference

| Alert | Threshold | Action |
|-------|-----------|--------|
| error_rate > 5% | P1 | Page on-call |
| error_rate 1-5% | P2 | Notify Slack |
| p99 latency > 2s | P2 | Investigate |
| pod restarts > 3 | P2 | Check logs |

## Symptoms
- Grafana alert: `order_service_error_rate > 0.05`
- Customers report orders failing
- Slack: #alerts-production notification

## Immediate Actions (< 5 minutes)

1. **Acknowledge alert** in PagerDuty
2. **Check service status**:
   ```bash
   kubectl get pods -n production -l app=order-service
   kubectl top pods -n production -l app=order-service
   ```
3. **Check recent deployments**:
   ```bash
   kubectl rollout history deployment/order-service -n production
   ```
4. **Check error logs**:
   ```bash
   kubectl logs -n production -l app=order-service --tail=100 | grep ERROR
   ```

## Diagnosis

### Scenario A: Recent Deployment (most common)
```bash
# Roll back to previous version
kubectl rollout undo deployment/order-service -n production
kubectl rollout status deployment/order-service -n production
```

### Scenario B: Database Issues
```bash
# Check DB connection pool
kubectl exec -n production deploy/order-service -- curl localhost:8080/metrics | grep "hikaricp"

# Check PostgreSQL
kubectl exec -n production postgres-0 -- psql -U postgres -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"

# Check for long-running queries
kubectl exec -n production postgres-0 -- psql -U postgres -c "SELECT pid, query, duration FROM pg_stat_activity WHERE state != 'idle' AND duration > interval '30 seconds';"
```

### Scenario C: Dependency Failure (Kafka/Redis)
```bash
# Check Kafka lag
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group order-service --describe

# Check Redis
kubectl exec -n production redis-0 -- redis-cli ping
```

### Scenario D: Memory/CPU issues
```bash
# Check resource usage
kubectl top pods -n production -l app=order-service

# Trigger heap dump if OOM
kubectl exec -n production <pod-name> -- kill -3 1  # SIGQUIT = thread dump
kubectl exec -n production <pod-name> -- jcmd 1 VM.heap_dump /tmp/heap.hprof
```

## Recovery Actions

### Scale up immediately
```bash
kubectl scale deployment order-service --replicas=10 -n production
```

### Enable circuit breaker / maintenance mode
```bash
kubectl set env deployment/order-service MAINTENANCE_MODE=true -n production
```

## Post-Incident
1. Write incident report (5 whys)
2. Create Jira ticket for root cause
3. Update this runbook if needed
4. Schedule blameless post-mortem within 48h

## Escalation Path
- L1 (on-call): Follow this runbook
- L2 (tech lead): Call if not resolved in 30 min
- L3 (engineering manager): Call if customer impact > 1 hour
```

---

## Step 994: On-Call Practices

```scala
// OnCallPractices.scala

/*
===== On-Call Best Practices =====

1. ALERT DESIGN
   - Alert on symptoms, not causes (high error rate, not "DB slow")
   - Alert must be actionable (every alert should have a runbook)
   - Reduce alert noise (fewer, higher-quality alerts)
   - Use multi-window burndown for SLO alerts

2. INCIDENT RESPONSE
   Phase 1: Detect     → < 5 min (from production to awareness)
   Phase 2: Respond    → < 15 min (on-call acknowledges)
   Phase 3: Mitigate   → ASAP (reduce customer impact)
   Phase 4: Resolve    → hours/days (fix root cause)
   Phase 5: Learn      → post-mortem within 48h

3. COMMUNICATION
   - Incident commander: one person coordinates
   - Status page updates every 30 min
   - Internal Slack: #incidents-live
   - Customer: status.mycompany.com

4. TOIL REDUCTION
   - Automate manual investigation steps
   - Fix recurring alerts permanently
   - Blameless post-mortems → no fear of sharing
*/

// ===== Alert configuration example =====
object AlertConfig {
  
  // Multi-burn rate alert for SLO
  val sloAlerts = Map(
    "page" -> Map(
      "window" -> "1h",
      "burnRate" -> 14.4,  // burn budget at 14.4x rate → 1h to exhaust 1% budget
      "severity" -> "P1"
    ),
    "ticket" -> Map(
      "window" -> "6h",
      "burnRate" -> 6,
      "severity" -> "P2"
    )
  )
  
  // PagerDuty severity mapping
  val severityMapping = Map(
    "P1" -> "critical",   // Immediate page, 5-min response
    "P2" -> "warning",    // Page if no response in 30 min
    "P3" -> "info"        // Slack notification only
  )
}
```

---

## Step 995: Technical Leadership

```markdown
# Technical Leadership Skills

## 1. RFC Process (Request for Comments)

Before starting significant work, write an RFC:
- Problem statement
- Proposed solution
- Alternative approaches
- Risks and trade-offs
- Success metrics

This prevents building the wrong thing.

## 2. Tech Debt Management

Categorize tech debt:
- **Critical**: Security vulnerabilities, outages imminent
- **High**: Slows feature development, significant risk  
- **Medium**: Annoying, slows debugging
- **Low**: Nice to have, cosmetic

Reserve 20% of sprint capacity for tech debt.

## 3. Engineering Principles

Define and document your team's principles:
- "Optimize for readability, not cleverness"
- "Default to async — sync only when necessary"
- "Every service must have a runbook"
- "Alert only on actionable conditions"
- "Measure before optimizing"

## 4. Mentoring

Senior engineers multiply impact through others:
- Regular 1:1s with mentees
- Pair programming on complex problems
- Design reviews before implementation
- Non-judgmental code reviews
```

```scala
// TechnicalLeadership.scala

object TechLeadPatterns {
  
  // ===== Measuring engineering productivity =====
  
  // DORA metrics
  case class DORAMetrics(
    deploymentFrequency: Double,     // deploys per day
    leadTimeForChanges: Double,      // hours from commit to production
    changeFailureRate: Double,       // % of deploys causing incidents
    meanTimeToRestore: Double        // hours to recover from incident
  )
  
  // Elite performers:
  val eliteTarget = DORAMetrics(
    deploymentFrequency  = 1.0,    // On-demand (multiple per day)
    leadTimeForChanges   = 24.0,   // Less than one day
    changeFailureRate    = 0.05,   // < 5%
    meanTimeToRestore    = 1.0     // < 1 hour
  )
  
  def assessTeam(metrics: DORAMetrics): String = {
    val scores = List(
      if (metrics.deploymentFrequency >= 1.0) 3 else if (metrics.deploymentFrequency >= 0.5) 2 else 1,
      if (metrics.leadTimeForChanges <= 24) 3 else if (metrics.leadTimeForChanges <= 168) 2 else 1,
      if (metrics.changeFailureRate <= 0.05) 3 else if (metrics.changeFailureRate <= 0.15) 2 else 1,
      if (metrics.meanTimeToRestore <= 1.0) 3 else if (metrics.meanTimeToRestore <= 24) 2 else 1
    )
    val avg = scores.sum.toDouble / scores.size
    
    if (avg >= 2.5) "Elite" else if (avg >= 2.0) "High" else if (avg >= 1.5) "Medium" else "Low"
  }
}
```

---

## Step 996: Production Readiness Checklist

```markdown
# Production Readiness Review Checklist

## Service Basics
- [ ] Service has a README with architecture diagram
- [ ] All configuration via environment variables
- [ ] Graceful shutdown (handle SIGTERM)
- [ ] Health check endpoint `/health`
- [ ] Readiness probe (separate from liveness)
- [ ] Service SLA defined (e.g., 99.9% uptime)

## Reliability
- [ ] Circuit breakers for all external calls
- [ ] Retry with exponential backoff + jitter
- [ ] Timeouts on all network calls
- [ ] Bulkhead pattern for resource isolation
- [ ] Chaos engineering tested (Chaos Monkey)
- [ ] Load tested to 2x peak traffic

## Observability
- [ ] Structured JSON logging
- [ ] Request ID propagated through all services
- [ ] Metrics exported to Prometheus (RED metrics)
- [ ] Distributed tracing (OpenTelemetry)
- [ ] Alerting for SLO burn rate
- [ ] Dashboard in Grafana

## Security
- [ ] No secrets in code or config files
- [ ] All endpoints authenticated
- [ ] Input validation on all user inputs
- [ ] Dependencies scanned for CVEs
- [ ] TLS everywhere (service mesh mTLS)
- [ ] PII data encrypted at rest

## Operations
- [ ] Runbook written and accessible
- [ ] Deployment process documented
- [ ] Rollback process tested
- [ ] Database migrations backwards compatible
- [ ] On-call rotation established
- [ ] Post-mortem process defined

## Data
- [ ] Data retention policy defined
- [ ] Backup tested (last restore tested)
- [ ] Data classified (PII, sensitive, public)
- [ ] GDPR deletion capability (if applicable)
```

---

## Step 997: Incident Post-Mortem

```markdown
# Post-Mortem: Order Service Outage — 2024-03-15

**Severity**: P1
**Duration**: 47 minutes (14:23 - 15:10 UTC)
**Impact**: 12% of order creation requests failed
**Customer Impact**: ~847 failed orders, $24k revenue lost

## Timeline

| Time  | Event |
|-------|-------|
| 14:23 | Alert triggered: error_rate > 5% |
| 14:28 | On-call acknowledged |
| 14:35 | Identified root cause: DB connection pool exhausted |
| 14:52 | Mitigation: scaled up pods from 3 to 10 |
| 15:05 | Error rate returned to normal |
| 15:10 | Incident closed |

## Root Cause

Database connection pool (HikariCP max=20) was exhausted due to:
1. Traffic spike (3x normal) from marketing campaign
2. Slow queries caused by missing index on `orders.customer_id`
3. Pool timeout caused cascading 500 errors

## 5 Whys

1. Why did orders fail? → DB connection pool exhausted
2. Why was pool exhausted? → Queries too slow (avg 2s instead of 50ms)
3. Why were queries slow? → Missing index on `customer_id`
4. Why was index missing? → Index not added when feature shipped
5. Why not caught in review? → No query performance check in PR process

## Action Items

| Action | Owner | Due |
|--------|-------|-----|
| Add index on orders.customer_id | @alice | 2024-03-16 |
| Add query performance check to PR checklist | @bob | 2024-03-22 |
| Set up automated slow query detection | @charlie | 2024-03-29 |
| Increase pool size (20 → 50) + tune HikariCP | @alice | 2024-03-17 |
| Add auto-scaling based on DB query time | @dave | 2024-04-01 |

## What Went Well
- Alert fired quickly (< 2 min from symptom)
- On-call responded within 5 minutes
- Good runbook — mitigation was fast

## What Could Be Better
- No pre-defined DB runbook for pool exhaustion
- Marketing campaign not communicated to platform team

## Blameless Note
This outage was caused by a systems failure (missing index), not by individual error. Everyone involved acted in good faith. The goal is to improve the system, not assign blame.
```

---

## Step 998: Documentation Culture

```scala
// DocumentationCulture.scala

/*
===== Types of Documentation =====

1. API Documentation (Scaladoc)
   - Every public method
   - @param, @return, @throws
   - @example with working code
   
2. Architecture Documentation (ADRs)
   - Why was this decision made?
   - What alternatives were considered?
   - What are the trade-offs?

3. Operational Documentation (Runbooks)
   - How to deploy
   - How to debug
   - How to respond to incidents

4. README
   - What does this do?
   - How to get started (< 5 minutes to first success)
   - How to contribute

5. Comments in Code
   - WHY not WHAT (code shows WHAT)
   - Non-obvious decisions
   - Links to issues, Stack Overflow answers

===== Documentation Anti-patterns =====

❌ Outdated documentation (worse than no docs)
❌ Documentation that just repeats code
❌ "It's obvious" — nothing is obvious to a newcomer
❌ Only in one person's head
❌ No examples

===== Good Comment Examples =====
*/

object GoodComments {
  
  // ❌ Bad: restates the code
  // Increment i by 1
  var i = 0
  i += 1
  
  // ✅ Good: explains non-obvious choice
  // We use Int.MaxValue/2 (not Int.MaxValue) to avoid overflow
  // when comparing sorted values. See issue #342.
  val SENTINEL_VALUE = Int.MaxValue / 2
  
  // ✅ Good: explains business rule
  // Orders from VIP customers (tier >= 3) skip the fraud check.
  // Product decision made 2023-11-01, see Jira PROD-445.
  def skipFraudCheck(customerTier: Int): Boolean = customerTier >= 3
  
  // ✅ Good: warns about a gotcha
  // Note: This method is NOT thread-safe. All callers must
  // synchronize externally. See ThreadingModel.md.
  def updateSharedState(value: Int): Unit = ???
}
```

---

## Step 999: Career as a World-Class Engineer

```markdown
# World-Class Scala Engineer — Career Path

## Junior (0-2 years)
- Write code that works
- Learn the language deeply
- Understand testing
- Learn git workflow
- Contribute to team PRs

## Mid-Level (2-5 years)
- Design small systems
- Mentor juniors
- Own features end-to-end
- Performance optimization
- On-call rotation

## Senior (5+ years)
- Design large systems
- Define team standards
- Technical leadership on projects
- Cross-team influence
- Identify tech debt strategically

## Staff/Principal (8+ years)
- Architecture across multiple teams
- Define multi-year technical strategy
- Industry influence (talks, papers, OSS)
- Force multiplier (make others 10x better)

## Continuous Learning (every level)
- Read papers (VLDB, OSDI, SOSP)
- Study successful open source projects
- Build side projects to experiment
- Teach others (blog, talks, mentoring)
- Stay curious

## Scala-Specific Excellence

### Deep Understanding
- How the type system works (type inference, implicits, variance)
- JVM bytecode and optimization
- Concurrency models (thread-based, actor, fiber)
- Effect systems (IO, ZIO, Future trade-offs)
- Category theory basics (functor, monad, applicative)

### Ecosystem Mastery
- Cats Effect / ZIO for production code
- Spark for data processing
- Akka for high-throughput services
- Play/http4s for HTTP services
- Doobie/Quill for database access

### Production Skills
- Profiling (Async Profiler, JFR)
- Observability (metrics, traces, logs)
- Kubernetes deployment
- Performance tuning
- Security hardening
```

---

## Step 1000: Congratulations!

```scala
// Congratulations.scala — You've completed the Scala Course!

/**
 * ================================
 *  Scala World-Class Engineer
 *  Certificate of Completion
 *  Parts 1-100
 * ================================
 */
object WorldClassEngineer {
  
  val completedTopics = Map(
    "Phase 1-2: Foundations"         -> "Scala basics, OOP, FP fundamentals",
    "Phase 3-4: Advanced Scala"      -> "Type system, implicits, type classes",
    "Phase 5-6: Akka & Play"         -> "Actors, HTTP services, WebSockets",
    "Phase 7: Databases & APIs"      -> "Doobie, REST, GraphQL",
    "Phase 8: Big Data"              -> "Spark 3.5.x, Kafka 3.6.x, Delta Lake",
    "Phase 9: Microservices & Cloud" -> "Lagom, Docker, K8s, gRPC, AWS, CI/CD",
    "Phase 10: Production"           -> "Performance, Security, Architecture, OSS"
  )
  
  val skillsAcquired = List(
    "Build production Spark streaming pipelines",
    "Design and implement microservices",
    "Deploy to Kubernetes with full observability",
    "Write secure, tested, documented code",
    "Tune JVM for production workloads",
    "Contribute to Scala OSS ecosystem",
    "Conduct effective code reviews",
    "Write ADRs and runbooks",
    "Respond to production incidents"
  )
  
  def printCertificate(name: String): Unit = {
    println("=" * 60)
    println(s"  SCALA WORLD-CLASS ENGINEER")
    println(s"  Awarded to: $name")
    println(s"  Completed: ${java.time.LocalDate.now()}")
    println("=" * 60)
    println("\n  Topics Mastered:")
    completedTopics.foreach { case (phase, desc) =>
      println(s"  ✓ $phase")
      println(s"    $desc")
    }
    println("\n  Skills Acquired:")
    skillsAcquired.foreach(skill => println(s"  ★ $skill"))
    println("\n  Next Steps:")
    println("  → Contribute to Scala OSS (cats, zio, akka)")
    println("  → Build and publish your own library")
    println("  → Speak at Scala Days or local meetup")
    println("  → Mentor others")
    println("=" * 60)
  }
  
  def main(args: Array[String]): Unit = {
    val name = args.headOption.getOrElse("Scala Developer")
    printCertificate(name)
  }
}
```

---

## สรุป Part 100: World-Class Practices

| Practice | Junior | Senior | Staff |
|----------|--------|--------|-------|
| Code Review | Receive | Give constructively | Define standards |
| Documentation | Write inline comments | Write ADRs | Define culture |
| On-Call | Shadow | Participate | Design runbooks |
| Architecture | Implement | Design components | Define strategy |
| Mentoring | N/A | 1-2 mentees | Team/organization |

---

## แบบฝึกหัด Part 100

1. **ADR**: เขียน ADR สำหรับ technical decision ที่ team ของคุณกำลัง consider เช่น "Should we use ZIO or Cats Effect?" หรือ "Should we adopt Kubernetes?"

2. **Runbook**: เขียน runbook สำหรับ service ที่คุณ own ที่ cover: top 3 alert scenarios, diagnosis steps, mitigation actions, escalation path

3. **Post-Mortem**: simulate post-mortem สำหรับ "3-hour database outage" — write root cause analysis, timeline, 5 whys, action items

4. **Production Readiness**: ทำ Production Readiness Review สำหรับ service ที่คุณสร้างใน Part 97 — identify gaps และ create action plan

5. **Capstone**: เลือก 1 topic จาก course นี้ ที่คุณอยากพัฒนาเพิ่มเติม เขียน learning plan 3 เดือน พร้อม milestones และ resources

---

## ขอแสดงความยินดี!

คุณได้เรียน **Scala 100 Parts** ครบถ้วนแล้ว:

- **Parts 1-70**: Scala foundations, OOP, FP, Akka, Play, Databases
- **Parts 71-80**: Spark, Kafka, Delta Lake, Data Pipelines
- **Parts 81-90**: Microservices, Lagom, Docker, K8s, gRPC, Service Mesh, AWS, Observability, CI/CD, API Gateway
- **Parts 91-100**: Performance, Profiling, Functional Architecture, Cats Effect/ZIO, Security, Scalability, Real-world Projects, OSS, World-Class Practices

**You are now a World-Class Scala Engineer.**

จาก `Hello World` สู่ Production-grade distributed systems ที่ handle millions of requests — ขอให้ใช้ความรู้นี้สร้าง software ที่ยอดเยี่ยมและส่งผลดีต่อโลก!

---

*"Programs must be written for people to read, and only incidentally for machines to execute."*
— Harold Abelson
