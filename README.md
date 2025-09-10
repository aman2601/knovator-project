# Project Web-1

This project demonstrates a **Dockerized Node.js (backend) and ReactJS (frontend) application** running behind an **Nginx reverse proxy** using `docker-compose`.  
It was built as part of a DevOps technical task.

---

## 🚀 Features
- **Backend**: Node.js + Express API (runs on port `5000`)
- **Frontend**: ReactJS (served via Nginx, runs on port `3000` inside container)
- **Reverse Proxy**: Nginx routes traffic:
  - `/api` → Node.js backend
  - `/` → React frontend
- **Multi-stage Docker builds** for lightweight images
- **Docker Compose** to orchestrate services

---

## 📂 Project Structure
        Project-Web-1/
        │── backend/ # Node.js API service
        │ ├── Dockerfile
        │ ├── package.json
        │ └── server.js
        │
        │── frontend/ # ReactJS frontend
        │ ├── Dockerfile
        │ ├── package.json
        │ ├── public/index.html
        │ └── src/
        │ ├── App.js
        │ └── index.js
        │
        │── nginx/ # Reverse proxy config
        │ └── nginx.conf
        │
        │── docker-compose.yml
        │── README.md



---

## ⚙️ Prerequisites
- Docker installed
- Docker Compose installed
- Git (for cloning the repo)
Build YOUR EC2 ubuntu with will script 

```
    #!/bin/bash
    # Update & upgrade
    apt update && apt upgrade -y
    
    # Install dependencies
    apt install -y apt-transport-https ca-certificates curl software-properties-common
    
    # Install Docker
    apt install -y docker.io
    
    # Install docker-compose (v1.29.2)
    curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" \
      -o /usr/local/bin/docker-compose
    chmod +x /usr/local/bin/docker-compose
    
    # Add default 'ubuntu' user to docker group
    usermod -aG docker ubuntu
    
    # Enable and start Docker
    systemctl enable docker
    systemctl start docker
```
---

## ▶️ How to Run

1. Clone the repo:
   ```bash
   git clone https://github.com/<your-username>/project-web-1.git
   cd project-web-1



docker-compose build
docker-compose up -d



Access the app:

Frontend (React): http://localhost

Backend (Node.js): http://localhost/api



--------------------------------------------------------

Create Custom Project Runner 


This document explains how we set up a project-scoped, self-hosted GitLab Runner on an Ubuntu EC2 instance to build & deploy Dockerized apps with docker-compose. It also records the issues we faced and their fixes.

Architecture (1-minute overview)

 * Repo: GitHub (mirrored/imported into GitLab).
 * CI/CD: GitLab Pipelines.
 * Runner: Self-hosted on our EC2 (Shell executor).
 * Images: Built on EC2, pushed to GitLab Container Registry.
 * Deploy: docker-compose on the same EC2.

 Prerequisites

  * Ubuntu 20.04/24.04 EC2 with:
```
sudo apt update && sudo apt install -y docker.io curl git
sudo systemctl enable --now docker
```

  * GitLab project created (imported from GitHub or native).

  * Your public IP/node security group allows SSH (22) and app ports (e.g., 80/443).

1) Install GitLab Runner (on EC2)

```
curl -L --output /usr/local/bin/gitlab-runner \
  https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64
sudo chmod +x /usr/local/bin/gitlab-runner

sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash || true
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo systemctl start gitlab-runner
```


2) Create a Project Runner in GitLab & get the Registration Token

In GitLab → Project → Settings → CI/CD → Runners → Expand → Create project runner
Add tags (we used ec2), then Create to see the registration token.


3) Register the Runner (Shell executor)

```
sudo gitlab-runner register
# GitLab URL: https://gitlab.com/
# Registration token: <PROJECT_RUNNER_TOKEN>
# Description: ec2-runner
# Tags: ec2
# Executor: shell
```

This creates /etc/gitlab-runner/config.toml with a [[runners]] section.

4) Run the Runner as root (simplifies Docker access)

```
# Create/modify systemd override to run as root
sudo systemctl edit gitlab-runner
# Add exactly:
# [Service]
# User=root
# (save & exit)

sudo systemctl daemon-reexec
sudo systemctl restart gitlab-runner
ps -o user,pid,cmd -C gitlab-runner   # should show 'root'
```

Alternative (more secure): keep service user gitlab-runner but add it to the docker group:
```
sudo usermod -aG docker gitlab-runner && sudo systemctl restart docker gitlab-runner
```

5) Avoid Shell profile noise (critical for Shell executor)

Some Ubuntu images load scripts that echo/printf during non-interactive shells, which breaks “prepare environment”.

Tell runner to use a clean bash:

Edit /etc/gitlab-runner/config.toml and in your [[runners]] block add:
```
executor = "shell"
shell = "bash"
environment = ["BASH_ENV="]   # don't load bash rc files
```
Guard noisy profile scripts:

Add this as first line to these files (if present):
```
# At the top of each file:
[ -n "$CI" ] && return 0
```

Common offenders:

/etc/bash.bashrc

/etc/profile

/etc/profile.d/Z99-cloud-locale-test.sh

/etc/profile.d/Z99-cloudinit-warnings.sh

(Or rename the offending /etc/profile.d/*.sh to *.disabled.)

Restart runner:
```
sudo systemctl restart gitlab-runner
```


Common Problems & Fixes (What we faced)
1) Runner shows Offline (gray)

Cause: Not registered or wrong token/URL.

Fix: Re-register with Project Runner token and URL https://gitlab.com/, then restart runner.

2) “This job is stuck… no runners matching tags”

Cause: Job tags don’t match runner tags.

Fix: Set default: { tags: [ec2] } in .gitlab-ci.yml, or edit runner’s tags to include ec2.

3) Job failed: prepare environment: exit status 1

Cause: Shell executor loads profile files that print to STDOUT/ERR (cloud-init/locale scripts).

Fix:

In config.toml: shell="bash", environment=["BASH_ENV="]

Guard files with:
[ -n "$CI" ] && return 0
in /etc/bash.bashrc, /etc/profile, /etc/profile.d/*.sh

Restart runner.

4) Docker permission denied

Cause: Runner user not allowed to talk to Docker daemon.

Fix (either):

Run service as root (systemd override).

Or add gitlab-runner to docker group:
sudo usermod -aG docker gitlab-runner && sudo systemctl restart docker gitlab-runner

5) Build succeeds but deploy uses old images

Cause: docker-compose is building locally or not pulling.

Fix: Use image: refs with tags ($CI_COMMIT_SHORT_SHA) and run docker-compose pull before up.