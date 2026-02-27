# Apache Flink Documentation

This directory contains comprehensive documentation on Apache Flink — the distributed stream and batch processing framework — with a focus on deploying and operating Flink on Kubernetes.

## Documents

| Document | Description |
|----------|-------------|
| [Overview](./overview.md) | Apache Flink framework: architecture, APIs, state management, time semantics, fault tolerance, connectors, and use cases |
| [Kubernetes Deployment Guide](./kubernetes-deployment.md) | All Kubernetes deployment strategies in depth: Standalone YAML, Native Kubernetes, Helm, and the Flink Operator |
| [Flink Kubernetes Operator Guide](./flink-operator.md) | Flink Kubernetes Operator: installation, CRDs, FlinkDeployment, FlinkSessionJob, lifecycle management, autoscaling, GitOps, security |

## Quick Reference: Choosing a Deployment Strategy

```
Are you deploying to Kubernetes?
└── Yes
    ├── Do you want the least operational overhead?
    │   └── Yes → Use the Flink Kubernetes Operator (recommended for production)
    │             See: flink-operator.md
    │
    ├── Do you need GitOps/Helm-based packaging without the Operator?
    │   └── Yes → Use Helm Chart deployment
    │             See: kubernetes-deployment.md → Strategy 3
    │
    ├── Do you need dynamic resource management without the Operator?
    │   └── Yes → Use Native Kubernetes mode (CLI-driven)
    │             See: kubernetes-deployment.md → Strategy 2
    │
    └── Are you learning or need a simple setup?
        └── Yes → Use Standalone Kubernetes (manual YAML)
                  See: kubernetes-deployment.md → Strategy 1
```

## Flink Execution Mode Quick Reference

| Mode | Cluster Lifetime | Jobs per Cluster | Resource Isolation | Best For |
|------|-----------------|------------------|--------------------|---------|
| **Application Mode** | Per application | 1 | Full | Production streaming jobs |
| **Session Mode** | Long-lived | Many | Shared | Dev/test, short jobs |
| **Per-Job Mode** | Per job | 1 | Full | Legacy (deprecated in 1.15) |
