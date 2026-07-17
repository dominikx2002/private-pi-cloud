# Introduction

## Purpose of this document

This is the **as-built system documentation** for *Private Pi Cloud* — a self-hosted family cloud running on a Raspberry Pi 5. It describes the system as deployed: its hardware, software, architecture, network design, security model, and operating procedures. It is a technical reference for operating, maintaining, and recovering the system.

It is distinct from the project **Wiki**, which is a step-by-step *build guide*. This document instead answers: *"What is this system, how is it put together, and how do I run it?"*

## Scope

The system provides two services to a household:

- **Photo and video backup** (Immich) — an iCloud Photos / Google Photos replacement.
- **File storage and sync** (Nextcloud) — an iCloud Drive replacement.

Both are reached securely from anywhere through a private Tailscale network, run in Docker containers, and store data as plain files on a local NVMe SSD. A 3-2-1 backup protects that data.

## Intended audience

The system administrator (owner) and any technically capable person who may need to operate or recover the system. Basic Linux, SSH, and Docker familiarity is assumed.

---

# System overview

*Private Pi Cloud* replaces commercial cloud services with a single self-hosted appliance. The design goals, in priority order:

1. **Data ownership** — all photos and files reside on the owner's hardware, stored as plain, recoverable files.
2. **Privacy** — nothing is exposed to the public internet; remote access is via a private encrypted network only.
3. **No recurring cost** — no subscriptions; capacity is limited only by the installed disk.
4. **Recoverability** — a proper 3-2-1 backup protects against disk failure, deletion, and disaster.

The system is a single Raspberry Pi 5 running Raspberry Pi OS from an NVMe SSD, hosting all services as Docker containers.

---

# Hardware specification

| Component | Specification | Notes |
|---|---|---|
| Compute | Raspberry Pi 5, 8 GB RAM | Runs both service stacks comfortably |
| Storage | NVMe SSD, M.2 2280 (1 TB) | OS and all data; connected via PCIe |
| NVMe interface | PCIe-to-M.2 HAT / case (Argon NEO 5 M.2) | Attaches the SSD to the Pi's PCIe port |
| Boot / rescue | microSD card (A2) | Used for install; retained as rescue media |
| Power | Official Raspberry Pi 27 W (5 V / 5 A) USB-C | **Required** — see note below |
| Network | Ethernet (primary) / Wi-Fi | Either supported |
| Backup target | External USB SSD/HDD | Second copy for 3-2-1 backup |

**Power supply note.** The Pi 5 with an NVMe SSD under load draws current spikes. An underpowered supply causes the NVMe to drop off the bus, producing I/O errors and risking data loss. Only the official 27 W supply is used; phone chargers, generic adapters, and powered USB hubs are explicitly unsuitable.

---

# Software stack

| Layer | Software | Role |
|---|---|---|
| Operating system | Raspberry Pi OS Lite (64-bit) | Minimal, headless base system |
| Container engine | Docker + Docker Compose | Runs and isolates all services |
| Remote access | Tailscale | Private encrypted network (WireGuard-based) |
| Photos | Immich | Photo/video backup and management |
| Files | Nextcloud | File storage and synchronisation |
| Databases | PostgreSQL (Immich), MariaDB (Nextcloud) | Service metadata |
| Cache | Redis / Valkey | Caching and job queues |
| Backup | restic | Encrypted, deduplicated, scheduled backups |

---

# Architecture

## Component model

All services run as Docker containers on the Pi. Each service stack (Immich, Nextcloud) has its own isolated Docker network; containers within a stack communicate by service name, and the two stacks do not interfere with each other.

- **Immich stack:** server, machine-learning, PostgreSQL, Redis.
- **Nextcloud stack:** application (PHP/Apache), MariaDB, Redis.

Tailscale runs on the host (not in a container) and publishes each service over HTTPS to the private network.

## Data flow

- **Photo backup:** a phone running the Immich app uploads photos to the Immich server, which writes them as plain files to the data directory and records metadata in PostgreSQL.
- **File sync:** a computer or phone running a Nextcloud client syncs files to the Nextcloud application, which stores them as plain files and records metadata in MariaDB.
- **Remote access:** all client-to-server traffic travels through the Tailscale encrypted tunnel; no service is exposed to the public internet.

## Storage principle

Photos and files are stored as **ordinary files** on the NVMe (not encrypted blobs), which maximises recoverability and makes backups straightforward. Confidentiality is provided by the network layer (Tailscale) rather than by encrypting the files at rest.

---

# Network and access

## Addressing

