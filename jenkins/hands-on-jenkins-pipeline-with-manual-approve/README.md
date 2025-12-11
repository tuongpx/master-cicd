# Hands on Lab Continuous Delivery Pipeline with Manual Approval
This lab demonstrates how to build a Continuous Delivery (CD) pipeline in Jenkins, where deployment to the Production environment requires manual approval from a privileged user (CTO). All steps before production deployment are fully automated.

## 🎯 Objective

Build a Jenkins pipeline that:

- Automatically builds frontend + backend Docker images

- Pushes the images to Harbor (Of course, in this lab i've already installed Harbor as Docker Registry)

- Deploys to a Staging-like environment (k8s) only after manual approval

- Demonstrates a classic Continuous Delivery workflow

## 📘 Scenario Overview

- A developer commits code to a GitLab repository.

- Jenkins automatically:

    - Builds Docker images for both frontend and backend

    - Pushes the images to Harbor Registry

- Before deployment to Production, the pipeline pauses and requires manual approval from the CTO.

- Once approved, Jenkins deploys the updated application to Minikube (local Kubernetes cluster).

This mirrors real-world Continuous Delivery practices where production releases require human validation.

## 🛠️ Hands-On: Jenkinsfile Used in the Lab

The Jenkinsfile implements:

- Docker builds

- Registry authentication

- Image push

- Manual approval

- Kubernetes deployment

### Create Pipeline
- New Item → Enter name → Select “Pipeline” → OK
    - Item name: corejs
    - Item type: Pipeline

- Configure Trigger
    - Build when a change is pushed to GitLab. GitLab webhook URL:  http://jenkins.defenselab.info:8080/project/CoreJS
        - Enabled GitLab triggers:
            - Push Events
            - Opened Merge Request Events
            - 

![Alt text](./images/jenkins-trigger-01.png)
- Configure Pipeline

    - Write your pipeline directly in the Pipeline script section in Jenkins, or
    - Select Pipeline script from SCM to load it from your Git repository\
    (In this example, we’ll use Pipeline script from SCM).
- Choose:

    - SCM: Git
    - Repository URL: http://gitlab.defenselab.info/defenselab/corejs.git
    - Credentials: Select the credential that was created in the previous lab.
    - Branches to build: e.g., master
    - Script Path: Path to your Jenkinsfile (e.g., root directory)

![Alt text](./images/jenkins-pipeline-01.png)
### Create Jenkinsfile and Push to GitLab
```bash
vim Jenkinsfile
```
```bash
// Jenkinsfile - corejs (updated, sandbox-safe)
// GitLab -> Jenkins -> Build FE/BE -> Push -> Helm deploy
def gitRepository = 'http://gitlab.defenselab.info/defenselab/corejs.git'
def gitBranch = 'master'

def frontendImageRepo = 'registry.defenselab.info/defenselab/corejs/corejs-frontend'
def backendImageRepo  = 'registry.defenselab.info/defenselab/corejs/corejs-backend'
def namespace = 'corejs'

def helmRelease = 'corejs'
def helmChart   = 'helmchart/corejs'
def helmValues  = 'helmchart/corejs/values.yaml'

def gitlabCredential = 'gitlab-user-ci'
def registryCredential = 'jenkins-harbor'   // Harbor credential in Jenkins
def kubeconfigCredential = 'kubeconfig-cred' // OPTIONAL: secret file credential id (Secret file). If not present, will fallback to agent kubecontext.

def version = "prod-0.${BUILD_NUMBER}"

pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'https://registry.defenselab.info'
        FRONTEND_IMAGE  = "${frontendImageRepo}"
        BACKEND_IMAGE   = "${backendImageRepo}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checkout ${gitBranch} from ${gitRepository}"
                git branch: gitBranch, credentialsId: gitlabCredential, url: gitRepository
                sh 'git reset --hard'
            }
        }

        stage('Pre-checks') {
            steps {
                echo "Pre-flight checks"
                sh 'docker --version || true'
                sh 'helm version --short || true'
            }
        }

        stage('Build & Push Images (parallel)') {
            parallel {
                stage('Frontend: Build & Push') {
                    steps {
                        script {
                            echo "Building & pushing frontend image: ${FRONTEND_IMAGE}:${version}"
                            docker.withRegistry("${DOCKER_REGISTRY}", registryCredential) {
                                sh """
                                    docker build -t ${FRONTEND_IMAGE}:${version} -f frontend/Dockerfile ./frontend
                                    docker push ${FRONTEND_IMAGE}:${version}
                                    docker tag ${FRONTEND_IMAGE}:${version} ${FRONTEND_IMAGE}:latest || true
                                    docker push ${FRONTEND_IMAGE}:latest || true
                                """
                            }
                        }
                    }
                }

                stage('Backend: Build & Push') {
                    steps {
                        script {
                            echo "Building & pushing backend image: ${BACKEND_IMAGE}:${version}"
                            docker.withRegistry("${DOCKER_REGISTRY}", registryCredential) {
                                sh """
                                    docker build -t ${BACKEND_IMAGE}:${version} -f CoreAPI/Dockerfile .
                                    docker push ${BACKEND_IMAGE}:${version}
                                    docker tag ${BACKEND_IMAGE}:${version} ${BACKEND_IMAGE}:latest || true
                                    docker push ${BACKEND_IMAGE}:latest || true
                                """
                            }
                        }
                    }
                }
            }
        }

        stage('Confirm from CTO') {
            steps {
                script {
                    timeout(time: 1, unit: 'HOURS') {
                        input message: "CTO approval required to push Docker images. Proceed?",
                        ok: "Approve and Continue",
                        submitter: "CTO"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes (Helm)') {
            steps {
                script {
                    // Try to use kubeconfig credential file if it exists (wrap in try/catch)
                    try {
                        withCredentials([file(credentialsId: kubeconfigCredential, variable: 'KUBECONFIG_FILE')]) {
                            echo "Using kubeconfig from credential: ${kubeconfigCredential}"
                            sh '''
                                export KUBECONFIG="${KUBECONFIG_FILE}"
                                helm lint ${HELM_CHART} || true
                                helm upgrade --install ${HELM_RELEASE} ${HELM_CHART} \
                                  --namespace ${NAMESPACE} --create-namespace \
                                  -f ${HELM_VALUES} \
                                  --set frontend.image.repository=${FRONTEND_IMAGE} \
                                  --set frontend.image.tag=${VERSION} \
                                  --set backend.image.repository=${BACKEND_IMAGE} \
                                  --set backend.image.tag=${VERSION}
                            '''.stripIndent()
                        }
                    } catch (err) {
                        // if credential not present or other error, fallback to cluster context on agent
                        echo "No kubeconfig credential available or failed to read it — falling back to agent kubecontext"
                        sh """
                            helm lint ${helmChart} || true
                            helm upgrade --install ${helmRelease} ${helmChart} \
                              --namespace ${namespace} --create-namespace \
                              -f ${helmValues} \
                              --set frontend.image.repository=${FRONTEND_IMAGE} \
                              --set frontend.image.tag=${version} \
                              --set backend.image.repository=${BACKEND_IMAGE} \
                              --set backend.image.tag=${version}
                        """
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh """
                    kubectl get ns ${namespace} -o jsonpath='{.metadata.name}' || true
                    kubectl get pods -n ${namespace} -o wide || true
                    kubectl get svc -n ${namespace} || true
                    kubectl rollout status deployment/corejs-backend -n ${namespace} --timeout=120s || true
                    kubectl rollout status deployment/corejs-frontend -n ${namespace} --timeout=120s || true
                """
            }
        }
    }

    post {
        success { echo "SUCCESS: corejs ${version} deployed to ${namespace}" }
        failure { echo "FAILURE: check logs above" }
        always  { echo "Pipeline finished at: ${new Date()}" }
    }
}

```
### Note
- The Confirm from CTO stage pauses the pipeline until a user with the role CTO approves the deployment.

- This is the key element that transforms CI/CD into Continuous Delivery.

### Verify
- cnDue to the trigger configuration, any change in GitLab will automatically notify Jenkins CI, and all stages will become visible in the pipeline.
![Alt text](./images/CTO-confirm-stage.png)
- You can check your build result as below
![Alt text](./images/CTO-approve.png)
- 🎉 Success! If all jobs pass (displayed in green), you have successfully set up the CI/CD pipeline  with manual approve. 

### ✅ Outcome of the Lab

After completing this hands-on lab, you will understand how to:

- Implement multi-stage CI/CD pipelines

- Integrate Jenkins with Harbor Registry

- Add manual approval gates

- Deploy applications to Kubernetes (k8s)

- Apply Continuous Delivery best practices in Jenkins