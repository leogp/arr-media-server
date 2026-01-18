## Project overwiev


This project provides a complete self-hosted media automation and streaming server using the Arr Suite, a collection of tools designed to automate media discovery, downloading, organization, and library management. The entire stack is containerized and designed for reliability, maintainability, and zero-touch automation once configured.

The stack includes:

* qBittorrent
* Prowlarr
* Radarr
* Sonarr
* Jellyfin
* Jellyseerr

### High-Level Workflow

1. A user makes a media request in Jellyseerr.

2. The request is routed to Radarr (movies) or Sonarr (TV shows).

3. Prowlarr handles indexer queries and returns matching results.

4. qBittorrent downloads the selected release.

5. Radarr/Sonarr process, rename, and move the completed media.

6. Jellyfin updates its library and makes the content available for streaming.

## Legal Disclaimer
For legal reasons, I must clarify that I do not support or encourage the unauthorized downloading of copyrighted content. This material is provided strictly for educational purposes. Please do not use it to download copyrighted works.

## Create and run containers
Define your host media path and change it in docker-compose.yml

For the purposes of this guide, I will use `/media` 

Then do

```sh
mv .env.example .env
docker compose up -d
```

## Configure services
### qBittorrent
First of all, you have to change the credentials. To find them see qBittorrent logs

```sh
docker logs qbittorrent
```
Access to http://localhost:8080, change it in __Tools > Options > WebUI__

### Sonarr/Radarr
Enter to http://localhost:7878 for Radar and create a user.

Go to __Settings > Media Management__ 

in Root folder add  _/data/movies_

If you get an error, you must adjust your media folder permissions

```sh
sudo chown -R $USER /media
sudo chgrp -R $USER /media
```

After that, go to __General > Backups__ and set _/data/backup_

Then go to __Settings > Download Clients__

Choose qBittorrent, and set:

```
host: qbittorrent
port: 8080
username: yourQbittorrentUser
password: yourQbittorrentPassword
```

Finally, copy the API key located in __General__

Repeat these steps for Sonarr (http://localhost:8989), but in Media Management add _/data/tvshows_

### Prowlarr
First access to the service in http://localhost:9696 and create a user.

Go to __Downloads clients__ and connect qBittorrent in the same way you did it before. 

Afterwards, go to __Settings > Apps__ and connect it with Radarr and Sonnarr using their URLs and API keys.

```
Prowlarr Server: http://prowlarr:9696
Sonarr Server: http://sonarr:8989
API Key: sonarrApiKey
```

Lastly, add content indexers in __Indexers > Add Indexer__

### Bazarr
To configure Bazarr access to http://localhost:6767

Then connec Bazarr to Sonarr and Radarr using the their respective hostnames, ports and API Keys. For instance, to connect to sonarr you must do:

```
Addres: sonarr
port: 8989
API Key: sonarrApiKey
```

You should see the Sonarr/Radarr version if you press the Test button

### jellyfin
Access to http://localhost:8096, select your language and configure your credentials

Next, configure the media libraries by importing TV shows and movies into Jellyfin, matching the folders you previously set up

Finally, select your preferred metadata language and login to the application.

### jellyseerr
Go to http://localhost:5055, choose Jellyfin as your media server and login to it.

```
host: http://jellyfin:8096
email address: yourEmail
username: yourJellyfinjellyfinUser
password: yourJellyfinPassword
```

Next, Sync TV shows and movies libraries

Add Radarr

```
Default Server: check
Server Name: Radarr
Hostname or IP Address: radarr
Port: 7878
API Key: radarrApiKey
Quality Profile: Any
Root Folder: /data/movies
Minimum Availability: Realeased
```

Add Sonarr

```
Default Server: check
Server Name: Sonarr
Hostname or IP Address: sonarr
Port: 8989
API Key: sonarrApiKey
Quality Profile: Any
Root Folder: /data/tvshows
```

## Fedora guide

On Fedora, SELinux runs in enforcing mode by default. Containers are confined by SELinux and are only allowed to read or write host directories that have the correct SELinux security context.

ARR services (qBittorrent, Sonarr, Radarr, Bazarr, Jellyfin, etc.) mount host paths under `/media` to store and manage downloads and media files. By default, directories under `/media` are labeled with a generic mount context `(mnt_t)`, which containers are **not permitted to access** when SELinux is enforcing.

To fix this, you must **permanently label** `/media` **with the** `container_file_t` **SELinux type**. This explicitly tells SELinux that the directory is intended for use by containers.

Because this context change is persistent and independent of container recreation, all ARR containers can read and write to `/media` normally after reboot, without disabling SELinux or relying on fragile volume flags.

__Label `/media` with `container_file_t`__

```sh
sudo semanage fcontext -a -t container_file_t "/media(/.*)?"
sudo restorecon -Rv /media
```

Verify the label with:
```sh
ls -Zd /media
```
Expected output:

```arduino
system_u:object_r:container_file_t:s0 /media
```

__If you have NVIDIA and want to caste, you have to install NVIDIA-container-toolkit to enable jellyfin to use NVIDIA drivers__
https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
