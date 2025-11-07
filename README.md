# Setting up an automated Jellyfin media server

This setup uses Arch Linux (but should work with other distros) and docker container gpu access with the nvidia-container-toolkit for transcoding.

This docker container stack uses:
- **Tailscale** for remote access
- **Gluetun** for vpn with port forwarding
- **Qbittorrent** as download client
- **Sonarr/Radarr** for TV/movie monitoring
- **Prowlarr** for indexers
- **Byparr** for bypassing cloudflare protection

Using this configuration, after adding shows and movies to be monitored with Sonarr/Radarr, the latter handle interaction with Prowlarr for searching (and cloudflare bypass, if necessary), qbittorrent for downloading, and copying those downloads to the appropriate Jellyfin media directories. Qbittorrent or Sonarr/Radarr can handle the removal of download files after they've been relocated, depending on preference and other considerations (covered later).

# Docker Installation

Docker is used to run the services in isolated containers. The docker engine can be installed by running:

```bash
sudo pacman -S docker
```

The docker daemon can then be enabled and started:

```bash
systemctl enable --now docker.service
```

This will ensure that docker starts when the system boots up.

*Optionally*, the user can be added to the docker group for running docker commands without sudo:

```bash
sudo usermod -aG docker $USER
```

This will require the user to logout and log back in or possibly reboot to update.

