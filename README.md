# Canvas LMS Complete Installation Guide for luxdevhq.ai

## Prerequisites Checklist

Before starting, ensure you have:

- [ ] Ubuntu 22.04 LTS server (fresh installation recommended)
- [ ] Minimum 8GB RAM (16GB+ recommended for production)
- [ ] 50GB+ free disk space
- [ ] Root or sudo access to the server
- [ ] Domain name: luxdevhq.ai pointing to your server IP
- [ ] Subdomain: files.luxdevhq.ai pointing to your server IP
- [ ] SMTP email credentials (Gmail, AWS SES, or other)

---

## Phase 1: Server Preparation

### Step 1: Connect to Your Server

```bash
# From your local computer, connect via SSH
ssh root@your_server_ip

# Or if using a non-root user with sudo
ssh your_username@your_server_ip
```

### Step 2: Update System Packages

```bash
# Update package lists
sudo apt-get update

# Upgrade existing packages
sudo apt-get upgrade -y

# This may take 5-10 minutes
```

### Step 3: Set Server Timezone (Optional but Recommended)

```bash
# Set timezone to your location
sudo timedatectl set-timezone Africa/Nairobi

# Verify
date
```

---

## Phase 2: Install Core Dependencies

### Step 4: Add Ruby Repository

```bash
# Install software properties
sudo apt-get install -y software-properties-common

# Add Instructure's Ruby PPA
sudo add-apt-repository -y ppa:instructure/ruby

# Update package lists
sudo apt-get update
```

### Step 5: Install Ruby 3.4 and Build Tools

```bash
# Install Ruby and essential libraries
sudo apt-get install -y ruby3.4 ruby3.4-dev zlib1g-dev libxml2-dev \
                       libsqlite3-dev postgresql libpq-dev \
                       libxmlsec1-dev libyaml-dev libidn11-dev \
                       curl make g++ git-core

# Verify Ruby installation
ruby -v
# Should show: ruby 3.4.x
```

### Step 6: Install Node.js 20

```bash
# Download and run Node.js setup script
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -

# Install Node.js
sudo apt-get install -y nodejs

# Update npm to latest version
sudo npm install -g npm@latest

# Verify installations
node -v
# Should show: v20.x.x

npm -v
# Should show: 10.x.x or higher
```

### Step 7: Install Yarn Package Manager

```bash
# Install Yarn globally
sudo npm install -g yarn

# Verify installation
yarn -v
# Should show: 1.x.x or higher
```

---

## Phase 3: Database Setup (PostgreSQL)

### Step 8: Install PostgreSQL 14

```bash
# Install PostgreSQL
sudo apt-get install -y postgresql-14

# Verify installation
sudo systemctl status postgresql
# Should show: active (running)
```

### Step 9: Create Canvas Database User

```bash
# Create database user (you'll be prompted for a password)
sudo -u postgres createuser canvas --no-createdb --no-superuser --no-createrole --pwprompt

# When prompted, enter a strong password and remember it!
# Example: CanvasDB2024SecurePass!
```

### Step 10: Create Canvas Database

```bash
# Create the production database
sudo -u postgres createdb canvas_production --owner=canvas

# Verify database was created
sudo -u postgres psql -c "\l" | grep canvas_production
```

---

## Phase 4: Install Redis Cache

### Step 11: Add Redis Repository

```bash
# Add Redis PPA
sudo add-apt-repository -y ppa:redislabs/redis

# Update package lists
sudo apt-get update
```

### Step 12: Install and Start Redis

```bash
# Install Redis server
sudo apt-get install -y redis-server

# Start Redis
sudo systemctl start redis-server

# Enable Redis to start on boot
sudo systemctl enable redis-server

# Test Redis connection
redis-cli ping
# Should respond: PONG
```

---

## Phase 5: Install Web Server (Apache + Passenger)

### Step 13: Install Apache

```bash
# Install Apache and dependencies
sudo apt-get install -y apache2 dirmngr gnupg apt-transport-https ca-certificates

# Verify Apache is running
sudo systemctl status apache2
# Should show: active (running)
```

### Step 14: Add Passenger Repository

```bash
# Add Passenger GPG key
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 561F9B9CAC40B2F7

# Add Passenger repository
sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger $(lsb_release -cs) main > /etc/apt/sources.list.d/passenger.list'

# Update package lists
sudo apt-get update
```

### Step 15: Install Passenger and XSendFile

