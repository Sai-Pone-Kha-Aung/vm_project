# Azure VM Deployment Guide: React Frontend + Next.js Backend

This guide walks you through deploying this project (`vm_project`) to an **Azure Virtual Machine** using **Docker**, **Docker Compose**, and **Nginx** as a reverse proxy.

---

## Architecture Overview

```
                 [ User Browser ]
                        │
                  Port 80 (HTTP)
                        │
                ┌───────▼───────┐
                │  Nginx Server │ (Reverse Proxy & Static Host)
                └───────┬───────┘
                        │
        ┌───────────────┴───────────────┐
        │                               │
   Requests to `/`              Requests to `/api/*`
        │                               │
┌───────▼──────────────┐       ┌────────▼─────────────┐
│ React Frontend (SPA) │       │ Next.js Backend      │
│  (Static files via   │       │ Container (Node.js)  │
│   Nginx try_files)   │       │ Port 3000 (internal) │
└──────────────────────┘       └──────────────────────┘
```

> **Why this design?**
> 1. **Single Entry Point**: Both frontend and backend are accessed via the same public IP on port 80.
> 2. **No CORS Issues**: Since both frontend and backend share the same origin, API calls to `/api` don't trigger cross-origin restrictions.
> 3. **Client-Side Routing Support**: Nginx handles React Router fallback (`try_files $uri $uri/ /index.html`) so refreshing deep links won't return 404.
> 4. **Isolated Backend**: The Next.js container port 3000 is internal to Docker and not directly exposed to the internet.

---

## Project Structure

Your project layout should look like this:

```text
vm_project/
├── docker-compose.yml              <-- Orchestrates both containers
├── .env                            <-- Environment variables (database, secrets)
├── instruction.md
├── next-backend/
│   ├── Dockerfile                  <-- Builds Next.js backend image
│   ├── .dockerignore
│   ├── package.json
│   └── src/
└── react-frontend/
    ├── Dockerfile                  <-- Multi-stage build (Vite build + Nginx serve)
    ├── .dockerignore
    ├── nginx.conf                  <-- Nginx reverse proxy configuration
    ├── package.json
    └── src/
```

---

## Step 1: Configure Azure Virtual Machine & Network Security Group (NSG)

### 1. VM Specifications Recommended
- **OS**: Ubuntu Server 22.04 LTS or 24.04 LTS
- **Size**: `Standard_B1s` (1 vCPU, 1 GiB RAM - eligible for Azure Free Tier) or `Standard_B2s` (2 vCPU, 4 GiB RAM - smoother builds)
- **Authentication**: SSH Public Key (recommended) or Password

### 2. Configure Inbound Port Rules (NSG)
Before connecting, ensure Azure firewall allows incoming web and SSH traffic:

1. In the [Azure Portal](https://portal.azure.com), navigate to your **Virtual Machine**.
2. Select **Networking** (or **Network settings** under Settings).
3. Under **Inbound port rules**, click **Add inbound port rule** for each:

| Rule Name | Destination Port | Protocol | Action | Priority | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Allow-SSH` | `22` | TCP | Allow | 300 | Access server terminal |
| `Allow-HTTP` | `80` | TCP | Allow | 310 | Public web traffic |
| `Allow-HTTPS` | `443` | TCP | Allow | 320 | *(Optional)* SSL/TLS traffic |

4. Copy your VM's **Public IP address** from the Overview page.

> [!IMPORTANT]
> **Database Whitelist (e.g., MongoDB Atlas / Azure Cosmos DB / Postgres):**
> If your backend connects to an external database, add your Azure VM's Public IP to the database provider's IP whitelist / Network Access list.

---

## Step 2: Configure Project Files Locally

Create the configuration files in their respective folders:

### 1. Backend Dockerfile (`next-backend/Dockerfile`)

Create or update [next-backend/Dockerfile](file:///Users/y2k/school/1_2026/web_dev/vm_project/next-backend/Dockerfile):

```dockerfile
# Use Node.js 20 LTS Alpine image
FROM node:20-alpine

WORKDIR /app

# Set environment
ENV NODE_ENV=production
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

# Install libc compatibility library (required for some Node packages)
RUN apk add --no-cache libc6-compat

# Install dependencies
COPY package.json package-lock.json* ./
RUN npm ci

# Copy source code
COPY . .

# Build Next.js application
RUN npm run build

# Expose internal port
EXPOSE 3000

# Start Next.js production server
CMD ["npm", "run", "start"]
```

### 2. Backend Dockerignore (`next-backend/.dockerignore`)

Create or update [next-backend/.dockerignore](file:///Users/y2k/school/1_2026/web_dev/vm_project/next-backend/.dockerignore):

```text
node_modules
.next
.git
.DS_Store
*.local
npm-debug.log*
```

---

### 3. Frontend Nginx Configuration (`react-frontend/nginx.conf`)

Create [react-frontend/nginx.conf](file:///Users/y2k/school/1_2026/web_dev/vm_project/react-frontend/nginx.conf):

```nginx
server {
    listen 80;
    server_name _;

    # Maximum request size (for file uploads if applicable)
    client_max_body_size 20M;

    # 1. Reverse Proxy for Backend API
    # All requests starting with /api will be routed to the Next.js container
    location /api {
        proxy_pass http://backend:3000;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_cache_bypass $http_upgrade;

        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # 2. Serve React SPA static files
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        # Crucial for client-side routing: fallback to index.html
        try_files $uri $uri/ /index.html;
    }

    # 3. Cache static assets (JS, CSS, images, fonts)
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        root /usr/share/nginx/html;
        expires 1y;
        add_header Cache-Control "public, no-transform";
    }

    # Error handling
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```

---

### 4. Frontend Dockerfile (`react-frontend/Dockerfile`)

Create or update [react-frontend/Dockerfile](file:///Users/y2k/school/1_2026/web_dev/vm_project/react-frontend/Dockerfile):

```dockerfile
# Stage 1: Build Vite React project
FROM node:20-alpine AS builder

WORKDIR /app

# Install dependencies
COPY package.json package-lock.json* ./
RUN npm ci

# Copy source files
COPY . .

# Set empty VITE_API_URL so that API calls use relative paths (e.g., fetch('/api/...'))
ENV VITE_API_URL=""

# Build production assets into dist/
RUN npm run build

# Stage 2: Serve static bundle using Nginx
FROM nginx:alpine

# Copy custom Nginx configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy compiled static files from the builder stage
COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### 5. Frontend Dockerignore (`react-frontend/.dockerignore`)

Create or update [react-frontend/.dockerignore](file:///Users/y2k/school/1_2026/web_dev/vm_project/react-frontend/.dockerignore):

```text
node_modules
dist
.git
.DS_Store
*.local
npm-debug.log*
```

---

### 6. Root Docker Compose (`docker-compose.yml`)

Create `docker-compose.yml` in the root `vm_project/` folder:

```yaml
version: "3.8"

services:
  # Next.js Backend Service
  backend:
    build:
      context: ./next-backend
      dockerfile: Dockerfile
    container_name: next_backend
    restart: always
    environment:
      - NODE_ENV=production
      - PORT=3000
    env_file:
      - .env
    networks:
      - app_network
    # Notice: port 3000 is NOT exposed publicly; requests route through Nginx

  # React Frontend + Nginx Reverse Proxy Service
  frontend:
    build:
      context: ./react-frontend
      dockerfile: Dockerfile
    container_name: react_frontend
    restart: always
    ports:
      - "80:80"
    depends_on:
      - backend
    networks:
      - app_network

networks:
  app_network:
    driver: bridge
```

---

### 7. Environment File (`.env`)

Create `.env` in the root `vm_project/` folder:

```env
# Database & Backend Environment Variables
NODE_ENV=production
PORT=3000

# Add your database connection string and secrets here if needed:
# MONGODB_URI=mongodb+srv://<user>:<password>@cluster0.mongodb.net/mydb
# JWT_SECRET=your_secret_key_here
```

> [!WARNING]
> Keep your `.env` private. Add `.env` to `.gitignore` so secrets are never pushed to a public repository.

---

## Step 3: Setup & Deploy on the Azure VM

### 1. Connect to your Azure VM via SSH

Run this command from your local terminal:

```bash
# If using an SSH key:
ssh -i /path/to/your-key.pem <azure-username>@<azure-vm-public-ip>

# Or if using password authentication:
ssh <azure-username>@<azure-vm-public-ip>
```

---

### 2. Configure Swap Space (Crucial for 1GB RAM / B1s VMs)

> [!TIP]
> If you are using Azure's `Standard_B1s` (1 GiB RAM), `npm run build` or `npm ci` may fail with **Exit code 137 (Out of Memory)**. Adding a 2 GB swap file prevents build crashes:

Run on the Azure VM:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Verify swap is active:
```bash
free -h
```

---

### 3. Install Docker and Docker Compose on Ubuntu

Run these commands on the Azure VM terminal:

```bash
# 1. Update package index
sudo apt update && sudo apt upgrade -y

# 2. Install prerequisites
sudo apt install -y ca-certificates curl gnupg lsb-release

# 3. Add Docker GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 4. Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 5. Install Docker Engine and Docker Compose plugin
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 6. Allow running Docker without sudo
sudo usermod -aG docker $USER

# 7. Activate Docker group changes
newgrp docker

# 8. Verify Docker installation
docker --version
docker compose version
```

---

### 4. Transfer Project to the Azure VM

Choose **Option A** (Git) or **Option B** (`rsync` from your local machine):

#### Option A: Clone via Git (Recommended)

On the Azure VM:
```bash
git clone <your-git-repository-url> app
cd app
```

Then create your `.env` file:
```bash
nano .env
# Paste your environment variables, then save (Ctrl+O, Enter, Ctrl+X)
```

#### Option B: Sync directly from your local machine via `rsync`

From your **local Mac terminal**:
```bash
rsync -avz --progress \
  --exclude 'node_modules' \
  --exclude '.next' \
  --exclude 'dist' \
  --exclude '.git' \
  /Users/y2k/school/1_2026/web_dev/vm_project/ \
  <azure-username>@<azure-vm-public-ip>:~/app/
```

Then SSH into the Azure VM:
```bash
ssh <azure-username>@<azure-vm-public-ip>
cd ~/app
```

---

### 5. Build and Start the Containers

Inside `~/app` on the Azure VM:

```bash
docker compose up -d --build
```

Docker will:
1. Build the Next.js backend image and start the container on internal port 3000.
2. Build the Vite React frontend image and start Nginx on port 80.
3. Wire them together on the Docker bridge network `app_network`.

---

### 6. Verify Deployment

1. **Check container status:**
   ```bash
   docker compose ps
   ```
   Both `next_backend` and `react_frontend` should report status `Up`.

2. **Inspect logs if needed:**
   ```bash
   # Live logs from all containers
   docker compose logs -f

   # Backend logs only
   docker compose logs -f backend

   # Frontend/Nginx logs only
   docker compose logs -f frontend
   ```

3. **Visit in your browser:**
   Open:
   ```
   http://<YOUR_AZURE_VM_PUBLIC_IP>
   ```
   - Main page loads the React frontend.
   - Any requests made to `http://<YOUR_AZURE_VM_PUBLIC_IP>/api/*` are reverse-proxied to Next.js.

---

## Management Cheat Sheet

| Operation | Command |
| :--- | :--- |
| View running containers | `docker compose ps` |
| View real-time logs | `docker compose logs -f` |
| Restart all containers | `docker compose restart` |
| Rebuild & restart after code changes | `docker compose up -d --build` |
| Stop all containers | `docker compose down` |
| Stop and remove volumes | `docker compose down -v` |
| Free up unused Docker images & cache | `docker system prune -af` |

---

## Troubleshooting Guide

### 1. Page won't load at `http://<AZURE_VM_IP>`
- Check Azure Portal > VM > Networking: Ensure **Inbound Rule for Port 80 (TCP)** is set to **Allow**.
- If Ubuntu's built-in firewall (`ufw`) is active, allow port 80:
  ```bash
  sudo ufw allow 80/tcp
  sudo ufw reload
  ```

### 2. `502 Bad Gateway` on `/api` calls
- The backend container is either still starting or crashed.
- Check backend logs:
  ```bash
  docker compose logs backend
  ```
- Make sure `HOSTNAME="0.0.0.0"` is present in the backend Dockerfile.

### 3. Docker build killed (`Exit Code 137`)
- This is an **Out of Memory (OOM)** error during `npm run build` or `npm ci`.
- Follow **Step 3.2** above to configure a 2GB swapfile.

### 4. Code changes not showing up
- Rebuild containers without using cached layers:
  ```bash
  docker compose build --no-cache
  docker compose up -d
  ```
