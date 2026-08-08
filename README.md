# 🚀 MyPortfolio — Build & Deploy Guide

![Vue](https://img.shields.io/badge/Frontend-Vue.js-42b883?logo=vue.js&logoColor=white)
![Node](https://img.shields.io/badge/Backend-Node.js-339933?logo=node.js&logoColor=white)
![Docker](https://img.shields.io/badge/Deploy-Docker-2496ED?logo=docker&logoColor=white)
![Nginx](https://img.shields.io/badge/Server-Nginx-009639?logo=nginx&logoColor=white)
![Let's Encrypt](https://img.shields.io/badge/SSL-Let's%20Encrypt-003A70?logo=letsencrypt&logoColor=white)

This guide covers the complete process of building Docker images for MyPortfolio, pushing them to Docker Hub, and deploying them to a server with automatic HTTPS via Let's Encrypt.

---

## 📑 Table of Contents

- [Project Overview](#project-overview)
- [Prerequisites](#prerequisites)
- [Local Development](#local-development)
- [Configure Environment](#configure-environment)
- [Build Docker Images](#build-docker-images)
- [Push to Docker Hub](#push-to-docker-hub)
- [Prepare the Server](#prepare-the-server)
- [Deploy with Docker Compose](#deploy-with-docker-compose)
- [Let's Encrypt SSL](#lets-encrypt-ssl)
- [Verify Deployment](#verify-deployment)
- [Admin Panel](#admin-panel)
- [Redeploy After Updates](#redeploy-after-updates)
- [Hosting Multiple Websites on One Server](#hosting-multiple-websites-on-one-server)
- [Troubleshooting](#troubleshooting)
- [Useful Commands](#useful-commands)

---

## Project Overview

MyPortfolio is a full-stack personal portfolio website:

- **Frontend**: Vue.js 3 + Vite (served by nginx)
- **Backend**: Node.js + Express API
- **Storage**: JSON files + uploaded images (stored in Docker volumes)
- **Auth**: JWT-protected admin panel
- **HTTPS**: Automatic Let's Encrypt certificates inside the frontend container

```
                         ┌─────────────────────────────┐
                         │           Browser            │
                         └───────────────┬───────────────┘
                                         │ HTTP / HTTPS
                                         ▼
                         ┌─────────────────────────────┐
                         │   nginx in Docker container  │
                         │  myportfolio.sopanha.digital │
                         │      (port 80 / 443)         │
                         └───────────────┬───────────────┘
                                         │
                  ┌──────────────────────┼──────────────────────┐
                  │                                              │
                  ▼                                              ▼
   ┌───────────────────────────┐                ┌───────────────────────────┐
   │   Static files (Vue dist) │                │   /api/*  → reverse proxy │
   │   /usr/share/nginx/html   │                │   http://backend:3001     │
   └───────────────────────────┘                └─────────────┬─────────────┘
                                                                │
                                                                ▼
                                                   ┌───────────────────────────┐
                                                   │   Node.js / Express API   │
                                                   │   (Docker container)      │
                                                   │      backend:3001         │
                                                   └───────────────────────────┘
```

---

## Prerequisites

### Local machine

| Tool | Purpose |
|---|---|
| Docker Desktop or Docker Engine | Build images |
| Docker Hub account | Store images |
| Git (optional) | Version control |

### Server

| Tool | Purpose |
|---|---|
| Ubuntu 22.04+ (recommended) | Host OS |
| Docker + Docker Compose | Run containers |
| Domain name | Point to server IP |
| Ports 22, 80, 443 open | SSH, HTTP, HTTPS |

---

## Local Development

```bash
# Backend
npm install
npm run dev
```

```bash
# Frontend
npm install
npm run dev
```

Backend runs on `http://localhost:3001`.  
Frontend runs on `http://localhost:5173`.

Admin panel: `http://localhost:5173/admin`  
Default password: ``

---

## Configure Environment

Create a `.env` file at the project root:

```bash
JWT_SECRET=your_very_strong_random_string
DOMAIN=myportfolio.sopanha.digital
EMAIL=your-email@example.com
```

| Variable | Required | Description |
|---|---|---|
| `JWT_SECRET` | Yes | Strong random string for JWT signing. Change in production. |
| `DOMAIN` | Yes | Your domain name. Used by Let's Encrypt and nginx. |
| `EMAIL` | Yes | Your email for Let's Encrypt notifications. |

> ⚠️ **Never commit `.env` to Git.** `.env.example` is provided as a template.

Set your domain in the frontend nginx config:

```bash
# frontend/nginx.conf
server_name myportfolio.sopanha.digital;
```

---

## Build Docker Images

Open a terminal in the project root.

### Build backend image

```bash
cd backend
docker build -t yourdockerhubusername/portfolio-backend:latest .
cd ..
```

### Build frontend image

```bash
cd frontend
docker build -t yourdockerhubusername/portfolio-frontend:latest .
cd ..
```

Replace `yourdockerhubusername` with your actual Docker Hub username.

### Verify images exist

```bash
docker images
```

You should see:
- `yourdockerhubusername/portfolio-backend`
- `yourdockerhubusername/portfolio-frontend`

---

## Push to Docker Hub

Login to Docker Hub:

```bash
docker login
```

Push both images:

```bash
docker push yourdockerhubusername/portfolio-backend:latest
docker push yourdockerhubusername/portfolio-frontend:latest
```

---

## Prepare the Server

### 1. Connect to the server

```bash
ssh root@your_server_ip
```

### 2. Update system

```bash
sudo apt update && sudo apt upgrade -y
```

### 3. Install Docker

```bash
sudo apt install -y docker.io docker-compose-plugin
sudo systemctl enable docker
sudo systemctl start docker
```

Add your user to the docker group (optional):

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### 4. Open firewall ports

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

> If you use a cloud provider (DigitalOcean, AWS, Azure, etc.), also open ports 80 and 443 in their dashboard firewall/security group.

---

## Deploy with Docker Compose

### 1. Create deployment directory

```bash
sudo mkdir -p /var/www/myportfolio
cd /var/www/myportfolio
```

### 2. Create `docker-compose.yml`

```yaml
services:
  backend:
    image: yourdockerhubusername/portfolio-backend:latest
    container_name: portfolio-backend
    restart: unless-stopped
    environment:
      DATA_DIR: /app/data
      PORT: 3001
      CORS_ORIGIN: https://myportfolio.sopanha.digital
      JWT_SECRET: ${JWT_SECRET}
    volumes:
      - portfolio-data:/app/data
      - backend-uploads:/app/uploads
    ports:
      - "3001:3001"

  frontend:
    image: yourdockerhubusername/portfolio-frontend:latest
    container_name: portfolio-frontend
    restart: unless-stopped
    depends_on:
      - backend
    ports:
      - "80:80"
      - "443:443"
    environment:
      DOMAIN: ${DOMAIN}
      EMAIL: ${EMAIL}
    volumes:
      - letsencrypt:/etc/letsencrypt

volumes:
  portfolio-data:
  backend-uploads:
  letsencrypt:
```

### 3. Create `.env`

```bash
nano .env
```

```bash
JWT_SECRET=your_very_strong_random_string
DOMAIN=myportfolio.sopanha.digital
EMAIL=your-email@example.com
```

### 4. Point domain DNS

In your domain registrar or DNS provider, create an A record:

| Type | Name | Value |
|---|---|---|
| A | `myportfolio` or `@` | `your_server_ip` |

Wait for DNS propagation. Verify with:

```bash
nslookup myportfolio.sopanha.digital
```

### 5. Pull and run

```bash
docker compose pull
docker compose up -d
```

### 6. Check logs

```bash
docker compose logs -f frontend
```

On first run, the frontend container will request a Let's Encrypt SSL certificate. You should see success messages in the logs.

---

## Let's Encrypt SSL

This project uses **Let's Encrypt** to provide free, automatic SSL certificates for HTTPS.

### How it works

1. **Certbot inside the frontend container**
   - The frontend Docker image installs `certbot` and the `certbot-nginx` plugin.
   - When the container starts, the `frontend/entrypoint.sh` script runs.

2. **First certificate request**
   - The script checks if a certificate already exists at `/etc/letsencrypt/live/$DOMAIN/`.
   - If not, it runs:
     ```bash
     certbot --nginx -d $DOMAIN --non-interactive --agree-tos --email $EMAIL --no-eff-email
     ```
   - Certbot proves domain ownership using the **HTTP-01 challenge**: it temporarily serves a verification file on port 80.
   - If successful, Let's Encrypt issues the certificate.

3. **Nginx configuration**
   - The `certbot-nginx` plugin automatically modifies `nginx.conf` to add the HTTPS server block on port 443.
   - It also redirects HTTP traffic on port 80 to HTTPS.

4. **Certificate storage**
   - Certificates are stored in `/etc/letsencrypt/`.
   - This directory is mounted to the Docker volume `letsencrypt` so certificates survive container restarts.

5. **Automatic renewal**
   - After nginx starts, the entrypoint script runs `certbot renew --quiet --nginx` every 12 hours.
   - Let's Encrypt certificates are valid for 90 days. Certbot renews them automatically before expiry.

### Requirements for Let's Encrypt

| Requirement | Why |
|---|---|
| Domain must resolve to server IP | Let's Encrypt needs to reach your server |
| Port 80 must be open | HTTP-01 challenge uses port 80 |
| Port 443 must be open | HTTPS traffic uses port 443 |
| Valid `DOMAIN` env var | Certbot requests a cert for this domain |
| Valid `EMAIL` env var | Let's Encrypt sends expiry notices here |

### View certificate details

Inside the running container:

```bash
docker exec portfolio-frontend certbot certificates
```

Output example:
```
Found the following certs:
  Certificate Name: myportfolio.sopanha.digital
    Domains: myportfolio.sopanha.digital
    Expiry Date: 2026-10-05 (VALID: 89 days)
    Certificate Path: /etc/letsencrypt/live/myportfolio.sopanha.digital/fullchain.pem
    Private Key Path: /etc/letsencrypt/live/myportfolio.sopanha.digital/privkey.pem
```

### Force certificate renewal

```bash
docker exec portfolio-frontend certbot renew --force-renewal --nginx
```

### Reset and reissue certificate

If you change domains or need a fresh certificate:

```bash
cd /var/www/myportfolio
docker compose down
docker volume rm myportfolio_letsencrypt
docker compose up -d
```

The container will request a new certificate on startup.

### Common certificate errors

| Error | Cause | Fix |
|---|---|---|
| `Connection refused` on port 80 | Firewall blocking | Open port 80 on server and cloud provider |
| `Could not resolve host` | DNS not propagated | Wait for DNS, verify with `nslookup` |
| `Unauthorized` | Domain points to wrong IP | Check A record |
| `Too many failed authorizations` | Rate limit hit | Wait 1 hour and try again |
| Cert not renewed automatically | Container stopped too long | Start container and run `certbot renew` manually |

### Rate limits

Let's Encrypt has rate limits:
- **50 certificates per registered domain per week**
- **5 duplicate certificates per week** for the exact same domain

Avoid repeatedly deleting and reissuing certificates unnecessarily.

---

## Verify Deployment

| URL | Expected Result |
|---|---|
| `http://myportfolio.sopanha.digital` | Redirects or serves site |
| `https://myportfolio.sopanha.digital` | Site loads with valid SSL |
| `https://myportfolio.sopanha.digital/api/health` | `{"status":"OK"}` |
| `https://myportfolio.sopanha.digital/admin` | Admin login page |

---

## Admin Panel

1. Visit `https://myportfolio.sopanha.digital/admin`
2. Login with password: `admin123`
3. You can now manage:
   - **Projects** — add/edit/delete with image uploads
   - **Skills** — add/edit/delete
   - **About / Hero** — edit text, upload profile photo, upload CV PDF
   - **Contact** — edit email, phone, location, social links

> ⚠️ Change the admin password in `backend/routes/auth.js` before going live.

---

## Redeploy After Updates

Whenever you change code and want to update the live site:

### 1. Rebuild locally

```bash
cd backend
docker build -t yourdockerhubusername/portfolio-backend:latest .
cd ../frontend
docker build -t yourdockerhubusername/portfolio-frontend:latest .
cd ..
```

### 2. Push to Docker Hub

```bash
docker push yourdockerhubusername/portfolio-backend:latest
docker push yourdockerhubusername/portfolio-frontend:latest
```

### 3. Update server

```bash
ssh root@your_server_ip
cd /var/www/myportfolio
docker compose pull
docker compose up -d
```

Data and uploaded files persist in Docker volumes, so they won't be lost.

---

## Hosting Multiple Websites on One Server

Your server has one IP address. When someone visits `myportfolio.com` or `othersite.com`, both domains can point to that same IP. Both requests arrive at the same server doors: port `80` (HTTP) and port `443` (HTTPS).

You need a **reverse proxy** — like a doorman — that checks the domain name and sends each visitor to the right website.

### The simple rule

- **Backend containers** must use different host ports: `3001`, `3002`, `3003`, etc.
- **Frontend containers** always listen on port `80` inside their container, but on the host they must either:
  - Use a reverse proxy like **Traefik** (no host port needed)
  - Use different host ports like `8080`, `8081` with host nginx

### Example: two websites on one server

| Website | Backend host port | Frontend host port | Domain |
|---|---|---|---|
| MyPortfolio | `3001` | `80` via Traefik | `myportfolio.com` |
| OtherSite | `3002` | `80` via Traefik | `othersite.com` |

With Traefik, both frontends share port `80` and `443` because Traefik routes by domain name internally.

---

### Option 1: Use Traefik (Recommended)

Traefik is a reverse proxy made for Docker. It handles domain routing and SSL automatically.

#### 1. Create a shared Docker network

```bash
docker network create web
```

#### 2. Run Traefik

Create `/var/www/traefik/docker-compose.yml`:

```yaml
services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    command:
      - "--api.dashboard=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.tlschallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.email=your-email@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt
    networks:
      - web

networks:
  web:
    external: true
```

Start it:

```bash
cd /var/www/traefik
docker compose up -d
```

#### 3. Update MyPortfolio compose file

Remove the `80:80` and `443:443` port mappings from the `frontend` service. Add Traefik labels and the shared network.

```yaml
services:
  backend:
    image: yourdockerhubusername/portfolio-backend:latest
    container_name: portfolio-backend
    restart: unless-stopped
    environment:
      DATA_DIR: /app/data
      PORT: 3001
      CORS_ORIGIN: https://myportfolio.sopanha.digital
      JWT_SECRET: ${JWT_SECRET}
    volumes:
      - portfolio-data:/app/data
      - backend-uploads:/app/uploads
    ports:
      - "3001:3001"
    networks:
      - web

  frontend:
    image: yourdockerhubusername/portfolio-frontend:latest
    container_name: portfolio-frontend
    restart: unless-stopped
    depends_on:
      - backend
    networks:
      - web
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.portfolio.rule=Host(`myportfolio.sopanha.digital`)"
      - "traefik.http.routers.portfolio.entrypoints=websecure"
      - "traefik.http.routers.portfolio.tls.certresolver=letsencrypt"
      - "traefik.http.services.portfolio.loadbalancer.server.port=80"
    environment:
      DOMAIN: ${DOMAIN}
      EMAIL: ${EMAIL}

volumes:
  portfolio-data:
  backend-uploads:

networks:
  web:
    external: true
```

> With Traefik handling SSL, the certbot inside the frontend image is not needed. You can keep it or build a clean nginx-only frontend image.

#### 4. Add a second website

Create another folder, for example `/var/www/othersite`, with this `docker-compose.yml`:

```yaml
services:
  backend:
    image: yourdockerhubusername/other-backend:latest
    container_name: other-backend
    restart: unless-stopped
    environment:
      DATA_DIR: /app/data
      PORT: 3001
      CORS_ORIGIN: https://othersite.com
      JWT_SECRET: ${JWT_SECRET}
    volumes:
      - other-data:/app/data
      - other-uploads:/app/uploads
    ports:
      - "3002:3001"
    networks:
      - web

  frontend:
    image: yourdockerhubusername/other-frontend:latest
    container_name: other-frontend
    restart: unless-stopped
    depends_on:
      - backend
    networks:
      - web
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.othersite.rule=Host(`othersite.com`)"
      - "traefik.http.routers.othersite.entrypoints=websecure"
      - "traefik.http.routers.othersite.tls.certresolver=letsencrypt"
      - "traefik.http.services.othersite.loadbalancer.server.port=80"

volumes:
  other-data:
  other-uploads:

networks:
  web:
    external: true
```

Important differences from MyPortfolio:
- Backend host port is `3002` instead of `3001`
- Container name is `other-backend` / `other-frontend`
- Traefik router name is `othersite`
- Domain is `othersite.com`

Start it:

```bash
cd /var/www/othersite
docker compose up -d
```

Now both websites share ports `80` and `443` on the server, and Traefik sends visitors to the right container based on the domain.

---

### Option 2: Use Host Nginx as Reverse Proxy

If you prefer not to use Traefik, install nginx directly on the server and route domains to different container ports.

#### 1. Install nginx on the host

```bash
sudo apt install -y nginx
```

#### 2. Use different frontend host ports

For MyPortfolio:

```yaml
frontend:
  ports:
    - "8080:80"
```

For OtherSite:

```yaml
frontend:
  ports:
    - "8081:80"
```

#### 3. Create nginx config for each site

MyPortfolio:

```bash
sudo nano /etc/nginx/sites-available/myportfolio
```

```nginx
server {
    listen 80;
    server_name myportfolio.sopanha.digital;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

OtherSite:

```bash
sudo nano /etc/nginx/sites-available/othersite
```

```nginx
server {
    listen 80;
    server_name othersite.com;

    location / {
        proxy_pass http://localhost:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Enable both:

```bash
sudo ln -s /etc/nginx/sites-available/myportfolio /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/othersite /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

#### 4. Add SSL with Certbot on host

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d myportfolio.sopanha.digital -d othersite.com
```

---

### Option 3: Use Different Ports (Testing only)

Run the second website on non-standard ports:

```yaml
frontend:
  ports:
    - "8080:80"
    - "8443:443"
```

Then access it at `http://your-server-ip:8080`. This is not recommended for real domains because users have to type the port number.

---

### Which option should you choose?

| Option | Best when | Difficulty |
|---|---|---|
| **Traefik** | Hosting many websites, want automatic SSL | Medium |
| **Host nginx** | You prefer full manual control | Medium |
| **Different ports** | Quick testing only | Easy |

For most people, **Traefik** is the best long-term choice.

---

## Troubleshooting

### Domain doesn't resolve

```bash
nslookup myportfolio.sopanha.digital
```

Wait for DNS propagation. Can take a few minutes to 24 hours.

### Let's Encrypt fails

Make sure:
- Port 80 is open
- Domain points to the server IP
- No other service is using port 80

Check logs:
```bash
docker compose logs -f frontend
```

### Certificate renewal

The frontend container runs `certbot renew` automatically every 12 hours. To renew manually:

```bash
docker exec portfolio-frontend certbot renew --nginx
```

### Reset SSL certificate

```bash
cd /var/www/myportfolio
docker compose down
docker volume rm myportfolio_letsencrypt
docker compose up -d
```

### Reset all data

⚠️ This deletes all projects, skills, settings, uploads, and CV.

```bash
cd /var/www/myportfolio
docker compose down
docker volume rm myportfolio_portfolio-data myportfolio_backend-uploads myportfolio_letsencrypt
docker compose up -d
```

---

## Useful Commands

```bash
# View running containers
docker ps

# View all logs
docker compose logs

# View frontend logs
docker compose logs -f frontend

# View backend logs
docker compose logs -f backend

# Restart services
docker compose restart

# Stop everything
docker compose down

# Enter backend container
docker exec -it portfolio-backend sh

# Enter frontend container
docker exec -it portfolio-frontend sh

# Backup data volume
docker run --rm -v myportfolio_portfolio-data:/data -v $(pwd):/backup alpine tar czf /backup/portfolio-data.tar.gz -C /data .

# Restore data volume
docker run --rm -v myportfolio_portfolio-data:/data -v $(pwd):/backup alpine sh -c "cd /data && tar xzf /backup/portfolio-data.tar.gz"
```

---

## Project Structure

```
myportfolio/
├── backend/
│   ├── Dockerfile
│   ├── server.js
│   ├── routes/
│   ├── data/          # Default local data (kept in repo)
│   └── uploads/       # Uploads directory
├── frontend/
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── entrypoint.sh  # Certbot + nginx
│   └── src/
├── docker-compose.yml
├── .env.example
├── DOCKER.md          # Detailed Docker reference
└── README.md          # This file
```

---

## Security Notes

- Change the default admin password before production.
- Use a strong, random `JWT_SECRET`.
- Keep `.env` out of Git.
- Regularly update base images and dependencies.
- The backend API port `3001` is exposed to the server. Only ports 80 and 443 need to be open to the internet.

---

<p align="center">
  <sub>Built and deployed with Docker + Let's Encrypt</sub>
</p>
