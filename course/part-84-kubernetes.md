# Part 84: Kubernetes — Steps 831-840

## บทนำ: Kubernetes สำหรับ Scala Microservices

Kubernetes (K8s) เป็น container orchestration platform ที่ช่วย deploy, scale, และ manage containerized applications ในระดับ production

---

## Step 831: Kubernetes Basics

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: scala-app
  labels:
    app.kubernetes.io/managed-by: kubectl
    environment: production
---
# deployment.yaml — Basic Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: scala-app
  labels:
    app: order-service
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  
  template:
    metadata:
      labels:
        app: order-service
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9095"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: order-service
      
      # Security context สำหรับ pod
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
      
      containers:
        - name: order-service
          image: registry.mycompany.com/order-service:1.0.0
          imagePullPolicy: Always
          
          ports:
            - name: http
              containerPort: 8080
            - name: metrics
              containerPort: 9095
          
          # Environment variables
          env:
            - name: APP_ENV
              value: "production"
            - name: SERVER_PORT
              value: "8080"
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          
          # From ConfigMap
          envFrom:
            - configMapRef:
                name: order-service-config
          
          # Secrets
            - secretRef:
                name: order-service-secrets
          
          # Resource limits
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          
          # Liveness probe
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            initialDelaySeconds: 60
            periodSeconds: 30
            timeoutSeconds: 10
            failureThreshold: 3
          
          # Readiness probe
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          
          # Startup probe (ให้เวลา startup นานขึ้น)
          startupProbe:
            httpGet:
              path: /health/live
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 30  # 5 minutes max
          
          # Volume mounts
          volumeMounts:
            - name: config
              mountPath: /app/config
              readOnly: true
            - name: tmp
              mountPath: /tmp
      
      volumes:
        - name: config
          configMap:
            name: order-service-config
        - name: tmp
          emptyDir: {}
      
      # Pod scheduling
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - order-service
                topologyKey: kubernetes.io/hostname
      
      # Graceful shutdown
      terminationGracePeriodSeconds: 60
```

---

## Step 832: ConfigMap และ Secrets

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  namespace: scala-app
data:
  # Application config
  APP_NAME: "order-service"
  APP_VERSION: "1.0.0"
  SERVER_PORT: "8080"
  
  # Kafka
  KAFKA_BROKERS: "kafka-headless.kafka.svc.cluster.local:9092"
  KAFKA_CONSUMER_GROUP: "order-service"
  
  # Service URLs (internal Kubernetes DNS)
  USER_SERVICE_URL: "http://user-service.scala-app.svc.cluster.local:8080"
  PRODUCT_SERVICE_URL: "http://product-service.scala-app.svc.cluster.local:8080"
  
  # JVM options
  JAVA_OPTS: >-
    -server
    -XX:+UseContainerSupport
    -XX:MaxRAMPercentage=75.0
    -XX:+UseG1GC
    -XX:+ExitOnOutOfMemoryError
    -Dfile.encoding=UTF-8
  
  # Application config file
  application.conf: |
    server {
      port = 8080
    }
    database {
      pool.min-connections = 5
      pool.max-connections = 20
    }
    features {
      enable-new-checkout = false
    }

---
# secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
  namespace: scala-app
type: Opaque
stringData:
  # Database
  DB_URL: "jdbc:postgresql://postgres.scala-app.svc.cluster.local:5432/orders"
  DB_USER: "orders_user"
  DB_PASSWORD: "very-secret-password"
  
  # External API keys
  PAYMENT_API_KEY: "pk_live_xxxxxxxxxxxx"
  STRIPE_WEBHOOK_SECRET: "whsec_xxxxxxxxxxxx"
```

