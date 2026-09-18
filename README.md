# Automated Weather App Deployment with Jenkins and Kubernetes

## Project Overview

This project demonstrates an end-to-end **CI/CD workflow for a containerized multi-service weather application** deployed to a local Kubernetes cluster.

The solution uses:

- **GitHub** as the source-code repository.
- **Jenkins** as the CI/CD orchestrator.
- **Dynamic Kubernetes Jenkins agents** to execute pipeline workloads.
- **BuildKit** to build container images without requiring a Docker daemon inside the Jenkins agent.
- **Docker Hub** as the container registry.
- **Kubernetes / Minikube** as the deployment platform.
- **NGINX Ingress Controller** to expose the application.
- **TLS** for HTTPS access.
- **Kubernetes Secrets** for application credentials.
- **MySQL StatefulSet + persistent storage** for authentication data.

Repository:

```text
https://github.com/Ibrahim-Gamal-Ibrahim/automated-weather_app-deployment.git
```

---

## Architecture

```text
Developer
   |
   | git push
   v
GitHub Repository
   |
   v
Jenkins CI Pipeline
   |
   |-- Checkout source code
   |
   |-- Build Auth image -----------\
   |-- Build Weather image ---------|--> BuildKit --> Docker Hub
   |-- Build UI image -------------/
   |
   v
Trigger Jenkins CD Pipeline
   |
   |-- Create Kubernetes Secrets
   |-- Deploy MySQL StatefulSet
   |-- Initialize Database
   |-- Deploy Kubernetes Services
   |-- Deploy Auth Service
   |-- Deploy Weather Service
   |-- Deploy UI Service
   |-- Deploy NGINX Ingress
   |-- Verify Kubernetes rollouts
   |
   v
Kubernetes / Minikube
   |
   +-------------------------------+
   |                               |
   v                               v
NGINX Ingress                 MySQL StatefulSet
weatherapp.local                    |
   |                                |
   v                                |
UI Deployment                      |
   |                               |
   +----------+--------------------+
              |
       +------+------+
       |             |
       v             v
Auth Service    Weather Service
:8080           :5000
```

---

# Application Components

The application consists of three containerized services.

## 1. Authentication Service

Directory:

```text
auth/
```

Technology:

```text
Go
```

Container port:

```text
8080
```

Docker image:

```text
ibrahimgamal10203040/weatherapp-auth:latest
```

The authentication service communicates with MySQL using:

```text
DB_HOST=mysql
DB_USER=authuser
DB_NAME=weatherapp
DB_PORT=3306
```

Sensitive values such as the database password and JWT secret are injected from Kubernetes Secrets.

---

## 2. Weather Service

Directory:

```text
weather/
```

Technology:

```text
Python
```

Container port:

```text
5000
```

Docker image:

```text
ibrahimgamal10203040/weatherapp-weather:latest
```

The service receives its external weather API key from a Kubernetes Secret.

The Kubernetes Deployment runs:

```text
2 replicas
```

and defines both readiness and liveness probes.

---

## 3. UI Service

Directory:

```text
ui/
```

Technology:

```text
Node.js
```

Container port:

```text
3000
```

Docker image:

```text
ibrahimgamal10203040/weatherapp-ui:latest
```

The UI communicates internally with:

```text
weatherapp-auth:8080
weatherapp-weather:5000
```

The Kubernetes Deployment runs:

```text
2 replicas
```

and exposes a health endpoint:

```text
/health
```

---

# Repository Structure

```text
automated-weather_app-deployment/
|
|-- auth/
|   `-- Dockerfile
|
|-- weather/
|   `-- Dockerfile
|
|-- ui/
|   `-- Dockerfile
|
|-- mysql-init/
|
|-- kubernetes/
|   |
|   |-- authentication/
|   |   |-- deployment.yaml
|   |   |-- service.yaml
|   |   |
|   |   `-- mysql/
|   |       |-- headless-service.yaml
|   |       |-- statefulset.yaml
|   |       `-- init-job.yaml
|   |
|   |-- weather/
|   |   |-- deployment.yaml
|   |   `-- service.yaml
|   |
|   `-- ui/
|       |-- deployment.yaml
|       |-- service.yaml
|       `-- ingress.yaml
|
|-- jenkins/
|
|-- Jenkinsfile-ci
|-- Jenkinsfile-cd
`-- .gitignore
```

---

# CI/CD Workflow

The project separates Continuous Integration and Continuous Deployment into two Jenkins pipelines.

```text
Jenkinsfile-ci
Jenkinsfile-cd
```

---

# CI Pipeline

File:

```text
Jenkinsfile-ci
```

The pipeline runs on a Kubernetes Jenkins agent with the label:

```text
weatherapp-agent
```

## CI Stages

### 1. Checkout

The pipeline clones:

```text
https://github.com/Ibrahim-Gamal-Ibrahim/automated-weather_app-deployment.git
```

from the:

```text
main
```

branch.

---

### 2. Prepare Docker Hub Authentication

Jenkins retrieves the Docker Hub username/password credential:

```text
dockerhub-credentials
```

and creates a temporary Docker configuration file:

```text
/home/jenkins/agent/.docker/config.json
```

This configuration is used by BuildKit to authenticate with Docker Hub.

No Docker daemon or `/var/run/docker.sock` is required.

---

### 3. Build Images

The following images are built in parallel:

```text
ibrahimgamal10203040/weatherapp-auth:latest

