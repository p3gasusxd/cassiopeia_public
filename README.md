# Cassiopeia

Guide to recreating my server

## Description

I had zero experience when I started and learned everything by googling, spending time on youtube, reddit and in documentations. Through hours and days of trial&error, I made lots of mistakes. Now, in case of disaster I will use the scripts in this repository myself to get up and running again. I am documenting this because I haven't found a single source online that provides all necessary information to get up and running. Also, lot's of things have been carefully chosen after testing alternatives. You can save lots of time with this guide! :)

## Getting Started

### Dependencies

* [Ubuntu Server](https://ubuntu.com/download/server)
* [Your Own Google Domain](https://domains.google.com/registrar/)

### Initial Setup

#### Bios Settings

* tweak fan curves
* adjust power consumption
* set to onboard graphics
* enable wake on lan

#### Ubuntu Install

* English language
* English keyboard
* Automatic (DHCP) ip
* Default archive mirror
* I installed linux on a boot ssd, no lvm
* Default drive setup
* Install openssh, no import
* No snaps

#### Router Configuration

* Set static ip to server
* Forward ssh port 22 to server ip

#### Domain Name

* 

#### Cloudflare

*

## Ubunutu Setup

### Static IP

edit ip config
```
sudo nano /etc/netplan/01-netcfg.yaml
```
append network config
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0: #Edit this line according to your network interface name you just saw.
      dhcp4: no
      addresses:
        - IP/24
      gateway4: GATEWAY
      nameservers:
        addresses:
          - DNS
```
apply new config
```
sudo netplan apply
```

### SSH

#### Change SSH Port

edit ssh config
```
sudo nano /etc/ssh/sshd_config
```
uncomment, change port #
```yaml
Port #
```
*Note: Make sure to update the port forwarding to match*

restart ssh service
```
sudo systemctl restart ssh.service
```

### Connect to your server

On the pc you're trying to remote in with,

setup alias 
```
nano .bashrc
```
append alias (replace UPPERCASE with your info)
```yaml
# Own definitions
alias cassiopeia='ssh -p PORT USER@IP'
```
update
```
. ~/.bashrc
```
run alias
```
SERVERNAME
```
accept fingerprint
```
yes
```
if error
```
ssh-keygen -f "/root/.ssh/known_hosts" -R "[IP]:PORT"
```

### UFW

check ufw status
```
sudo ufw status
```

rule setup
```
sudo ufw reset
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow SSHPORT/tcp comment 'SSH'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'
sudo ufw enable
sudo ufw status
```

### Aliases
edit .bashrc
```
nano .bashrc
```
append aliases
```yaml
# apt update
alias upd='sudo apt update'

# apt upgrade, remove
alias upg='sudo apt upgrade; sudo apt autoremove'

# Tail last 50 lines of docker logs
alias dtail='docker logs -tf --tail='50' '

# Shorthand, customise docker-compose.yaml location as needed
alias dcp='docker-compose -f ~/docker-compose.yaml '

# Remove unused images (useful after an upgrade)
alias dprune='docker image prune'

# Remove unused images, unused networks *and data* (use with care)
alias dprunesys='docker system prune --all'

# Prints the IP, network and listening ports for each container.
alias dcips=$'docker inspect -f \'{{.Name}}-{{range  $k, $v := .NetworkSettings.Networks}}{{$k}}-{{.IPAddress}} {{end}}-{{range $k, $v := .NetworkSettings.Ports}}{{ if not $v }}{{$k}} {{end}}{{end}} -{{range $k, $v := .NetworkSettings.Ports}}{{ if $v }}{{$k}} => {{range . }}{{ .HostIp}}:{{.HostPort}}{{end}}{{end}} {{end}}\' $(docker ps -aq) | column -t -s-'
```
reload aliases
```
. ~/.bashrc
```

### Remove Snaps
check for unwanted snaps
```
snap list
```
remove unwanted snaps & snapd
```
sudo snap remove lxd
sudo snap remove core20
sudo snap remove snapd
```
purge snapd
```
sudo apt purge snapd
```
remove snap home
```
sudo rm -rf ~/snap
```

Now's a good time to udpate!

### Fail2ban

install
```
sudo apt install fail2ban -y
```
edit config
```
sudo nano /etc/fail2ban/fail2ban.conf
```
append sshd jail
```yaml
[sshd]
enabled = true
port = PORT
filter = sshd
logpath = /var/log.auth.log
maxretry = 3
bantime = 3600
```
restart fail2ban
```
sudo service fail2ban restart
```
check sshd jail
```
sudo fail2ban-client status sshd
```

### Timezone

https://en.wikipedia.org/wiki/List_of_tz_database_time_zones

list timezones
```
timedatectl list-timezones
```
set timezone
```
sudo timedatectl set-timezone EST5EDT
```
verify
```
timedatectl
```

### Glances *(optional)*

install
```
sudo apt install glances -y
```
run glance
```
glances
```

## DRIVE SETUP

### Identify Drives

list drives
```
ls /dev/disk/by-id
```
map drive (replace ID) *repeat for each storage drive*
```
ls -la /dev/disk/by-id/ID

```
output (sdX = drive identity)

### Partition

start partition *repeat for each storage drive*
```
sudo gdisk /dev/sda
```
output
```
Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present
```
partition sequence
```
o y n default default default default w y
```

### Filesystem

create ext4 (replace X)*repeat for each storage drive*
```
sudo mkfs.ext4 /dev/sda1
```

### Mountpoints

make directories *adjust # based on your setup*
* disks - your data drives
* parity - stores snapshots of drives
* storage - mergerfs drive of disks
```
sudo mkdir /mnt/disk{1,2,...}
sudo mkdir /mnt/parity{1,2,...}
sudo mkdir /mnt/storage
```

### MergerFS

install mergerfs
```
sudo apt install mergerfs -y
```

#### Fstab

list drive UUID
```
ls -l /dev/disk/by-uuid
```
edit fstab entries
```
sudo nano /etc/fstab
```
append your mountpoints *adjust # based on your setup*
```yaml
# disks
UUID=ID /mnt/disk# ext4 defaults 0 0

# parity
UUID=ID /mnt/parity# ext4 defaults 0 0

# storage
/mnt/disk* /mnt/storage fuse.mergerfs defaults,nonempty,allow_other,use_ino,cache.files=off,moveonenospc=true,dropcacheonclose=true,minfreespace=200G,fsname=mergerfs 0 0
```
reload fstab
```
sudo mount -a
```

### Tree (optional)

install
```
apt install tree
```
verify drive setup
```
tree PATH
```

### SNAPRAID

install snapraid
```
sudo apt install snapraid
```
edit config
```
sudo nano /etc/snapraid.conf
```
append config *adjust to your setup*
```yaml
# Example configuration for snapraid

# Defines the file to use as parity storage
# It must NOT be in a data disk
# Format: "parity FILE_PATH"
parity /mnt/parity1/snapraid.parity

# Defines the files to use as content list
# You can use multiple specification to store more copies
# You must have least one copy for each parity file plus one. Some more don't hurt
# They can be in the disks used for data, parity or boot,
# but each file must be in a different disk
# Format: "content FILE_PATH"
content /var/snapraid.content
content /mnt/disk#/snapraid.content

# Defines the data disks to use
# The order is relevant for parity, do not change it
# Format: "disk DISK_NAME DISK_MOUNT_POINT"
disk data# /mnt/disk#/

# Excludes hidden files and directories (uncomment to enable).
#nohidden

# Defines files and directories to exclude
# Remember that all the paths are relative at the mount points
# Format: "exclude FILE"
# Format: "exclude DIR/"
# Format: "exclude /PATH/FILE"
# Format: "exclude /PATH/DIR/"

exclude /.lost+found/
exclude /.snapshots/
exclude *.unrecoverable
exclude /tmp/
```
snyc drives
```
sudo snapraid sync
```

#### Snapraid-Runner (recommended)

install snapraid-runner
```
sudo git clone https://github.com/Chronial/snapraid-runner.git /opt/snapraid-runner
```
edit config
```
sudo nano /opt/snapraid-runner/snapraid-runner.conf
```
append config
* for gmail... SMTP password is the same as your web password, host smtp.gmail.com, port 465, SSL true *
```yaml
[snapraid]
; path to the snapraid executable (e.g. /bin/snapraid)
executable = /bin/snapraid
; path to the snapraid config to be used
config = /etc/snapraid.conf
; abort operation if there are more deletes than this, set to -1 to disable
deletethreshold = 250
; if you want touch to be ran each time
touch = true

[logging]
; logfile to write to, leave empty to disable
file = snapraid.log
; maximum logfile size in KiB, leave empty for infinite
maxsize = 5000

[email]
; when to send an email, comma-separated list of [success, error]
sendon = success,error
; set to false to get full programm output via email
short = true
subject = [SnapRAID] Status Report:
from = EMAIL
to = EMAIL
; maximum email size in KiB
maxsize = 500

[smtp]
host = SMPTPHOST
; leave empty for default port, gmail 465
port = 465
; set to "true" to activate
ssl = true
tls = false
user = 
password = 

[scrub]
; set to true to run scrub after sync
enabled = true
; scrub plan - either a percentage or one of [bad, new, full]
plan = 22
; minimum block age (in days) for scrubbing. Only used with percentage plans
older-than = 8
```
create crontab
```
sudo crontab -e
```
append snapraid-runner
```yaml
# snapraid-runner
0 1 * * * python3 /opt/snapraid-runner/snapraid-runner.py -c /opt/snapraid-runner/snapraid-runner.conf
```

### SAMBA

install samba
```
sudo apt install samba -y
```
create samba user
```
sudo smbpasswd -a USER
```
edit samba config
```
sudo nano /etc/samba/smb.conf
```
append public storage
```yaml
[public]
  path = /mnt/storage
  writable = yes
  read only = no
  guest ok = yes
  guest only = yes
```
append private storage
```yaml
[private]
  path = /mnt/storage
  writable = yes
  read only = no
  guest ok = no
  valid users = @USER
  inherit permissions = yes
```
grant public access
```
sudo chgrp -R USER /mnt/storage
sudo chmod 2770 /mnt/storage
```
grant private access
```
sudo chgrp -R USER /mnt/storage
sudo chmod 2775 /mnt/storage
```
ufw
```
sudo ufw allow 137/udp comment 'SAMBA'
sudo ufw allow 138/udp comment 'SAMBA'
sudo ufw allow 139/tcp comment 'SAMBA'
sudo ufw allow 445/tcp comment 'SAMBA'
sudo ufw reload
```
restart samba
```
sudo service smbd restart
```

## Docker Containers

### Docker Setup

#### Docker
https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-22-04

install prereq and docker
```
sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
apt-cache policy docker-ce
sudo apt install docker-ce
```
enable and start docker
```
sudo systemctl enable docker
```
configure use
```
sudo usermod -aG docker pegasus
su - pegasus
```

#### Docker-Compose

install (make sure to update to latest version)
```
sudo mkdir -p ~/.docker/cli-plugins/
sudo curl -SL https://github.com/docker/compose/releases/download/v2.11.2/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose
sudo chmod +x ~/.docker/cli-plugins/docker-compose
docker compose version
```
edit compose file
```
nano docker-compose.yml
```
append version
```yaml
version: "3.8"
services:
```
run compose file
```
docker compose up -d
```
stop containers
```
docker compose down
```
update containers
```
docker compose pull
docker compose up -d --remove-orphans
```

### Traefik

https://i12bretro.github.io/tutorials/0758.html

setup config file
```
mkdir /docker/traefik -p
```
edit config files
```
nano /docker/config/traefik/traefik.yml
```
append config
```yaml
providers:
  docker:
    defaultRule: Host(`{{ trimPrefix `/` .Name }}.DOMAIN`)
api:
  insecure: true
```

edit compose file
```
nano docker-compose.yml
```
append to compose file
```yaml
  traefik:
    image: traefik:latest
    container_name: traefik
    ports:
      - 8080:8080
      - 443:443
      - 80:80
    volumes:
      - ~/docker/config/traefik/traefik.yml:/etc/traefik/traefik.yml
      - /var/run/docker.sock:/var/run/docker.sock
```
ufw
```
ufw allow 8080/tcp comment 'TRAEFIK'
ufw reload
```
enable other containers
```yaml
    labels:
```

### Portainer

append to compose file
```yaml
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ~/docker/portainer/data:/data
    ports:
      - 9000:9000
```
ufw
```
ufw allow 9000/tcp comment 'PORTAINER'
ufw reload
```
### AdGuard

### Media Server Stack

https://wiki.servarr.com/docker-guide
https://github.com/17hoehbr/automated-plex-guide

#### Setup
create users
```
sudo useradd sonarr -u 13005
sudo useradd radarr -u 13006
sudo useradd prowlarr -u 13004
sudo useradd qbittorrent -u 13002
```
create group
```
sudo groupadd arr -g 13000
```
configure group
```
sudo usermod -aG arr sonarr
sudo usermod -aG arr radarr
sudo usermod -aG arr prowlarr
sudo usermod -aG arr qbittorrent
```

create directories
```
sudo mkdir /docker
sudo mkdir /docker/{sonarr,radarr,prowlarr,qbittorrent}
sudo mkdir /mnt/disk{1,2,3}/{anime,movies,tv}
sudo mkdir /data
sudo mkdir /data/torrents
```
set permissions
```
sudo chmod -R 775 /data/torrents
sudo chmod -R 775 /mnt/storage
sudo chgrp -R pegasus:arr /data/torrents
sudo chgrp -R pegasus:arr /mnt/storage/anime
sudo chgrp -R pegasus:arr /mnt/storage/tv
sudo chgrp -R pegasus:arr /mnt/storage/movies
sudo chown -R sonarr:arr /docker/sonarr
sudo chown -R radarr:arr /docker/radarr
sudo chown -R prowlarr:arr /docker/prowlarr
sudo chown -R qbittorrent:arr /docker/qbittorrent
```

#### Plex

https://www.plex.tv/claim/

append to compose file
```yaml
  plex:
    image: linuxserver/plex:latest
    container_name: plex
    network_mode: host
    environment:
      - PUID=1000
      - PGID=13000
      - VERSION=docker
      - PLEX_CLAIM= #optional
    devices:
      - /dev/dri:/dev/dri
    volumes:
      - /docker/plex:/config
      - /mnt/storage:/storage
    ports:
      - 32400:32400/tcp
    restart: unless-stopped
```
ufw
```
sudo ufw allow 32400/tcp comment 'PLEX'
sudo ufw reload
```
anime scanners
```
sudo mkdir -p '/docker/plex/Library/Application Support/Plex Media Server/Scanners/Series'
sudo wget -O '/docker/plex/Library/Application Support/Plex Media Server/Scanners/Series/Absolute Series Scanner.py' https://raw.githubusercontent.com/ZeroQI/Absolute-Series-Scanner/master/Scanners/Series/Absolute%20Series%20Scanner.py
cd '/docker/plex/Library/Application Support/Plex Media Server/Plug-ins'
```

#### Tautulli *optional*

append to compose file
```yaml
  tautulli:
    image: linuxserver/tautulli:latest
    container_name: tautulli
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=EST5EDT
    volumes:
      - /docker/tautulli:/config
    ports:
      - 8181:8181
    depends_on:
      - plex
    restart: unless-stopped
```
ufw
```
ufw allow 8181/tcp comment 'TAUTULLI'
ufw reload
```

#### Qbittorrent

append to compose file
```yaml	  
  qbittorrent:
    image: linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=13007
      - PGID=13000
      - UMASK=002
      - TZ=EST5EDT
      - WEBUI_PORT=8681
    volumes:
      - /docker/qbittorrent:/config
      - /data/torrents:/data/torrents
    ports:
      - 6881:6881
      - 6881:6881/udp
      - 8681:8681
    depends_on:
      - plex
    restart: unless-stopped
```
ufw
```
ufw allow 8080/tcp comment 'QBITTORRENT'
ufw allow 6881/tcp comment 'QBITTORRENT'
ufw allow 6881/udp comment 'QBITTORRENT'
ufw reload
```
default login
```
admin
adminadmin
```

#### Prowlarr

append to compose file
```yaml
  prowlarr:
    image: linuxserver/prowlarr:develop
    container_name: prowlarr
    environment:
      - PUID=13006
      - PGID=13000
      - UMASK=002
      - TZ=EST5EDT
    volumes:
      - /docker/prowlarr:/config
    ports:
      - 9696:9696
    restart: unless-stopped
```
ufw
```
ufw allow 9696/tcp comment 'PROWLARR'
ufw reload
```

#### Sonarr

append to compose file
```yaml
  sonarr:
    image: linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=13001
      - PGID=13000
      - UMASK=002
      - TZ=EST5EDT
    volumes:
      - /docker/sonarr:/config
      - /mnt/storage:/data/torrents
    ports:
      - 8989:8989
    restart: unless-stopped
```
ufw
```
ufw allow 8989/tcp comment 'SONARR'
ufw reload
```

#### Radarr

append to compose file
```yaml
  radarr:
    image: linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=13002
      - PGID=13000
      - UMASK=002
      - TZ=EST5EDT
    volumes:
      - /docker/radarr:/config
      - /mnt/storage:/storage
      - /data/torrents:/data/torrents
    ports:
      - 7878:7878
    restart: unless-stopped
```
ufw
```
ufw allow 7878/tcp comment 'RADARR'
ufw reload
```


### Lancache Stack

https://lancache.net/
https://www.reddit.com/r/selfhosted/comments/sp38c9/two_dns_server_on_one_physical_server/


edit .env
```
nano .env
```
append
```
##LANCACHE

## Set this to true if you're using a load balancer, or set it to false if you're using seperate IPs for each service.
## If you're using monolithic (the default), leave this set to true
USE_GENERIC_CACHE=true

## IP addresses that the lancache monolithic instance is reachable on
## Specify one or more IPs, space separated - these will be used when resolving DNS hostnames through lancachenet-dns. Multiple IPs can improve cache priming performance for some services (e.g. Steam)
## Note: This setting only affects DNS, monolithic and sniproxy will still bind to all IPs by default
LANCACHE_IP=IP2

## IP address on the host that the DNS server should bind to
DNS_BIND_IP=IP1

## DNS Resolution for forwarded DNS lookups
UPSTREAM_DNS=DNS

## Storage path for the cached data
## Note that by default, this will be a folder relative to the docker-compose.yml file
CACHE_ROOT=/data

## Change this to customise the size of the disk cache (default 2000000m)
## If you have more storage, you'll likely want to increase this
## The cache server will prune content on a least-recently-used basis if it
## starts approaching this limit.
## Set this to a little bit less than your actual available space 
CACHE_DISK_SIZE=250000m

## Change this to allow sufficient index memory for the nginx cache manager (default 500m)
## We recommend 250m of index memory per 1TB of CACHE_DISK_SIZE 
CACHE_INDEX_SIZE=125m

## Change this to limit the maximum age of cached content (default 3650d)
CACHE_MAX_AGE=90d

## Set the timezone for the docker containers, useful for correct timestamps on logs (default Europe/London)
## Formatted as tz database names. Example: Europe/Oslo or America/Los_Angeles
TZ=EST5EDT
```
append to compose file
```yaml
  lancache:
    image: lancachenet/lancache-dns:latest
    container_name: lancachenet-dns
    env_file: .env
    networks:
      lancache:
        ipv4_address: IP2
    ports:
      - ${DNS_BIND_IP}:53:53/udp
      - ${DNS_BIND_IP}:53:53/tcp
    restart: always
  monolithic:
    image: lancachenet/monolithic:latest
    container_name: monolithic
    env_file: .env
    volumes:
      - ${CACHE_ROOT}/cache:/data/cache
      - ${CACHE_ROOT}/logs:/data/logs
    networks:
      lancache:
        ipv4_address: IP3
    ports:
      - 80:80/tcp
      - 443:443/tcp
    depends_on: 
      - lancache
    restart: unless-stopped

networks:
  lancache:
    driver: macvlan
    driver_opts:
      parent: enp3s0
    ipam:
      config:
        - subnet: SUBNET/24
          ip_range: IP1/32
          gateway: GATEWAY
```
ufw
```
ufw allow 53/tcp comment 'LANCACHE'
ufw allow 53/udp comment 'LANCACHE'
ufw reload
```
set router DNS Server IP to your IP
test DNS on client
```
ipconfig /flushdns
nslookup steam.cache.lancache.net
nslookup lancache.steamcontent.com
```

## References

https://namingschemes.com/

## Help

## Authors

Brandon Bridges
[@p3gasus_xd](https://twitter.com/p3gasus_xd)

## Version History

* 0.1
    * Initial Release

## License

This project is licensed under the [NAME HERE] License - see the LICENSE.md file for details

## Acknowledgments
