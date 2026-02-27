# Apache Flink on Kubernetes: Complete Deployment Guide

This guide provides an exhaustive reference for deploying Apache Flink on Kubernetes. It covers every supported deployment strategy, resource configuration, networking, storage, security, observability, and operational best practices.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Flink Execution Modes](#flink-execution-modes)
  - [Session Mode](#session-mode)
  - [Application Mode](#application-mode)
  - [Per-Job Mode (Deprecated)](#per-job-mode-deprecated)
- [Deployment Strategies](#deployment-strategies)
  - [Strategy 1: Standalone Kubernetes (Manual YAML)](#strategy-1-standalone-kubernetes-manual-yaml)
  - [Strategy 2: Native Kubernetes Mode](#strategy-2-native-kubernetes-mode)
  - [Strategy 3: Helm Chart Deployment](#strategy-3-helm-chart-deployment)
  - [Strategy 4: Flink Kubernetes Operator](#strategy-4-flink-kubernetes-operator)
- [High Availability on Kubernetes](#high-availability-on-kubernetes)
- [Storage and State Backend Configuration](#storage-and-state-backend-configuration)
- [Networking and Service Exposure](#networking-and-service-exposure)
- [Resource Management and Autoscaling](#resource-management-and-autoscaling)
- [Security Considerations](#security-considerations)
- [Observability: Metrics, Logging, and Tracing](#observability-metrics-logging-and-tracing)
- [Docker Image Management](#docker-image-management)
- [Operational Procedures](#operational-procedures)
- [Comparison of Deployment Strategies](#comparison-of-deployment-strategies)
- [Additional Resources](#additional-resources)

---

## Overview

Kubernetes has become the preferred platform for deploying Apache Flink in production. It provides:

- **Dynamic resource provisioning**: TaskManagers are created as Pods on demand and released when idle
- **Fault tolerance**: Kubernetes restarts failed Pods; Flink recovers state from checkpoints
- **Multi-tenancy**: Namespaces, RBAC, and resource quotas isolate workloads
- **Self-service infrastructure**: Teams deploy Flink jobs using standard Kubernetes tooling
- **Integration with ecosystem**: Prometheus, Grafana, Jaeger, OPA, Vault, etc.

Flink's integration with Kubernetes has evolved significantly:

| Version | Milestone |
|---------|-----------|
| Flink 1.9 | Basic standalone Kubernetes support |
| Flink 1.10 | Native Kubernetes mode (reactive resource management) |
| Flink 1.13 | Application mode on Kubernetes GA |
| Flink 1.14 | Fine-grained resource management |
| Flink 1.15 | Per-job mode deprecated; Application mode recommended |
| 2022 | Flink Kubernetes Operator v1.0 released (separate project) |
| 2023+ | Flink Kubernetes Operator v1.x — autoscaling, SQL jobs, HA improvements |

---

## Prerequisites

### Kubernetes Cluster Requirements

| Resource | Minimum | Recommended (Production) |
|----------|---------|--------------------------|
| Kubernetes version | 1.21+ | 1.26+ |
| Node CPU | 2 cores | 8+ cores per node |
| Node RAM | 4 GiB | 32+ GiB per node |
| Persistent storage | Optional (dev) | Required (HA + state) |
| DNS | CoreDNS | CoreDNS |

### Required Permissions

The Flink service account needs the following Kubernetes RBAC permissions for native mode and the Operator:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: flink-role
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "persistentvolumeclaims", "secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["flink.apache.org"]          # Only needed for Flink Operator
    resources: ["flinkdeployments", "flinksessionjobs"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

### Tools

```bash
# Verify prerequisites
kubectl version --client
helm version
java -version   # Java 11 or 17 recommended

# Download Flink distribution (for CLI operations)
wget https://archive.apache.org/dist/flink/flink-1.18.1/flink-1.18.1-bin-scala_2.12.tgz
tar xzf flink-1.18.1-bin-scala_2.12.tgz
export PATH=$PATH:$PWD/flink-1.18.1/bin
```

---

## Flink Execution Modes

Before choosing a deployment strategy, it is important to understand the three execution modes. These modes control **how user code is packaged and how cluster resources are allocated**. They are orthogonal to the underlying deployment strategy.

### Session Mode

In Session Mode, a **long-running Flink cluster** is created first. Multiple jobs are then submitted to the cluster and share its TaskManager slots.

```
┌─────────────────────────────────────────────────┐
│              Flink Session Cluster              │
│                                                 │
│  ┌──────────────┐    ┌────────────────────────┐ │
│  │  JobManager  │    │  TaskManagers (shared) │ │
│  │  (long-lived)│    │  [slot][slot][slot]    │ │
│  └──────────────┘    └────────────────────────┘ │
│                                                 │
│  Job A ──► submitted ──► running in slots       │
│  Job B ──► submitted ──► running in slots       │
│  Job C ──► submitted ──► running in slots       │
└─────────────────────────────────────────────────┘
```

**Characteristics:**
- Cluster lifetime is independent of any individual job
- Fast job submission (no cluster startup overhead)
- Jobs share resources — a failing job can affect resource availability for others
- In native Kubernetes mode, TaskManagers are still created dynamically per-slot-request
- User JAR is uploaded at submission time, not baked into the image

**Best for:** Development environments, ad-hoc analytics, short-lived jobs, job orchestrators (Airflow, Argo)

**Trade-offs:**
- Lower isolation between jobs
- Cluster may be idle (wasteful) or overloaded
- Harder to tune per-job JVM settings

### Application Mode

In Application Mode, a **dedicated Flink cluster is created per application**. The `main()` method of the user application is executed **inside the JobManager**, not on the client. This means the JAR is part of the Docker image or pre-loaded in the cluster.

```
┌────────────────────────────────────────────────────────┐
│               Flink Application Cluster               │
│  (One cluster per application; one job per cluster)   │
│                                                        │
│  ┌────────────────────────────────────────────┐        │
│  │  JobManager + Application main()          │        │
│  │  (executes user code, builds JobGraph)    │        │
│  └────────────────────────────────────────────┘        │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  TaskManagers (created dynamically per request)  │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

**Characteristics:**
- Cluster is created and destroyed with the job
- Full resource isolation — one job per cluster
- JAR is embedded in the container image (or mounted via volume/init container)
- No client-side dependency resolution; the cluster does everything
- Reduces client-side resource usage for complex job graphs

**Best for:** Production workloads, GitOps deployments, strict isolation requirements

**Trade-offs:**
- Higher startup overhead per job (but amortized for long-running streaming jobs)
- Requires image rebuild or dynamic artifact loading for each new version

### Per-Job Mode (Deprecated)

Per-Job Mode was an earlier mechanism where a dedicated cluster was spun up per job. The key difference from Application Mode is that the `main()` runs on the **client** side, producing a JobGraph that is then submitted to a freshly created cluster.

This mode is **deprecated since Flink 1.15** and removed from some backends. **Use Application Mode instead.**

---

## Deployment Strategies

### Strategy 1: Standalone Kubernetes (Manual YAML)

This is the most basic approach: you write and apply raw Kubernetes YAML manifests to deploy Flink's JobManager and TaskManager components.

#### Architecture

```
Kubernetes Cluster
├── Deployment: flink-jobmanager
│   └── Pod: flink-jobmanager-xxxx
│       └── Container: flink (runs jobmanager.sh)
├── Deployment: flink-taskmanager
│   └── Pod: flink-taskmanager-xxxx (replica count = N)
│       └── Container: flink (runs taskmanager.sh)
├── Service: flink-jobmanager (ClusterIP → ports 8081, 6123)
└── ConfigMap: flink-config
```

#### Step 1: ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: flink-config
  namespace: flink
data:
  flink-conf.yaml: |+
    jobmanager.rpc.address: flink-jobmanager
    jobmanager.rpc.port: 6123
    jobmanager.memory.process.size: 1600m
    taskmanager.memory.process.size: 1728m
    taskmanager.numberOfTaskSlots: 2
    parallelism.default: 2
    state.backend: rocksdb
    state.checkpoints.dir: s3://my-bucket/flink/checkpoints
    state.savepoints.dir: s3://my-bucket/flink/savepoints
    s3.access-key: ${S3_ACCESS_KEY}
    s3.secret-key: ${S3_SECRET_KEY}
  log4j-console.properties: |+
    rootLogger.level = INFO
    rootLogger.appenderRef.console.ref = ConsoleAppender
    appender.console.name = ConsoleAppender
    appender.console.type = CONSOLE
    appender.console.layout.type = PatternLayout
    appender.console.layout.pattern = %d{yyyy-MM-dd HH:mm:ss,SSS} %-5p %-60c %x - %m%n
```

#### Step 2: JobManager Deployment and Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flink-jobmanager
  namespace: flink
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flink
      component: jobmanager
  template:
    metadata:
      labels:
        app: flink
        component: jobmanager
    spec:
      serviceAccountName: flink-service-account
      containers:
        - name: jobmanager
          image: flink:1.18.1-scala_2.12-java11
          args: ["jobmanager"]
          ports:
            - containerPort: 6123    # RPC
              name: rpc
            - containerPort: 6124    # Blob server
              name: blob-server
            - containerPort: 8081    # Web UI / REST
              name: webui
          env:
            - name: FLINK_PROPERTIES
              valueFrom:
                configMapKeyRef:
                  name: flink-config
                  key: flink-conf.yaml
          resources:
            requests:
              cpu: "500m"
              memory: "1600Mi"
            limits:
              cpu: "2"
              memory: "2Gi"
          livenessProbe:
            httpGet:
              path: /overview
              port: 8081
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /overview
              port: 8081
            initialDelaySeconds: 20
            periodSeconds: 5
          volumeMounts:
            - name: flink-config-volume
              mountPath: /opt/flink/conf
      volumes:
        - name: flink-config-volume
          configMap:
            name: flink-config
            items:
              - key: flink-conf.yaml
                path: flink-conf.yaml
              - key: log4j-console.properties
                path: log4j-console.properties
---
apiVersion: v1
kind: Service
metadata:
  name: flink-jobmanager
  namespace: flink
spec:
  selector:
    app: flink
    component: jobmanager
  ports:
    - name: rpc
      port: 6123
      targetPort: 6123
    - name: blob-server
      port: 6124
      targetPort: 6124
    - name: webui
      port: 8081
      targetPort: 8081
  type: ClusterIP
```

#### Step 3: TaskManager Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flink-taskmanager
  namespace: flink
spec:
  replicas: 2                      # Manually scaled
  selector:
    matchLabels:
      app: flink
      component: taskmanager
  template:
    metadata:
      labels:
        app: flink
        component: taskmanager
    spec:
      serviceAccountName: flink-service-account
      containers:
        - name: taskmanager
          image: flink:1.18.1-scala_2.12-java11
          args: ["taskmanager"]
          ports:
            - containerPort: 6122    # Task RPC
              name: rpc
            - containerPort: 6125    # Query service
              name: query-state
          env:
            - name: FLINK_PROPERTIES
              valueFrom:
                configMapKeyRef:
                  name: flink-config
                  key: flink-conf.yaml
          resources:
            requests:
              cpu: "1"
              memory: "1728Mi"
            limits:
              cpu: "4"
              memory: "3Gi"
          livenessProbe:
            tcpSocket:
              port: 6122
            initialDelaySeconds: 30
            periodSeconds: 60
          volumeMounts:
            - name: flink-config-volume
              mountPath: /opt/flink/conf
            - name: flink-tmp
              mountPath: /tmp/flink-rocksdb   # Local RocksDB state
      volumes:
        - name: flink-config-volume
          configMap:
            name: flink-config
        - name: flink-tmp
          emptyDir: {}
```

#### Step 4: Apply and Submit a Job

```bash
# Create namespace and apply all resources
kubectl create namespace flink
kubectl apply -f flink-config.yaml
kubectl apply -f flink-jobmanager.yaml
kubectl apply -f flink-taskmanager.yaml

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app=flink -n flink --timeout=120s

# Port-forward to access the Web UI locally
kubectl port-forward service/flink-jobmanager 8081:8081 -n flink &

# Submit a JAR job
flink run -m localhost:8081 path/to/application.jar --arg1 value1

# Or submit via kubectl exec
kubectl exec -n flink deployment/flink-jobmanager -- \
  flink run /opt/flink/examples/streaming/WordCount.jar
```

#### Limitations of This Approach

- **Manual scaling**: you must update `replicas` in the TaskManager Deployment
- **No slot-level scheduling**: Kubernetes doesn't understand Flink slots
- **No HA out of the box**: you must add ZooKeeper or Kubernetes ConfigMap HA separately
- **No automated lifecycle management**: job restarts, savepoints, and upgrades are manual

---

### Strategy 2: Native Kubernetes Mode

Native Kubernetes mode enables Flink to **directly communicate with the Kubernetes API server** to manage its own infrastructure. When a Flink job needs more slots, the JobManager creates new TaskManager Pods via the Kubernetes API. When slots are no longer needed, TaskManager Pods are terminated.

This eliminates the need to pre-provision TaskManagers — you no longer define a TaskManager Deployment. Flink becomes a **Kubernetes-native application** that manages its own resources dynamically.

#### How It Works

```
flink run-application -t kubernetes-application
           │
           ▼
   ┌──────────────────────────┐
   │  Flink Client            │
   │  (compiles JobGraph)     │
   │  Calls: kubectl apply    │
   └──────────┬───────────────┘
              │ Creates:
              ▼
   ┌──────────────────────────┐      ┌─────────────────────────────┐
   │  JobManager Pod          │─────►│  Kubernetes API Server      │
   │  (main() executes here   │      │  (Flink's ResourceManager   │
   │   in Application Mode)   │◄─────│   calls it to create Pods)  │
   └──────────────────────────┘      └─────────────────────────────┘
              │ Requests slots
              ▼
   ┌──────────────────────────┐
   │  TaskManager Pod 1       │  ← Created on-demand by Flink
   │  TaskManager Pod 2       │  ← Created on-demand by Flink
   │  TaskManager Pod N       │  ← Destroyed when idle
   └──────────────────────────┘
```

#### Session Cluster in Native Mode

```bash
# Start a native Kubernetes session cluster
./bin/kubernetes-session.sh \
  -Dkubernetes.cluster-id=my-flink-session \
  -Dkubernetes.namespace=flink \
  -Dkubernetes.container.image=my-registry/flink:1.18.1 \
  -Djobmanager.memory.process.size=1g \
  -Dtaskmanager.memory.process.size=2g \
  -Dresourcemanager.taskmanager-timeout=60s \
  -Dkubernetes.jobmanager.cpu=1.0 \
  -Dkubernetes.taskmanager.cpu=2.0 \
  -Dkubernetes.taskmanager.labels="app:flink,component:taskmanager"
```

After the session cluster is running, submit jobs to it:

```bash
# Submit job to existing session cluster
./bin/flink run \
  --target kubernetes-session \
  -Dkubernetes.cluster-id=my-flink-session \
  -Dkubernetes.namespace=flink \
  examples/streaming/TopSpeedWindowing.jar
```

#### Application Cluster in Native Mode

```bash
# Submit as a dedicated Application Mode cluster
./bin/flink run-application \
  --target kubernetes-application \
  -Dkubernetes.cluster-id=my-app-cluster \
  -Dkubernetes.namespace=flink \
  -Dkubernetes.container.image=my-registry/my-flink-app:1.0.0 \
  -Djobmanager.memory.process.size=1g \
  -Dtaskmanager.memory.process.size=2g \
  -Dkubernetes.taskmanager.cpu=2.0 \
  -Dkubernetes.jobmanager.service-account=flink-service-account \
  -Dstate.backend=rocksdb \
  -Dstate.checkpoints.dir=s3://my-bucket/checkpoints \
  local:///opt/flink/usrlib/my-application.jar
```

> The `local://` scheme tells Flink the JAR is already present inside the container image at that path.

#### Key Native Kubernetes Configuration Properties

| Property | Description | Example |
|----------|-------------|---------|
| `kubernetes.cluster-id` | Unique name for the Flink cluster (used as label/prefix) | `my-flink-app` |
| `kubernetes.namespace` | Kubernetes namespace | `flink-prod` |
| `kubernetes.container.image` | Docker image for JobManager and TaskManagers | `my-registry/flink:1.18` |
| `kubernetes.jobmanager.cpu` | CPU request for JobManager Pod | `1.0` |
| `kubernetes.taskmanager.cpu` | CPU request per TaskManager Pod | `4.0` |
| `kubernetes.jobmanager.memory.limit-factor` | Memory limit = process size × factor | `1.5` |
| `kubernetes.taskmanager.memory.limit-factor` | Same for TaskManager | `1.5` |
| `kubernetes.service-account` | Service account for Flink Pods | `flink-sa` |
| `kubernetes.pod-template-file` | Path to a Pod template YAML for customization | `/opt/flink/pod-template.yaml` |
| `kubernetes.rest-service.exposed.type` | How to expose the REST service | `ClusterIP`, `NodePort`, `LoadBalancer` |
| `kubernetes.taskmanager.node-selector` | Node affinity via labels | `gpu: "true"` |
| `kubernetes.taskmanager.tolerations` | Toleration for tainted nodes | `[{key: gpu, effect: NoSchedule}]` |
| `resourcemanager.taskmanager-timeout` | Time before idle TaskManager is reclaimed | `60s` |

#### Pod Templates

Pod templates allow you to inject sidecar containers, custom volumes, annotations, and any Kubernetes-specific configuration that Flink's properties don't natively expose:

```yaml
# pod-template.yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9249"
spec:
  serviceAccountName: flink-service-account
  nodeSelector:
    workload: flink
  tolerations:
    - key: "flink"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  containers:
    - name: flink-main-container    # This name is required by Flink
      env:
        - name: AWS_REGION
          value: us-east-1
      volumeMounts:
        - name: aws-credentials
          mountPath: /var/secrets/aws
          readOnly: true
    - name: prometheus-exporter     # Sidecar for metrics
      image: prom/statsd-exporter:v0.24.0
      ports:
        - containerPort: 9249
          name: metrics
  volumes:
    - name: aws-credentials
      secret:
        secretName: aws-flink-credentials
  initContainers:
    - name: download-jars
      image: alpine/curl:latest
      command: ["/bin/sh", "-c"]
      args:
        - curl -o /flink-usrlib/connector.jar https://artifact-repo/connector-1.0.jar
      volumeMounts:
        - name: flink-usrlib
          mountPath: /flink-usrlib
```

Use the template:

```bash
./bin/flink run-application \
  --target kubernetes-application \
  -Dkubernetes.pod-template-file=/path/to/pod-template.yaml \
  -Dkubernetes.jobmanager.pod-template-file=/path/to/jm-template.yaml \
  -Dkubernetes.taskmanager.pod-template-file=/path/to/tm-template.yaml \
  # ... other options
```

---

### Strategy 3: Helm Chart Deployment

Helm is the package manager for Kubernetes and provides a templated, version-controlled way to deploy Flink clusters. While there is no single "official" Flink Helm chart in the Flink project itself, several widely used charts exist.

#### Option A: Official Flink Docker + Community Helm Chart

The [flink-kubernetes-operator](https://github.com/apache/flink-kubernetes-operator) project maintains a Helm chart for the **Operator** (covered in Strategy 4). For a standalone Flink cluster, community charts like the Bitnami chart or custom organizational charts are commonly used.

#### Option B: Custom Helm Chart

Here is a production-quality custom Helm chart structure for deploying a Flink Application Mode cluster:

```
flink-app/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── configmap.yaml
│   ├── serviceaccount.yaml
│   ├── rbac.yaml
│   ├── jobmanager-deployment.yaml
│   ├── jobmanager-service.yaml
│   ├── taskmanager-deployment.yaml
│   └── ingress.yaml
```

**Chart.yaml:**

```yaml
apiVersion: v2
name: flink-app
description: Apache Flink Application Mode Deployment
type: application
version: 0.1.0
appVersion: "1.18.1"
```

**values.yaml:**

```yaml
image:
  repository: my-registry/my-flink-app
  tag: "1.0.0"
  pullPolicy: IfNotPresent

flinkVersion: "1.18.1"

jobmanager:
  replicaCount: 1
  resources:
    requests:
      cpu: 500m
      memory: 1600Mi
    limits:
      cpu: 2
      memory: 2Gi
  service:
    type: ClusterIP
    restPort: 8081
    rpcPort: 6123
    blobPort: 6124

taskmanager:
  replicaCount: 2
  numberOfTaskSlots: 2
  resources:
    requests:
      cpu: 1
      memory: 1728Mi
    limits:
      cpu: 4
      memory: 3Gi

flink:
  conf:
    taskmanager.numberOfTaskSlots: "2"
    parallelism.default: "2"
    state.backend: "rocksdb"
    state.checkpoints.dir: "s3://my-bucket/checkpoints"
    high-availability: "kubernetes"
    high-availability.storageDir: "s3://my-bucket/ha"
    kubernetes.cluster-id: "my-app"

serviceAccount:
  create: true
  name: flink-sa

ingress:
  enabled: false
  host: flink.example.com

metrics:
  enabled: true
  port: 9249

nodeSelector: {}
tolerations: []
affinity: {}
```

**templates/configmap.yaml:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "flink-app.fullname" . }}-config
data:
  flink-conf.yaml: |
    jobmanager.rpc.address: {{ include "flink-app.fullname" . }}-jobmanager
    jobmanager.rpc.port: {{ .Values.jobmanager.service.rpcPort }}
    {{- range $key, $value := .Values.flink.conf }}
    {{ $key }}: {{ $value | quote }}
    {{- end }}
```

**Deploy with Helm:**

```bash
# Add any required repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install the chart
helm install my-flink-app ./flink-app \
  --namespace flink-prod \
  --create-namespace \
  --values ./my-override-values.yaml

# Upgrade (rolling update)
helm upgrade my-flink-app ./flink-app \
  --namespace flink-prod \
  --set image.tag=2.0.0

# Check release status
helm status my-flink-app -n flink-prod

# Rollback on failure
helm rollback my-flink-app 1 -n flink-prod

# Uninstall
helm uninstall my-flink-app -n flink-prod
```

#### Using Helm with GitOps (ArgoCD / Flux)

```yaml
# ArgoCD Application manifest
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: flink-my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/flink-charts
    targetRevision: main
    path: flink-app
    helm:
      values: |
        image:
          tag: "1.2.3"
        taskmanager:
          replicaCount: 4
  destination:
    server: https://kubernetes.default.svc
    namespace: flink-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

### Strategy 4: Flink Kubernetes Operator

The [Flink Kubernetes Operator](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-stable/) is an official Apache Flink sub-project that provides a Kubernetes-native way to manage Flink deployments using **Custom Resource Definitions (CRDs)**. It is the **recommended approach for production deployments**.

> This strategy is covered in exhaustive detail in the [Flink Kubernetes Operator Guide](./flink-operator.md).

**Summary of how it works:**

```bash
# Install the operator via Helm
helm repo add flink-operator-repo https://downloads.apache.org/flink/flink-kubernetes-operator-1.9.0/
helm install flink-kubernetes-operator flink-operator-repo/flink-kubernetes-operator \
  --namespace flink-operator \
  --create-namespace

# Deploy a Flink application using a CRD
kubectl apply -f - <<EOF
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  name: my-streaming-app
  namespace: flink-prod
spec:
  image: my-registry/my-flink-app:1.0.0
  flinkVersion: v1_18
  flinkConfiguration:
    taskmanager.numberOfTaskSlots: "2"
    state.checkpoints.dir: s3://my-bucket/checkpoints
    high-availability: kubernetes
  serviceAccount: flink-sa
  jobManager:
    resource:
      memory: "2048m"
      cpu: 1
  taskManager:
    resource:
      memory: "2048m"
      cpu: 2
  job:
    jarURI: local:///opt/flink/usrlib/my-app.jar
    parallelism: 4
    upgradeMode: stateful
    state: running
EOF
```

The Operator watches for `FlinkDeployment` resources and reconciles the actual Kubernetes state toward the desired state, handling lifecycle management, HA, upgrades, savepoints, and autoscaling automatically.

---

## High Availability on Kubernetes

Without HA, the loss of the JobManager Pod means the entire job must restart from the beginning. Flink supports two HA providers on Kubernetes:

### ZooKeeper-Based HA (Legacy)

ZooKeeper stores the JobManager leader election and job metadata:

```yaml
# flink-conf.yaml additions
high-availability: zookeeper
high-availability.zookeeper.quorum: zookeeper.flink.svc.cluster.local:2181
high-availability.storageDir: s3://my-bucket/flink/ha
high-availability.zookeeper.path.root: /flink
high-availability.cluster-id: /my-app-cluster
```

Deploy ZooKeeper:

```bash
helm install zookeeper bitnami/zookeeper \
  --set replicaCount=3 \
  --namespace flink
```

### Kubernetes-Native HA (Recommended)

Since Flink 1.12, Flink can use **Kubernetes ConfigMaps and Leader Election** (via the `leases.coordination.k8s.io` API) instead of ZooKeeper:

```yaml
# flink-conf.yaml
high-availability: kubernetes
high-availability.storageDir: s3://my-bucket/flink/ha
kubernetes.cluster-id: my-flink-cluster
```

**How it works:**
1. All JobManager replicas compete to acquire a Kubernetes **Lease** object as leader
2. The leader stores the JobGraph, checkpoint metadata, and running job state in **ConfigMaps**
3. On leader failure, Kubernetes automatically invalidates the Lease; a standby becomes the new leader and reads state from ConfigMaps
4. The new leader restores jobs from the latest checkpoint

**JobManager Deployment for HA:**

```yaml
spec:
  replicas: 2          # Active + 1 standby
  template:
    spec:
      containers:
        - name: jobmanager
          args: ["jobmanager"]
          env:
            - name: FLINK_PROPERTIES
              value: |
                high-availability: kubernetes
                high-availability.storageDir: s3://my-bucket/flink/ha
                kubernetes.cluster-id: my-cluster
                # Required RBAC: get/update/create leases.coordination.k8s.io
```

> **Important**: For Kubernetes HA, the service account must have `get`, `create`, and `update` permissions on `leases.coordination.k8s.io` resources.

---

## Storage and State Backend Configuration

### Checkpoint Storage

Checkpoints must be written to **shared, durable storage** accessible by all Pods:

| Storage | Configuration | Notes |
|---------|--------------|-------|
| Amazon S3 | `s3://bucket/path` | Requires `flink-s3-fs-hadoop` or `flink-s3-fs-presto` plugin |
| GCS | `gs://bucket/path` | Requires `flink-gs-fs-hadoop` plugin |
| Azure ADLS | `abfs://container@account.dfs.core.windows.net/path` | Requires `flink-azure-fs-hadoop` plugin |
| HDFS | `hdfs://namenode:8020/path` | Requires HDFS access |
| Kubernetes PVC | Mount a ReadWriteMany PVC | NFS or similar; less recommended |

**S3 configuration example:**

```yaml
# flink-conf.yaml
state.checkpoints.dir: s3://my-flink-bucket/checkpoints
state.savepoints.dir: s3://my-flink-bucket/savepoints
high-availability.storageDir: s3://my-flink-bucket/ha
s3.endpoint: https://s3.us-east-1.amazonaws.com
s3.path.style.access: false
# For IRSA (IAM Roles for Service Accounts) - no keys needed:
# Annotate service account with: eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/ROLE
```

**Loading the S3 filesystem plugin:**

The S3 plugin JARs must be available in the Flink plugins directory. In the official Docker image:

```dockerfile
FROM flink:1.18.1-scala_2.12-java11
# Enable S3 filesystem
RUN mkdir -p /opt/flink/plugins/s3-fs-presto && \
    cp /opt/flink/opt/flink-s3-fs-presto-*.jar /opt/flink/plugins/s3-fs-presto/
```

### RocksDB State with Local SSDs

For large-state jobs, mount a fast local SSD for RocksDB:

```yaml
spec:
  template:
    spec:
      containers:
        - name: taskmanager
          volumeMounts:
            - name: rocksdb-state
              mountPath: /flink-rocksdb
          env:
            - name: FLINK_PROPERTIES
              value: |
                state.backend: rocksdb
                state.backend.rocksdb.localdir: /flink-rocksdb
                state.backend.incremental: true
      volumes:
        - name: rocksdb-state
          hostPath:             # Use actual NVMe SSD on the node
            path: /mnt/nvme/flink-rocksdb
            type: DirectoryOrCreate
```

For Cloud environments, use an `emptyDir` with `medium: ""` or a PVC backed by an SSD StorageClass:

```yaml
volumes:
  - name: rocksdb-state
    emptyDir:
      sizeLimit: 50Gi
```

---

## Networking and Service Exposure

### Internal Service Topology

A typical Flink deployment on Kubernetes has the following services:

| Service | Type | Ports | Purpose |
|---------|------|-------|---------|
| `<cluster-id>-rest` | ClusterIP or LoadBalancer | 8081 | REST API, Web UI |
| `<cluster-id>` | ClusterIP (headless) | 6123, 6124 | RPC, Blob server |

### Exposing the Web UI

**Option 1: Port-forwarding (development)**

```bash
kubectl port-forward service/flink-jobmanager 8081:8081 -n flink
# Open http://localhost:8081
```

**Option 2: NodePort**

```yaml
spec:
  type: NodePort
  ports:
    - name: webui
      port: 8081
      targetPort: 8081
      nodePort: 30081
```

**Option 3: LoadBalancer (cloud)**

```yaml
spec:
  type: LoadBalancer
  ports:
    - name: webui
      port: 80
      targetPort: 8081
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
```

**Option 4: Ingress (recommended for production)**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: flink-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"    # Long-lived connections
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - flink.example.com
      secretName: flink-tls
  rules:
    - host: flink.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: flink-jobmanager
                port:
                  number: 8081
```

### Network Policies

Restrict Flink Pod communication to only what is required:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: flink-network-policy
  namespace: flink
spec:
  podSelector:
    matchLabels:
      app: flink
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: flink
      ports:
        - port: 6122   # TaskManager RPC
        - port: 6123   # JobManager RPC
        - port: 6124   # Blob service
    - from: []          # Allow REST from anywhere (or restrict further)
      ports:
        - port: 8081
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: flink
    - to: []            # Allow DNS, S3, Kafka etc.
      ports:
        - port: 53      # DNS
        - port: 443     # HTTPS (S3, cloud APIs)
        - port: 9092    # Kafka
```

---

## Resource Management and Autoscaling

### Pod Resource Requests and Limits

Flink's memory model maps directly to container resources:

```yaml
resources:
  requests:
    memory: "1728Mi"    # Should match taskmanager.memory.process.size
    cpu: "1"
  limits:
    memory: "2160Mi"    # 1.25× process size to accommodate JVM overhead bursts
    cpu: "4"
```

> **Warning**: Setting memory limit equal to `memory.process.size` can cause OOMKills. Flink's process size configuration does not account for all JVM overhead. Use a 1.25–1.5× multiplier for limits.

### Horizontal Pod Autoscaling (HPA)

You can scale TaskManagers with HPA based on CPU/memory, but this is a crude mechanism — it scales Pods, not Flink slots, and is unaware of job demand. **The Flink Operator's autoscaler is significantly more intelligent** (see [Flink Operator Guide](./flink-operator.md)).

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: flink-taskmanager-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: flink-taskmanager
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### Vertical Pod Autoscaling (VPA)

VPA can automatically adjust container resource requests based on observed usage — useful for right-sizing Flink Pods:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: flink-taskmanager-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: flink-taskmanager
  updatePolicy:
    updateMode: "Auto"    # Or "Off" for recommendations only
  resourcePolicy:
    containerPolicies:
      - containerName: taskmanager
        minAllowed:
          cpu: 500m
          memory: 1Gi
        maxAllowed:
          cpu: 8
          memory: 16Gi
```

### KEDA-Based Autoscaling

[KEDA (Kubernetes Event-Driven Autoscaling)](https://keda.sh/) can scale Flink TaskManagers based on external metrics like Kafka consumer lag:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: flink-kafka-scaler
spec:
  scaleTargetRef:
    name: flink-taskmanager
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: flink-consumer-group
        topic: my-input-topic
        lagThreshold: "50000"
        offsetResetPolicy: latest
```

### Node Selection and Affinity

Pin Flink workloads to specific node pools:

```yaml
# NodeSelector (simple)
nodeSelector:
  workload-type: flink

# Node Affinity (more expressive)
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: cloud.google.com/gke-nodepool
              operator: In
              values: [flink-pool]
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              component: taskmanager
          topologyKey: kubernetes.io/hostname   # Spread TMs across nodes
```

---

## Security Considerations

### RBAC

Follow the principle of least privilege. The Flink service account for **Application Mode** needs:

```yaml
rules:
  # For Kubernetes HA
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["get", "watch", "list", "delete", "update", "create"]
  # For native K8s resource management
  - apiGroups: [""]
    resources: ["pods", "configmaps", "services"]
    verbs: ["get", "watch", "list", "create", "delete", "update"]
  # For blob server and logs
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get", "list"]
```

### Secrets Management

Never store credentials in ConfigMaps. Use Kubernetes Secrets or an external secrets manager:

```yaml
# Inject credentials via environment variables from Secrets
env:
  - name: AWS_ACCESS_KEY_ID
    valueFrom:
      secretKeyRef:
        name: aws-credentials
        key: access-key-id
  - name: AWS_SECRET_ACCESS_KEY
    valueFrom:
      secretKeyRef:
        name: aws-credentials
        key: secret-access-key
```

**Preferred approach: IRSA / Workload Identity** — Annotate the Flink service account with a cloud IAM role ARN so Pods receive temporary credentials without static keys:

```yaml
# AWS IRSA
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flink-sa
  namespace: flink-prod
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/flink-s3-role
```

### Network Security

- Use **Network Policies** to restrict Pod-to-Pod communication
- Enable **TLS** on the REST API using Flink's SSL configuration:

```yaml
# flink-conf.yaml
security.ssl.rest.enabled: true
security.ssl.rest.keystore: /var/secrets/keystore.jks
security.ssl.rest.keystore-password: ${KEYSTORE_PASSWORD}
security.ssl.rest.truststore: /var/secrets/truststore.jks
security.ssl.rest.truststore-password: ${TRUSTSTORE_PASSWORD}
```

### Pod Security

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 9999
  fsGroup: 9999
  seccompProfile:
    type: RuntimeDefault

containers:
  - name: flink
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: false   # Flink needs to write temp files
      capabilities:
        drop: ["ALL"]
```

---

## Observability: Metrics, Logging, and Tracing

### Metrics

Flink exposes rich metrics through its metrics system. On Kubernetes, the recommended approach is **Prometheus**:

**1. Configure the Prometheus metrics reporter in flink-conf.yaml:**

```yaml
metrics.reporters: prom
metrics.reporter.prom.factory.class: org.apache.flink.metrics.prometheus.PrometheusReporterFactory
metrics.reporter.prom.port: 9249
```

**2. Annotate Pods for Prometheus scraping:**

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9249"
    prometheus.io/path: "/metrics"
```

**3. Add PrometheusReporter JAR to plugins:**

```dockerfile
RUN mkdir -p /opt/flink/plugins/metrics-prometheus && \
    cp /opt/flink/opt/flink-metrics-prometheus-*.jar \
       /opt/flink/plugins/metrics-prometheus/
```

**Key Flink metrics to monitor:**

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `flink_jobmanager_job_uptime` | Job uptime in ms | Alert if drops to 0 (restarted) |
| `flink_jobmanager_job_numRestarts` | Number of job restarts | Alert if > 0 in 5 min |
| `flink_taskmanager_job_task_numRecordsInPerSecond` | Input throughput | Drop may indicate upstream issue |
| `flink_taskmanager_job_task_numRecordsOutPerSecond` | Output throughput | Should match input |
| `flink_taskmanager_job_task_isBackPressured` | Backpressure ratio | Alert if > 0.5 for > 5 min |
| `flink_jobmanager_job_lastCheckpointDuration` | Last checkpoint duration | Alert if > 5 minutes |
| `flink_jobmanager_job_lastCheckpointSize` | Last checkpoint size | Track for growth trends |
| `flink_taskmanager_Status_JVM_Memory_Heap_Used` | Heap memory usage | Alert if > 80% of max |
| `flink_taskmanager_Status_JVM_GC_*_Time` | GC time | Alert if GC overhead > 10% |

**Grafana Dashboard**: The Flink community maintains official Grafana dashboard JSON files at [flink-metrics-grafana](https://github.com/apache/flink/tree/master/flink-metrics/flink-metrics-prometheus).

### Logging

Flink uses **Log4j2** by default. Configure structured JSON logging for production:

```properties
# log4j2.properties
appender.console.type = Console
appender.console.name = ConsoleAppender
appender.console.layout.type = JsonTemplateLayout
appender.console.layout.eventTemplateUri = classpath:EcsLayout.json

rootLogger.level = INFO
rootLogger.appenderRef.console.ref = ConsoleAppender

# Reduce noise from verbose loggers
logger.flink.name = org.apache.flink
logger.flink.level = INFO
logger.kafka.name = org.apache.kafka
logger.kafka.level = WARN
```

Collect logs with **Fluentd** or **Fluent Bit** as a DaemonSet, shipping to Elasticsearch or a cloud log service:

```yaml
# Fluent Bit ConfigMap snippet
[INPUT]
    Name              tail
    Tag               flink.*
    Path              /var/log/containers/flink-*.log
    Parser            docker
    Mem_Buf_Limit     5MB

[OUTPUT]
    Name  es
    Match flink.*
    Host  elasticsearch.logging.svc.cluster.local
    Port  9200
    Index flink-logs
```

### Distributed Tracing

For request tracing across Flink operators and external services, integrate with **OpenTelemetry**:

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
    <version>1.30.1</version>
</dependency>
```

Configure the OpenTelemetry Java agent as a JVM argument:

```yaml
env:
  - name: FLINK_ENV_JAVA_OPTS
    value: "-javaagent:/opt/otel/opentelemetry-javaagent.jar"
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://otel-collector.observability:4318"
  - name: OTEL_SERVICE_NAME
    value: "flink-my-app"
```

---

## Docker Image Management

### Using the Official Image

The official Flink Docker image is available on Docker Hub:

```
flink:1.18.1-scala_2.12-java11
flink:1.18.1-scala_2.12-java17
flink:1.18.1-java11          # Scala included
```

### Building a Custom Image

```dockerfile
FROM flink:1.18.1-scala_2.12-java11

# Enable filesystem plugins
RUN mkdir -p /opt/flink/plugins/s3-fs-presto \
             /opt/flink/plugins/metrics-prometheus && \
    cp /opt/flink/opt/flink-s3-fs-presto-*.jar \
       /opt/flink/plugins/s3-fs-presto/ && \
    cp /opt/flink/opt/flink-metrics-prometheus-*.jar \
       /opt/flink/plugins/metrics-prometheus/

# Add user application JAR (Application Mode)
COPY target/my-flink-app-1.0.0.jar /opt/flink/usrlib/

# Add additional connector JARs
COPY lib/flink-connector-kafka-*.jar /opt/flink/lib/

# Optional: custom flink-conf.yaml defaults
COPY config/flink-conf.yaml /opt/flink/conf/

# Switch to non-root user
USER flink
```

**Multi-stage build (production):**

```dockerfile
# Build stage
FROM maven:3.9.5-eclipse-temurin-17 AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline -q
COPY src ./src
RUN mvn package -DskipTests -q

# Runtime stage
FROM flink:1.18.1-java17
COPY --from=builder /build/target/my-app-*-shaded.jar /opt/flink/usrlib/my-app.jar
RUN mkdir -p /opt/flink/plugins/s3-fs-presto && \
    cp /opt/flink/opt/flink-s3-fs-presto-*.jar /opt/flink/plugins/s3-fs-presto/
USER flink
```

---

## Operational Procedures

### Submitting and Monitoring Jobs

```bash
# List running jobs via REST API
curl http://flink-jobmanager:8081/jobs

# Get job details
curl http://flink-jobmanager:8081/jobs/<job-id>

# Get job exceptions
curl http://flink-jobmanager:8081/jobs/<job-id>/exceptions

# Get job checkpoint info
curl http://flink-jobmanager:8081/jobs/<job-id>/checkpoints
```

### Triggering Savepoints

```bash
# Via CLI
flink savepoint <job-id> s3://my-bucket/savepoints -m flink-jobmanager:8081

# Via REST API
curl -X POST http://flink-jobmanager:8081/jobs/<job-id>/savepoints \
  -H "Content-Type: application/json" \
  -d '{"target-directory": "s3://my-bucket/savepoints", "cancel-job": false}'

# Check savepoint status
curl http://flink-jobmanager:8081/jobs/<job-id>/savepoints/<request-id>
```

### Upgrading a Flink Job

For stateful streaming jobs, always upgrade via savepoint:

```bash
# 1. Trigger savepoint and record the path
SAVEPOINT=$(flink savepoint <job-id> s3://my-bucket/savepoints -m localhost:8081 \
  | grep "Savepoint completed" | awk '{print $NF}')

# 2. Cancel the job
flink cancel <job-id> -m localhost:8081

# 3. Update the Docker image or JAR

# 4. Restart from savepoint
flink run-application \
  --target kubernetes-application \
  -Dkubernetes.cluster-id=my-app \
  -s $SAVEPOINT \
  local:///opt/flink/usrlib/my-app.jar
```

### Scaling a Session Cluster

```bash
# Scale TaskManagers up
kubectl scale deployment flink-taskmanager --replicas=6 -n flink

# Scale down (Flink will trigger checkpoint before releasing slots)
kubectl scale deployment flink-taskmanager --replicas=2 -n flink
```

### Draining a Node

When a Kubernetes node needs maintenance:

```bash
# Cordoning the node prevents new Pods from being scheduled
kubectl cordon <node-name>

# Drain the node (Flink will restart affected tasks from checkpoint)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --grace-period=120
```

---

## Comparison of Deployment Strategies

| Aspect | Standalone YAML | Native Kubernetes | Helm | Flink Operator |
|--------|----------------|-------------------|------|----------------|
| **Complexity** | Low (simple) | Medium | Medium | Medium-High |
| **Dynamic resource management** | ❌ Manual | ✅ Automatic | ❌ Manual | ✅ Automatic |
| **Lifecycle management** | ❌ Manual | ❌ Partial | ❌ Manual | ✅ Full |
| **Savepoint management** | ❌ Manual | ❌ Manual | ❌ Manual | ✅ Automatic |
| **High availability** | Manual config | Manual config | Manual config | ✅ Built-in |
| **Autoscaling** | HPA only | HPA + Flink reactive | HPA only | ✅ Intelligent autoscaler |
| **GitOps friendly** | ✅ | ❌ (CLI-driven) | ✅✅ | ✅✅ |
| **Multi-job management** | Per-cluster | Session or per-app | Per-chart | ✅ CRD per job |
| **Upgrade support** | Manual | Manual | Helm rollback | ✅ Stateful upgrades |
| **Best for** | Dev/learning | Advanced users | Packaging | Production |
| **Official support** | ✅ | ✅ | Community | ✅ Apache project |

---

## Additional Resources

- [Apache Flink Kubernetes Documentation](https://nightlies.apache.org/flink/flink-docs-stable/docs/deployment/resource-providers/native_kubernetes/)
- [Flink Kubernetes Operator Documentation](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-stable/)
- [Official Flink Docker Images](https://hub.docker.com/_/flink)
- [Flink on Kubernetes Examples](https://github.com/apache/flink-kubernetes-operator/tree/main/examples)
- [Flink Configuration Reference](https://nightlies.apache.org/flink/flink-docs-stable/docs/deployment/config/)
- [Flink Memory Configuration](https://nightlies.apache.org/flink/flink-docs-stable/docs/deployment/memory/mem_setup/)
- [Flink Kubernetes Operator Guide](./flink-operator.md)
- [Apache Flink Overview](./overview.md)