ibrahimgamal10203040/weatherapp-weather:latest

ibrahimgamal10203040/weatherapp-ui:latest
```

BuildKit runs with:

```text
buildctl-daemonless.sh
```

Example:

```bash
buildctl-daemonless.sh build \
  --frontend dockerfile.v0 \
  --local context=auth \
  --local dockerfile=auth \
  --output type=image,name=ibrahimgamal10203040/weatherapp-auth:latest,push=true
```

The image is built and pushed directly to Docker Hub.

---

### 4. Trigger CD

After CI succeeds, Jenkins triggers:

```text
weatherapp-cd
```

using:

```groovy
build job: 'weatherapp-cd',
      wait: false,
      propagate: false
```

Therefore:

```text
GitHub
   |
   v
CI Pipeline
   |
   +--> Build Auth
   +--> Build Weather
   +--> Build UI
   |
   v
Docker Hub
   |
   v
Trigger CD Pipeline
```

---

# CD Pipeline

File:

```text
Jenkinsfile-cd
```

The deployment namespace is:

```text
default
```

The pipeline also runs on:

```text
weatherapp-agent
```

---

## CD Stages

### 1. Checkout

The latest repository version is cloned again so that the deployment pipeline uses the latest Kubernetes manifests.

---

### 2. Create Kubernetes Secrets

Jenkins reads sensitive values from the **Jenkins Credentials Store**.

The CD pipeline then creates or updates Kubernetes Secrets using:

```bash
kubectl create secret ... --dry-run=client -o yaml | kubectl apply -f -
```

This approach makes the operation idempotent.

The application secrets are therefore **not stored as plaintext inside the Git repository**.

---

### 3. Deploy MySQL

The pipeline deploys:

```text
kubernetes/authentication/mysql/headless-service.yaml
kubernetes/authentication/mysql/statefulset.yaml
```

MySQL uses:

```text
mysql:5.7
```

with:

```text
1 replica
```

and persistent storage:

```text
1Gi
```

using:

```text
storageClassName: standard
```

This is compatible with Minikube's default storage provisioner.

---

### 4. Initialize the Database

The pipeline runs:

```text
mysql-init-job
```

The job:

1. Waits until MySQL becomes available.
2. Creates the database:

```text
weatherapp
```

3. Creates the application database user:

```text
authuser
```

4. Grants access to the application database.

The pipeline waits for the Job to complete:

```bash
kubectl wait \
  --for=condition=complete \
  job/mysql-init-job \
  -n default \
  --timeout=120s
```

---

### 5. Deploy Kubernetes Services

Three internal ClusterIP Services are created:

```text
weatherapp-auth       :8080
weatherapp-weather    :5000
weatherapp-ui         :3000
```

These services provide stable Kubernetes DNS names between application components.

---

### 6. Deploy Application Components

The pipeline deploys:

```text
weatherapp-auth
weatherapp-weather
release-name-weatherapp-ui
```

If a Deployment already exists, Jenkins performs:

```bash
kubectl rollout restart deployment/<deployment>
```

This is important because the manifests use:

```text
imagePullPolicy: Always
```

and images are pushed using the:

```text
latest
```

tag.

The rollout restart forces the Pods to be recreated and pull the newest image.

---

### 7. Deploy Ingress

The application is exposed with an NGINX Ingress using:

```text
https://weatherapp.local
```

Ingress class:

```text
nginx
```

TLS secret:

```text
weatherapp-ui-tls
```

The Ingress routes traffic to:

```text
weatherapp-ui:3000
```

---

### 8. Verify Deployment

The pipeline waits for successful rollouts of:

```text
weatherapp-auth
weatherapp-weather
release-name-weatherapp-ui
```

and then displays:

```bash
kubectl get deployments
kubectl get statefulsets
kubectl get pods -o wide
kubectl get services
kubectl get ingress
```

---

# Prerequisites

Before running the project, the following components must exist.

## Local Machine

Install:

```text
Git
kubectl
Helm
Minikube
```

Check:

```bash
git --version

