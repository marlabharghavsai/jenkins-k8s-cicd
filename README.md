# Jenkins CI/CD on Kubernetes (Enterprise Grade)

## Objective
Build an enterprise-grade CI/CD pipeline using Jenkins running on Kubernetes with:
- Dynamic Jenkins agents
- Pipeline as Code
- SonarQube quality gates
- Trivy security scanning
- Docker image build & push
- Blue-Green deployment strategy

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
