<div align="center">
<img src="https://ui.immich.app/_app/immutable/assets/immich-logo-inline-dark.C4PioLn8.svg" height="70" align="absmiddle"/>&nbsp;&nbsp;&nbsp;&nbsp;
<img src="https://raw.githubusercontent.com/dominikx2002/private-pi-cloud/main/assets/logo_nextcloud_blue.svg" height="70" align="absmiddle"/>&nbsp;&nbsp;&nbsp;&nbsp;
<img src="https://raw.githubusercontent.com/dominikx2002/private-pi-cloud/main/assets/tailscale-grey.svg" height="70" align="absmiddle"/>&nbsp;&nbsp;&nbsp;&nbsp;
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/docker/docker-original-wordmark.svg" height="70" align="absmiddle"/>
</div>

<h1 align="center">Private Pi Cloud</h1>

<p align="center">
A self-hosted family cloud on a <b>Raspberry Pi 5</b> — a full replacement for iCloud that lives in your home, on your hardware, with your data.
</p>

<p align="center">
  <a href="https://www.raspberrypi.com/products/raspberry-pi-5/">
    <img src="https://img.shields.io/badge/Raspberry%20Pi-5-C51A4A?logo=raspberrypi&logoColor=white" alt="Raspberry Pi 5"/>
  </a>
  <a href="https://www.docker.com/">
    <img src="https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white" alt="Docker"/>
  </a>
  <a href="https://github.com/dominikx2002/private-pi-cloud/main/LICENSE.md">
    <img src="https://img.shields.io/badge/License-MIT-green" alt="MIT License"/>
  </a>
</p>

---

## What it is

Photos back up automatically to **Immich**, files sync through **Nextcloud**, and you reach everything securely from anywhere with **Tailscale** — all running in **Docker** on a single Raspberry Pi 5. No subscription, no port-forwarding, and nothing on anyone else's servers. Your photos and files are stored as plain, recoverable files on a fast NVMe SSD, and the whole thing is protected by a proper 3-2-1 backup.

| Service | Role | Address |
|---|---|---|
| **Immich** | Photos & videos — *your iCloud Photos* | `https://pi-cloud.<tailnet>.ts.net:8443` |
| **Nextcloud** | Files & documents — *your iCloud Drive* | `https://pi-cloud.<tailnet>.ts.net` |
| **Tailscale** | Private, encrypted remote access | — |
| **Docker** | Runs every service in isolated containers | — |

## Full guide → the Wiki

**This repo holds the configuration; the complete step-by-step guide lives in the [Wiki](https://github.com/dominikx2002/private-pi-cloud/wiki).** Every command is explained and every design choice has a reason.

1. [Hardware & Design](https://github.com/dominikx2002/private-pi-cloud/wiki/01-%E2%80%90-Hardware-&-Design)
2. [OS on NVMe](https://github.com/dominikx2002/private-pi-cloud/wiki/02-%E2%80%90-OS-on-NVMe)
3. [Docker](https://github.com/dominikx2002/private-pi-cloud/wiki/03-%E2%80%90-Docker)
4. [Tailscale](https://github.com/dominikx2002/private-pi-cloud/wiki/04-%E2%80%90-Tailscale)
5. [Nextcloud](https://github.com/dominikx2002/private-pi-cloud/wiki/05-%E2%80%90-Nextcloud)
6. [Immich](https://github.com/dominikx2002/private-pi-cloud/wiki/06-%E2%80%90-Immich)
7. [Phones & Photo Backup](https://github.com/dominikx2002/private-pi-cloud/wiki/07-%E2%80%90-Phones-&-Photo-Backup)
8. [Backup (3-2-1)](https://github.com/dominikx2002/private-pi-cloud/wiki/08-%E2%80%90-Backup-(3%E2%80%902%E2%80%901))
9. [Troubleshooting](https://github.com/dominikx2002/private-pi-cloud/wiki/09-%E2%80%90-Troubleshooting)
10. [Maintenance & Updates](https://github.com/dominikx2002/private-pi-cloud/wiki/10-%E2%80%90-Maintenance-&-Updates)

> The Wiki links above use special characters in their titles; if one doesn't open, browse from the [Wiki home page](https://github.com/dominikx2002/private-pi-cloud/wiki) instead.

## Repository structure

```
private-pi-cloud/
├── assets/          # images used by the Wiki
├── nextcloud/       # Nextcloud stack: docker-compose.yml + .env.example
├── immich/          # Immich stack: .env.example (compose is pulled from upstream)
├── .gitignore       # keeps real .env files out of git
├── LICENSE          # MIT
└── README.md
```

## Quick start

> Full details, prerequisites and explanations are in the [Wiki](https://github.com/dominikx2002/private-pi-cloud/wiki). This is the short version for the config in this repo.

```bash
# On the Pi, after installing Docker (see Wiki page 03):
git clone https://github.com/dominikx2002/private-pi-cloud.git
cd private-pi-cloud

# Nextcloud
cd nextcloud
cp .env.example .env          # then edit .env with real values
docker compose up -d
```

For Immich, download the official compose file and configure `.env` as described on [Wiki page 06](https://github.com/dominikx2002/private-pi-cloud/wiki/06-%E2%80%90-Immich).

## Trademarks

This is a personal, non-commercial project and is **not affiliated with or endorsed by** any of the projects it uses. Raspberry Pi is a trademark of Raspberry Pi Ltd. Immich, Nextcloud, Tailscale and Docker are trademarks of their respective owners. Logos are used only to identify the software this project is built on.

## License

Released under the [MIT License](LICENSE).

<div align="center">
<sub>Your data, your hardware, your home. @Dominik Serafin</sub>
</div>
