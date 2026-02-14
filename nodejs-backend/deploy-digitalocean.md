# Deploy Node.js Backend to DigitalOcean Droplet

A complete guide to deploying a Node.js/Express application to a DigitalOcean Droplet with Nginx, PM2, and GitHub Actions CI/CD.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Create DigitalOcean Droplet](#create-digitalocean-droplet)
3. [Initial Server Setup](#initial-server-setup)
4. [Install Node.js and Dependencies](#install-nodejs-and-dependencies)
5. [Configure Nginx](#configure-nginx)
6. [Set Up PM2 Process Manager](#set-up-pm2-process-manager)
7. [Configure SSL with Let's Encrypt](#configure-ssl-with-lets-encrypt)
8. [GitHub Actions CI/CD Pipeline](#github-actions-cicd-pipeline)
9. [Environment Variables](#environment-variables)
10. [Monitoring and Maintenance](#monitoring-and-maintenance)
11. [Troubleshooting](#troubleshooting)

---

## Prerequisites

Before starting, ensure you have:

- A DigitalOcean account ([Sign up here](https://cloud.digitalocean.com/registrations/new))
- A GitHub repository with your Node.js application
- A domain name (optional, but recommended for SSL)
- SSH key pair on your local machine

### Generate SSH Key (if needed)

```bash
# Generate SSH key pair
ssh-keygen -t ed25519 -C "your-email@example.com"

# View your public key
cat ~/.ssh/id_ed25519.pub
```

---

## Create DigitalOcean Droplet

### Step 1: Create Droplet via Control Panel

1. Log in to [DigitalOcean Control Panel](https://cloud.digitalocean.com)
2. Click **Create** → **Droplets**
3. Configure the Droplet:

| Setting | Recommended Value |
|---------|-------------------|
| Region | Choose closest to your users |
| Image | Ubuntu 24.04 LTS |
| Size | Basic → Regular → $6/mo (1GB RAM) or $12/mo (2GB RAM) |
| Authentication | SSH Keys (add your public key) |
| Hostname | `nodejs-app` or your preferred name |

4. Click **Create Droplet**
5. Note your Droplet's **public IP address**

### Step 2: Configure Firewall

1. Go to **Networking** → **Firewalls** → **Create Firewall**
2. Name it `nodejs-firewall`
3. Add Inbound Rules:

| Type | Protocol | Port Range | Sources |
|------|----------|------------|---------|
| SSH | TCP | 22 | All IPv4, All IPv6 |
| HTTP | TCP | 80 | All IPv4, All IPv6 |
| HTTPS | TCP | 443 | All IPv4, All IPv6 |

4. Apply to your Droplet
5. Click **Create Firewall**

---

## Initial Server Setup

### Connect to Your Droplet

```bash
ssh root@YOUR_DROPLET_IP
```

### Create Deploy User

```bash
# Create a new user for deployments
adduser deploy
usermod -aG sudo deploy

# Set up SSH for deploy user
mkdir -p /home/deploy/.ssh
cp ~/.ssh/authorized_keys /home/deploy/.ssh/
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys
```

### Update System

```bash
apt update && apt upgrade -y
```

### Configure UFW Firewall (Additional Layer)

```bash
ufw allow OpenSSH
ufw allow 'Nginx Full'
ufw enable
ufw status
```

---

## Install Node.js and Dependencies

### Install Node.js via NVM (Recommended)

NVM (Node Version Manager) is the preferred way to install Node.js — it allows easy version switching, doesn't require sudo for global packages, and simplifies upgrades.

```bash
# Switch to deploy user
su - deploy

# Install NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# Load NVM (or log out and back in)
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

# Install Node.js LTS
nvm install 20
nvm use 20
nvm alias default 20

# Verify installation
node --version
npm --version
```

### Make Node Available to PM2 and System Services

When using NVM with PM2 and systemd, you need to ensure the correct Node path is used:

```bash
# Create symlinks for system-wide access (run as root)
exit  # Exit back to root
ln -s /home/deploy/.nvm/versions/node/$(ls /home/deploy/.nvm/versions/node/)/bin/node /usr/local/bin/node
ln -s /home/deploy/.nvm/versions/node/$(ls /home/deploy/.nvm/versions/node/)/bin/npm /usr/local/bin/npm
ln -s /home/deploy/.nvm/versions/node/$(ls /home/deploy/.nvm/versions/node/)/bin/npx /usr/local/bin/npx
```

Alternatively, specify the full path in your PM2 ecosystem config (see PM2 section).

### Install PM2 Globally

```bash
# As deploy user
su - deploy
npm install -g pm2

# Create symlink for system access (as root)
exit
ln -s /home/deploy/.nvm/versions/node/$(ls /home/deploy/.nvm/versions/node/)/bin/pm2 /usr/local/bin/pm2
```

### Install Build Essentials (for native modules)

```bash
# As root
apt install -y build-essential
```

### Pin Node Version in Your Project

Create a `.nvmrc` file in your project root to ensure consistent Node versions:

```bash
# In your project directory
echo "20" > .nvmrc
```

Then developers and CI can simply run `nvm use` to switch to the correct version.

---

## Configure Nginx

### Install Nginx

```bash
apt install nginx -y
systemctl start nginx
systemctl enable nginx
systemctl status nginx
```

### Create Application Directory

```bash
mkdir -p /var/www/nodejs-app
chown -R deploy:deploy /var/www/nodejs-app
```

### Create Nginx Configuration

```bash
nano /etc/nginx/sites-available/nodejs-app
```

Add the following configuration:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name YOUR_DOMAIN_OR_IP;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Logging
    access_log /var/log/nginx/nodejs-app.access.log;
    error_log /var/log/nginx/nodejs-app.error.log;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_types text/plain text/css text/xml text/javascript application/javascript application/json application/xml;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 90;
    }

    # Health check endpoint
    location /health {
        proxy_pass http://localhost:3000/health;
        access_log off;
    }
}
```

### Enable the Site

```bash
# Create symlink
ln -s /etc/nginx/sites-available/nodejs-app /etc/nginx/sites-enabled/

# Remove default site (optional)
rm /etc/nginx/sites-enabled/default

# Test configuration
nginx -t

# Reload Nginx
systemctl reload nginx
```

---

## Set Up PM2 Process Manager

### Create PM2 Ecosystem File

Create this file in your project repository as `ecosystem.config.cjs`:

```javascript
module.exports = {
  apps: [
    {
      name: 'nodejs-app',
      script: './dist/index.js',  // Adjust to your entry point
      
      // NVM compatibility: specify the Node interpreter path
      // Update the version path if using a different Node version
      interpreter: '/home/deploy/.nvm/versions/node/v20.18.0/bin/node',
      
      instances: 'max',           // Use all CPU cores
      exec_mode: 'cluster',
      autorestart: true,
      watch: false,
      max_memory_restart: '500M',
      env: {
        NODE_ENV: 'production',
        PORT: 3000,
        // Ensure NVM paths are available
        PATH: '/home/deploy/.nvm/versions/node/v20.18.0/bin:' + process.env.PATH
      },
      env_production: {
        NODE_ENV: 'production',
        PORT: 3000
      },
      // Logging
      error_file: '/var/log/pm2/nodejs-app-error.log',
      out_file: '/var/log/pm2/nodejs-app-out.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
      // Graceful shutdown
      kill_timeout: 5000,
      wait_ready: true,
      listen_timeout: 10000
    }
  ]
};
```

### Create Log Directory

```bash
mkdir -p /var/log/pm2
chown -R deploy:deploy /var/log/pm2
```

### PM2 Startup Configuration

```bash
# Generate startup script (run as deploy user)
su - deploy
pm2 startup systemd -u deploy --hp /home/deploy

# After running the app, save the process list
pm2 save
```

---

## Configure SSL with Let's Encrypt

### Install Certbot

```bash
apt install certbot python3-certbot-nginx -y
```

### Obtain SSL Certificate

```bash
certbot --nginx -d YOUR_DOMAIN.com -d www.YOUR_DOMAIN.com
```

### Auto-Renewal

Certbot automatically creates a cron job. Verify with:

```bash
certbot renew --dry-run
```

---

## GitHub Actions CI/CD Pipeline

### Step 1: Create SSH Deploy Key

On your local machine:

```bash
# Generate a dedicated deploy key
ssh-keygen -t ed25519 -f ~/.ssh/do_deploy_key -C "github-actions-deploy"

# View private key (add to GitHub Secrets)
cat ~/.ssh/do_deploy_key

# View public key (add to Droplet)
cat ~/.ssh/do_deploy_key.pub
```

### Step 2: Add Public Key to Droplet

```bash
# On the Droplet, as deploy user
su - deploy
echo "YOUR_PUBLIC_KEY_CONTENT" >> ~/.ssh/authorized_keys
```

### Step 3: Configure GitHub Secrets

Go to your repository → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

Add these secrets:

| Secret Name | Value |
|-------------|-------|
| `DO_HOST` | Your Droplet's public IP address |
| `DO_USERNAME` | `deploy` |
| `DO_SSH_KEY` | Contents of `~/.ssh/do_deploy_key` (private key) |
| `DO_SSH_PORT` | `22` |

### Step 4: Create GitHub Actions Workflow

Create `.github/workflows/deploy.yml` in your repository:

```yaml
name: Deploy to DigitalOcean

on:
  push:
    branches: [main]
  workflow_dispatch:  # Allow manual trigger

env:
  NODE_VERSION: '20.x'

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint --if-present

      - name: Run tests
        run: npm test --if-present

  build:
    name: Build Application
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build TypeScript
        run: npm run build

      - name: Create deployment package
        run: |
          mkdir -p deploy
          cp -r dist deploy/
          cp package*.json deploy/
          cp ecosystem.config.cjs deploy/
          [ -f .env.example ] && cp .env.example deploy/ || true
          tar -czf deploy.tar.gz -C deploy .

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: deploy-package
          path: deploy.tar.gz
          retention-days: 1

  deploy:
    name: Deploy to DigitalOcean
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: deploy-package

      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.DO_SSH_KEY }}" > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key
          ssh-keyscan -H ${{ secrets.DO_HOST }} >> ~/.ssh/known_hosts

      - name: Copy files to server
        run: |
          scp -i ~/.ssh/deploy_key -P ${{ secrets.DO_SSH_PORT }} \
            deploy.tar.gz \
            ${{ secrets.DO_USERNAME }}@${{ secrets.DO_HOST }}:/var/www/nodejs-app/

      - name: Deploy application
        run: |
          ssh -i ~/.ssh/deploy_key -p ${{ secrets.DO_SSH_PORT }} \
            ${{ secrets.DO_USERNAME }}@${{ secrets.DO_HOST }} << 'DEPLOY_SCRIPT'
          
          set -e
          
          # Load NVM
          export NVM_DIR="$HOME/.nvm"
          [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
          
          cd /var/www/nodejs-app
          
          # Backup current deployment
          if [ -d "current" ]; then
            rm -rf previous
            mv current previous
          fi
          
          # Extract new deployment
          mkdir -p current
          tar -xzf deploy.tar.gz -C current
          rm deploy.tar.gz
          
          # Install production dependencies
          cd current
          npm ci --only=production
          
          # Reload application with zero-downtime
          pm2 reload ecosystem.config.cjs --env production || pm2 start ecosystem.config.cjs --env production
          
          # Save PM2 process list
          pm2 save
          
          # Cleanup old logs (keep last 7 days)
          find /var/log/pm2 -name "*.log" -mtime +7 -delete 2>/dev/null || true
          
          echo "Deployment completed successfully!"
          DEPLOY_SCRIPT

      - name: Health check
        run: |
          sleep 10
          curl -f http://${{ secrets.DO_HOST }}/health || exit 1

      - name: Rollback on failure
        if: failure()
        run: |
          ssh -i ~/.ssh/deploy_key -p ${{ secrets.DO_SSH_PORT }} \
            ${{ secrets.DO_USERNAME }}@${{ secrets.DO_HOST }} << 'ROLLBACK_SCRIPT'
          
          # Load NVM
          export NVM_DIR="$HOME/.nvm"
          [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
          
          cd /var/www/nodejs-app
          if [ -d "previous" ]; then
            rm -rf current
            mv previous current
            cd current
            pm2 reload ecosystem.config.cjs --env production
            echo "Rollback completed!"
          fi
          ROLLBACK_SCRIPT
```

---

## Environment Variables

### On the Server

Create environment file:

```bash
nano /var/www/nodejs-app/.env
```

Add your environment variables:

```env
NODE_ENV=production
PORT=3000
DATABASE_URL=your_database_connection_string
JWT_SECRET=your_jwt_secret
# Add other environment variables as needed
```

Set permissions:

```bash
chmod 600 /var/www/nodejs-app/.env
chown deploy:deploy /var/www/nodejs-app/.env
```

### Update PM2 Ecosystem to Load .env

Modify `ecosystem.config.cjs`:

```javascript
module.exports = {
  apps: [
    {
      name: 'nodejs-app',
      script: './dist/index.js',
      cwd: '/var/www/nodejs-app/current',
      // Load environment from file
      env_file: '/var/www/nodejs-app/.env',
      // ... rest of config
    }
  ]
};
```

---

## Monitoring and Maintenance

### PM2 Monitoring Commands

```bash
# View running processes
pm2 list

# View logs
pm2 logs nodejs-app

# View detailed status
pm2 show nodejs-app

# Monitor resources
pm2 monit

# View metrics
pm2 status
```

### PM2 Web Dashboard (Optional)

```bash
# Install PM2 Plus for web monitoring
pm2 link <your-secret> <your-public-key>
```

### Nginx Log Monitoring

```bash
# Access logs
tail -f /var/log/nginx/nodejs-app.access.log

# Error logs
tail -f /var/log/nginx/nodejs-app.error.log
```

### System Monitoring

```bash
# Install htop for resource monitoring
apt install htop -y
htop

# Check disk usage
df -h

# Check memory usage
free -m
```

---

## Troubleshooting

### Common Issues and Solutions

#### Application Not Starting

```bash
# Check PM2 logs
pm2 logs nodejs-app --lines 100

# Check if port is in use
lsof -i :3000

# Manually test the app
cd /var/www/nodejs-app/current
node dist/index.js
```

#### Nginx 502 Bad Gateway

```bash
# Check if Node.js app is running
pm2 status

# Check Nginx error logs
tail -f /var/log/nginx/error.log

# Verify proxy pass port matches app port
cat /etc/nginx/sites-available/nodejs-app | grep proxy_pass
```

#### Permission Denied Errors

```bash
# Fix ownership
chown -R deploy:deploy /var/www/nodejs-app

# Fix permissions
chmod -R 755 /var/www/nodejs-app
```

#### SSH Connection Issues

```bash
# On local machine, test SSH connection
ssh -vvv deploy@YOUR_DROPLET_IP

# Check SSH service on server
systemctl status ssh
```

#### Memory Issues

```bash
# Check memory usage
free -m

# Add swap space if needed
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

### Useful Commands Reference

| Task | Command |
|------|---------|
| Restart app | `pm2 restart nodejs-app` |
| Reload app (zero-downtime) | `pm2 reload nodejs-app` |
| Stop app | `pm2 stop nodejs-app` |
| View logs | `pm2 logs nodejs-app` |
| Clear logs | `pm2 flush` |
| Restart Nginx | `systemctl restart nginx` |
| Test Nginx config | `nginx -t` |
| Check ports | `netstat -tlnp` |
| Check disk space | `df -h` |
| Check memory | `free -m` |

---

## Quick Start Checklist

- [ ] Create DigitalOcean Droplet (Ubuntu 24.04)
- [ ] Configure Firewall (ports 22, 80, 443)
- [ ] Create `deploy` user with SSH access
- [ ] Install NVM and Node.js 20.x
- [ ] Create `.nvmrc` file in your project
- [ ] Install and configure Nginx
- [ ] Install PM2 globally via NVM
- [ ] Create application directory `/var/www/nodejs-app`
- [ ] Add `ecosystem.config.cjs` to your repository (with correct interpreter path)
- [ ] Create `.github/workflows/deploy.yml`
- [ ] Configure GitHub Secrets
- [ ] Push to main branch to trigger deployment
- [ ] Configure SSL with Let's Encrypt (optional)
- [ ] Set up monitoring

---

## Security Best Practices

1. **Keep system updated**: Run `apt update && apt upgrade -y` regularly
2. **Use SSH keys only**: Disable password authentication
3. **Configure firewall**: Only allow necessary ports
4. **Use non-root user**: Deploy with a regular user, not root
5. **Secure environment variables**: Never commit secrets to Git
6. **Enable fail2ban**: Protect against brute-force attacks
7. **Regular backups**: Use DigitalOcean Backups or Snapshots
8. **Monitor logs**: Set up log rotation and monitoring

```bash
# Install and configure fail2ban
apt install fail2ban -y
systemctl enable fail2ban
systemctl start fail2ban
```

---

*Last updated: February 2026*
