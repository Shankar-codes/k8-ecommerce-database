# 🛒 k8-ecommerce-database

A production-ready, cloud-native e-commerce platform deployed on **Kubernetes** with a microservices architecture. The project provisions and manages all stateful data layers — relational, document, cache, and message broker — as Kubernetes workloads backed by persistent AWS EBS volumes.

---

## 📐 Architecture Overview

The application is decomposed into independent microservices, each with its own dedicated data store, deployed inside a dedicated Kubernetes namespace (`ellamma-namaspace`).

```
┌─────────────────────────────────────────────────────────┐
│                  Kubernetes Cluster (AWS EKS)            │
│  Namespace: ellamma-namaspace                           │
│                                                          │
│  ┌──────────┐  ┌───────────┐  ┌──────────┐             │
│  │ Frontend │  │   User    │  │ Catalogue│             │
│  │ (React / │  │ Service   │  │ Service  │             │
│  │  Nginx)  │  │ (MySQL)   │  │(MongoDB) │             │
│  └──────────┘  └─────┬─────┘  └────┬─────┘             │
│                       │              │                   │
│  ┌──────────┐  ┌─────▼─────┐  ┌────▼─────┐             │
│  │  Cart    │  │   MySQL   │  │ MongoDB  │             │
│  │ Service  │  │StatefulSet│  │StatefulSet│            │
│  │ (Redis)  │  └───────────┘  └──────────┘             │
│  └────┬─────┘                                           │
│       │       ┌───────────┐  ┌──────────┐              │
│  ┌────▼─────┐ │  Payment  │  │ Shipping │              │
│  │  Redis   │ │  Service  │  │ Service  │              │
│  │StatefulSet│ └─────┬─────┘  └────┬─────┘             │
│  └──────────┘        │              │                   │
│                ┌─────▼──────────────▼──┐                │
│                │     RabbitMQ          │                │
│                │  (Message Broker)     │                │
│                └───────────────────────┘                │
└─────────────────────────────────────────────────────────┘
```

---

## 📦 Services & Components

| Service       | Role                                        | Data Store       |
|---------------|---------------------------------------------|------------------|
| **frontend**  | UI layer served via Nginx / React            | —                |
| **user**      | User auth, registration, profile management  | MySQL            |
| **catalogue** | Product listings and inventory               | MongoDB          |
| **cart**      | Shopping cart (session-based)               | Redis            |
| **payment**   | Payment processing and order confirmation    | RabbitMQ (async) |
| **shipping**  | Order fulfilment and shipment tracking       | RabbitMQ (async) |
| **mysql**     | Relational database for structured data      | EBS PersistentVolume |
| **mongodb**   | Document database for product catalogue      | EBS PersistentVolume |
| **redis**     | In-memory cache / session store for cart     | EBS PersistentVolume |
| **rabbitmq**  | Message broker for async event-driven flows  | EBS PersistentVolume |

---

## 🗂️ Repository Structure

```
k8-ecommerce-database/
├── 01-namespace.yaml          # Kubernetes namespace definition
├── 02-ebs-storageclass.yaml   # AWS EBS StorageClass (Retain policy)
├── cart/                      # Cart service manifests
├── catalogue/                 # Catalogue service manifests
├── frontend/                  # Frontend manifests
├── mongodb/                   # MongoDB StatefulSet, Service, PVC
├── mysql/                     # MySQL StatefulSet, Service, PVC, Secrets
├── payment/                   # Payment service manifests
├── rabbitmq/                  # RabbitMQ StatefulSet, Service, PVC
├── redis/                     # Redis StatefulSet, Service, PVC
├── shipping/                  # Shipping service manifests
└── user/                      # User service manifests
```

---

## ⚙️ Infrastructure Details

### Namespace

All resources live in a single namespace to provide isolation:

```yaml
# 01-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ellamma-namaspace
  labels:
    project: ellamma-ecommerce-database-project
    environment: dev
```