```scala
// SecretsFromK8s.scala
// ใน Kubernetes ไม่ต้อง read secrets programmatically
// inject เป็น environment variables ผ่าน secretRef

object AppConfigFromK8s {
  
  // Read from environment (injected by Kubernetes)
  val dbUrl: String      = sys.env.getOrElse("DB_URL", "jdbc:postgresql://localhost:5432/myapp")
  val dbUser: String     = sys.env.getOrElse("DB_USER", "myapp")
  val dbPassword: String = sys.env.getOrElse("DB_PASSWORD", "password")
  
  // Pod metadata (injected via Downward API)
  val podName: String      = sys.env.getOrElse("POD_NAME", "unknown")
  val podNamespace: String = sys.env.getOrElse("POD_NAMESPACE", "unknown")
  val podIP: String        = sys.env.getOrElse("POD_IP", "127.0.0.1")
  
  def main(args: Array[String]): Unit = {
    println(s"Running as pod: $podName in namespace: $podNamespace")
    println(s"IP: $podIP")
    println(s"DB: $dbUrl")
  }
}
```

---

## Step 833: Services

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: scala-app
  labels:
    app: order-service
spec:
  type: ClusterIP  # internal only
  selector:
    app: order-service
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
    - name: metrics
      port: 9095
      targetPort: 9095

---
# service-external.yaml (LoadBalancer for prod, NodePort for dev)
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
  namespace: scala-app
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: api-gateway
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: https
      port: 443
      targetPort: 8443
  loadBalancerSourceRanges:
    - "10.0.0.0/8"

---
# headless-service.yaml (สำหรับ stateful apps เช่น Akka Cluster)
apiVersion: v1
kind: Service
metadata:
  name: order-service-headless
  namespace: scala-app
spec:
  clusterIP: None  # headless
  selector:
    app: order-service
  ports:
    - port: 2551
      name: akka-management
    - port: 25520
      name: akka-remoting
```

---

## Step 834: Ingress

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: scala-app
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rpm: "100"
    nginx.ingress.kubernetes.io/limit-rps: "10"
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://myapp.com"
spec:
  tls:
    - hosts:
        - api.myapp.com
      secretName: api-tls-cert
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /api/v1/orders(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
          - path: /api/v1/products(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 80
          - path: /api/v1/users(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80
```

---

## Step 835: HPA (Horizontal Pod Autoscaler)

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: scala-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  
  minReplicas: 2
  maxReplicas: 20
  
  metrics:
    # CPU-based scaling
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    
    # Memory-based scaling
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    
    # Custom metric: requests per second
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
    
    # External metric: Kafka consumer lag
    - type: External
      external:
        metric:
          name: kafka_consumer_lag
          selector:
            matchLabels:
              topic: order-events
              group: order-service
        target:
          type: AverageValue
          averageValue: "1000"
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Pods
          value: 4
          periodSeconds: 15  # max 4 pods per 15 seconds
    scaleDown:
      stabilizationWindowSeconds: 300  # wait 5 min before scale down
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60  # remove max 25% per minute
```

---

## Step 836: StatefulSet สำหรับ Akka Cluster

```yaml
# statefulset-akka.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: order-service-cluster
  namespace: scala-app
spec:
  serviceName: order-service-headless
  replicas: 3
  selector:
    matchLabels:
      app: order-service-cluster
  
  template:
    metadata:
      labels:
        app: order-service-cluster
    spec:
      containers:
        - name: order-service
          image: registry.mycompany.com/order-service:1.0.0
          env:
            - name: AKKA_CLUSTER_BOOTSTRAP_SERVICE_NAME
              value: order-service-headless
            - name: AKKA_MANAGEMENT_HTTP_PORT
              value: "8558"
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 8558
              name: management
            - containerPort: 25520
              name: remoting
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "1000m"
          
          volumeMounts:
            - name: data
              mountPath: /app/data
  
  # Persistent volumes for each pod
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "gp3"
        resources:
          requests:
            storage: 10Gi
```

```scala
// AkkaClusterBootstrap.scala
// application.conf สำหรับ Kubernetes Akka Cluster Bootstrap

