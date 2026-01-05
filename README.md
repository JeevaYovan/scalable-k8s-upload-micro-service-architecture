# Scalable Laravel Architecture on Kubernetes

This repository contains the infrastructure solution for migrating a legacy Laravel application to a cloud-native, scalable Kubernetes architecture. [cite_start]The solution addresses critical bottlenecks—specifically **resource contention** and **stateful local file storage**—without requiring changes to the application's business logic[cite: 13, 14, 20, 31].

## 📂 Repository Structure & Manifest Breakdown

| File | Purpose |
| :--- | :--- |
| **`Dockerfile`** | Multi-stage build that installs system dependencies (Redis, BCMath), sets permissions, and configures the container to run as a secure non-root user (`www-data`). |
| **`k8s/laravel-config.yaml`** | Stores non-sensitive configs (`APP_ENV`) in a ConfigMap and sensitive data (`DB_PASSWORD`) in a Secret (base64 encoded). |
| **`k8s/laravel-api.yaml`** | [cite_start]Deploys the Web API with `replicas: 2`, Liveness probes for self-healing, and strict CPU limits to prevent resource starvation[cite: 23]. |
| **`k8s/laravel-worker.yaml`** | [cite_start]Deploys the Horizon Worker with `terminationGracePeriodSeconds: 60` and `preStop` hooks to ensure jobs finish before the pod shuts down[cite: 23]. |
| **`k8s/01-storage.yaml`** | [cite_start]Configures the **ReadWriteMany (RWX)** Persistent Volume (AWS EFS) to allow file sharing between API and Worker pods across different nodes[cite: 25]. |
| **`k8s/autoscaling.yaml`** | [cite_start]Configures **HPA** to scale the API based on CPU usage and **KEDA** to scale Workers based on the Redis Queue depth[cite: 24]. |
| **`k8s/cleaner.yaml`** | [cite_start]Runs a scheduled CronJob to delete "orphaned" temporary files from the shared storage that may be left behind if a worker crashes[cite: 26]. |

## 1. Architectural Overview

### Key Design Decisions
1.  **Workload Separation:**
    * **API Tier:** Handles HTTP traffic only. [cite_start]Scales on CPU/Memory usage (HPA)[cite: 23].
    * **Worker Tier:** Handles file processing (Horizon). [cite_start]Scales on Redis Queue Depth (KEDA)[cite: 23].
2.  **State Management (The "Shared Disk" Solution):**
    * [cite_start]Since the application logic relies on local temporary files, I utilized a **ReadWriteMany (RWX) Persistent Volume** backed by **AWS EFS**[cite: 8, 25].
    * This allows the API to write a temporary file that a Worker on a different node can immediately read and process.
3.  **Data Integrity:**
    * [cite_start]Implemented `terminationGracePeriodSeconds` and `preStop` hooks to ensure Horizon finishes active jobs before shutting down[cite: 26].
    * Added a **Cleaner CronJob** to remove orphaned files from EFS if a worker crashes mid-process.

## 2. Infrastructure Components
| Component | Technology | Purpose |
| :--- | :--- | :--- |
| **Compute** | Amazon EKS | [cite_start]Managed Kubernetes Cluster [cite: 20] |
| **Storage** | Amazon EFS | Shared temporary storage for file hand-off |
| **Queue/Cache**| Amazon ElastiCache | [cite_start]Managed Redis for job queues [cite: 9] |
| **Database** | Amazon RDS | Managed MySQL/PostgreSQL |
| **Auto-scaling**| HPA + KEDA | Dual-strategy scaling (Resource & Event-based) |

## 3. CI/CD Pipeline Strategy

[cite_start]The included `.github/workflows/deploy.yaml` demonstrates the automation strategy.

### Continuous Integration (CI)
* [cite_start]**Automated Builds:** Triggered on every push to `main`.
* **Immutable Artifacts:** Docker images are tagged with the **Git Commit SHA** (e.g., `image:v1-a1b2c`) rather than `latest`. [cite_start]This ensures we know exactly what code is running in production.

### Continuous Delivery (CD) with Helm
[cite_start]While this repository provides raw Kubernetes manifests (`.yaml`) for architectural clarity, in a production environment, these files serve as the solution templates to be incorporated into a **Helm Chart**.

[cite_start]The CD pipeline applies changes using `helm upgrade`, ensuring atomic deployments and safe rollbacks:

```bash
# Example of how the pipeline applies these configurations
helm upgrade --install laravel-app ./charts/laravel \
  --set image.tag=${GITHUB_SHA} \
  --values ./charts/laravel/values-prod.yaml \
  --atomic

Atomic Deployments: If the new pods fail to start (e.g., code crash), Helm automatically rolls back to the previous stable release.

Consistent Delivery: The same Docker image is promoted from Staging to Production, varying only the configuration values.

4. Manual Deployment Instructions (For Evaluation)
To deploy the raw manifests directly for testing purposes:

Bash

# 1. Apply Storage & Config
kubectl apply -f k8s/laravel-config.yaml
kubectl apply -f k8s/01-storage.yaml

# 2. Deploy Workloads
kubectl apply -f k8s/laravel-api.yaml
kubectl apply -f k8s/laravel-worker.yaml

# 3. Setup Autoscaling & Maintenance
kubectl apply -f k8s/autoscaling.yaml
kubectl apply -f k8s/cleaner.yaml
