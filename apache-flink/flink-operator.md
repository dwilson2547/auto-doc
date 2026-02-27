# Flink Kubernetes Operator: Complete Guide

The Flink Kubernetes Operator is an official Apache Flink sub-project that extends the Kubernetes API with custom resources for managing Apache Flink applications natively on Kubernetes. It is the **recommended production deployment approach** for Flink on Kubernetes as of 2022+.

## Table of Contents

- [Overview and Design Goals](#overview-and-design-goals)
- [Architecture](#architecture)
  - [Custom Resource Definitions](#custom-resource-definitions)
  - [Operator Controller Loop](#operator-controller-loop)
  - [Webhook Validation](#webhook-validation)
- [Installation](#installation)
  - [Prerequisites](#prerequisites)
  - [Install via Helm](#install-via-helm)
  - [Install with Custom Configuration](#install-with-custom-configuration)
  - [Verify Installation](#verify-installation)
- [FlinkDeployment Resource](#flinkdeployment-resource)
  - [Application Mode Deployment](#application-mode-deployment)
  - [Session Mode Deployment](#session-mode-deployment)
  - [Full FlinkDeployment Specification Reference](#full-flinkdeployment-specification-reference)
- [FlinkSessionJob Resource](#flinksessionjob-resource)
  - [Submitting Jobs to a Session Cluster](#submitting-jobs-to-a-session-cluster)
  - [Managing Multiple Session Jobs](#managing-multiple-session-jobs)
- [Job Lifecycle Management](#job-lifecycle-management)
  - [Starting and Stopping Jobs](#starting-and-stopping-jobs)
  - [Upgrade Modes](#upgrade-modes)
  - [Savepoint Management](#savepoint-management)
  - [Job Suspension and Resumption](#job-suspension-and-resumption)
- [High Availability Configuration](#high-availability-configuration)
- [Autoscaling with the Flink Autoscaler](#autoscaling-with-the-flink-autoscaler)
  - [How the Autoscaler Works](#how-the-autoscaler-works)
  - [Autoscaler Configuration](#autoscaler-configuration)
  - [Autoscaler Metrics and Observability](#autoscaler-metrics-and-observability)
  - [Autoscaler Standalone Mode](#autoscaler-standalone-mode)
- [SQL Jobs with FlinkDeployment](#sql-jobs-with-flinkdeployment)
- [Advanced Configuration](#advanced-configuration)
  - [Pod Templates](#pod-templates)
  - [Ingress Configuration](#ingress-configuration)
  - [Native Kubernetes Configuration Overrides](#native-kubernetes-configuration-overrides)
- [Operator Configuration Reference](#operator-configuration-reference)
- [Multi-Namespace Management](#multi-namespace-management)
- [GitOps Integration](#gitops-integration)
  - [ArgoCD](#argocd)
  - [Flux CD](#flux-cd)
- [Observability](#observability)
  - [Operator Metrics](#operator-metrics)
  - [Job Metrics](#job-metrics)
- [Upgrading the Operator](#upgrading-the-operator)
- [Troubleshooting](#troubleshooting)
- [Security](#security)
- [Additional Resources](#additional-resources)

---

## Overview and Design Goals

The Flink Kubernetes Operator follows the **Kubernetes Operator Pattern**: it encodes operational knowledge about running Flink applications into software, automating tasks that would otherwise require human intervention.

**Before the Operator:** Deploying and managing Flink on Kubernetes required:
- Manually writing and maintaining YAML for Deployments, Services, ConfigMaps
- Writing scripts to trigger savepoints before upgrades
- Manually recovering from failures
- Custom tooling for job lifecycle management

**With the Operator:** You declare the **desired state** of your Flink application in a `FlinkDeployment` custom resource, and the Operator continuously reconciles actual state to match:

```
User applies FlinkDeployment YAML
            │
            ▼
┌───────────────────────────────────┐
│    Flink Kubernetes Operator      │
│  ┌──────────────────────────────┐ │
│  │  Reconciliation Loop         │ │
│  │  1. Read desired state       │ │
│  │  2. Compare to actual state  │ │
│  │  3. Take corrective action   │ │
│  │     - Create/delete Pods     │ │
│  │     - Trigger savepoints     │ │
│  │     - Restart jobs           │ │
│  │     - Scale up/down          │ │
│  └──────────────────────────────┘ │
└───────────────────────────────────┘
            │
            ▼
   Flink Cluster on Kubernetes
   (JobManager + TaskManagers)
```

**Key capabilities:**
- **Declarative lifecycle management**: Start, stop, suspend, resume jobs via YAML
- **Stateful upgrades**: Automatically savepoint, upgrade, and restore jobs
- **Kubernetes-native HA**: Uses ConfigMaps and leader election; no ZooKeeper needed
- **Intelligent autoscaling**: Scales parallelism based on actual data throughput metrics
- **Multi-namespace support**: One Operator instance can manage multiple namespaces
- **SQL job support**: Deploy SQL scripts directly without writing Java/Scala
- **Validation webhooks**: Catch configuration errors at apply time, not runtime

---

## Architecture

### Custom Resource Definitions

The Operator installs two CRDs into the Kubernetes API:

| CRD | Kind | Purpose |
|-----|------|---------|
| `flinkdeployments.flink.apache.org` | `FlinkDeployment` | Manages a complete Flink cluster (Application or Session mode) |
| `flinksessionjobs.flink.apache.org` | `FlinkSessionJob` | Manages a single job submitted to a Session cluster |

These CRDs extend the Kubernetes API so that `kubectl get flinkdeployments` and `kubectl describe flinkdeployment my-app` work natively.

### Operator Controller Loop

The Operator runs as a standard Kubernetes **Deployment** (typically 1 replica, leader-elected for HA). Its controller loop:

1. **Watches** `FlinkDeployment` and `FlinkSessionJob` resources for changes (via Kubernetes watch API)
2. **Validates** the resource spec (additional validation beyond the webhook)
3. **Determines** required actions by comparing spec to current cluster state
4. **Executes** actions:
   - Creates/updates Kubernetes resources (Deployments, Services, ConfigMaps)
   - Calls Flink's REST API to submit jobs, trigger savepoints, cancel jobs
   - Updates the resource's `status` subresource with current state
5. **Re-queues** the resource for future reconciliation

The Operator uses the **Fabric8 Kubernetes client** under the hood and is built with the [Java Operator SDK](https://javaoperatorsdk.io/).

### Webhook Validation

The Operator deploys a **validating webhook** that intercepts `FlinkDeployment` and `FlinkSessionJob` create/update requests and validates them before they are persisted. This provides fast feedback on configuration errors:

```
kubectl apply -f my-flinkdeployment.yaml
        │
        ▼
Kubernetes API Server
        │  Calls webhook
        ▼
Flink Operator Webhook
  - Is flinkVersion valid?
  - Does upgradeMode match jobType?
  - Are required fields present?
  - Are resource values positive?
        │
   ✅ Valid → Persist resource → Trigger reconciliation
   ❌ Invalid → Reject with error message
```

---

## Installation

### Prerequisites

- Kubernetes 1.21+
- Helm 3.x
- Flink Kubernetes Operator requires a service account with appropriate RBAC permissions (created by the Helm chart)
- cert-manager (for webhook TLS) — either installed separately or disabled in favor of Helm-managed self-signed certificates

### Install via Helm

```bash
# Add the Flink Operator Helm repository
helm repo add flink-operator-repo \
  https://downloads.apache.org/flink/flink-kubernetes-operator-1.9.0/
helm repo update

# Create namespace for the operator
kubectl create namespace flink-operator

# Install with default settings
helm install flink-kubernetes-operator \
  flink-operator-repo/flink-kubernetes-operator \
  --namespace flink-operator

# Verify
kubectl get pods -n flink-operator
```

Expected output:

```
NAME                                         READY   STATUS    RESTARTS   AGE
flink-kubernetes-operator-6d7c9f7f5b-xk9v2  1/1     Running   0          30s
```

### Install with Custom Configuration

For production, override default values:

```yaml
# operator-values.yaml
operatorNamespace: flink-operator

# Number of operator replicas (use 2 for HA)
replicas: 2

# Operator image
image:
  repository: apache/flink-kubernetes-operator
  tag: 1.9.0

# RBAC: watch all namespaces or specific ones
watchNamespaces: []          # Empty = watch all namespaces
# watchNamespaces: [flink-prod, flink-staging]

# Operator configuration
operatorConfiguration:
  reconciler.reschedule.interval.ms: "15000"
  operator.reconcile.interval.seconds: "15"
  operator.observer.rest-ready.delay: "10s"

  # Autoscaler defaults
  job.autoscaler.enabled: "true"
  job.autoscaler.stabilization.interval: "5m"
  job.autoscaler.metrics.window: "10m"

# Default Flink configuration applied to all managed deployments
defaultConfiguration:
  create: true
  append: true    # Merge with job-level config
  flink-conf.yaml: |+
    kubernetes.operator.metrics.reporter.slf4j.factory.class: org.apache.flink.metrics.slf4j.Slf4jReporterFactory
    kubernetes.operator.metrics.reporter.slf4j.interval: 5 MINUTE
    metrics.reporters: prom
    metrics.reporter.prom.factory.class: org.apache.flink.metrics.prometheus.PrometheusReporterFactory
    metrics.reporter.prom.port: 9249

# Webhook
webhook:
  create: true

# cert-manager for TLS
certManager:
  enabled: true    # Set false if cert-manager is not installed
```

```bash
helm install flink-kubernetes-operator \
  flink-operator-repo/flink-kubernetes-operator \
  --namespace flink-operator \
  --create-namespace \
  --values operator-values.yaml
```

### Verify Installation

```bash
# Check operator pod
kubectl get pods -n flink-operator

# Check CRDs are registered
kubectl get crd | grep flink
# flinkdeployments.flink.apache.org
# flinksessionjobs.flink.apache.org

# Check operator logs
kubectl logs deployment/flink-kubernetes-operator -n flink-operator

# Check webhook is configured
kubectl get validatingwebhookconfigurations | grep flink
```

---

## FlinkDeployment Resource

`FlinkDeployment` is the primary CRD and describes a complete Flink cluster — either an **Application Mode** cluster running a single job, or a **Session Mode** cluster running multiple jobs.

### Application Mode Deployment

The most common production use case: one `FlinkDeployment` per streaming application.

```yaml
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  name: my-streaming-app
  namespace: flink-prod
spec:
  # Flink version (determines base image tag if image is not overridden)
  flinkVersion: v1_18

  # Docker image containing the user application JAR
  image: my-registry/my-flink-app:1.2.3
  imagePullPolicy: IfNotPresent

  # Shared Flink configuration for this deployment
  flinkConfiguration:
    taskmanager.numberOfTaskSlots: "2"
    state.backend: rocksdb
    state.backend.incremental: "true"
    state.checkpoints.dir: s3://my-bucket/checkpoints/my-streaming-app
    state.savepoints.dir: s3://my-bucket/savepoints/my-streaming-app
    execution.checkpointing.interval: "60000"
    execution.checkpointing.mode: EXACTLY_ONCE
    high-availability: kubernetes
    high-availability.storageDir: s3://my-bucket/ha/my-streaming-app
    kubernetes.operator.periodic.savepoint.interval: 6h

  # Kubernetes ServiceAccount for Flink Pods
  serviceAccount: flink-sa

  # JobManager configuration
  jobManager:
    resource:
      memory: "2048m"
      cpu: 1
    replicas: 1          # Set to 2 for standby HA

  # TaskManager configuration (per-TaskManager Pod)
  taskManager:
    resource:
      memory: "4096m"
      cpu: 2

  # The job to run (Application Mode only)
  job:
    # JAR path inside the container image
    jarURI: local:///opt/flink/usrlib/my-app.jar
    # Entry class (optional if jar has a manifest Main-Class)
    entryClass: com.example.MyStreamingJob
    # Command-line arguments for the job
    args: ["--kafka.bootstrap.servers", "kafka:9092", "--env", "prod"]
    # Job parallelism
    parallelism: 8
    # Desired job state
    state: running         # or: suspended
    # Upgrade mode (controls how state is handled on upgrade)
    upgradeMode: stateful  # or: savepoint, last-state, stateless
    # Allow empty state on initial startup
    allowNonRestoredState: false
    # Initial savepoint to restore from (optional)
    # initialSavepointPath: s3://my-bucket/savepoints/my-app/savepoint-abc123
```

Apply and observe:

```bash
kubectl apply -f my-streaming-app.yaml

# Watch deployment status
kubectl get flinkdeployment my-streaming-app -n flink-prod -w

# Get detailed status
kubectl describe flinkdeployment my-streaming-app -n flink-prod

# Check the operator reconciled the deployment
kubectl get pods -n flink-prod -l app=my-streaming-app
```

The `status` field is populated by the Operator:

```yaml
status:
  jobStatus:
    state: RUNNING
    jobId: a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4
    startTime: "2024-01-15T10:30:00Z"
    updateTime: "2024-01-15T10:31:05Z"
    savepointInfo:
      lastSavepoint:
        location: s3://my-bucket/savepoints/my-streaming-app/savepoint-xxx
        triggerType: PERIODIC
        triggerTimestamp: "2024-01-15T16:30:00Z"
        formatType: NATIVE
  lifecycleState: STABLE
  clusterInfo:
    flink-revision: 1.18.1
    total-cpu: "8.0"
    total-memory: "4294967296"
  reconciliationStatus:
    lastReconciledSpec: '...'
    reconciliationTimestamp: "2024-01-15T10:31:05Z"
    state: DEPLOYED
```

### Session Mode Deployment

A Session cluster is a long-running Flink cluster that accepts multiple job submissions. The `FlinkDeployment` configures the cluster; `FlinkSessionJob` resources submit individual jobs.

```yaml
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  name: flink-session-cluster
  namespace: flink-dev
spec:
  flinkVersion: v1_18
  image: flink:1.18.1-scala_2.12-java11

  flinkConfiguration:
    taskmanager.numberOfTaskSlots: "4"
    state.checkpoints.dir: s3://my-bucket/checkpoints/session
    high-availability: kubernetes
    high-availability.storageDir: s3://my-bucket/ha/session

  serviceAccount: flink-sa

  jobManager:
    resource:
      memory: "2048m"
      cpu: 1

  taskManager:
    resource:
      memory: "2048m"
      cpu: 2
    # Start with a fixed number of TaskManagers (can be scaled later)
    replicas: 3

  # No 'job' field for a session cluster
```

### Full FlinkDeployment Specification Reference

| Field | Type | Description |
|-------|------|-------------|
| `spec.flinkVersion` | string | Flink version: `v1_15`, `v1_16`, `v1_17`, `v1_18`, `v1_19` |
| `spec.image` | string | Docker image (defaults to official Flink image for version) |
| `spec.imagePullPolicy` | string | `Always`, `IfNotPresent`, `Never` |
| `spec.imagePullSecrets` | list | Secrets for private registries |
| `spec.serviceAccount` | string | Kubernetes service account name |
| `spec.flinkConfiguration` | map | Key-value pairs added to `flink-conf.yaml` |
| `spec.logConfiguration` | map | Key-value pairs for `log4j-console.properties` |
| `spec.podTemplate` | PodTemplateSpec | Applied to all Pods (JM + TM) |
| `spec.jobManager.resource.memory` | string | JM container memory (e.g., `2048m`, `2g`) |
| `spec.jobManager.resource.cpu` | number | JM container CPU request |
| `spec.jobManager.replicas` | integer | JM replicas (1 or 2 for HA standby) |
| `spec.jobManager.podTemplate` | PodTemplateSpec | Override pod template for JM only |
| `spec.taskManager.resource.memory` | string | TM container memory |
| `spec.taskManager.resource.cpu` | number | TM container CPU request |
| `spec.taskManager.replicas` | integer | Fixed TM count (Session mode; Application mode uses dynamic) |
| `spec.taskManager.podTemplate` | PodTemplateSpec | Override pod template for TM only |
| `spec.job.jarURI` | string | JAR URI (`local://`, `https://`, `s3://`) |
| `spec.job.entryClass` | string | Main class (optional) |
| `spec.job.args` | list | Job arguments |
| `spec.job.parallelism` | integer | Job parallelism |
| `spec.job.state` | string | `running` or `suspended` |
| `spec.job.upgradeMode` | string | `stateful`, `savepoint`, `last-state`, `stateless` |
| `spec.job.savepointTriggerNonce` | integer | Increment to trigger a savepoint |
| `spec.job.initialSavepointPath` | string | Savepoint to restore on first startup |
| `spec.job.allowNonRestoredState` | boolean | Skip unmappable operator state on restore |
| `spec.ingress` | IngressSpec | Ingress configuration |

---

## FlinkSessionJob Resource

`FlinkSessionJob` submits and manages a single Flink job within an existing `FlinkDeployment` Session cluster.

### Submitting Jobs to a Session Cluster

```yaml
apiVersion: flink.apache.org/v1beta1
kind: FlinkSessionJob
metadata:
  name: word-count-job
  namespace: flink-dev
spec:
  # Reference to the FlinkDeployment (Session cluster) to submit to
  deploymentName: flink-session-cluster

  job:
    jarURI: https://repo1.maven.org/maven2/org/apache/flink/flink-examples-streaming_2.12/1.18.1/flink-examples-streaming_2.12-1.18.1-WordCount.jar
    parallelism: 4
    upgradeMode: stateful
    state: running
    args: ["--input", "s3://my-bucket/input/", "--output", "s3://my-bucket/output/"]
```

```bash
kubectl apply -f word-count-job.yaml

# Check status
kubectl get flinksessionjob word-count-job -n flink-dev
```

### Managing Multiple Session Jobs

```bash
# List all session jobs
kubectl get flinksessionjob -n flink-dev

# Output:
# NAME             DEPLOYMENT NAME          JOB STATUS   AGE
# word-count-job   flink-session-cluster    RUNNING      2h
# etl-pipeline     flink-session-cluster    RUNNING      1h
# analytics-agg    flink-session-cluster    SUSPENDED    30m
```

Session jobs are independently managed — upgrading, suspending, or deleting one `FlinkSessionJob` does not affect the others.

**Upgrading a session job:**

```yaml
# Update the jarURI or other spec fields, then re-apply
spec:
  job:
    jarURI: https://repo/my-app-2.0.jar
    upgradeMode: savepoint    # Take savepoint, then restart with new JAR
    state: running
```

---

## Job Lifecycle Management

### Starting and Stopping Jobs

Control job state by changing `spec.job.state`:

```yaml
# Suspend (stop with savepoint)
spec:
  job:
    state: suspended
    upgradeMode: savepoint    # Save state before stopping

# Resume
spec:
  job:
    state: running
```

When a job is suspended with `upgradeMode: savepoint`, the Operator:
1. Triggers a savepoint via the Flink REST API
2. Waits for the savepoint to complete
3. Cancels the job
4. Scales down the TaskManager Pods (if in Application mode)
5. Records the savepoint path in the resource status

On resume, it restarts from that savepoint automatically.

### Upgrade Modes

The `upgradeMode` field is the most important configuration for production stateful jobs. It determines how state is handled when the `FlinkDeployment` spec changes (e.g., new image, new parallelism, new JAR).

| Mode | State Handling | Downtime | Use Case |
|------|---------------|----------|---------|
| `stateful` | Uses last successful checkpoint | Seconds (restart from checkpoint) | Default; fast restart |
| `savepoint` | Triggers a new savepoint before upgrade | Seconds to minutes | Safest; guaranteed state consistency |
| `last-state` | Uses last available checkpoint without graceful shutdown | Minimal | Fast upgrades when checkpoint is recent |
| `stateless` | No state restoration; starts fresh | None (but data replay needed) | Stateless jobs; initial deployments |

**`stateful` mode (default for production streaming jobs):**

```yaml
spec:
  job:
    upgradeMode: stateful
```

When a change is detected:
1. Operator saves the last checkpoint location
2. Cancels the running job (not gracefully — just cancels)
3. Updates cluster resources (new image, config, etc.)
4. Restarts job from the last checkpoint

This is the fastest upgrade path and works when:
- Checkpoints are frequent (< 60 seconds)
- The checkpoint is compatible with the new topology

**`savepoint` mode (safest for production):**

```yaml
spec:
  job:
    upgradeMode: savepoint
  flinkConfiguration:
    kubernetes.operator.savepoint.format.type: NATIVE   # or CANONICAL
    kubernetes.operator.savepoint.trigger.grace-period: 1h
```

When a change is detected:
1. Operator triggers a savepoint via REST API
2. Waits for savepoint to complete (with configurable timeout)
3. Cancels the job
4. Updates cluster resources
5. Restarts from the savepoint

Savepoints are stored at `state.savepoints.dir` and recorded in the resource status for auditing.

**`last-state` mode (fastest, lowest downtime):**

Uses whatever checkpoint is available without triggering a new one. Appropriate when:
- The job already has very frequent checkpoints
- Upgrade speed matters more than state guarantee
- You have end-to-end exactly-once from source (Kafka offsets in state)

### Savepoint Management

#### Manual Savepoint Trigger

To trigger a savepoint without upgrading, increment `savepointTriggerNonce`:

```yaml
spec:
  job:
    state: running
    savepointTriggerNonce: 1    # Change this value to trigger a new savepoint
```

Each time you apply a change to this field, the Operator triggers a savepoint.

#### Periodic Savepoints

Configure automatic periodic savepoints:

```yaml
flinkConfiguration:
  kubernetes.operator.periodic.savepoint.interval: 6h    # Every 6 hours
  kubernetes.operator.savepoint.history.max.count: "10"  # Keep last 10
  kubernetes.operator.savepoint.history.max.age: 24h     # Or keep 24h worth
```

#### Check Savepoint Status

```bash
kubectl describe flinkdeployment my-streaming-app -n flink-prod

# Look for savepointInfo in status:
# savepointInfo:
#   lastSavepoint:
#     location: s3://my-bucket/savepoints/savepoint-abc123
#     triggerType: PERIODIC
#     triggerTimestamp: "2024-01-15T12:00:00Z"
#   savepointHistory:
#     - location: s3://my-bucket/savepoints/savepoint-aaa111
#       triggerTimestamp: "2024-01-15T06:00:00Z"
#     - location: s3://my-bucket/savepoints/savepoint-bbb222
#       triggerTimestamp: "2024-01-15T00:00:00Z"
```

### Job Suspension and Resumption

Suspending a job gracefully stops it while preserving state for later resumption:

```yaml
# Suspend
spec:
  job:
    state: suspended
    upgradeMode: savepoint

# Resume
spec:
  job:
    state: running
    # upgradeMode: savepoint  ← will restore from the savepoint taken during suspension
```

**Use cases for suspension:**
- Scheduled maintenance windows
- Cost savings during off-peak hours
- Investigating job behavior (pause without losing state)
- Manual state inspection

---

## High Availability Configuration

The Flink Kubernetes Operator makes Kubernetes-native HA easy to configure:

```yaml
spec:
  flinkConfiguration:
    high-availability: kubernetes
    high-availability.storageDir: s3://my-bucket/ha/my-app

  # Optional: standby JobManager replica
  jobManager:
    replicas: 2
    resource:
      memory: "2048m"
      cpu: 1
```

**What happens during HA failover:**

1. The active JobManager Pod crashes (OOMKill, node failure, etc.)
2. Kubernetes detects the Pod failure and creates a replacement Pod
3. The replacement Pod starts and acquires the Kubernetes Lease (leader election)
4. The new leader reads the latest checkpoint path from the ConfigMap
5. The job resumes from the checkpoint — typically in 10–30 seconds

**RBAC requirements for Kubernetes HA:**

```yaml
rules:
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

---

## Autoscaling with the Flink Autoscaler

The Flink Kubernetes Operator includes a sophisticated **autoscaler** that dynamically adjusts job parallelism based on actual data throughput. Unlike CPU/memory-based HPA, the Flink autoscaler understands Flink's execution model and can scale individual operators independently.

### How the Autoscaler Works

The autoscaler operates on a **utilization-based** model:

1. **Collects metrics** from the Flink REST API over a configurable time window
2. **Computes utilization** per operator: `actual_throughput / max_throughput`
3. **Detects bottleneck operators** (those with utilization > target threshold)
4. **Calculates the optimal parallelism** for each operator based on the observed throughput
5. **Triggers a rescale** by updating `spec.job.parallelism` in the `FlinkDeployment`
6. **Waits for stabilization** before evaluating again

```
Kafka Source ──► Filter ──► Enrichment ──► Aggregation ──► Kafka Sink
   p=2             p=2         p=8             p=4            p=4

Autoscaler observes:
- Source: 70% utilized → keep p=2
- Filter: 45% utilized → scale down to p=1
- Enrichment: 95% utilized → scale up to p=12
- Aggregation: 60% utilized → keep p=4
- Sink: 72% utilized → keep p=4
```

The autoscaler scales individual operators within a job, not just the entire job uniformly.

### Autoscaler Configuration

Enable and configure the autoscaler in the `FlinkDeployment`:

```yaml
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  name: autoscaled-app
spec:
  flinkConfiguration:
    # Enable autoscaler
    job.autoscaler.enabled: "true"

    # ─── Core scaling parameters ───
    # Target operator utilization (0.0–1.0). Scale up when above, down when below
    job.autoscaler.target.utilization: "0.7"
    # Boundary around target before triggering scaling
    job.autoscaler.target.utilization.boundary: "0.1"
    # Time window for metric collection (longer = more stable decisions)
    job.autoscaler.metrics.window: "10m"
    # Minimum time between scaling operations (stabilization period)
    job.autoscaler.stabilization.interval: "5m"
    # Minimum time a scaling operation is considered valid before another
    job.autoscaler.scale-up.grace-period: "1h"

    # ─── Parallelism bounds ───
    job.autoscaler.min.parallelism: "1"
    job.autoscaler.max.parallelism: "64"

    # ─── Scale-down settings ───
    # Only scale down if utilization drops below this threshold
    job.autoscaler.scale-down.interval: "10m"
    # Maximum factor by which parallelism can be reduced in one step
    job.autoscaler.scale-down.max.factor: "0.6"
    # Factor by which to increase parallelism when scaling up
    job.autoscaler.scale-up.max.factor: "4.0"

    # ─── Restart strategy for rescaling ───
    # Use savepoint (safest) or last-state (fastest) for rescale restarts
    job.autoscaler.restart.time: "3m"

    # ─── Exclusions (don't autoscale specific operators) ───
    # job.autoscaler.vertex.exclude: "source-operator-uid,sink-operator-uid"

  job:
    jarURI: local:///opt/flink/usrlib/my-app.jar
    parallelism: 4        # Initial parallelism; autoscaler will adjust
    upgradeMode: stateful  # Used for rescaling restarts
    state: running
```

### Autoscaler Metrics and Observability

The autoscaler exposes its own metrics that can be scraped by Prometheus:

| Metric | Description |
|--------|-------------|
| `flink.kubernetes.operator.autoscaler.current.parallelism` | Current parallelism per operator |
| `flink.kubernetes.operator.autoscaler.recommended.parallelism` | Target parallelism per operator |
| `flink.kubernetes.operator.autoscaler.vertex.busy.time` | Fraction of time operator is busy |
| `flink.kubernetes.operator.autoscaler.vertex.backlog.processing.time` | Time to clear current backlog |
| `flink.kubernetes.operator.autoscaler.scaling.report` | Summary of last scaling decision |

To view autoscaler decisions:

```bash
# Check events on the FlinkDeployment
kubectl describe flinkdeployment autoscaled-app -n flink-prod | grep -A 20 Events

# Check autoscaler logs in the operator
kubectl logs deployment/flink-kubernetes-operator -n flink-operator \
  | grep -i autoscal
```

### Autoscaler Standalone Mode

Since Flink Kubernetes Operator 1.6.0, the autoscaler can be deployed as a **standalone component** separate from the main Operator. This is useful when:
- The Operator is managed by platform teams but jobs are managed by individual teams
- You want to enable autoscaling without the full Operator footprint
- You are using a non-Operator deployment (native Kubernetes mode)

```bash
helm install flink-autoscaler \
  flink-operator-repo/flink-kubernetes-operator \
  --namespace flink-operator \
  --set operatorMode=standalone-autoscaler
```

---

## SQL Jobs with FlinkDeployment

The Operator supports running Flink SQL statements directly, without writing any Java/Scala code. This is enabled via the `sql-runner` image:

```yaml
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  name: sql-etl-job
  namespace: flink-prod
spec:
  flinkVersion: v1_18
  image: apache/flink-kubernetes-operator:1.9.0    # sql-runner included
  imagePullPolicy: IfNotPresent

  flinkConfiguration:
    taskmanager.numberOfTaskSlots: "2"
    state.checkpoints.dir: s3://my-bucket/checkpoints/sql-etl

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
    jarURI: local:///opt/flink/opt/flink-sql-runner.jar
    entryClass: org.apache.flink.sql.runner.SqlJobRunner
    parallelism: 4
    upgradeMode: savepoint
    state: running
    args:
      - |
        CREATE TABLE orders (
          order_id    BIGINT,
          product_id  STRING,
          amount      DECIMAL(10, 2),
          order_time  TIMESTAMP(3),
          WATERMARK FOR order_time AS order_time - INTERVAL '5' SECOND
        ) WITH (
          'connector' = 'kafka',
          'topic'     = 'orders',
          'properties.bootstrap.servers' = 'kafka:9092',
          'format'    = 'json'
        );

        CREATE TABLE order_aggregates (
          product_id      STRING,
          window_start    TIMESTAMP(3),
          total_revenue   DECIMAL(12, 2),
          order_count     BIGINT,
          PRIMARY KEY (product_id, window_start) NOT ENFORCED
        ) WITH (
          'connector' = 'jdbc',
          'url'       = 'jdbc:postgresql://db:5432/analytics',
          'table-name' = 'order_aggregates'
        );

        INSERT INTO order_aggregates
        SELECT
          product_id,
          TUMBLE_START(order_time, INTERVAL '1' HOUR) AS window_start,
          SUM(amount)    AS total_revenue,
          COUNT(*)       AS order_count
        FROM orders
        GROUP BY
          product_id,
          TUMBLE(order_time, INTERVAL '1' HOUR);
```

---

## Advanced Configuration

### Pod Templates

Pod templates allow full Kubernetes Pod customization — sidecars, volumes, tolerations, affinity, security contexts — without those fields being natively supported in `FlinkDeployment`.

```yaml
spec:
  # Applied to both JobManager and TaskManager Pods
  podTemplate:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9249"
    spec:
      serviceAccountName: flink-sa
      nodeSelector:
        workload: flink
      tolerations:
        - key: flink-workload
          operator: Equal
          value: "true"
          effect: NoSchedule
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: my-streaming-app
      containers:
        - name: flink-main-container    # Required name
          env:
            - name: AWS_REGION
              value: us-east-1
            - name: SECRET_VALUE
              valueFrom:
                secretKeyRef:
                  name: my-secret
                  key: value
          resources:
            limits:
              memory: "5Gi"    # Override memory limit (process size × 1.25)
          volumeMounts:
            - name: config-files
              mountPath: /app/config
              readOnly: true
        - name: log-forwarder            # Sidecar container
          image: fluent/fluent-bit:2.0
          volumeMounts:
            - name: varlog
              mountPath: /var/log/flink
      volumes:
        - name: config-files
          configMap:
            name: my-app-config
        - name: varlog
          emptyDir: {}

  # TaskManager-specific overrides
  taskManager:
    podTemplate:
      spec:
        containers:
          - name: flink-main-container
            volumeMounts:
              - name: rocksdb-storage
                mountPath: /flink-rocksdb
        volumes:
          - name: rocksdb-storage
            emptyDir:
              sizeLimit: 50Gi
```

### Ingress Configuration

The Operator can manage ingress resources for the Flink Web UI:

```yaml
spec:
  ingress:
    template: "/{{namespace}}/{{name}}(/|$)(.*)"
    className: nginx
    annotations:
      nginx.ingress.kubernetes.io/rewrite-target: "/$2"
      nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
      cert-manager.io/cluster-issuer: letsencrypt-prod
    tls:
      - hosts:
          - flink.example.com
        secretName: flink-tls
```

With this configuration, the UI for deployment `my-app` in namespace `flink-prod` is accessible at `https://flink.example.com/flink-prod/my-app`.

### Native Kubernetes Configuration Overrides

All native Kubernetes configuration properties from Flink are available:

```yaml
flinkConfiguration:
  # Custom labels on all Pods
  kubernetes.jobmanager.labels: "team:data,env:prod"
  kubernetes.taskmanager.labels: "team:data,env:prod"

  # Custom annotations
  kubernetes.jobmanager.annotations: "iam.amazonaws.com/role: arn:aws:iam::123:role/flink"

  # Node selector
  kubernetes.taskmanager.node-selector: "cloud.google.com/gke-nodepool:flink-pool"

  # Tolerations (JSON)
  kubernetes.taskmanager.tolerations: >
    [{"key":"flink-exclusive","operator":"Equal","value":"true","effect":"NoSchedule"}]

  # Environment variables
  kubernetes.taskmanager.env.vars: "ENV:prod,REGION:us-east-1"

  # Image pull secret
  kubernetes.container.image.pull-secrets: "my-registry-secret"

  # Owner references for garbage collection
  kubernetes.jobmanager.owner.reference: "true"
```

---

## Operator Configuration Reference

The Operator itself is configured via its `flink-operator-conf.yaml` (set in the Helm values under `operatorConfiguration`):

| Property | Default | Description |
|----------|---------|-------------|
| `kubernetes.operator.reconcile.interval.seconds` | `15` | How often the operator reconciles each resource |
| `kubernetes.operator.observer.rest-ready.delay` | `10s` | Delay after pod ready before connecting to REST |
| `kubernetes.operator.savepoint.history.max.count` | `10` | Max savepoints to retain in status history |
| `kubernetes.operator.savepoint.history.max.age` | `24h` | Max age of savepoints in status history |
| `kubernetes.operator.periodic.savepoint.interval` | (none) | Interval for automatic periodic savepoints |
| `kubernetes.operator.savepoint.trigger.grace-period` | `1h` | Grace period for savepoint trigger timeout |
| `kubernetes.operator.savepoint.format.type` | `NATIVE` | Savepoint format: `NATIVE` or `CANONICAL` |
| `kubernetes.operator.cluster.health-check.enabled` | `true` | Enable job health checking |
| `kubernetes.operator.cluster.health-check.restarts.window` | `2m` | Window for restart health check |
| `kubernetes.operator.cluster.health-check.restarts.threshold` | `64` | Max restarts in window before marking unhealthy |
| `kubernetes.operator.job.upgrade.last-state.checkpoint.max.age` | `10m` | Max age of checkpoint for last-state upgrade |
| `kubernetes.operator.watched.namespaces` | (all) | Namespaces to watch (comma-separated) |
| `kubernetes.operator.metrics.reporter.slf4j.interval` | `5 MINUTE` | Metrics reporting interval |

---

## Multi-Namespace Management

By default, the Operator watches all namespaces. For multi-tenant clusters, you may want to restrict the Operator to specific namespaces:

```yaml
# Helm values
watchNamespaces:
  - flink-prod
  - flink-staging
  - flink-dev
```

The Operator creates the necessary RBAC (Role/RoleBinding) in each watched namespace automatically. For very large clusters, consider deploying multiple Operator instances:

```bash
# Operator for production
helm install flink-operator-prod flink-operator-repo/flink-kubernetes-operator \
  --namespace flink-operator-prod \
  --set watchNamespaces={flink-prod}

# Operator for non-prod
helm install flink-operator-nonprod flink-operator-repo/flink-kubernetes-operator \
  --namespace flink-operator-nonprod \
  --set watchNamespaces={flink-staging,flink-dev}
```

---

## GitOps Integration

The `FlinkDeployment` and `FlinkSessionJob` CRDs are standard Kubernetes resources — they work naturally with any GitOps tool.

### ArgoCD

Store `FlinkDeployment` YAML in a Git repository and let ArgoCD sync it:

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-streaming-app
  namespace: argocd
spec:
  project: data-platform
  source:
    repoURL: https://github.com/my-org/flink-deployments
    targetRevision: main
    path: apps/my-streaming-app
  destination:
    server: https://kubernetes.default.svc
    namespace: flink-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: false    # Disable selfHeal to prevent Operator status updates triggering re-sync
    syncOptions:
      - ServerSideApply=true
      - PrunePropagationPolicy=foreground
      - RespectIgnoreDifferences=true
  ignoreDifferences:
    - group: flink.apache.org
      kind: FlinkDeployment
      jsonPointers:
        - /status                    # Ignore status (managed by Operator)
        - /spec/job/parallelism      # Ignore if autoscaler is managing parallelism
```

> **Important**: Add `ignoreDifferences` for `/status` and autoscaler-managed fields to prevent ArgoCD from continuously reverting Operator-managed changes.

### Flux CD

```yaml
# flux-kustomization.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: flink-apps
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: flink-deployments-repo
  path: ./apps/production
  prune: true
  healthChecks:
    - apiVersion: flink.apache.org/v1beta1
      kind: FlinkDeployment
      name: my-streaming-app
      namespace: flink-prod
  postBuild:
    substituteFrom:
      - kind: ConfigMap
        name: flink-env-config
```

---

## Observability

### Operator Metrics

The Operator itself exposes metrics about its reconciliation activity:

```yaml
# Enable Prometheus metrics in Helm values
operatorConfiguration:
  metrics.reporter.prom.factory.class: org.apache.flink.metrics.prometheus.PrometheusReporterFactory
  metrics.reporter.prom.port: "9249"
```

Key operator metrics:

| Metric | Description |
|--------|-------------|
| `flink.kubernetes.operator.lifecycle.state` | Current lifecycle state per deployment |
| `flink.kubernetes.operator.resource.transitions.count` | Number of state transitions |
| `flink.kubernetes.operator.reconciler.failures` | Reconciliation failures |
| `flink.kubernetes.operator.jm.deployment.time` | Time to start JobManager |
| `flink.kubernetes.operator.cleanup.count` | Resources cleaned up |

### Job Metrics

Each `FlinkDeployment` status provides a lifecycle state, which can be used to create alerts:

| Lifecycle State | Description |
|----------------|-------------|
| `CREATED` | Resource just created, not yet reconciled |
| `SUSPENDED` | Job intentionally suspended |
| `UPGRADING` | Upgrade in progress |
| `DEPLOYED` | Cluster deployed, job starting |
| `STABLE` | Job running and healthy |
| `ROLLING_BACK` | Upgrade failed; rolling back to last stable state |
| `FAILED` | Job/cluster in unrecoverable failed state |

Create Prometheus alerts on lifecycle states:

```yaml
# Prometheus alerting rule
groups:
  - name: flink-operator
    rules:
      - alert: FlinkJobFailed
        expr: |
          flink_kubernetes_operator_lifecycle_state{state="FAILED"} == 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Flink job {{ $labels.resource_name }} is in FAILED state"

      - alert: FlinkJobRestartLoop
        expr: |
          increase(flink_jobmanager_job_numRestarts[10m]) > 3
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Flink job {{ $labels.job_name }} is restarting frequently"
```

---

## Upgrading the Operator

Upgrade the Operator using Helm:

```bash
# Update the Helm repo
helm repo update

# Check available versions
helm search repo flink-operator-repo/flink-kubernetes-operator --versions

# Upgrade (non-breaking: existing FlinkDeployments are unaffected)
helm upgrade flink-kubernetes-operator \
  flink-operator-repo/flink-kubernetes-operator \
  --namespace flink-operator \
  --version 1.9.0 \
  --values operator-values.yaml

# Verify the new Operator pod is running
kubectl rollout status deployment/flink-kubernetes-operator -n flink-operator
```

> **Note**: Upgrading the Operator does NOT automatically restart existing Flink deployments. The new Operator version will begin reconciling existing `FlinkDeployment` resources with the new logic.

**CRD upgrades**: When upgrading the Operator, CRDs may need to be updated too. The Helm chart handles this automatically:

```bash
helm upgrade flink-kubernetes-operator ... --set crds.install=true
```

Or manually:

```bash
kubectl apply -f https://raw.githubusercontent.com/apache/flink-kubernetes-operator/release-1.9/helm/flink-kubernetes-operator/crds/flinkdeployments.flink.apache.org-v1.yml
kubectl apply -f https://raw.githubusercontent.com/apache/flink-kubernetes-operator/release-1.9/helm/flink-kubernetes-operator/crds/flinksessionjobs.flink.apache.org-v1.yml
```

---

## Troubleshooting

### Common Issues and Resolutions

**1. FlinkDeployment stuck in `UPGRADING` state**

```bash
# Check operator logs
kubectl logs deployment/flink-kubernetes-operator -n flink-operator | tail -100

# Check events on the deployment
kubectl describe flinkdeployment my-app -n flink-prod

# Force a reconciliation by adding/removing an annotation
kubectl annotate flinkdeployment my-app -n flink-prod \
  kubernetes.io/change-cause="manual-reconcile-$(date +%s)" --overwrite
```

**2. Savepoint trigger timeout**

```bash
# Check if the savepoint is actually in progress in the Flink REST API
JM_POD=$(kubectl get pods -n flink-prod -l component=jobmanager -o name | head -1)
kubectl exec -n flink-prod $JM_POD -- \
  curl -s http://localhost:8081/jobs/<job-id>/savepoints

# Increase the savepoint grace period if jobs are large
# flinkConfiguration:
#   kubernetes.operator.savepoint.trigger.grace-period: "2h"
```

**3. Job enters `FAILED` lifecycle state**

```bash
# Get failure reason from status
kubectl get flinkdeployment my-app -n flink-prod -o jsonpath='{.status.error}'

# Check Flink exception
kubectl exec -n flink-prod $JM_POD -- \
  curl -s http://localhost:8081/jobs/<job-id>/exceptions | jq .

# Check TaskManager logs for the root cause
kubectl logs -n flink-prod -l component=taskmanager --tail=200
```

**4. Webhook refusing FlinkDeployment**

```bash
# Check webhook configuration
kubectl get validatingwebhookconfigurations flink-operator-webhook -o yaml

# Test what validation error is returned
kubectl apply -f my-deployment.yaml --dry-run=server
```

**5. TaskManagers not starting (slots unavailable)**

```bash
# Check if resource quota is exceeded
kubectl describe quota -n flink-prod

# Check if there are pending Pods
kubectl get pods -n flink-prod | grep Pending
kubectl describe pod <pending-pod-name> -n flink-prod
```

### Diagnostic Commands Cheatsheet

```bash
# List all Flink deployments
kubectl get flinkdeployments -A

# Get deployment status summary
kubectl get flinkdeployments -n flink-prod \
  -o custom-columns='NAME:.metadata.name,STATE:.status.lifecycleState,JOB:.status.jobStatus.state'

# Stream operator logs
kubectl logs -f deployment/flink-kubernetes-operator -n flink-operator

# Port-forward to a specific deployment's Web UI
JM_SVC=$(kubectl get svc -n flink-prod -l app=my-streaming-app,component=jobmanager -o name | head -1)
kubectl port-forward -n flink-prod $JM_SVC 8081:8081

# Get all events for a deployment (useful for debugging)
kubectl events --for flinkdeployment/my-streaming-app -n flink-prod

# Force delete a stuck deployment (WARNING: does not save state)
kubectl patch flinkdeployment my-app -n flink-prod \
  -p '{"metadata":{"finalizers":null}}' --type merge
kubectl delete flinkdeployment my-app -n flink-prod
```

---

## Security

### Operator RBAC

The Operator Helm chart creates a `ClusterRole` with the following permissions:

```yaml
# ClusterRole for the Flink Operator
rules:
  - apiGroups: [flink.apache.org]
    resources: [flinkdeployments, flinksessionjobs, flinkdeployments/status, flinksessionjobs/status, flinkdeployments/finalizers, flinksessionjobs/finalizers]
    verbs: [get, list, watch, create, update, patch, delete]
  - apiGroups: [""]
    resources: [pods, services, configmaps, events, persistentvolumeclaims]
    verbs: [get, list, watch, create, update, patch, delete]
  - apiGroups: [apps]
    resources: [deployments]
    verbs: [get, list, watch, create, update, patch, delete]
  - apiGroups: [networking.k8s.io]
    resources: [ingresses]
    verbs: [get, list, watch, create, update, patch, delete]
  - apiGroups: [coordination.k8s.io]
    resources: [leases]
    verbs: [get, list, watch, create, update, patch, delete]
```

### Restricting Operator Scope

For namespace-scoped operation (reduce blast radius):

```yaml
# Helm values
watchNamespaces: [flink-prod]    # Restrict to specific namespaces
rbac:
  clusterScope: false            # Use Role + RoleBinding instead of ClusterRole
```

### Pod Security Admission

Configure PSA labels on Flink namespaces:

```bash
# Restricted mode (most secure; Flink may need some relaxations)
kubectl label namespace flink-prod \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted

# Baseline mode (good balance for Flink)
kubectl label namespace flink-prod \
  pod-security.kubernetes.io/enforce=baseline
```

Flink Pod security configuration for `restricted` PSA:

```yaml
spec:
  podTemplate:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 9999
        fsGroup: 9999
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: flink-main-container
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false   # Flink needs to write /tmp
            capabilities:
              drop: [ALL]
```

### Secret Management Integration

**External Secrets Operator:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: flink-app-secrets
  namespace: flink-prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: flink-app-secrets
  data:
    - secretKey: kafka-password
      remoteRef:
        key: secret/flink/kafka
        property: password
    - secretKey: db-password
      remoteRef:
        key: secret/flink/database
        property: password
```

Then reference in the `FlinkDeployment` pod template:

```yaml
spec:
  podTemplate:
    spec:
      containers:
        - name: flink-main-container
          env:
            - name: KAFKA_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: flink-app-secrets
                  key: kafka-password
```

---

## Additional Resources

- [Flink Kubernetes Operator Documentation](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-stable/)
- [Flink Kubernetes Operator GitHub](https://github.com/apache/flink-kubernetes-operator)
- [Operator Helm Chart Reference](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-stable/docs/operations/helm/)
- [FlinkDeployment CRD Reference](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-stable/docs/custom-resource/reference/)
- [Autoscaler Documentation](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-stable/docs/custom-resource/autoscaler/)
- [Operator Examples Repository](https://github.com/apache/flink-kubernetes-operator/tree/main/examples)
- [Flink Kubernetes Deployment Guide](./kubernetes-deployment.md)
- [Apache Flink Overview](./overview.md)
