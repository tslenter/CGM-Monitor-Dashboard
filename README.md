# Design for Medtronic 780G

The current design is for Medtronic, but with a few small changes, it can be adapted for Abbott or Dexcom.

![Empty Nightscout](https://github.com/tslenter/CGM-Monitor-Dashboard/blob/main/images/Design.png)

# 1. Nightscout server

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
## 1.12 Result

![Empty Nightscout](images/empty-nightscout.png)

# 2. Kiosk / Dashboard
````markdown
Software requirements:

CPU: Dual-core
RAM: 512 MB
Disk: 50 GB
Machine ID: 807e6781b48049a884fa7d5094c8b00c
Boot ID: 800f915e02824627b1fd93f47d1cf7f1
Operating System: Raspbian GNU/Linux 11 (bullseye)
Kernel: Linux 6.1.21-v7+
Architecture: arm

Hardware requirements:
Display: 7 Inch Touchscreen Display Voor Raspberry Pi Zero (WAV2041)
Computer: Raspberry Pi Zero 2 W
Storage: Transcend 16, 32, 64, 128GB Micro SD
Casing: Build it from a solid piece of wood, but any case will do.

Adjust the WEBSITE_URL="https://nightscout.lan.local/?token=monitorscr-strong-token" with the URL of your own server.
````

## 2.1 Update system

```bash
apt-get update --allow-releaseinfo-change && sudo apt-get upgrade -y
reboot
````

## 2.2 Install the base
```bash
apt update && sudo apt-get install --no-install-recommends xserver-xorg x11-xserver-utils xinit openbox -y
apt install --no-install-recommends chromium-browser -y
reboot
````

## 2.2 Configuration
```bash
cat > ~/.profile <<'EOF'
# ~/.profile: executed by the command interpreter for login shells.
# This file is not read by bash(1), if ~/.bash_profile or ~/.bash_login
# exists.
# see /usr/share/doc/bash/examples/startup-files for examples.
# the files are located in the bash-doc package.

# the default umask is set in /etc/profile; for setting the umask
# for ssh logins, install and configure the libpam-umask package.
#umask 022

# if running bash
if [ -n "$BASH_VERSION" ]; then
    # include .bashrc if it exists
    if [ -f "$HOME/.bashrc" ]; then
        . "$HOME/.bashrc"
    fi
fi

# set PATH so it includes user's private bin if it exists
if [ -d "$HOME/bin" ] ; then
    PATH="$HOME/bin:$PATH"
fi

# set PATH so it includes user's private bin if it exists
if [ -d "$HOME/.local/bin" ] ; then
    PATH="$HOME/.local/bin:$PATH"
fi

[[ -z $DISPLAY && $XDG_VTNR -eq 1 ]] && startx -- -nocursor
EOF

sudo tee /etc/xdg/openbox/autostart > /dev/null <<'EOF'
# Configuration
WEBSITE_URL="https://nightscout.lan.local/?token=monitorscr-strong-token"

# Disable screensaver, screen blanking, and power management
xset s off
xset s noblank
xset -dpms

# Auto-detect screen resolution
RESOLUTION=$(xrandr 2>/dev/null | grep '*' | awk '{print $1}')
if [ -z "$RESOLUTION" ]; then
    RESOLUTION="1024x600"  # Default fallback can be ="1280x720"
fi
SCREEN_WIDTH=$(echo $RESOLUTION | cut -d 'x' -f1)
SCREEN_HEIGHT=$(echo $RESOLUTION | cut -d 'x' -f2)

echo "Detected screen resolution: ${SCREEN_WIDTH}x${SCREEN_HEIGHT}"

# Check internet connection using ping before launching Chromium
if ping -c 1 -W 2 google.com >/dev/null 2>&1; then
    echo "Internet connected. Proceeding with Chromium launch."
else
    echo "No internet connection detected. Exiting."
fi

# Allow quitting the X server with CTRL-ALT-Backspace
# setxkbmap -option terminate:ctrl_alt_bksp

# Prevent Chromium restore prompts
sed -i 's/"exited_cleanly":false/"exited_cleanly":true/' ~/.config/chromium/'Local State'
sed -i 's/"exit_type":"[^"]\+"/"exit_type":"Normal"/' ~/.config/chromium/Default/Preferences

# Clean up Chromium cache, cookies, and logs
find ~/.config/chromium/Default/ -type f \( -name "Cookies" -o -name "History" -o -name "*.log" -o -name "*.ldb" -o -name "*.sqlite" \) -delete
rm -rf ~/.config/chromium/Default/Logs/*

# Clear system logs
sudo journalctl --vacuum-time=1d
sudo find /var/log -type f \( -name "*.log" -o -name "*.gz" -o -name "*.1" \) -delete
sudo truncate -s 0 /var/log/syslog /var/log/dmesg

# Kill any existing Chromium instances
pkill -9 chromium-browser 2>/dev/null
pkill -9 chrome 2>/dev/null

# Start Chromium in kiosk mode
chromium-browser --kiosk --disable-gpu --noerrdialogs --no-memcheck --disable-infobars --disable-features=TranslateUI \
    --disable-session-crashed-bubble --no-sandbox --disable-notifications --disable-sync-preferences \
    --no-sandbox --disable-background-mode --disable-popup-blocking --no-first-run \
    --enable-gpu-rasterization --disable-translate --disable-logging --disable-default-apps \
    --disable-extensions --disable-crash-reporter --disable-pdf-extension --disable-new-tab-first-run \
    --disable-dev-shm-usage --start-maximized --mute-audio --disable-crashpad --hide-scrollbars \
    --ash-hide-cursor --memory-pressure-off --force-device-scale-factor=1 --window-position=0,0 \
    --window-size=${SCREEN_WIDTH},${SCREEN_HEIGHT} "$WEBSITE_URL" &

if [ $? -eq 0 ]; then
    echo "Chromium started successfully."
else
    echo "Failed to start Chromium."
    sudo reboot
fi
EOF

systemctl ssh status
systemctl sshd status
systemctl status ssh
systemctl disable hciuart
systemctl enable systemd-timesyncd
systemctl restart systemd-timesyncd

sudo tee /etc/dphys-swapfile > /dev/null <<'EOF'
# /etc/dphys-swapfile - user settings for dphys-swapfile package
# author Neil Franklin, last modification 2010.05.05
# copyright ETH Zuerich Physics Departement
#   use under either modified/non-advertising BSD or GPL license

# this file is sourced with . so full normal sh syntax applies

# the default settings are added as commented out CONF_*=* lines


# where we want the swapfile to be, this is the default
#CONF_SWAPFILE=/var/swap

# set size to absolute value, leaving empty (default) then uses computed value
#   you most likely don't want this, unless you have an special disk situation
CONF_SWAPSIZE=0

# set size to computed value, this times RAM size, dynamically adapts,
#   guarantees that there is enough swap without wasting disk space on excess
#CONF_SWAPFACTOR=2

# restrict size (computed and absolute!) to maximally this limit
#   can be set to empty for no limit, but beware of filled partitions!
#   this is/was a (outdated?) 32bit kernel limit (in MBytes), do not overrun it
#   but is also sensible on 64bit to prevent filling /var or even / partition
#CONF_MAXSWAP=2048
EOF

reboot
````
## 2.3 Result

### 2.3.1 7 inch diplay

![Empty Nightscout](https://github.com/tslenter/CGM-Monitor-Dashboard/blob/main/images/7%20inch%20dashboard.png)

### 2.3.2 27 inch diplay

![Empty Nightscout](https://github.com/tslenter/CGM-Monitor-Dashboard/blob/main/images/27inchdashboard.png)

# 3. Configure DRIPX with CareLink and Nightscout

This chapter describes how to configure **DRIPX** to retrieve patient data from **CareLink** and upload the glucose data to **Nightscout**.

---

<table>
<tr>
<td width="50%" valign="top">

### 3.1 Configure CareLink

Open DRIPX and configure the **CareLink** connection and login credentials.

<p align="center">
  <img src="images/DRIPX%20-%20LOGIN%20and%20CareLink%20option.jpeg" alt="DRIPX CareLink Login" width="50%">
</p>

</td>
<td width="50%" valign="top">

### 3.2 Select the Patient Data Source

Select **CareLink** as the source for the patient's glucose data.

<p align="center">
  <img src="images/DRIPX%20-%20source%20selection.jpeg" alt="DRIPX Source Selection" width="50%">
</p>

</td>
</tr>
</table>

<table>
<tr>
<td width="50%" valign="top">

### 3.3 Configure the Patient Name

Enter or select the **CareLink patient name**.

<p align="center">
  <img src="images/DRIPX%20-%20CareLink%20Patientname.jpeg" alt="DRIPX CareLink Patient Name" width="50%">
</p>

</td>
<td width="50%" valign="top">

### 3.4 Configure the Patient Country

Select the country associated with the patient's **CareLink account**.

<p align="center">
  <img src="images/DRIPX%20-%20Countru%20Selection.jpeg" alt="DRIPX Country Selection" width="50%">
</p>

</td>
</tr>
</table>

---

<table>
<tr>
<td width="50%" valign="top">

### 3.5 Configure Nightscout Upload

Open the **DRIPX Settings** to configure the upload to Nightscout.

<p align="center">
  <img src="images/DRIPX%20-%20Settings.jpeg" alt="DRIPX Settings" width="50%">
</p>

</td>
<td width="50%" valign="top">

### 3.6 Configure Cloud Upload

Enable and configure the **Cloud Upload** functionality.

<p align="center">
  <img src="images/DRIPX%20-%20Cloud%20upload.jpeg" alt="DRIPX Cloud Upload" width="50%">
</p>

</td>
</tr>
</table>

<table>
<tr>
<td width="50%" valign="top">

### 3.7 Configure the REST API

Select and configure the **REST API** option for uploading the glucose data to Nightscout.

<p align="center">
  <img src="images/DRIPX%20-%20REST-API.jpeg" alt="DRIPX REST API" width="50%">
</p>

</td>
<td width="50%" valign="top">

### 3.8 Configure the Nightscout Base URL

Enter the **Nightscout Base URL**.

<p align="center">
  <img src="images/DRIPX%20-%20Nightscout%20base%20URL.jpeg" alt="DRIPX Nightscout Base URL" width="50%">
</p>

</td>
</tr>
</table>

The Nightscout URL should use the following format:

```text
https://token@hostname/api/v1/
```

## 3.9 Example Results on the Phone

The following screenshots show examples of the Nightscout dashboard on a phone.

<table>
<tr>
<td width="50%" valign="top">

### 3.9.1 Without Login

Example of the Nightscout dashboard on lockscreen.

<p align="center">
  <img src="images/DRIP-X-Telefoon-geenlogin.png" alt="Nightscout without login" width="50%">
</p>

</td>
<td width="50%" valign="top">

### 3.9.2 With Login

Example of the Nightscout dashboard when loged in.

<p align="center">
  <img src="images/DRIP-X-Telefoon-login.png" alt="Nightscout with login" width="50%">
</p>

</td>
</tr>
</table>

# 4. Configure Power Settings (Samsung S25 Ultra)

To keep the **Medtronic app connected to the insulin pump**, the phone's **power and battery management settings** must be configured correctly.

Android power-saving features may restrict or stop applications that are running in the background. If the **Medtronic app** is restricted, the connection between the Medtronic app and the insulin pump may be interrupted.

The following power settings should therefore be configured to allow the Medtronic app to continue running in the background.

> [!IMPORTANT]
> These settings are intended to help maintain the connection between the **Medtronic app and the insulin pump**. They are not related to the connection between DRIPX and CareLink.
>
> The exact names and locations of these settings may differ depending on the Android version and phone manufacturer.

---

<table>
<tr>
<td width="50%" valign="top">

### 4.1 Open the Power Settings

Open the phone's **Power/Battery settings** and locate the settings that control background activity and power management.

<p align="center">
  <img src="images/engergie-uit1.jpg" alt="Android Power Settings - Step 1" width="50%">
</p>

</td>
<td width="50%" valign="top">

### 4.2 Configure Battery Optimization

Configure the battery optimization settings so that the **Medtronic app** is not restricted by Android's power-saving functionality.

<p align="center">
  <img src="images/engergie-uit2.jpg" alt="Android Power Settings - Step 2" width="50%">
</p>

</td>
</tr>
</table>

<table>
<tr>
<td width="50%" valign="top">

### 4.3 Allow Background Activity

Make sure that the **Medtronic app** is allowed to continue running in the background.

This is important for maintaining communication between the Medtronic app and the insulin pump when the phone screen is turned off.

<p align="center">
  <img src="images/engergie-uit3.jpg" alt="Android Power Settings - Step 3" width="50%">
</p>

</td>
<td width="50%" valign="top">

### 4.4 Disable Power Restrictions

Disable any remaining power or battery restrictions that could prevent the **Medtronic app** from running continuously in the background.

<p align="center">
  <img src="images/engergie-uit4.jpg" alt="Android Power Settings - Step 4" width="50%">
</p>

</td>
</tr>
</table>

---

## 4.5 Why These Settings Are Important

The **Medtronic app must remain active in the background** to maintain its connection with the insulin pump.

Android may automatically limit background applications to reduce battery consumption. When this happens to the Medtronic app, communication between the **Medtronic app and the pump** may be interrupted.

After completing the configuration, verify that:

- The Medtronic app is excluded from battery optimization.
- Background activity for the Medtronic app is allowed.
- Power-saving restrictions for the Medtronic app are disabled.
- The Medtronic app can continue running when the phone screen is turned off.
- The connection between the Medtronic app and the insulin pump remains active.

> [!NOTE]
> If the connection between the **Medtronic app and the insulin pump** is regularly interrupted when the phone screen is turned off or after the phone has been idle for some time, check these power settings first.

# 5. Configure Glucose Data for the Smartwatch

To display glucose data on a **Wear OS smartwatch**, DRIPX can be configured to broadcast the glucose values locally on the Android phone.

**GlucoDataHandler (GDH)** can then receive these locally broadcast glucose values and make them available to the smartwatch.

The data flow is:

**Medtronic / CareLink → DRIPX → Local Broadcast → GlucoDataHandler → Wear OS Smartwatch**

> [!IMPORTANT]
> DRIPX and GlucoDataHandler must be installed on the same Android phone for this local broadcast configuration.
>
> GlucoDataHandler is not a standalone Wear OS glucose source. The phone application is required to receive and process the glucose data before it is made available to the smartwatch.

---

<table>
<tr>
<td width="50%" valign="top">

### 5.1 Enable Local Broadcast in DRIPX

Open the **DRIPX settings** and enable the option to send glucose data locally.

This allows other compatible applications on the same Android device, such as **GlucoDataHandler**, to receive the glucose values.

<p align="center">
  <img src="images/DRIPX%20-%20SEND%20LOCAL.jpeg" alt="DRIPX Send Local" width="50%">
</p>

</td>
<td width="50%" valign="top">

### 5.2 Configure the Source in GlucoDataHandler

Open **GlucoDataHandler** and navigate to the **Sources** configuration.

Select and enable the local source that corresponds with the broadcast format used by DRIPX.

<p align="center">
  <img src="images/GlucoDataHandler%20-%20Sources.jpeg" alt="GlucoDataHandler Sources" width="50%">
</p>

</td>
</tr>
</table>

---

## 5.3 Configure the Local Broadcast

Make sure GlucoDataHandler is configured to receive the locally broadcast glucose values from DRIPX.

For an xDrip+-compatible local broadcast, GlucoDataHandler supports receiving glucose values through the **xDrip+ Broadcast locally** source.

When required, make sure the local broadcast configuration allows GlucoDataHandler to receive the data.

<p align="center">
  <img src="images/GlucoDataHandler%20-%20Configure%20Send%20Local.jpeg" alt="GlucoDataHandler Configure Local Broadcast" width="25%">
</p>

> [!TIP]
> If the glucose values are not received by GlucoDataHandler, verify that local broadcasting is enabled in DRIPX and that the corresponding local source is enabled in GlucoDataHandler.
>
> For an xDrip+-compatible broadcast, the GlucoDataHandler documentation identifies its Android receiver as:
>
> `de.michelinside.glucodatahandler`

---

## 5.4 Configure the Wear OS Smartwatch

Once GlucoDataHandler receives the glucose values on the phone, the data can be displayed on a compatible **Wear OS smartwatch**.

GlucoDataHandler provides **Wear OS complications**. These complications can be added to a compatible watchface to display information such as:

- Current glucose value
- Glucose trend
- Delta/change in glucose
- Timestamp
- Phone and watch battery information

The exact information that can be displayed depends on the selected complication and watchface.

> [!NOTE]
> GlucoDataHandler does not provide a complete standalone watchface. It provides Wear OS **complications** that can be placed in complication areas of compatible watchfaces.

---

## 5.5 Result

After the configuration is complete, glucose information from DRIPX can be displayed directly on the smartwatch through GlucoDataHandler.

The final data flow is:

```text
Medtronic / CareLink
        │
        ▼
      DRIPX
        │
        │ Local Broadcast
        ▼
GlucoDataHandler
        │
        ▼
 Wear OS Smartwatch
```

An example of the final result is shown below.

<p align="center">
  <img src="images/Horloge.png" alt="Glucose Data on Wear OS Smartwatch" width="25%">
</p>

---

## 5.6 Troubleshooting

If glucose data is not displayed on the smartwatch, verify the following:

- DRIPX is receiving current glucose values.
- Local broadcasting is enabled in DRIPX.
- The correct local source is enabled in GlucoDataHandler.
- GlucoDataHandler is receiving current glucose values on the phone.
- GlucoDataHandler is installed and configured for Wear OS.
- The GlucoDataHandler complication has been added to the selected watchface.
- Bluetooth/Wear OS connectivity between the phone and smartwatch is working.
- Android battery optimization is not preventing the required applications from running in the background.

> [!TIP]
> First verify that the current glucose value is visible in **GlucoDataHandler on the phone**. If the value is visible on the phone but not on the smartwatch, the problem is most likely related to the Wear OS configuration or phone-to-watch communication rather than the DRIPX local broadcast.
