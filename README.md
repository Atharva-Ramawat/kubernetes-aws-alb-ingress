Unattended 24/7 Cloud Deployment Plan: leadpharma.ai

This document outlines the step-by-step production deployment plan to run the entire leadpharma.ai ecosystem (FastAPI Backend, PostgreSQL, 24/7 IndiaMART Chrome Extension Bot, and React Web Platform) unattended in the cloud for under ₹600 / month ($7/mo).

1. System Architecture
2. Pre-Deployment Checklist & Accounts

Before beginning the deployment, acquire the following accounts and credentials:

VPS Provider: Hetzner Cloud
 account.
Plan: CX22 (2 vCPU, 4GB RAM, 40GB NVMe SSD, 20TB traffic).
Location: Falkenstein (FSN1) or Nuremberg (NBG1).
Operating System: Ubuntu 24.04 LTS.
Cost: €3.79/month (~₹380/mo).
Domain & DNS: Any .com or .in domain managed via Cloudflare
 (Free plan).
Google AI Studio Key: Free API key for Gemini 1.5 Flash from aistudio.google.com
.
SMS Gateway Account: Fast2SMS
 with ₹100 prepaid balance for OTPs.
Telegram Bot: Token and Chat ID generated from @BotFather and @userinfobot.
3. Phase 1: Server Provisioning & Base Hardening
1.1 Connect & Update System
bash
ssh root@YOUR_SERVER_IP
apt update && apt upgrade -y
apt install -y curl git ufw fail2ban unzip htop ca-certificates gnupg
1.2 Firewall Setup (UFW)

Only expose SSH, HTTP, and HTTPS ports to the public internet:

bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
1.3 Install Docker Engine & Compose Plugin
bash
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker
4. Phase 2: Database & Backend Deployment
2.1 Directory Structure
bash
mkdir -p /opt/leadpharma/backend /opt/leadpharma/nginx /opt/leadpharma/bot
cd /opt/leadpharma
2.2 Docker Compose Configuration (/opt/leadpharma/docker-compose.yml)
yaml
version: '3.8'
services:
  postgres:
    image: postgres:16-alpine
    container_name: leadpharma_postgres
    restart: always
    environment:
      POSTGRES_DB: pharmalead
      POSTGRES_USER: pharmalead_admin
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - internal_net
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: leadpharma_backend
    restart: always
    depends_on:
      - postgres
    env_file:
      - ./backend/.env
    networks:
      - internal_net
    ports:
      - "127.0.0.1:8000:8000"
volumes:
  pgdata:
networks:
  internal_net:
    driver: bridge
2.3 Backend Production Environment (/opt/leadpharma/backend/.env)
ini
PROJECT_NAME="leadpharma.ai"
ENVIRONMENT="production"
DEBUG=False
API_V1_STR="/api"
# Database
DATABASE_URL=postgresql://pharmalead_admin:${DB_PASSWORD}@postgres:5432/pharmalead
# Authentication
SECRET_KEY=GENERATE_STRONG_64_CHAR_HEX_KEY
ACCESS_TOKEN_EXPIRE_MINUTES=43200
# Integrations
GEMINI_API_KEY=YOUR_GEMINI_API_KEY
FAST2SMS_API_KEY=YOUR_FAST2SMS_KEY
TELEGRAM_BOT_TOKEN=YOUR_TELEGRAM_BOT_TOKEN
TELEGRAM_CHAT_ID=YOUR_TELEGRAM_CHAT_ID
# CORS Allowed Origins
CORS_ORIGINS=["https://app.yourdomain.com", "http://localhost:5173"]
2.4 Deploy Backend
bash
docker compose up -d --build
# Run initial migrations and seed admin user
docker compose exec backend alembic upgrade head
5. Phase 3: 24/7 Unattended Chrome Bot with noVNC

To avoid paying for a Windows VPS, we run Chrome inside an isolated virtual screen using Xvfb and expose a password-protected remote desktop via noVNC on port 6080.

5.1 Install Virtual Desktop & Chrome Packages
bash
apt install -y xvfb x11vnc novnc websockify openbox xterm
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
apt install -y ./google-chrome-stable_current_amd64.deb
rm google-chrome-stable_current_amd64.deb
5.2 Upload Built Extension

Upload your local extension/dist bundle to /opt/leadpharma/bot/extension_dist.

5.3 Configure Systemd Bot Service (/etc/systemd/system/leadpharma-bot.service)
ini
[Unit]
Description=leadpharma.ai 24/7 Headless Chrome IndiaMART Bot
After=network.target docker.service
[Service]
Type=simple
User=root
Environment="DISPLAY=:99"
WorkingDirectory=/opt/leadpharma/bot
ExecStartPre=/usr/bin/pkill -f "Xvfb :99" || true
ExecStart=/bin/bash -c '\
  Xvfb :99 -screen 0 1920x1080x24 & \
  sleep 2; \
  openbox-session & \
  x11vnc -display :99 -forever -nopw -listen 127.0.0.1 -xkb & \
  websockify --web /usr/share/novnc 127.0.0.1:6080 127.0.0.1:5900 & \
  google-chrome \
    --user-data-dir=/opt/leadpharma/bot/chrome_profile \
    --load-extension=/opt/leadpharma/bot/extension_dist \
    --no-first-run \
    --disable-notifications \
    --window-size=1920,1080 \
    https://seller.indiamart.com/bltxn/?pref=recent'
Restart=always
RestartSec=10
[Install]
WantedBy=multi-user.target
bash
systemctl daemon-reload
systemctl enable --now leadpharma-bot.service
6. Phase 4: Nginx Reverse Proxy & SSL Certificates
6.1 Install Nginx & Certbot
bash
apt install -y nginx certbot python3-certbot-nginx apache2-utils
# Create password for the remote bot desktop
htpasswd -c /etc/nginx/.bot_htpasswd admin
6.2 Nginx Configuration (/etc/nginx/sites-available/leadpharma.conf)
nginx
# 1. Backend CRM API
server {
    server_name api.yourdomain.com;
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
# 2. Remote Bot Management Portal (noVNC)
server {
    server_name bot.yourdomain.com;
    auth_basic "leadpharma.ai Bot Secure Access";
    auth_basic_user_file /etc/nginx/.bot_htpasswd;
    location / {
        proxy_pass http://127.0.0.1:6080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
bash
ln -s /etc/nginx/sites-available/leadpharma.conf /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
certbot --nginx -d api.yourdomain.com -d bot.yourdomain.com
7. Phase 5: Web Frontend Hosting (Cloudflare Pages - ₹0)
Build your frontend production bundle:
bash
cd frontend
pnpm run build
Log into Cloudflare Dashboard -> Workers & Pages -> Create application -> Pages.
Upload the frontend/dist folder (or link your GitHub repository).
In Environment Variables, set:
VITE_API_URL=https://api.yourdomain.com
Map custom domain: app.yourdomain.com.
8. Phase 6: Post-Deployment Verification
8.1 Initialize IndiaMART Session
Visit https://bot.yourdomain.com in your browser.
Enter your HTTP basic auth credentials.
You will see Chrome opened on IndiaMART Seller Portal.
Log in using your registered IndiaMART mobile number and OTP.
The session token is saved permanently in /opt/leadpharma/bot/chrome_profile. Even if the server reboots, Chrome launches logged in.
8.2 Verify Real-Time Lead Ingestion
Ensure the leadpharma.ai extension HUD at the bottom right displays Bot Running.
Inspect https://app.yourdomain.com/discovery.
Confirm that matching leads appear in your private lead stream and Telegram alerts fire automatically upon sniper action.
