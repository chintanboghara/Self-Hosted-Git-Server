# Self-Hosted Git Server on Ubuntu using Gitea

Deploy a self-hosted Git server using Gitea on an Ubuntu machine, enabling private Git repository hosting, user management, and CI/CD automation.

## Step-by-Step Implementation

### 1. Install Ubuntu & Update System
Ensure your Ubuntu system is up to date:
```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Required Packages
Install necessary dependencies:
```bash
sudo apt install git mariadb-server nginx certbot python3-certbot-nginx -y
```

### 3. Secure MariaDB
Run the MariaDB secure installation script:
```bash
sudo mysql_secure_installation
```
- Set a strong root password
- Remove anonymous users
- Disable remote root login

### 4. Create Gitea Database & User
Access MySQL:
```bash
sudo mysql -u root -p
```
Run the following SQL commands:
```sql
CREATE DATABASE gitea CHARACTER SET 'utf8mb4' COLLATE 'utf8mb4_unicode_ci';
CREATE USER 'gitea'@'localhost' IDENTIFIED BY 'GitServer@123';
GRANT ALL PRIVILEGES ON gitea.* TO 'gitea'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### 5. Install & Configure Gitea
#### Download & Install Gitea
```bash
sudo mkdir -p /var/lib/gitea
sudo useradd --system --home /var/lib/gitea --shell /bin/bash gitea
sudo wget -O /usr/local/bin/gitea https://dl.gitea.com/gitea/1.23.4/gitea-1.23.4-linux-amd64
sudo chmod +x /usr/local/bin/gitea
```

#### Create Configuration & Data Directories
```bash
sudo mkdir -p /etc/gitea /var/lib/gitea/{custom,data,log}
sudo chown -R gitea:gitea /var/lib/gitea /etc/gitea
sudo chmod -R 750 /var/lib/gitea /etc/gitea
sudo touch /etc/gitea/app.ini
sudo chmod 640 /etc/gitea/app.ini
```

### 6. Configure `app.ini`
Open the Gitea configuration file:
```bash
sudo vi /etc/gitea/app.ini
```
Update the following sections:

#### Repository Paths
```
[repository]
ROOT = /var/lib/gitea/data/gitea-repositories
```

#### Server Configuration
```
[server]
APP_DATA_PATH = /var/lib/gitea/data
DOMAIN = git.chintanboghara.dev
SSH_DOMAIN = git.chintanboghara.dev
HTTP_PORT = 3000
ROOT_URL = https://git.chintanboghara.dev/
LFS_CONTENT_PATH = /var/lib/gitea/data/fs
```

#### Log Configuration
```
[log]
ROOT_PATH = /var/lib/gitea/log
```

Fix permissions:
```bash
sudo chown -R gitea:gitea /etc/gitea
sudo chmod 640 /etc/gitea/app.ini
```

### 7. Configure Systemd Service for Gitea
Create a systemd service file:
```bash
sudo vi /etc/systemd/system/gitea.service
```
Add the following content:
```
[Unit]
Description=Gitea Self-Hosted Git Server
After=network.target mariadb.service

[Service]
User=gitea
Group=gitea
ExecStart=/usr/local/bin/gitea web --config /etc/gitea/app.ini
Restart=always
WorkingDirectory=/var/lib/gitea
Environment=USER=gitea HOME=/var/lib/gitea

[Install]
WantedBy=multi-user.target
```
Reload Systemd and start Gitea:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now gitea
sudo systemctl status gitea
```

### 8. Set Up Reverse Proxy with Nginx
Create an Nginx configuration file:
```bash
sudo vi /etc/nginx/sites-available/gitea
```
Add the following content:
```
server {
    listen 80;
    server_name git.chintanboghara.dev;
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
    }
}
```
Enable and restart Nginx:
```bash
sudo ln -s /etc/nginx/sites-available/gitea /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

### 9. Secure Gitea with SSL (Let's Encrypt)
Issue an SSL certificate using Certbot:
```bash
sudo certbot --nginx -d git.chintanboghara.dev
```
Set up auto-renewal:
```bash
sudo crontab -e
```
Add the following line:
```
0 3 * * * certbot renew --quiet
```

### 10. Enable SSH Access for Git Repositories
```bash
sudo usermod -aG gitea $(whoami)
sudo systemctl restart gitea
```
Clone a repository:
```bash
git clone git@git.chintanboghara.dev:your-username/your-repo.git
```

### 11. Access Gitea Web Interface
- Open `https://git.chintanboghara.dev`
- Complete the initial setup
- Use the Gitea database credentials
- Create an admin user
