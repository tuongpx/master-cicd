# What is Jenkins? Master/Agent Architecture
----
## 🔧 What is Jenkins?
- Jenkins is an open-source CI/CD (Continuous Integration / Continuous Delivery) automation server that helps development teams continuously build, test, and deploy software quickly, reliably, and at scale.

- It provides a flexible pipeline system that integrates with thousands of plugins, making Jenkins a central tool in modern DevOps workflows. With Jenkins, you can:
    - ✅ Continuous Integration (CI)
        - Automatically build and test code on every commit
        - Detect issues early and improve code quality

    - ✅ Continuous Delivery & Deployment (CD)
        - Deploy to dev/test/staging environments
        - Automate production deployments with strategies like Blue–Green or Canary
        - Enable fast, reliable releases

    - ✅ Docker & Container Workflows
        - Build and push Docker images
        - Run pipelines using Docker agents
        - Integrate with container registries

    - ✅ Kubernetes Deployment
        - Deploy apps to Kubernetes clusters
        - Apply manifests, run Helm charts
        - Integrate with GitOps tools like Argo CD

    - ✅ Infrastructure Automation
        - Run Terraform, Ansible, and other IaC tools
        - Provision or update cloud resources

    - ✅ Automated Testing & Security
        - Run unit, integration, and E2E tests
        - Perform security scans (SAST, DAST, dependency checks)
----

## 🏗️ Jenkins Architecture Overview
Jenkins uses a Master/Agent (Controller/Worker) architecture to distribute and scale workloads effectively

### 🖥️ Jenkins Master (Controller)
The Master is the central brain of Jenkins and is responsible for:

- Receiving and scheduling CI/CD jobs

- Managing pipelines and orchestrating task execution

- Assigning workloads to agents

- Monitoring build status and storing results

- Providing the Jenkins user interface (UI) and API

All user interactions happen through the Jenkins Master.

----

### ⚙️ Jenkins Agent (Worker Node)
Agents are machines that perform the actual work requested by the Master.
Each agent can run:

- Inside a Docker container

- As a virtual machine

- As a physical server

- As a Kubernetes pod

Jenkins supports multiple agents running in parallel, enabling scalable and efficient execution of:

- Builds

- Tests

- Deployments

Agents allow Jenkins to distribute workloads across different environments and isolate job execution.

----

## 🔄 How the Jenkins Workflow Operates

   [ Developer ]
         |
         | Commit code (GitLab)
         v
   [ Jenkins Master ]
         |
         |----------------------|
         |                      |
         v                      v
[ Agent 1: Docker Build ]   [ Agent 2: Kubernetes Deploy ]
         |
         v
[ Artifact Registry / Production Server ]
----
This structure gives Jenkins:

- Scalability → run many builds in parallel

- Flexibility → use specialized agents for Docker, Kubernetes, testing, etc.

- Security → isolate workloads in controlled agents

- Reliability → distribute tasks and avoid overloading a single node

Jenkins enables fully automated DevOps pipelines across diverse environments using the Master/Agent model.