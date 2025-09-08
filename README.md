Mobile Network Traffic

This guide provides a recipe for setting up a man-in-the-middle (MITM) attack to view an emulated Android device’s encrypted HTTPS requests and responses in clear text.
This setup is useful for security research, journalism, and troubleshooting mobile advertising.

🚀 Waydroid Setup
Install Waydroid

Waydroid is an Android emulator for Linux. It requires Wayland.

waydroid init -s GAPPS


Set OTA values for GAPPS:

System OTA: https://ota.waydro.id/system

Vendor OTA: https://ota.waydro.id/vendor

Troubleshooting Waydroid
Debugging Errors
sudo waydroid shell
logcat

APK Installation Fails

On x86 CPUs (Intel/AMD), APK installs may fail. Use CasualSnek’s Waydroid Script
 to install Magisk:

sudo venv/bin/python3 main.py install libndk     # Native Android libraries
sudo venv/bin/python3 main.py install libhoudini # ARM translation layer

Restarting Waydroid
sudo waydroid session stop
sudo waydroid container stop
sudo systemctl stop waydroid-container.service
sudo systemctl start waydroid-container.service

Networking Issues

Allow Waydroid through firewall → Docs

Docker conflicts → remove docker0 interface.

Fedora nftables issues → set LXC_USE_NFT="false".

🔧 Magisk Setup
Install Magisk

Instead of CasualSnek’s script, use mistrmochov/magiskWaydroid
:

git clone https://github.com/mistrmochov/magiskWaydroid
cd magiskWaydroid
./magisk install --modules


This installs Magisk + LSPosed + Busybox.

Install Magisk Modules

Download .zip module in Waydroid (use Firefox, not default browser).

Open Magisk → Modules → Install from Storage.

Select the .zip file.

Recommended Module

pwnlogs/cert-fixer
 → moves user CA certs into system CA store (bypassing many app restrictions).

🕵️ MITM Setup
Install MITMProxy

From source:

git clone https://github.com/mitmproxy/mitmproxy ~/mitmproxy
python3.11 -m venv ~/mitm-env
source ~/mitm-env/bin/activate
pip install -e ".[dev]"


From binary:

tar -xf archive.tar.gz
mv mitmweb mitmdump mitmproxy /usr/local/bin/
chmod +x /usr/local/bin/mitm*

Configure Transparent Proxy

Use the included proxysetup.sh (example for port 8080):

#!/bin/bash
sudo iptables -t nat -A PREROUTING -i waydroid0 -p tcp --dport 80  -j REDIRECT --to-port $port
sudo iptables -t nat -A PREROUTING -i waydroid0 -p tcp --dport 443 -j REDIRECT --to-port $port
sudo ip6tables -t nat -A PREROUTING -i waydroid0 -p tcp --dport 80  -j REDIRECT --to-port $port
sudo ip6tables -t nat -A PREROUTING -i waydroid0 -p tcp --dport 443 -j REDIRECT --to-port $port
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1
sudo sysctl -w net.ipv4.conf.all.send_redirects=0
mitmweb --mode transparent --showhost --set block_global=false


Access MITM dashboard:
👉 http://127.0.0.1:8081

Verify Setup

Open Waydroid Browser → http://mitm.it
 (HTTP only).

Install CA certificates via Magisk modules (not manually).

Relaunch Waydroid + MITM.

Now, you should see cleartext HTTPS requests inside MITM.

⚠️ Play Certification Issues

Google enforces Play Certification on many apps. Without it, apps may reject connections.

Check Android ID:

sudo waydroid shell
ANDROID_RUNTIME_ROOT=/apex/com.android.runtime \
ANDROID_DATA=/data \
ANDROID_TZDATA_ROOT=/apex/com.android.tzdata \
ANDROID_I18N_ROOT=/apex/com.android.i18n \
sqlite3 /data/data/com.google.android.gsf/databases/gservices.db \
"select * from main where name = \"android_id\";"


Docs: Google Play Certification FAQ

🔑 Certificate Handling in Android

Android supports three CA levels:

User CA → Installed via settings, apps may ignore.

System CA → Root CA, trusted by most apps (requires root/Magisk).

App-Pinned CA → Apps hardcode certificates; MITM bypass requires unpinning.

Useful tool: mitmproxy/android-unpinner

🛠️ Alternatives

Charles Proxy → GUI, shareware (30-day trial).

Burp Suite → Community Edition available, includes CA handling.

✅ Quick Start
./proxysetup.sh -w   # Start transparent proxy
# OR
./tmuxlauncher.sh    # Launch in tmux session


Monitor traffic:
👉 http://127.0.0.1:8081/#/flows

Then open target apps in Waydroid to view live traffic.
