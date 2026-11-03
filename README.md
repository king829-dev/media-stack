# Media stack

A Docker Compose media server: torrents through a VPN, the *arr apps to find
and organise movies, TV and subtitles, Jellyfin to watch it, plus a request
portal, a queue janitor, drive-health monitoring and an optional tunnel for
off-LAN access. Runs happily on a Raspberry Pi 5 with a disk array; runs on
anything else too.

## What is in it

| Service | Port | What it does |
|---|---|---|
| gluetun | 8080, 6881 | WireGuard VPN container. qBittorrent lives inside its network namespace, so torrent traffic cannot leave except through the tunnel. |
| qBittorrent | via gluetun 8080 | Download client. |
| Prowlarr | 9696 | Indexer manager; syncs indexers to Radarr/Sonarr. |
| FlareSolverr | 8191 | Solves Cloudflare challenges for indexers that need it. |
| Radarr | 7878 | Movies. |
| Sonarr | 8989 | TV. |
| Bazarr | 6767 | Subtitles for both. |
| Unpackerr | none | Extracts rar/zip releases so the arrs can import them. |
| Cleanuparr | 11011 | Removes stalled, fake and malware downloads and re-searches. |
| Jellyfin | 8096 | Media server. |
| Jellyseerr (Seerr) | 5055 | "Request a movie" portal for the household; talks to the arrs. |
| Recommendarr | 3000 | Optional. AI recommendations based on your library. |
| Jellystat | 3001 | Optional. Watch statistics for Jellyfin. |
| Scrutiny | 8085 | Optional. SMART health dashboard for your drives. |
| cloudflared | none | Optional. Cloudflare Tunnel for public hostnames without opening ports. |

Delete any optional service you do not want; nothing else depends on it.

## Layout

- **Config:** `${CONFIG_ROOT}/<service>` bind mounts. This tree is the whole
  state of the stack (databases, API keys, watch history). Back it up nightly
  to different physical media than the boot disk.
- **Media:** `${MEDIA_ROOT}` mounted at `/data` in every container that
  touches media. Suggested tree:

  ```
  /data
  ├── torrents/      <- qBittorrent save path
  ├── movies/        <- Radarr root folder
  └── tv/            <- Sonarr root folder
  ```

  Keeping all three under one mount lets Radarr/Sonarr hard link on import
  (instant, no duplicate disk use) and keeps seeding while you watch.

## Hardware notes (Raspberry Pi 5)

Learned the expensive way. If you are building this on a Pi, start here.

### Skip the SATA HAT, go straight to a USB DAS

The Pi 5 SATA HATs (Radxa Penta and similar) look like the tidy option and
are a dead end for a media server:

- They take 2.5-inch drives only, so you are buying SSDs. Four 1 TB SSDs
  filled in months and every new file had to go to the external array anyway.
- The stack needs the HAT's top board for its fan and display, and the Pi +
  HAT + upright drives tower fits no off-the-shelf case or rack tray. A
  weekend of searching and printing did not produce one.
- Nothing carries over when you outgrow it.

A 4-bay USB DAS such as the **Terramaster D4-320** is the better first
purchase. Each bay has its own USB bridge chip; the Pi sees four ordinary
disks over UAS at 5 Gbps with no `usb-storage.quirks` or kernel tweaks.
Cable gotcha: the Pi 5 has no USB-C host port and the box ships with a C-to-C
cable, so buy a **USB-A to USB-C 3.2 Gen 2** cable with it. Keep the DAS
within that cable's reach of the Pi.

### Drives: used or recertified enterprise 3.5-inch

Refurbished Toshiba MG07/MG08 and Seagate Exos 14 TB drives from a reputable
reseller cost a fraction of new consumer drives, are built for 24/7 duty, and
the recertified ones come with a real warranty. Mixed models are fine. Buy
one more drive than you think you need: the parity disk must be at least as
large as the largest data disk.

