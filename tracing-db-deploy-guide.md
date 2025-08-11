# Deploying App + OpenTelemetry to new tenant/namespace `tracing-db` (OpenShift 4.18)

This guide updates your original markdown to deploy the application in a **new namespace `tracing-db`**, deploy a **tenant-scoped OpenTelemetry Collector** that exports to the Tempo **gateway** using **`X-Scope-OrgID: tracing-db`**, and **updates the TempoStack** to include the new tenant.

> Assumptions: OpenTelemetry and Tempo operators are already installed (per your original guide), and you already have a working `TempoStack` named `tracing-tempo` in namespace `tracing-system` with a gateway enabled.

---

## 0) Clone or open the repo with the manifests

If you downloaded the overlay zip I provided (`tracing-db-update.zip`), you have this layout:
```
k8s/
  00-namespace.yaml
  01-instrumentation.yaml
  02-otel-collector-tracing-db.yaml
  03-tempostack-tracing-system.yaml
  04-tracing-demo-deployment.yaml
README-tracing-db.md
```

You can also just copy the YAMLs inline below.

---

## 1) Create the **tracing-db** namespace

```bash
oc apply -f k8s/00-namespace.yaml
```

**k8s/00-namespace.yaml**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tracing-db
  labels:
    name: tracing-db
```

---

## 2) Update **TempoStack** to add a new tenant `tracing-db`

Apply this **in the `tracing-system` namespace** (same as your existing TempoStack). It appends the new tenant; keep your existing fields (storage, gateway, etc.) aligned with your current stack.

```bash
oc apply -f k8s/03-tempostack-tracing-system.yaml
```

**k8s/03-tempostack-tracing-system.yaml**
```yaml
apiVersion: tempo.grafana.com/v1alpha1
kind: TempoStack
metadata:
  name: tracing-tempo
  namespace: tracing-system
spec:
  storage:
    secret:
      name: tempo-storage-secret
      type: s3
    tls:
      enabled: true
      caName: minio-ca-bundle
  storageSize: 10Gi
  template:
    queryFrontend:
      jaegerQuery:
        enabled: true
    gateway:
      enabled: true
  tenants:
    mode: openshift
    authentication:
      - tenantName: tracing-demo
        tenantId: tracing-demo
      - tenantName: dev
        tenantId: dev
      - tenantName: prod
        tenantId: prod
      - tenantName: tracing-db
        tenantId: tracing-db
```

> If your existing spec differs (e.g., another `storage` configuration or additional tenants), merge accordingly. The important part is adding the `tracing-db` tenant under `.spec.tenants.authentication`.

---

## 3) Create `Instrumentation` (auto-inject Java agent) in `tracing-db`

This enables the OTel Java auto-instrumentation via annotation on pods in this namespace.

```bash
oc apply -f k8s/01-instrumentation.yaml
```

**k8s/01-instrumentation.yaml**
```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: java-auto
  namespace: tracing-db
spec:
  exporter:
    otlp:
      endpoint: http://otel-collector.tracing-db.svc.cluster.local:4317
  sampler:
    type: parentbased_traceidratio
    argument: "1.0"
  propagators:
    - tracecontext
    - baggage
    - b3
  resource:
    attributes:
      service.namespace: tracing-db
      deployment.environment: dev
```

> The endpoint points to the collector we deploy in the next step.

---

## 4) Deploy a **tenant-scoped OpenTelemetry Collector** for `tracing-db`

This collector forwards traces to the **Tempo gateway**, tagging with `X-Scope-OrgID: tracing-db`. If your gateway uses internal TLS, mount the CA and configure `tls.ca_file` instead of skipping verification.

```bash
oc apply -f k8s/02-otel-collector-tracing-db.yaml
```

**k8s/02-otel-collector-tracing-db.yaml**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: otel-collector-sa
  namespace: tracing-db
---
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
  namespace: tracing-db
spec:
  mode: deployment
  serviceAccount: otel-collector-sa
  config:
    extensions:
      bearertokenauth:
        filename: "/var/run/secrets/kubernetes.io/serviceaccount/token"
    receivers:
      otlp:
        protocols:
          grpc: {}
          http: {}
      jaeger:
        protocols:
          thrift_http: {}
      zipkin: {}
    processors:
      memory_limiter:
        check_interval: 5s
        limit_percentage: 75
        spike_limit_percentage: 20
      batch: {}
    exporters:
      otlphttp:
        endpoint: https://tracing-tempo-gateway.tracing-system.svc.cluster.local:8080
        tls:
          insecure_skip_verify: false
        headers:
          X-Scope-OrgID: "tracing-db"
        auth:
          authenticator: bearertokenauth
      debug:
        verbosity: detailed
        sampling_initial: 5
        sampling_thereafter: 200
    service:
      extensions: [bearertokenauth]
      pipelines:
        traces:
          receivers: [otlp, jaeger, zipkin]
          processors: [memory_limiter, batch]
          exporters: [otlphttp, debug]
```

