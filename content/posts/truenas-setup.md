+++
date = "2024-12-22T13:29:29Z"
draft = false
title = 'Truenas Setup'
tags = ["truenas","docker"]
+++

## Prerequisites 
Being an `archlinux` user means dependencies are for `arch` but this blog post should work with any `nix` distribution.
- [truenas](https://www.truenas.com/download-truenas-core)
- [docker](https://get.docker.com)
- [portainer](https://docs.portainer.io/start/install-ce/server/docker/linux)

## Background
I have a Proxmox setup on an old Xeon server with TrueNAS Scale running as VM. But with my new WTR PRO, I wanted to explore using `docker` for my containers instead of LXCs (Linux Containers). I am keeping the Proxmox running as backup and planned on `terraforming` the Proxmox installation in the future.

![wtrpro](../wtrpro.webp)

I installed TrueNAS ElectricEel-24.10.1 on this bad boy and will document the `docker` containers I'm going to deploy.

## WARNING:
Do not enable `apt` if you don't know what you're doing. It can destroy the appliance.

```
$ sudo mount -o remount,rw 'boot-pool/ROOT/24.10.1/usr'
$ sudo chmod +x /usr/bin/apt*
$ sudo chmod +x /usr/bin/dpkg
```

## Install cockpit
```
$ sudo nvim /etc/apt/sources.list.d/backports.list
deb http://deb.debian.org/debian bookworm-backports main

$ sudo apt install -t bookworm-backports cockpit --no-install-recommends
$ sudo systemctl start cockpit
$ wget https://github.com/45Drives/cockpit-navigator/releases/download/v0.5.10/cockpit-navigator_0.5.10-1focal_all.deb
$ sudo apt install ./cockpit-navigator_0.5.10-1focal_all.deb
```

### Portainer
I installed Portainer Community Edition from the community Train. It's available by default and I only need to click Discover Apps to reveal itself. It's a box-standard setup where you need to fill up the UI for Portainer, Network, Storage and Resources Configuration. It's running on 2 CPUs with 4096 MB of RAM. The rest of the containers are `stack` deployment within `portainer`.

### Authelia
I have `configuration.yml`, `users_database.yml` and `notification.txt` on the mounted volume so its persistent.
```
services:
 authelia:
   image: authelia/authelia
   container_name: authelia
   restart: always
   volumes:
     - /mnt/fast/authelia/config:/config
   ports:
     - 9091:9091
   environment:
     - TZ=Europe/London
```

### Gitea
```
services:
 server:
   image: docker.io/gitea/gitea
   container_name: gitea
   environment:
     - USER_UID=568
     - USER_GID=568
   restart: always
   volumes:
     - /mnt/fast/gitea:/data
     - /etc/timezone:/etc/timezone:ro
     - /etc/localtime:/etc/localtime:ro
   ports:
     - 3000:3000
     - 222:22
```

I have to use a custom `config` file since it's running on a non-standard SSH port.
```
❯ cat ~/.ssh/config
Host 192.168.x.x
  Port 222
  User git
```  

### Heimdall
```
services:
  heimdall:
    image: lscr.io/linuxserver/heimdall:latest
    container_name: heimdall
    environment:
      - PUID=999
      - PGID=999
      - TZ=Europe/Lond
    volumes:
      - /mnt/fast/heimdall/config:/config
    ports:
      - 7980:80
      - 7943:443
    restart: unless-stopped
```

### HomeAssistant
```
services:
  homeassistant:
    container_name: homeassistant
    image: lscr.io/linuxserver/homeassistant:latest
    network_mode: host
    environment:
      - PUID=999
      - PGID=999
      - TZ=Europe/London
    ports:
      - 8123:8123
    volumes:
      - /mnt/fast/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
    restart: always
```

### Immich
This is the most challenging container to setup. In my Proxmox, I have a bare-metal installation of this since I don't have success mounting volumes inside the LXC container.
```
services:
  immich-server:
    container_name: immich-server
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    volumes:
      - ${UPLOAD_LOCATION}:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    env_file:
      - stack.env
    ports:
      - '2283:2283'
    depends_on:
      - redis
      - database
    restart: always
    user: 3002:568
    healthcheck:
      disable: false

  immich-machine-learning:
    container_name: immich-machine-learning
    image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}
    volumes:
      - model-cache:/cache
    env_file:
      - stack.env
    restart: always
    user: 568:568
    healthcheck:
      disable: false

  redis:
    container_name: immich-redis
    image: docker.io/redis:6.2-alpine@sha256:eaba718fecd1196d88533de7ba49bf903ad33664a92debb24660a922ecd9cac8
    healthcheck:
      test: redis-cli ping || exit 1
    restart: always

  database:
    container_name: immich-postgres
    image: docker.io/tensorchord/pgvecto-rs:pg14-v0.2.0@sha256:90724186f0a3517cf6914295b5ab410db9ce23190a2d9d0b9dd6463e3fa298f0
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_INITDB_ARGS: '--data-checksums'
    volumes:
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
    healthcheck:
      test: >-
        pg_isready --dbname="$${POSTGRES_DB}" --username="$${POSTGRES_USER}" || exit 1;
        Chksum="$$(psql --dbname="$${POSTGRES_DB}" --username="$${POSTGRES_USER}" --tuples-only --no-align
        --command='SELECT COALESCE(SUM(checksum_failures), 0) FROM pg_stat_database')";
        echo "checksum failure count is $$Chksum";
        [ "$$Chksum" = '0' ] || exit 1
      interval: 5m
      start_interval: 30s
      start_period: 5m
    command: >-
      postgres
      -c shared_preload_libraries=vectors.so
      -c 'search_path="$$user", public, vectors'
      -c logging_collector=on
      -c max_wal_size=2GB
      -c shared_buffers=512MB
      -c wal_compression=on
    restart: always

volumes:
  model-cache:
```

I have the following on my `stack.env` deployed using Portainer's web editor.
```
# You can find documentation for all the supported env variables at https://immich.app/docs/install/environment-variables
# The location where your uploaded files are stored
UPLOAD_LOCATION=/mnt/tank0/immich

# The location where your database files are stored
DB_DATA_LOCATION=/mnt/fast/postgres14

TZ=Europe/London
IMMICH_VERSION=v1.120.1
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
DB_PASSWORD=YOUR_STRONG_PASSWORD
```

### Navidrome
```
services:
  navidrome:
    image: deluan/navidrome:latest
    user: 568:568
    ports:
      - 4533:4533
    restart: unless-stopped
    environment:
      ND_SCANSCHEDULE: 1h
      ND_LOGLEVEL: info  
      ND_SESSIONTIMEOUT: 24h
      ND_BASEURL: ""
    volumes:
      - /mnt/fast/navidrome/data:/data
      - /mnt/tank0/data/media/music:/music:ro
```

## Nextcloud
```
services:
  nextcloud:
    image: lscr.io/linuxserver/nextcloud:latest
    container_name: nextcloud
    environment:
      - PUID=568
      - PGID=568
      - TZ=Europe/London
    volumes:
      - /mnt/fast/nextcloud/config:/config
      - /mnt/fast/nextcloud/data:/data
    ports:
      - 4443:443
    restart: unless-stopped
```    

### Nginx Proxy Manager
```
services:
  proxy:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy
    network_mode: bridge
    restart: always
    environment:
      - PUID=568
      - PGID=568
      - TZ=Europe/London
    ports:
      - 8081:81
      - 8443:443
    volumes:
      - /mnt/fast/nginx/data:/data
      - /mnt/fast/nginx/letsencrypt:/etc/letsencrypt

  ddns:
    image: favonia/cloudflare-ddns:latest
    container_name: cloudflare-ddns
    restart: always
    read_only: true 
    cap_drop: [all] 
    security_opt: [no-new-privileges:true]
    environment:
     - CLOUDFLARE_API_TOKEN=GET_THIS_FROM_YOUR_CLOUDFLARE_ACCOUNT
     - DOMAINS=lekat.uk,auth.lekat.uk
     - PROXIED=true
     - IP6_PROVIDER=none
```     

### Pihole
```
networks:
  dns_net:
    driver: bridge
    ipam:
        config:
        - subnet: 172.16.8.0/24

services:
  pihole:
    container_name: pihole
    hostname: pihole
    image: pihole/pihole:latest
    networks:
      dns_net:
        ipv4_address: 172.16.8.7
    ports:
      - 53:53/tcp
      - 53:53/udp
      - 81:80/tcp
    environment:
      TZ: Europe/London
      WEBPASSWORD: STRONG_PASSWORD
      PIHOLE_DNS_: 172.16.8.8#5053
    volumes:
      - /mnt/fast/pihole:/etc/pihole
      - /mnt/fast/pihole/dnsmasq.d:/etc/dnsmasq.d
    restart: unless-stopped

  unbound:
    container_name: unbound
    image: mvance/unbound:latest
    networks:
      dns_net:
        ipv4_address: 172.16.8.8
    volumes:
      - /mnt/fast/pihole/unbound:/opt/unbound/etc/unbound
    ports:
      - 5053:53/tcp
      - 5053:53/udp
    healthcheck:
      test: ["NONE"]
    restart: unless-stopped
```

I edited the `unbound.conf` to remove the zone configurations.
```
    ###########################################################################
    # FORWARD ZONE
    ###########################################################################

    #include: /opt/unbound/etc/unbound/forward-records.conf

    ###########################################################################
    # LOCAL ZONE
    ###########################################################################

    # Include file for local-data and local-data-ptr
    #include: /opt/unbound/etc/unbound/a-records.conf
    #include: /opt/unbound/etc/unbound/srv-records.conf

    ###########################################################################
    # WILDCARD INCLUDE
    ###########################################################################
    #include: "/opt/unbound/etc/unbound/*.conf"
```

### PostgreSQL
```
services:
  db:
    container_name: postgresql
    image: postgres:17.2-bullseye
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=STRONG_PASSWORD
    ports:
      - 5432:5432
    restart: always
    volumes: 
      - /mnt/fast/postgres17:/var/lib/postgresql/data
    command: >-
      bash -c "apt-get update &&
               apt-get install -y postgresql-17-pgvector &&
               docker-entrypoint.sh postgres
               -c shared_preload_libraries=vector
               -c 'search_path=\"$$user\",public,vectors'
               -c logging_collector=on
               -c max_wal_size=2GB
               -c shared_buffers=512MB
               -c wal_compression=on"
```

### Trilium
```
services:
 trilium:
   container_name: trilium
   image: triliumnext/notes:v0.90.4
   restart: always
   environment:
     - TRILIUM_DATA_DIR=/home/node/trilium-data
     - USER_GID=1000
   ports:
     - 8082:8080
   volumes:
     - /mnt/fast/trilium/notes:/home/node/trilium-data
```
     