/*
akka {
  actor.provider = cluster
  
  management {
    http {
      hostname = ${?POD_IP}
      port = 8558
    }
    
    cluster.bootstrap {
      contact-point-discovery {
        service-name = ${?AKKA_CLUSTER_BOOTSTRAP_SERVICE_NAME}
        service-name = "order-service-headless"
        discovery-method = kubernetes-api
        port-name = management
      }
    }
  }
  
  discovery {
    method = kubernetes-api
    kubernetes-api {
      pod-namespace = ${?POD_NAMESPACE}
      pod-namespace = "scala-app"
      pod-label-selector = "app=order-service-cluster"
      pod-port-name = management
    }
  }
  
  cluster {
    shutdown-after-unsuccessful-join-seed-nodes = 60s
    
    downing-provider-class = "akka.cluster.sbr.SplitBrainResolverProvider"
    split-brain-resolver {
      active-strategy = keep-majority
    }
  }
  
  remote.artery {
    canonical {
      hostname = ${?POD_IP}
      port = 25520
    }
  }
}
*/

object AkkaClusterApp {
  def main(args: Array[String]): Unit = {
    import akka.actor.typed.ActorSystem
    import akka.actor.typed.scaladsl.Behaviors
    import akka.management.scaladsl.AkkaManagement
    import akka.management.cluster.bootstrap.ClusterBootstrap
    
    val system = ActorSystem(Behaviors.empty, "ClusterSystem")
    
    // Start Akka Management HTTP endpoint
    AkkaManagement(system).start()
    
    // Start Cluster Bootstrap
    ClusterBootstrap(system).start()
    
    println(s"Akka Cluster starting on ${sys.env.getOrElse("POD_IP", "localhost")}")
  }
}
```

---

## Step 837: PodDisruptionBudget

```yaml
# pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: scala-app
spec:
  minAvailable: 2  # always keep at least 2 pods running
  selector:
    matchLabels:
      app: order-service

---
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: scala-app-quota
  namespace: scala-app
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "50"
    services: "20"
    persistentvolumeclaims: "20"

---
# limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: scala-app-limits
  namespace: scala-app
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "256Mi"
      max:
        cpu: "4"
        memory: "8Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
```

---

## Step 838: Kubernetes Secrets Management with External Secrets

```yaml
# external-secrets.yaml
# ใช้ External Secrets Operator กับ AWS Secrets Manager

apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: order-service-secrets
  namespace: scala-app
spec:
  refreshInterval: "1h"
  
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  
  target:
    name: order-service-secrets
    creationPolicy: Owner
  
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: myapp/order-service
        property: db_password
    
    - secretKey: PAYMENT_API_KEY
      remoteRef:
        key: myapp/payment
        property: api_key
    
    - secretKey: JWT_SECRET
      remoteRef:
        key: myapp/auth
        property: jwt_secret

---
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

---

## Step 839: Helm Chart

```yaml
# charts/order-service/Chart.yaml
apiVersion: v2
name: order-service
description: Order Service Helm Chart
type: application
version: 0.1.0
appVersion: "1.0.0"

---
# charts/order-service/values.yaml
replicaCount: 3

image:
  repository: registry.mycompany.com/order-service
  tag: "1.0.0"
  pullPolicy: Always

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  host: api.myapp.com
  tls: true

resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "500m"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

config:
  kafkaBrokers: "kafka:9092"
  logLevel: "INFO"

secrets:
  dbUrl: ""       # set via --set or values-override.yaml
  dbPassword: ""

---
# charts/order-service/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "order-service.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "order-service.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "order-service.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "order-service.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          env:
            - name: KAFKA_BROKERS
              value: {{ .Values.config.kafkaBrokers }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

---

## Step 840: kubectl Commands

```bash
#!/bin/bash
# Common kubectl commands สำหรับ Scala app management

NAMESPACE="scala-app"
APP="order-service"

# ===== Deployment =====
# Apply all resources
kubectl apply -f k8s/ -n $NAMESPACE

# Check rollout status
kubectl rollout status deployment/$APP -n $NAMESPACE

