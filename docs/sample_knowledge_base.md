# Monk Platform — Technical Overview

## What is Monk?

Monk is a cloud-agnostic infrastructure orchestration platform that allows teams to define, deploy, and manage distributed applications using a declarative language called MonkScript. It abstracts away cloud provider differences, letting you write one definition and deploy to AWS, GCP, Azure, or bare metal.

## Core Concepts

### Runnables

A **runnable** is the basic unit of deployment in Monk. It wraps a container image with its configuration: environment variables, volumes, ports, resources, and health checks.

```yaml
namespace: my-app

app:
  defines: runnable
  containers:
    web:
      image: nginx:1.25
      ports:
        - 80:80
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
```

### Groups

A **group** is a set of runnables deployed together as a logical unit. Groups are the primary way to model multi-service applications.

```yaml
stack:
  defines: process-group
  runnable-list:
    - my-app/api
    - my-app/worker
    - my-app/db
```

### MonkScript

MonkScript is a YAML-based DSL that compiles to Monk's internal representation. It supports:

- **ArrowScript expressions** for dynamic values (`<- env("PORT") | "8080"`)
- **Variables** with default values and type annotations
- **Connections** between services using DNS-based service discovery
- **Lifecycle hooks** for init, pre-start, post-start, and stop phases

---

## Deployment Model

Monk uses a **peer-to-peer cluster** model. Each node in the cluster runs a `monkd` daemon that participates in a distributed hash table (DHT). There is no central control plane — any node can deploy workloads.

### Cluster Operations

| Command | Description |
|---|---|
| `monk cluster new` | Bootstrap a new cluster |
| `monk cluster grow` | Add a cloud instance to the cluster |
| `monk cluster peers` | List connected peers |
| `monk run <group>` | Deploy a runnable or group |
| `monk logs <runnable>` | Stream logs from a workload |

---

## Secrets Management

Secrets are stored encrypted in the cluster's distributed store. They are injected at runtime and never written to disk in plaintext.

```yaml
variables:
  db-password:
    type: string
    env: DB_PASSWORD
    value: <- secret("my-app/db-password")
```

To set a secret:

```bash
monk secrets set my-app/db-password "s3cr3t!"
```

---

## Networking

Monk provides automatic service discovery within a group. Services can reference each other by name:

```yaml
variables:
  db-host:
    value: <- connection-hostname("database")
```

Inter-group traffic is routed through an encrypted overlay network (WireGuard by default).

---

## Scaling and Resources

Runnables declare resource requests and limits. The scheduler places workloads on peers with sufficient capacity.

```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 2
    memory: 2Gi
```

Horizontal scaling is done by increasing the `scale` field on a runnable or by using auto-scaling policies tied to CPU/memory metrics.

---

## CI/CD Integration

Monk integrates with GitHub Actions and GitLab CI through the `monk-ci` plugin. A typical workflow:

1. Push to `main` triggers the pipeline.
2. The CI job loads the MonkScript manifest.
3. `monk load` validates and uploads the definitions.
4. `monk run` deploys the updated workloads.
5. Health checks confirm the deployment succeeded.

---

## Common Troubleshooting

### Workload stuck in `pending`

- Check that the cluster has enough resources: `monk cluster status`
- Verify that required secrets are set: `monk secrets list`
- Inspect scheduler decisions: `monk describe <runnable>`

### Logs not streaming

- Confirm `monkd` is running on the target peer: `monk cluster peers`
- Check firewall rules allow port 44004 (Monk P2P port)

### Image pull failures

- Ensure the registry credentials are configured: `monk cluster registry-ensure`
- Check that the image tag exists and the repository is accessible from the peer's network

---

## Glossary

| Term | Definition |
|---|---|
| Runnable | A single containerised workload unit |
| Group / Process-group | A named set of runnables deployed together |
| Peer | A machine participating in the Monk cluster |
| MonkScript | Monk's YAML-based declarative DSL |
| ArrowScript | Expression language embedded in MonkScript values |
| DHT | Distributed Hash Table used for cluster state |
| Overlay network | Encrypted virtual network connecting cluster peers |
