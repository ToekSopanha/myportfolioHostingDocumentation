# What Changed: From Nginx/PM2 to Docker Deployment

This document explains the differences between the original deployment method (Ubuntu + nginx + PM2) and the new deployment method (Docker + Docker Compose + Let's Encrypt in a container).

---

## Overview

| | Old Setup | New Setup |
|---|---|---|
| **Web server** | Host nginx (installed on Ubuntu) | nginx inside Docker container |
| **Backend runner** | PM2 process manager | Docker container |
| **SSL certificates** | Certbot on host | Certbot inside frontend container |
| **Data storage** | `backend/data/` on host filesystem | Docker volume at `/app/data` |
| **Uploads** | `backend/uploads/` on host filesystem | Docker volume at `/app/uploads` |
| **Deployment** | `scp` files + `pm2 restart` | `docker build`, `docker push`, `docker compose up -d` |
| **Multiple sites** | One host nginx config per site | Traefik or host nginx as reverse proxy |

---

## New Code Files

These files were added to support Docker and the new features.

### Backend

| File | Purpose |
|---|---|
| `backend/lib/dataPath.js` | Makes the JSON data directory configurable via `DATA_DIR` env var |
| `backend/routes/settings.js` | API for hero/greeting text |
| `backend/routes/about.js` | API for about section text and profile photo |
| `backend/routes/contact.js` | API for contact info and social links |
| `backend/routes/upload.js` | API for uploading profile photos, project images, and CV PDF |
| `backend/routes/cv.js` | API for downloading the CV PDF |
| `backend/Dockerfile` | Builds the backend Docker image |
| `backend/.dockerignore` | Excludes files from backend Docker image |

### Frontend

| File | Purpose |
|---|---|
| `frontend/src/components/GradientBackground.vue` | Animated gradient background |
| `frontend/src/components/MobileMenu.vue` | Mobile navigation drawer |
| `frontend/Dockerfile` | Builds the frontend Docker image with nginx + certbot |
| `frontend/nginx.conf` | Nginx config inside the frontend container |
| `frontend/entrypoint.sh` | Requests and auto-renews Let's Encrypt certificates |

### Project root

| File | Purpose |
|---|---|
| `docker-compose.yml` | Runs backend + frontend containers together |
| `.dockerignore` | Excludes files from Docker build context |
| `.env.example` | Template for required environment variables |
| `DOCKER.md` | Detailed Docker reference |

---

## Modified Code Files

These files were changed from the original portfolio.

### Backend

| File | What Changed |
|---|---|
| `backend/server.js` | Mounts new API routes, serves `/uploads` as static files, reads `CORS_ORIGIN` and `JWT_SECRET` env vars |
| `backend/routes/projects.js` | Now uses `dataPath.js` helper for configurable data location |
| `backend/routes/skills.js` | Now uses `dataPath.js` helper for configurable data location |
| `backend/routes/auth.js` | JWT secret now reads from `process.env.JWT_SECRET` with dev fallback |
| `backend/package.json` | Added `multer` dependency for file uploads |

### Frontend

| File | What Changed |
|---|---|
| `frontend/src/main.js` | Imports Bootstrap and Bootstrap-Icons CSS |
| `frontend/src/App.vue` | New cyan/purple/pink gradient theme, glassmorphism styles |
| `frontend/src/components/NavBar.vue` | Added mobile hamburger menu, fetches contact social links |
| `frontend/src/components/HeroSection.vue` | Fetches content from `/api/settings`, added CV download button |
| `frontend/src/components/AboutSection.vue` | Fetches content from `/api/about`, shows uploaded profile photo |
| `frontend/src/components/ContactSection.vue` | Fetches content from `/api/contact` |
| `frontend/src/components/ProjectsSection.vue` | Handles uploaded project images from `/uploads/projects/` |
| `frontend/src/components/SkillsSection.vue` | Improved responsive breakpoints |
| `frontend/src/components/AdminPanel.vue` | New About/Hero tab, Contact tab, file uploads for photos and CV |

---

## What Changed in Configuration

### Web server

**Before:**
- Nginx installed directly on the Ubuntu server.
- Config file at `/etc/nginx/sites-available/myportfolio`.
- Served static files from `/var/www/myportfolio/frontend/dist`.
- Reverse proxied `/api/` to `http://localhost:3001`.

**Now:**
- Nginx runs inside the `frontend` Docker container.
- Config file is `frontend/nginx.conf`.
- Serves static files from `/usr/share/nginx/html` inside the container.
- Reverse proxies `/api/` and `/uploads/` to the `backend` container.

### Backend runner

**Before:**
- Backend started with `pm2 start server.js`.
- PM2 kept the process alive and restarted it on crash.

**Now:**
- Backend runs as a Docker container.
- Docker `restart: unless-stopped` policy keeps it running.

### SSL / HTTPS

**Before:**
- Certbot installed on the host.
- Command: `sudo certbot --nginx -d myportfolio.sopanha.digital`.
- Certbot modified the host nginx config.
- Manual renewal or cron job on host.

**Now:**
- Certbot installed inside the frontend container.
- `frontend/entrypoint.sh` requests the certificate on first run.
- Automatically runs `certbot renew` every 12 hours inside the container.
- Certificates stored in `/etc/letsencrypt/` which is mounted to a Docker volume.

### Data storage

**Before:**
- JSON data files in `backend/data/` on the host.
- You could edit them directly with `nano` or `vim`.

**Now:**
- JSON data files in Docker volume mounted at `/app/data` inside the backend container.
- Default local development still uses `backend/data/`.
- Docker uses `/app/data` via the `DATA_DIR` environment variable.
- Data persists across container restarts because of the volume.

### Uploads

**Before:**
- Project images were external URLs.
- Profile photo was `src/assets/profile.jpg` inside the frontend build.
- No CV upload feature.

**Now:**
- Profile photos, project thumbnails, and CV PDF are uploaded to the backend.
- Stored in Docker volume mounted at `/app/uploads` inside the backend container.
- Served statically at `/uploads/`.

---

## What Changed in Deployment Commands

| Task | Before | Now |
|---|---|---|
| Deploy frontend | `npm run build` then `scp -r dist server:/var/www/...` | Build image, push to Docker Hub, pull on server |
| Deploy backend | `git pull && npm install && pm2 restart` | Build image, push to Docker Hub, pull on server |
| Start everything | `pm2 start server.js` + `systemctl start nginx` | `docker compose up -d` |
| Restart | `pm2 restart ...` + `systemctl reload nginx` | `docker compose restart` |
| View logs | `pm2 logs`, `tail /var/log/nginx/error.log` | `docker compose logs -f` |
| Update after code change | SSH in, pull code, restart | Rebuild image, push, server pulls and restarts |

---

## What Stayed the Same

These things did not change:

- Vue.js 3 + Vite for the frontend.
- Node.js + Express for the backend.
- JWT authentication for the admin panel.
- JSON files for data storage (no database added).
- The admin panel URL: `/admin`.
- Default admin password: `admin123`.
- Project structure of `frontend/src/components/`.
- Existing API routes for projects and skills.

---

## Why Docker?

Docker makes deployment more consistent and portable:

- **Same environment everywhere**: local machine, test server, production server.
- **No dependency conflicts**: each container has exactly what it needs.
- **Easy rollback**: just pull the previous image version.
- **Simpler SSL**: certbot runs inside the container with the app.
- **Data safety**: JSON files and uploads live in Docker volumes, separate from the image.

---

## Quick Migration Checklist

If you already have the old nginx/PM2 setup and want to switch to Docker:

1. **Backup your data**
   - Copy `backend/data/*.json`
   - Copy `backend/uploads/` (if any)

2. **Install Docker on the server**
   ```bash
   sudo apt install -y docker.io docker-compose-plugin
   ```

3. **Set environment variables**
   - Create `.env` with `JWT_SECRET`, `DOMAIN`, `EMAIL`.

4. **Build and push images**
   ```bash
   docker build -t yourname/portfolio-backend:latest ./backend
   docker build -t yourname/portfolio-frontend:latest ./frontend
   docker push yourname/portfolio-backend:latest
   docker push yourname/portfolio-frontend:latest
   ```

5. **Stop old services**
   ```bash
   sudo systemctl stop nginx
   pm2 stop all
   ```

6. **Run Docker on the server**
   ```bash
   docker compose pull
   docker compose up -d
   ```

7. **Copy old data into the Docker volume** (optional)
   ```bash
   # Copy local backend/data into the running container
   docker cp ./backend/data/. portfolio-backend:/app/data/
   ```

8. **Verify**
   - Visit `https://yourdomain.com`
   - Check `/api/health`
   - Test admin login

---

## Multiple Websites on One Server

With the old host nginx setup, adding a second site meant creating another config file in `/etc/nginx/sites-available/`.

With Docker, you have two main choices:

1. **Traefik reverse proxy** — recommended. One container listens on ports 80/443 and routes by domain name to other containers.
2. **Host nginx reverse proxy** — install nginx on the host, route domains to different container ports like `8080`, `8081`.

See `README.md` → "Hosting Multiple Websites on One Server" for full details.

---

## Files You Can Delete After Migration

These old deployment files are no longer needed if you fully switch to Docker:

- Host nginx config: `/etc/nginx/sites-available/myportfolio`
- PM2 config (if any): `ecosystem.config.js`
- The old manual deployment README section (kept in git history)

> Do **not** delete `backend/data/` or `backend/uploads/` until you have copied your data into Docker volumes.
