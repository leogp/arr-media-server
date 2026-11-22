# Troubleshooting

This guide assumes the layout described in the [README](README.md): `DATA_ROOT` (for example `/srv/media`) mounted as `/data` in qBittorrent, Radarr, Sonarr, Bazarr and Jellyfin. Replace `/srv/media` with your own `DATA_ROOT` in the commands below.

Start by locating **where the pipeline stopped**. Every stage leaves a visible trace:

| Symptom | Stage that failed | Section |
|---|---|---|
| Request approved in Seerr, nothing in Radarr/Sonarr | Seerr → Radarr/Sonarr | [Seerr](#seerr) |
| Movie/series is *Missing* in Radarr/Sonarr, nothing in qBittorrent | Search / indexers | [Nothing reaches qBittorrent](#nothing-reaches-qbittorrent) |
| Radarr/Sonarr say *Grabbed*, nothing in qBittorrent | Download client | [Grabbed but not in qBittorrent](#grabbed-but-not-in-qbittorrent) |
| Download finished, never imported | Import (paths, permissions, SELinux) | [Downloaded but not imported](#downloaded-but-not-imported) |
| Disk filling up, content duplicated | Hardlinks | [Duplicated content](#duplicated-content-and-disk-filling-up) |
| Imported, not visible in Jellyfin | Library scan / permissions | [Jellyfin](#jellyfin) |
| Worked yesterday, "permission denied" today | SELinux or ownership | [SELinux](#selinux-fedora), [Permissions](#permissions) |
| `http://<service>.localhost` does not open | Caddy | [Reverse proxy](#reverse-proxy) |

---

## Nothing reaches qBittorrent

Radarr and Sonarr search **once**, when an item is added. After that they only read the indexers' RSS feeds, which carry *new* releases. If that first search failed, an older title is never grabbed on its own, even though it shows as *monitored*.

A common cause is that the indexers were failing at that moment. Radarr/Sonarr temporarily disable an indexer after failures (*back-off*), and when all of them are disabled the search returns nothing. *System > Status* shows a warning like "Indexers unavailable due to failures". Testing the indexers again (*Settings > Indexers > Test*) clears that state.

1. **Check the match.** Open the item and verify the title and year (remakes share titles).
2. **Run an Interactive Search** (person icon on the item):
   * **No results**: go to step 3.
   * **Results, all rejected** (red icon): hover the icon to see the reason (quality profile, minimum seeders, size limits, blocklist).
   * **The grab fails**: see [Grabbed but not in qBittorrent](#grabbed-but-not-in-qbittorrent).
3. **Compare with Prowlarr.** Search the same title in Prowlarr's *Search* page, then open Prowlarr's *History* right after an Interactive Search in Radarr/Sonarr. History shows every query, the app that sent it, whether it searched by ID (`IMDb tt…`) or by text, and what each indexer returned.
   * Prowlarr finds it, Radarr gets nothing and History has **no rows from Radarr**: check *Settings > Indexers* in Radarr. The synced indexers must have *Enable Interactive Search* and *Enable Automatic Search* checked and **no tags** (an indexer with a tag is only used for items with the same tag).
   * Prowlarr finds nothing either: the indexers are down or blocked. Indexers behind Cloudflare need [FlareSolverr](README.md#flaresolverr-optional).
4. **Search everything that is stuck:** *Wanted > Missing > Search All*.

**Keep the indexer list lean.** Anime-only indexers (Nyaa, nekoBT) add nothing to movie searches, and a slow indexer (for example Internet Archive, which only holds public-domain content) delays every search. In Prowlarr, remove them from the Radarr sync or give them a tag so they only sync to Sonarr.

**Watch the quality profile.** With the `Any` profile Radarr prefers the highest-ranked release, which can be a 50+ GB remux or an AI-upscaled "2160p". Limit the profile (for example *HD-1080p*) or set maximum sizes in *Settings > Quality*.

### Periodic missing search

A weekly search for missing items covers the "first search failed" case automatically. Create `/usr/local/bin/arr-missing-search.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
source /etc/arr-media-server.env   # RADARR_API_KEY=... and SONARR_API_KEY=...
curl -fsS -X POST http://localhost:7878/api/v3/command \
  -H "X-Api-Key: $RADARR_API_KEY" -H 'Content-Type: application/json' \
  -d '{"name":"MissingMoviesSearch"}'
curl -fsS -X POST http://localhost:8989/api/v3/command \
  -H "X-Api-Key: $SONARR_API_KEY" -H 'Content-Type: application/json' \
  -d '{"name":"MissingEpisodeSearch"}'
```

`/etc/systemd/system/arr-missing-search.service`:

```ini
[Unit]
Description=Search missing items in Radarr and Sonarr

[Service]
Type=oneshot
ExecStart=/usr/local/bin/arr-missing-search.sh
```

`/etc/systemd/system/arr-missing-search.timer`:

```ini
[Unit]
Description=Weekly missing search for Radarr and Sonarr

[Timer]
OnCalendar=Sun 04:00
Persistent=true

[Install]
WantedBy=timers.target
```

```sh
sudo chmod 700 /usr/local/bin/arr-missing-search.sh
sudo chmod 600 /etc/arr-media-server.env
sudo systemctl daemon-reload
sudo systemctl enable --now arr-missing-search.timer
```

Keep it weekly: searching a large library too often can get you rate-limited by the indexers.

---

## Grabbed but not in qBittorrent

```sh
docker exec radarr curl -s -o /dev/null -w '%{http_code}\n' http://qbittorrent:8080   # expect 200
docker exec radarr tail -n 100 /config/logs/radarr.txt
docker logs qbittorrent --tail 100
```

* Test the client in Radarr/Sonarr: *Settings > Download Clients > qBittorrent > Test*.
* **IP ban.** After repeated failed logins qBittorrent bans the client's IP. Restart qBittorrent, fix the credentials and, optionally, whitelist the Docker network in *Tools > Options > WebUI > Bypass authentication for clients in whitelisted IP subnets* (for example `172.16.0.0/12`).
* **Torrent added but stalled:** no seeders, or the torrent port (6881) is not reachable. Check your router and firewall.

---

## Downloaded but not imported

*Activity > Queue* shows a yellow or red icon with the reason.

* **Path not found.** Radarr/Sonarr must see the download at the same path qBittorrent reports: `/data/downloads/...` in both containers. A torrent reporting `/downloads/...` was added before the single-mount layout; move it with *Set location* (see [Migrating an existing install](README.md#migrating-an-existing-install)). With this layout no *Remote Path Mapping* is needed: delete any you have.
* **Access denied.** See [Permissions](#permissions), then [SELinux](#selinux-fedora).
* **Unknown release, sample or extra files.** Use *Manual Import* from the queue.

---

## Duplicated content and disk filling up

Radarr/Sonarr import with **hardlinks**: the file in `downloads` and the file in the library are the same data on disk, so content is stored once and qBittorrent keeps seeding. When a hardlink cannot be created, they silently **copy** instead and the content is duplicated. Run these checks after any change to `compose.yaml`, `.env` or the download settings.

**1. Every container that handles media mounts the same single volume.**

```sh
docker inspect -f '{{.Name}}:{{range .Mounts}} {{.Source}} -> {{.Destination}};{{end}}' qbittorrent radarr sonarr
```

Each one must show `/srv/media -> /data` (plus its own `/config`) and nothing like `-> /downloads` or `-> /data/movies`. Separate mounts are separate filesystems from the container's point of view, even on the same disk.

**2. Downloads and library are one filesystem inside the containers.** The first column (device ID) must be identical on every line:

```sh
docker exec radarr stat -c '%d %n' /data/downloads /data/movies
docker exec sonarr stat -c '%d %n' /data/downloads /data/tvshows
```

**3. qBittorrent writes inside `/data`.** In *Tools > Options > Downloads*:

* *Default Torrent Management Mode*: `Automatic`.
* *Default Save Path*: `/data/downloads`.
* *Keep incomplete torrents in*: disabled, or a folder inside `/data` (for example `/data/downloads/incomplete`).
* Categories (`radarr`, `tv-sonarr`, ...) in the left panel: no custom save path outside `/data/downloads`.
* The *Save path* column of the existing torrents always starts with `/data/downloads`.

**4. Radarr/Sonarr are allowed to hardlink.**

* *Settings > Media Management > Show Advanced > Importing*: *Use Hardlinks instead of Copy* enabled.
* Root folders: `/data/movies` and `/data/tvshows`.
* *Settings > Download Clients > Remote Path Mappings*: empty.

**5. Verify on disk after the next import.** Both names must show the same inode (second column) and a link count of `2` (first column):

```sh
stat -c '%h %i %n' "/srv/media/downloads/<release>/<file>" "/srv/media/movies/<Movie (Year)>/<file>"
```

**6. Periodic health check.** A dry run of `hardlink` reports how much space duplicated copies take. Something close to zero means there is nothing to fix:

```sh
sudo hardlink -c -n /srv/media/downloads /srv/media/movies /srv/media/tvshows | tail -n 5
```

If it reports duplicates, run it again without `-n` to replace them with hardlinks (the content and seeding are not affected). Another quick signal: `du` counts hardlinked data once when it scans the folders together, so the second number must be clearly smaller than the sum of the first three:

```sh
sudo du -sh /srv/media/downloads /srv/media/movies /srv/media/tvshows
sudo du -sh /srv/media
```

**Freeing space.** With hardlinks, deleting a torrent does not free space while the library file exists, and vice versa: space is freed when both are gone. To stop seeding old content automatically, set *Tools > Options > BitTorrent > Seeding Limits* in qBittorrent.

---

## SELinux (Fedora)

The persistent setup is described in the [Fedora guide](README.md#fedora-guide). Quick checks:

```sh
# The policy must expect container_file_t inside the library (not <<none>> or mnt_t)
matchpathcon /srv/media/movies/test.mkv

# No file may carry private MCS categories (should print nothing)
sudo find /srv/media "$(pwd -P)" -context '*:s0:c*'

# Recent denials
sudo ausearch -m AVC -ts recent
```

Fixes:

* `matchpathcon` prints `<<none>>`: `DATA_ROOT` is under `/media`, where the base policy never relabels content. Move it to `/srv/media` (see [Migrating an existing install](README.md#migrating-an-existing-install)).
* Files with categories (`:s0:c123,c456`): `sudo restorecon -RF /srv/media "$(pwd -P)"` with the stack stopped. The `-F` is required.
* Never add `:z` or `:Z` to a volume of this stack.

Gotchas:

* `mv` keeps the original label: a file moved from your home folder keeps `user_home_t` and containers cannot read it. Use `cp`, or run `restorecon -RF` on it afterwards.
* `chcon` is temporary: the next `restorecon` reverts it. Use `semanage fcontext` rules.

---

## Permissions

All LinuxServer containers run as `PUID:PGID` from `.env`, with `UMASK=002`. Seerr runs as UID 1000 and ignores them.

```sh
id                                     # your UID/GID, normally equal to PUID/PGID
ls -ld /srv/media/{downloads,movies,tvshows}

# Write test as the app user (abc is the PUID user inside LinuxServer images)
docker exec -u abc radarr touch /data/movies/.wtest && echo OK && docker exec radarr rm /data/movies/.wtest
docker exec -u abc qbittorrent touch /data/downloads/.wtest && echo OK && docker exec qbittorrent rm /data/downloads/.wtest
```

If `touch` fails while the Unix permissions look right, the cause is SELinux.

---

## Backups

The Arr apps keep their backups in `/config/Backups` (persisted in `./<service>/config`). Older versions of this guide used `/data/backup`; if *System > Status* reports backup errors, set *System > Backup > Folder* back to the default `Backups` in Radarr, Sonarr and Prowlarr (Prowlarr no longer mounts `/data`). Also check that Radarr/Sonarr's *Settings > Media Management > Recycling Bin*, if set, points inside `/data`.

---

## Seerr

* *Settings > Services > Radarr/Sonarr > Test*. Hostnames are `radarr` and `sonarr`; the root folders are `/data/movies` and `/data/tvshows`.
* A request stays as *Processing* after the file is already in Jellyfin: run *Settings > Jobs > Jellyfin Full Scan*.
* Set `LOG_LEVEL=debug` in `compose.yaml`, recreate the container and follow `docker logs -f jellyseerr` while making a request.

---

## Jellyfin

* Library folders must be `/data/movies` and `/data/tvshows`. Run *Dashboard > Libraries > Scan All Libraries* and check *Dashboard > Logs*.
* Not visible after a scan: `docker exec -u abc jellyfin ls /data/movies`. If that fails, see [Permissions](#permissions) and [SELinux](#selinux-fedora).
* Check whether a playback is transcoding at all in *Dashboard > Activity*. *Direct Play* does not use the GPU.

**VAAPI stopped working** (`vainfo` fails or transcodes fall back to the CPU):

```sh
ls -l /dev/dri/by-path
lspci | grep -E 'VGA|3D'
docker exec jellyfin /usr/lib/jellyfin-ffmpeg/vainfo --display drm --device /dev/dri/renderD128
```

* On machines with two GPUs the `renderD*` numbers depend on the order in which the drivers load, and can swap after a kernel or driver update. Match the PCI address of the GPU with its node and update `VAAPI_DEVICE` in `.env` and the device in Jellyfin's transcoding settings.
* On laptops with a MUX switch, the integrated GPU disappears in "discrete GPU only" mode. Keep the hybrid mode to use VAAPI.

---

## Reverse proxy

```sh
docker compose ps caddy
docker logs caddy --tail 50
curl -s -o /dev/null -w '%{http_code} %{url}\n' http://radarr.localhost/
```

* **Connection refused:** Caddy is not running. Check `COMPOSE_PROFILES` in `.env` and whether port 80 is taken: `sudo ss -ltnp | grep ':80 '`.
* **502 Bad Gateway:** wrong upstream in `caddy/Caddyfile`, or the target container is down. Upstreams use container hostnames and internal ports (`jellyseerr:5055`, `qbittorrent:8080`, ...).
* **qBittorrent answers "Unauthorized":** see [Reverse proxy](README.md#reverse-proxy-optional) in the README.
* **A host development app answers 502:** it must listen on `0.0.0.0`, not only on `127.0.0.1`.
* Changes to the Caddyfile need `docker compose restart caddy`.

---

## Get notified before it bites

In Radarr and Sonarr, add a connection in *Settings > Connect* (ntfy, Telegram, Discord, email, ...) with the **On Health Issue** event. You will hear about failing indexers, a broken download client or backup errors when they happen, instead of days later when a request never arrives.

---

## Cheat sheet

```sh
docker compose ps                                        # status and health
docker logs -f --tail 100 <service>
docker inspect -f '{{.Name}}:{{range .Mounts}} {{.Source}} -> {{.Destination}};{{end}}' qbittorrent radarr sonarr
sudo hardlink -c -n /srv/media/downloads /srv/media/movies /srv/media/tvshows | tail -n 5
sudo ausearch -m AVC -ts recent                          # SELinux denials
sudo find /srv/media -context '*:s0:c*'                  # leftover private labels
docker exec -u abc <service> touch /data/<folder>/.wtest # write test
docker exec radarr curl -s http://prowlarr:9696/ping     # container-to-container connectivity
```
