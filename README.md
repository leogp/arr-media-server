## Project overview


This project provides a complete self-hosted media automation and streaming server using the Arr Suite, a collection of tools designed to automate media discovery, downloading, organization, and library management. The entire stack is containerized and designed for reliability, maintainability, and zero-touch automation once configured.

The stack includes:

* qBittorrent
* Prowlarr
* Radarr
* Sonarr
* Bazarr
* Jellyfin
* Seerr (successor of Jellyseerr)
* FlareSolverr (optional)
* Caddy reverse proxy (optional)

### High-Level Workflow

1. A user makes a media request in Seerr.

2. The request is routed to Radarr (movies) or Sonarr (TV shows).

3. Prowlarr handles indexer queries and returns matching results.

4. qBittorrent downloads the selected release.

5. Radarr/Sonarr process, rename, and hardlink the completed media into the library.

6. Jellyfin updates its library and makes the content available for streaming.

7. Bazarr downloads subtitles for the imported media.

## Legal Disclaimer
For legal reasons, I must clarify that I do not support or encourage the unauthorized downloading of copyrighted content. This material is provided strictly for educational purposes. Please do not use it to download copyrighted works.

## Choose your compose file

There is a single `compose.yaml` for every OS. The only thing that changes between Fedora, Ubuntu and Windows is the `DATA_ROOT` value in `.env`, so `docker compose up -d` works everywhere without `-f`.

