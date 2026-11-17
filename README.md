# Home Server — Proxmox VE Homelab

**English** | [Українська](README.uk.md)

A small home server on **Proxmox VE**. It provides media streaming, file sharing, torrent downloads, network-wide ad blocking, and home automation.

> Status: in production · Updated: 2026-11-17 · This is a living document; new services will be added (see [Roadmap](#7-roadmap)).

## 1. Hardware and network

| Component | Value |
|---|---|
| Server | HP EliteDesk 800 G2 Mini: i5-6500T (Skylake, HD 530), 12 GB RAM |
| Disks | 128 GB SATA SSD for the system and VM/CT disks; 1 TB NVMe for media and backups; 64 GB USB flash drive for a second copy of backups |
| Hypervisor | Proxmox VE 9.2 (Debian 13), `no-subscription` repo |
| Router | Keenetic Hero, LAN `192.168.1.0/24`, domain `zelena.home`, WAN PPPoE behind CGNAT, WAN IPv6 off |
| Clients | Windows PC, Samsung QE65QN85D TV (wired) |

DHCP hands out `192.168.1.33–152`. Servers use static addresses below `.33`. All DNS goes through AdGuard: DHCP DNS points to `.10`, Keenetic itself forwards to `.10` (global `ip name-server`, "Ignore DNSv4 from ISP"). The old built-in Keenetic apps (DLNA, Transmission, SMB) are disabled.

![Proxmox summary](screenshots/proxmox-summary.png)

## 2. Services and addresses

| ID | Service | IP | Access | vCPU / RAM / Disk | Boot order |
|---|---|---|---|---|---|
| — | Proxmox host | 192.168.1.120 | https://192.168.1.120:8006 | — | — |
| CT 104 | AdGuard Home | 192.168.1.10 | http://192.168.1.10 | 1 / 512 MB / 4 GB | 1 |
| VM 100 | Home Assistant OS | 192.168.1.41 | http://192.168.1.41:8123 | 2 / 4 GB / 32 GB | 2 |
| CT 101 | Jellyfin | 192.168.1.11 | http://192.168.1.11:8096 | 4 / 2 GB / 16 GB | 3 |
| CT 103 | Samba | 192.168.1.13 | `\\192.168.1.13\media` | 1 / 512 MB / 4 GB | 4 |
| CT 102 | qBittorrent | 192.168.1.12 | http://192.168.1.12:8090 | 2 / 1 GB / 4 GB | 5 |

All containers are **unprivileged LXC** and all guests start on boot. AdGuard starts first so DNS is ready for the others.

**AdGuard Home.** Network-wide DNS with ad/tracker blocking. Upstreams are Cloudflare and Quad9 over DoH. Local names (`zelena.home`) and reverse lookups go to the router. Blocklists: AdGuard DNS filter + HaGeZi Multi NORMAL.
![AdGuard](screenshots/adguard.png)

**Home Assistant OS.** The home automation hub, running as a full VM. It is reserved for future camera and intercom integration.
![Home Assistant](screenshots/home-assistant.png)

**[Jellyfin](jellyFin/README.md).** The media server, with Intel Quick Sync passed through (`/dev/dri`). Libraries are `/media/movies` and `/media/shows`, with real-time monitoring. Verified: 4K HDR10 **Direct Play** on the TV, and Dolby Vision 8.1 plays as HDR10. Details: [Jellyfin settings, users and plugins](jellyFin/README.md).
![Jellyfin](screenshots/jellyfin.png)

**[Samba](samba/README.md).** SMB2+ file server. NetBIOS is off, so map shares by IP (`net use M: \\192.168.1.13\media /persistent:yes` from a non-admin shell).

| Share | Path | Access |
|---|---|---|
| `media` | `/media` | read/write, user `media` (admin) |
| `backups` | `/backups` | read-only, user `media` |
| `family` | `/media/family` | read/write, users `family` and `media` |

The `family` user sees only the `family` share: `files/` for shared files and `torrents/{movies,shows}/` as a drop box for `.torrent` files (see qBittorrent). Windows cannot connect to one server as two different users at once (error 1219), so use `family` on other family devices. Details: [Samba shares, users and folder structure](samba/README.md).
![Samba](screenshots/samba.png)

**qBittorrent.** Torrent client (Web UI). Listening port 52624, UPnP off. Categories `movies` and `shows` save straight into the Jellyfin libraries; unfinished files go to `downloads/incomplete`.

Family drop box: qBittorrent watches `family/torrents/movies` and `family/torrents/shows`. A `.torrent` file dropped there is added automatically and saved to `/media/movies` or `/media/shows`; the `.torrent` file is then deleted. Family members never need the qBittorrent Web UI.
![qBittorrent](screenshots/qbittorrent.png)

## 3. Storage and permissions

```
/mnt/nvme-pool/                (NVMe, ext4, noatime,nofail)
├── media/        → /media in CT 101, 102, 103
│   ├── movies/   shows/   downloads/{incomplete,complete}
│   └── family/   files/   torrents/{movies,shows}
└── backups/      → Proxmox storage "nvme-backup"; /backups (ro) in CT 103

/mnt/usb-backup/               (USB flash, ext4, noatime,nofail)
├── dump/         → Proxmox storage "usb-backup"
└── host-config/  copies of the weekly host config archives
```

The system SSD holds `local-lvm` (LVM-thin, ~67 GB) for guest disks.

Unprivileged containers map UID 1000 to host UID 101000. The media tree is `101000:101000`, mode `775`. Every service writes as `media` (qBittorrent via a systemd override with `UMask=0002`, Samba via `force user`, Jellyfin as a group member), so all services can read and write each other's files.

Naming for Jellyfin: `shows/Show (Year)/Season 01/Show S01E01.mkv`, `movies/Movie (Year)/Movie (Year).mkv`.

## 4. Backups

| What | Schedule | Retention |
|---|---|---|
| All VMs/CTs (vzdump, snapshot, zstd) → `nvme-backup` | daily 03:00 | 7 daily / 4 weekly / 3 monthly |
| All VMs/CTs → `usb-backup` (USB flash) | Sundays 04:00 | 4 weekly / 3 monthly |
| Host config (`/etc/pve`, network, fstab, …) tar.gz → NVMe | weekly | 60 days |
| Copy of the host config archives → USB flash | weekly, right after the NVMe one | 180 days |

One full set of guest backups is about 3.7 GB, so the USB drive holds the whole history with room to spare. The USB storage is marked `is_mountpoint`: if the drive is unplugged, the job fails instead of filling the system disk, and the server boots fine without it (`nofail`).

To restore: **Datacenter → `nvme-backup` or `usb-backup` → Backups → Restore**.

Media is not backed up because it can be re-downloaded.

![Backups](screenshots/backups.png)

## 5. Potential issues

- **Single NVMe, no redundancy.** Media and the daily backups share one disk. If it fails, the media is gone, and the guests can be restored only from the weekly USB copy (up to a week old).
- **USB copy sits in the same box.** It does not protect against theft, fire or a power surge, and flash drives wear out. Take the drive offsite from time to time, and replace it if backup tasks show write errors.
- **NVMe not mounted.** Because of `nofail`, the host still boots, but the containers see an empty `/media` and backups fail. Check `mountpoint /mnt/nvme-pool`.
- **DNS single point of failure.** If CT 104 is down, the LAN has no DNS. Rollback on Keenetic: `no ip name-server 192.168.1.10`, set DHCP DNS back to the router. Re-enabling WAN IPv6 lets clients bypass AdGuard.
- **Transcoding limits (Skylake).** There is no hardware decoding for HEVC 10-bit or AV1. 4K HDR transcoding falls back to the CPU and stutters, so keep to Direct Play. Image subtitles (PGS) force transcoding; prefer SRT.
- **Dolby Vision profile 5.** Samsung does not support DV, and P5 has no HDR10 fallback, so colours come out wrong. Avoid these releases.
- **Wi‑Fi** is often too slow for 4K remuxes; use cable.
- **CGNAT.** No inbound torrent connections, so downloads are slower, and no direct remote access. Torrent traffic is not behind a VPN.
- **Home Assistant IP from the DHCP pool.** `192.168.1.41` is inside the pool, so pin it in Keenetic (permanent IP) or the address may change.
- **Small system SSD.** LVM-thin can be over-committed; check `lvs` before adding guests.
- **Updates** may change the iGPU group IDs (`render`/`video`) and break Jellyfin hardware acceleration. Back up before `apt full-upgrade`.
- **Security.** Do not port-forward Proxmox, Samba, or qBittorrent. Never commit passwords to this repository.

## 6. Clients

| Client | Connection |
|---|---|
| Windows | `M:` → `\\192.168.1.13\media`, browser → Jellyfin |
| Family devices | `\\192.168.1.13\family`, user `family` |
| Samsung TV | Jellyfin app → `http://192.168.1.11:8096`, wired |

## 7. Roadmap

| Planned | Notes |
|---|---|
| Remote access from phone | CGNAT rules out port forwarding, so use an overlay VPN (e.g. Tailscale) in a small LXC |
| Photo storage | Self-hosted photo backup (e.g. Immich) on the NVMe, with its own backup copy |
| Cameras / intercom | Hikvision + Dahua cameras and the intercom via Home Assistant |
| Offsite backups | The USB copy exists; still missing a copy outside the house (cloud or a disk kept elsewhere) |

## Changelog

| Date | Change |
|---|---|
| 2026-11 | Initial setup: Proxmox, AdGuard, HAOS, Jellyfin, Samba, qBittorrent, backups |
| 2026-11-13 | Screenshots added |
| 2026-11-15 | Jellyfin page: settings, users, plugins |
| 2026-11-15 | `family` share and torrent drop box; `tv` renamed to `shows` |
| 2026-11-16 | Samba page: shares, folder structure, users |
| 2026-11-17 | Weekly second copy of backups on a USB flash drive |
