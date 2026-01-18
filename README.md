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
Component                 	 Purpose
Jenkins	                     CI/CD orchestration
Kubernetes	                 Runtime platform & dynamic agents
Jenkins Kubernetes Plugin	   Dynamic agent provisioning
SonarQube	                   Static code analysis & quality gates
Nexus	                       Docker image registry
Trivy	                       Container vulnerability scanning
GitHub	                     Source code management
Demo App	                   Sample application
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
jenkins-k8s-cicd
├── Jenkinsfile
├── README.md
├── app/
│   └── demo/
├── k8s/
│   ├── demo-staging.yaml
│   ├── demo-blue.yaml
│   ├── demo-green.yaml
│   └── demo-prod-service.yaml
├── helm/
│   └── jenkins-values.yaml
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

## Dynamic Jenkins Agents
Jenkins uses the Kubernetes plugin to dynamically provision build agents as pods.
- Maven agent for build & tests
- Docker agent for image creation
Agents are created on demand and destroyed after job completion.