```bash
# Install Passenger module for Apache
sudo apt-get install -y libapache2-mod-passenger

# Install XSendFile for optimized file downloads
sudo apt-get install -y libapache2-mod-xsendfile

# Enable Apache modules
sudo a2enmod rewrite
sudo a2enmod passenger
sudo a2enmod ssl
sudo a2enmod headers

# Verify Passenger is loaded
sudo apachectl -M | grep passenger
# Should show: passenger_module (shared)
```

---

## Phase 6: Download Canvas LMS

### Step 16: Clone Canvas Repository

```bash
# Create Canvas directory
sudo mkdir -p /var/canvas

# Change ownership to your current user temporarily
sudo chown -R $USER:$USER /var/canvas

# Clone Canvas from GitHub
git clone https://github.com/instructure/canvas-lms.git /var/canvas

# This will take 5-10 minutes depending on internet speed
```

### Step 17: Switch to Production Branch

```bash
# Navigate to Canvas directory
cd /var/canvas

# Checkout the production-stable branch
git checkout prod

# Verify current branch
git branch
# Should show: * prod
```

---

## Phase 7: Create Canvas System User

### Step 18: Create Dedicated Canvas User

```bash
# Create user without login capability
sudo adduser --disabled-password --gecos canvas canvasuser

# This user will run Canvas processes for security
```

---

## Phase 8: Install Ruby Dependencies

### Step 19: Install Bundler

```bash
# Install Bundler gem (without documentation to save time)
sudo gem install bundler --no-document

# Verify installation
bundler -v
# Should show: Bundler version 2.x.x
```

### Step 20: Install Canvas Ruby Gems

```bash
# Navigate to Canvas directory
cd /var/canvas

# Configure bundler to install gems locally
bundle config set --local path vendor/bundle

# Install all Ruby dependencies (this takes 15-30 minutes)
bundle install

# You'll see many gems being installed...
```

---

## Phase 9: Install Node.js Dependencies

### Step 21: Install Canvas Node Modules

```bash
# Still in /var/canvas directory
cd /var/canvas

# Install Node.js dependencies (this takes 10-20 minutes)
yarn install

# You'll see a progress bar...
```

---

## Phase 10: Configure Canvas

### Step 22: Copy Configuration Templates

```bash
cd /var/canvas

# Copy all example config files
for config in amazon_s3 database vault_contents delayed_jobs domain file_store outgoing_mail security external_migration dynamic_settings cache_store redis; do
  sudo cp config/$config.yml.example config/$config.yml
done
```

### Step 23: Configure Database Connection

```bash
# Edit database configuration
sudo nano config/database.yml
```

**Find the `production:` section and update it to:**

```yaml
production:
  adapter: postgresql
  encoding: utf8
  database: canvas_production
  host: localhost
  username: canvas
  password: YOUR_DATABASE_PASSWORD_FROM_STEP_9
  timeout: 5000
```

**Save and exit:** Press `Ctrl+X`, then `Y`, then `Enter`

### Step 24: Configure Domain Settings

```bash
# Edit domain configuration
sudo nano config/domain.yml
```

**Update the production section:**

```yaml
production:
  domain: "luxdevhq.ai"
  files_domain: "files.luxdevhq.ai"
  ssl: true
```

**Save and exit:** `Ctrl+X`, `Y`, `Enter`

### Step 25: Configure Email (SMTP)

```bash
# Edit outgoing mail configuration
sudo nano config/outgoing_mail.yml
```

**For Gmail (using App Password):**

```yaml
production:
  address: "smtp.gmail.com"
  port: 587
  user_name: "your-email@gmail.com"
  password: "your-app-password"
  authentication: "plain"
  domain: "luxdevhq.ai"
  outgoing_address: "canvas@luxdevhq.ai"
  enable_starttls_auto: true
```

**For AWS SES:**

```yaml
production:
  address: "email-smtp.us-east-1.amazonaws.com"
  port: 587
  user_name: "YOUR_SMTP_USERNAME"
  password: "YOUR_SMTP_PASSWORD"
  authentication: "plain"
  domain: "luxdevhq.ai"
  outgoing_address: "canvas@luxdevhq.ai"
  enable_starttls_auto: true
```

**Save and exit:** `Ctrl+X`, `Y`, `Enter`

### Step 26: Configure Security Keys

```bash
# Edit security configuration
sudo nano config/security.yml
```

**Generate random keys and update:**

```yaml
production:
  encryption_key: "PASTE_RANDOM_STRING_HERE"
  signing_secret: "PASTE_RANDOM_STRING_HERE"
  lti_iss: "https://luxdevhq.ai"
```

**To generate random strings, open another terminal and run:**

