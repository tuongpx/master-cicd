# Hands-on Jenkins Setting Up

## 📌 Introduction
There are multiple ways to deploy Jenkins depending on your infrastructure and operational requirements. Common deployment options include containers, virtual machines, on-prem servers, and even Kubernetes clusters.

In this lab, we will demonstrate two practical installation methods:

### Deploy Jenkins using Docker Compose

- Fast, portable, easy to maintain

- Suitable for testing, labs, and container-based environments

### Install Jenkins directly on a virtual machine (self-hosted)

- Traditional installation method

- Suitable for dedicated servers or environments without container infrastructure

These two approaches will help you understand both containerized and non-containerized Jenkins setups, giving you flexibility to choose the best deployment model for your DevOps workflow.

### 1. Deploy Jenkins using Docker Compose
#### 
Project folder structure
```bash
cicd-install/
├── docker-compose.yml
├── gitlab
│   ├── config
│   ├── data
│   └── logs
└── jenkins
    └── Dockerfile
```
- Create 'docker-compose.yml' file
```bash
version: '3.9'

services:
  gitlab:
    image: 'gitlab/gitlab-ce:latest'
    container_name: gitlab
    restart: always
    hostname: 'gitlab.defenselab.info'
    extra_hosts:
      - "jenkins.defenselab.info:host-gateway"
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.defenselab.info'
        gitlab_rails['initial_root_password'] = 'your_strong_password'

        # Registry configuration
        registry_external_url 'https://gitlab.defenselab.info:5050'
        gitlab_rails['registry_enabled'] = true
        gitlab_rails['registry_host'] = "gitlab.defenselab.info"
        gitlab_rails['registry_port'] = "5050"
        gitlab_rails['registry_api_url'] = "http://localhost:5050"

    ports:
      - '80:80'
      - '443:443'
      - '22:22'
      - '5050:5050'
    volumes:
      - './config:/etc/gitlab'
      - './logs:/var/log/gitlab'
      - './data:/var/opt/gitlab'
    shm_size: '256m'
    networks:
      gitlab-net:
        aliases:
          - gitlab.defenselab.info

  gitlab-runner:
    image: 'gitlab/gitlab-runner:latest'
    container_name: gitlab-runner
    privileged: true
    depends_on:
      - gitlab
    restart: always
    volumes:
      - './runner-config:/etc/gitlab-runner'
      - '/var/run/docker.sock:/var/run/docker.sock'
    networks:
      - gitlab-net

  jenkins:
    build:
      context: ./jenkins
    container_name: jenkins
    restart: unless-stopped
    privileged: true
    hostname: jenkins.defenselab.info
    user: root
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - /home/secops/.kube:/root/.kube
      - /home/secops/.minikube:/root/.minikube
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - gitlab-net

# -----------------------------
# Volumes
# -----------------------------
volumes:
  jenkins_home:
    name: jenkins_home

# -----------------------------
# Networks
# -----------------------------
networks:
  gitlab-net:
    name: gitlab-net
```
- Create `jenkins/Dockerile`
```bash
FROM jenkins/jenkins:lts

USER root

# Install Docker, kubectl and dependencies
RUN apt-get update && \
    apt-get install -y ca-certificates curl apt-transport-https gnupg2 lsb-release docker.io && \
    curl -fsSL "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" -o /usr/local/bin/kubectl && \
    chmod +x /usr/local/bin/kubectl /usr/bin/docker || true && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

USER jenkins
```
----

- Run this command `docker compose up -d` to setting up jenkins.
- After Jenkins start, access to web gui: `http://jenkins.defenselab.info:8080`
- Take password for the first time login
```bash
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```
----
### 2. Install Jenkins directly on a virtual machine