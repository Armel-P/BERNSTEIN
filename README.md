# BERNSTEIN – Kubernetes Voting Application

A containerized voting application deployed on **Azure Kubernetes Service (AKS)** using Kubernetes Deployments, Services, Horizontal Pod Autoscalers, persistent storage and Azure Container Registry.

The application is composed of three microservices:

- **Poll:** Web application that accepts votes and publishes them to Redis.
- **Worker:** Background service that processes votes from Redis and stores the results in PostgreSQL.
- **Result:** Web application that displays voting results retrieved from PostgreSQL.

---

## 1. Architecture

The application uses the following architecture:

```text
                   Internet
                      |
          +-----------+-----------+
          |                       |
    Poll LoadBalancer       Result LoadBalancer
          |                       |
      Poll Service            Result Service
          |                       |
     Poll Pods (2)           Result Pods (2)
          |                       |
          v                       v
        Redis <--- Worker Pods ---> PostgreSQL
          |          (2)             |
          +-------------------------+
                                     |
                              Persistent Volume
```

### Components

| Component | Technology | Purpose |
|---|---|---|
| Poll | Python / Flask | Accepts votes |
| Worker | Java | Processes votes |
| Result | Node.js / Express / Socket.IO | Displays results |
| Redis | Redis | Temporary vote queue |
| PostgreSQL | PostgreSQL | Persistent vote storage |
| Container Registry | Azure Container Registry | Stores application images |
| Kubernetes | Azure Kubernetes Service | Container orchestration |

## 2. Repository structure

```text
BERNSTEIN/
├── deploy/
│   └── deploy.sh
├── k8s/
│   ├── namespace.yaml
│   ├── config.yaml
│   ├── poll.yaml
│   ├── worker.yaml
│   ├── result.yaml
│   ├── redis.yaml
│   └── postgres.yaml
├── poll/
│   ├── Dockerfile
│   └── application files
├── worker/
│   ├── Dockerfile
│   └── application files
├── result/
│   ├── Dockerfile
│   └── application files
├── docker-compose.yaml
├── schema.sql
└── README.md
```

## 3. Application deployment

The application images are built and pushed to Azure Container Registry (ACR).

The Kubernetes cluster pulls the images from the private registry.

Example image naming:

```text
pitou1312.azurecr.io/poll:dev
pitou1312.azurecr.io/worker:dev
pitou1312.azurecr.io/result:dev
```

The image tag is configurable through environment variables.

### Prerequisites

Install and configure:

- Docker
- Docker Compose
- Azure CLI
- kubectl
- envsubst

You also need:

- An Azure subscription.
- An AKS cluster.
- An Azure Container Registry.
- Access to the Kubernetes cluster.
- ACR pull permissions for the AKS cluster.

### Configure Azure access

Log in to Azure:

```bash
az login
```

Get AKS credentials:

```bash
az aks get-credentials \
  --resource-group <RESOURCE_GROUP> \
  --name <AKS_CLUSTER_NAME>
```

Verify the connection:

```bash
kubectl get nodes
```

For ACR integration, an administrator can configure the AKS cluster with:

```bash
az aks update \
  --resource-group <RESOURCE_GROUP> \
  --name <AKS_CLUSTER_NAME> \
  --attach-acr <ACR_NAME>
```

The identity used by AKS must have permission to pull images from ACR.

## 4. Environment configuration

Deployment settings and credentials are supplied through environment variables.

Example `.env` file:

```dotenv
REGISTRY=pitou1312.azurecr.io
TAG=dev

POSTGRES_PASSWORD=CHANGE_ME
REDIS_PASSWORD=CHANGE_ME
```

Keep this file private.

**Never commit real passwords, tokens or credentials to Git.**

The Kubernetes deployment uses:

- ConfigMaps for non-sensitive application configuration.
- Kubernetes Secrets for database and Redis passwords.

## 5. Kubernetes namespace

All application resources are deployed into the `poll` namespace.

Create the namespace:

