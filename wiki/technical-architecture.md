# Technical Architecture Document

*For `iac_books` – Kubernetes Deployment (Local + EKS)*

---

## 1. Overview
**Purpose**:
The `iac_books` repository contains **Kubernetes manifests (YAML)** built with **Kustomize** to deploy a **book storage web application** and its **PostgreSQL database**. The infrastructure is designed to run on:
- **Local Kubernetes** (e.g., Minikube, k3s) for development/testing
- **Amazon EKS** for production-grade deployments

**Scope**:
This document describes the **logical schema, network design, and Kubernetes architecture** for the application.

---

## 2. System Architecture
**High-Level Design**:
```
┌───────────────────────┐     ┌───────────────────────┐
│     Web Application    │────▶│    PostgreSQL DB      │
│  (Book Storage Frontend │     │  (Persistent Storage)  │
│   + Backend API)       │◀────│                       │
└───────────────────────┘     └───────────────────────┘
```
- **Web App**: Handles book CRUD operations (Create, Read, Update, Delete)
- **PostgreSQL**: Stores book data, user metadata, and application state

**Data Flow**:
1. User interacts with the web app (e.g., add/edit a book)
2. App validates input and writes/reads to PostgreSQL
3. PostgreSQL persists data to **Kubernetes PersistentVolumes**

---

## 3. Kubernetes Infrastructure
### 3.1 Cluster Topology
| Environment  | Type               | Nodes          | Namespaces       |
|--------------|--------------------|----------------|------------------|
| **Local**    | Minikube/k3s       | Single-node    | `default`, `books` |
| **EKS**      | Managed Kubernetes | Multi-AZ       | `dev`, `preprod`, `prod` |

### 3.2 Networking
| Component       | Local Setup               | EKS Setup                     |
|-----------------|---------------------------|-------------------------------|
| **Service Type** | NodePort/ClusterIP        | LoadBalancer (ALB/NLB)        |
| **Ingress**     | Optional (e.g., Nginx)    | ALB Ingress Controller        |
| **DNS**         | Local hosts file          | Route 53 + ACM (SSL)          |
| **Firewall**    | None (local)              | Security Groups + NACLs      |
| **CIDR**        | Default (10.0.0.0/8)      | Custom VPC (e.g., 10.1.0.0/16) |

**Network Policies**:
- Restrict PostgreSQL access to the web app pods only
- Allow ingress traffic on ports `80/443` (EKS) or `30000-32767` (NodePort for local)

---

## 4. Database Architecture
### 4.1 PostgreSQL Schema
**Tables**:
- `books` (id, title, author, isbn, published_date, created_at)
- `users` (id, username, email, hashed_password)
- `reviews` (id, book_id, user_id, rating, comment)

### 4.2 Storage
| Environment | Storage Class       | Persistence Mechanism       |
|-------------|---------------------|------------------------------|
| **Local**   | `standard` (local)  | `hostPath` or `local-volume` |
| **EKS**     | `gp3` (SSD)         | EBS (for statefulsets)       |

**Backup Strategy**:
- **Local**: Manual `pg_dump` to local storage
- **EKS**: Automated snapshots (RDS) or `pg_dump` to S3

---

## 5. Kustomize & YAML Structure
**Directory Layout**:
```
iac_books/
├── base/
│   ├── deployment.yaml       # Web app + PostgreSQL
│   ├── service.yaml          # ClusterIP/NodePort
│   ├── kustomization.yaml    # Common labels/annotations
│   └── configmap.yaml        # App configs (non-sensitive)
├── overlays/
│   ├── local/
│   │   ├── kustomization.yaml # Patches for local (NodePort)
│   │   └── storage.yaml       # hostPath PVC
│   └── eks/
│       ├── kustomization.yaml # Patches for EKS (LoadBalancer)
│       └── storage.yaml       # EBS PVC + StorageClass
└── README.md
```

**Key Files**:
- **`deployment.yaml`**:
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: books-app
  spec:
    replicas: 1  # 2+ for EKS
    template:
      spec:
        containers:
        - name: app
          image: books-app:<TAG>  # Tag from GitLab
          envFrom:
          - configMapRef: { name: app-config }
          - secretRef: { name: db-credentials }
        - name: postgres
          image: postgres:15
          volumeMounts:
          - name: pg-data
            mountPath: /var/lib/postgresql/data
        volumes:
        - name: pg-data
          persistentVolumeClaim:
            claimName: pg-pvc
  ```

- **`kustomization.yaml` (EKS Overlay)**:
  ```yaml
  resources:
  - ../base
  patches:
  - path: deployment-patch.yaml  # Increase replicas, add resources
  - path: service-patch.yaml     # Change to LoadBalancer
  ```

---

## 6. Security
| Aspect               | Local Setup               | EKS Setup                     |
|----------------------|---------------------------|-------------------------------|
| **Secrets**          | Kubernetes Secrets        | AWS Secrets Manager + External Secrets Operator |
| **RBAC**             | Minimal (local dev)       | IAM Roles for Service Accounts (IRSA) |
| **Network Policies** | Optional                  | Enforced (Calico/Cilium)      |
| **Encryption**       | None (local)              | EBS encryption + TLS (ACM)   |