| Service | Internal port | External address |
|---|---|---|
| Nextcloud | 8080 | `https://pi-cloud.<tailnet>.ts.net` |
| Immich | 2283 | `https://pi-cloud.<tailnet>.ts.net:8443` |
| SSH | 22 | `pi-cloud` (via Tailscale) / `pi-cloud.local` (home LAN) |

Immich uses HTTPS port 8443 because Nextcloud occupies the primary HTTPS port (443) on the tailnet address.

## Access modes

- **On the home network** — clients reach the Pi directly over the local LAN (`pi-cloud.local`); Tailscale is not required.
- **Away from home** — clients use Tailscale to tunnel back to the Pi securely.

## Security boundary

No inbound ports are forwarded on the router. The only path to the services is the Tailscale private network, which requires authentication to the owner's Tailscale account. This eliminates exposure to public internet scanning and attacks.

---

# Services

## Immich (photos)

- **Purpose:** automatic phone photo/video backup, timeline, albums, face and object recognition, search.
- **Containers:** `immich_server`, `immich_machine_learning`, `immich_postgres`, `immich_redis`.
- **Compose source:** official Immich release compose file (not hand-written); configured via `.env`.
- **Data location:** `~/data/immich` (plain files, e.g. `library/<user>/YYYY/YYYY-MM-DD/`).
- **Database:** PostgreSQL with vector extension, data under `~/docker/immich/postgres`.
- **Storage template:** default (`{{y}}/{{y}}-{{MM}}-{{dd}}/{{filename}}`) — human-readable layout on disk.
- **Accounts:** one administrator; each family member has a separate private library.

## Nextcloud (files)

- **Purpose:** file storage, synchronisation, sharing.
- **Containers:** `nextcloud` (app), `nextcloud-db` (MariaDB 11.8), `nextcloud-redis`.
- **Compose source:** project-maintained `docker-compose.yml` (in the repository).
- **Data location:** `~/data/nextcloud` (bind-mounted into the container).
- **Database:** MariaDB 11.8 (version pinned; not `lts`, which may pull an unsupported release).
- **Reverse-proxy config:** `TRUSTED_PROXIES` and `OVERWRITE*` settings make Nextcloud work correctly behind Tailscale HTTPS.
- **Accounts:** administrator (server management) plus a separate daily-use account; family accounts as needed.

---

# Storage layout

```
~/docker/                 configuration ("recipes"): compose files + .env
   immich/
   nextcloud/
~/data/                   the actual data (grows large)
   immich/                Immich photo library — plain files
   nextcloud/             Nextcloud user files
```

- **Configuration vs data** are deliberately separated: `~/docker` holds small text files; `~/data` holds the large, irreplaceable content.
- **Databases** live in Docker-managed volumes (not in `~/data`), which persist across container rebuilds.
- **User files and photos** are the bind-mounted `~/data` folders, backed up directly.

The 1 TB NVMe is treated as a single shared pool (no fixed per-user partitions); per-user quotas can be applied within each application if required.

---

# Security model

- **No public exposure.** No router ports are forwarded; access is exclusively through the authenticated Tailscale network.
- **Encrypted transport.** All remote traffic is encrypted by Tailscale (WireGuard). Services are served over HTTPS with valid certificates.
- **Secrets management.** Database passwords are stored in `.env` files, which are excluded from version control (`.gitignore`); only `.env.example` templates are published.
- **Role separation.** Administrative accounts are used only for management; day-to-day use is under separate normal accounts. Each service has its own independent account list.
- **Physical/rescue.** The retained rescue microSD card allows recovery if the NVMe fails; boot order falls back to it automatically.

---

# Backup and recovery

## Strategy

A **3-2-1** backup is used: 3 copies of the data, on 2 different media, with 1 copy off-site.

| Copy | Location | Medium |
|---|---|---|
| 1 (live) | NVMe SSD in the Pi | Internal storage |
| 2 (local) | External USB disk on the Pi | Separate local disk |
| 3 (off-site) | Cloud or a disk stored elsewhere | Remote / removable |

## What is backed up

- **Data:** `~/data/immich` and `~/data/nextcloud`.
- **Database dumps:** consistent exports of the Immich (PostgreSQL) and Nextcloud (MariaDB) databases. Live database files are never copied directly, as they may be inconsistent.
- **Configuration:** `~/docker` (compose files and `.env`).

## Mechanism

The backup engine is **restic** — encrypted, deduplicated, and snapshot-based. A script dumps both databases, then runs a restic backup of data, dumps, and configuration, then prunes old snapshots per the retention policy. The script is scheduled nightly via a systemd timer.

## Recovery

To recover: rebuild the service stacks from the configuration, restore files from the restic repository, and re-import the database dumps. Restores are tested periodically; an untested backup is not considered reliable.

