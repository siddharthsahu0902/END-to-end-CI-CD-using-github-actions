# End-to-End CI/CD Pipeline (Spring Boot → Docker Hub → Kubernetes)

## Architecture

``` text
Developer
    │
    ▼
GitHub Repository (Source Code)
    │
    ▼
GitHub Actions (CI/CD)
    │
    ├── Checkout Repository
    ├── Build Spring Boot Application
    ├── Run Tests
    ├── Build Docker Image
    ├── Push Image to Docker Hub
    └── Deploy to Kubernetes
                 │
                 ▼
        Kubernetes API Server
                 │
                 ▼
        Deployment Controller
                 │
                 ▼
             ReplicaSet
                 │
                 ▼
                Pods
                 │
                 ▼
              Service
                 │
                 ▼
              Ingress
                 │
                 ▼
                Users
```

## CI Process

### 1. Developer pushes code

``` bash
git add .
git commit -m "Added login API"
git push origin main
```

### 2. GitHub Actions Trigger

``` yaml
on:
  push:
    branches:
      - main
```

GitHub starts a temporary Ubuntu runner.

### 3. Checkout Repository

``` yaml
- uses: actions/checkout@v4
```

Downloads your source code to the runner.

### 4. Install Java

``` yaml
- uses: actions/setup-java@v4
  with:
    distribution: temurin
    java-version: 21
    cache: maven
```

### 5. Build Spring Boot

``` yaml
- name: Build
  run: mvn clean package
```

Produces:

    target/jobapp.jar

### 6. Build Docker Image

``` yaml
- name: Build Docker Image
  run: docker build -t siddharth/jobapp:${{ github.sha }} .
```

### 7. Login to Docker Hub

``` yaml
- uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}
```

### 8. Push Image

``` yaml
- name: Push Image
  run: docker push siddharth/jobapp:${{ github.sha }}
```

CI ends here.

------------------------------------------------------------------------

# CD Process

### 9. Install kubectl

``` yaml
- uses: azure/setup-kubectl@v4
```

### 10. Configure kubeconfig

Store the complete kubeconfig as a GitHub Secret named `KUBECONFIG`.

Example:

``` yaml
apiVersion: v1

clusters:
- cluster:
    server: https://10.0.1.2:6443
    certificate-authority-data: <base64-ca>

users:
- name: github-actions
  user:
    token: <token>

contexts:
- context:
    cluster: default
    user: github-actions
  name: default

current-context: default
```

Workflow step:

``` yaml
- name: Configure kubeconfig
  run: |
    mkdir -p ~/.kube
    echo "${{ secrets.KUBECONFIG }}" > ~/.kube/config
```

### 11. Deploy

``` yaml
- name: Deploy
  run: |
    kubectl set image deployment/jobapp \
      jobapp=siddharth/jobapp:${{ github.sha }}
```

`kubectl` sends an HTTPS request to the Kubernetes API Server.

### 12. Kubernetes

The API Server updates the Deployment.

The Deployment Controller creates a new ReplicaSet.

The Scheduler selects worker nodes.

Each Kubelet pulls the new Docker image from Docker Hub and starts new
Pods.

A rolling update replaces old Pods with new Pods.

The Service and Ingress continue routing traffic to healthy Pods.

------------------------------------------------------------------------

# Complete GitHub Actions Workflow

``` yaml
name: Spring Boot CI/CD

on:
  push:
    branches:
      - main

jobs:
  build-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21
          cache: maven

      - name: Build Application
        run: mvn clean package

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build Docker Image
        run: docker build -t siddharth/jobapp:${{ github.sha }} .

      - name: Push Docker Image
        run: docker push siddharth/jobapp:${{ github.sha }}

      - name: Install kubectl
        uses: azure/setup-kubectl@v4

      - name: Configure kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBECONFIG }}" > ~/.kube/config

      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/jobapp \
            jobapp=siddharth/jobapp:${{ github.sha }}
```

------------------------------------------------------------------------

# Where Each Step Happens

  Step                 Location
  -------------------- -------------------
  Git Push             Developer Laptop
  Workflow Trigger     GitHub
  Checkout             GitHub Runner
  Maven Build          GitHub Runner
  Docker Build         GitHub Runner
  Docker Push          Docker Hub
  Install kubectl      GitHub Runner
  kubectl Deployment   GitHub Runner
  Kubernetes API       Control Plane
  Scheduler            Control Plane
  Image Pull           Worker Node
  New Pods             Worker Node
  Traffic              Service + Ingress