For other distros and troubleshooting, see [here](https://docs.docker.com/engine/install/).

# Docker Configuration

Create a directory to store the configuration files for the services and a docker compose file for the container stack:

```bash
mkdir ~/docker
touch ~/docker/docker-compose.yml
```

Add the first line to the docker compose file:

```
services:
```

Under this each service will be added. The specific indentation in the following snippets are important.

## Tailscale

**Note:** Tailscale account required. Visit https://login.tailscale.com/.

```
  tailscale:
    image: tailscale/tailscale:latest
    container_name: tailscale
    hostname: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TS_AUTHKEY=your auth key goes here
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_SERVE_CONFIG=/config/jellyfin.json
    volumes:
      - ./tailscale/config:/config
      - /var/lib/tailscale:/var/lib/tailscale
    devices:
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - net_admin
    ports:
      - 8096:8096 # jellyfin
      - 7359:7359 # jellyfin
    dns:
      - 1.1.1.1
      - 8.8.8.8
    restart: unless-stopped
```

```image: tailscale/tailscale:latest``` specifies the online image to be pulled for the service.

```hostname: jellyfin``` is the machine name you will see/use on your tailnet. 

```PUID=1000``` and ```PGID=1000``` refer to your user. Confirm they are correct by running ```id``` in the terminal. 

```TS_AUTHKEY``` will need to be a generated auth key from the tailscale [admin site](https://login.tailscale.com/admin). Navigate to Settings->Personal Settings->Keys and generate auth key. It is displayed only once upon creation. Copy it and replace "your auth key goes here" above.

```TS_SERVE_CONFIG``` defines the tailscale server config, in this case for Jellyfin. This will enable the tailscale funnel for remote access to the Jellyfin server.

Create the tailscale server config json for Jellyfin:

```bash
mkdir ~/docker/tailscale
touch ~/docker/tailscale/jellyfin.json
```

Add the following to the config file:

```
{
  "TCP": {
    "443": {
      "HTTPS": true
    }
  },
  "Web": {
    "${TS_CERT_DOMAIN}:443": {
      "Handlers": {
        "/": {
          "Proxy": "http://127.0.0.1:8096"
        }
      }
    }
  },
  "AllowFunnel": {
    "${TS_CERT_DOMAIN}:443": true
  }
}
```

```volumes:``` are allocated for the service.

```devices:``` the container is given access to. In this case, the hosts tun device, which allows the container to create and manage its own virtual network tunnel.

```ports:``` for Jellyfin are mapped to the host for LAN access.

```dns``` defines DNS handlers for Tailscale so that Jellyfin can resolve its DNS queries for metadata retrieval.

```restart: unless-stopped``` tells docker to restart the service on boot unless it had been stopped before rebooting.

For more information on using tailscale with docker, see [here](https://github.com/tailscale-dev/docker-guide-code-examples).

## Gluetun

```
  gluetun:
    image: qmcgaw/gluetun:v3.40.0
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    ports:
      - 9696:9696 # prowlarr
      - 8989:8989 # sonarr
      - 7878:7878 # radarr
      - 8191:8191 # flaresolverr
      - 8080:8080 # qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
      - VPN_SERVICE_PROVIDER=protonvpn
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=[your private key goes here]
      - PORT_FORWARD_ONLY=on
      - VPN_PORT_FORWARDING=on
      - VPN_PORT_FORWARDING_UP_COMMAND=/bin/sh -c 'wget -O- --retry-connrefused --post-data "json={\"listen_port\":{{PORTS}}}" http://127.0.0.1:8080/api/v2/app/setPreferences 2>&1'
    restart: unless-stopped
```

```ports:``` shows the services being routed through the gluetun VPN service and mapped to the host for LAN access. 

```TZ=America/Toronto``` is a timezone which can be modified. 

```WIREGUARD_PRIVATE_KEY``` is the private key from a manually generated wireguard configuration file which can be made when signing into a proton account. NAT-PMP must also be enabled for port forwarding.

```VPN_PORT_FORWARDING_UP_COMMAND``` runs a command that automatically updates qbittorrent's listening port when a port has been successfully forwarded.

This configuration is set up for protonVPN which supports port forwarding over the wireguard protocol. Other providers/protocols will require modified variables. To that end, see gluetun documentation [here](https://github.com/qdm12/gluetun-wiki/blob/main/setup/readme.md#setup).

**Note:** Port forwarding has not worked when manually specifying countries and/or cities to connect to using the ```countries``` and ```cities``` environment variables. 

## Qbittorrent

```
  qbittorrent:
    image: linuxserver/qbittorrent
    container_name: qbittorrent
    network_mode: service:gluetun
    environment:
      - PUID=1000
      - PGID=1000
      - WEBUI_PORT=8080
    volumes:
      - ./qbittorrent:/config
      - /path/to/qbittorrent-downloads:/downloads
    restart: unless-stopped
```

```network_mode: service: gluetun``` instructs the container to use gluetun's network interface, routing its traffic through the VPN.

```WEBUI_PORT=8080``` specifies the port for accessing qbittorrent's web UI.

Replace ```/path/to/qbittorrent-downloads``` with actual qbittorrent downloads directory.

## Jellyfin

```
  jellyfin:
    image: ghcr.io/linuxserver/jellyfin
    container_name: jellyfin
    network_mode: service:tailscale
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - ./jellyfin:/config
      - /path/to/media:/media
    runtime: nvidia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: unless-stopped
```

```network-mode: service:tailscale``` instructs the container to use the tailscale network interface, which is used to remotely access the Jellyfin media server.

Replace ```/path/to/media``` with the actual media directory.

```runtime: nvidia``` and ```deploy:``` allow for use of the nvidia-container-runtime/GPU access.

## Prowlarr

```
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:develop
    container_name: prowlarr
    network_mode: service:gluetun
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - ./prowlarr:/config
    restart: unless-stopped
```

## Sonarr/Radarr

```
  sonarr:
    image: ghcr.io/linuxserver/sonarr
    container_name: sonarr
    network_mode: service:gluetun
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - ./sonarr:/config
      - /path/to/media/tv:/tv
      - /path/to/qbittorrent-downloads:/downloads
    restart: unless-stopped

  radarr:
    image: ghcr.io/linuxserver/radarr
    container_name: radarr
    network_mode: service:gluetun
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - ./radarr:/config
      - /path/to/media/movies:/movies
      - /path/to/qbittorrent-downloads:/downloads
    restart: unless-stopped
```

Replace ```/path/to/media/tv```, ```/path/to/media/movies```, and both instances of ```/path/to/qbittorrent-downloads``` with the actual respective directories.

## Byparr

```
  byparr:
    image: ghcr.io/thephaseless/byparr:latest
    container_name: byparr
    network_mode: service:gluetun
    environment:
      - PUID=1000
      - PGID=1000
    init: true
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
```

# NVIDIA-container-toolkit

**Aside:** Initially GPU passthrough was attempted with rootless docker. With rootless docker the docker daemon runs as a non-root user. However, GPU access requires the docker daemon to have root permissions. With rootful docker the containers also run as root, though the processes inside the container can be run as a user. For information on rootless docker, see [here](https://docs.docker.com/engine/security/rootless).

The proprietary NVIDIA drivers are required to use the nvidia-container-toolkit:

```bash
sudo pacman -S nvidia nvidia-utils
```

For troubleshooting, see [here](https://wiki.archlinux.org/title/NVIDIA#Installation).

Next, install the nvidia-container-toolkit:

```bash
sudo pacman -S nvidia-container-toolkit
```

To configure docker to use the nvidia-ctk runtime, the following command modifies the host /etc/docker/daemon.json file:

```bash
sudo nvidia-ctk runtime configure --runtime=docker
```

Now restart the docker daemon:

```bash
sudo systemctl restart docker
```

There is a current known issue when using systemd cgroups that causes containers to lose access to GPUs and requires modifying /etc/nvidia-container-runtime/config.toml and changing "no-cgroups = true" to "no-cgroups = false":

```bash
sudo nano /etc/nvidia-container-runtime/config.toml
```

Finally, add user to the video group:

```bash
sudo usermod -aG video $USER
```

For issues with nvidia-ctk, see [here](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#rootless-mode) and [here](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/nvidia/#configure-with-linux-virtualization).

## Verifying container GPU access

Start the containers:

```
cd ~/docker
docker compose up -d
```

Verify that Jellyfin can access the GPU:

```
docker exec -it jellyfin nvidia-smi
```

If successful, there should be a print out of GPU information/status and processes.

# Configuring the Services

## Jellyfin

1) Go to http://localhost:8096
2) Follow the setup wizard.
3) Add your tv and movies libraries, using the directories specified in the docker configuration.

## Qbittorrent

### Set password

1) Run ```docker logs qbittorrent``` and find the initial temporary session password in the log
2) Go to http://localhost:8080
3) Log in with username admin and temporary password.
4) Navigate to Tools->Options->WebUI and set the password.
5) Click save.