```bash
# Generate encryption key
openssl rand -hex 32

# Generate signing secret
openssl rand -hex 32

# Copy each output and paste into security.yml
```

**Save and exit:** `Ctrl+X`, `Y`, `Enter`

### Step 27: Configure Cache (Redis)

```bash
# Edit cache configuration
sudo nano config/cache_store.yml
```

**Update all environments:**

```yaml
test:
  cache_store: redis_cache_store
development:
  cache_store: redis_cache_store
production:
  cache_store: redis_cache_store
```

**Save and exit:** `Ctrl+X`, `Y`, `Enter`

### Step 28: Configure Redis Connection

```bash
# Edit Redis configuration
sudo nano config/redis.yml
```

**Update the production section:**

```yaml
production:
  url:
    - redis://localhost:6379/0
```

**Save and exit:** `Ctrl+X`, `Y`, `Enter`

---

## Phase 11: Set Up Directories and Permissions

### Step 29: Create Required Directories

```bash
cd /var/canvas

# Create necessary directories
sudo mkdir -p log tmp/pids public/assets app/stylesheets/brandable_css_brands

# Create required files
sudo touch app/stylesheets/_brandable_variables_defaults_autogenerated.scss
sudo touch Gemfile.lock
sudo touch log/production.log
```

### Step 30: Set Ownership and Permissions

```bash
# Set ownership of Canvas directory to canvasuser
sudo chown -R canvasuser:canvasuser /var/canvas

# Secure configuration files
sudo chmod 400 config/*.yml
```

---

## Phase 12: Initialize Database

### Step 31: Generate Asset Manifests

```bash
cd /var/canvas

# Generate initial asset files
sudo -u canvasuser bash -c "cd /var/canvas && yarn gulp rev"

# This takes 2-5 minutes
```

### Step 32: Initialize Canvas Database

```bash
cd /var/canvas

# Set up environment variables for initial setup
export CANVAS_LMS_ADMIN_EMAIL="admin@luxdevhq.ai"
export CANVAS_LMS_ADMIN_PASSWORD="TempPassword123!"
export CANVAS_LMS_ACCOUNT_NAME="LuxDev HQ"
export CANVAS_LMS_STATS_COLLECTION="opt_out"

# Run database setup (this takes 5-10 minutes)
sudo -u canvasuser bash -c "cd /var/canvas && RAILS_ENV=production bundle exec rake db:initial_setup"

# You'll see many migration messages...
```

**Important:** Change `TempPassword123!` to a strong password you'll remember!

---

## Phase 13: Compile Assets

### Step 33: Compile Canvas Assets

```bash
cd /var/canvas

# Compile all CSS, JavaScript, and other assets
# WARNING: This takes 20-45 minutes!
sudo -u canvasuser bash -c "cd /var/canvas && RAILS_ENV=production bundle exec rake canvas:compile_assets"

# You'll see a progress bar...
# Perfect time for a coffee break! ☕
```

### Step 34: Set Asset Permissions

```bash
# Set proper ownership for compiled assets
sudo chown -R canvasuser:canvasuser /var/canvas/public/dist/brandable_css
```

---

## Phase 14: Configure Apache Web Server

### Step 35: Disable Default Apache Site

```bash
# Disable the default Apache site
sudo a2dissite 000-default.conf
```

### Step 36: Create Canvas Apache Configuration

```bash
# Create new Apache configuration file
sudo nano /etc/apache2/sites-available/canvas.conf
```

**Paste this configuration:**

```apache
<VirtualHost *:80>
  ServerName luxdevhq.ai
  ServerAlias files.luxdevhq.ai
  ServerAdmin admin@luxdevhq.ai
  DocumentRoot /var/canvas/public
  
  RewriteEngine On
  RewriteCond %{HTTP:X-Forwarded-Proto} !=https
  RewriteCond %{REQUEST_URI} !^/health_check
  RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI} [L]
  
  ErrorLog /var/log/apache2/canvas_errors.log
  LogLevel warn
  CustomLog /var/log/apache2/canvas_access.log combined
  
  SetEnv RAILS_ENV production
  
  <Directory /var/canvas/public>
    Options All
    AllowOverride All
    Require all granted
  </Directory>
</VirtualHost>

<VirtualHost *:443>
  ServerName luxdevhq.ai
  ServerAlias files.luxdevhq.ai
  ServerAdmin admin@luxdevhq.ai
  DocumentRoot /var/canvas/public
  
  ErrorLog /var/log/apache2/canvas_errors.log
  LogLevel warn
  CustomLog /var/log/apache2/canvas_ssl_access.log combined
  
  SSLEngine on
  SSLCertificateFile /etc/ssl/certs/ssl-cert-snakeoil.pem
  SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key
  
  # X-Sendfile for optimized file downloads
  XSendFile On
  XSendFilePath /var/canvas
  
  SetEnv RAILS_ENV production
  PassengerDefaultUser canvasuser
  
  <Directory /var/canvas/public>
    Options All
    AllowOverride All
    Require all granted
  </Directory>
</VirtualHost>
```