kubectl version --client

helm version

minikube version
```

---

# Start Minikube

Example:

```bash
minikube start
```

Verify:

```bash
kubectl get nodes
```

Expected:

```text
STATUS   Ready
```

---

# Enable NGINX Ingress

Enable the Minikube NGINX Ingress addon:

```bash
minikube addons enable ingress
```

Verify:

```bash
kubectl get pods -n ingress-nginx
```

The ingress-controller Pod should become:

```text
Running
```

---

# Jenkins Requirements

Jenkins must be installed and connected to the same Kubernetes cluster.

A typical setup is:

```text
Minikube
|
|-- jenkins namespace
|    |
|    |-- Jenkins Controller
|    `-- Dynamic Jenkins Agent Pods
|
`-- default namespace
     |
     `-- Weather Application
```

The Jenkins Kubernetes plugin creates temporary agent Pods to execute pipeline jobs.

---

# Required Jenkins Plugins

At minimum Jenkins should have plugins that provide:

```text
Pipeline / workflow support
Git
Credentials Binding
Kubernetes
```

Typical plugin names include:

```text
kubernetes
workflow-aggregator
git
credentials-binding
```

---

# Jenkins Kubernetes Cloud

Configure Jenkins to use Kubernetes dynamic agents.

Navigate to:

```text
Manage Jenkins
    ->
Clouds
    ->
New Cloud
    ->
Kubernetes
```

The Jenkins Kubernetes cloud must be able to create agent Pods in the cluster.

The pipeline expects an agent label:

```text
weatherapp-agent
```

---

# Required Jenkins Agent Containers

The Kubernetes Pod Template associated with:

```text
weatherapp-agent
```

must provide the containers referenced by the pipelines.

## Git Container

Required by both pipelines:

```text
git
```

It must contain the Git CLI.

Typical image:

```text
alpine/git
```

---

## kubectl Container

Required by the CD pipeline:

```text
kubectl
```

It must contain a compatible `kubectl` binary and run using a Kubernetes ServiceAccount with permission to deploy resources.

---

## BuildKit Containers

The CI pipeline expects:

```text
buildkit-auth
buildkit-weather
buildkit-ui
```

Each container must provide:

```text
buildctl-daemonless.sh
```

A rootless BuildKit image can be used for these containers.

The containers must share the Jenkins workspace so each BuildKit container can access the cloned source code.

They must also be able to write/read:

```text
/home/jenkins/agent/.docker
```

because this directory contains the temporary Docker Hub authentication configuration.

---

# Jenkins Credentials

The following credentials **must be added to the Jenkins Controller before running the pipelines**.

Navigate to:

```text
Manage Jenkins
    ->
Credentials
    ->
System
    ->
Global credentials
    ->
Add Credentials
```

The **Credential IDs must match exactly** because the Jenkinsfiles reference these IDs directly.

| Jenkins Credential ID | Jenkins Type | Purpose |
|---|---|---|
| `dockerhub-credentials` | Username with password | Authenticate BuildKit to Docker Hub |
| `mysql-root-password` | Secret text | MySQL root password |
| `mysql-auth-password` | Secret text | Password for application DB user `authuser` |
| `jwt-secret` | Secret text | JWT / authentication signing secret |
| `weather-api-key` | Secret text | External weather API key |
| `weatherapp-tls-crt` | Secret file | TLS certificate used by the application Ingress |
| `weatherapp-tls-key` | Secret file | Private key associated with the TLS certificate |

---

# 1. Docker Hub Credential

Create:

```text
Kind:
Username with password
```

Credential ID:

```text
dockerhub-credentials
```

Username:

```text
Your Docker Hub username
```

Password:

```text
Docker Hub password or preferably an access token
```

The current pipeline publishes images under:

```text
ibrahimgamal10203040
```

If another Docker Hub account is used, change:

```groovy
DOCKERHUB_USER = 'ibrahimgamal10203040'
```

inside:

