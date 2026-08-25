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
## 1.1 System & Packages

```bash
apt update && apt upgrade -y
apt autoremove
reboot
````

## 2. Apache

### Install & manage

```bash
sudo apt install -y apache2
sudo systemctl enable --now apache2
sudo systemctl status apache2
sudo systemctl restart apache2
sudo systemctl reload apache2
```

### Test configuration

```bash
sudo apache2ctl configtest
sudo apache2ctl -S
```

### Enable modules

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
sudo a2ensite SITE.conf
sudo a2dissite SITE.conf
```

## 3. Website Files

### Create directories

```bash
sudo mkdir -p /var/www/remotesyslog.com/public
sudo mkdir -p /var/www/cloud.remotesyslog.com/public
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

### Copy files with SCP

```bash
scp -r USER@SERVER:/path/* .
scp -r . USER@SERVER:/path/
```

## 4. SSL Certificates

```bash
sudo mkdir -p /etc/cert
cd /etc/cert
ls -lah
sudo chmod 400 remotesyslog.key
```

### Copy certificates

```bash
scp -r USER@SERVER:/etc/cert/* .
```

## 5. Firewall / UFW

```bash
sudo ufw status
sudo ufw status numbered
sudo ufw allow ssh
sudo ufw allow 'Apache Full'
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
```

### Allow SSH from a specific IP

```bash
sudo ufw allow from YOUR_IP to any port 22
```

### Delete a rule

```bash
sudo ufw delete RULE_NUMBER
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
sudo apt install gnupg curl -y

curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | \
sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

sudo apt update
sudo apt install -y mongodb-org
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
