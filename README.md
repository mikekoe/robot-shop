# Stan's Robot Shop — Capstone 2

A sample microservice application used as a sandbox for containerised application
orchestration, observability, and incident response. Built on the original
[instana/robot-shop](https://github.com/instana/robot-shop) and extended with
a full observability stack, SLO monitoring, and distributed tracing.

---

## Table of Contents

1. [Architecture](#architecture)
2. [Services](#services)
3. [Run Locally with Docker Compose](#run-locally-with-docker-compose)
4. [Kubernetes Deployment with Helm](#kubernetes-deployment-with-helm)
5. [Observability Stack](#observability-stack)
6. [Distributed Tracing with Jaeger](#distributed-tracing-with-jaeger)
7. [SLO and Alerting](#slo-and-alerting)
8. [Incident Drill and Postmortem](#incident-drill-and-postmortem)
9. [Infrastructure](#infrastructure)
10. [Prometheus Metrics](#prometheus-metrics)
11. [Notes and Known Issues](#notes-and-known-issues)

---

## Architecture

```
Browser
  │
  ▼
web (Nginx + AngularJS)
  │
  ├──▶ catalogue (Node.js)  ──▶ MongoDB
  ├──▶ user (Node.js)       ──▶ MongoDB
  ├──▶ cart (Node.js)       ──▶ Redis
  ├──▶ payment (Python)     ──▶ RabbitMQ ──▶ dispatch (Go)
  ├──▶ shipping (Java)      ──▶ MySQL
  └──▶ ratings (PHP)        ──▶ MySQL
```

### Observability stack

```
Pods (all namespaces)
  │
  ├──▶ Promtail ──▶ Loki ──▶ Grafana (log search)
  ├──▶ Prometheus ──▶ Grafana (RED dashboard)
  └──▶ OpenTelemetry Collector ──▶ Jaeger (traces)

Alertmanager ──▶ Slack #general
```

---

## Services

| Service | Language | Role | Data store |
|---|---|---|---|
| web | Nginx + AngularJS | Front-end, serves the store UI | — |
| catalogue | Node.js | Product catalogue API | MongoDB |
| user | Node.js | User accounts and auth | MongoDB |
| cart | Node.js | Shopping cart | Redis |
| payment | Python | Payment processing | RabbitMQ |
| shipping | Java / Spring Boot | Shipping cost calculation | MySQL |
| ratings | PHP | Product ratings | MySQL |
| dispatch | Go | Order dispatch | RabbitMQ |
| mongodb | MongoDB | Document store | — |
| redis | Redis | In-memory queue | — |
| mysql | MySQL + MaxMind | Relational store | — |
| rabbitmq | RabbitMQ | Message broker | — |

---

## Run Locally with Docker Compose

**Prerequisites:** [Docker Desktop](https://www.docker.com/products/docker-desktop)

```bash
git clone https://github.com/mikekoe/robot-shop.git
cd robot-shop
docker compose up
```

The store is available at http://localhost:8080.

To run with load generation:

```bash
docker compose -f docker-compose.yaml -f docker-compose-load.yaml up
```

---

## Kubernetes Deployment with Helm

### Prerequisites

- EKS cluster running (see [Infrastructure](#infrastructure))
- kubectl pointed at the cluster
- [Helm](https://helm.sh/docs/intro/install/) 3+
- Nodes: at least 2 x t3.medium (17 pods per node)

### Install

```bash
cd k8s
kubectl create namespace robot-shop

helm install robot-shop ./helm \
  --namespace robot-shop \
  --wait --timeout 10m

kubectl get pods -n robot-shop
```

All 12 pods should reach Running:

```
NAME                         READY   STATUS    RESTARTS
cart-...                     1/1     Running   0
catalogue-...                1/1     Running   0
dispatch-...                 1/1     Running   0
mongodb-...                  1/1     Running   0
mysql-...                    1/1     Running   0
payment-...                  1/1     Running   0
rabbitmq-...                 1/1     Running   0
ratings-...                  1/1     Running   0
redis-0                      1/1     Running   0
shipping-...                 1/1     Running   0
user-...                     1/1     Running   0
web-...                      1/1     Running   0
```

### Access the store

```bash
kubectl port-forward -n robot-shop svc/web 8080:8080
```

Open http://localhost:8080. The four focus services are confirmed working:

| Service | Test |
|---|---|
| web | Storefront loads |
| catalogue | Products and categories listed with prices |
| cart | Add to cart, totals calculated correctly |
| payment | Checkout flow reachable |

### Helm chart changes from upstream

The upstream chart uses a StatefulSet with a PersistentVolumeClaim for Redis.
EKS does not install the EBS CSI driver by default (Kubernetes 1.23+), so the
PVC never binds. The fix is in k8s/helm/templates/redis-statefulset.yaml:

- Removed volumeClaimTemplates entirely
- Changed the Redis data volume to emptyDir: {}
- Changed storageClassName from standard to gp2 in values.yaml

Redis data does not persist across pod restarts, which is acceptable since Redis
is used only as a queue for this application.

### Load generation

The upstream load-gen image (robotshop/rs-load-gen) does not exist on Docker Hub.
Use a curl-based alternative instead:

```bash
kubectl run load-gen \
  --image=curlimages/curl \
  --namespace robot-shop \
  --restart=Always \
  -- sh -c '
    while true; do
      curl -s http://web:8080/ > /dev/null
      curl -s http://web:8080/api/catalogue/categories > /dev/null
      curl -s http://web:8080/api/catalogue/products/robots > /dev/null
      sleep 3
    done
  '
```

### Uninstall

```bash
helm uninstall robot-shop -n robot-shop
```

---

## Observability Stack

All components are installed in the monitoring namespace.

### Install kube-prometheus-stack

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --wait --timeout 10m
```

### Access Grafana

```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

Open http://localhost:3000. Default credentials: admin / prom-operator
(or reset with kubectl exec or Helm --set grafana.adminPassword=admin123).

### RED Dashboard

A custom dashboard (Robot Shop — RED Dashboard) was created with six panels:

| Panel | Metric | Source |
|---|---|---|
| Requests per second | Network receive rate per pod | cAdvisor |
| Error rate | Pod restart rate | kube-state-metrics |
| Latency p99 | Memory working set per pod | cAdvisor |
| Pod restarts | Container restarts in 1h | kube-state-metrics |
| Running pods | Count of running pods | kube-state-metrics |
| CPU usage | CPU cores per pod | cAdvisor |

Note: robot-shop services do not expose Prometheus HTTP metrics natively.
The dashboard uses cAdvisor and kube-state-metrics as proxies for RED signals.
The cart and payment services expose /metrics endpoints (see Prometheus Metrics).

### Log collection with Loki and Promtail

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# install Loki
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=false \
  --set prometheus.enabled=false \
  --set loki.persistence.enabled=false

# install Promtail (points to Loki gateway)
cat > promtail-values.yaml << 'EOF'
config:
  clients:
    - url: http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push
EOF

helm install promtail grafana/promtail \
  --namespace monitoring \
  --values promtail-values.yaml
```

#### Add Loki as a Grafana datasource

In Grafana: Connections > Data sources > Add > Loki
- URL: http://loki.monitoring.svc.cluster.local:3100

#### Search logs in Grafana Explore

```logql
# All robot-shop logs
{namespace="robot-shop"}

# Logs for a specific service
{namespace="robot-shop", app="catalogue"}

# Error logs only
{namespace="robot-shop"} |= "error"

# Payment service errors
{namespace="robot-shop", app="payment"} |= "error"
```

Available labels: namespace, app, pod, container, node_name, stream

---

## Distributed Tracing with Jaeger

### Install Jaeger

```bash
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

cat > jaeger-values.yaml << 'EOF'
allInOne:
  enabled: true
  args:
    - "--memory.max-traces=10000"
storage:
  type: memory
collector:
  enabled: false
query:
  enabled: false
agent:
  enabled: false
EOF

helm install jaeger jaegertracing/jaeger \
  --namespace monitoring \
  --values jaeger-values.yaml
```

### Install OpenTelemetry Collector

```bash
helm repo add open-telemetry \
  https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

cat > otel-collector-values.yaml << 'EOF'
image:
  repository: otel/opentelemetry-collector-contrib
mode: deployment
config:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318
  exporters:
    otlp/jaeger:
      endpoint: jaeger.monitoring.svc.cluster.local:14250
      tls:
        insecure: true
    debug:
      verbosity: detailed
  service:
    pipelines:
      traces:
        receivers: [otlp]
        exporters: [otlp/jaeger, debug]
ports:
  otlp:
    enabled: true
    containerPort: 4317
    servicePort: 4317
    protocol: TCP
  otlp-http:
    enabled: true
    containerPort: 4318
    servicePort: 4318
    protocol: TCP
EOF

helm install otel-collector open-telemetry/opentelemetry-collector \
  --namespace monitoring \
  --values otel-collector-values.yaml
```

### Send a test trace to the catalogue service

```bash
kubectl port-forward -n monitoring svc/jaeger 4318:4318 &

curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": {
        "attributes": [{
          "key": "service.name",
          "value": {"stringValue": "catalogue"}
        }]
      },
      "scopeSpans": [{
        "spans": [{
          "traceId": "5b8efff798038103d269b633813fc60c",
          "spanId": "eee19b7ec3c1b174",
          "name": "GET /categories",
          "startTimeUnixNano": "1793170800000000000",
          "endTimeUnixNano": "1793170801000000000",
          "kind": 2,
          "attributes": [
            {"key": "http.method", "value": {"stringValue": "GET"}},
            {"key": "http.url", "value": {"stringValue": "/categories"}},
            {"key": "http.status_code", "value": {"intValue": 200}}
          ],
          "status": {"code": 1}
        }]
      }]
    }]
  }'
```

### View traces in Jaeger UI

```bash
kubectl port-forward -n monitoring svc/jaeger 16686:16686 &
```

Open http://localhost:16686, select catalogue from the Service dropdown
and click Find Traces.

---

## SLO and Alerting

### SLO definition

| Property | Value |
|---|---|
| Service | catalogue |
| SLI | Availability: HTTP requests not returning 5xx |
| SLO target | 99% over a 30-day rolling window |
| Error budget | 1% (~7.2 hours per 30 days) |

See error-budget-policy.md for the full error budget policy.

### Apply the PrometheusRule

```bash
kubectl apply -f slo-alert.yaml
kubectl get prometheusrule -n monitoring catalogue-slo
```

Three alert rules are defined:

| Alert | Condition | Severity |
|---|---|---|
| CatalogueDown | 0 available replicas for 1m | critical |
| CatalogueErrorBudgetBurn | >2 restarts in 1h for 5m | warning |
| CatalogueSLOAtRisk | availability < 99% for 2m | warning |

### Slack alerting

Alertmanager sends alerts to #general in the devlearn Slack workspace.
The webhook URL is stored as a Kubernetes secret and never committed to Git.
The alertmanager-slack.yaml file contains a SLACK_WEBHOOK_URL placeholder only.

To apply the Alertmanager config with a real webhook:

```bash
kubectl create secret generic \
  alertmanager-monitoring-kube-prometheus-alertmanager \
  --from-file=alertmanager.yaml=/tmp/alertmanager.yaml \
  --namespace monitoring \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl rollout restart statefulset/alertmanager-monitoring-kube-prometheus-alertmanager \
  -n monitoring
```

### Trigger a test alert

```bash
# scale catalogue to 0 to trigger CatalogueDown
kubectl scale deployment catalogue -n robot-shop --replicas=0

# wait 90 seconds, check Alertmanager at http://localhost:9093
# and Slack #general for the notification

# restore catalogue
kubectl scale deployment catalogue -n robot-shop --replicas=1
```

---

## Incident Drill and Postmortem

An incident drill was run as part of the capstone. See postmortem.md for the
completed postmortem. The process followed three phases:

### Phase 1: Mitigate (stop the bleeding first)

```bash
# identify what is broken
kubectl get pods -A | grep -vE "Running|Completed"
kubectl get events -A --sort-by=.lastTimestamp | tail -20
kubectl get deployments -A | grep -v "1/1\|2/2"

# common mitigations
kubectl scale deployment <name> -n robot-shop --replicas=1
kubectl rollout undo deployment/<name> -n robot-shop
kubectl rollout restart deployment/<name> -n robot-shop
```

### Phase 2: Find the root cause

```bash
# check logs around the incident time in Loki Explore:
# {namespace="robot-shop"} |= "error"

# check what changed
kubectl describe deployment <name> -n robot-shop
kubectl logs -n robot-shop <pod> --previous
```

### Phase 3: Write the postmortem

See postmortem.md covering: timeline, impact, root cause,
contributing factors, what went well, what went wrong, and action items.

---

## Infrastructure

The cluster runs on AWS EKS in eu-north-1. Terraform code is in
../voting-app/k8s-terraform/.

### Build the cluster

```bash
cd ../voting-app/k8s-terraform
terraform init
terraform apply
$(terraform output -raw kubeconfig_command)
kubectl get nodes
```

Nodes: 2 x t3.medium, 17 pods per node, 34 total pod slots.

### Destroy the cluster

```bash
helm uninstall robot-shop -n robot-shop
helm uninstall monitoring -n monitoring
helm uninstall loki -n monitoring
helm uninstall promtail -n monitoring
helm uninstall jaeger -n monitoring
helm uninstall otel-collector -n monitoring
terraform destroy
aws eks list-clusters --region eu-north-1
```

Cost: EKS control plane ~$0.10/hour + 2 x t3.medium nodes.
Destroy when not in use.

---

## Prometheus Metrics

The cart and payment services expose Prometheus metrics on /metrics.

Cart (/api/cart/metrics):
- Counter: number of items added to cart

Payment (/api/payment/metrics):
- Counter: number of items purchased
- Histogram: total number of items per cart
- Histogram: total value per cart

```bash
kubectl port-forward -n robot-shop svc/web 8080:8080 &
curl http://localhost:8080/api/cart/metrics
curl http://localhost:8080/api/payment/metrics
```

---

## Notes and Known Issues

- Redis persistence: Redis uses emptyDir in Kubernetes. Cart data is lost if
  the Redis pod restarts. A PVC backed by the EBS CSI driver would be needed
  for production.

- Load generator: The upstream robotshop/rs-load-gen image does not exist on
  Docker Hub. The curl-based alternative is used instead.

- Instana agent: The EUM settings (eum.key, eum.url) are left at null in
  values.yaml. No Instana account is needed to run the application.

- Node capacity: Both t3.medium nodes allow 17 pods each. With the full
  observability stack alongside robot-shop, pods may hit the limit. Scale
  down non-essential monitoring components or add a third node if needed.

- Loki version: loki-stack (deprecated) is used because newer Loki chart
  versions (6.x, 7.x) require the EBS CSI driver for cache StatefulSets.
  Promtail is configured to point at the Loki gateway service.

- Jaeger traces: robot-shop services use Instana auto-instrumentation, not
  OpenTelemetry. Traces are sent via the OTel HTTP endpoint directly from
  the local machine using curl when nodes are at pod capacity.

- Secrets: The Slack webhook URL is stored only in the Kubernetes secret
  alertmanager-monitoring-kube-prometheus-alertmanager. It is never committed
  to Git. GitHub push protection blocks any commit containing the real URL.
