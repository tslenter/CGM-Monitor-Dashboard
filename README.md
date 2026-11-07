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
## 2.3 Raspberry Pi OS Trixie — Nightscout Kiosk Client

> **Platform:** Raspberry Pi Zero 2 W (512 MB RAM) · Raspberry Pi OS Trixie Lite (32-bit) · Waveshare 7-inch HDMI touchscreen (1024×600).
>
> **Stack:** X11, Openbox, Chromium, and the Xorg `fbdev` driver.

This section describes the Trixie-based Nightscout client installation. All commands assume the regular Linux user is `tom` with home directory `/home/tom`, unless stated otherwise. Replace both values if your installation uses a different account. Run the commands as `tom` over SSH, not from a root shell, except where `sudo` is explicitly used.

### 2.3.1 Install Raspberry Pi OS Trixie Lite

Use Raspberry Pi Imager to install **Raspberry Pi OS Lite (32-bit) — Trixie**. Configure your Wi-Fi network, Wi-Fi country, SSH access, username (`tom` in this example), hostname (`lumon0001` in this example), and time zone (`Europe/Amsterdam`).

Connect via SSH, then update the system:

```bash
ssh tom@lumon0001.local
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

If `.local` name resolution is unavailable, connect using the Raspberry Pi's IP address instead. Reconnect after the reboot before continuing.

### 2.3.2 Verify the Waveshare HDMI display

Check that HDMI is connected and the display advertises 1024×600:

```bash
cat /sys/class/drm/card0-HDMI-A-1/status
cat /sys/class/drm/card0-HDMI-A-1/modes
```

Expected: `connected` and a `1024x600` mode. The tested Trixie installation uses VC4 KMS (`dtoverlay=vc4-kms-v3d`) in `/boot/firmware/config.txt`. **If the Linux console already displays correctly, do not change the HDMI boot configuration.** Do not blindly copy older Bullseye `hdmi_group`, `hdmi_mode`, or `hdmi_cvt` settings.

The DRM connector path can differ between installations; inspect `/sys/class/drm/` if the path above does not exist.

### 2.3.3 Install X11, Openbox, and Chromium

Install the minimal graphical stack without a full desktop environment:

```bash
sudo apt update
sudo apt install -y \
  xserver-xorg \
  xserver-xorg-video-fbdev \
  xinit openbox x11-xserver-utils x11-utils \
  chromium ca-certificates
```

#### 2.3.3.1 Fix the black screen using Xorg `fbdev`

On the tested Waveshare setup, the Linux console worked, but `startx` produced a black screen. Xorg was using the `modesetting` driver with `glamor` acceleration. Forcing the `fbdev` driver resolved the issue:

```bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo tee /etc/X11/xorg.conf.d/20-waveshare-fbdev.conf >/dev/null <<'EOF_XORG'
Section "Device"
    Identifier "Waveshare framebuffer"
    Driver "fbdev"
    Option "fbdev" "/dev/fb0"
EndSection
EOF_XORG
```

**Keep this file in place on the tested hardware.** It is the key display fix for this setup.

### 2.3.4 Configure the Nightscout URL

Create the configuration directory and edit the URL file:

```bash
mkdir -p /home/tom/.config/nightscout-kiosk
nano /home/tom/.config/nightscout-kiosk/url
```

Enter your Nightscout URL on **one line**, for example:

```text
https://nightscout.example.com/
```

Include an access token only if your Nightscout deployment requires one. **Never commit a real token or token-bearing URL to GitHub.** Restrict access to the local URL file:

```bash
chmod 600 /home/tom/.config/nightscout-kiosk/url
```

### 2.3.5 Create the Chromium kiosk launcher

Create the browser launcher using this copy-and-paste console block:

```bash
cat > /home/tom/.config/nightscout-kiosk/start-browser.sh <<'EOF_BROWSER'
#!/bin/bash

URL_FILE="/home/tom/.config/nightscout-kiosk/url"

if [ ! -s "$URL_FILE" ]; then
    echo "ERROR: Nightscout URL is missing"
    exit 1
fi

WEBSITE_URL="$(head -n 1 "$URL_FILE")"

