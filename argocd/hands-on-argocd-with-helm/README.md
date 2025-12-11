# Deployment Application with ArgoCD and Helm

## Prepare environment

- In the "Hands-0n-jenkins-pipeline-with-manual-approval" I have already setup Jenkinsfile with helm deployment. And I'm still use the repository `http://gitlab.defenselab.info/defenselab/corejs.git`.

- The Helm chart directory structure will be organized as follows:
```bash
helmchart/
└── corejs
    ├── Chart.yaml
    ├── templates
    │   ├── backend-deployment.yaml
    │   ├── backend-service.yaml
    │   ├── frontend-deployment.yaml
    │   ├── frontend-service.yaml
    │   └── ingress.yaml
    └── values.yaml
```
## Create application on ArgoCD