```bash
kubectl apply -f k8s/namespace.yaml
```

Check it:

```bash
kubectl get namespaces
```

## 6. Microservices and replicas

The application runs multiple replicas to improve availability.

| Deployment | Desired replicas |
|---|---:|
| Poll | 2 |
| Worker | 2 |
| Result | 2 |
| Redis | 1 |
| PostgreSQL | 1 |

Poll, Worker and Result use Kubernetes Deployments.

PostgreSQL uses a StatefulSet because it requires persistent storage and a stable identity.

Redis currently runs as a single Deployment replica.

## 7. Rolling updates and availability

Poll, Worker and Result use the RollingUpdate deployment strategy.

The configuration uses:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

This allows Kubernetes to create a replacement pod before removing an existing available pod.

Readiness probes help prevent traffic from being sent to pods that are not ready.

PodDisruptionBudgets are also configured to help maintain application availability during voluntary disruptions.

## 8. Horizontal Pod Autoscaling

Poll and Result use Kubernetes Horizontal Pod Autoscalers.

The autoscaling configuration uses:

| Setting | Value |
|---|---:|
| Minimum replicas | 2 |
| Maximum replicas | 5 |
| CPU utilization target | 70% |

The HPA uses CPU utilization to determine whether additional replicas are needed.

Example configuration:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

The Kubernetes Metrics Server must be available for CPU-based autoscaling.

Check autoscaling:

```bash
kubectl -n poll get hpa
```

Check resource metrics:

```bash
kubectl top nodes
kubectl top pods -n poll
```

## 9. Pod anti-affinity

Poll and Result use pod anti-affinity to distribute replicas across different Kubernetes nodes.

Example:

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: poll
        topologyKey: kubernetes.io/hostname
```

This helps prevent multiple replicas of the same application from being scheduled on one node.

The cluster currently has two worker nodes. With required anti-affinity, more than two replicas of a particular application may remain Pending unless additional eligible nodes are available.

## 10. Resource requests and limits

Application containers have CPU and memory requests and limits.

Example:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

Requests reserve resources for scheduling.

Limits prevent containers from exceeding their configured CPU and memory allocations.

Resource settings are important for predictable scheduling and effective autoscaling.

## 11. Health checks

Application containers use Kubernetes health probes.

The application endpoints are:

| Endpoint | Purpose |
|---|---|
| `/healthz` | Liveness check |
| `/readyz` | Readiness check |

The liveness probe checks whether the application should continue running.

The readiness probe checks whether the application can receive traffic.

Example:

```yaml
readinessProbe:
  httpGet:
    path: /readyz
    port: 80
  periodSeconds: 5

livenessProbe:
  httpGet:
    path: /healthz
    port: 80
  initialDelaySeconds: 30
  periodSeconds: 10