```text
Jenkinsfile-ci
```

and update the image names in the Kubernetes Deployment manifests.

---

# 2. MySQL Root Password

Create:

```text
Kind:
Secret text
```

ID:

```text
mysql-root-password
```

Example value:

```text
a-strong-root-password
```

Do not commit the real value to Git.

---

# 3. Application Database Password

Create:

```text
Kind:
Secret text
```

ID:

```text
mysql-auth-password
```

This password is assigned to:

```text
authuser
```

and injected into the authentication service.

---

# 4. JWT Secret

Create:

```text
Kind:
Secret text
```

ID:

```text
jwt-secret
```

Use a long random secret.

Example generation:

```bash
openssl rand -hex 32
```

---

# 5. Weather API Key

Create:

```text
Kind:
Secret text
```

ID:

```text
weather-api-key
```

Use the API key required by the weather backend.

The CD pipeline creates:

```text
Secret name: weather
Key:         apikey
```

---

# 6. TLS Certificate

Create a Jenkins credential:

```text
Kind:
Secret file
```

ID:

```text
weatherapp-tls-crt
```

Upload:

```text
tls.crt
```

---

# 7. TLS Private Key

Create:

```text
Kind:
Secret file
```

ID:

```text
weatherapp-tls-key
```

Upload:

```text
tls.key
```

The CD pipeline combines both files into the Kubernetes TLS Secret:

```text
weatherapp-ui-tls
```

---

# Generate a Local TLS Certificate

For a local Minikube lab, a self-signed certificate can be generated.

```bash
openssl req \
  -x509 \
  -nodes \
  -days 365 \
  -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=weatherapp.local/O=weatherapp" \
  -addext "subjectAltName=DNS:weatherapp.local"
```

Upload:

```text
tls.crt -> weatherapp-tls-crt
tls.key -> weatherapp-tls-key
```

to Jenkins Credentials.

Do not commit:

```text
tls.key
```

to the Git repository.

---

# Kubernetes Secrets Created by Jenkins

The CD pipeline creates the following Kubernetes Secrets automatically.

## mysql-secret

Contains:

```text
root-password
auth-password
```

Created from:

```text
mysql-root-password
mysql-auth-password
```

---

## auth-secret

Contains:

```text
secret-key
```

Created from:

```text
jwt-secret
```

---

## weather

Contains:

```text
apikey
```

Created from:

```text
weather-api-key
```

---

## weatherapp-ui-tls

Type:

```text
kubernetes.io/tls
```

Created from:

```text
weatherapp-tls-crt
weatherapp-tls-key
```

---

# Jenkins RBAC Requirement

This is one of the most important prerequisites.

The Jenkins dynamic agent executes:

```bash
kubectl apply
kubectl get
kubectl wait
kubectl rollout restart
kubectl rollout status
```

against the:

```text
default
```

namespace.

Therefore, the **ServiceAccount used by the Jenkins agent must have RBAC permission in the `default` namespace**.

Having permissions only inside:

```text
jenkins
```

is not enough.

For example, if the agent uses:

```text
system:serviceaccount:jenkins:jenkins-agent
```

and the application is deployed to:

```text
default
```

then a Role/RoleBinding must grant that ServiceAccount access to the required resources in `default`.

---

# Example Jenkins Deployment Role

Create:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: weatherapp-deployer
  namespace: default
rules:
  - apiGroups: [""]
    resources:
      - services
      - secrets
      - pods
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      - delete

  - apiGroups: ["apps"]
    resources:
      - deployments
      - statefulsets
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      - delete

  - apiGroups: ["batch"]
    resources:
      - jobs
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      - delete

  - apiGroups: ["networking.k8s.io"]
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      - delete
```

---

# Example RoleBinding

If your Jenkins agent ServiceAccount is:

```text
jenkins-agent
```

inside namespace:

```text
jenkins
```

create:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: weatherapp-jenkins-deployer
  namespace: default

subjects:
  - kind: ServiceAccount
    name: jenkins-agent
    namespace: jenkins

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: weatherapp-deployer
```

Apply:

```bash
kubectl apply -f weatherapp-deployer-role.yaml

kubectl apply -f weatherapp-deployer-rolebinding.yaml
```

---

# Verify Jenkins Agent Permissions

Test permissions before running the CD pipeline.

```bash
kubectl auth can-i get services \
  -n default \
  --as=system:serviceaccount:jenkins:jenkins-agent
```

Expected:

```text
yes
```

Test Deployment access:

