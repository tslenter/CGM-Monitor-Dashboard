# 1. Server
````markdown
Requirements:

CPU: Dual-core
RAM: 4 GB
Disk: 50 GB
Operating System: Ubuntu 26.04 LTS
Database: MongoDB 8.3
Application Runtime: Node.js 24.19.0

It is recommended to run the installation on a VPS.

Run all commands as root, unless otherwise specified.

Wherever a password such as my-strong-pw is shown, replace it with your own strong password. Make sure to also update the corresponding password in all relevant configuration files.

The Apache2 configuration uses virtual hosts, which requires DNS to be configured correctly. In the examples, the hostname is set to nightscout.lan.local. Replace this with the DNS hostname you have configured for your environment.

If the DNS name and Apache2 virtual host configuration do not match, you may see the default/status webpage instead of the Nightscout application.
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
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.3.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.3 multiverse" | tee /etc/apt/sources.list.d/mongodb-org-8.3.list
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

## 1.5 Configure MongoDB 8.3

```bash
ulimit -n 64000
mongosh --port 27017
use admin
db.createUser({user: "administrator", pwd: "my-strong-pw", roles: [ { role: "userAdminAnyDatabase", db: "admin" }] })
db.updateUser(
  "administrator",
  { pwd: "my-strong-pw" }
)
exit
tee /etc/mongod.conf >/dev/null <<'EOF'
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# Where and how to store data.
storage:
  dbPath: /var/lib/mongodb
#  engine:
#  wiredTiger:

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

# network interfaces
net:
  port: 27017
  bindIp: 127.0.0.1


# how the process runs
processManagement:
  timeZoneInfo: /usr/share/zoneinfo

security:
  authorization: enabled

#operationProfiling:

#replication:

#sharding:

## Enterprise-Only Options:

#auditLog:
EOF
systemctl daemon-reload
systemctl stop mongod
systemctl start mongod
systemctl status mongod
mongosh --port 27017

mongosh -u administrator -p --authenticationDatabase admin
use nightscout
db.createUser({user: "nightscout", pwd: "my-strong-pw", roles: [ { role: "readWrite", db: "nightscout" }]})
exit
mongosh -u nightscout -p --authenticationDatabase nightscout
exit
```

## 1.6 Create nightscout user

```bash
adduser nightscout
usermod -aG sudo nightscout
su - nightscout
exit
```

## 1.7 Install additional packages

```bash
apt install build-essential checkinstall libssl-dev -y
npm install pm2 -g
```

## 1.6 Install NodeJS 24.x

```bash
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v
```

## 1.7 Install NVM 24.19.0

```bash
su - nightscout
wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.33.8/install.sh | bash
source ~/.nvm/nvm.sh
nvm install 24.19.0
nvm use 24.19.0
exit
```

## 1.8 Install nightscout

```bash
su - nightscout
git clone https://github.com/nightscout/cgm-remote-monitor.git
ln -s cgm-remote-monitor nightscout
cd /home/nightscout/nightscout
npm install
exit
```

## 1.9 Configure nightscout
```bash
su - nightscout
cd /home/nightscout/nightscout
tee my.env >/dev/null <<'EOF'
MONGODB_URI=mongodb://nightscout:my-strong-pw@127.0.0.1:27017/nightscout
MONGODB_COLLECTION=entries
DISPLAY_UNITS=mmol/l
BASE_URL=http://127.0.0.1:1337
AUTH_DEFAULT_ROLES=denied
ALARM_HIGH=on
ALARM_LOW=on
ALARM_TIMEAGO_URGENT=on
ALARM_TYPES=simple
ALARM_TIMEAGO_WARN=on
ALARM_URGENT_HIGH=on
ALARM_URGENT_LOW=on
BRIDGE_SERVER=EU
NIGHT_MODE=on
THEME=colors
TIME_FORMAT=24
ALARM_TIMEAGO_URGENT_MINS=30
ALARM_TIMEAGO_WARN_MINS=15
API_SECRET=my-strong-pw
BG_HIGH=15
BG_LOW=3
BG_TARGET_BOTTOM=4
BG_TARGET_TOP=10
BOLUS_RENDER_OVER=1
BRIDGE_PASSWORD=
BRIDGE_USER_NAME=
CUSTOM_TITLE=Nightscout
ENABLE=careportal%20basal%20dbsize%20rawbg%20iob%20maker%20cob%20bwp%20cage%20iage%20sage%20boluscalc%20pushover%20treatmentnotify%20loop%20pump%20profile%20food%20openaps%20bage%20alexa%20override%20speech%20cors
SHOW_PLUGINS=careportal%20dbsize
EOF
npm install pm2 -g
pm2 cleardump
env $(cat /home/nightscout/nightscout/my.env) PORT=1337 pm2 start server.js
sudo env PATH=$PATH:/home/nightscout/.nvm/versions/node/v24.19.0/bin /usr/bin/pm2 startup systemd -u nightscout --hp /home/nightscout
pm2 status
pm2 startup
pm2 save
exit
```

## 1.10 Configure apache 2 as proxy

```bash
a2enmod ssl
a2enmod headers
a2enmod rewrite
a2enmod http2
a2enmod proxy
a2enmod proxy_http
tee /etc/apache2/sites-available/nightscout.conf > /dev/null <<'EOF'
# Redirect http => HTTPS
<VirtualHost *:80>
        ServerName nightscout.lan.local

        ServerAdmin info@lan.local
        DocumentRoot /var/www/server-status/public/

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        <Location />
                RewriteEngine On
                RewriteCond %{HTTPS}  !=on
                RewriteRule ^/?(.*) https://%{SERVER_NAME}/ [R,L]
        </Location>
</VirtualHost>

<VirtualHost *:443>
  ServerName nightscout.lan.local

  Protocols h2 http/1.1

  Header always set Strict-Transport-Security "max-age=63072000"

  SSLEngine on
  SSLCertificateFile /etc/cert/apache2.crt
  SSLCertificateKeyFile /etc/cert/apache2.key

  SSLProtocol             all -SSLv3 -TLSv1 -TLSv1.1
  SSLCipherSuite          ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
  SSLHonorCipherOrder     off
  SSLSessionTickets       off

  RewriteEngine On
  RewriteCond %{HTTP:Upgrade} =websocket [NC]
  RewriteRule /(.*) ws://127.0.0.1:1337/$1 [P,L]
  RewriteRule .* - [E=upgrade:1]

  SSLProxyEngine on
  ProxyRequests Off

  <Location "/">
    ProxyPreserveHost on
    ProxyPass         http://127.0.0.1:1337/
    ProxyPassReverse  http://127.0.0.1:1337/
    RequestHeader     set "X-Forwarded-Proto" expr=%{REQUEST_SCHEME}
    RequestHeader     set Connection "upgrade" env=upgrade
    RequestHeader     set Upgrade "%{HTTP:Upgrade}e" env=upgrade
  </Location>
</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
EOF
apache2ctl configtest && a2ensite nightscout.conf && systemctl reload apache2
```

## 1.11 Important Paths

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