**Save and exit:** `Ctrl+X`, `Y`, `Enter`

### Step 37: Enable Canvas Site

```bash
# Enable the Canvas Apache site
sudo a2ensite canvas.conf

# Test Apache configuration
sudo apache2ctl configtest
# Should show: Syntax OK
```

---

## Phase 15: Set Up Background Jobs

### Step 38: Configure Automated Jobs Daemon

```bash
# Create symbolic link for jobs daemon
sudo ln -sf /var/canvas/script/canvas_init /etc/init.d/canvas_init

# Configure to start on boot
sudo update-rc.d canvas_init defaults
```

### Step 39: Start Background Jobs

```bash
# Start the Canvas jobs daemon
sudo /etc/init.d/canvas_init start

# Verify it's running
sudo /etc/init.d/canvas_init status
# Should show: Running
```

---

## Phase 16: Final Configuration and Launch

### Step 40: Restart All Services

```bash
# Restart Apache
sudo systemctl restart apache2

# Restart Redis
sudo systemctl restart redis-server

# Check all services are running
sudo systemctl status apache2
sudo systemctl status postgresql
sudo systemctl status redis-server
```

### Step 41: Configure Firewall

```bash
# Allow SSH (important - don't lock yourself out!)
sudo ufw allow 22/tcp

# Allow HTTP
sudo ufw allow 80/tcp

# Allow HTTPS
sudo ufw allow 443/tcp

# Enable firewall
sudo ufw enable

# Check firewall status
sudo ufw status
```

---

## Phase 17: DNS Configuration

### Step 42: Set Up DNS Records

**In your domain registrar (Namecheap, GoDaddy, Cloudflare, etc.):**

Add these A records:

```
Type: A
Name: @
Value: YOUR_SERVER_IP
TTL: 300

Type: A
Name: files
Value: YOUR_SERVER_IP
TTL: 300
```

**Verify DNS propagation:**

```bash
# From your local computer
dig luxdevhq.ai +short
dig files.luxdevhq.ai +short

# Both should return your server IP
```

---

## Phase 18: Install SSL Certificate (Critical!)

### Step 43: Install Certbot

```bash
# Install Certbot for Let's Encrypt
sudo apt-get install -y certbot python3-certbot-apache
```

### Step 44: Obtain SSL Certificate

```bash
# Get SSL certificate for both domains
sudo certbot --apache -d luxdevhq.ai -d files.luxdevhq.ai

# Follow the prompts:
# 1. Enter email address
# 2. Agree to terms (Y)
# 3. Share email? (N)
# 4. Choose redirect option (2 - Redirect HTTP to HTTPS)
```

### Step 45: Test Auto-Renewal

```bash
# Test certificate renewal process
sudo certbot renew --dry-run

# Should show: Congratulations, all renewals succeeded
```

---

## Phase 19: First Login and Setup

### Step 46: Access Canvas

Open your browser and go to: **https://luxdevhq.ai**

### Step 47: Login

```
Email: admin@luxdevhq.ai
Password: TempPassword123! (or whatever you set in Step 32)
```

### Step 48: Change Admin Password Immediately

1. Click on **Account** (top right)
2. Click **Settings**
3. Scroll to **Login Settings**
4. Click **Change Password**
5. Enter old password and new secure password
6. Click **Update Settings**

---

## Phase 20: Post-Installation Configuration

### Step 49: Configure Account Settings

1. Go to **Admin** → **Account Settings**
2. Update these settings:
   - **Account Name:** LuxDev HQ
   - **Time Zone:** Africa/Nairobi
   - **Language:** English
   - **Default Storage:** Local (for now)

### Step 50: Set Up Notifications

1. Go to **Admin** → **Notifications**
2. Click **Notifications Settings**
3. Configure email notifications as needed

### Step 51: Create Your First Course

1. Click **Courses** in the left menu
2. Click **+ Course**
3. Fill in course details:
   - Course Name
   - Course Code
   - Term
   - Start/End dates
4. Click **Add Course**

---

## Verification Checklist

