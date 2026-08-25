# 1. Server
````markdown
Requirements:
CPU: Dual core
RAM: 4GB
DISK: 50GB
O/S: Ubuntu 26.04 LTS
Database: MongoDB 8.3
Applications: NodeJS 24.19.0

Best is to run a VPS.
Run commands as root, unless specified otherwise.
````
## 1.1 Update system

```bash
apt update && apt upgrade -y
apt autoremove
reboot
````

## 1.2 Firewall

```bash
ufw allow 'http'
ufw allow 'https'
ufw allow 'ssh'
ufw default deny incoming
ufw default allow outgoing
ufw enable
````

## 1.2 Installation Apache 2

```bash
apt install -y apache2
systemctl enable --now apache2
mkdir /etc/cert/
chmod 700 /etc/cert
openssl req -x509 -nodes -newkey rsa:4096 \
  -sha256 \
  -days 365 \
  -keyout /etc/cert/apache2.key \
  -out /etc/cert/apache2.crt \
  -subj "/CN=your-server.example.com" \
  -addext "subjectAltName=DNS:nightscout.lan.local"
chown root:root /etc/cert/apache2.key /etc/cert/apache2.crt
chmod 600 /etc/cert/apache2.key
chmod 644 /etc/cert/apache2.crt
a2enmod ssl
apache2ctl configtest
systemctl reload apache2
systemctl status apache2
```

## 1.3 Create default status webpage

```bash
mkdir -p /var/www/server-status/public
cd /var/www/server-status/public
tee /var/www/server-status/public/index.html >/dev/null <<'EOF'
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Server Running</title>
    <style>
        body {
            margin: 0;
            min-height: 100vh;
            display: grid;
            place-items: center;
            background: #f5f7fa;
            font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
            color: #1f2937;
        }
        .card {
            max-width: 520px;
            margin: 20px;
            padding: 40px;
            text-align: center;
            background: white;
            border-radius: 16px;
            box-shadow: 0 10px 30px rgba(0,0,0,.08);
        }
        .status {
            display: inline-block;
            padding: 8px 16px;
            border-radius: 999px;
            background: #dcfce7;
            color: #166534;
            font-weight: 600;
        }
        h1 {
            margin: 24px 0 10px;
        }
        p {
            color: #6b7280;
        }
    </style>
</head>
<body>
    <main class="card">
        <span class="status">● Server Running</span>
        <h1>Nightscout</h1>
        <p>The web server is online and responding.</p>
    </main>
</body>
</html>
EOF
chown -R www-data:www-data /var/www/server-status
find /var/www -type d -exec chmod 755 {} \;
find /var/www -type f -exec chmod 644 {} \;
tee /etc/apache2/sites-available/000-server-status.conf >/dev/null <<'EOF'
<VirtualHost *:80>
    ServerName _default_

    DocumentRoot /var/www/server-status/public

    <Directory /var/www/server-status/public>
        Options -Indexes
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName _default_

    DocumentRoot /var/www/server-status/public

    <Directory /var/www/server-status/public>
        Options -Indexes
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    SSLEngine on

    SSLCertificateFile /etc/cert/apache2.crt
    SSLCertificateKeyFile /etc/cert/apache2.key

    SSLProtocol -all +TLSv1.2 +TLSv1.3

    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"
</VirtualHost>
EOF
a2dissite 000-default.conf
a2enmod ssl
a2enmod headers
a2ensite 000-server-status.conf
apache2ctl configtest
systemctl reload apache2
```

## 1.4 Install MongoDB 8.3
```bash
apt install gnupg curl -y
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-8.3.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.3.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.3 multiverse" | tee /etc/apt/sources.list.d/mongodb-org-8.0.list
apt update
apt install -y mongodb-org && apt upgrade -y
systemctl enable mongod
systemctl edit mongod

START OF EDIT:

[Service]
ExecStart=
ExecStart=/usr/bin/mongod --config /etc/mongod.conf
Environment="GLIBC_TUNABLES=glibc.pthread.rseq=1"

END OF EDIT

systemctl daemon-reload
systemctl stop mongod
systemctl start mongod
systemctl status mongod
```

### Enable apache2 modules