### Bind to VPN interface

**This is very important for ensuring torrent traffic is routed through the VPN**

1) Go to http://localhost:8080
2) Navigate to Tools->Options->Advanced
3) Under Network Interface, select the VPN interface from the dropdown, which should be labelled tun0.
4) Click save.

### Verify download location

1) Go to http://localhost:8080
2) Nagivate to Tools->Options->Downloads
3) Verify that 'Default Save Path' matches directory set in docker compose file.
4) Click save.

### (Optional) Automatic removal of finished torrents

If there is difficulty with seeding torrents, especially while behind VPN (which can occur without port forwarding), use qbittorrent's built-in torrent removal mechanism rather than Sonarr/Radarr's Seeding Ratio removal mechanism:

1) Go to http://localhost:8080
2) Navigate to Tools->Options->Bittorrent
3) Under Seeding Limits, check the box for When Seeding Time Reaches and set it to 10 minutes, this will give Sonarr/Radarr time to move the completed downloads to the appropriate directories.
4) Click save.

## Prowlarr

### Add Byparr for cloudflare protected indexers:

1) Go to http://localhost:9696
2) Navigate to Settings->Indexers and click the plus icon to add.
3) Fill in the following:\
   Name: Byparr\
   Tags: Byparr\
   Host: http://localhost:8191
5) Test and save.

### Add indexers

1) Go to http://localhost:9696
2) Navigate to Indexers and click Add Indexer.
3) Select indexer from the list, a dialogue box popup appears.
4) If intending to use the built-in torrent removal mechanism of Sonarr/Radarr, set a Seeding Ratio in that field after which a torrent will be removed from the download client.
5) If the dialog box popup indicates under Flaresolverr info that the indexer uses cloudflare protection, add Byparr to tags.
6) Test and save.

Some good, tested indexers include: 1337x, Bitsearch, Extratorrent.st, EZTV, kickasstorrents, LimeTorrents, The Pirate Bay, and TorrentGalaxyClone.

### Add Sonarr to Prowlarr for indexing:

1) Go to http://localhost:9696
2) Navigate to Settings->Apps and click the plus icon to add.
3) Fill in the following: \
   Name: Sonarr\
   Prowlarr Server: http://localhost:9696 \
   Sonarr Service: http://localhost:8989 \
   API Key:  
   1) Go to http://localhost:8989
   2) Navigate to Settings->General
   3) Copy the API Key
   4) Paste key into API Key field
4) Test and save.

### Add Radarr to Prowlarr for indexing:

1) Go to http://localhost:9696
2) Navigate to Settings->Apps and click the plus icon to add.
3) Fill in the following: \
   Name: Radarr\
   Prowlarr Server: http://localhost:9696 \
   Radarr Service: http://localhost:7878 \
   API Key:  
   1) Go to http://localhost:7878
   2) Navigate to Settings->General
   3) Copy the API Key
   4) Paste key into API Key field
4) Test and save.

## Sonarr/Radarr

### Set root directory

1) Go to http://localhost:8989 (Sonarr) or http://localhost:7878 (Radarr)
2) Navigate to Settings->Media Management
3) Click Add Root Folder
4) Navigate to the appropriate directory specified in the docker configurations (/path/to/media/tv for Sonarr, /path/to/media/movies for Radarr)

### Add qbittorrent to download clients:

1) Go to http://localhost:8989 (Sonarr) or http://localhost:7878 (Radarr) 
2) Navigate to Settings->Download Clients and click the plus icon to add.
3) Fill in the following: \
   Host: localhost \
   Port: 8080 \
   Username: the username (admin, if left as default) \
   Password: the password set when configuring qbittorrent earlier \
   Check 'Remove Completed' if you want to use Sonarr/Radarr's torrent removal mechanism. \
   **NOTE:** If qbittorrent's automatic torrent removal mechanism was set earlier, Sonarr/Radarr will complain. This will need to be disabled to add the client. Remember to reenable it afterwards.
5) Test and save.

## Tailscale

### Enable MagicDNS, HTTPS Certificates, and Funnel

1) Sign into your tailscale [admin panel](https://login.tailscale.com/admin).
2) Navigate to DNS and ensure MagicDNS and HTTPS Certificates are enabled.
3) Navigate to Access Controls->Node Attributes->Add Node Attribute.
4) In the targets dropdown select all users and devices. In the attributes dropdown select funnel. Click Save node attribute.

### To access the Jellyfin media server remotely

1) Run ```docker exec tailscale tailscale serve status```
2) Use the given URL to sign in to the server on the Jellyfin app. This URL can also be found and customized in the Tailscale [admin panel](https://login.tailscale.com/admin) under Machines and the addresses dropdown for your host machine
3) Use your Jellyfin credentials to sign in to your user on the server.
