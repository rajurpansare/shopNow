# ShopNow - DevOps Containerization and Container Orchestration Assignment

## 1. Objective

This assignment containerizes the ShopNow MERN application and deploys it on Kubernetes. The solution includes:

- Docker containers for Backend, Frontend and Admin applications.
- MongoDB deployed as a Kubernetes StatefulSet.
- Raw Kubernetes YAML manifests.
- A reusable Helm chart.
- Horizontal Pod Autoscaling for the backend.
- Jenkins Groovy pipeline for build, image push and Helm deployment.
- Verification and troubleshooting steps.

## 2. Application Architecture

```text
                         +----------------------+
                         |      Kubernetes      |
                         |      Namespace       |
                         |       shopnow        |
                         +----------+-----------+
                                    |
              +---------------------+----------------------+
              |                     |                      |
       +------v------+       +------v------+       +-------v------+
       |  Frontend   |       |    Admin    |       |   Backend    |
       | React+Nginx |       | React+Nginx |       | Node/Express |
       | 2 replicas  |       | 2 replicas  |       | 2-5 replicas |
       +------+------+       +------+------+       +-------+------+
              |                     |                        |
              +---------------------+------------------------+
                                    |
                           +--------v--------+
                           |     MongoDB     |
                           |   StatefulSet   |
                           |   persistent PV |
                           +-----------------+
```

The frontend and admin Nginx containers proxy API requests to the Kubernetes service `backend-service:5000`. The backend connects to MongoDB through the service name `mongo`.

## 3. Repository

GitHub repository:

https://github.com/rajurpansare/shopNow

## 4. Important project observations

The supplied project already contained Dockerfiles for all three application components and a Node/Express backend. The backend exposes `/api/health`, which is used by Kubernetes readiness and liveness probes.

The original frontend/admin Dockerfiles used the example path `aryan`. For a normal Kubernetes service deployment without a URL subpath, the Dockerfiles were adjusted so the React applications are built with:

- `PUBLIC_URL=/`
- `REACT_APP_API_BASE_URL=/api`

This avoids broken static asset/API paths when the application is opened from the root URL.

## 5. Prerequisites

Install or have available:

- Git
- Docker Desktop
- Node.js 18+
- kubectl
- kind or another Kubernetes cluster
- Helm 3+
- Jenkins (for CI/CD stage)

Verify:

```bash
git --version
docker --version
node --version
npm --version
kubectl version --client
helm version
kind version
```

## 6. Clone repository

```bash
git clone https://github.com/rajurpansare/shopNow.git
cd shopNow
```

If using the ZIP supplied for this assignment, copy the generated Kubernetes/Jenkins/docs files into the repository before committing.

## 7. Build Docker images locally

```bash
docker build -t shopnow-backend:latest ./backend
docker build -t shopnow-frontend:latest ./frontend
docker build -t shopnow-admin:latest ./admin
```

Verify:

```bash
docker images | grep shopnow
```

## 8. Create a local Kubernetes cluster with kind

```bash
kind create cluster --name shopnow
kubectl cluster-info
kubectl get nodes
```

### Load local images into kind

```bash
kind load docker-image shopnow-backend:latest --name shopnow
kind load docker-image shopnow-frontend:latest --name shopnow
kind load docker-image shopnow-admin:latest --name shopnow
```

## 9. Deploy using raw Kubernetes manifests

The raw files are in `kubernetes/base/`.

```bash
kubectl apply -k kubernetes/base/
```

Check:

```bash
kubectl get all -n shopnow
kubectl get pods -n shopnow
kubectl get pvc -n shopnow
kubectl get hpa -n shopnow
```

Wait until all application pods show `Running` and `READY`.

## 10. Verify MongoDB

```bash
kubectl get statefulset -n shopnow
kubectl get pods -n shopnow -l app=mongo
```

Check MongoDB logs:

```bash
kubectl logs statefulset/mongo -n shopnow
```

## 11. Verify backend

```bash
kubectl get svc -n shopnow
kubectl port-forward svc/backend-service 5000:5000 -n shopnow
```

