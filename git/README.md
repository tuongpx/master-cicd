# 🔐 A Quick Git Guide
Git is one of the essential tools in any DevSecOps workflow. Whether you’re building CI/CD pipelines, managing infrastructure as code (IaC), or reviewing secure changes, Git ensures that every line of code is tracked, auditable, and versioned properly. This quick guide will guide you through Git basics with a DevSecOps perspective—so you can work securely, efficiently, and collaboratively.

----
## 🧩 1. What is Git?
Git is a distributed version control system (DVCS) that tracks changes in your code and allows teams to collaborate without risking data loss or overwriting work.
In DevSecOps, Git is the foundation for:
- Secure code review
- Pipeline automation
- Policy-as-Code
- Infrastructure-as-Code
- Auditable changes (compliance)
- Rollback and disaster recovery

----
## 🏗️ 2. Setting Up Git
### Install Git
#### Linux
```bash
sudo apt install git     # Ubuntu/Debian
sudo yum install git     # RHEL/CentOS/Rocky
```
#### Mac
```bash
brew install git
```
#### Windows
#### Check version
```bash
git --version
```
#### Configure user identity
```bash
git config --global user.name "Your Name" 
git config --global user.email "you@example.com"
```
----
## 📦 3. Creating or Cloning a Repository
### Initialize a Git Repository
Create a new repository for your project:
```bash
mkdir your_git_project 
cd your_git_project 
git init
```
### Clone an existing repository
```bash
git clone https://github.com/username/project.git
```
----
### ✍️ 4. The Git Workflow
![Alt text](./images/gitflow.png)

- **Branches:**
- master → Production-ready code. Each commit here represents a released version (tagged).

- develop → Integration branch containing code for the next release.

- Feature branches (feature/*) → Used to develop new features. Created from develop and merged back into develop.

- Release branches (release/*) → Created from develop to prepare a new version (testing, polishing). Merged into both master and develop.

- Hotfix branches (hotfix/*) → For urgent production fixes. Created from master, then merged back into both master and develop.