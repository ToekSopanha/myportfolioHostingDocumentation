# 🚀 MyPortfolio — Deployment Documentation

![Vue](https://img.shields.io/badge/Frontend-Vue.js-42b883?logo=vue.js&logoColor=white)
![Node](https://img.shields.io/badge/Backend-Node.js-339933?logo=node.js&logoColor=white)
![Nginx](https://img.shields.io/badge/Server-Nginx-009639?logo=nginx&logoColor=white)
![PM2](https://img.shields.io/badge/Process%20Manager-PM2-2B037A?logo=pm2&logoColor=white)
![Ubuntu](https://img.shields.io/badge/OS-Ubuntu-E95420?logo=ubuntu&logoColor=white)

This document describes the full production deployment of **MyPortfolio** — a Vue.js single-page application backed by a Node.js/Express API, served and reverse-proxied through nginx on an Ubuntu server.

| | |
|---|---|
| **Live domain** | `myportfolio.sopanha.digital` |
| **OS** | Ubuntu (DigitalOcean Droplet) |
| **Frontend** | Vue.js (built with Vite) |
| **Backend** | Node.js + Express, port `3001` |
| **Web server** | nginx (static file server + reverse proxy) |
| **Process manager** | PM2 |

---

## 📑 Table of Contents

- [1. Architecture Overview](#1-architecture-overview)
- [2. Project Structure](#2-project-structure)
- [3. Prerequisites](#3-prerequisites)
- [4. Step-by-Step Deployment](#4-step-by-step-deployment)
  - [4.1 Connect to the Server](#41-connect-to-the-server)
  - [4.2 Update the System & Install Nginx](#42-update-the-system--install-nginx)
  - [4.3 Install Node.js](#43-install-nodejs)
  - [4.4 Transfer Project Files](#44-transfer-project-files)
  - [4.5 Build the Frontend](#45-build-the-frontend)
  - [4.6 Set Up the Backend](#46-set-up-the-backend)
  - [4.7 Run the Backend with PM2](#47-run-the-backend-with-pm2)
  - [4.8 Configure Nginx](#48-configure-nginx)
  - [4.9 Enable the Site & Test Config](#49-enable-the-site--test-config)
  - [4.10 Configure the Firewall](#410-configure-the-firewall)
  - [4.11 Point the Domain (DNS)](#411-point-the-domain-dns)
  - [4.12 Enable HTTPS](#412-enable-https)
- [5. Verifying the Deployment](#5-verifying-the-deployment)
- [6. Troubleshooting Guide](#6-troubleshooting-guide)
- [7. Maintenance & Redeployment](#7-maintenance--redeployment)
- [8. Command Cheat Sheet](#8-command-cheat-sheet)

---

## 1. Architecture Overview

```
                         ┌─────────────────────────────┐
                         │           Browser            │
                         └───────────────┬───────────────┘
                                         │ HTTP/HTTPS
                                         ▼
                         ┌─────────────────────────────┐
                         │     nginx (port 80 / 443)    │
                         │  myportfolio.sopanha.digital │
                         └───────────────┬───────────────┘
                                         │
                  ┌──────────────────────┼──────────────────────┐
                  │                                              │
                  ▼                                              ▼
   ┌───────────────────────────┐                ┌───────────────────────────┐
   │   Static files (Vue dist) │                │   /api/*  → reverse proxy │
   │ /var/www/myportfolio/      │                │   http://localhost:3001   │
   │   frontend/dist            │                └─────────────┬─────────────┘
   └───────────────────────────┘                                │
                                                                 ▼
                                                   ┌───────────────────────────┐
                                                   │   Node.js / Express API   │
                                                   │   managed by PM2          │
                                                   │   /var/www/myportfolio/   │
                                                   │   backend/server.js       │
                                                   └───────────────────────────┘
```

**Design rationale:**
- The frontend is compiled into static HTML/CSS/JS ahead of time, so nginx can serve it directly with no runtime overhead — fast, and no Node.js needed on the request path.
- The backend is a long-running Node.js process, kept alive and auto-restarted by PM2, completely decoupled from nginx's own lifecycle.
- nginx acts as the single entry point on ports 80/443, routing traffic to the right place based on URL path — this means only nginx needs to be exposed to the internet; the backend stays internal on `localhost:3001`.

---

## 2. Project Structure

```
/var/www/myportfolio/
├── frontend/
│   ├── src/                       # Vue source code
│   ├── public/                    # Static assets used during dev
│   ├── package.json
│   ├── vite.config.js
│   └── dist/                      # ✅ Production build — this is what nginx serves
│       ├── index.html
│       ├── favicon.svg
│       └── assets/
│           ├── index-[hash].js
│           └── index-[hash].css
│
├── backend/
│   ├── server.js                  # Entry point — listens on port 3001
│   ├── routes/
│   ├── package.json
│   └── node_modules/              # Installed via `npm install --production`
│
└── img/                            # Static images served independently via nginx
```

---

## 3. Prerequisites

| Software | Purpose | Install Command |
|---|---|---|
| **nginx** | Web server & reverse proxy | `sudo apt install nginx -y` |
| **Node.js (v20.x) + npm** | Build frontend, run backend | See [4.3](#43-install-nodejs) |
| **PM2** | Persistent backend process manager | `sudo npm install -g pm2` |
| **Certbot** | Free SSL/TLS certificates | `sudo apt install certbot python3-certbot-nginx -y` |
| **ufw** | Server-level firewall | Pre-installed on most Ubuntu images |

---

## 4. Step-by-Step Deployment

### 4.1 Connect to the Server

```bash
ssh 
```

> Replace with your actual user if not using `root`.

---

### 4.2 Update the System & Install Nginx

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

**Verify:** visit `ServerIP` in a browser — you should see the default "Welcome to nginx!" page at this point. This confirms nginx is installed and running before any custom config is applied.

```bash
sudo systemctl status nginx
```

Expected output should show `active (running)`.

---

### 4.3 Install Node.js

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

**Verify:**

```bash
node -v     # e.g. v20.x.x
npm -v      # e.g. 10.x.x
```

---

### 4.4 Transfer Project Files

From your **local machine**, copy the project to the server:

```bash
scp -r frontend root@159.223.78.106:/var/www/myportfolio/
scp -r backend root@159.223.78.106:/var/www/myportfolio/
scp -r img root@159.223.78.106:/var/www/myportfolio/
```

> Alternatively, use `git clone` directly on the server if the project is in a Git repository.

---

### 4.5 Build the Frontend

```bash
cd /var/www/myportfolio/frontend
npm install
npm run build
```

This compiles the Vue app into the `dist/` folder — plain static HTML/CSS/JS, requiring **no Node.js runtime** to serve afterward.

**Verify:**

```bash
ls /var/www/myportfolio/frontend/dist
cat /var/www/myportfolio/frontend/dist/index.html
```

The output should reference hashed asset files, e.g.:

```html
<script type="module" crossorigin src="/assets/index-DJ7wh-1S.js"></script>
<link rel="stylesheet" crossorigin href="/assets/index-4C8ebk0s.css">
```

---

### 4.6 Set Up the Backend

```bash
cd /var/www/myportfolio/backend
npm install --production
```

Identify the entry file and listening port:

```bash
cat package.json | grep main          # "main": "server.js"
grep -n "PORT" server.js              # const PORT = process.env.PORT || 3001;
```

---

### 4.7 Run the Backend with PM2

```bash
sudo npm install -g pm2
pm2 start server.js --name myportfolio-backend
pm2 save
pm2 startup
```

Run whatever command `pm2 startup` prints — this registers PM2 with `systemd` so the backend restarts automatically on server reboot.

**Verify:**

```bash
pm2 status
```

Expected:

```
┌────┬────────────────────┬──────────┬──────┬───────────┬──────────┬──────────┐
│ id │ name               │ mode     │ ↺    │ status    │ cpu      │ memory   │
├────┼────────────────────┼──────────┼──────┼───────────┼──────────┼──────────┤
│ 0  │ myportfolio-backend│ fork     │ 0    │ online    │ 0%       │ 62.1mb   │
└────┴────────────────────┴──────────┴──────┴───────────┴──────────┴──────────┘
```

Test the backend directly, bypassing nginx entirely:

```bash
curl http://localhost:3001/api/projects
```

A valid JSON response confirms the backend itself is healthy, independent of any nginx configuration.

---

### 4.8 Configure Nginx

```bash
sudo nano /etc/nginx/sites-available/myportfolio
```

```nginx
server {
    listen 80;
    server_name myportfolio.sopanha.digital;

    root /var/www/myportfolio/frontend/dist;
    index index.html;

    # Serve the Vue SPA — fallback to index.html for client-side routing
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Serve static images
    location /img/ {
        alias /var/www/myportfolio/img/;
    }

    # Reverse proxy API requests to the Node.js backend
    location /api/ {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

<details>
<summary>📌 Why each directive matters (click to expand)</summary>

| Directive | Purpose |
|---|---|
| `try_files $uri $uri/ /index.html;` | Without this, refreshing on a Vue Router route like `/about` returns a 404, since nginx looks for a literal file named `about`. This falls back to `index.html` and lets Vue Router take over. |
| `alias /var/www/myportfolio/img/;` | Serves the `img/` folder independently of the frontend build. |
| `proxy_pass http://localhost:3001;` | Forwards `/api/*` requests to the Node.js backend. The port **must** match the backend's actual listening port exactly. |
| `proxy_set_header Upgrade/Connection` | Required if the backend uses WebSockets (e.g. Socket.io); without these, WebSocket connections fail silently. |

</details>

---

### 4.9 Enable the Site & Test Config

```bash
sudo ln -s /etc/nginx/sites-available/myportfolio /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

`nginx -t` should output:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

---

### 4.10 Configure the Firewall

**Server-level (ufw):**

```bash
sudo ufw status
sudo ufw allow 'Nginx Full'
sudo ufw reload
```

**Cloud provider firewall (DigitalOcean):**

Go to **DigitalOcean Dashboard → Networking → Firewalls** and confirm inbound rules allow:

| Type | Protocol | Port | Source |
|---|---|---|---|
| HTTP | TCP | 80 | All IPv4, All IPv6 |
| HTTPS | TCP | 443 | All IPv4, All IPv6 |
| SSH | TCP | 22 | Your IP (recommended) |

> ⚠️ This is a separate layer from `ufw` and is one of the most common reasons a server is unreachable externally even when everything *inside* the server checks out fine.

---

### 4.11 Point the Domain (DNS)

In your domain registrar / DNS provider, add an **A record**:

| Type | Name | Value |
|---|---|---|
| A | `myportfolio` (or `@`) | `ServerIP` |

Verify propagation:

```bash
nslookup myportfolio.sopanha.digital
```

Expected:

```
Non-authoritative answer:
Name:    myportfolio.sopanha.digital
Address: ServerIP
```

---

### 4.12 Enable HTTPS

```bash
sudo certbot --nginx -d myportfolio.sopanha.digital
```

Certbot automatically:
- Obtains a free SSL certificate from Let's Encrypt
- Edits the nginx config to add a `listen 443 ssl;` block
- Sets up a scheduled task for automatic renewal

**Verify auto-renewal:**

```bash
sudo certbot renew --dry-run
```

---

## 5. Verifying the Deployment

Run through this checklist after deployment:

- [ ] `curl http://localhost:3001/api/projects` returns valid JSON (backend healthy)
- [ ] `curl -s http://myportfolio.sopanha.digital` returns the Vue `index.html` (not the default nginx page)
- [ ] `nslookup myportfolio.sopanha.digital` resolves to `159.223.78.106`
- [ ] `pm2 status` shows `myportfolio-backend` as `online`
- [ ] Visiting `https://myportfolio.sopanha.digital` in a browser loads the site with a valid SSL padlock
- [ ] Refreshing on a non-root route (e.g. `/projects`) does **not** return a 404

---

## 6. Troubleshooting Guide

| Symptom | Likely Cause | Resolution |
|---|---|---|
| Default "Welcome to nginx!" page shown | Old default site still enabled, or wrong `root` path | `sudo rm /etc/nginx/sites-enabled/default`; verify `root` matches `dist/` path |
| `502 Bad Gateway` on API calls | `proxy_pass` port mismatch | Run `grep PORT server.js`, align with `proxy_pass` in nginx config |
| `404` on page refresh (e.g. `/about`) | Missing SPA fallback | Add `try_files $uri $uri/ /index.html;` |
| Domain doesn't resolve | DNS not propagated, or missing A record | `nslookup yourdomain.com`; check registrar's DNS settings |
| Site unreachable despite correct nginx + DNS | Cloud firewall blocking ports 80/443 | Check cloud provider's firewall/security group dashboard |
| `npm run build` fails referencing `#` character | Project path contains `#` | Rename directory to remove special characters |
| PM2 shows `online` but app doesn't respond | App crashed internally but PM2 hasn't flagged it yet | `pm2 logs myportfolio-backend --lines 50` to inspect actual errors |
| WebSocket connections drop | Missing `Upgrade`/`Connection` headers in proxy config | Add the WebSocket-related `proxy_set_header` lines shown in [4.8](#48-configure-nginx) |

---

## 7. Maintenance & Redeployment

**Updating the frontend:**

```bash
cd /var/www/myportfolio/frontend
git pull            # if using version control
npm install
npm run build
# No restart needed — nginx serves the new dist/ immediately
```

**Updating the backend:**

```bash
cd /var/www/myportfolio/backend
git pull
npm install --production
pm2 restart myportfolio-backend
```

**Applying nginx config changes:**

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 8. Command Cheat Sheet

```bash
# nginx
sudo systemctl status nginx
sudo systemctl reload nginx
sudo nginx -t
tail -f /var/log/nginx/error.log

# PM2
pm2 status
pm2 logs myportfolio-backend
pm2 restart myportfolio-backend
pm2 stop myportfolio-backend

# Networking
ufw status
ss -tlnp | grep :80
nslookup myportfolio.sopanha.digital
curl -I http://myportfolio.sopanha.digital

# SSL
sudo certbot renew --dry-run
```

---

<p align="center">
  <sub>Maintained by Toek Sopanha · Software Engineer | Network Engineer | Cloud/DevOps Engineer</sub>
</p>