| OS | Compose file | `DATA_ROOT` example | Guide |
|---|---|---|---|
| Fedora | `compose.yaml` | `/srv/media` | [Fedora guide](#fedora-guide) |
| Ubuntu | `compose.yaml` | `/srv/media` | [Ubuntu guide](#ubuntu-guide) |
| Windows (Docker Desktop + WSL2) | `compose.yaml` | `/home/<user>/media` (inside WSL2) | [Windows guide](#windows-guide) |

Optional pieces are added from `.env`, never by editing `compose.yaml`:

* **GPU overlays** (`docker-compose.gpu-vaapi.yml`, `docker-compose.gpu-nvidia.yml`) are added with `COMPOSE_FILE`. The separator is `:` on Linux and `;` on Windows:

  ```env
  # Linux
  COMPOSE_FILE=compose.yaml:docker-compose.gpu-vaapi.yml
  # Windows
  COMPOSE_FILE=compose.yaml;docker-compose.gpu-nvidia.yml
  ```

* **FlareSolverr** is enabled with `COMPOSE_PROFILES=flaresolverr`.

* **Caddy reverse proxy** is enabled with `COMPOSE_PROFILES=proxy`. Profiles are combined with commas, for example `COMPOSE_PROFILES=flaresolverr,proxy`. See [Reverse proxy](#reverse-proxy-optional).

When `COMPOSE_FILE` is unset, Compose loads `compose.yaml` only.

## Folder layout

All media lives under one host folder, `DATA_ROOT`, which keeps the usual structure:

```
${DATA_ROOT}            (e.g. /srv/media)
├── downloads           qBittorrent saves here
├── movies              Radarr library
└── tvshows             Sonarr library
```

qBittorrent, Radarr, Sonarr, Bazarr and Jellyfin mount the whole folder as a **single volume**, `${DATA_ROOT}:/data`, so inside every container the paths are the same:

```
/data/downloads
/data/movies
/data/tvshows
```

Why a single mount matters: a hardlink is a second name for the same file, and it can only be created inside one filesystem. When `downloads` and `movies` are mounted as two separate volumes, the container sees two different filesystems even if they live on the same host disk, so Radarr/Sonarr have to **copy** every import. That doubles the disk usage, and moving instead of copying would break seeding.

With one mount, Radarr/Sonarr create a hardlink from `/data/downloads/...` to `/data/movies/...` (or `/data/tvshows/...`) instantly:

* the content is stored only once on disk,
* qBittorrent keeps seeding the original file,
* the library gets a properly renamed file,
* deleting the torrent later does not remove the library file.

Prowlarr does not need access to media, so it only mounts its config folder. Backups of each Arr service stay in their default location, `/config/Backups`, which is already persisted in `./<service>/config`.

## Create and run containers
Copy the example environment file and set `DATA_ROOT` to your host media folder:

```sh
cp .env.example .env
```

For the purposes of this guide, I will use `/srv/media` on Linux. Avoid `/media`: it is reserved for removable drives, and on Fedora SELinux never labels its content (see the [Fedora guide](#fedora-guide)).

`.env` contains:

| Variable | Purpose |
|---|---|
| `PUID` / `PGID` | User and group the containers run as (`id -u` / `id -g`) |
| `TZ` | Time zone |
| `UMASK` | `002`, so files are group-writable and shared between containers |
| `DATA_ROOT` | Host folder mounted as `/data` |
| `COMPOSE_FILE` | Optional, adds GPU overlays |
| `COMPOSE_PROFILES` | Optional, enables FlareSolverr (`flaresolverr`) and/or the Caddy reverse proxy (`proxy`) |
| `VAAPI_DEVICE` | Optional, render node for the VAAPI overlay |

Prepare the host folders following your OS guide ([Fedora](#fedora-guide), [Ubuntu](#ubuntu-guide) or [Windows](#windows-guide)). On Linux, Seerr runs as UID 1000, so create its config folder with that owner before the first start:

```sh
mkdir -p jellyseerr/config
sudo chown -R 1000:1000 jellyseerr/config
```

Then do

```sh
docker compose up -d
```

## Configure services
### qBittorrent
First of all, you have to change the credentials. To find them see qBittorrent logs

```sh
docker logs qbittorrent
```
Access to http://localhost:8080 (or http://qbit.localhost with the [reverse proxy](#reverse-proxy-optional)), change it in __Tools > Options > WebUI__

Then go to __Tools > Options > Downloads__ and set:

```
Default Torrent Management Mode: Automatic
Default Save Path: /data/downloads
```

With the automatic mode, torrents sent by Radarr/Sonarr are saved in a subfolder named after their category (for example `/data/downloads/radarr`), always inside `/data`.

### Sonarr/Radarr
Enter to http://localhost:7878 (or http://radarr.localhost with the [reverse proxy](#reverse-proxy-optional)) for Radarr and create a user.

Go to __Settings > Media Management__

in Root folder add  _/data/movies_

Then click __Show Advanced__ and, under __Importing__, enable __Use Hardlinks instead of Copy__.

If you get an error, you must adjust your media folder permissions (see your OS guide).

Then go to __Settings > Download Clients__

Choose qBittorrent, and set:

```
host: qbittorrent
port: 8080
username: yourQbittorrentUser
password: yourQbittorrentPassword
```

Finally, copy the API key located in __General__

Repeat these steps for Sonarr (http://localhost:8989 or http://sonarr.localhost), but in Media Management add _/data/tvshows_

### Prowlarr
First access to the service in http://localhost:9696 (or http://prowlarr.localhost with the [reverse proxy](#reverse-proxy-optional)) and create a user.

Go to __Download Clients__ and connect qBittorrent in the same way you did it before.

Afterwards, go to __Settings > Apps__ and connect it with Radarr and Sonarr using their URLs and API keys.

```
Prowlarr Server: http://prowlarr:9696
Sonarr Server: http://sonarr:8989
API Key: sonarrApiKey
```

Lastly, add content indexers in __Indexers > Add Indexer__

#### FlareSolverr (optional)
Some indexers sit behind Cloudflare's anti-bot protection. You need FlareSolverr only if Prowlarr fails to add or test an indexer with a Cloudflare error (for example "blocked by CloudFlare Protection" or a challenge/captcha message).

Enable it in `.env` and start it:

```env
COMPOSE_PROFILES=flaresolverr
```

```sh
docker compose up -d
```

FlareSolverr does not publish any port on the host; only the other containers can reach it. In Prowlarr go to __Settings > Indexers__, add a __FlareSolverr__ proxy and set:

```
Name: FlareSolverr
Tags: flaresolverr
Host: http://flaresolverr:8191/
```

Then edit only the indexers that need it and add the `flaresolverr` tag to them. Indexers without the tag keep connecting directly.

### Bazarr
To configure Bazarr access to http://localhost:6767 (or http://bazarr.localhost with the [reverse proxy](#reverse-proxy-optional))

Then connect Bazarr to Sonarr and Radarr using their respective hostnames, ports and API Keys. For instance, to connect to sonarr you must do:

```
Address: sonarr
port: 8989
API Key: sonarrApiKey
```

You should see the Sonarr/Radarr version if you press the Test button

### Jellyfin
Access to http://localhost:8096 (or http://jellyfin.localhost with the [reverse proxy](#reverse-proxy-optional)), select your language and configure your credentials

Next, configure the media libraries by importing TV shows (`/data/tvshows`) and movies (`/data/movies`) into Jellyfin, matching the folders you previously set up

Finally, select your preferred metadata language and login to the application.

### Seerr
Seerr is the successor of Jellyseerr. The service keeps the `jellyseerr` name and its `./jellyseerr/config` folder, so existing installs are migrated automatically on first start.

Go to http://localhost:5055 (or http://seerr.localhost with the [reverse proxy](#reverse-proxy-optional)), choose Jellyfin as your media server and login to it.

```
host: http://jellyfin:8096
email address: yourEmail
username: yourJellyfinUser
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
Minimum Availability: Released
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

## Reverse proxy (optional)

Every service publishes its own port, so you have to remember `8080`, `7878`, `8989`, ... The optional Caddy reverse proxy gives each web UI a name instead: open `http://radarr.localhost` rather than `http://localhost:7878`. Caddy publishes only port 80 and forwards each name to the container hostname and internal port defined in `caddy/Caddyfile`. The `localhost:<port>` URLs keep working.

Enable it in `.env` and start it. Combine it with FlareSolverr using commas:

```env
COMPOSE_PROFILES=proxy
# or
COMPOSE_PROFILES=flaresolverr,proxy
```

```sh
docker compose up -d
```

| Service | URL | Upstream |
|---|---|---|
| Jellyfin | http://jellyfin.localhost | `jellyfin:8096` |
| Seerr | http://seerr.localhost | `jellyseerr:5055` |
| Radarr | http://radarr.localhost | `radarr:7878` |
| Sonarr | http://sonarr.localhost | `sonarr:8989` |
| Prowlarr | http://prowlarr.localhost | `prowlarr:9696` |
| Bazarr | http://bazarr.localhost | `bazarr:6767` |
| qBittorrent | http://qbit.localhost | `qbittorrent:8080` |

FlareSolverr has no web UI, so it has no entry.

Every site in the Caddyfile uses an explicit `http://` address, so Caddy serves plain HTTP and does not try to obtain HTTPS certificates. The warnings `no automatic HTTPS will be applied` and `HTTP/2 skipped because it requires TLS` in `docker logs caddy` are expected.

__Why `*.localhost` works__

`localhost` is a reserved name that always points to the machine itself ([RFC 6761](https://www.rfc-editor.org/rfc/rfc6761#section-6.3)), and so does any name under it. Nothing has to be added to `/etc/hosts` or to a DNS server:

* Chrome and Firefox resolve every `*.localhost` name to the loopback address on their own, on every OS.
* On Linux, the rest of the system gets the same answer from `nss-myhostname`, the `myhostname` module in the `hosts:` line of `/etc/nsswitch.conf` (enabled by default on Fedora). Check it with:

  ```sh
  getent hosts radarr.localhost
  ```

__qBittorrent answers "Unauthorized"__

qBittorrent validates the `Host` header of each request. If http://qbit.localhost answers `Unauthorized`, go to __Tools > Options > WebUI__ and either disable __Enable Host header validation__ or add `qbit.localhost` to __Server domains__ (entries are separated by `;`).

__Proxy a development app running on the host__

Caddy can also give a name to an app that runs directly on the host, outside Docker. Uncomment the example at the end of `caddy/Caddyfile` and adjust the name and port:

```
http://myapp.localhost {
	reverse_proxy host.docker.internal:3000
}
```

`host.docker.internal` points to the host thanks to `extra_hosts: host-gateway` in `compose.yaml`. The app must listen on `0.0.0.0`, not only on `127.0.0.1`, because the requests come from the Docker network instead of the loopback interface. Then restart Caddy:

```sh
docker compose restart caddy
```

A restart also picks up the file when your editor saves it as a new file, which a single-file bind mount does not follow until the container starts again.

__Port 80 conflicts__

Caddy needs port 80 on the host. If `docker compose up` reports that the port is already allocated, or another program (a local web server, another proxy) needs it, stop Caddy:

```sh
docker compose stop caddy
```

The rest of the stack keeps running and the `localhost:<port>` URLs keep working. `docker compose start caddy` brings it back. To see what is using the port, run `sudo ss -ltnp | grep ':80 '`.

__Other devices on your LAN__

`*.localhost` always means "this device", so on a phone, a TV or another computer `http://jellyfin.localhost` points to that device, not to the server. From the LAN, use the server IP and port, for example `http://192.168.1.50:8096`, and give the laptop or computer that runs the stack a DHCP reservation in your router so that IP does not change. If you want names on the LAN, run a local DNS server (your router, AdGuard Home or Pi-hole) with names under `home.arpa`, the domain reserved for home networks ([RFC 8375](https://www.rfc-editor.org/rfc/rfc8375)), for example `jellyfin.home.arpa`.

## Hardware transcoding

Hardware transcoding can significantly improve streaming performance by offloading video encoding to your GPU. It is configured with optional overlay files, so `compose.yaml` stays the same on every machine. Add the overlay to `COMPOSE_FILE` in `.env` (separator `:` on Linux, `;` on Windows) and recreate Jellyfin:

```sh
docker compose up -d jellyfin
```

| Overlay | GPU | Linux | Windows (Docker Desktop) |
|---|---|---|---|
| `docker-compose.gpu-vaapi.yml` | Intel Quick Sync / AMD (VAAPI) | Yes | No |
| `docker-compose.gpu-nvidia.yml` | NVIDIA (NVENC/NVDEC) | Yes | Yes |

### Intel and AMD GPUs on Linux (VAAPI)

1. Find the render node of your GPU. With a single GPU it is usually `/dev/dri/renderD128`. With more than one, match the PCI address of each node with the GPU model:

   ```sh
   ls -l /dev/dri/by-path
   lspci | grep -E 'VGA|3D'
   ```

   For example `pci-0000:00:02.0-render -> ../renderD128` belongs to the GPU listed as `00:02.0` by `lspci`.

2. Set the node and add the overlay in `.env`:

   ```env
   VAAPI_DEVICE=/dev/dri/renderD128
   COMPOSE_FILE=compose.yaml:docker-compose.gpu-vaapi.yml
   ```

   The LinuxServer image adds its user to the group that owns the device automatically.

3. Recreate Jellyfin and check that VAAPI works inside the container:

   ```sh
   docker compose up -d jellyfin
   docker exec jellyfin /usr/lib/jellyfin-ffmpeg/vainfo --display drm --device /dev/dri/renderD128
   ```

   The output lists the supported profiles (for example `VAProfileH264Main`, `VAProfileHEVCMain`, ...). An error here means Jellyfin will not be able to use the GPU either.

4. In Jellyfin go to __Dashboard > Playback > Transcoding__ and set:

   ```
   Hardware acceleration: Video Acceleration API (VAAPI)
   VA-API Device: /dev/dri/renderD128
   Enable hardware decoding for: only the codecs listed by vainfo
   ```

   Save, play a video that needs transcoding and check __Dashboard > Activity__ or the playback info to confirm the transcode is hardware accelerated.

__SELinux on Fedora__: the `container-selinux` policy defines the boolean `container_use_dri_devices`, which allows containers to use DRI devices mounted into them, and it is enabled by default ([container.te](https://github.com/containers/container-selinux/blob/main/container.te)). No extra labeling is needed for `/dev/dri`. Check it with:

```sh
getsebool container_use_dri_devices
```

If it reports `off`, enable it persistently:

```sh
sudo setsebool -P container_use_dri_devices on
```

### NVIDIA GPUs (Linux and Windows)

Prerequisites:

* **Linux**: the proprietary NVIDIA driver and the **NVIDIA Container Toolkit**, following the official guide:
  https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
* **Windows**: Docker Desktop with the WSL 2 backend and an up-to-date NVIDIA driver with WSL 2 support (`wsl --update` keeps the WSL kernel current):
  https://docs.docker.com/desktop/features/gpu/

Then add the overlay in `.env`:

```env
# Linux
COMPOSE_FILE=compose.yaml:docker-compose.gpu-nvidia.yml
# Windows
COMPOSE_FILE=compose.yaml;docker-compose.gpu-nvidia.yml
```

The overlay reserves the GPU for Jellyfin:

```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: all
          capabilities: [gpu]
```

Recreate Jellyfin, check the GPU with `docker exec jellyfin nvidia-smi` and select __NVIDIA NVENC__ in __Dashboard > Playback > Transcoding__.

### Intel Quick Sync and AMD GPUs on Windows

Docker Desktop only supports GPU passthrough to Linux containers for NVIDIA GPUs (GPU paravirtualization on the WSL 2 backend), see https://docs.docker.com/desktop/features/gpu/. Intel and AMD GPUs cannot be used for transcoding from the container, so Jellyfin falls back to software transcoding.

## Fedora guide

On Fedora, SELinux runs in enforcing mode by default. Containers are confined by SELinux and are only allowed to read from or write to host directories that have the correct SELinux security context.

With the proposed configuration, ARR services (qBittorrent, Sonarr, Radarr, Bazarr, Jellyfin, etc.) mount `${DATA_ROOT}` (`/srv/media`) to store and manage downloads and media files. In addition, ARR containers also mount configuration directories from the project root folder (for example, `./qbittorrent/config`, `./sonarr/config`, etc.).

By default, directories on the host filesystem are labeled with contexts that are **not permitted for container access** (such as `mnt_t` or standard user contexts) when SELinux is enforcing.

To fix this, you must **permanently label** `DATA_ROOT` and your project root directory **with the** `container_file_t` **SELinux type**. This explicitly tells SELinux that these directories are intended to be accessed by containers.

Because these context changes are persistent and independent of container recreation, all ARR containers can read from and write to both `DATA_ROOT` and their configuration directories normally after a reboot, without disabling SELinux or relying on fragile volume flags.

__Why `/srv/media` and not `/media`__

On Fedora, `/media` is reserved for removable devices that are mounted automatically. The base SELinux policy labels `/media` and its direct subfolders as `mnt_t`, and defines everything deeper as `<<None>>` (from `/etc/selinux/targeted/contexts/files/file_contexts`; `semanage fcontext -l` shows the same rules):

```
/media(/[^/]*)?	-d	system_u:object_r:mnt_t:s0
/media/[^/]*/.*	<<none>>
```

`<<None>>` means "never relabel": `restorecon` skips the content of `/media/movies`, `/media/tvshows`, etc., so a `semanage fcontext` rule for `/media(/.*)?` has no effect on your media files. `/srv` is meant for data served by the system and has no such exception, so a rule for `/srv/media` is applied to the whole tree. Use `/srv/media` (or any other path outside `/media`) as `DATA_ROOT`:

```env
DATA_ROOT=/srv/media
```

If your media is currently under `/media`, see [Migrating an existing install](#migrating-an-existing-install).

__Create the media folders__

If `/srv/media` is the mount point of a dedicated disk, mount it first (an `/etc/fstab` entry with `/srv/media` as mount point). Then create the folders:

```sh
sudo mkdir -p /srv/media/{downloads,movies,tvshows}
sudo chown -R 1000:1000 /srv/media/{downloads,movies,tvshows}
```

Use your own `PUID:PGID` if they are not `1000:1000`.

__Label `DATA_ROOT` with `container_file_t`__

```sh
sudo semanage fcontext -a -t container_file_t "/srv/media(/.*)?"
sudo restorecon -RFv /srv/media
```

__Label the project root directory__

Run this from the project root (the folder that contains `compose.yaml`). `pwd -P` resolves symlinks, because SELinux rules must match the real path:

```sh
sudo semanage fcontext -a -t container_file_t "$(pwd -P)(/.*)?"
sudo restorecon -RFv "$(pwd -P)"
```

If `semanage` answers that the rule is already defined, replace `-a` (add) with `-m` (modify).

__Why `restorecon -F`__

Without `-F`, `restorecon` only fixes the *type* part of the context. Files that were previously relabeled by a `:Z` volume keep their private MCS categories (for example `system_u:object_r:container_file_t:s0:c123,c456`), and those categories still block every other container. `-F` forces a full reset of the context, categories included, back to plain `container_file_t:s0`.

__Why the volumes have no `:z`/`:Z` flags__

* `:Z` relabels the volume with categories that belong to **one** container. On a folder shared by qBittorrent, Radarr, Sonarr, Bazarr and Jellyfin, the last container that starts takes it over and the others lose access every time a container is recreated.
* `:z` uses a shared label, but Docker relabels the **whole tree** on every container start. On a large media library that is slow and unnecessary when the label is already persistent.

Persistent labeling with `semanage fcontext` is done once and survives reboots, container recreation and `restorecon` runs.

__Verify__

First check what the policy *expects* for a file inside the library. `matchpathcon` reads the rules, so it works even for a path that does not exist yet:

```sh
matchpathcon /srv/media/movies/test.mkv
```
Expected output:

```arduino
/srv/media/movies/test.mkv	system_u:object_r:container_file_t:s0
```

If it prints `<<none>>` or `mnt_t`, `DATA_ROOT` is still under `/media` or the rule was not added.

Then check the actual labels on disk:

```sh
ls -Zd /srv/media /srv/media/downloads /srv/media/movies /srv/media/tvshows
```
Expected output:

```arduino
system_u:object_r:container_file_t:s0 /srv/media
...
```

Check that no file still carries private MCS categories. This command should print nothing:

```sh
sudo find /srv/media -context '*:s0:c*'
sudo find "$(pwd -P)" -context '*:s0:c*'
```

## Ubuntu guide

Ubuntu does not use SELinux. Its default security module, AppArmor, confines containers through the `docker-default` profile and does not require labeling bind-mounted volumes, so only ownership needs attention.

### Prerequisites

Install Docker Engine and the Compose plugin from Docker's official repository:
https://docs.docker.com/engine/install/ubuntu/

Optionally, run Docker without `sudo` by adding your user to the `docker` group (log out and back in afterwards):

```sh
sudo usermod -aG docker $USER
```

### Prepare media folders

Create the folders and give them to the user and group set in `PUID`/`PGID` (`id -u` / `id -g`, usually `1000`):

```sh
sudo mkdir -p /srv/media/{downloads,movies,tvshows}
sudo chown -R 1000:1000 /srv/media/{downloads,movies,tvshows}
sudo chmod -R u=rwX,g=rwX,o=rX /srv/media/{downloads,movies,tvshows}
```

`/srv/media` is used instead of `/media` because Ubuntu uses `/media/<user>` to automount removable drives. Only the subfolders change owner, so `/srv/media` itself can stay owned by root.

Then follow [Create and run containers](#create-and-run-containers).

## Windows guide

### Prerequisites

1. Install **Docker Desktop** from https://www.docker.com/products/docker-desktop/
2. During installation, enable the **WSL 2** backend (recommended) when prompted.
3. After install, open Docker Desktop → Settings → General → confirm **"Use the WSL 2 based engine"** is checked.

### Where to store the data

Hardlinks are what avoid duplicated content (see [Folder layout](#folder-layout)). Whether they work on a bind mount of a Windows drive (for example `D:\media`) under Docker Desktop is **not documented and not guaranteed**: Windows drives reach the Linux VM through a file-sharing layer, not as a native Linux filesystem. Docker also recommends bind-mounting files from the Linux filesystem rather than from Windows drives ([WSL 2 best practices](https://docs.docker.com/desktop/features/wsl/best-practices/)).

The recommended setup is to keep both the project and the media **inside the WSL 2 filesystem**:

1. Install a WSL distribution, for example `wsl --install -d Ubuntu`.
2. In Docker Desktop → Settings → Resources → WSL integration, enable it for that distribution.
3. Open the distribution's shell, clone this repository in your home folder and create the media folders:

   ```sh
   mkdir -p ~/media/{downloads,movies,tvshows}
   ```

4. Run every `docker compose` command from that shell.

You can browse the files from Windows Explorer at `\\wsl$\Ubuntu\home\<user>\media`.

If you still prefer a Windows drive, create `D:\media\downloads`, `D:\media\movies` and `D:\media\tvshows`. Radarr/Sonarr fall back to copying when a hardlink cannot be created, so the stack keeps working, but content may be duplicated. Check it after the first import with the `stat` command from [Migrating an existing install](#migrating-an-existing-install).

### Set up .env

```sh
cp .env.example .env
```

(`copy .env.example .env` in Command Prompt.) Open `.env` and set `DATA_ROOT` instead of editing paths in `compose.yaml`:

```env
PUID=1000
PGID=1000
TZ=America/Argentina/Buenos_Aires
UMASK=002
# Inside WSL 2 (recommended)
DATA_ROOT=/home/<user>/media
# Or a Windows drive, with forward slashes
# DATA_ROOT=D:/media
```

On a Windows drive, `PUID`/`PGID` are accepted by LinuxServer images but have no effect on file ownership — the containers still work correctly. Inside the WSL 2 filesystem the folders are regular Linux folders, so keep `PUID`/`PGID` equal to the owner of `~/media` (`id -u` / `id -g` in the WSL shell, usually `1000`).

To use the NVIDIA overlay on Windows, remember that `COMPOSE_FILE` uses `;` as separator when Compose runs from PowerShell or Command Prompt, and `:` when it runs from the WSL shell.

### Start the stack

Open a terminal in the project folder (the WSL shell, or PowerShell/Command Prompt for a Windows drive):

```sh
docker compose up -d
```

All service web UIs are then reachable at `http://localhost:<port>` as listed in the README, or at `http://<service>.localhost` with the [reverse proxy](#reverse-proxy-optional).

> **Note:** Jellyfin publishes `7359/udp` (client auto-discovery) and `1900/udp` (DLNA). DLNA has been a plugin since Jellyfin 10.9, and discovery broadcasts may not reach the Docker Desktop VM. If Docker reports that one of these ports is not available on Windows, comment out that line in `compose.yaml`; streaming on port 8096 is not affected.

## Migrating an existing install

These steps move an install that used separate `/downloads`, `/data/movies` and `/data/tvshows` mounts to the single `/data` mount. The folder structure (`downloads/`, `movies/`, `tvshows/`) does not change. On Fedora the data root also moves from `/media` to `/srv/media` (see [Why `/srv/media` and not `/media`](#fedora-guide)); paths inside the containers stay `/data/...`, so no service setting depends on the host path.

1. **Stop the stack and update `.env`.** Pull the new version of the repository and add the new variables from `.env.example` (`UMASK=002` and `DATA_ROOT`):

   ```sh
   docker compose down
   git pull
   ```

   ```env
   UMASK=002
   DATA_ROOT=/srv/media
   ```

   Seerr replaces Jellyseerr and runs as UID 1000. Back up its config and fix the owner before starting:

   ```sh
   sudo cp -a jellyseerr/config jellyseerr/config.bak
   sudo chown -R 1000:1000 jellyseerr/config
   ```

2. **Move the data root from `/media` to `/srv/media`.** Required on Fedora, optional on Ubuntu (where `/media` works but is also used for automounted drives). The stack must be stopped (step 1), otherwise the mount point is busy.

   If the media is on a **dedicated disk mounted at `/media`**, change its mount point in `/etc/fstab`:

   ```sh
   sudo mkdir -p /srv/media
   sudoedit /etc/fstab
   ```

   Edit only the mount point (second field) of the media disk line, for example:

   ```
   # before
   UUID=1234-abcd  /media      ext4  defaults  0 2
   # after
   UUID=1234-abcd  /srv/media  ext4  defaults  0 2
   ```

   Then unmount the old path, let systemd regenerate its mount units from the new `fstab`, and mount the new path:

   ```sh
   sudo umount /media
   sudo systemctl daemon-reload
   sudo mount /srv/media
   findmnt /srv/media
   ```

   If the folders are **plain directories on the root filesystem** instead, move them. `mv` within one filesystem is an instant rename that keeps existing hardlinks; check first that `df /media /srv` shows the same device:

   ```sh
   sudo mkdir -p /srv/media
   sudo mv /media/downloads /media/movies /media/tvshows /srv/media/
   ```

   On Fedora, replace the old SELinux rule with one for the new path. List your local rules first and delete the `/media` one only if it is there:

   ```sh
   sudo semanage fcontext -l -C | grep media
   sudo semanage fcontext -d "/media(/.*)?"
   sudo semanage fcontext -a -t container_file_t "/srv/media(/.*)?"
   ```

3. **Fedora only: relabel.** The new `compose.yaml` no longer uses `:Z`. Make sure the project rule from the [Fedora guide](#fedora-guide) exists, then apply the labels and reset the private categories that `:Z` left behind:

   ```sh
   sudo restorecon -RFv /srv/media
   sudo restorecon -RFv "$(pwd -P)"
   matchpathcon /srv/media/movies/test.mkv               # should print container_file_t
   sudo find /srv/media "$(pwd -P)" -context '*:s0:c*'   # should print nothing
   ```

   Start the stack again with `docker compose up -d` and check the Seerr migration with `docker logs -f jellyseerr`.

4. **qBittorrent: update the save path.** In __Tools > Options > Downloads__ set __Default Save Path__ to `/data/downloads` and __Default Torrent Management Mode__ to `Automatic`. Existing torrents still point to `/downloads/...`, which no longer exists inside the container. Select them, right click → __Set location__ and replace the prefix `/downloads/...` with `/data/downloads/...` (same folder on the host). If a torrent shows missing files afterwards, right click → __Force recheck__.

5. **Radarr/Sonarr: enable hardlinks.** In __Settings > Media Management__ click __Show Advanced__ and enable __Use Hardlinks instead of Copy__. Root folders (`/data/movies`, `/data/tvshows`) do not change. If you had created a Remote Path Mapping for `/downloads`, delete it.

6. **Deduplicate the existing copies.** Files imported before this change are full copies. `hardlink` (from util-linux) replaces identical files with hardlinks; `-c` compares content only and `-n` makes it a dry run. All three folders must be on the same host filesystem (`df /srv/media/downloads /srv/media/movies /srv/media/tvshows` shows the same device):

   ```sh
   sudo hardlink -c -n -v /srv/media/downloads /srv/media/movies /srv/media/tvshows
   ```

   Review the output, then run it for real:

   ```sh
   sudo hardlink -c -v /srv/media/downloads /srv/media/movies /srv/media/tvshows
   ```

7. **Verify.** The first column of `stat` is the link count; `2` means the file is shared between `downloads` and the library:

   ```sh
   stat -c '%h %n' /srv/media/movies/*/*.mkv
   ```

   New imports should show `2` as well. A `1` on a file that is still seeding means it was copied.

## Official documentation

This repository provides a basic setup for each service. For advanced configuration, troubleshooting, and full feature support, refer to the official documentation:

- **qBittorrent**: https://github.com/qbittorrent/qBittorrent/wiki
- **Prowlarr**: https://wiki.servarr.com/prowlarr
- **Sonarr**: https://wiki.servarr.com/sonarr
- **Radarr**: https://wiki.servarr.com/radarr
- **Bazarr**: https://wiki.bazarr.media
- **Seerr**: https://docs.seerr.dev
- **FlareSolverr**: https://github.com/FlareSolverr/FlareSolverr
- **Jellyfin**: https://jellyfin.org/docs
- **Caddy**: https://caddyserver.com/docs/
