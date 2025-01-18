+++
date = "2025-01-18T13:49:38Z"
draft = false
title = "Servarr Stack"
tags = ["docker","truenas"]
+++

# DISCLAIMER
I am not endorsing piracy nor supporting online illegal activities.

## Prerequisites 
Being an `archlinux` user means dependencies are for `arch` but this blog post should work with any `nix` distribution.
- [truenas](https://wiki.archlinux.org/title/Installation_guide)
- [docker](https://get.docker.com)
- [portainer](https://docs.portainer.io/start/install-ce/server/docker/linux)

## Getting nordpvn private key
Steps were taken from [qdm12
/
gluetun-wiki](https://github.com/qdm12/gluetun-wiki/blob/main/setup/providers/nordvpn.md). Please pick your current VPN provider from the list.
```
❯ sudo pacman -S wireguard-tools
❯ sudo wg show nordlynx private-key
```

## Getting nordvpn public key
```
❯ sudo wg show nordlynx public-key
```

## Getting nordvpn endpoint IP
Please make sure you are connected to the tunnel before clicking this link.

https://api.nordvpn.com/v1/servers/recommendations?&filters%5C%5Bservers_technologies%5C%5D%5C%5Bidentifier%5C%5D=wireguard_udp&limit=1

## gluetun
```
networks:
  media-network:
    name: media-network
    driver: bridge

###########################################################################
##  Docker Compose File:    Gluetun (qmcgaw)
##  Function:               VPN Client
##  Documentation:          https://github.com/qdm12/gluetun-wiki
###########################################################################
services:
  gluetun:
    image: qmcgaw/gluetun:v3.35.0
    container_name: gluetun
    restart: always
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    healthcheck:
      test: ping -c 1 www.google.com || exit 1
      interval: 60s
      retries: 3
      start_period: 20s
      timeout: 10s
    ports:
      - 8488:8888/tcp                         # Gluetun Local Network HTTP proxy
      - 8388:8388/tcp                         # Gluetun Local Network Shadowsocks
      - 8388:8388/udp                         # Gluetun Local Network Shadowsocks
      - ${WEBUI_PORT_QBITTORRENT:?err}:${WEBUI_PORT_QBITTORRENT:?err}   # WebUI Portal: qBittorrent
      - ${QBIT_PORT_TCP:?err}:6881/tcp        # Transmission Torrent Port TCP
      - ${QBIT_PORT_UDP:?err}:6881/udp        # Transmission Torrent Port UDP
      - ${WEBUI_PORT_PROWLARR:?err}:9696      # Prowlarr
      - ${WEBUI_PORT_SABNZBD:?err}:8080       # Sabnzbd
    volumes:
      - ${FOLDER_FOR_CONFIGS:?err}/gluetun:/gluetun
    environment:
      - PUID=${PUID:?err}
      - PGID=${PGID:?err}
      - TZ=${TIMEZONE:?err}
      - VPN_SERVICE_PROVIDER=${VPN_SERVICE_PROVIDER:?err}
      - SERVER_REGIONS=${SERVER_REGIONS:?err}
      - SERVER_COUNTRIES=${SERVER_COUNTRIES}
      - SERVER_CITIES=${SERVER_CITIES}
      - SERVER_HOSTNAMES=${SERVER_HOSTNAMES}
      - FIREWALL_OUTBOUND_SUBNETS=${LOCAL_SUBNET:?err}
      - OPENVPN_CUSTOM_CONFIG=${OPENVPN_CUSTOM_CONFIG}
      - VPN_TYPE=${VPN_TYPE}
      - VPN_ENDPOINT_IP=${VPN_ENDPOINT_IP}
      - VPN_ENDPOINT_PORT=${VPN_ENDPOINT_PORT}
      - WIREGUARD_PUBLIC_KEY=${WIREGUARD_PUBLIC_KEY}
      - WIREGUARD_PRIVATE_KEY=${WIREGUARD_PRIVATE_KEY}
      - WIREGUARD_PRESHARED_KEY=${WIREGUARD_PRESHARED_KEY}
      - WIREGUARD_ADDRESSES=${WIREGUARD_ADDRESSES}
      - HTTPPROXY=on
      - SHADOWSOCKS=on
# NOTE: Gluetun VPN container MUST ONLY connect to the media-network
    networks:
      - media-network
```

## sabnzbd
```
###########################################################################
##  Docker Compose File:  SABnzbd (LinuxServer.io)
##  Function:             Usenet Download Client
##  Documentation:        https://docs.linuxserver.io/images/docker-sabnzbd
###########################################################################
  sabnzbd:
    image: lscr.io/linuxserver/sabnzbd:latest
    container_name: sabnzbd
    restart: unless-stopped
    volumes:
      - ${FOLDER_FOR_CONFIGS:?err}/sabnzbd:/config
      - ${FOLDER_FOR_MEDIA:?err}:/data

    environment:
      - PUID=${PUID:?err}
      - PGID=${PGID:?err}
      - TZ=${TIMEZONE:?err}
    network_mode: service:gluetun
```

## qbittorrent
```
###########################################################################
##  Docker Compose File:  qBittorrent (LinuxServer.io)
##  Function:             Torrent Download Client
##  Documentation:        https://docs.linuxserver.io/images/docker-qbittorrent
###########################################################################
  qbittorrent:
    image: linuxserver/qbittorrent:4.6.0
    container_name: qbittorrent
    restart: unless-stopped
    depends_on:
      gluetun:
        condition: service_healthy
    volumes:
      - ${FOLDER_FOR_CONFIGS:?err}/qbittorrent:/config
      - ${FOLDER_FOR_MEDIA:?err}:/data
    environment:
      - PUID=${PUID:?err}
      - PGID=${PGID:?err}
      - UMASK=${UMASK:?err}
      - TZ=${TIMEZONE:?err}
      - WEBUI_PORT=${WEBUI_PORT_QBITTORRENT:?err}
    healthcheck:
      test: ping -c 1 www.google.com || exit 1
      interval: 60s
      retries: 3
      start_period: 20s
      timeout: 10s
## Do Not Change Network for qBittorrent
## qBittorrent MUST always use a VPN / Secure Internet connection
    network_mode: service:gluetun
```

## jellyfin
```
###########################################################################
##  Docker Compose File:  Jellyfin (LinuxServer.io)
##  Function:             Media Server
##  Documentation:        https://jellyfin.org/docs/general/administration/installing#docker
##                        https://jellyfin.org/docs/general/administration/hardware-acceleration/
###########################################################################
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfinrr
    restart: unless-stopped
    # Hardware Acceleration
    devices:
      - /dev/dri:/dev/dri
    group_add:
      - 107 # render host group id
      - 44 # video host group id
    # End Hardware Acceleration
    volumes:
      - ${FOLDER_FOR_CONFIGS:?err}/jellyfin:/config
      - ${FOLDER_FOR_MEDIA:?err}/data:/data/media
    ports:
      - ${WEBUI_PORT_JELLYFIN:?err}:8096
    environment:
      - PUID=${PUID:?err}
      - PGID=${PGID:?err}
      - UMASK=${UMASK:?err}
      - TZ=${TIMEZONE:?err}
      - ROC_ENABLE_PRE_VEGA=1 # Hardware Acceleration
    networks:
      - media-network
```

## jellyseerr
```
###########################################################################
##  Docker Compose File:  Jellyseerr (fallenbagel)
##  Function:             Media Request Manager
##  Documentation:        https://hub.docker.com/r/fallenbagel/jellyseerr
###########################################################################
  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    restart: unless-stopped
    volumes:
      - ${FOLDER_FOR_CONFIGS:?err}/jellyseerr:/app/config
    ports:
      - ${WEBUI_PORT_JELLYSEERR:?err}:5055
    environment:
      - PUID=${PUID:?err}
      - PGID=${PGID:?err}
      - UMASK=${UMASK:?err}
      - TZ=${TIMEZONE:?err}
    networks:
      - media-network
```

## bazarr
```
###########################################################################
##  Docker Compose File:  Bazarr (LinuxServer.io)
##  Function:             Download subtitles for Radarr and Sonarr
##  Documentation:        https://docs.linuxserver.io/images/docker-bazarr
###########################################################################
  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    restart: unless-stopped
    volumes:
      - ${FOLDER_FOR_CONFIGS:?err}/bazarr:/config
      - ${FOLDER_FOR_MEDIA:?err}:/data
    ports:
      - ${WEBUI_PORT_BAZARR:?err}:6767
    environment:
      - PUID=${PUID:?err}
      - PGID=${PGID:?err}
      - TZ=${TIMEZONE:?err}
    networks:
      - media-network
```

## lidarr
```
###########################################################################
##  Docker Compose File:  Lidarr (LinuxServer.io)
##  Function:             Music Library Manager
##  Documentation:        https://docs.linuxserver.io/images/docker-lidarr
###########################################################################
  lidarr:
    image: lscr.io/linuxserver/lidarr:latest
    container_name: lidarr
    restart: unless-stopped
    volumes:
      - ${FOLDER_FOR_CONFIGS:?err}/lidarr:/config
      - ${FOLDER_FOR_MEDIA:?err}:/data
    ports:
      - ${WEBUI_PORT_LIDARR:?err}:8686
    environment:
      - PUID=${PUID:?err}
      - PGID=${PGID:?err}
      - TZ=${TIMEZONE:?err}
    networks:
      - media-network
```

## prowlarr
```
###########################################################################
##  Docker Compose File:  Prowlarr (LinuxServer.io)
##  Function:             Indexer and Search Manager
##  Documentation:        https://docs.linuxserver.io/images/docker-prowlarr
###########################################################################
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    restart: unless-stopped
    volumes:
      - ${FOLDER_FOR_CONFIGS:?err}/prowlarr:/config
    environment:
      - PUID=${PUID:?err}
      - PGID=${PGID:?err}
      - TZ=${TIMEZONE:?err}
    network_mode: service:gluetun
```

## radarr
```
###########################################################################
##  Docker Compose File:  Radarr (LinuxServer.io)
##  Function:             Movie Library Manager
##  Documentation:        https://docs.linuxserver.io/images/docker-radarr
###########################################################################
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    restart: unless-stopped
    volumes:
      - ${FOLDER_FOR_CONFIGS:?err}/radarr:/config
      - ${FOLDER_FOR_MEDIA:?err}:/data
    ports:
      - ${WEBUI_PORT_RADARR:?err}:7878
    environment:
      - PUID=${PUID:?err}
      - PGID=${PGID:?err}
      - TZ=${TIMEZONE:?err}
    networks:
      - media-network
```

## readarr
```
###########################################################################
##  Docker Compose File:  Sonarr (LinuxServer.io)
##  Function:             Series Library Manager (Books)
##  Documentation:        https://docs.linuxserver.io/images/docker-readarr
###########################################################################
  readarr:
    image: lscr.io/linuxserver/readarr:develop
    container_name: readarr
    environment:
      - PUID=${PUID:?err}
      - PGID=${PGID:?err}
      - TZ=${TIMEZONE:?err}
    volumes:
      - ${FOLDER_FOR_CONFIGS:?err}/radarr:/config
      - ${FOLDER_FOR_MEDIA:?err}:/data
    ports:
      - ${WEBUI_PORT_READARR:?err}:8787
    restart: unless-stopped
    networks:
      - media-network
```

## sonarr
```
###########################################################################
##  Docker Compose File:  Sonarr (LinuxServer.io)
##  Function:             Series Library Manager (TV Shows)
##  Documentation:        https://docs.linuxserver.io/images/docker-sonarr
###########################################################################
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    restart: unless-stopped
    volumes:
      - ${FOLDER_FOR_CONFIGS:?err}/sonarr:/config
      - ${FOLDER_FOR_MEDIA:?err}:/data
    ports:
      - ${WEBUI_PORT_SONARR:?err}:8989
    environment:
      - PUID=${PUID:?err}
      - PGID=${PGID:?err}
      - TZ=${TIMEZONE:?err}
    networks:
      - media-network
```

## unpackerr
```
###########################################################################
##  Docker Compose File:  Unpackerr (Hotio.Dev)
##  Function:             Archive Media Extraction
##  Documentation:        https://github.com/davidnewhall/unpackerr
##                        https://github.com/davidnewhall/unpackerr/blob/master/examples/docker-compose.yml
###########################################################################
  unpackerr:
    image: golift/unpackerr:latest
    container_name: unpackerr
    restart: unless-stopped
    volumes:
      - ${FOLDER_FOR_CONFIGS:?err}/unpackerr:/config
      - ${FOLDER_FOR_MEDIA:?err}/:/data
    environment:
      - PUID=${PUID:?err}
      - PGID=${PGID:?err}
      - UMASK=${UMASK:?err}
      - TZ=${TIMEZONE:?err}
# Documentation on all Environment Variables can be found at:
# https://github.com/davidnewhall/unpackerr#docker-env-variables
      - UN_DEBUG=false
      - UN_LOG_FILE=
      - UN_LOG_FILES=10
      - UN_LOG_FILE_MB=10
      - UN_INTERVAL=2m
      - UN_START_DELAY=1m
      - UN_RETRY_DELAY=5m
      - UN_MAX_RETRIES=3
      - UN_PARALLEL=1
      - UN_FILE_MODE=0664
      - UN_DIR_MODE=0775
      # Sonarr Config - Copy API Key from: http://sonarr:8989/general/settings
      - UN_SONARR_0_URL=http://sonarr:8989
      - UN_SONARR_0_API_KEY=${SONARR_API_KEY}
      - UN_SONARR_0_PATHS_0=/data/torrents/anime
      - UN_SONARR_0_PATHS_1=/data/torrents/series
      - UN_SONARR_0_PROTOCOLS=torrent
      - UN_SONARR_0_TIMEOUT=10s
      - UN_SONARR_0_DELETE_ORIG=false
      - UN_SONARR_0_DELETE_DELAY=5m
      # Radarr Config - Copy API Key from: http://radarr:7878/general/settings
      - UN_RADARR_0_URL=http://radarr:7878
      - UN_RADARR_0_API_KEY=${RADARR_API_KEY}
      - UN_RADARR_0_PATHS_0=/data/torrents/movies
      - UN_RADARR_0_PROTOCOLS=torrent
      - UN_RADARR_0_TIMEOUT=10s
      - UN_RADARR_0_DELETE_ORIG=false
      - UN_RADARR_0_DELETE_DELAY=5m
      # Lidarr Config - Copy API Key from: http://lidarr:8686/general/settings
      - UN_LIDARR_0_URL=http://lidarr:8686
      - UN_LIDARR_0_API_KEY=${LIDARR_API_KEY}
      - UN_LIDARR_0_PATHS_0=/data/torrents/music
      - UN_LIDARR_0_PROTOCOLS=torrent
      - UN_LIDARR_0_TIMEOUT=10s
      - UN_LIDARR_0_DELETE_ORIG=false
      - UN_LIDARR_0_DELETE_DELAY=5m
    security_opt:
      - no-new-privileges:true
    network_mode: none
```

## Environment variables
```
COMPOSE_PROJECT_NAME=media-stack
DOCKER_SUBNET=172.16.12.0/24
DOCKER_GATEWAY=172.16.12.1
LOCAL_SUBNET=192.168.0.0/24
LOCAL_DOCKER_IP=192.168.0.100
FOLDER_FOR_CONFIGS=/mnt/fast/servarr
FOLDER_FOR_MEDIA=/mnt/tank0/data
PUID=568
PGID=568
UMASK=0002
TIMEZONE=Europe/London
VPN_TYPE=wireguard
VPN_SERVICE_PROVIDER=nordvpn
SERVER_REGIONS=Europe
SERVER_COUNTRIES=Switzerland
SERVER_CITIES=Zurich
SERVER_HOSTNAMES=ch418.nordvpn.com
VPN_ENDPOINT_IP=185.7.34.112
WIREGUARD_PUBLIC_KEY=FT46M53w4dhBep/ZbpBgzvk71FlLZLDBM=
WIREGUARD_PRIVATE_KEY=GNHMMbKoNWQWwfhuuyzwGJstPyiV2g=
QBIT_PORT_TCP=6881
QBIT_PORT_UDP=6881
FLARESOLVERR_PORT=8191
TDARR_SERVER_PORT=8266
WEBUI_PORT_TDARR=8265
WEBUI_PORT_BAZARR=6767
WEBUI_PORT_JELLYFIN=8095
WEBUI_PORT_JELLYSEERR=5055
WEBUI_PORT_LIDARR=8686
WEBUI_PORT_MYLAR3=8090
WEBUI_PORT_PORTAINER=9443
WEBUI_PORT_PROWLARR=9696
WEBUI_PORT_QBITTORRENT=8200
WEBUI_PORT_RADARR=7878
WEBUI_PORT_READARR=8787
WEBUI_PORT_SONARR=8989
WEBUI_PORT_WHISPARR=6969
WEBUI_PORT_SABNZBD=6789
RADARR_API_KEY=14f60f87100240c58dcd09692a
SONARR_API_KEY=3ab52198f5e4427a82ab1670b3
LIDARR_API_KEY=60dedcc1848a4ea5a1129431f7
```