# Disable screen blanking and display power management
xset s off
xset s noblank
xset -dpms

# Start the lightweight window manager
openbox &

CHROMIUM="$(command -v chromium || command -v chromium-browser)"
if [ -z "$CHROMIUM" ]; then
    echo "ERROR: Chromium is not installed"
    exit 1
fi

# Restart Chromium if it exits
while true; do
    "$CHROMIUM" \
        --kiosk \
        --no-first-run \
        --no-default-browser-check \
        --noerrdialogs \
        --no-memcheck \
        --disable-session-crashed-bubble \
        --disable-infobars \
        --disable-extensions \
        --disable-gpu \
        --disable-dev-shm-usage \
        --disable-background-mode \
        --disable-sync \
        --hide-scrollbars \
        --mute-audio \
        --window-size=1024,600 \
        --user-data-dir=/home/tom/.config/nightscout-kiosk/chromium-profile \
        "$WEBSITE_URL"

    echo "Chromium exited. Restarting in 10 seconds."
    sleep 10
done
EOF_BROWSER

chmod 700 /home/tom/.config/nightscout-kiosk/start-browser.sh
```

On the tested installation, `--no-memcheck` suppressed Chromium's low-memory warning. We deliberately do **not** use `--no-sandbox` or install `unclutter`.

### 2.3.6 Configure X11 and test the kiosk

Create `.xinitrc` and ensure the files belong to the kiosk user:

```bash
cat > /home/tom/.xinitrc <<'EOF_XINIT'
#!/bin/sh
exec /home/tom/.config/nightscout-kiosk/start-browser.sh
EOF_XINIT

chmod 700 /home/tom/.xinitrc
sudo chown -R tom:tom /home/tom/.config/nightscout-kiosk
sudo chown tom:tom /home/tom/.xinitrc
```

Log in as `tom` on the **physical Raspberry Pi console**, then test:

```bash
startx
```

Nightscout should open full-screen. Do **not** run `startx` as root.

### 2.3.7 Enable automatic login and kiosk startup

Use `agetty` autologin on `tty1`, then launch X11 from the Bash login profile. The Xorg `-nocursor` option hides the pointer without `unclutter`.

Create the systemd override:

```bash
sudo mkdir -p /etc/systemd/system/getty@tty1.service.d
sudo tee /etc/systemd/system/getty@tty1.service.d/autologin.conf >/dev/null <<'EOF_GETTY'
[Service]
ExecStart=
ExecStart=-/sbin/agetty --autologin tom --noclear %I $TERM
EOF_GETTY
```

Append the following block to `/home/tom/.bash_profile` **only once**, preserving any existing content:

```bash
cat >> /home/tom/.bash_profile <<'EOF_PROFILE'

# Nightscout kiosk: launch only on the physical console
if [ -z "${DISPLAY:-}" ] && [ "$(tty)" = "/dev/tty1" ]; then
    exec startx -- :0 vt1 -nocursor