In another terminal:

```bash
curl http://localhost:5000/api/health
```

Expected response is a successful health response from the ShopNow backend.

## 12. Access frontend and admin

For the easiest local demonstration, use port forwarding.

Frontend:

```bash
kubectl port-forward svc/frontend 3000:80 -n shopnow
```

Open:

http://localhost:3000

Admin:

```bash
kubectl port-forward svc/admin 3001:80 -n shopnow
```

Open:

http://localhost:3001

## 13. Test scaling

Check replicas:

```bash
kubectl get deployment -n shopnow
```

Scale frontend manually:

```bash
kubectl scale deployment frontend --replicas=3 -n shopnow
kubectl get pods -n shopnow -l app=frontend
```

Scale back:

```bash
kubectl scale deployment frontend --replicas=2 -n shopnow
```

The backend also has an HPA configured for CPU utilization.

```bash
kubectl get hpa -n shopnow
kubectl describe hpa backend-hpa -n shopnow
```

For HPA metrics on a local cluster, install Metrics Server if your cluster does not provide the Metrics API.

## 14. Deploy using Helm

The Helm chart is located at:

`kubernetes/helm/shopnow`

Validate it:

```bash
helm lint kubernetes/helm/shopnow
helm template shopnow kubernetes/helm/shopnow
```

For local kind deployment, first remove the raw deployment if it is already installed:

```bash
kubectl delete namespace shopnow
```

Then load images again if required and install Helm:

```bash
helm upgrade --install shopnow kubernetes/helm/shopnow \
  --namespace shopnow \
  --create-namespace
```

Check:

```bash
helm list -n shopnow
kubectl get all -n shopnow
```

## 15. Change image names with Helm

For Docker Hub images:

```bash
helm upgrade --install shopnow kubernetes/helm/shopnow \
  -n shopnow --create-namespace \
  --set images.backend=YOUR_DOCKERHUB_USERNAME/shopnow-backend:latest \
  --set images.frontend=YOUR_DOCKERHUB_USERNAME/shopnow-frontend:latest \
  --set images.admin=YOUR_DOCKERHUB_USERNAME/shopnow-admin:latest
```

## 16. Jenkins CI/CD

The pipeline is stored in the repository root as:

`Jenkinsfile`

### Jenkins plugins

Install/configure at minimum:

- Pipeline
- Git
- Credentials Binding
- Docker Pipeline, if required by the Jenkins installation

### Jenkins credentials

Create:

1. `dockerhub-creds` - Username/password credential for Docker Hub.
2. `shopnow-kubeconfig` - Secret file containing the kubeconfig used by Jenkins to access the Kubernetes cluster.

### Jenkins tools

The Jenkins agent must have:

```text
git
docker
node
npm
kubectl
helm
```

### Pipeline flow

```text
Git Checkout
     |
     v
Validate project
     |
     v
npm ci + React builds
     |
     v
Docker build x3
     |
     v
Docker login
     |
     v
Push images
     |
     v
Helm upgrade --install
     |
     v
kubectl rollout status
```

### Jenkins setup

Create a Pipeline job:

1. Jenkins Dashboard -> New Item.
2. Select Pipeline.
3. Choose `Pipeline script from SCM`.
4. SCM: Git.
5. Repository URL: `https://github.com/rajurpansare/shopNow.git`.
6. Branch: `*/main`.
7. Script Path: `Jenkinsfile`.
8. Save.
9. Build with Parameters.
10. Enter the Docker Hub username.
11. Set `DEPLOY=true` when the Jenkins agent can access the Kubernetes cluster.

## 17. Troubleshooting

### Pod is CrashLoopBackOff

```bash
kubectl get pods -n shopnow
kubectl describe pod POD_NAME -n shopnow
kubectl logs POD_NAME -n shopnow
```

### Backend cannot connect to MongoDB

```bash
kubectl get pods -n shopnow -l app=mongo
kubectl get svc mongo -n shopnow
kubectl logs deployment/backend -n shopnow
```

The expected MongoDB hostname inside Kubernetes is `mongo`, not `localhost`.