**Critical:** the restic repository password is stored in a password manager, off the Pi. Without it, the encrypted backups are unrecoverable.

---

# Operations and maintenance

## Routine

- **Monthly:** update the OS; update the application containers (one stack at a time); check disk space; confirm backups are running.
- **Periodically:** test a restore from backup; remove unused Docker images.
- **Before major upgrades:** read release notes; verify the previous night's backup succeeded.

## Updates

- **OS:** `sudo apt update && sudo apt full-upgrade -y`.
- **Applications:** in each stack folder, `docker compose pull && docker compose up -d`. Data and databases are preserved across updates.
- **Cautions:** Immich can introduce breaking changes between versions; Nextcloud must be upgraded one major version at a time. Automatic app auto-updaters are deliberately avoided.

## Monitoring

- Container health: `docker compose ps` in each stack folder.
- Disk space: `df -h`.
- Under-voltage history: `vcgencmd get_throttled` (`0x0` = healthy).
- Backup schedule: `systemctl list-timers backup.timer`.

---

# Runbook — common tasks

| Task | Procedure |
|---|---|
| Check service status | `docker compose ps` (in the stack folder) |
| View a service's logs | `docker compose logs -f <service>` |
| Restart a service | `docker compose restart <service>` |
| Update a stack | `docker compose pull && docker compose up -d` |
| Run a backup now | `~/backup.sh` |
| List backup snapshots | `restic -r /mnt/backup/restic --password-file <file> snapshots` |
| Check remote access | `tailscale status` / `tailscale serve status` |
| Reboot the Pi | `sudo reboot` |
| Shut down safely | `sudo poweroff` |

---

# Troubleshooting reference

| Symptom | Likely cause | Action |
|---|---|---|
| I/O errors; commands fail | Under-powered NVMe | Use official 27 W supply; check `vcgencmd get_throttled` |
| Cannot reach services (Tailscale connected) | A second VPN is active | Disable the other VPN |
| `pi-cloud.local` won't resolve | mDNS delay/blocked | Use the Pi's IP; set a DHCP reservation |
| SSH host key warning after rebuild | New host key | `ssh-keygen -R pi-cloud.local`, reconnect |
| Nextcloud container restarting | Wrong MariaDB version | Pin `mariadb:11.8`; recreate empty DB volume |
| "Untrusted domain" in Nextcloud | Domain not trusted | Add tailnet domain to trusted domains |
| Immich container unhealthy on first run | ML model downloading | Wait; check logs |
| Photos won't upload from iPhone | "Optimize iPhone Storage" | Check iOS Photos settings; finish backup on Wi-Fi |
| System won't boot from NVMe | Disk/boot issue | Insert rescue microSD (automatic fallback) |

For detailed troubleshooting, see the project Wiki.

---

# Appendix A — Command reference

```
# System / hardware
vcgencmd get_throttled          under-voltage check (0x0 = OK)
vcgencmd measure_temp           CPU temperature
findmnt /                       confirm root is on the NVMe
lsblk / df -h / free -h         disks, space, memory

# Docker (run in a stack folder)
docker compose ps               container status
docker compose logs -f <svc>    follow logs
docker compose pull             fetch newer images
docker compose up -d            (re)create containers

# Tailscale
tailscale status                tailnet devices
tailscale serve status          HTTPS mappings
tailscale ip -4                 this Pi's tailnet IP

# Nextcloud occ (in container)
docker exec -u www-data nextcloud php occ status
docker exec -u www-data nextcloud php occ maintenance:mode --on|--off

# Backup
~/backup.sh                     run a backup
restic ... snapshots            list snapshots
systemctl list-timers backup.timer
```

# Appendix B — Configuration file locations

| Item | Path |
|---|---|
| Nextcloud compose + env | `~/docker/nextcloud/{docker-compose.yml, .env}` |
| Immich compose + env | `~/docker/immich/{docker-compose.yml, .env}` |
| Immich database files | `~/docker/immich/postgres` |
| Immich photo library | `~/data/immich` |
| Nextcloud files | `~/data/nextcloud` |
| Backup script | `~/backup.sh` |
| Backup schedule | `/etc/systemd/system/backup.{service,timer}` |
| Boot configuration | `/boot/firmware/config.txt` |

# Appendix C — Glossary

| Term | Meaning |
|---|---|
| Tailnet | The owner's private Tailscale network of devices |
| MagicDNS | Tailscale feature for reaching devices by name |
| Bind mount | A host folder mapped into a container |
| Volume | Docker-managed persistent storage |
| 3-2-1 | Backup rule: 3 copies, 2 media, 1 off-site |
| restic | The encrypted, deduplicating backup tool used |
| occ | Nextcloud's command-line administration tool |
