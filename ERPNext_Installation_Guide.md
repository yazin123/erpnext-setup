
# ERPNext Installation Guide

This README provides step-by-step instructions for setting up ERPNext on an Ubuntu server with Nginx, MariaDB, and all necessary dependencies.

## Prerequisites

- Ubuntu Server (22.04 LTS or 24.04 LTS)
- Minimum t2.medium instance (2 vCPU, 4GB RAM) or equivalent
- A domain name with DNS access
- SSH access with sudo privileges

## Installation Steps

### 1. System Setup

Update your system and install required dependencies:
```bash
sudo apt update
sudo apt upgrade -y
sudo apt install git python3-dev python3-pip redis-server libmysqlclient-dev mariadb-server mariadb-client python3-setuptools python3-venv software-properties-common fontconfig libfontconfig1 libxrender1 libxext6 xvfb
```

### 2. Configure MariaDB

Secure your MariaDB installation:
```bash
sudo mysql_secure_installation
```

Add UTF-8 support for MariaDB by adding the following to `/etc/mysql/my.cnf`:
```
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[mysql]
default-character-set = utf8mb4
```

Restart MariaDB:
```bash
sudo systemctl restart mariadb
```

### 3. Install Node.js and Yarn

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 18
npm install -g yarn
```

### 4. Install wkhtmltopdf

```bash
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-2/wkhtmltox_0.12.6.1-2.jammy_amd64.deb
sudo dpkg -i wkhtmltox_0.12.6.1-2.jammy_amd64.deb
sudo apt-get install -f
```

### 5. Set Up Python Environment

```bash
python3 -m venv frappe-env
source frappe-env/bin/activate
```

### 6. Install Bench

```bash
pip install frappe-bench
```

### 7. Initialize Bench and Install ERPNext

```bash
# Initialize a new bench
bench init frappe-bench --frappe-branch version-15
cd frappe-bench

# Create a new site
bench new-site yourdomain.com

# Get and install ERPNext
bench get-app erpnext --branch version-15
bench --site yourdomain.com install-app erpnext
```

### 8. Set Up Production

Install Nginx and Supervisor:
```bash
sudo apt install nginx supervisor
```

Configure the production environment:
```bash
# Setup Nginx
bench setup nginx
sudo ln -sf `pwd`/config/nginx.conf /etc/nginx/conf.d/frappe-bench.conf

# Fix Nginx configuration if needed
sudo nano /etc/nginx/nginx.conf
# Add this inside the http block:
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';
                
# Disable default site
sudo rm -f /etc/nginx/sites-enabled/default

# Setup supervisor
bench setup supervisor
sudo ln -sf `pwd`/config/supervisor.conf /etc/supervisor/conf.d/frappe-bench.conf

# Fix permissions
sudo usermod -aG ubuntu www-data
sudo chown -R ubuntu:www-data ~/frappe-bench
sudo chmod -R 755 ~/frappe-bench/sites/assets
sudo find ~/frappe-bench/sites -type d -exec chmod 755 {} \;
sudo find ~/frappe-bench/sites -type f -exec chmod 644 {} \;

# Start the services
sudo supervisorctl update
sudo supervisorctl start all
sudo systemctl restart nginx
```

### 9. DNS Configuration

Configure DNS with an A record pointing your domain to your server's IP address.

### 10. SSL Setup

Install SSL with Let's Encrypt:
```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com
```

## Troubleshooting

### Permission Issues

If you encounter "Permission denied" errors in Nginx logs:
```bash
sudo chown -R ubuntu:www-data ~/frappe-bench
sudo chmod -R 755 ~/frappe-bench/sites/assets
sudo find ~/frappe-bench/sites -type d -exec chmod 755 {} \;
sudo find ~/frappe-bench/sites -type f -exec chmod 644 {} \;
```

### Nginx Configuration Issues

If Nginx shows a default page instead of ERPNext:
```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo systemctl restart nginx
```

### Check Service Status

```bash
sudo supervisorctl status
sudo systemctl status nginx
```

### View Logs

```bash
sudo tail -f /var/log/nginx/error.log
tail -f ~/frappe-bench/logs/web.error.log
```

## Maintenance

### Update Bench and Apps

```bash
cd ~/frappe-bench
bench update
```

### Backup Your Site

```bash
bench --site yourdomain.com backup
```

## Security Tips

- Always use strong passwords
- Keep your server updated
- Configure a firewall (UFW)
- Consider using Cloudflare for additional protection
