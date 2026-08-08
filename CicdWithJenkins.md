# 🔄 CI/CD Pipeline — Jenkins Auto Build & Deploy on Merge to `main`

![Jenkins](https://img.shields.io/badge/CI%2FCD-Jenkins-D24939?logo=jenkins&logoColor=white)
![GitLab](https://img.shields.io/badge/Source-GitLab-FC6D26?logo=gitlab&logoColor=white)
![Docker](https://img.shields.io/badge/Registry-Docker%20Hub-2496ED?logo=docker&logoColor=white)
![Ubuntu](https://img.shields.io/badge/OS-Ubuntu-E95420?logo=ubuntu&logoColor=white)
![Nginx](https://img.shields.io/badge/Server-Nginx-009639?logo=nginx&logoColor=white)
![Telegram](https://img.shields.io/badge/Notify-Telegram-26A5E4?logo=telegram&logoColor=white)

This document describes the **CI/CD pipeline** for **POS / posnews** — every time a change is pushed/merged to a branch (default `main`), a Jenkins job runs automatically: it checks out the latest code, builds the **frontend** and **backend** into Docker images, pushes them to **Docker Hub**, then SSHes into the production server, updates the image versions in `.env`, and runs `docker compose up` to deploy. Build status is sent to **Telegram**. No manual steps needed anymore.

| | |
|---|---|
| **Pipeline tool** | Jenkins (self-hosted on Ubuntu) |
| **Source control** | GitLab (`gitlab.com/ToekSopanha/pos`) |
| **Image registry** | Docker Hub (`mrlovermannn/posnews-frontend`, `posnews-backend`) |
| **Trigger** | GitLab webhook on push/merge to `main` |
| **Frontend** | Vue.js (built with Vite) → Docker image served by nginx |
| **Backend** | Node.js + Express container, exposed on a host port |
| **Production server** | Ubuntu server, path `/var/www/pos` |
| **Notifications** | Telegram bot (HTML messages) |

---

## 📑 Table of Contents

- [1. What Problem This Solves](#1-what-problem-this-solves)
- [2. Pipeline Overview](#2-pipeline-overview)
- [3. Pipeline Stages (Flow)](#3-pipeline-stages-flow)
- [4. Prerequisites](#4-prerequisites)
- [5. Step-by-Step Setup](#5-step-by-step-setup)
  - [5.1 Install Jenkins](#51-install-jenkins)
  - [5.2 Configure Jenkins Credentials](#52-configure-jenkins-credentials)
  - [5.3 Install Required Plugins](#53-install-required-plugins)
  - [5.4 Create the Pipeline Job](#54-create-the-pipeline-job)
  - [5.5 Jenkinsfile](#55-jenkinsfile)
  - [5.6 Configure the GitLab Webhook](#56-configure-the-gitlab-webhook)
  - [5.7 Configure the Production Server for SSH Deploys](#57-configure-the-production-server-for-ssh-deploys)
- [6. Docker Setup — Dockerfiles, nginx & Compose](#6-docker-setup--dockerfiles-nginx--compose)
  - [6.1 Backend Dockerfile](#61-backend-dockerfile)
  - [6.2 Frontend Dockerfile](#62-frontend-dockerfile)
  - [6.3 Frontend nginx config](#63-frontend-nginx-config)
  - [6.4 docker-compose.yml](#64-docker-composeyml)
- [7. Environment & Secrets](#7-environment--secrets)
- [8. Verifying the Pipeline](#8-verifying-the-pipeline)
- [9. Troubleshooting Guide](#9-troubleshooting-guide)
- [10. Security Best Practices](#10-security-best-practices)
- [11. Command Cheat Sheet](#11-command-cheat-sheet)

---

## 1. What Problem This Solves

Before CI/CD, deploying meant manually running a long chain of commands on the server every time you wanted changes live:

```bash
git pull
npm install
npm run build
docker compose up -d --build
sudo nginx -t && sudo systemctl reload nginx
```

That process was slow, error-prone, and depended on someone remembering all the steps. The CI/CD pipeline **automates all of it**:

- ✅ Push or merge to `main` → build & deployment start automatically
- ✅ Every build uses the exact same, reproducible steps
- ✅ Images are built once, tagged with the build number, and versioned in the registry
- ✅ Failed builds never reach production (pipeline aborts before deploy)
- ✅ Full audit trail of every build & deploy in Jenkins + Telegram notifications

---

## 2. Pipeline Overview

```
 ┌──────────────┐   push / merge   ┌──────────────┐     HTTP push      ┌──────────────┐
 │   GitLab     │ ───────────────► │   Webhook    │ ──────────────────►│   Jenkins    │
 │  pos repo    │   (main)         │  (Payload    │                    │  (Pipeline)  │
 │              │                  │   URL)       │                    │              │
 └──────────────┘                  └──────────────┘                    └──────┬───────┘
                                                                              │ checkout
                                                                              ▼
                                                     ┌──────────────┐  build + push   ┌──────────────┐
                                                     │   Docker     │ ◄─────────────── │   Stages     │
                                                     │   Hub        │    frontend:     │  frontend/   │
                                                     │              │    backend:vN    │  backend/    │
                                                     └──────┬───────┘                  └──────────────┘
                                                            │ docker compose pull
                                                            ▼
                                                     ┌──────────────┐
                                                     │  Production  │  ssh + sed .env + compose up
                                                     │  /var/www/pos│
                                                     └──────────────┘
```

**Key idea:** Jenkins is the "gatekeeper." Code only reaches production through the pipeline. Images are versioned (`v<build-number>`), so a rollback is just re-running a previous Jenkins build — it re-deploys the old tag.

---

## 3. Pipeline Stages (Flow)

The pipeline in the `Jenkinsfile` runs these stages in order:

| # | Stage | What happens | Fails the build if |
|---|-------|--------------|--------------------|
| 1 | **Notify Start** | Sends a Telegram "BUILD STARTED" message | Telegram token invalid / API down (non-blocking-ish) |
| 2 | **Checkout** | Clones the `pos` repo from GitLab at the selected branch | Git unreachable / bad `gitlab-jenkins` credentials |
| 3 | **Build Frontend** | `npm install` + `npm run build`, then `docker build` → `posnews-frontend:v<N>` | Any compile error, or Docker build failure |
| 4 | **Build Backend** | `docker build` → `posnews-backend:v<N>` | Docker build failure |
| 5 | **Push Images** | `docker login` with `docker_pass`, push both images to Docker Hub | Bad registry credentials / network |
| 6 | **Deploy** | SSH to server → `sed` the `FVERSION`/`BVERSION` in `.env` → `docker compose pull` → `docker compose up -d --force-recreate` | SSH failure, compose error, or image not found |

> ⚠️ This stage assumes **registry-based** compose (`image: ...:${FVERSION}`). Your actual `docker-compose.yml` in [6.4](#64-docker-composeyml) uses `build:` contexts — read that section and pick **one** deploy flow so the Jenkinsfile and compose match.

After the stages, the `post` block sends **BUILD SUCCESSFUL** or **BUILD FAILED** to Telegram with the build number, branch, and last commit message.

> 🔄 **Rollback note:** Images are tagged `v<build-number>`, so rolling back is trivial — rebuild a previous Jenkins run (or set `FVERSION`/`BVERSION` back to an older tag in the server's `.env`) and `docker compose pull && docker compose up -d --force-recreate`.

---

## 4. Prerequisites

| Software | Where | Purpose |
|---|---|---|
| **Jenkins (LTS)** | A server | Runs the pipeline |
| **Git + Node.js (v20.x)** | Jenkins server | Checkout + build the frontend |
| **Docker** | Jenkins server | Build the images |
| **SSH key pair** | Jenkins server → production server | Passwordless deploys |
| **Docker Hub account** | Registry | Store & pull the built images |
| **GitLab repo access** | Jenkins credentials (`gitlab-jenkins`) | Clone the repo |
| **Telegram bot + chat** | Jenkins credentials | Build notifications |
| **Docker + Docker Compose** | Production server | Run the app containers |
| **A reachable URL for Jenkins** | Public IP or domain | So GitLab can hit the webhook |

> This guide assumes Jenkins is installed on a server that can reach the production server over SSH. A single droplet running both is fine for this project's scale.

---

## 5. Step-by-Step Setup

### 5.1 Install Jenkins

On the Jenkins server (Ubuntu):

```bash
sudo apt update
sudo apt install -y openjdk-17-jre
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install -y jenkins
```

**Start & enable Jenkins:**

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

**Unlock Jenkins** — grab the initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Then open `http://JENKINS_SERVER_IP:8080`, paste the password, and complete the setup wizard (install **Suggested plugins**).

> 🔥 Open port `8080` in the server firewall for Jenkins, but later restrict it (see [9. Security](#9-security-best-practices)).

---

### 5.2 Configure Jenkins Credentials

The pipeline needs **four** credentials in Jenkins (**Manage Jenkins → Credentials → Global → Add Credentials**):

| Credential ID | Kind | Value | Used for |
|---|---|---|---|
| `gitlab-jenkins` | Username with password | GitLab username + Personal Access Token | Cloning the repo |
| `docker_pass` | Secret text | Docker Hub token/password | `docker login` before pushing |
| `telegram_token` | Secret text | Telegram bot API token | Sending notifications |
| `telegram_chatID` | Secret text | Your chat ID with the bot | Sending notifications |
| `prod-ssh-key` | SSH username with private key | Deploy SSH key (see [5.7](#57-configure-the-production-server-for-ssh-deploys)) | SSHing into production |

**GitLab PAT:** GitLab → **Settings → Access Tokens** → create one with `read_repository` scope. Use it as the password for the `gitlab-jenkins` credential.

**Telegram bot:** talk to [@BotFather](https://t.me/BotFather) → `/newbot` → copy the token. Then start a chat with your bot and message it once; get your chat ID (e.g. via `https://api.telegram.org/bot<TOKEN>/getUpdates`).

---

### 5.3 Install Required Plugins

Via **Manage Jenkins → Plugins → Available**:

- **Pipeline** (usually bundled)
- **Git** (usually bundled)
- **SSH Agent** or **Publish Over SSH** — for secure SSH deploys
- **GitLab** (GitLab integration / webhook auto-trigger)
- **Telegram Notification** *(optional)* — or use the `curl`-based `telegramSend` helper already in the Jenkinsfile (no plugin required)

---

### 5.4 Create the Pipeline Job

1. Jenkins → **New Item** → name it `pos-ci-cd`
2. Select **Pipeline**
3. Under **General**: check **GitLab Connection** / **GitLab project** → paste repo URL `https://gitlab.com/ToekSopanha/pos`
4. Under **Build Triggers**: check **GitLab webhook** (or **Build when a change is pushed to GitLab**) and enable `push` events on `main`
5. Under **Pipeline** → **Definition: Pipeline script from SCM**:

| Field | Value |
|---|---|
| SCM | Git |
| Repository URL | `https://gitlab.com/ToekSopanha/pos.git` |
| Credentials | `gitlab-jenkins` |
| Branches to build | `*/main` |
| Script Path | `Jenkinsfile` |

Now the `Jenkinsfile` living in the repo (next section) drives the entire pipeline. Because the pipeline has a `Git_Branch` parameter, you can also trigger **Build with Parameters** manually to build any branch.

---

### 5.5 Jenkinsfile

Create this file at the **root of the repo** (next to `frontend/` and `backend/`). This is the actual production pipeline — keep it in sync with the pipeline job's **Script Path** (`Jenkinsfile`).

```groovy
pipeline {
    agent any
    options {
        disableConcurrentBuilds()
    }
    environment {
        docker_pass     = credentials('docker_pass')  // Docker Hub password/token, Secret text
        telegram_token  = ""
        telegram_chatID = ""

        GIT_REPO       = "https://gitlab.com/ToekSopanha/pos.git"
        DOCKER_USER    = "mrlovermannn"                          // <-- confirm/update this
        FRONTEND_IMAGE = "${DOCKER_USER}/posnews-frontend"
        BACKEND_IMAGE  = "${DOCKER_USER}/posnews-backend"

        DEPLOY_HOST = ""                     // <-- confirm/update this
        DEPLOY_PATH = "/var/www/pos"                      // <-- confirm/update this
    }
    parameters {
        string(name: 'Git_Branch', defaultValue: 'main', description: 'Branch to build')
    }

    stages {
        stage('Notify Start') {
            steps {
                script {
                    telegramSend("""
🔨 <b>BUILD STARTED</b> 🔨
📦 <b>Job:</b> ${env.JOB_NAME}
🔗 <b>Repo:</b> ${env.GIT_REPO}
🔢 <b>Build:</b> #${env.BUILD_NUMBER}
🕒 <b>Started:</b> ${new Date().format('yyyy-MM-dd HH:mm:ss')}
🌿 <b>Branch:</b> ${params.Git_Branch}
🔗 <a href="${env.BUILD_URL}">View Console Output</a>
⏳ Sit tight, building now...
""")
                }
            }
        }

        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${params.Git_Branch}"]],
                    userRemoteConfigs: [[
                        url: env.GIT_REPO,
                        credentialsId: 'gitlab-jenkins'
                    ]]
                ])
                script {
                    env.COMMIT_MESSAGE = sh(script: "git log -1 --pretty=%B", returnStdout: true).trim()
                    env.GIT_REF        = sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
                }
            }
        }

        stage('Build Frontend') {
            steps {
                dir('frontend') {
                    sh '''
                        rm -rf node_modules package-lock.json
                        npm install
                        npm run build
                        docker build -t ${FRONTEND_IMAGE}:v${BUILD_NUMBER} .
                    '''
                }
            }
        }

        stage('Build Backend') {
            steps {
                dir('backend') {
                    sh 'docker build -t ${BACKEND_IMAGE}:v${BUILD_NUMBER} .'
                }
            }
        }

        stage('Push Images') {
            steps {
                sh '''
                    echo "${docker_pass}" | docker login -u ${DOCKER_USER} --password-stdin
                    docker push ${FRONTEND_IMAGE}:v${BUILD_NUMBER}
                    docker push ${BACKEND_IMAGE}:v${BUILD_NUMBER}
                    docker logout
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    ssh ${DEPLOY_HOST} "cd ${DEPLOY_PATH}; \
                        sed -i 's/^FVERSION=.*/FVERSION=v${BUILD_NUMBER}/' ${DEPLOY_PATH}/.env; \
                        sed -i 's/^BVERSION=.*/BVERSION=v${BUILD_NUMBER}/' ${DEPLOY_PATH}/.env; \
                        docker compose pull; \
                        docker compose up -d --force-recreate"
                """
            }
        }
    }

    post {
        success {
            script {
                telegramSend("""
✅ <b>BUILD SUCCESSFUL</b> ✅
📦 <b>Job:</b> ${env.JOB_NAME}
🔢 <b>Build:</b> #${env.BUILD_NUMBER}
🕒 <b>Finished:</b> ${new Date().format('yyyy-MM-dd HH:mm:ss')}
🌿 <b>Branch:</b> ${env.GIT_REF ?: params.Git_Branch}
💬 <b>Last commit:</b> ${env.COMMIT_MESSAGE ?: 'n/a'}
🔗 <a href="${env.BUILD_URL}">View Console Output</a>
🎉 Great job, ship it! 🚀
""")
            }
        }
        failure {
            script {
                telegramSend("""
❌ <b>BUILD FAILED</b> ❌
📦 <b>Job:</b> ${env.JOB_NAME}
🔢 <b>Build:</b> #${env.BUILD_NUMBER}
🕒 <b>Finished:</b> ${new Date().format('yyyy-MM-dd HH:mm:ss')}
🌿 <b>Branch:</b> ${env.GIT_REF ?: params.Git_Branch}
💬 <b>Last commit:</b> ${env.COMMIT_MESSAGE ?: 'n/a'}
🔗 <a href="${env.BUILD_URL}console">View Console Output</a>
🔥 Something broke — check the logs!
""")
            }
        }
    }
}

// Sends an HTML-formatted Telegram message using bot token / chat id
// stored as Jenkins Secret text credentials (telegram_token, telegram_chat_id).
def telegramSend(String message) {
    writeFile file: 'tg_message.txt', text: message
    sh '''
        curl -s -X POST "https://api.telegram.org/bot${telegram_token}/sendMessage" \
            -d chat_id="${telegram_chatID}" \
            -d parse_mode="HTML" \
            --data-urlencode text@tg_message.txt > /dev/null
    '''
    sh 'rm -f tg_message.txt'
}
```

**Deployment strategy — registry + version pinning, not building on the server:**

1. Images are built **on the Jenkins server** and tagged `v<BUILD_NUMBER>` (e.g. `mrlovermannn/posnews-frontend:v42`).
2. They're pushed to **Docker Hub** in the *Push Images* stage.
3. The *Deploy* stage SSHes to production and uses `sed` to rewrite two variables in `/var/www/pos/.env`:

   ```bash
   FVERSION=v42   # frontend image tag
   BVERSION=v42   # backend image tag
   ```

4. `docker compose pull` fetches the exact tagged images, then `docker compose up -d --force-recreate` swaps the containers.

> 📌 For this to work, `docker-compose.yml` on the server must reference those variables, e.g.:
>
> ```yaml
> services:
>   frontend:
>     image: mrlovermannn/posnews-frontend:${FVERSION}
>   backend:
>     image: mrlovermannn/posnews-backend:${BVERSION}
> ```
>
> ⚠️ **Before you run this, fill in the blanks** in the Jenkinsfile:
> - `DOCKER_USER` → confirm it's `mrlovermannn`
> - `DEPLOY_HOST` → your server's user@host (currently empty)
> - `telegram_token` / `telegram_chatID` → wire these to the `telegram_token` / `telegram_chatID` **Secret text** credentials (currently empty strings — the helper function expects them)

---

### 5.6 Configure the GitLab Webhook

1. GitLab repo → **Settings → Webhooks** → **Add webhook**:

| Field | Value |
|---|---|
| URL | `http://JENKINS_SERVER_IP:8080/project/pos-ci-cd` |
| Trigger | **Push events** (fires on merge to `main` too) |
| Secret token | *(optional)* the Jenkins GitLab plugin token |

2. Click **Add webhook**, then **Test → Push events** and wait for a `200` response.

Now every **push/merge to `main`** sends a POST to Jenkins, which starts the pipeline automatically. Other branches are ignored by the SCM branch filter (`*/main`), but you can still build them manually via **Build with Parameters → Git_Branch**.

---

### 5.7 Configure the Production Server for SSH Deploys

Jenkins SSHes into production to deploy — set this up so it's passwordless and non-interactive. The Jenkinsfile's `Deploy` stage runs `ssh ${DEPLOY_HOST} "cd ${DEPLOY_PATH}; ..."`, so the Jenkins user must be able to `ssh` to `DEPLOY_HOST` and run `docker compose` there without a password.

1. **Generate a dedicated deploy key** on the Jenkins server:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/jenkins_deploy -C "jenkins-ci"
```

2. **Authorize it on production** (use your real user/host — this is what you'll put in `DEPLOY_HOST`):

```bash
ssh-copy-id -i ~/.ssh/jenkins_deploy.pub root@YOUR_SERVER_IP
```

3. **Make sure that user can run docker compose on the server:**

```bash
# as the deploy user on production
sudo usermod -aG docker root          # or your deploy user
docker compose version
```

4. Test the connection manually from the Jenkins server (with the key in `~/.ssh/`):

```bash
ssh -o StrictHostKeyChecking=no root@YOUR_SERVER_IP "echo connected"
```

5. **Add the private key to Jenkins** (so builds are reproducible even on a fresh agent):

- **Manage Jenkins → Credentials → Global → Add Credentials**
- Kind: **SSH Username with private key**, ID: `prod-ssh-key`
- If the Jenkins server agent already has the key in its `~/.ssh/`, the pipeline works as-is — the `Deploy` stage uses plain `ssh`, so `StrictHostKeyChecking=no` (or a `known_hosts` entry) is recommended to avoid prompts.

---

## 6. Docker Setup — Dockerfiles, nginx & Compose

The pipeline builds the two Docker images from the `frontend/` and `backend/` folders of the repo. These are the actual production Dockerfiles, the frontend nginx config, and how they're wired together with `docker-compose.yml` on the server.

### 6.1 Backend Dockerfile — `backend/Dockerfile`

A Node.js 20 runtime image that installs production deps only, pre-creates the `data/` and `uploads/` volumes so they mount cleanly, and runs `server.js` on port `3001`.

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine

WORKDIR /app

# Install production dependencies
COPY package*.json ./
RUN npm install --production && npm cache clean --force

# Copy the rest of the backend source
COPY . .

# Pre-create the data and uploads directories so Docker volumes mount cleanly
# and the runtime has writable targets from the first request.
RUN mkdir -p /app/data /app/uploads

ENV NODE_ENV=production
ENV PORT=3001

EXPOSE 3001

CMD ["node", "server.js"]
```

### 6.2 Frontend Dockerfile — `frontend/Dockerfile`

A multi-stage build: the first stage compiles the Vue app with Vite into static files; the second stage (nginx) ships those files, installs Certbot for Let's Encrypt, and hands control to `entrypoint.sh` (certificate bootstrap + renewal).

```dockerfile
# syntax=docker/dockerfile:1

# ---------- Build stage ----------
FROM node:20-alpine AS build

WORKDIR /app

# Install dependencies (including dev deps needed by Vite)
COPY package*.json ./
RUN npm install && npm cache clean --force

# Build the Vue app
COPY . .
RUN npm run build

# ---------- Runtime stage ----------
FROM nginx:alpine

# Install Certbot and the nginx plugin for Let's Encrypt
RUN apk add --no-cache certbot certbot-nginx

# Replace the default nginx site with our SPA + reverse-proxy config.
# Certbot's nginx plugin will append the 443 server block once a
# certificate is issued, so this file only needs to be valid on port 80.
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy built static assets from the build stage
COPY --from=build /app/dist /usr/share/nginx/html

# Certbot state and issued certificates live here
RUN mkdir -p /etc/letsencrypt

# Entrypoint script handles certificate bootstrap + renewals
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

EXPOSE 80
EXPOSE 443

ENTRYPOINT ["/entrypoint.sh"]
```

### 6.3 Frontend nginx config — `frontend/nginx.conf`

Serves the SPA, gzips static assets, and reverse-proxies `/api/` and `/uploads/` to the backend service (reachable inside the Docker network as `backend:3001`).

```nginx
server {
    listen 80;
    server_name sopanha.digital;

    root /usr/share/nginx/html;
    index index.html;

    # Gzip compression for static assets
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript image/svg+xml;
    gzip_min_length 256;

    # Proxy API requests to the backend service
    location /api/ {
        proxy_pass http://backend:3001;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
    }

    # Proxy uploaded files (profile photos, project thumbnails, CV) to backend
    location /uploads/ {
        proxy_pass http://backend:3001;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Long-cache hashed Vite assets
    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }

    # SPA fallback — every unknown path returns index.html so Vue Router handles it
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

> 💡 `entrypoint.sh` (referenced by the frontend Dockerfile) handles the Certbot bootstrap/renewal. Keep it in `frontend/entrypoint.sh` so it's copied during the Docker build.

### 6.4 docker-compose.yml — `docker-compose.yml` (on the server, `/var/www/pos`)

The actual production compose file. It builds both images from the checked-out source (local `build:` contexts), mounts the backend's `data/`/`uploads/` directories and Certbot state as named volumes, and exposes the frontend on 80/443 plus the backend on 3001:

```yaml
services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: portfolio-backend
    restart: unless-stopped
    environment:
      DATA_DIR: /app/data
      PORT: 3001
      # Adjust if you need to allow additional origins (comma-separated).
      CORS_ORIGIN: http://localhost
      # JWT_SECRET should be set in production (e.g. via a .env file).
      # The fallback below is intentionally weak so the container still boots
      # locally without configuration, but it MUST be overridden for any
      # non-dev deployment.
      JWT_SECRET: ${JWT_SECRET:-change_me_in_production}
    volumes:
      - portfolio-data:/app/data
      - backend-uploads:/app/uploads
    ports:
      - "3001:3001"

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: portfolio-frontend
    restart: unless-stopped
    depends_on:
      - backend
    ports:
      - "80:80"
      - "443:443"
    environment:
      - DOMAIN=${DOMAIN}
      - EMAIL=${EMAIL}
    volumes:
      - letsencrypt:/etc/letsencrypt

volumes:
  portfolio-data:
  backend-uploads:
  letsencrypt:
```

Supported env vars for `/var/www/pos/.env`:

| Variable | Example | Used by | Required |
|---|---|---|---|
| `DOMAIN` | `sopanha.digital` | `frontend` (Certbot) | ✅ |
| `EMAIL` | `you@example.com` | `frontend` (Certbot renewal alerts) | ✅ |
| `JWT_SECRET` | a long random string | `backend` (token signing) | ✅ for production |
| `FVERSION` / `BVERSION` | `v1` | **only if** you switch to registry-based deploys | only for registry flow |

> ⚠️ **Important — this compose file uses `build:`, but the Jenkinsfile Deploy stage runs `docker compose pull`.** Those two only work together if you change one of them:
>
> - **Option A — build on the server (matches this compose):** the Deploy stage should ship the code to the server and run `docker compose up -d --build`, instead of `docker compose pull`. No Docker Hub needed; the server must have the source (and Node.js inside the Docker build is enough).
> - **Option B — registry flow (matches the current Jenkinsfile):** change the compose services to `image: mrlovermannn/posnews-frontend:${FVERSION}` / `...:${BVERSION}` (no `build:`), and Jenkins keeps pushing tags to Docker Hub then `docker compose pull`. The server never needs the source.
>
> Pick one and keep the Jenkinsfile and compose consistent — a mix (build context + `docker compose pull`) will fail with `pull access denied for ...`. See the Deploy-stage alternatives in section [5.5](#55-jenkinsfile).

---

## 7. Environment & Secrets

| Secret | Where it lives | Used for |
|---|---|---|
| GitLab PAT | Jenkins credential `gitlab-jenkins` | Cloning the repo |
| Docker Hub token | Jenkins credential `docker_pass` | `docker login` + push images |
| Telegram bot token | Jenkins credential `telegram_token` | Build notifications |
| Telegram chat ID | Jenkins credential `telegram_chatID` | Where to send notifications |
| SSH key | `~/.ssh/` on Jenkins (or credential `prod-ssh-key`) | SSHing into production |
| `DEPLOY_HOST` / `DEPLOY_PATH` | Jenkinsfile `environment` block | Deploy target (currently `""` / `/var/www/pos` — update it!) |
| `FVERSION` / `BVERSION` | `/var/www/pos/.env` on production | Which image tags compose deploys |

> ⚠️ Never commit secrets to the repo. `docker_pass`, `telegram_token`, and `telegram_chatID` are read from Jenkins Secret-text credentials — the pipeline never prints them (the `curl` helper sends to `/dev/null`). Note: in the current Jenkinsfile the `telegram_token` / `telegram_chatID` vars are defined as empty strings; wire them to the credentials object instead, e.g.:
>
> ```groovy
> telegram_token  = credentials('telegram_token')
> telegram_chatID = credentials('telegram_chatID')
> ```

---

## 8. Verifying the Pipeline

After the first successful build, walk through this checklist:

- [ ] Jenkins job `pos-ci-cd` shows **`#1 SUCCESS`** in blue
- [ ] Stage view shows all 6 stages passing (Notify → Checkout → Build F/E → Build B/E → Push → Deploy)
- [ ] Both images appear on Docker Hub: `mrlovermannn/posnews-frontend:v1` and `mrlovermannn/posnews-backend:v1`
- [ ] Telegram received **BUILD STARTED** then **BUILD SUCCESSFUL** messages
- [ ] On production: `cat /var/www/pos/.env` shows `FVERSION=v1` and `BVERSION=v1`
- [ ] `docker compose ps` on production shows frontend + backend containers `Up`
- [ ] Push another commit to `main` and confirm a **new build triggers automatically** (no manual click)
- [ ] GitLab webhook page shows the last delivery with `200 OK`

---

## 9. Troubleshooting Guide

| Symptom | Likely Cause | Resolution |
|---|---|---|
| Pipeline doesn't trigger on push | Webhook misconfigured, or Jenkins port unreachable | Check GitLab → Webhooks → **Recent Deliveries** for `200` responses; confirm `8080` is open |
| `Checkout` fails | Repo is private, bad PAT / scope | Verify `gitlab-jenkins` credential; token needs `read_repository` |
| `docker login` fails (`denied`) | Wrong Docker Hub user / token | Verify `docker_pass` and that `DOCKER_USER` matches the Hub account owner of the images |
| `push access to repository denied` | Image namespace ≠ Hub account | `FRONTEND_IMAGE`/`BACKEND_IMAGE` must be `mrlovermannn/...` (or an org you're member of) |
| `Deploy` fails with `Permission denied (publickey)` | SSH key not authorized on production | Re-run `ssh-copy-id`; confirm `DEPLOY_HOST` is set correctly |
| `sed` didn't change versions | `.env` lacks `FVERSION`/`BVERSION` lines | Add them first: `echo 'FVERSION=v0' >> /var/www/pos/.env`, same for `BVERSION` |
| `manifest for ...v1 not found` | Tag not pushed / typo in compose `.env` | Verify the tag exists on Docker Hub; re-run push stage |
| `docker compose pull` pulls old image | Compose references `latest` or a cached tag | Ensure compose uses `image: ...:${FVERSION}` / `:${BVERSION}` |
| `docker compose pull` → `pull access denied` | Compose has `build:` contexts but the Deploy stage does `docker compose pull` | Pick one flow (build-on-server vs registry) as explained in [6.4](#64-docker-composeyml) and make Jenkinsfile + compose consistent |
| `DOMAIN`/`EMAIL` empty in frontend | `.env` missing those vars for Certbot | Add `DOMAIN=sopanha.digital` and `EMAIL=you@example.com` to `/var/www/pos/.env` |
| Backend `PORT`/`CORS_ORIGIN` ignored | Overridden via compose `environment:` | `docker compose up -d --force-recreate` after changing env; check `docker compose config` |
| Telegram messages missing | Empty `telegram_token` / `telegram_chatID` | Wire them to the Secret-text credentials as shown in [7](#7-environment--secrets) |
| Container keeps restarting (`Restarting`) | App crashes on startup, or missing runtime env | `docker compose logs --tail=50 backend` on production |
| Build triggers on other branches | Branch filter not set | Ensure **Branches to build** is `*/main` (and the webhook only pushes `main`) |

---

## 10. Security Best Practices

- 🔒 **Restrict Jenkins access** — put it behind a VPN/firewall rule that only allows your IP on port `8080`. Do not expose it to the whole internet.
- 🔑 **Use the deploy key**, never Jenkins' own password, to reach production.
- 🧹 **Rotate credentials** — regenerate the GitLab PAT, Docker Hub token, and SSH key periodically (add to your calendar).
- 🛡️ **Keep `docker_pass` as a Secret-text credential** — never hardcode it in the repo or the pipeline output. The `docker login` step reads it via `credentials('docker_pass')`.
- 📁 **Keep secrets server-side** — `/var/www/pos/.env` (DB passwords, API keys, etc.) stays on the server; the pipeline only rewrites `FVERSION`/`BVERSION` in it.
- 🎯 **Least privilege** — limit the SSH user's permissions to `/var/www/pos` and docker compose if possible (a dedicated `deploy` user instead of `root` is safer).
- 🛑 **Never merge to `main` directly on a broken build** — require the Jenkins build to be green before merging (use GitLab branch protection + pipeline status as a required check).

---

## 11. Command Cheat Sheet

```bash
# Jenkins service
sudo systemctl status jenkins
sudo systemctl restart jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Jenkins pipeline build log
sudo tail -f /var/lib/jenkins/jobs/pos-ci-cd/builds/latest/log

# Manual trigger (build any branch)
curl -X POST http://JENKINS_IP:8080/job/pos-ci-cd/buildWithParameters \
  -u user:token --data Git_Branch=feature/xyz

# Test SSH from Jenkins to production
ssh root@YOUR_SERVER_IP "docker compose -f /var/www/pos/docker-compose.yml ps"

# On production — deploy a specific version manually
cd /var/www/pos
sed -i 's/^FVERSION=.*/FVERSION=v42/' .env
sed -i 's/^BVERSION=.*/BVERSION=v42/' .env
docker compose pull
docker compose up -d --force-recreate

# Production checks after deploy
docker compose ps
docker compose logs --tail=50 backend
docker images | grep posnews
cat /var/www/pos/.env
```

---

<p align="center">
  <sub>Maintained by Toek Sopanha · Software Engineer | Network Engineer | Cloud/DevOps Engineer</sub>
</p>
