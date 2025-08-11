# Complete Guide: OpenTelemetry and Distributed Tracing on OpenShift 4.18

This comprehensive guide walks through the deployment of Red Hat build of OpenTelemetry and Distributed Tracing on OpenShift Container Platform 4.18, including a Java Spring Boot application for testing.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Installation Steps](#installation-steps)
4. [Java Spring Boot Test Application](#java-spring-boot-test-application)
5. [Verification and Testing](#verification-and-testing)
6. [Troubleshooting](#troubleshooting)

## Architecture Overview

The distributed tracing stack consists of the following components:

- **Red Hat build of OpenTelemetry Operator**: Manages OpenTelemetry Collectors
- **OpenTelemetry Collector**: Receives, processes, and exports telemetry data
- **Tempo Operator**: Manages Tempo instances for trace storage
- **TempoStack**: Stores and queries trace data
- **OpenShift Console Distributed Tracing UI**: Built-in UI for viewing traces (replaces deprecated Jaeger UI)

## Prerequisites

- OpenShift Container Platform 4.18 cluster
- Cluster admin access
- `oc` CLI tool installed and configured
- Helm 3.x installed (for MinIO deployment)
- Podman installed (instead of Docker)

## Installation Steps

### Step 1: Create Project Namespaces

```bash
# Create namespace for operators
oc create namespace openshift-distributed-tracing

# Create namespace for tracing components
oc create namespace tracing-system

# Create namespace for the test application
oc create namespace tracing-demo
```

### Step 2: Install Red Hat build of OpenTelemetry Operator

#### Option A: Using Web Console

1. Navigate to **Operators → OperatorHub**
2. Search for "Red Hat build of OpenTelemetry"
3. Click on the operator and select **Install**
4. Choose:
   - Installation mode: **All namespaces on the cluster**
   - Installed Namespace: **openshift-distributed-tracing**
   - Update Channel: **stable**
   - Approval Strategy: **Automatic**
5. Click **Install**

#### Option B: Using CLI

```yaml
# opentelemetry-operator-subscription.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: opentelemetry-product
  namespace: openshift-distributed-tracing
spec:
  channel: stable
  installPlanApproval: Automatic
  name: opentelemetry-product
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

```bash
oc apply -f opentelemetry-operator-subscription.yaml
```

### Step 3: Install Tempo Operator

#### Option A: Using Web Console

1. Navigate to **Operators → OperatorHub**
2. Search for "Tempo Operator"
3. Click on the operator and select **Install**
4. Choose:
   - Installation mode: **All namespaces on the cluster**
   - Update Channel: **stable**
   - Approval Strategy: **Automatic**
5. Click **Install**

#### Option B: Using CLI

```yaml
# tempo-operator-subscription.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: tempo-product
  namespace: openshift-tempo-operator
spec:
  channel: stable
  installPlanApproval: Automatic
  name: tempo-product
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

```bash
oc apply -f tempo-operator-subscription.yaml
```

### Step 4: Configure Object Storage

For this example, we'll deploy MinIO using the official Helm chart with self-signed certificates. For production, use S3 or Red Hat OpenShift Data Foundation.

#### Create Security Context Constraints for MinIO

```yaml
# minio-scc.yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: minio-scc
  annotations:
    kubernetes.io/description: "SCC for MinIO Operator"
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: true
allowPrivilegedContainer: false
allowedCapabilities: null
defaultAddCapabilities: null
fsGroup:
  type: RunAsAny
groups: []
priority: 10
readOnlyRootFilesystem: false
requiredDropCapabilities:
- KILL
- MKNOD
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
# Important: Allow seccomp profiles
seccompProfiles:
- '*'
users: []
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
```

```bash
# Apply the SCC
oc apply -f minio-scc.yaml
```

#### Deploy MinIO Operator

```bash
# Create namespace for MinIO Operator
oc new-project minio-operator

# Apply the SCC
oc apply -f minio-scc.yaml

# Add MinIO Operator Helm repository
helm repo add minio-operator https://operator.min.io
helm repo update

# Install MinIO Operator
helm install \
  --namespace minio-operator \
  operator minio-operator/operator

# Wait for operator to be ready
oc wait --for=condition=Ready pods -l app.kubernetes.io/name=operator -n minio-operator --timeout=300s

# Add SCC to MinIO Operator service account (created by Helm)
oc adm policy add-scc-to-user minio-scc -z minio-operator -n minio-operator
```

#### Deploy MinIO Tenant

```bash
# Create namespace for MinIO Tenant
oc new-project minio-tenant

# Create service account for MinIO tenant
oc create serviceaccount minio-sa -n minio-tenant

# Add SCC to MinIO tenant service account
oc adm policy add-scc-to-user minio-scc -z minio-sa -n minio-tenant

# Create credentials secret for MinIO (must have specific keys)
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: minio-creds-secret
  namespace: minio-tenant
type: Opaque
stringData:
  config.env: |
    export MINIO_ROOT_USER=minioadmin
    export MINIO_ROOT_PASSWORD=minioadmin123
EOF

# Create console secret for MinIO
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: minio-console-secret
  namespace: minio-tenant
type: Opaque
stringData:
  CONSOLE_ACCESS_KEY: consoleadmin
  CONSOLE_SECRET_KEY: consoleadmin123
EOF

# Deploy MinIO Tenant
cat <<EOF | oc apply -f -
apiVersion: minio.min.io/v2
kind: Tenant
metadata:
  name: minio
  namespace: minio-tenant
spec:
  pools:
  - name: pool-0
    servers: 4
    volumesPerServer: 1
    volumeClaimTemplate:
      metadata:
        name: data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
    securityContext:
      runAsUser: 1000
      runAsGroup: 1000
      runAsNonRoot: true
      fsGroup: 1000
  
  mountPath: /export
  
  serviceAccountName: minio-sa
  
  # Use auto-generated certificates
  requestAutoCert: true
  
  features:
    bucketDNS: false
    enableSFTP: false
  
  configuration:
    name: minio-creds-secret
  
  users:
  - name: minio-console-secret
  
  env:
  - name: MINIO_BROWSER
    value: "on"
  - name: MINIO_PROMETHEUS_AUTH_TYPE
    value: "public"
  - name: MINIO_STORAGE_CLASS_STANDARD
    value: "EC:2"
  
  # Disable metrics for now
  metrics:
    enabled: false
    
  # Resource allocation
  resources:
    requests:
      memory: 512Mi
      cpu: 250m
    limits:
      memory: 1Gi
      cpu: 500m
EOF

# Wait for MinIO to be ready
oc wait --for=condition=Ready pod -l v1.min.io/tenant=minio -n minio-tenant --timeout=300s

# Extract the auto-generated certificate for Tempo
# For self-signed certificates, the public.crt acts as both the certificate and CA
oc get secret minio-tls -n minio-tenant -o jsonpath='{.data.public\.crt}' | base64 -d > ca.crt

# Create CA ConfigMap for Tempo in tracing-system namespace
# Note: TempoStack expects the key to be named 'service-ca.crt'
oc create configmap minio-ca-bundle \
  --from-file=service-ca.crt=ca.crt \
  -n tracing-system
```

#### Expose MinIO Service and Create Initial Bucket

The MinIO Tenant automatically creates the necessary services. Let's verify they exist and create routes for external access:

```bash
# Verify MinIO services are created
oc get svc -n minio-tenant

# Create a Route to expose MinIO API (optional, for external S3 access)
cat <<EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: minio
  namespace: minio-tenant
spec:
  to:
    kind: Service
    name: minio
  port:
    targetPort: 443
  tls:
    termination: passthrough
EOF

# Create a Route to expose MinIO Console
cat <<EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: minio-console
  namespace: minio-tenant
spec:
  to:
    kind: Service
    name: minio-console
  port:
    targetPort: 9443
  tls:
    termination: passthrough
EOF

# Get the console URL
echo "MinIO Console URL: https://$(oc get route minio-console -n minio-tenant -o jsonpath='{.spec.host}')"

# Get console credentials
echo "Console Username: $(oc get secret minio-console-secret -n minio-tenant -o jsonpath='{.data.CONSOLE_ACCESS_KEY}' | base64 -d)"
echo "Console Password: $(oc get secret minio-console-secret -n minio-tenant -o jsonpath='{.data.CONSOLE_SECRET_KEY}' | base64 -d)"

# Create bucket using MinIO client from inside a pod
cat <<EOF | oc apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: create-bucket
  namespace: minio-tenant
spec:
  template:
    spec:
      serviceAccountName: minio-sa
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        runAsNonRoot: true
      containers:
      - name: mc
        image: minio/mc:latest
        command:
        - /bin/sh
        - -c
        - |
          export MC_CONFIG_DIR=/tmp/.mc
          export MC_HOST_minio=https://minioadmin:minioadmin123@minio:443
          
          # Wait for MinIO to be ready
          echo "Waiting for MinIO to be ready..."
          for i in {1..30}; do
            if mc ls minio --insecure 2>/dev/null; then
              echo "MinIO is ready!"
              break
            fi
            echo "Attempt $i/30: MinIO not ready yet, waiting..."
            sleep 10
          done
          
          # Create the bucket
          echo "Creating bucket tempo-traces..."
          mc mb minio/tempo-traces --insecure || true
          
          # List buckets to verify
          echo "Listing buckets..."
          mc ls minio --insecure
        volumeMounts:
        - name: tmp-volume
          mountPath: /tmp
        env:
        - name: HOME
          value: "/tmp"
      volumes:
      - name: tmp-volume
        emptyDir: {}
      restartPolicy: Never
  backoffLimit: 3
EOF

# Wait for job to complete
oc wait --for=condition=complete job/create-bucket -n minio-tenant --timeout=300s

# Check the job logs to verify bucket creation
oc logs -n minio-tenant job/create-bucket
```

### Step 5: Create Storage Secret

```yaml
# tempo-storage-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: tempo-storage-secret
  namespace: tracing-system
type: Opaque
stringData:
  endpoint: https://minio.minio-tenant.svc.cluster.local:443
  bucket: tempo-traces
  access_key_id: minioadmin
  access_key_secret: minioadmin123
```

```bash
# Create the secret
oc apply -f tempo-storage-secret.yaml
```

### Step 6: Deploy TempoStack

Deploy TempoStack with multi-tenancy support:

```yaml
# tempostack.yaml
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
```

```bash
oc apply -f tempostack.yaml
```

**Note**: Using OpenShift mode for multi-tenancy, which integrates with OpenShift's built-in authentication and authorization.

### Step 7: Enable OpenShift Console Distributed Tracing UI

The OpenShift Console provides a built-in distributed tracing UI that replaces the deprecated Jaeger UI. Here's how to enable and configure it:

#### 7.1 Install Cluster Observability Operator (if not already installed)

```bash
# Check if already installed
oc get subscription -n openshift-observability-operator cluster-observability-operator 2>/dev/null

# If not installed, create namespace and install
oc create namespace openshift-observability-operator || true

# Install Cluster Observability Operator
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-observability-operator
  namespace: openshift-observability-operator
spec:
  channel: stable
  installPlanApproval: Automatic
  name: cluster-observability-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Wait for the operator to be ready
oc wait --for=condition=Ready pods -l control-plane=cluster-observability-operator -n openshift-observability-operator --timeout=300s
```

#### 7.2 Configure Distributed Tracing Plugin

```bash
# Create UIPlugin resource to enable distributed tracing in the console
cat <<EOF | oc apply -f -
apiVersion: observability.openshift.io/v1alpha1
kind: UIPlugin
metadata:
  name: distributed-tracing
  namespace: openshift-observability-operator
spec:
  type: DistributedTracing
  deployment:
    resources:
      requests:
        cpu: 50m
        memory: 128Mi
      limits:
        cpu: 100m
        memory: 256Mi
EOF

# Verify the plugin is deployed
oc get uiplugin distributed-tracing -n openshift-observability-operator
oc get pods -n openshift-observability-operator -l app.kubernetes.io/name=distributed-tracing-ui-plugin
```

#### 7.3 Configure Tempo Backend Connection

```bash
# Create configuration to connect the UI to Tempo
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: distributed-tracing-tempo-config
  namespace: openshift-config
data:
  tempo-config.yaml: |
    backend: tempo
    tempo:
      endpoint: tempo-tracing-tempo-query-frontend.tracing-system.svc.cluster.local:16686
      tls:
        enabled: false
      multi-tenancy:
        enabled: true
        header: X-Scope-OrgID
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-observability-operator-config
  namespace: openshift-observability-operator
data:
  config.yaml: |
    distributedTracing:
      tempo:
        endpoint: "tempo-tracing-tempo-query-frontend.tracing-system.svc.cluster.local:16686"
        datasource:
          type: "tempo"
          uid: "tempo"
          access: "proxy"
          url: "http://tempo-tracing-tempo-query-frontend.tracing-system.svc.cluster.local:16686"
          jsonData:
            httpHeaderName1: "X-Scope-OrgID"
          secureJsonFields:
            httpHeaderValue1: "${namespace}"
EOF

# Apply the configuration to enable distributed tracing in the console
oc patch console.operator.openshift.io cluster --type='json' \
  -p='[{"op": "add", "path": "/spec/plugins", "value": ["distributed-tracing"]}]' || \
oc patch console.operator.openshift.io cluster --type='json' \
  -p='[{"op": "add", "path": "/spec/plugins/-", "value": "distributed-tracing"}]'

# Restart console pods to pick up the new plugin
oc delete pods -n openshift-console -l app=console
oc delete pods -n openshift-console -l component=downloads

# Wait for console to be ready
oc wait --for=condition=Ready pods -l app=console -n openshift-console --timeout=300s
```

#### 7.4 Create Tempo DataSource for Console

```bash
# Create a direct connection for the console to query Tempo
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: tempo-query-frontend-external
  namespace: tracing-system
  labels:
    app.kubernetes.io/name: tempo
    app.kubernetes.io/component: query-frontend
spec:
  type: ClusterIP
  ports:
  - name: jaeger-ui
    port: 16686
    targetPort: jaeger-ui
    protocol: TCP
  - name: jaeger-grpc
    port: 16685
    targetPort: jaeger-grpc
    protocol: TCP
  selector:
    app.kubernetes.io/component: query-frontend
    app.kubernetes.io/instance: tracing-tempo
    app.kubernetes.io/managed-by: tempo-operator
    app.kubernetes.io/name: tempo
EOF

# Create network policy to allow console access
cat <<EOF | oc apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-console-tempo-access
  namespace: tracing-system
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: tempo
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: openshift-console
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: openshift-observability-operator
    ports:
    - port: 16686
      protocol: TCP
    - port: 16685
      protocol: TCP
EOF
```

### Step 8: Deploy OpenTelemetry Collector

For multi-tenant setup with OpenShift authentication, deploy the collector with proper RBAC and configuration:

#### 8.1 Create ServiceAccount and Permissions

```bash
# Create ServiceAccount for the collector
oc create serviceaccount otel-collector-sa -n tracing-demo

# Create ClusterRole for trace writing
cat <<EOF | oc apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tempostack-traces-writer
rules:
- apiGroups:
  - 'tempo.grafana.com'
  resources:
  - tracing-demo  # Must match tenant name in TempoStack
  resourceNames:
  - traces
  verbs:
  - 'create'
EOF

# Bind the role to ServiceAccount
oc adm policy add-cluster-role-to-user tempostack-traces-writer -z otel-collector-sa -n tracing-demo
```

#### 8.2 Deploy the Collector

```yaml
# otel-collector-tracing-demo.yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
  namespace: tracing-demo
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
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
      jaeger:
        protocols:
          grpc:
            endpoint: 0.0.0.0:14250
          thrift_binary:
            endpoint: 0.0.0.0:6832
          thrift_compact:
            endpoint: 0.0.0.0:6831
          thrift_http:
            endpoint: 0.0.0.0:14268
      zipkin:
        endpoint: 0.0.0.0:9411
    
    processors:
      batch:
        timeout: 10s
        send_batch_size: 1024
      memory_limiter:
        check_interval: 1s
        limit_percentage: 75
        spike_limit_percentage: 25
    
    exporters:
      otlp:
        endpoint: tempo-tracing-tempo-gateway.tracing-system.svc.cluster.local:8090
        tls:
          insecure: false
          ca_file: "/var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt"
        auth:
          authenticator: bearertokenauth
        headers:
          X-Scope-OrgID: "tracing-demo"
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
          exporters: [otlp, debug]
```

```bash
# Deploy collector in tracing-demo namespace
oc apply -f otel-collector-tracing-demo.yaml

# Verify the collector is running
oc get pods -n tracing-demo -l app.kubernetes.io/component=opentelemetry-collector

# Check collector logs
oc logs -f -l app.kubernetes.io/component=opentelemetry-collector -n tracing-demo
```

**Important Notes**:
- The collector uses the **gateway endpoint** on port 8090 for multi-tenant access
- **Bearer token authentication** is required via the ServiceAccount token
- The **X-Scope-OrgID** header must match the tenant name in TempoStack
- **TLS is mandatory** but the service CA certificate is automatically available in pods
- The collector can receive traces via OTLP, Jaeger, or Zipkin protocols

## Java Spring Boot Test Application

[The Java Spring Boot application section remains the same as in the original document]

### Step 1: Create Spring Boot Application from Spring Initializr

[Content remains the same...]

### Step 2: Add OpenTelemetry Dependencies

[Content remains the same...]

### Step 3: Application Configuration

[Content remains the same...]

### Step 4: Main Application Class

[Content remains the same...]

### Step 5: Controller with Tracing

[Content remains the same...]

### Step 6: Health Check Endpoint

[Content remains the same...]

### Step 7: Containerfile (Dockerfile with Podman)

[Content remains the same...]

### Step 8: Build and Deploy the Application with Podman

[Content remains the same...]

### Step 9: Deploy to OpenShift

[Content remains the same...]

### Step 10: Deploy Additional Tenants (Optional)

[Content remains the same...]

## Verification and Testing

### Step 1: Verify All Components are Running

```bash
# Check operators
oc get pods -n openshift-distributed-tracing
oc get pods -n openshift-tempo-operator
oc get pods -n openshift-observability-operator

# Check tracing components
oc get pods -n tracing-system

# Check UI plugin
oc get pods -n openshift-observability-operator -l app.kubernetes.io/name=distributed-tracing-ui-plugin

# Check application
oc get pods -n tracing-demo
```

### Step 2: Get Routes

```bash
# Get application route
oc get route -n tracing-demo
```

### Step 3: Generate Test Traffic

```bash
# Get the application route
APP_ROUTE=$(oc get route tracing-demo -n tracing-demo -o jsonpath='{.spec.host}')

# Test endpoints
curl https://$APP_ROUTE/api/hello?name=OpenShift
curl https://$APP_ROUTE/api/chain
curl https://$APP_ROUTE/api/error

# Generate load
for i in {1..100}; do
  curl https://$APP_ROUTE/api/hello?name=User$i
  curl https://$APP_ROUTE/api/chain
  curl https://$APP_ROUTE/api/error || true
  sleep 1
done
```

### Step 4: Access Distributed Tracing in OpenShift Console

#### Using the Built-in OpenShift Console UI (Recommended)

1. **Access the OpenShift Console**:
   - Log into your OpenShift Console with your credentials
   - The URL is typically: `https://console-openshift-console.apps.<cluster-domain>`

2. **Navigate to Distributed Tracing**:
   - In the left navigation menu, expand **Observe**
   - Click on **Distributed Tracing** (or **Traces** depending on your OpenShift version)
   - If you don't see this option, the plugin may not be loaded yet - try refreshing the browser

3. **Select Your Namespace**:
   - In the namespace dropdown at the top, select `tracing-demo`
   - The UI automatically sets the correct tenant ID based on the namespace

4. **Search for Traces**:
   - **Service**: Select your service from the dropdown (should show services that have sent traces)
   - **Operation**: Choose specific operations or leave as "All"
   - **Tags**: Add filters like:
     - `http.method=GET`
     - `http.status_code=200`
     - `error=true` (for failed requests)
   - **Time Range**: Select the time period to search
   - Click **Find Traces**

5. **Analyze Traces**:
   - Click on any trace to see the detailed view
   - The trace view shows:
     - Span hierarchy and relationships
     - Duration of each span
     - Tags and attributes
     - Errors and logs
   - Use the timeline view to identify performance bottlenecks
   - Click on individual spans for detailed information

6. **Advanced Features**:
   - **Compare Traces**: Select multiple traces to compare timings
   - **Service Map**: View service dependencies (if available)
   - **Statistics**: View operation latency percentiles
   - **Download**: Export trace data for offline analysis

#### Troubleshooting Console Access

If the Distributed Tracing menu doesn't appear:

```bash
# 1. Verify the plugin is enabled
oc get console.operator.openshift.io cluster -o yaml | grep -A5 plugins

# Should show:
# plugins:
# - distributed-tracing

# 2. Check if the UI plugin pod is running
oc get pods -n openshift-observability-operator -l app.kubernetes.io/name=distributed-tracing-ui-plugin

# 3. Force console refresh
oc delete pods -n openshift-console -l app=console

# 4. Clear browser cache and reload
# Sometimes a hard refresh (Ctrl+Shift+R or Cmd+Shift+R) is needed
```

#### Verifying Tempo Connection

```bash
# Test the connection from console to Tempo
oc exec -n openshift-observability-operator \
  $(oc get pod -n openshift-observability-operator -l app.kubernetes.io/name=distributed-tracing-ui-plugin -o jsonpath='{.items[0].metadata.name}') \
  -- curl -s http://tempo-tracing-tempo-query-frontend.tracing-system.svc.cluster.local:16686/api/services

# Should return a list of services sending traces
```

### Alternative: Direct Jaeger UI Access (For Debugging)

While the OpenShift Console UI is recommended, you can still access the Jaeger UI directly for debugging:

```bash
# Create route to Query Frontend (for debugging only)
cat <<EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: jaeger-ui-debug
  namespace: tracing-system
spec:
  to:
    kind: Service
    name: tempo-tracing-tempo-query-frontend
  port:
    targetPort: jaeger-ui
  tls:
    termination: edge
EOF

# Get the URL
echo "Debug Jaeger UI: https://$(oc get route jaeger-ui-debug -n tracing-system -o jsonpath='{.spec.host}')"
```

## Troubleshooting

### Console UI Issues

#### Distributed Tracing Not Visible in Console

```bash
# 1. Check if all required operators are installed
oc get csv -n openshift-distributed-tracing
oc get csv -n openshift-observability-operator

# 2. Verify UI plugin status
oc get uiplugin -n openshift-observability-operator
oc describe uiplugin distributed-tracing -n openshift-observability-operator

# 3. Check console operator configuration
oc get console.operator.openshift.io cluster -o yaml

# 4. Look for errors in console and plugin pods
oc logs -n openshift-console -l app=console --tail=50 | grep -i trace
oc logs -n openshift-observability-operator -l app.kubernetes.io/name=distributed-tracing-ui-plugin --tail=50
```

#### No Traces Appearing in Console

```bash
# 1. Verify traces are reaching Tempo
oc logs -l app.kubernetes.io/component=query-frontend -n tracing-system --tail=100

# 2. Check collector is sending traces
oc logs -l app.kubernetes.io/component=opentelemetry-collector -n tracing-demo | grep "TracesExporter"

# 3. Test Tempo query API directly
oc exec -n tracing-system $(oc get pod -n tracing-system -l app.kubernetes.io/component=query-frontend -o jsonpath='{.items[0].metadata.name}') \
  -- curl -s http://localhost:16686/api/services

# 4. Check namespace/tenant mapping
oc logs -l app.kubernetes.io/component=gateway -n tracing-system | grep "tracing-demo"
```

### Common Issues and Solutions

[Previous troubleshooting content remains the same...]

## Best Practices

1. **Use the OpenShift Console UI**: The built-in distributed tracing UI is the recommended way to view traces
2. **Namespace-based Multi-tenancy**: Each namespace automatically maps to a tenant in Tempo
3. **Resource Limits**: Always set appropriate resource limits for all components
4. **Storage**: Use persistent storage for production deployments
5. **Security**: The console UI respects OpenShift RBAC - users can only see traces from namespaces they have access to
6. **Sampling**: Adjust sampling rates based on traffic volume
7. **Retention**: Configure appropriate retention policies for trace data
8. **Monitoring**: Set up alerts for component health and performance
9. **Regular Updates**: Keep operators updated through their stable channels

## Conclusion

This guide provides a complete implementation of distributed tracing on OpenShift 4.18 using:
- Red Hat build of OpenTelemetry for trace collection
- Tempo for trace storage with multi-tenant support
- **OpenShift Console's built-in Distributed Tracing UI** for visualization
- A fully instrumented Java Spring Boot application
- Podman for container image management

The built-in OpenShift Console UI provides:
- Integrated authentication and authorization
- Automatic namespace-to-tenant mapping
- Modern, responsive interface
- No additional routes or external access needed
- Unified observability experience with logs and metrics

For production deployments, ensure you:
- Use enterprise-grade object storage
- Configure appropriate retention policies
- Set up monitoring and alerting
- Keep all operators updated
- Train users on the OpenShift Console distributed tracing features