**TLS note:** If the gateway uses an internal CA, add a Secret with the CA and mount it, then set:
```yaml
    exporters:
      otlphttp:
        endpoint: https://tracing-tempo-gateway.tracing-system.svc.cluster.local:8080
        tls:
          ca_file: /etc/otel/certs/ca.crt
          insecure_skip_verify: false
```
and mount `/etc/otel/certs` into the collector pod.

---

## 5) Build and deploy your **application** in `tracing-db`

If you’re using the simple example from the original document (`tracing-demo`), you can build in-cluster using Docker strategy + binary input:

```bash
# (from your app directory that contains Dockerfile and code)
oc project tracing-db

oc new-build --name=tracing-demo --binary --strategy=docker -n tracing-db
oc start-build tracing-demo --from-dir=. --follow -n tracing-db
```

Now deploy the app with the OTel Java agent auto-injected. The Deployment specifies the annotation and uses the ImageStream tag we just built.

```bash
oc apply -f k8s/04-tracing-demo-deployment.yaml
```

**k8s/04-tracing-demo-deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tracing-demo
  namespace: tracing-db
  labels:
    app: tracing-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tracing-demo
  template:
    metadata:
      labels:
        app: tracing-demo
      annotations:
        instrumentation.opentelemetry.io/inject-java: "true"
        instrumentation.opentelemetry.io/container-names: "tracing-demo"
    spec:
      containers:
        - name: tracing-demo
          image: image-registry.openshift-image-registry.svc:5000/tracing-db/tracing-demo:latest
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: tracing-demo
  namespace: tracing-db
spec:
  selector:
    app: tracing-demo
  ports:
    - name: http
      port: 8080
      targetPort: 8080
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: tracing-demo
  namespace: tracing-db
spec:
  to:
    kind: Service
    name: tracing-demo
  port:
    targetPort: http
  tls:
    termination: edge
```

Get the route:
```bash
oc get route tracing-demo -n tracing-db -o jsonpath='{.spec.host}{"\n"}'
```

Open `https://<host>` to hit the app.

---

## 6) Validate tracing

- Check the collector logs in `tracing-db`:
  ```bash
  oc logs deploy/otel-collector -n tracing-db | grep -E "X-Scope-OrgID|exporter|error|trace"
  ```
- Ensure traces are exported with header `X-Scope-OrgID: tracing-db`.
- Open your Tempo/Jaeger UI and search for the service(s) with `service.namespace=tracing-db`.

---

## 7) Cleanup

```bash
oc delete -f k8s/04-tracing-demo-deployment.yaml
oc delete -f k8s/02-otel-collector-tracing-db.yaml
oc delete -f k8s/01-instrumentation.yaml
# Consider whether to remove the tenant or keep it:
# oc apply a modified 03-tempostack-tracing-system.yaml without the tracing-db entry
oc delete project tracing-db
```

---

### Troubleshooting

- **TLS unknown authority to Tempo gateway:** mount your internal CA into the collector and set `tls.ca_file`. Avoid `insecure_skip_verify` outside of dev.
- **No traces:** verify the app pod has the auto-injection annotation, the `Instrumentation` exists in the namespace, and the collector receives spans (`oc logs deploy/otel-collector -n tracing-db`).
- **Tempo tenant not found/unauthorized:** confirm `X-Scope-OrgID: tracing-db` matches a tenant in `.spec.tenants.authentication` of the TempoStack and that the gateway is enabled.