# Rollback if needed
kubectl rollout undo deployment/$APP -n $NAMESPACE

# Scale manually
kubectl scale deployment/$APP --replicas=5 -n $NAMESPACE

# ===== Debugging =====
# Get pod logs
kubectl logs -l app=$APP -n $NAMESPACE --tail=100 -f

# Exec into pod
kubectl exec -it $(kubectl get pods -l app=$APP -n $NAMESPACE -o name | head -1) -n $NAMESPACE -- /bin/sh

# Describe pod (events, resource usage)
kubectl describe pod -l app=$APP -n $NAMESPACE

# Check resource usage
kubectl top pods -l app=$APP -n $NAMESPACE
kubectl top nodes

# ===== Port Forward (for debugging) =====
kubectl port-forward svc/$APP 8080:80 -n $NAMESPACE

# ===== Config =====
# View configmap
kubectl get configmap order-service-config -n $NAMESPACE -o yaml

# Edit configmap (triggers pod restart if using envFrom)
kubectl edit configmap order-service-config -n $NAMESPACE

# ===== Secrets =====
# View secret keys (values are base64 encoded)
kubectl get secret order-service-secrets -n $NAMESPACE -o yaml

# Decode a secret value
kubectl get secret order-service-secrets -n $NAMESPACE -o jsonpath='{.data.DB_PASSWORD}' | base64 -d

# ===== Health =====
# Check HPA status
kubectl get hpa -n $NAMESPACE
kubectl describe hpa order-service-hpa -n $NAMESPACE

# Check PDB
kubectl get pdb -n $NAMESPACE

# ===== Helm =====
# Install/upgrade with Helm
helm upgrade --install $APP charts/order-service \
  --namespace $NAMESPACE \
  --create-namespace \
  --set image.tag="1.2.0" \
  --set-string secrets.dbPassword="$DB_PASSWORD" \
  -f values-production.yaml

# Check Helm releases
helm list -n $NAMESPACE
helm history $APP -n $NAMESPACE
```

---

## สรุป Part 84: Kubernetes

| Resource | Purpose | Key Config |
|---------|---------|-----------|
| Deployment | Run stateless apps | replicas, strategy, probes |
| StatefulSet | Stateful apps (Akka Cluster) | headless service, PVC |
| ConfigMap | App config | envFrom, volumes |
| Secret | Sensitive data | env injection |
| Service | Networking | ClusterIP, LoadBalancer |
| Ingress | External traffic | TLS, routing |
| HPA | Auto-scaling | CPU/memory metrics |
| PDB | High availability | minAvailable |

### Production Checklist
- [ ] Resource requests/limits set
- [ ] Liveness + readiness probes
- [ ] HPA configured
- [ ] PDB for HA
- [ ] Non-root container
- [ ] Network policies
- [ ] Secret management
- [ ] Graceful shutdown (terminationGracePeriodSeconds)

---

## แบบฝึกหัด Part 84

1. **Complete K8s Stack**: สร้าง Kubernetes manifests ครบสำหรับ Scala microservice ที่มี Deployment, Service, ConfigMap, Secret, Ingress, HPA, PDB

2. **Akka Cluster**: deploy Akka Cluster ใน Kubernetes ด้วย StatefulSet, headless service, และ Kubernetes bootstrap discovery

3. **Helm Chart**: สร้าง production-ready Helm chart ที่มี values hierarchy (base, dev, prod), notes.txt, และ RBAC resources

4. **Scaling Test**: test HPA โดย generate load ด้วย k6 หรือ vegeta แล้ว observe auto-scaling behavior

5. **Zero-Downtime Deployment**: ทดสอบ rolling deployment strategy, verify ไม่มี connection drop ระหว่าง deploy

---

## ไปต่อ: Part 85 — gRPC with Scala
ใน Part ถัดไปจะเรียน ScalaPB, proto definitions, gRPC streaming

[→ Part 85: gRPC with Scala](./part-85-grpc-with-scala.md)