```bash
kubectl auth can-i create deployments.apps \
  -n default \
  --as=system:serviceaccount:jenkins:jenkins-agent
```

Expected:

```text
yes
```

Test Secrets:

```bash
kubectl auth can-i create secrets \
  -n default \
  --as=system:serviceaccount:jenkins:jenkins-agent
```

Expected:

```text
yes
```

Test Ingress:

```bash
kubectl auth can-i create ingresses.networking.k8s.io \
  -n default \
  --as=system:serviceaccount:jenkins:jenkins-agent
```

Expected:

```text
yes
```

---

# Create Jenkins Jobs

Create two Jenkins Pipeline jobs.

## CI Job

Job name example:

```text
weatherapp-ci
```

Pipeline definition:

```text
Pipeline script from SCM
```

SCM:

```text
Git
```

Repository:

```text
https://github.com/Ibrahim-Gamal-Ibrahim/automated-weather_app-deployment.git
```

Branch:

```text
*/main
```

Script Path:

```text
Jenkinsfile-ci
```

---

## CD Job

The CD Job **must be named**:

```text
weatherapp-cd
```

because the CI Jenkinsfile triggers exactly this job name.

Pipeline definition:

```text
Pipeline script from SCM
```

Repository:

```text
https://github.com/Ibrahim-Gamal-Ibrahim/automated-weather_app-deployment.git
```

Branch:

```text
*/main
```

Script Path:

```text
Jenkinsfile-cd
```

---

# Hostname Configuration

The Ingress expects:

```text
weatherapp.local
```

Get the Minikube IP:

```bash
minikube ip
```

Example:

```text
192.168.49.2
```

Add it to:

```text
/etc/hosts
```

Example:

```bash
echo "$(minikube ip) weatherapp.local" | sudo tee -a /etc/hosts
```

Check:

```bash
getent hosts weatherapp.local
```

---

# Run the Pipeline

Run the CI pipeline:

```text
weatherapp-ci
```

The expected flow is:

```text
Checkout
   |
Prepare Docker Auth
   |
Build Auth --------\
Build Weather ------+--> Docker Hub
Build UI -----------/
   |
Trigger weatherapp-cd
   |
Create Kubernetes Secrets
   |
Deploy MySQL
   |
Initialize Database
   |
Deploy Services
   |
Deploy Auth
   |
Deploy Weather
   |
Deploy UI
   |
Deploy Ingress
   |
Verify Rollouts
```

---

# Verify Kubernetes Resources

Run:

```bash
kubectl get pods
```

Then:

```bash
kubectl get deployments
```

```bash
kubectl get statefulsets
```

```bash
kubectl get services
```

```bash
kubectl get jobs
```

```bash
kubectl get ingress
```

```bash
kubectl get secrets
```

---

# Expected Main Resources

Deployments:

```text
weatherapp-auth
weatherapp-weather
release-name-weatherapp-ui
```

StatefulSet:

```text
mysql
```

Job:

```text
mysql-init-job
```

Services:

```text
mysql
weatherapp-auth
weatherapp-weather
weatherapp-ui
```

Ingress:

```text
weatherapp-ui-ingress
```

Secrets:

```text
mysql-secret
auth-secret
weather
weatherapp-ui-tls
```

---

# Access the Application

Open:

```text
https://weatherapp.local
```

Because the example uses a self-signed certificate, the browser may display a certificate warning.

---

# Troubleshooting

## Jenkins Agent Cannot Access Kubernetes Resources

Example:

```text
Error from server (Forbidden):
services "mysql" is forbidden:
User "system:serviceaccount:jenkins:jenkins-agent"
cannot get resource "services"
```

Cause:

The Jenkins ServiceAccount does not have sufficient RBAC permissions in the target namespace.

Verify:

```bash
kubectl auth can-i get services \
  -n default \
  --as=system:serviceaccount:jenkins:jenkins-agent
```

Fix the Role / RoleBinding if the result is:

```text
no
```

---

## Jenkins Cannot Find Agent Label

Example:

```text
Jenkins doesn't have label 'weatherapp-agent'
```

Check:

```text
Manage Jenkins
->
Clouds
->
Kubernetes
->
Pod Templates
```

Confirm the Pod Template label is:

```text
weatherapp-agent
```

---

## Jenkins Cannot Find a Container

Example:

```text
container kubectl not found
```

The Pod Template does not contain the container expected by the Jenkinsfile.

Required container names include:

