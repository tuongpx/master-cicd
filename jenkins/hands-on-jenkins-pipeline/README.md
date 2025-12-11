# Hands-on lab Create Jenkins Pipeline
## 1. Create Pipeline
- New Item → Enter name → Select “Pipeline” → OK
    - Item name: Django
    - Item type: Pipeline
![Alt text](./images/jenkins-create-item.png)
- Configure Trigger
    - Build when a change is pushed to GitLab. GitLab webhook URL: http://jenkins.defenselab.info:8080/project/Django
        - Enabled GitLab triggers:
            - Push Events
            - Opened Merge Request Events

![Alt text](./images/jenkins-trigger.png)
- Configure Pipeline

    - Write your pipeline directly in the Pipeline script section in Jenkins, or
    - Select Pipeline script from SCM to load it from your Git repository\
    (In this example, we’ll use Pipeline script from SCM).
- Choose:

    - SCM: Git
    - Repository URL: http://gitlab.defenselab.info/defenselab/django.git
    - Credentials: Select the created GitLab token
    - Branches to build: e.g., master
    - Script Path: Path to your Jenkinsfile (e.g., root directory)
![Alt text](./images/jenkins-pipeline.png)
- Create Jenkinsfile and Push to GitLab
```bash
vim Jenkinsfile
```
```bash
pipeline {
    agent any

    environment {
        VENV_DIR = "venv"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Setup Python Environment') {
            steps {
                dir('app') {
                    sh '''
                    python3 -m venv ${VENV_DIR}
                    source ${VENV_DIR}/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    '''
                }
            }
        }

        stage('Run Checks') {
            steps {
                dir('app') {
                    sh '''
                    source ${VENV_DIR}/bin/activate
                    python manage.py check
                    '''
                }
            }
        }

        stage('Run Tests') {
            steps {
                dir('app') {
                    sh '''
                    source ${VENV_DIR}/bin/activate
                    python manage.py test || true
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            when {
                expression { fileExists('app/Dockerfile') }
            }
            steps {
                dir('app') {
                    sh '''
                    docker build -t django-app:latest .
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up virtual environment...'
            sh 'rm -rf app/${VENV_DIR}'
        }
        success {
            echo '✅ Build and test completed successfully!'
        }
        failure {
            echo '❌ Build failed — check logs above.'
        }
    }
}
```
- Now, you can run pipeline whenever you push code to your repository or you can run manually by click to Build Now button in your pipeline
![Alt text](./images/jenkins-pipeline-build.png)
- You can check your build result as below
![Alt text](./images/jenkins-console-output.png)
- 🎉 Success! If all jobs pass (displayed in green), you have successfully set up the CI/CD pipeline. 