### Storage Class (AWS EBS)

A custom `StorageClass` backed by the **AWS EBS CSI driver** is used for all persistent volumes:

```yaml
# 02-ebs-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ellamma-ebs
reclaimPolicy: Retain
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
```

- **`reclaimPolicy: Retain`** — volumes are preserved when a PVC is deleted, preventing accidental data loss.
- **`volumeBindingMode: WaitForFirstConsumer`** — volumes are provisioned only after a Pod is scheduled, ensuring AZ-aware placement.

---

## 🚀 Getting Started

### Prerequisites

- AWS EKS cluster (or any Kubernetes 1.24+ cluster)
- `kubectl` configured and pointing at your cluster
- AWS EBS CSI driver installed on the cluster
- Sufficient IAM permissions for EBS volume provisioning

### Deployment Order

Resources must be applied in sequence to satisfy dependencies:

```bash
# 1. Create the namespace
kubectl apply -f 01-namespace.yaml

# 2. Create the StorageClass (requires admin privileges)
kubectl apply -f 02-ebs-storageclass.yaml

# 3. Deploy stateful data stores
kubectl apply -f mysql/
kubectl apply -f mongodb/
kubectl apply -f redis/
kubectl apply -f rabbitmq/

# 4. Deploy application microservices
kubectl apply -f user/
kubectl apply -f catalogue/
kubectl apply -f cart/
kubectl apply -f payment/
kubectl apply -f shipping/

# 5. Deploy the frontend
kubectl apply -f frontend/
```

### Verify Deployment

```bash
# Check all resources in the namespace
kubectl get all -n ellamma-namaspace

# Check PersistentVolumeClaims
kubectl get pvc -n ellamma-namaspace

# Check StatefulSets are ready
kubectl get statefulsets -n ellamma-namaspace
```

---

## 🔌 Service Communication

- **Synchronous (HTTP/REST):** Frontend → User, Catalogue, Cart services
- **Asynchronous (AMQP):** Cart/Payment → RabbitMQ → Payment/Shipping consumers
- **Caching:** Cart service reads/writes session data to Redis
- **Persistence:** User service reads/writes to MySQL; Catalogue service reads/writes to MongoDB

---

## 💾 Persistent Storage

Each stateful component has its own `PersistentVolumeClaim` referencing the `ellamma-ebs` StorageClass:

| Component | Storage Type | Claim |
|-----------|-------------|-------|
| MySQL     | EBS (gp2/gp3) | mysql-pvc |
| MongoDB   | EBS (gp2/gp3) | mongodb-pvc |
| Redis     | EBS (gp2/gp3) | redis-pvc |
| RabbitMQ  | EBS (gp2/gp3) | rabbitmq-pvc |

---

## 🏷️ Labels & Conventions

All resources follow a consistent labelling strategy:

```yaml
labels:
  project: ellamma-ecommerce-database-project
  environment: dev
```

---

## 🛠️ Tech Stack

| Layer         | Technology                  |
|---------------|-----------------------------|
| Orchestration | Kubernetes (AWS EKS)        |
| Storage       | AWS EBS (via CSI Driver)    |
| Relational DB | MySQL                       |
| Document DB   | MongoDB                     |
| Cache         | Redis                       |
| Message Queue | RabbitMQ                    |
| Frontend      | React / Nginx               |

---

## 📝 Notes

- This project is configured for a **`dev`** environment. For production, consider enabling resource requests/limits, HPA (Horizontal Pod Autoscaler), network policies, and secrets management (e.g., AWS Secrets Manager or Vault).
- The `StorageClass` must be created by a cluster admin before deploying any StatefulSets.
- RabbitMQ and Redis data is also persisted to EBS, ensuring durability across Pod restarts.

---

## 🤝 Contributing

Pull requests are welcome. For significant changes, please open an issue first to discuss what you would like to change.

---

## 📄 License

This project is open source. See the repository for license details.
