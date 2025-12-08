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
sudo apt install git     # Ubuntu/Debian \
sudo yum install git     # RHEL/CentOS/Rocky\
#### Mac
brew install git
#### Windows
#### Check version
git --version
#### Configure user identity
git config --global user.name "Your Name" \
git config --global user.email "you@example.com"

----
## 📦 3. Creating or Cloning a Repository
### Create a new repository
git init
### Clone an existing repository
git clone https://github.com/username/project.git