### Layout that works

- Pool the data drives with **mergerfs** so `MEDIA_ROOT` is one big tree.
- Keep one drive as **SnapRAID** parity so a dead drive is recoverable. Run
  `snapraid sync` nightly and a scrub weekly.
- Boot, Docker's data-root and `CONFIG_ROOT` on a small SSD or NVMe, never
  the SD card.

### Two things that will bite

1. **Never run `smartctl -x` / `--xall` against a drive behind the DAS
   bridge.** The extended SMART pages hang the ASMedia bridge, the bridge
   resets, and real reads time out across the whole array. `smartctl -a` is
   safe. Scrutiny's collector runs `--xall` by default on every container
   recreate, so if you keep the Scrutiny service, drop a
   `collector.yaml` in `${CONFIG_ROOT}/scrutiny/` that overrides the args
   for each DAS drive:

   ```yaml
   devices:
     - device: /dev/sdX
       type: sat
       metrics_smart_args: --all --json
   ```

   Use stable `/dev/disk/by-id/` paths if drive letters move between boots.

2. **Power the Pi off before power-cycling the DAS.** If the array
   disappears while the Pi is up, mergerfs keeps writing to the now-empty
   mountpoints on the boot disk, and you get to untangle that by hand.

## First deploy

1. **Storage first.** Have `MEDIA_ROOT` mounted and healthy before starting
   anything. Containers that start against an empty mountpoint will happily
   write to the boot disk.
2. `cp .env.example .env` and fill in paths, `PUID`/`PGID`, the WireGuard
   key, and the Jellystat secrets. Leave the two arr API keys blank for now.
3. `mkdir -p "$CONFIG_ROOT"` and make sure `PUID:PGID` owns it.
4. `docker compose up -d`. With Portainer: add the folder as a git stack and
   enter `.env` values as stack environment variables.
5. **Configure, in this order:**
   - qBittorrent: `http://<host>:8080`. Default password is in the container
     log (`docker logs qbittorrent`). Set save paths under `/data/torrents`,
     then apply the two settings in the gotchas section below.
   - Prowlarr: add indexers. Settings -> Indexers -> add FlareSolverr at
     `http://flaresolverr:8191` and tag indexers that need it.
   - Radarr / Sonarr: root folders `/data/movies` and `/data/tv`. Download
     client: qBittorrent at host `gluetun`, port 8080 (not `qbittorrent`;
     it has no network of its own). Then in Prowlarr, Settings -> Apps, add
     both so indexers sync.
   - Bazarr: link Sonarr and Radarr, add subtitle providers.
   - Jellyfin: `http://<host>:8096`. Libraries from `/data/movies` and
     `/data/tv`. Create users.
   - Jellyseerr: `http://<host>:5055`. Sign in with Jellyfin, link Radarr
     and Sonarr.
6. Copy the Sonarr and Radarr API keys (Settings -> General) into `.env`,
   then `docker compose up -d unpackerr` to restart it with them.
7. Optional: Cleanuparr at `:11011` (add the arrs and qBittorrent), Scrutiny
   at `:8085` (edit the `devices:` list to match `lsblk` first), tunnel
   hostnames in the Cloudflare dashboard.

## Portainer (optional, recommended)

Portainer gives you a web UI for the stack, a place to keep the secrets that
is not a file on disk, and a one-click redeploy. The way that has held up:

1. **Install Portainer CE** on the Pi with its data volume on the SSD, not
   the SD card.
2. **Fork this repo** (or push a copy to your own GitHub) so edits are yours.
3. **Stacks -> Add stack -> Repository.** Repository URL is your fork,
   compose path `docker-compose.yml`. Enter every variable from
   `.env.example` under *Environment variables* instead of creating a
   `.env` file. If the fork is private, give Portainer a GitHub
   fine-grained personal access token scoped to that one repo with
   *Contents: read-only*, one token per stack, and keep it in your
   password manager.