fi
EOF_PROFILE
sudo chown tom:tom /home/tom/.bash_profile
```

Disable any previously created, competing kiosk service, then reboot:

```bash
sudo systemctl disable --now nightscout-kiosk.service 2>/dev/null || true
sudo systemctl daemon-reload
sudo reboot
```

**Verification:** Nightscout should appear automatically after reboot. The autostart settings were configured during the original setup, but a successful post-reboot check was not separately confirmed.

### 2.3.8 Disable Bluetooth

If Bluetooth is not required, disable only its service:

```bash
sudo systemctl disable --now bluetooth.service
systemctl is-enabled bluetooth.service
systemctl is-active bluetooth.service
```

Confirmed output on the tested installation:

```text
disabled
inactive
```

Leave Wi-Fi, NetworkManager, SSH, Avahi, Cron, and the other required system services enabled.

### 2.3.9 Schedule a weekly reboot: Monday at 02:00

Create a Cron entry to reboot the kiosk every Monday at 02:00 local time:

```bash
sudo tee /etc/cron.d/nightscout-reboot >/dev/null <<'EOF_CRON'
# Nightscout kiosk: reboot every Monday at 02:00
0 2 * * 1 root /usr/sbin/reboot
EOF_CRON
sudo chmod 644 /etc/cron.d/nightscout-reboot
sudo timedatectl set-timezone Europe/Amsterdam
systemctl is-active cron.service
cat /etc/cron.d/nightscout-reboot
```

Cron was confirmed as `active`, and the time zone was confirmed as `Europe/Amsterdam`. The first actual scheduled reboot had not yet been independently verified.

### 2.3.10 Manage swap on Trixie

The tested Trixie installation uses `rpi-swap` with zram. Swap was successfully disabled **temporarily**:

```bash
sudo swapoff -a
free -h
swapon --show
```

Afterward, `free -h` showed `Swap: 0B`. **Permanent swap disablement was not confirmed.** Before modifying `/etc/rpi/swap.conf`, inspect the options supported by your installed version:

```bash
cat /etc/rpi/swap.conf
man swap.conf
```

**Caution:** the Raspberry Pi Zero 2 W has only 512 MB of RAM. Without swap, Chromium may terminate under memory pressure. Disabling swap is optional; test stability before relying on it.

### 2.3.11 Diagnostics and maintenance

Use the following commands to inspect the running kiosk:

```bash
# Memory and running processes
free -h
ps -eo pid,comm,rss --sort=-rss | head -20
ps -ef | grep -E 'Xorg|openbox|chromium' | grep -v grep

# HDMI and Xorg
cat /sys/class/drm/card0-HDMI-A-1/status
cat /home/tom/.local/share/xorg/Xorg.0.log

# Bluetooth, Cron, and the weekly reboot
systemctl is-enabled bluetooth.service
systemctl is-active bluetooth.service
systemctl is-active cron.service
cat /etc/cron.d/nightscout-reboot

# Local Nightscout hostname resolution (replace with your host)
getent hosts nightscout.lan.local
```

#### 2.3.11.1 Common troubleshooting cases

- **Black screen after `startx`:** check `/etc/X11/xorg.conf.d/20-waveshare-fbdev.conf`.
- **Chromium does not start:** check `command -v chromium`, the kiosk directory, and ownership of files under `/home/tom`.
- **1 GB memory warning:** verify `--no-memcheck` is present in the Chromium launcher.
- **Mouse cursor visible:** verify `-nocursor` is included in the X11 startup command.
- **Nightscout cannot be reached:** check Wi-Fi, DNS resolution, server availability, and the access token.

### 2.3.12 Security notes

- Use a Nightscout access token with the minimum required permissions.
- Never publish real tokens, passwords, or patient data in a public repository.
- Keep Chromium's sandbox enabled; do not add `--no-sandbox`.
- Restrict SSH access to trusted devices and keep the OS updated.
- Use console autologin only where physical access to the Raspberry Pi is trusted.

### 2.3.13 Configuration status

| Component | Status |
|---|---|
| Raspberry Pi OS Trixie Lite (32-bit), Waveshare HDMI 1024×600 | Tested |
| Xorg with `fbdev` | Tested; fixed the black screen |
| Openbox and Chromium Nightscout kiosk | Tested |
| Chromium low-memory warning with `--no-memcheck` | Resolved |
| Console autologin and kiosk autostart | Configured; verify after reboot |
| Hidden cursor using `-nocursor` | Configured |
| Bluetooth disabled | Confirmed |
| Weekly Monday 02:00 reboot | Configured; Cron active |
| Swap temporarily disabled | Confirmed |
| Swap permanently disabled | Not confirmed |

## 2.4 Result

### 2.4.1 7 inch diplay

![Empty Nightscout](https://github.com/tslenter/CGM-Monitor-Dashboard/blob/main/images/7%20inch%20dashboard.png)

### 2.4.2 27 inch diplay

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

## 5.7 Used Resources
```
https://www.waveshare.com/wiki/Zero-DISP-7A
https://read.sanjaysikdar.dev/raspberry-pi-zero-2w-kiosk
http://www.undr.com/understatement/2024/free_kiosk_dashboard_setup_for_raspi/index.html
```

