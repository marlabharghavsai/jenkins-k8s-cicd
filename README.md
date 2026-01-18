# Enterprise CI/CD Pipeline on Kubernetes using Jenkins
## Project Overview
- This project implements an enterprise-grade CI/CD pipeline using Jenkins running on Kubernetes with dynamic agent provisioning.
- The pipeline automates the complete software delivery lifecycle—from Git commit to production deployment—with strong quality, security, and deployment gates.
- The solution integrates:
  - Jenkins (Pipeline as Code)
  - Kubernetes (dynamic build agents & deployments)
  - SonarQube (code quality & quality gates)
  - Trivy (container security scanning)
  - Nexus Repository (Docker image registry)
  - Blue-Green deployment strategy for zero-downtime production releases

## Architecture Overview
### Components
```
Component                 	             Purpose
Jenkins	                              CI/CD orchestration
Kubernetes	                          Runtime platform & dynamic agents
Jenkins Kubernetes Plugin	          Dynamic agent provisioning
SonarQube	                          Static code analysis & quality gates
Nexus	                              Docker image registry
Trivy	                              Container vulnerability scanning
GitHub	                              Source code management
Demo App	                          Sample application
```

## Architecture Diagram (Logical)
```
┌──────────┐
│  GitHub  │
│  (Push)  │
└────┬─────┘
     │ Webhook
     ▼
┌──────────────┐
│   Jenkins    │
│ (Controller) │
└────┬─────────┘
     │ Dynamic Pods
     ▼
┌──────────────────────────────────┐
│ Kubernetes Cluster               │
│                                  │
│  ┌────────────┐   ┌───────────┐  │
│  │ Maven Agent│   │DockerAgent│  │
│  └────────────┘   └───────────┘  │
│                                  │
│  ┌────────────┐   ┌──────────┐   │
│  │ SonarQube  │   │  Nexus   │   │
│  └────────────┘   └──────────┘   │
│                                  │
│  ┌───────────────┐               │
│  │ Blue / Green  │               │
│  │ Deployments   │               │
│  └───────────────┘               │
└──────────────────────────────────┘
```
## Objective
Build an enterprise-grade CI/CD pipeline using Jenkins running on Kubernetes with:
- Dynamic Jenkins agents
- Pipeline as Code
- SonarQube quality gates
- Trivy security scanning
- Docker image build & push
- Blue-Green deployment strategy

## CI/CD Pipeline Stages
### Checkout
- Pulls source code from GitHub
- Triggered automatically via webhook or manually

### Build & Unit Test
- Uses Maven agent
- Compiles code and runs unit tests

### SonarQube Analysis
- Performs static code analysis
- Uploads results to SonarQube server

### Quality Gate
- Pipeline fails automatically if quality gate is not met

### Build & Push Docker Image
- Uses Docker agent
- Builds Docker image
- Pushes image to Nexus Docker Hosted repository

### Security Scan (Trivy)
- Scans the built Docker image
- Pipeline fails if CRITICAL vulnerabilities are found

### Deploy to Staging
- Automatically deploys to staging environment
- Kubernetes Deployment updated with new image

### Manual Approval (Production)
- Requires human approval before production deployment

### Production Deployment (Blue-Green)
- Deploys new version to GREEN environment
- Switches traffic using Kubernetes Service
- Zero-downtime deployment
- Easy rollback by switching back to BLUE

## Blue-Green Deployment Flow
- Production traffic initially points to BLUE
- New version deployed to GREEN
- Health & rollout checks performed
- Service selector switched to GREEN
- Old BLUE version remains available for rollback

## Rollback Strategy
- If issues are detected:
```
kubectl patch svc demo-prod -n demo \
  -p '{"spec":{"selector":{"app":"demo","version":"blue"}}}'
```
- Rollback is instant and does not require redeploying.

## Prerequisites
### Required Tools
- Docker Desktop (with Kubernetes enabled)
- kubectl
- Helm
- Git
- GitHub account

## Kubernetes Setup
- Namespaces
```
kubectl create namespace jenkins
kubectl create namespace demo
kubectl create namespace nexus
kubectl create namespace sonarqube
```
## Jenkins Installation (Helm)
```
helm repo add jenkins https://charts.jenkins.io
helm repo update

helm install jenkins jenkins/jenkins \
  -n jenkins \
  -f values.yaml
```

### Jenkins uses a PersistentVolumeClaim to preserve:
- Jobs
- Plugins
- Credentials
- Configuration

## Jenkins Credentials
```
Credential ID	            Purpose
nexus-creds	              Nexus Docker login
sonarqube-token	          SonarQube authentication
kubeconfig	              Kubernetes access
```
- No secrets are hardcoded in Jenkinsfile

