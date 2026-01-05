# Scalable Laravel Architecture on Kubernetes

This repository contains the infrastructure solution for migrating a legacy **Laravel** application to a cloud-native, scalable **Kubernetes** architecture.

The design addresses two major bottlenecks **without requiring changes to application business logic**:

- **Resource contention** between API and background jobs  
- **Stateful local file storage** in a distributed container environment  

---

## Repository Structure & Manifest Breakdown

| File | Purpose |
| :--- | :--- |
| **`Dockerfile`** | Multi-stage build that installs system dependencies (Redis, BCMath), sets secure permissions, and runs the container as a non-root user (`www-data`). |
| **`k8s/laravel-config.yaml`** | Stores non-sensitive configuration (`APP_ENV`) in a ConfigMap and sensitive data (`DB_PASSWORD`) in a Secret (base64 encoded). |
| **`k8s/laravel-api.yaml`** | Deploys the Laravel Web API with `replicas: 2`, liveness probes for self-healing, and strict CPU limits to prevent resource starvation. |
| **`k8s/laravel-worker.yaml`** | Deploys Laravel Horizon workers with `terminationGracePeriodSeconds: 60` and `preStop` hooks to ensure active jobs complete safely. |
| **`k8s/01-storage.yaml`** | Configures a **ReadWriteMany (RWX)** Persistent Volume backed by **AWS EFS** for shared file access across nodes. |
| **`k8s/autoscaling.yaml`** | Configures **HPA** for API autoscaling (CPU-based) and **KEDA** for worker autoscaling (Redis queue depth). |
| **`k8s/cleaner.yaml`** | Scheduled CronJob to remove orphaned temporary files from shared storage in case of worker crashes. |

---

## 1. Architectural Overview

### Key Design Decisions

#### 1. Workload Separation
- **API Tier**
  - Handles HTTP traffic only
  - Scales using **Horizontal Pod Autoscaler (HPA)**
- **Worker Tier**
  - Handles background file processing (Laravel Horizon)
  - Scales using **KEDA** based on Redis queue depth

This separation prevents background jobs from starving API resources.

---

#### 2. State Management (Shared Disk Strategy)

Because the legacy application relies on local temporary files:

- A **ReadWriteMany (RWX) Persistent Volume** backed by **AWS EFS** is used
- API pods write temporary files
- Worker pods (on any node) can immediately read and process them

This preserves legacy behavior while enabling horizontal scaling.

---

#### 3. Data Integrity & Safety

- `terminationGracePeriodSeconds` ensures workers are not killed mid-job
- `preStop` hooks allow Horizon to finish processing active jobs
- A **Cleaner CronJob** removes orphaned files left behind after crashes

---

## 2. Infrastructure Components

| Component | Technology | Purpose |
| :--- | :--- | :--- |
| **Compute** | Amazon EKS | Managed Kubernetes cluster |
| **Storage** | Amazon EFS | Shared RWX storage for temporary files |
| **Queue / Cache** | Amazon ElastiCache (Redis) | Job queues & caching |
| **Database** | Amazon RDS | Managed MySQL / PostgreSQL |
| **Auto-scaling** | HPA + KEDA | Resource-based + event-driven scaling |

---

## 3. CI/CD Pipeline Strategy

The repository includes `.github/workflows/deploy.yaml` demonstrating the deployment automation approach.

---

### Continuous Integration (CI)

- **Automated Builds** triggered on every push
- **Immutable Artifacts**
  - Docker images are tagged with the **Git Commit SHA**
  - Example: `image:v1-a1b2c`
  - Prevents ambiguity caused by `latest` tags

---

### Continuous Delivery (CD) with Helm

While raw Kubernetes manifests are included for architectural clarity, **production deployments use Helm**.

Helm ensures:
- Atomic deployments
- Safe rollbacks
- Environment-specific configuration overrides

#### Example Helm Deployment

helm upgrade --install laravel-app ./charts/laravel \
  --set image.tag=${GITHUB_SHA} \
  --values ./charts/laravel/values-env.yaml \
  --atomic

## Atomic Deployments

If new pods fail to start (for example, due to an application crash), **Helm automatically rolls back** to the last stable release.  
This ensures zero-downtime deployments and protects production stability.

---

## Consistent Delivery

The **same Docker image** is promoted from **Staging → Production**, with **only configuration values changing**.  
This guarantees consistency across environments and eliminates “works in staging but not prod” issues.

---

## 4. Manual Deployment Instructions (For Evaluation)

For testing or evaluation purposes, the application can be deployed directly using raw Kubernetes manifests.

### Apply Storage & Configuration

```bash
# Deploy Workloads
kubectl apply -f k8s/laravel-config.yaml
kubectl apply -f k8s/01-storage.yaml

kubectl apply -f k8s/laravel-api.yaml
kubectl apply -f k8s/laravel-worker.yaml

# Setup Autoscaling & Maintenance
kubectl apply -f k8s/autoscaling.yaml
kubectl apply -f k8s/cleaner.yaml
