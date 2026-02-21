# Backend Deployment Guide (Node.js + Docker + Redis + EC2 + CI/CD)

This document explains **step-by-step how to deploy a production-ready backend** using:

* AWS EC2
* Docker
* Redis (Docker container)
* Nginx reverse proxy
* SSL (Let's Encrypt)
* GitHub Actions CI/CD
* Docker Hub

Every step includes the **exact commands used in the terminal**.

---

# 1. Create EC2 Instance (AWS)

1. Go to AWS Console
2. Open EC2
3. Click **Launch Instance**
4. Choose:

AMI:
Ubuntu Server 22.04 LTS

Instance type:
t2.micro (Free tier)

Create a new key pair:
example-key.pem

Launch the instance.

---

# 2. Connect to EC2 from Local Terminal

Move your key to a safe folder and connect.

Example:

```bash
chmod 400 example-key.pem

ssh -i example-key.pem ubuntu@EC2_PUBLIC_IP
```

Example real command:

```bash
ssh -i example-key.pem ubuntu@13.233.120.10
```

Now you are inside the EC2 server.

---

# 3. Update Server

```bash
sudo apt update
sudo apt upgrade -y
```

---

# 4. Install Docker

```bash
sudo apt install docker.io -y
```

Start Docker

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

Allow current user to run Docker

```bash
sudo usermod -aG docker ubuntu
```

Reconnect to server.

---

# 5. Install Docker Compose

```bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins

sudo curl -SL https://github.com/docker/compose/releases/download/v2.21.0/docker-compose-linux-x86_64 \
-o /usr/local/lib/docker/cli-plugins/docker-compose

sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
```

Verify:

```bash
docker compose version
```

---

# 6. Create Deployment Folder

```bash
mkdir BeatBay-Deploy
cd BeatBay-Deploy
```

Check folder

```bash
ls
```

---

# 7. Create docker-compose.yml

```bash
nano docker-compose.yml
```

Example:

```yaml
version: "3.9"

services:
  backend:
    image: yourdockerhubusername/beatbay-backend:latest
    container_name: beatbay-backend
    restart: always
    ports:
      - "5000:5000"
    env_file:
      - .env
    depends_on:
      - redis

  redis:
    image: redis:7
    container_name: beatbay-redis
    restart: always
    ports:
      - "6379:6379"
```

Save:

CTRL + X
Y
ENTER

---

# 8. Create .env File

```bash
nano .env
```

Example:

```env
PORT=5000
MONGO_URI=mongodb://your-db
JWT_SECRET=your-secret
REDIS_HOST=redis
REDIS_PORT=6379
STRIPE_SECRET=your-stripe-key
```

Save the file.

---

# 9. Run Containers

```bash
docker compose up -d
```

Check running containers:

```bash
docker ps
```

Check logs:

```bash
docker logs beatbay-backend
```

---

# 10. Install Nginx

```bash
sudo apt install nginx -y
```

Start Nginx

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

---

# 11. Configure Nginx

Create config:

```bash
sudo nano /etc/nginx/sites-available/beatbay
```

Example configuration:

```nginx
server {
    listen 80;
    server_name beatbay.online www.beatbay.online;

    location / {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Enable config:

```bash
sudo ln -s /etc/nginx/sites-available/beatbay /etc/nginx/sites-enabled/
```

Test:

```bash
sudo nginx -t
```

Reload:

```bash
sudo systemctl reload nginx
```

---

# 12. Install SSL Certificate

Install certbot

```bash
sudo apt install certbot python3-certbot-nginx -y
```

Create webroot

```bash
sudo mkdir -p /var/www/certbot
sudo chown -R www-data:www-data /var/www/certbot
```

Generate certificate

```bash
sudo certbot certonly --webroot -w /var/www/certbot -d beatbay.online -d www.beatbay.online
```

Certificates saved at:

/etc/letsencrypt/live/beatbay.online/

---

# 13. Update Nginx for HTTPS

Edit config again:

```bash
sudo nano /etc/nginx/sites-available/beatbay
```

Add SSL configuration.

Restart nginx:

```bash
sudo systemctl reload nginx
```

---

# 14. Setup Docker Hub

Login

```bash
docker login
```

Build image locally (on dev machine)

```bash
docker build -t yourdockerhubusername/beatbay-backend .
```

Push image

```bash
docker push yourdockerhubusername/beatbay-backend
```

---

# 15. Setup CI/CD (GitHub Actions)

Create workflow file:

```bash
mkdir -p .github/workflows
nano .github/workflows/deploy.yml
```

Example:

```yaml
name: Deploy Backend

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Login to DockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build Image
        run: docker build -t yourdockerhubusername/beatbay-backend .

      - name: Push Image
        run: docker push yourdockerhubusername/beatbay-backend

      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.EC2_KEY }}
          script: |
            cd BeatBay-Deploy
            docker compose pull
            docker compose up -d
```

---

# 16. GitHub Secrets Required

Add these in repository settings:

EC2_HOST
EC2_KEY
DOCKER_USERNAME
DOCKER_PASSWORD

---

# Deployment Flow

Developer pushes code →
GitHub Actions builds Docker image →
Pushes to Docker Hub →
SSH into EC2 →
Pull latest image →
Restart containers

---

# Useful Debug Commands

Check containers

```bash
docker ps
```

Check logs

```bash
docker logs container-name
```

Restart containers

```bash
docker compose restart
```

Stop containers

```bash
docker compose down
```

---

# Industry Best Practices

Use environment variables
Do not commit .env files
Use CI/CD pipeline
Use reverse proxy (Nginx)
Use HTTPS
Monitor logs
Use container restart policies

---

# Deployment Completed

Your backend is now production ready.