## Nexus Configuration
- Docker Hosted Repository: docker-hosted
- Exposed via NodePort
- Image format:
```
localhost:30083/docker-hosted/demo:<BUILD_NUMBER>
```

## SonarQube Configuration
- SonarQube deployed in Kubernetes
- Jenkins configured with SonarQube server
- Quality gate enforced in pipeline

## Kubernetes Manifests
- Included in repository:
- demo-staging.yaml
- demo-blue.yaml
- demo-green.yaml
- demo-prod-service.yaml
### Resource limits defined
### imagePullSecrets configured for Nexus


## Repository Structure
```
jenkins-k8s-cicd/
├── app/
│   └── demo/
│       ├── .mvn/wrapper/
│       ├── src/
│       │   ├── main/java/com/example/demo
│       │   └── test/java/com/example/demo
│       ├── .gitattributes
│       ├── .gitignore
│       ├── Dockerfile
│       ├── mvnw
│       ├── mvnw.cmd
│       └── pom.xml
│
├── helm/
│   └── jenkins-values.yaml
│
├── k8s/
│   ├── jenkins-rbac.yaml
│   ├── namespaces.yaml
│   ├── nexus.yaml
│   └── sonarqube.yaml
│
├── Jenkinsfile
├── README.md
│
├── demo-blue.yaml
├── demo-green.yaml
├── demo-prod-svc.yaml
├── demo-staging.yaml
│
└── docs/
   └── architecture-diagram.png
```

## Tech Stack
- Jenkins (Helm)
- Kubernetes (Docker Desktop / Minikube)
- GitHub
- SonarQube
- Trivy
- Nexus Repository
- Spring Boot application

## Environments
- Staging (Auto Deploy)
- Production (Manual Approval + Blue-Green)

## Kubernetes RBAC Configuration for Jenkins
- This project follows Kubernetes Role-Based Access Control (RBAC) best practices to ensure Jenkins has only the minimum required permissions to operate.

### Jenkins Service Account
- Jenkins runs inside the Kubernetes cluster using a dedicated ServiceAccount:
```
kubectl get serviceaccount -n jenkins
```
- This ServiceAccount is used by Jenkins controller and dynamically created agent pods.

## Permissions Scope
### Jenkins requires permissions to:
- Create and delete build agent pods
- Read pod logs
- Deploy applications to Kubernetes
- Update Deployments and Services for blue-green deployments
### All permissions are namespace-scoped and restricted to only required API resources.

## RBAC Role Example
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: jenkins-deployer
  namespace: demo
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch", "create", "delete", "patch"]
```
## RoleBinding Example
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-deployer-binding
  namespace: demo
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: jenkins
roleRef:
  kind: Role
  name: jenkins-deployer
  apiGroup: rbac.authorization.k8s.io
```
## Security Best Practices
- Jenkins uses least-privilege RBAC
- No cluster-admin access is granted
- All permissions are restricted to required namespaces
- Secrets are managed via Jenkins Credentials
- This ensures secure and controlled CI/CD operations inside Kubernetes.

## Dynamic Jenkins Agents
Jenkins uses the Kubernetes plugin to dynamically provision build agents as pods.
- Maven agent for build & tests
- Docker agent for image creation
Agents are created on demand and destroyed after job completion.



## Jenkins UI Note (Local Environment Limitation)
Note on Jenkins UI Access During Demo Recording

- During the final stage of this project, the Jenkins controller pod encountered a local environment issue on my laptop related to Docker Desktop / WSL2 resource constraints while using Helm-based Jenkins deployment with persistent volumes.
As a result, the Jenkins UI could not be accessed reliably for screen recording at the time of demo video creation.

- This issue is environment-specific and not related to the Jenkinsfile, pipeline logic, or Kubernetes configuration provided in this repository.

### The CI/CD pipeline was previously executed successfully, demonstrating:

- Dynamic Kubernetes agent provisioning
- Multi-stage pipeline execution
- SonarQube quality gate enforcement
- Docker image build and push to Nexus
- Trivy security scan stage
- Automated staging deployment
- Manual approval gate
- Blue–green production deployment strategy

On a fresh Kubernetes cluster or another machine, Jenkins starts normally using the same Helm chart and configuration, and the pipeline runs as expected.

- To ensure clarity, this repository includes:
- A complete Jenkinsfile (Pipeline as Code)
- All Kubernetes manifests used for deployment
- Architecture diagram illustrating the CI/CD flow
- Detailed documentation explaining each stage of the pipeline