### Frontend shows API errors

Check that Nginx contains:

```text
proxy_pass http://backend-service:5000/api/;
```

Also check:

```bash
kubectl get svc backend-service -n shopnow
kubectl logs deployment/backend -n shopnow
```

### ImagePullBackOff with local kind

Load the image into kind:

```bash
kind load docker-image shopnow-backend:latest --name shopnow
kind load docker-image shopnow-frontend:latest --name shopnow
kind load docker-image shopnow-admin:latest --name shopnow
```

### HPA shows unknown metrics

Install Metrics Server and verify:

```bash
kubectl top nodes
kubectl top pods -n shopnow
```

## 18. Challenges and solutions

### Challenge 1: Example user path in Dockerfiles

The supplied Dockerfiles contained the example `aryan` path. This can cause React assets to be generated under a path that is not used by a normal root deployment.

**Solution:** Build the React applications with root public/API paths for this Kubernetes deployment.

### Challenge 2: MongoDB needs persistent storage

A normal Deployment does not provide stable identity or persistent storage for MongoDB.

**Solution:** Use a StatefulSet with a PersistentVolumeClaim.

### Challenge 3: Kubernetes service discovery

Containers must not use `localhost` to reach another Kubernetes pod.

**Solution:** Backend connects to `mongo:27017`, and Nginx proxies API traffic to `backend-service:5000`.

### Challenge 4: Container health

Kubernetes needs a reliable indication that a container is ready to receive traffic.

**Solution:** Configure readiness and liveness probes. Backend uses `/api/health`; Nginx containers use `/health`.

### Challenge 5: Scaling

A fixed number of backend pods cannot automatically react to CPU load.

**Solution:** Add an HPA with minimum 2 and maximum 5 replicas at 70% CPU utilization.

## 19. Evidence/screenshots to capture for VLearn

Capture screenshots showing:

1. GitHub repository with the added folders.
2. `docker images` showing ShopNow images.
3. `kubectl get nodes`.
4. `kubectl get pods -n shopnow`.
5. `kubectl get svc -n shopnow`.
6. `kubectl get deployment -n shopnow`.
7. `kubectl get statefulset -n shopnow`.
8. `kubectl get hpa -n shopnow`.
9. `helm list -n shopnow`.
10. ShopNow frontend in browser.
11. ShopNow admin page in browser.
12. Jenkins successful pipeline.
13. Jenkins console output showing Docker build/push and Helm deployment.

## 20. Submission checklist

- [ ] Dockerfiles available for backend/frontend/admin.
- [ ] Raw Kubernetes manifests committed.
- [ ] MongoDB StatefulSet committed.
- [ ] Services and health probes committed.
- [ ] HPA committed.
- [ ] Helm chart committed.
- [ ] Jenkinsfile committed.
- [ ] Documentation committed.
- [ ] Local/Kubernetes deployment tested.
- [ ] GitHub repository updated.
- [ ] Repository link placed in a TXT, Word or PDF file.
- [ ] VLearn submission completed.

## 21. Final repository structure

```text
shopNow/
├── admin/
│   ├── Dockerfile
│   └── nginx/default.conf
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
├── frontend/
│   ├── Dockerfile
│   └── nginx/default.conf
├── kubernetes/
│   ├── base/
│   │   ├── namespace.yaml
│   │   ├── mongo.yaml
│   │   ├── backend.yaml
│   │   ├── frontend.yaml
│   │   ├── admin.yaml
│   │   ├── hpa.yaml
│   │   └── kustomization.yaml
│   └── helm/
│       └── shopnow/
│           ├── Chart.yaml
│           ├── values.yaml
│           └── templates/
├── jenkins/ 
├── docs/
│   └── ASSIGNMENT-REPORT.md
└── Jenkinsfile
```

## 22. Conclusion

The ShopNow application is containerized as independent frontend, admin and backend workloads. Kubernetes manages deployment, service discovery, health checks, scaling and persistent MongoDB storage. Helm packages the Kubernetes resources into a reusable release, while Jenkins automates validation, Docker image creation, image publishing and Helm deployment.