### Test These Features:

- [ ] Login works with admin credentials
- [ ] Can create a new course
- [ ] Can upload files
- [ ] Can create assignments
- [ ] Email notifications work (send test email)
- [ ] Can invite users
- [ ] Files download correctly
- [ ] HTTPS works (green padlock in browser)

---

## Maintenance Commands

### Check Service Status

```bash
# Check Apache
sudo systemctl status apache2

# Check PostgreSQL
sudo systemctl status postgresql

# Check Redis
sudo systemctl status redis-server

# Check Canvas jobs
sudo /etc/init.d/canvas_init status
```

### View Logs

```bash
# Canvas application log
sudo tail -f /var/canvas/log/production.log

# Apache error log
sudo tail -f /var/log/apache2/canvas_errors.log

# PostgreSQL log
sudo tail -f /var/log/postgresql/postgresql-14-main.log
```

### Restart Services

```bash
# Restart Apache
sudo systemctl restart apache2

# Restart Canvas jobs
sudo /etc/init.d/canvas_init restart

# Restart Redis
sudo systemctl restart redis-server
```

---

## Troubleshooting Common Issues

### Issue 1: "502 Bad Gateway" Error

**Solution:**

```bash
# Check Passenger status
sudo passenger-status

# Restart Apache
sudo systemctl restart apache2

# Check logs
sudo tail -f /var/canvas/log/production.log
```

### Issue 2: Assets Not Loading

**Solution:**

```bash
cd /var/canvas
sudo -u canvasuser bash -c "RAILS_ENV=production bundle exec rake canvas:compile_assets"
sudo systemctl restart apache2
```

### Issue 3: Database Connection Error

**Solution:**

```bash
# Test database connection
sudo -u postgres psql -c "\l" | grep canvas

# Check credentials in config/database.yml
sudo nano /var/canvas/config/database.yml

# Restart PostgreSQL
sudo systemctl restart postgresql
```

### Issue 4: Background Jobs Not Running

**Solution:**

```bash
# Check jobs status
sudo /etc/init.d/canvas_init status

# Restart jobs
sudo /etc/init.d/canvas_init restart

# Check logs
sudo tail -f /var/canvas/log/delayed_job.log
```

---

## Backup Instructions

### Create Database Backup

```bash
# Create backup directory
sudo mkdir -p /var/backups/canvas

# Backup database
sudo -u postgres pg_dump canvas_production > /var/backups/canvas/canvas_db_$(date +%Y%m%d).sql

# Compress backup
gzip /var/backups/canvas/canvas_db_$(date +%Y%m%d).sql
```

### Backup Canvas Files

```bash
# Backup uploaded files
sudo tar -czf /var/backups/canvas/canvas_files_$(date +%Y%m%d).tar.gz /var/canvas/public/files

# Backup configuration
sudo tar -czf /var/backups/canvas/canvas_config_$(date +%Y%m%d).tar.gz /var/canvas/config/*.yml
```

---

## Updating Canvas

### To Update Canvas to Latest Version

```bash
# Navigate to Canvas directory
cd /var/canvas

# Stop jobs
sudo /etc/init.d/canvas_init stop

# Pull latest code
sudo -u canvasuser git fetch
sudo -u canvasuser git checkout prod
sudo -u canvasuser git pull

# Update dependencies
sudo -u canvasuser bundle install
sudo -u canvasuser yarn install

# Run migrations
sudo -u canvasuser bash -c "RAILS_ENV=production bundle exec rake db:migrate"

# Compile assets
sudo -u canvasuser bash -c "RAILS_ENV=production bundle exec rake canvas:compile_assets"

# Start jobs
sudo /etc/init.d/canvas_init start

# Restart Apache
sudo systemctl restart apache2
```

---

## Support Resources

- **Canvas Community:** https://community.canvaslms.com
- **Canvas Guides:** https://community.canvaslms.com/docs
- **GitHub Issues:** https://github.com/instructure/canvas-lms/issues
- **IRC Channel:** #canvas-lms on Freenode

---

## Congratulations! 🎉

Your Canvas LMS is now installed and running at **https://luxdevhq.ai**

**Next Steps:**
1. Explore the admin interface
2. Create courses and content
3. Invite teachers and students
4. Customize your Canvas theme
5. Set up integrations (Google Drive, Office 365, etc.)

**Security Reminders:**
- Keep your server updated: `sudo apt-get update && sudo apt-get upgrade`
- Regularly backup your database and files
- Monitor logs for suspicious activity
- Keep Canvas updated to the latest stable version
