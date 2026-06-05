# Jenkins and Kubernetes (k8s) GitOps Pipeline Analysis

This document details the containerization, manifest tokenization, and infrastructure orchestration strategies used in the `Jenkins-k8s` project structure and provides targeted technical interview preparation materials.

---

## Core Topics Covered in this Repository

### 1. Node.js Application Stack Integration
* **Concept**: Managing dependencies and entry-points for microservices.
* **Backend Utilities**: `package.json` tracks project dependencies and start scripts. `server.js` functions as the core Node.js runtime script rendering application traffic.

### 2. Microservice Containerization
* **Concept**: Packing an ephemeral layer into a light application runtime.
* **Backend Utilities**: `Dockerfile` describes how the Node.js application is layered, built, and exposed for isolated execution.

### 3. Dynamic Manifest Modification (GitOps Preparation)
* **Concept**: Dynamically updating Kubernetes configurations during pipeline runtime.
* **Backend Utilities**: `changeTag.sh` uses text streams or utilities (like `sed`) to update image tags inside Kubernetes manifest files on the fly, matching the exact build number produced by Jenkins.

### 4. Kubernetes Declarative Infrastructure 
* **Concept**: Maintaining state configuration models for containers at scale.
* **Backend Utilities**: `pods.yml` structures how target workloads or replica pods launch, while `services.yml` creates internal/external networking routes to expose the application to traffic.

---

## Technical Interview Questions & Answers

### Topic 1: Jenkins & Kubernetes Orchestration Patterns

#### Q1: Why is a script like `changeTag.sh` necessary in a Jenkins-to-Kubernetes CI/CD pipeline?
* **Answer**: In automated deployment configurations, hardcoding an image tag like `:latest` or a static version number inside your `pods.yml` is an anti-pattern. A pipeline script like `changeTag.sh` dynamically captures the unique build identifier (such as `env.BUILD_NUMBER` or a Git commit SHA) and injects it into the manifest text before executing orchestration commands. This ensures that a unique, traceable image layer is deployed with every single pipeline run.

#### Q2: What command utility must be installed and configured on your Jenkins worker node to process `pods.yml` and `services.yml`?
* **Answer**: The Kubernetes command-line tool `kubectl` must be installed on the worker node. Additionally, a valid Kubernetes authentication token file—typically the `kubeconfig` bundle (`~/.kube/config`)—must be safely mounted or injected into the execution space via Jenkins Secure Credentials so `kubectl` has authorized permissions to reach and manage the target cluster.

---

### Topic 2: Node.js Container Isolation

#### Q3: Your repository includes a Node.js application (`package.json`). How should a production-ready `Dockerfile` be optimized to prevent bloated image layers?
* **Answer**: You should use a two-step optimization strategy:
  1. Use a lightweight base layer (such as `node:alpine` or `node:slim`) instead of full operating system images.
  2. Separate your dependency caching layer by copying the package files and executing installation steps *before* copying the remaining application codebase files. This ensures subsequent runs reuse cached image layers unless external dependencies change.
* **Example Structure**:
  ```dockerfile
  FROM node:18-alpine
  WORKDIR /app
  COPY package*.json ./
  RUN npm install --only=production
  COPY . .
  EXPOSE 3000
  CMD ["node", "server.js"]
  ```

---

### Topic 3: Kubernetes Deployment Architectures

#### Q4: What is the main operational risk of using a raw `pods.yml` file to run your application in production compared to using a Kubernetes `Deployment` object?
* **Answer**: A raw `Pod` configuration describes a single, standalone instance. If the node hosting that pod crashes or runs out of memory, the pod is destroyed and will not automatically restart or reschedule onto a healthy node. In production, workloads should be managed via a **Deployment** controller. Deployments manage replicasets, monitor pod health, scale instances automatically, and execute rolling updates without causing application downtime.

#### Q5: What is the relationship between the configurations defined in `pods.yml` and `services.yml`? How do they communicate?
* **Answer**: They communicate through **Labels and Selectors**. Inside `pods.yml`, the metadata section defines custom key-value pairs (e.g., `labels: app: my-node-app`). In `services.yml`, the network rule sets a corresponding selector targeting that exact label. The Service matches those endpoints dynamically, ensuring that routing traffic finds the correct pods even as they are destroyed and recreated with new IP addresses.

---

### Topic 4: Advanced GitOps & Scaling Challenges

#### Q6: How can you structure your Jenkinsfile to gracefully handle a scenario where a deployment update fails (e.g., due to an invalid image path or a runtime application crash)?
* **Answer**: You can include a post-deployment verification script using the `kubectl rollout status` command. If this command returns a non-zero exit status, the pipeline will fail, and you can trigger a fallback rollout step inside the pipeline's `post { failure { ... } }` block to safely revert to the previous working state.
* **Example**:
  ```groovy
  stage('Deploy to K8s') {
      steps {
          sh 'kubectl apply -f pods.yml -f services.yml'
          // If using a Deployment object:
          sh 'kubectl rollout status deployment/my-node-app-deployment --timeout=2m'
      }
  }
  ```


