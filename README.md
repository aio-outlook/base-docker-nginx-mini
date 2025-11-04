# 🐳 Production Setup Guide — Docker + NGINX on Ubuntu (DigitalOcean / EC2)

A complete step-by-step guide to set up **Docker**, **Docker Compose**, and **NGINX** on a production Ubuntu 22.04 server such as a **DigitalOcean Droplet**, **AWS EC2**, or any **VPS**.  
Includes production hardening (rate limiting, upload limit, security headers) and SSL preparation for Certbot.

---

## 📋 Prerequisites

Before starting, ensure you have:

- A **DigitalOcean**, **AWS EC2**, or **VPS** account  
- **Ubuntu 22.04 LTS** installed  
- **sudo/root** access  
- (Optional) A domain name — e.g. `admin.haul247.co` and `www.admin.haul247.co`

---

## 🏗️ Step 1: Create Your Server

### On DigitalOcean
1. Visit [https://cloud.digitalocean.com](https://cloud.digitalocean.com)
2. Click **Create → Droplet**
3. Choose:
   - **Image:** Ubuntu 22.04 LTS  
   - **Plan:** Basic  
   - **Region:** Closest to your users  
   - **Authentication:** SSH key (recommended)
4. Click **Create Droplet** and note your server’s public IP.

### On AWS EC2
1. Go to [AWS Console](https://console.aws.amazon.com/ec2)
2. **Launch Instance**
3. Choose:
   - **Image:** Ubuntu 22.04 LTS  
   - **Type:** t2.micro or higher  
   - Open ports 22 (SSH), 80 (HTTP), 443 (HTTPS)
4. Connect:
   ```bash
   ssh ubuntu@your_server_ip
   ```

### On Other VPS Providers
Create a Linux server (Ubuntu 22.04 LTS) and connect:
```bash
ssh root@your_server_ip
```

---

## 🧰 Step 2: Update and Prepare the System
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install ca-certificates curl gnupg lsb-release -y
```

---

## 🐋 Step 3: Install Docker
```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
```

Verify Docker:
```bash
sudo docker run hello-world
```

(Optional) Add your user to the docker group:
```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## ⚙️ Step 4: Install Docker Compose
```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

---

## 🌐 Step 5: Install NGINX
```bash
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
systemctl status nginx
```

---

## 🔥 Step 6: Configure Firewall
```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
sudo ufw status
```

---

## 🚀 Step 7: Configure NGINX (Production-Grade)

Create a new NGINX config file:
```bash
sudo nano /etc/nginx/sites-available/haul247.conf
```

Paste the following configuration:

```nginx
# =========================================================
# Production NGINX Reverse Proxy for admin.haul247.co
# SSL will be handled later by Certbot
# =========================================================

# Define rate limit zone: 10MB shared memory, 10 requests/sec per IP
limit_req_zone $binary_remote_addr zone=haul247_limit:10m rate=10r/s;

server {
    listen 80;
    server_name admin.haul247.co www.admin.haul247.co;

    # Increase upload size for production
    client_max_body_size 300M;

    # Timeout & security tuning
    keepalive_timeout 65;
    send_timeout 30;
    proxy_read_timeout 90;
    proxy_connect_timeout 90;

    # Basic security headers
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";
    server_tokens off;

    # Reverse proxy to backend (Docker app on port 3000)
    location / {
        limit_req zone=haul247_limit burst=20 nodelay;

        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
    }

    access_log /var/log/nginx/haul247.access.log;
    error_log /var/log/nginx/haul247.error.log;
}
```

Enable and reload NGINX:
```bash
sudo ln -s /etc/nginx/sites-available/haul247.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 🔒 Step 8: (Optional) Enable SSL with Certbot
Once your domain DNS is pointed to the server:
```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d admin.haul247.co -d www.admin.haul247.co
sudo systemctl reload nginx
```
Certbot will automatically generate and apply SSL certificates and redirect HTTP → HTTPS.

---

## 🧩 Step 9: Test Everything
Visit:
```
http://admin.haul247.co
```
If your app loads correctly, NGINX is proxying to your backend.

---

## 🧯 Step 10: Troubleshooting

| Task | Command |
|------|----------|
| Check NGINX logs | `sudo tail -f /var/log/nginx/error.log` |
| Check Docker logs | `docker ps` + `docker logs <container_id>` |
| Restart Docker | `sudo systemctl restart docker` |
| Restart NGINX | `sudo systemctl restart nginx` |

---

## 🏁 Summary

✅ You’ve successfully:

1. Created a cloud server (DigitalOcean / EC2)  
2. Updated and secured your system  
3. Installed and configured Docker + Docker Compose  
4. Installed and secured NGINX with rate limiting  
5. Increased upload size to 300 MB  
6. Prepared the system for SSL via Certbot  

Your environment is now **production-ready**, scalable, and secure. 🚀