```bash
sudo a2enmod ssl
sudo a2enmod headers
sudo a2enmod rewrite
sudo a2enmod http2
sudo a2enmod proxy
sudo a2enmod proxy_http
```

### Manage websites

```bash
cd /etc/apache2/sites-available/
ls
sudo a2ensite nightscout.conf
sudo a2dissite nightscout.conf
```

## 3. Website Files

### Create directories

```bash
sudo mkdir -p /var/www/nightscout/public
sudo mkdir -p /var/www/server-status/public
```

### Ownership & permissions

```bash
sudo chown -R www-data:www-data /var/www/
sudo find /var/www -type d -exec chmod 755 {} \;
sudo find /var/www -type f -exec chmod 644 {} \;
```

### File management

```bash
ls -lah
cd /path/to/directory
cd ..
nano FILE
```

## 6. SSH

```bash
sudo nano /etc/ssh/sshd_config
sudo sshd -t
sudo systemctl restart ssh
```

## 7. Users

```bash
sudo adduser nightscout
sudo usermod -aG sudo nightscout
su - nightscout
```

## 8. MongoDB

### Install

```bash

```

### Manage MongoDB

```bash
sudo systemctl enable --now mongod
sudo systemctl status mongod
sudo systemctl restart mongod
```

### Configure MongoDB

```bash
sudo nano /etc/mongod.conf
sudo systemctl edit mongod
sudo systemctl daemon-reload
```

### MongoDB shell

```bash
mongosh --port 27017
mongosh -u administrator -p --authenticationDatabase admin
mongosh -u nightscout -p --authenticationDatabase nightscout
```

## 9. Node.js / NVM

### Node.js

```bash
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v
```

### NVM

```bash
source ~/.nvm/nvm.sh
nvm install 24.19.0
nvm use 24.19.0
```

## 10. Nightscout

### Install

```bash
cd /opt
git clone https://github.com/nightscout/cgm-remote-monitor.git
ln -s cgm-remote-monitor nightscout
cd /opt/nightscout
npm install
```

### Configuration

```bash
nano my.env
```

### Test locally

```bash
curl 127.0.0.1:1337
```

## 11. PM2 / Nightscout

### Install PM2

```bash
sudo npm install pm2 -g
```

### Start Nightscout

```bash
env $(cat my.env) PORT=1337 pm2 start server.js
```

### Manage Nightscout

```bash
pm2 status
pm2 logs
pm2 restart nightscout --update-env
pm2 stop server.js
pm2 delete server
```

### Start automatically after reboot

```bash
pm2 startup
pm2 save
```

## 12. Nightscout + Apache

### Enable proxy modules

```bash
sudo a2enmod proxy
sudo a2enmod proxy_http
```

### Configure virtual host

```bash
sudo nano /etc/apache2/sites-available/nightscout.remotesyslog.com.conf
```

### Enable site

```bash
sudo a2ensite nightscout.remotesyslog.com.conf
```

### Test and reload Apache

```bash
sudo apache2ctl configtest
sudo systemctl reload apache2
```

## 13. Logs & Troubleshooting

### System logs

```bash
sudo journalctl
sudo journalctl -p warning -b
cat /var/log/syslog
```

### Nightscout

```bash
pm2 logs
pm2 status
```

### MongoDB

```bash
sudo systemctl status mongod
cd /var/log/mongodb/
ls -lah
```

### Network

```bash
sudo ss -lntp
netstat -tuna
```

### Packet capture

```bash
sudo tcpdump -i any port 17010
sudo tcpdump -i any port 27017
```

## 14. Reboot

```bash
sudo reboot
```

## 15. Quick Server Check

```bash
sudo systemctl status apache2
sudo systemctl status mongod
pm2 status
sudo apache2ctl configtest
sudo ss -lntp
sudo ufw status
curl 127.0.0.1:1337
```

## Important Paths

```text
/etc/apache2/
/etc/apache2/sites-available/
/etc/apache2/sites-enabled/
/etc/mongod.conf
/etc/cert/
/var/www/
/opt/nightscout/
/home/nightscout/nightscout/
```

In Notepad, select **Save as type → All Files**, and make sure it doesn't become `server-manual.md.txt`.
```