## Advanced Jenkins and Kubernetes (k8s) Interview Guide (Part 2)

This document contains supplementary, high-level technical interview questions and answers focusing on dynamic agent scaling, RBAC security, container hardening, and zero-downtime microservice architecture.

---

### 1. Advanced Jenkins-Kubernetes Integration

#### Q1: Instead of using a permanent Jenkins worker node with `kubectl` installed, how can you optimize your infrastructure to scale dynamically inside Kubernetes?
* **Answer**: You can utilize the **Jenkins Kubernetes Plugin**. This plugin dynamically spins up ephemeral, lightweight container pods inside the cluster to act as isolated Jenkins execution agents for a single build run. Once the pipeline finishes executing, the agent pod is automatically deleted, freeing up cluster resources.
* **Example Structure**:
  ```groovy
  pipeline {
      agent {
          kubernetes {
              yaml '''
  apiVersion: v1
  kind: Pod
  spec:
    containers:
    - name: maven
      image: maven:3.8.1-jdk-11
      command: ['cat']
      tty: true
    - name: kubectl
      image: bitnami/kubectl:latest
      command: ['cat']
      tty: true
  '''
          }
      }
      stages {
          stage('Build') {
              steps {
                  container('maven') { sh 'mvn clean package' }
              }
          }
          stage('Deploy') {
              steps {
                  container('kubectl') { sh 'kubectl apply -f deployment.yml' }
              }
          }
      }
  }
  ```

#### Q2: How can your Jenkins pipeline extract the assigned Cluster IP or External LoadBalancer URL of a newly created service from `services.yml` to pass to a testing stage?
* **Answer**: You use `kubectl get service` along with JSONPath filtering to capture the runtime networking attributes directly into a pipeline variable.
* **Example**:
  ```groovy
  script {
      def externalIp = sh(
          script: "kubectl get svc my-node-service -o jsonpath='{.status.loadBalancer.ingress.ip}'",
          returnStdout: true
      ).trim()
      echo "Testing application endpoint at: http://\${externalIp}:3000"
  }
  ```

---

### 2. Security and RBAC (Role-Based Access Control)

#### Q3: Why is checking a raw `kubeconfig` file directly into your source code repository a massive security violation, and what is the standard enterprise solution?
* **Answer**: A `kubeconfig` bundle contains root-level administrative cluster credentials, access tokens, and API endpoints. Checking it into SCM exposes your entire infrastructure cluster to anyone with repository access. The enterprise fix is to implement **Kubernetes Service Accounts** with tightly scoped **Role-Based Access Control (RBAC)** permissions. You generate a restricted token for Jenkins, map it only to the target namespace, and inject it safely during runtime using the Jenkins **Secret Text Credentials** or HashiCorp Vault.

#### Q4: How do you handle container runtime security for the Node.js application (`server.js`) inside your `Dockerfile`?
* **Answer**: By default, Docker containers run as the root user. If an attacker exploits a code vulnerability in your Node.js application, they gain root access to the underlying container namespace. To prevent this, configure a non-root system user inside the `Dockerfile` to drop privileges before executing the runtime command.
* **Example**:
  ```dockerfile
  FROM node:18-alpine
  WORKDIR /app
  COPY package*.json ./
  RUN npm install --only=production
  COPY . .
  # Switch to the built-in, unprivileged system 'node' user
  USER node 
  EXPOSE 3000
  CMD ["node", "server.js"]
  ```

---

### 3. High-Availability & Production Zero-Downtime

#### Q5: How do you configure your Kubernetes manifests to implement a rolling update strategy, ensuring zero downtime when your Jenkins job pushes a code change?
* **Answer**: You must transition from a raw `Pod` to a `Deployment` manifest and define a `RollingUpdate` strategy pattern. This allows you to configure rules that control how many pods can be taken offline and how many new pods can be spun up simultaneously during an update.
* **Example Manifest Snippet**:
  ```yaml
  spec:
    replicas: 3
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxSurge: 1       # Spins up 1 new pod before killing an old one
        maxUnavailable: 0 # Ensures all original replicas stay up during updates
  ```

#### Q6: If your Node.js application takes 20 seconds to establish database connections on startup, how do you prevent Kubernetes from routing user traffic to it prematurely?
* **Answer**: You implement **Readiness and Liveness Probes** inside the container definition block of your Kubernetes manifest.
  * **Readiness Probe**: Periodically checks an application health endpoint (e.g., `/health`). Kubernetes will hold back network traffic from reaching the container pod until this probe returns a successful `HTTP 200` response code.
  * **Liveness Probe**: Monitors the container to verify it hasn't entered a deadlocked state. If this probe fails repeatedly, the cluster automatically kills and restarts the faulty pod instance.
* **Example Manifest Snippet**:
  ```yaml
  readinessProbe:
    httpGet:
      path: /health
      port: 3000
    initialDelaySeconds: 15
    periodSeconds: 5
  ```