```

PostgreSQL and Redis use appropriate service-specific health checks.

## 12. PostgreSQL persistence

PostgreSQL runs as a StatefulSet with a PersistentVolumeClaim.

The PVC provides persistent storage independently of the PostgreSQL container lifecycle.

Current storage configuration:

| Setting | Value |
|---|---|
| Storage request | 1 GiB |
| Access mode | ReadWriteOnce |
| PVC | `data-postgres-0` |
| Status | Bound |

The database schema is initialized using `schema.sql`, mounted through a ConfigMap.

PostgreSQL is exposed internally through a Kubernetes Service.

It is not exposed directly to the public internet.

**Important:** Do not delete the PostgreSQL PVC when updating the application. Deleting it may result in permanent data loss.

## 13. Redis

Redis provides temporary queue storage between the Poll and Worker services.

It is accessible internally through:

```text
redis:6379
```

Redis requires authentication using a Kubernetes Secret.

Redis is exposed through a ClusterIP Service, so it is accessible inside the Kubernetes cluster rather than directly from the internet.

## 14. Kubernetes Services and external access

Poll and Result are exposed through LoadBalancer Services.

Current external endpoints:

| Service | Type | External IP | Port |
|---|---|---|---|
| Poll | LoadBalancer | 9.205.140.48 | 80 |
| Result | LoadBalancer | 9.205.148.49 | 80 |
| Redis | ClusterIP | Internal only | 6379 |
| PostgreSQL | Headless ClusterIP | Internal only | 5432 |

Application URLs:

- Poll: http://9.205.140.48
- Result: http://9.205.148.49

The external IP addresses may change if the corresponding Azure load-balancer resources are recreated.

## 15. Deployment procedure

The deployment script builds and pushes the application images, prepares Kubernetes configuration and applies the manifests.

The deployment process includes:

1. Load environment variables.
2. Authenticate with Azure Container Registry.
3. Build application images.
4. Push images to ACR.
5. Create or update the Kubernetes namespace.
6. Create or update Kubernetes Secrets.
7. Create or update ConfigMaps.
8. Deploy Redis and PostgreSQL.
9. Wait for the database and Redis to become ready.
10. Deploy Poll, Worker and Result.
11. Wait for application rollouts.

Run the deployment script from the repository root:

```bash
chmod +x deploy/deploy.sh
./deploy/deploy.sh
```

The script must resolve Kubernetes and Docker Compose paths relative to the repository root.

## 16. Verification

Check Kubernetes nodes:

```bash
kubectl get nodes -o wide
```

Check application pods:

```bash
kubectl -n poll get pods -o wide
```

Check deployments:

```bash
kubectl -n poll get deployments
```

Check Services:

```bash
kubectl -n poll get services
```

Check autoscaling:

```bash
kubectl -n poll get hpa
```

Check persistent storage:

```bash
kubectl -n poll get pvc
```

Check resource usage:

```bash
kubectl top nodes
kubectl top pods -n poll
```

Check rollout status:

```bash
kubectl -n poll rollout status deployment/poll
kubectl -n poll rollout status deployment/worker
kubectl -n poll rollout status deployment/result
```

Check application logs:

```bash
kubectl -n poll logs deployment/poll
kubectl -n poll logs deployment/worker
kubectl -n poll logs deployment/result
```

## 17. Troubleshooting

### Pod in CrashLoopBackOff

Inspect the current and previous container logs:

```bash
kubectl -n poll logs <POD_NAME>
kubectl -n poll logs <POD_NAME> --previous
```

Inspect events and probe failures:

```bash
kubectl -n poll describe pod <POD_NAME>
```

### HPA cannot retrieve CPU metrics

Check the Metrics API:

```bash
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl top nodes
kubectl top pods -n poll
```

### PVC not bound

Check storage resources:

```bash
kubectl -n poll get pvc
kubectl get storageclass
kubectl -n poll describe pvc data-postgres-0
```

### Image pull errors

Check the image name and tag:

```bash
kubectl -n poll describe pod <POD_NAME>
```

Verify that AKS has permission to pull images from ACR.

### Application health checks failing

Check the application logs and test the endpoints:

```bash
kubectl -n poll get pods
kubectl -n poll describe pod <POD_NAME>
```

## 18. Security considerations

- Store credentials in Kubernetes Secrets rather than ConfigMaps.
- Do not commit `.env` files containing real credentials.
- Keep PostgreSQL and Redis internal to the cluster.
- Use a private container registry.
- Restrict access to the Kubernetes API.
- Use resource requests and limits.
- Keep application images and dependencies updated.
- Configure backups for persistent database data.

## 19. Conclusion

BERNSTEIN demonstrates a microservice-based voting application deployed on Azure Kubernetes Service.

The project uses containerization, private image storage, Kubernetes orchestration, rolling updates, horizontal autoscaling, health probes, resource management, service discovery and persistent database storage.

The architecture separates the web applications, background processing and data services to support maintainability, scalability and availability.

---

**Project:** BERNSTEIN  
**Platform:** Microsoft Azure / AKS  
**Orchestration:** Kubernetes  
**Container Registry:** Azure Container Registry
