> v.2026-03-30

An <a href = "https://tech.ebu.ch/docs/events/ibc11-ebutechnical/presentations/ibc11_10things_r128.pdf">EBU R128</a> loudness meter for <a href = "http://en.wikipedia.org/wiki/Livewire_%28networking%29">Axia Livewire</a>, an AoIP standard used in broadcast. The client subscribes to an Axia channel's multicast UDP/RTP stream, computes an EBU R128 short-term loudness measurement, and logs the results to your <a href = "http://graphite.readthedocs.org/en/latest/overview.html">Graphite</a> server via its <a href = "http://graphite.readthedocs.org/en/latest/feeding-carbon.html#the-plaintext-protocol">plaintext protocol</a>. Resolution is one measurement per second.  Very useful for loudness logging, visualization, or putting together a <a href = "https://raw.githubusercontent.com/ykmn/axia-loudness-graphite-client/master/grafana.gif">realtime loudness dashboard</a>.

Usage: `axialufsgraphite <Multicast Livewire IP> <Graphite Server IP> <Graphite Metric>`

Example: `axialufsgraphite 239.192.16.205 127.0.0.1 LUFS.Retro.PGM1`

> [!TIP]
> Graphite default port for plain-text data connection: 2003

### Compilation
```bash
cd ~
git clone https://github.com/ykmn/axia-loudness-graphite-client.git
# ebur128 is included with this repo
# git clone https://github.com/jiixyj/libebur128.git
# cp -r libebur128/ebur128/ axia-loudness-graphite-client/
cd axia-loudness-graphite-client
gcc axialufsgraphite.c ebur128/ebur128.c \
     -o ../axialufsgraphite \
    -I$HOME/axia-loudness-graphite-client/ebur128/ \
    -L$HOME/axia-loudness-graphite-client/ebur128/ -lm

cd ..
sudo chmod +x ./axialufsgraphite
```
### Configure network
```bash
# Enable multicast on Livewire NIC
ip link set dev ens33 multicast on
# Enable multicast on LAN NIC
ip link set dev ens32 multicast off

# Add route: send all multicast to Livewire NIC
ip route add 224.0.0.0/4 dev ens33
# or: send only Livewire multicast to Livewire NIC
sudo route add -net 239.192.0.0 netmask 255.255.0.0 dev ens33

# Create persistant route
sudo nano /etc/netplan/50-cloud-init.yaml
```
```yaml
network:
  version: 2
  ethernets:
    ens33:                    # this is your Livewire NIC
      addresses:
        - 172.22.0.99/24      # static IP + netmask 255.255.255.0
      gateway4: none          # no default gateway
      nameservers: {}         # no DNS servers
      routes:
        - to: 224.0.0.0/4     # full multicast range is reachable 
#        - to: 239.192.0.0/16  # only Liwevire multicast range is reachable
          via: 0.0.0.0        # on‑link (no next‑hop)
          metric: 100
```
```bash
# Run axialufsgraphite and check multicast subscriptions and data arrived:
netstat -gn
```
> [!TIP]
> [Livewire multicast calculator](https://unnamed.media/livewire.html)

### Create startup script
```bash
sudo nano ~/getlufs.sh
```
```ini
#!/bin/bash
killall axialufsgraphite 2>/dev/null || true
#                           multicast addr    graphite  metrics name
# 10004
/home/axia/axialufsgraphite 239.192.39.20     127.0.0.1   LUFS.Novoe.FM          &\
# 10006
/home/axia/axialufsgraphite 239.192.39.22     127.0.0.1   LUFS.Studio21.FM       &\
# 10007
/home/axia/axialufsgraphite 239.192.39.23     127.0.0.1   LUFS.Dorognoe.FM       &\
# 10002
/home/axia/axialufsgraphite 239.192.39.18     127.0.0.1   LUFS.Radio7.FM         &\
# 10003
/home/axia/axialufsgraphite 239.192.39.19     127.0.0.1   LUFS.Retro.FM          &\
# 10001
/home/axia/axialufsgraphite 239.192.39.17     127.0.0.1   LUFS.EuropaPlus.FM     &\
echo "All instances started."
wait
```
```bash
sudo chmod +x ~/getlufs.sh
```

### Create Axia LUFS service
```bash
sudo cat > /etc/systemd/system/axialufs.service
<<'EOF'
[Unit]
Description=Send Livewire LUFS values to Graphite metrics service
After=network.target

[Service]
Type=simple
ExecStart=/home/axia/getlufs.sh
ExecStop=/usr/bin/killall axialufsgraphite
Restart=always
RestartSec=10
User=axia
# Group=axia   # if needed
WorkingDirectory=/home/axia

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now axialufs.service
sudo systemctl start axialufs.service
sudo systemctl status axialufs.service
sudo systemctl stop  axialufs.service
sudo systemctl restart axialufs.service
```

---
# Basic Ubuntu config

### DNS setup
```bash
sudo nano /etc/resolv.conf
```

```ini
nameserver 192.168.0.100
nameserver 8.8.8.8
```
### Host name
```bash
sudo nano /etc/hostname
sudo nano /etc/hosts
# or
sudo hostnamectl set-hostname MyNewServer2
```
### Time zone and clock
```bash
sudo dpkg-reconfigure tzdata
cat /etc/timezone

# Set date-time
# follow the format mmddHHMMyyyy.SS
sudo date 123123592026.00
# Transfer system time to hardware clock
sudo hwclock --localtime --systohc
# Write to hardware clock
sudo hwclock -w
```
### Enable unattended upgrades
```bash
sudo dpkg-reconfigure --priority=low unattended-upgrades
sudo nano /etc/apt/apt.conf.d/10periodic
```

```ini
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
```
### Ubuntu release upgrade
```bash
sudo apt-get install update-manager-core
sudo nano /etc/update-manager/release-upgrades
```

```ini
Prompt=normal   # for odd version
Prompt=lts      # for LTS version
```
```bash
sudo do-release-upgrade -d
```
### System proxy setup
```bash
sudo nano /etc/environment
```

```ini
http_proxy="http://172.16.110.000:10808/"
https_proxy="http://172.16.110.000:10808/"
ftp_proxy="http://172.16.110.000:10808/"
no_proxy="localhost,127.0.0.1,localaddress,yourdomain.local"
HTTP_PROXY="http://172.16.110.000:10808/"
HTTPS_PROXY="http://172.16.110.000:10808/"
FTP_PROXY="http://172.16.110.000:10808/"
NO_PROXY="localhost,127.0.0.1,localaddress,yourdomain.local"
```

### APT proxy setup
```bash
sudo nano /etc/apt/apt.conf.d/90curtin-aptproxy
```

```ini
Acquire::http::Proxy "http://172.16.110.000:10808";
Acquire::https::Proxy "http://172.16.110.000:10808";
```
### Old kernels cleanup
```bash
sudo dpkg -l | grep linux-image

sudo echo $(dpkg --list | grep linux-image | awk '{ print $2 }' | sort -V | sed -n '/'`uname -r`'/q;p') $(dpkg --list | grep linux-headers | awk '{ print $2 }' | sort -V | sed -n '/'"$(uname -r | sed "s/\([0-9.-]*\)-\([^0-9]\+\)/\1/")"'/q;p') | xargs sudo apt-get -y purge

sudo apt purge $(for tag in "linux-image" "linux-headers"; do dpkg-query -W -f'${Package}\n' "$tag-[0-9]*.[0-9]*.[0-9]*" | sort -V | awk 'index($0,c){exit} //' c=$(uname -r | cut -d- -f1,2); done)
```


# Graphite, Carbon and Whisper
### What is it all about

| Component | Role in the stack | Key responsibilities | Typical usage |
|-----------|-------------------|----------------------|---------------|
| **Graphite** | The overall monitoring and graphing system | Collects metrics from applications, stores them, and renders visual graphs via a web UI or allows access to data via its own API | Infrastructure monitoring, performance dashboards, capacity planning |
| **Carbon** | The data‑ingestion and storage backend for Graphite | - **Carbon Daemon** (`carbon-cache`) receives metric data over the **Carbon protocol** (`plain‑text` or `pickle`) and writes it to disk.<br>- Optional **Carbon Relay** forwards data to one or more Carbon instances for load‑balancing or redundancy. | Runs on each host that receives metrics; can be scaled horizontally by adding more carbon daemons or relays. |
| **Whisper** | The time‑series database format used by Carbon | - Stores each metric in its own file.<br>- Fixed‑size circular buffers define *retention policies* (e.g., keep 1‑minute data for 24 h, 5‑minute data for 30 d).<br>- No external DB server; reads/writes are simple file I/O. | Provides efficient, low‑overhead storage for high‑cardinality metric streams. |

### How they fit together  

1. Metric producers (applications, servers, *collectd*, *statsd*, *axialufsgraphite* etc.) send data points to the Carbon daemon using the Carbon protocol.
2. The Carbon daemon writes each point into a Whisper file according to the metric’s *retention schema*.  
3. Graphite reads the Whisper files on demand, aggregates the data, and generates PNG or interactive graphs that can be viewed via its web interface or queried via its API.  

### Common deployment notes  

- Retention schemas are defined in `storage-schemas.conf`; they control how long high‑resolution data is kept before being aggregated to coarser granularity.  
- Because Whisper uses a fixed‑size file per metric, disk usage is predictable, but large numbers of unique metric names can lead to many small files; strategies such as metric name aggregation or using a newer storage backend (e.g., Ceres, CarbonAPI) are sometimes employed.  
- Carbon Relay is useful for high‑throughput environments: producers send to the relay, which distributes to multiple carbon caches, providing redundancy and load distribution. There's no need to use it in current project.

In short, Graphite is the visualization/query layer, Carbon is the data ingest‑and‑store service, and Whisper is the on‑disk time‑series format that actually holds the metric data.

### Installation
```bash
sudo apt update
sudo apt install -y graphite-web graphite-carbon python3-whisper \
                    python3-setuptools python3-twisted
```
### Fix Python 3.12+ compatibility issue
```bash
sudo sed -i 's/import imp/import importlib as imp/g' /usr/lib/python3/dist-packages/carbon/routers.py
sudo sed -i 's/import imp/import importlib as imp/g' /usr/lib/python3/dist-packages/carbon/conf.py
```
### Create folders structure
```bash
sudo mkdir -p /var/log/carbon/carbon-cache-a/
sudo mkdir -p /var/run/carbon/
sudo mkdir -p /var/lib/graphite/whisper/
sudo mkdir -p /opt/graphite/conf
sudo mkdir -p /opt/graphite/storage/log/webapp/
sudo ln -sf /etc/carbon/storage-schemas.conf /opt/graphite/conf/storage-schemas.conf
sudo ln -sf /etc/carbon/storage-aggregation.conf /opt/graphite/conf/storage-aggregation.conf
```
### Set permissions
```bash
sudo chown -R root:root /var/log/carbon/ /var/lib/graphite/whisper/ /var/run/carbon/
sudo chmod -R 777 /var/log/carbon/ /var/run/carbon/
```
### Configure Carbon
```bash
sudo sed -i "s|^USER =.*|USER = root|" /etc/carbon/carbon.conf
sudo sed -i "s|^LOG_DIR =.*|LOG_DIR = /var/log/carbon/|" /etc/carbon/carbon.conf
sudo sed -i "s|^PID_DIR =.*|PID_DIR = /var/run/carbon/|" /etc/carbon/carbon.conf
sudo sed -i "s|^LOCAL_DATA_DIR =.*|LOCAL_DATA_DIR = /var/lib/graphite/whisper/|" /etc/carbon/carbon.conf
sudo sed -i "s|^LINE_RECEIVER_INTERFACE =.*|LINE_RECEIVER_INTERFACE = 0.0.0.0|" /etc/carbon/carbon.conf
sudo sed -i "s|^LINE_RECEIVER_PORT =.*|LINE_RECEIVER_PORT = 2003|" /etc/carbon/carbon.conf
# or use
# sudo nano /etc/carbon/carbon.conf
# 
# [cache]
# USER = root
# LOG_DIR = /var/log/carbon/
# PID_DIR = /var/run/carbon/
# LOCAL_DATA_DIR = /var/lib/graphite/whisper/
# LINE_RECEIVER_INTERFACE = 0.0.0.0
# LINE_RECEIVER_PORT = 2003
```

### Configure Carbon storage schemas
```bash
echo -e "[LUFS]\npattern = ^LUFS\.\nretentions = 1s:48h, 10s:30d, 1m:90d\n" > /tmp/new_schemas.conf
cat /etc/carbon/storage-schemas.conf >> /tmp/new_schemas.conf
sudo mv /tmp/new_schemas.conf /etc/carbon/storage-schemas.conf
sudo cp /etc/carbon/storage-schemas.conf /opt/graphite/conf/storage-schemas.conf
# or
# sudo nano /etc/carbon/storage-schemas.conf
# 
# [LUFS]
# pattern = ^LUFS\.
# retentions = 1s:48h, 10s:30d, 1m:90d
# [default]
# pattern = .*
# retentions = 10s:1d, 1m:30d, 1h:1y
```

> [!IMPORTANT]
> If you have to modify already stored data, use:
> ```bash
> sudo whisper-resize /var/lib/graphite/whisper/test/metric.wsp 10s:1d 1m:30d
> # show data info:
> whisper-info /var/lib/graphite/whisper/LUFS/EuropaPlus/FM.wsp
> ```

### Configure Graphite
```bash
sudo sed -i "s/.*SECRET_KEY =.*/SECRET_KEY = '123asd123'/" /etc/graphite/local_settings.py
sudo sed -i "s/.*ALLOWED_HOSTS =.*/ALLOWED_HOSTS = [ '*' ]/" /etc/graphite/local_settings.py
sudo sed -i "s/.*TIME_ZONE =.*/TIME_ZONE = 'Europe\/Moscow'/" /etc/graphite/local_settings.py
sudo grep -E "SECRET_KEY|ALLOWED_HOSTS|TIME_ZONE" /etc/graphite/local_settings.py
# or
# sudo nano /etc/graphite/local_settings.py
# 
# SECRET_KEY = '123asd123'
# ALLOWED_HOSTS = [ '*' ]
# TIME_ZONE = 'Europe/Moscow'
```
### Configure web interface metadata db for API
```bash
sudo graphite-manage migrate
sudo chown root:root /var/lib/graphite/graphite.db
```
### Configure Carbon Cache as a service
```bash
sudo cat > /etc/systemd/system/carbon-cache.service
<<'EOF'
[Unit]
Description=Graphite Carbon Cache
After=network.target

[Service]
Type=forking
User=root
Group=root
ExecStart=/usr/bin/python3 /opt/graphite/bin/carbon-cache.py --config=/etc/carbon/carbon.conf start
PIDFile=/var/run/carbon/carbon-cache-a.pid
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```
### Start Carbon Cache
```bash
sudo ufw allow 2003/tcp
sudo systemctl daemon-reload
sudo systemctl enable carbon-cache
sudo systemctl restart carbon-cache
```
### Check data arrival
```bash
echo "prod.server.cpu 25 $(date +%s)" | nc -q0 127.0.0.1 2003
echo "test.success 1 $(date +%s)" | nc -q0 127.0.0.1 2003
find /var/lib/graphite/whisper -name "success.wsp"
cat /var/lib/graphite/whisper/prod/server/cpu.wsp
```

### Configure Graphite API as a web-service on :8080
> API will be used by Grafana
```bash
sudo apt update
sudo apt install gunicorn -y
sudo nano /etc/systemd/system/graphite-api.service
sudo cat > /etc/systemd/system/graphite-api.service
<<'EOF'
[Unit]
Description=Graphite API for Grafana (Gunicorn)
After=network.target carbon-cache.service

[Service]
Type=simple
User=root
Group=root
WorkingDirectory=/usr/share/graphite-web
# Running on :8080. If this port is not available, change to :8081
ExecStart=/usr/bin/gunicorn --workers 2 --bind 127.0.0.1:8080 graphite.wsgi:application
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```
### Start Graphite API
```bash
sudo ufw allow 8080/tcp
sudo systemctl daemon-reload
sudo systemctl enable graphite-api
sudo systemctl restart graphite-api
sudo ss -tlpn | grep 8080
```

> [!TIP]
> You should get Graphite API web-interface on http://localhost:8080

### Graphite settings summary:
- **Graphite settings:** `/etc/graphite/local_settings.py`
- **Whisper databases location:** `/var/lib/graphite/whisper`
Folder permissions are for Carbon user (check Carbon Cache servce)
- **Graphite log:** `/var/log/graphite/info.log`
- **Carbon log:** `/var/log/carbon/carbon-cache-a/listener.log`

# Grafana installation 
https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/

```bash
sudo apt install -y apt-transport-https wget gnupg
```
### Install repo
```bash
sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/grafana.asc https://apt.grafana.com/gpg-full.key
sudo chmod 644 /etc/apt/keyrings/grafana.asc
echo "deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```
### Download the new key
```bash
sudo wget -q -O /etc/apt/keyrings/grafana.key https://apt.grafana.com/gpg.key
echo "deb [signed-by=/etc/apt/keyrings/grafana.key] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
```
### Update your repositories list and install latest release
> [!WARNING]
> You may want to install Grafana without repo signature check, as it may be broken.
 
```bash
sudo apt update -o Acquire::AllowInsecureRepositories=true
sudo apt install grafana -o APT::Get::AllowUnauthenticated=true
```
### Configure Grafana for Active Directory
> https://grafana.com/docs/grafana/latest/setup-grafana/configure-access/configure-authentication/ldap/
```bash
sudo cp /etc/grafana/ldap.toml /etc/grafana/ldap.toml.bak
# copy preconfigured LDAP file, skip if you have not
sudo cp /etc/grafana/ad.toml /etc/grafana/ldap.toml

sudo nano /etc/grafana/grafana.ini
```
```ini
#################################### Basic Auth ##########################
[auth.basic]
enabled = true

#################################### Auth LDAP ##########################
[auth.ldap]
enabled = false
config_file = /etc/grafana/ldap.toml
allow_sign_up = true
```
### Enable Grafana service
```bash
sudo ufw allow 3000/tcp
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl status grafana-server
sudo systemctl enable grafana-server.service
```

> [!TIP]  
> Graphana should be available at http://localhost:3000/
> Defalut credentials: admin:admin

### Connect Graphite data source to Grafana

Open Grafana web iu:

1. **Connections** -> **Data Sources** -> **Add data source**.
2. Select **Graphite**.
3. Enter `http://127.0.0.1:8080` to **URL** (if you have local Grafana, or use `http://your-server:8080`).
4. Select **Default** in **Graphite type**.
5. Click **Save & Test**.

If everything's good, Grafana responds: _"Data source is working"_.

### Visualisation
Open Grafana web iu, create a New Dashboard, select Data Source, select Visualization type (Time Series, Bar Chart etc.), select metric in Queries span. You should see your test metrics tree you sent before (`test` -> `success`) in **Series** field.

![](grafana.gif)