# Scalable Laravel Architecture on Kubernetes

## 1. Architectural Overview
This solution transforms the monolithic Laravel application into a decoupled, cloud-native architecture capable of horizontal scaling.

### Key Design Decisions
1.  **Workload Separation:** - **API Tier:** Handles HTTP traffic only. Scales on CPU/Memory usage (HPA).
    - **Worker Tier:** Handles file processing (Horizon). Scales on Redis Queue Depth (KEDA).
2.  **State Management (The "Shared Disk" Solution):**
    - Since the application logic relies on local temporary files, I utilized a **ReadWriteMany (RWX) Persistent Volume** backed by **AWS EFS**.
    - This allows the API to write a temporary file that a Worker on a different node can immediately read and process.
3.  **Data Integrity:**
    - Implemented `terminationGracePeriodSeconds` and `preStop` hooks to ensure Horizon finishes active jobs before shutting down.
    - Added a **Cleaner CronJob** to remove orphaned files from EFS if a worker crashes mid-process.

## 2. Infrastructure Components
| Component | Technology | Purpose |
| :--- | :--- | :--- |
| **Compute** | Amazon EKS | Managed Kubernetes Cluster |
| **Storage** | Amazon EFS | Shared temporary storage for file hand-off |
| **Queue/Cache**| Amazon ElastiCache | Managed Redis for job queues |
| **Database** | Amazon RDS | Managed MySQL/PostgreSQL |
| **Auto-scaling**| HPA + KEDA | Dual-strategy scaling (Resource & Event-based) |

## 3. Deployment Instructions
The solution is packaged as standard Kubernetes manifests.

```bash
# 1. Apply Storage & Config
kubectl apply -f k8s/00-config.yaml
kubectl apply -f k8s/01-storage.yaml

# 2. Deploy Workloads
kubectl apply -f k8s/02-api.yaml
kubectl apply -f k8s/03-worker.yaml

# 3. Setup Autoscaling & Maintenance
kubectl apply -f k8s/04-autoscaling.yaml
kubectl apply -f k8s/05-cleaner.yaml