```text
git
kubectl
buildkit-auth
buildkit-weather
buildkit-ui
```

The names must match the Jenkinsfile exactly.

---

## BuildKit Authentication Failure

If Docker Hub returns:

```text
unauthorized
```

verify:

```text
dockerhub-credentials
```

and ensure the credential contains a valid Docker Hub username and access token/password.

---

## Image Does Not Update

The project currently uses:

```text
latest
```

image tags.

The CD pipeline handles this by running:

```bash
kubectl rollout restart
```

for existing Deployments.

Confirm:

```bash
kubectl rollout status deployment/weatherapp-auth
```

and inspect the image:

```bash
kubectl describe deployment weatherapp-auth
```

---

## MySQL Init Job Fails

Inspect:

```bash
kubectl get jobs
```

Then:

```bash
kubectl logs job/mysql-init-job
```

Also verify MySQL:

```bash
kubectl get pods -l app=mysql
```

and:

```bash
kubectl logs mysql-0
```

---

## PVC Is Pending

Check:

```bash
kubectl get pvc
```

Then:

```bash
kubectl get storageclass
```

The MySQL StatefulSet expects:

```text
storageClassName: standard
```

Minikube normally provides this StorageClass.

---

## Ingress Does Not Work

Check:

```bash
kubectl get pods -n ingress-nginx
```

Then:

```bash
kubectl get ingress
```

Check DNS/hosts resolution:

```bash
getent hosts weatherapp.local
```

Test with:

```bash
curl -k https://weatherapp.local
```

---

# Security Decisions

The project intentionally avoids storing application secrets directly in the Git repository.

Sensitive values are stored inside:

```text
Jenkins Credentials Store
```

and converted during deployment into:

```text
Kubernetes Secrets
```

The flow is:

```text
Jenkins Credentials
        |
        v
withCredentials()
        |
        v
Temporary environment variable/file
        |
        v
kubectl create secret
        |
        v
Kubernetes Secret
        |
        v
Application Pod
```

This means passwords, JWT secrets, API keys, and TLS private keys do not need to be committed to source control.

---

# DevOps Concepts Demonstrated

This project demonstrates practical experience with:

- Jenkins declarative pipelines
- CI/CD pipeline separation
- Kubernetes-based Jenkins agents
- Containerized multi-service applications
- Docker image creation
- Rootless / daemonless BuildKit
- Docker registry authentication
- Parallel image builds
- Kubernetes Deployments
- Kubernetes StatefulSets
- Kubernetes Jobs
- Kubernetes Services
- Kubernetes Secrets
- Kubernetes RBAC
- Persistent Volumes
- Readiness probes
- Liveness probes
- NGINX Ingress
- TLS termination
- Service discovery
- Pipeline-to-pipeline triggering
- Automated rollout verification
- Secret management
- Application deployment automation

---

# Possible Production Improvements

This project is designed as a hands-on CI/CD and Kubernetes lab. For a more production-oriented implementation, consider:

- Replace `latest` tags with immutable image tags such as the Git commit SHA.
- Add automated application/unit tests before image publishing.
- Add image vulnerability scanning.
- Add static code analysis.
- Add Helm or Kustomize for environment-specific deployments.
- Add resource requests and limits.
- Add Horizontal Pod Autoscaling.
- Add NetworkPolicies.
- Use an external secret manager such as HashiCorp Vault or a cloud secret manager.
- Use a trusted CA certificate instead of a self-signed certificate.
- Use a managed or highly available database.
- Add Prometheus/Grafana monitoring.
- Add centralized logging.
- Add deployment rollback handling.
- Add separate staging and production namespaces.
- Use GitOps tooling such as Argo CD for Continuous Deployment.

---

# Summary

This repository implements an automated CI/CD workflow in which Jenkins:

1. Checks out the application source from GitHub.
2. Authenticates to Docker Hub using Jenkins Credentials.
3. Builds three application images in parallel using BuildKit.
4. Pushes the images to Docker Hub.
5. Triggers a separate CD pipeline.
6. Converts Jenkins-managed secrets into Kubernetes Secrets.
7. Deploys MySQL with persistent storage.
8. Initializes the application database.
9. Deploys the authentication, weather, and UI services.
10. Exposes the UI using an NGINX Ingress with TLS.
11. Restarts existing workloads when new `latest` images are published.
12. Verifies all application rollouts.

The project demonstrates a complete Jenkins-to-Kubernetes deployment workflow while keeping sensitive values outside the Git repository.