4. **Leave GitOps automatic updates off.** Redeploy on purpose with the
   *Pull and redeploy* button after you change the repo. Unattended
   redeploys of a stack that holds your VPN and DNS are how you lose a
   quiet evening.

Gotchas:

- *Pull and redeploy* recreates **every** service in the stack, which kills
  active Jellyfin streams for ~15 seconds. Do it in quiet hours. To restart
  one service, use its own container page instead.
- The stack's environment variables exist only inside Portainer's data
  volume. Back that volume up, or at minimum keep every value in a password
  manager, or a dead SD card takes your VPN key and API keys with it.
- Stacks deployed through Portainer must use prebuilt images; a `build:`
  section fails. Everything in this compose is image-only already.
- If you ever script a redeploy through the Portainer API, pass the
  existing environment variables back in the request. A git redeploy call
  without them wipes the stack's env.

## Gotchas that cost real hours

### qBittorrent inside gluetun: two settings that live in qBittorrent's own config

They are in `qBittorrent.conf` under the qbittorrent config dir, not in
compose. If that config is ever lost, downloads break in these exact ways.

1. **Bind to the VPN address.** Advanced -> Network interface: leave blank,
   and set "Optional IP address to bind to" to gluetun's `tun0` address
   (find it with `docker exec gluetun ip -4 -brief addr show tun0`; usually
   `10.5.0.2`). In the file that is `Session\InterfaceAddress=10.5.0.2`.
   Without it libtorrent pins its sockets to the Docker bridge interface,
   packets leave with a source address the tunnel will not carry, and you get
   `dht_nodes = 0` forever plus every tracker stuck at "not contacted".
   Binding by interface *name* (`tun0`) does not work; libtorrent fails to
   bind on a point-to-point device. Use the address, and re-check it if the
   VPN subnet ever changes.

2. **Disk IO type = POSIX-compliant** (Advanced -> Disk IO type; `disk_io_type: 2`
   via the API). Only needed if `MEDIA_ROOT` is a mergerfs pool mounted with
   `cache.files=off`: libtorrent 2.x defaults to memory-mapped file I/O,
   which fails there with `file_mmap ... error: No such device` and sends
   every torrent to `error` state the moment it has metadata. The other fix
   is remounting mergerfs with `cache.files=partial`, which means stopping
   everything that uses the pool; the qBittorrent setting is easier.

qBittorrent rewrites `qBittorrent.conf` on shutdown. Always stop the
container before editing the file by hand, or your edits are discarded.

### Jellyfin version pin

`lscr.io/linuxserver/jellyfin:latest` jumped a major version (10 -> 12)
silently one day and broke every client that used the old auth flow,
including Jellyseerr 2.x. The compose pins an exact build. When you bump it,
check that your TV/phone clients and Jellyseerr support the new major first.

### Bind mounts that appear after the container starts

If a volume source is itself a mount (a network share, rclone, a late-mounted
disk) and the container starts before it exists, Docker binds the empty
directory and keeps seeing it empty forever, even after the mount appears.
Fix: use the long volume syntax with `bind: { propagation: rslave }` for that
volume, or add `RequiresMountsFor=` ordering on the host.

### Redeploys drop streams

A compose or Portainer redeploy recreates every service in the stack, which
kills active Jellyfin streams for ~15 seconds. Do it in quiet hours.

## Backups

The whole stack's state is `CONFIG_ROOT`. A nightly job that stops the
SQLite-backed containers (jellyfin, sonarr, radarr, prowlarr, jellyseerr,
qbittorrent) for the ~2 minutes a tar takes, then restarts them, gives you a
consistent copy including watch history. Keep gluetun up throughout; nothing
downloads during the pause anyway. Exclude caches, transcodes and logs.
Keep secrets (`.env`, Portainer's data volume) in a separate archive that
never leaves the machine.
