# Tilez Hub

Tilez Hub runs on a machine in your home and lets a web browser show a Tilez dashboard. It serves the Tilez app to the browsers paired with it, and it is their bridge to your home network: requests to your hubs and APIs, live connections and camera streams go through it, because a browser cannot open them itself. It also keeps your dashboards, so that a browser or a Tilez app can load and save them there.

The image is `ghcr.io/activa/tilez-hub`, for 64-bit ARM (a Raspberry Pi 4 or 5 with a 64-bit system) and x86-64. Docker or Podman is all a machine needs. It is meant for Linux: a Raspberry Pi, a NAS, a home server.

## Install

**With Docker:**

```sh
docker run -d --name tilez-hub --network host --restart unless-stopped \
  -v tilez-hub-data:/data ghcr.io/activa/tilez-hub:latest
docker logs tilez-hub
```

**With Podman,** run the same command with `podman` instead of `docker`. A container started this way comes back after a reboot only when Podman's restart service is on:
- as root: `systemctl enable podman-restart`;
- rootless: `systemctl --user enable podman-restart` and `loginctl enable-linger`.

The log shows the addresses a browser on your home network opens, for example `http://192.168.1.10:8080/`. Open one: the browser is paired, without a code, and shows Tilez's first-start page.

- **The network:** `--network host` lets the hub reach the devices on your network as the machine does, and announce itself so that the Tilez apps find it. It is needed: through Docker's port mapping (`-p`) the hub cannot tell which devices are on your home network. On Docker Desktop (Windows, macOS), turn on host networking first (Settings → Resources → Network); this has not been tried.
- **Its data** (paired devices, dashboards) is in the volume `tilez-hub-data`. Keep that volume when updating.
- **Settings** go in `-e NAME=value` options, or in a file of `NAME=value` lines given with `--env-file <file>`: see [Settings](#settings). For example, `-e CASA_PORT=8090` when another program uses port 8080.
- **The command line:** `docker exec -it tilez-hub node cli.cjs <command>`; see [Command line](#command-line).
- **Versions:** `latest` is the newest. You can pin one instead, such as `ghcr.io/activa/tilez-hub:1.0.1` (the newest 1.0.1) or a full version such as `1.0.1.11071`.

**To update:**

```sh
docker pull ghcr.io/activa/tilez-hub:latest
docker rm -f tilez-hub
docker run -d --name tilez-hub ...      # the same command as above, with your settings
```

**With Docker Compose** (or `podman compose`), put this `compose.yaml` in a folder of its own:

```yaml
services:
  tilez-hub:
    image: ghcr.io/activa/tilez-hub:latest
    network_mode: host
    restart: unless-stopped
    volumes:
      - tilez-hub-data:/data
    # Settings, for example:
    # environment:
    #   CASA_PUBLIC_URL: https://tilez.example.com/

volumes:
  tilez-hub-data:
```

Then run `docker compose up -d`. To update: `docker compose pull && docker compose up -d`. For the command line: `docker compose exec tilez-hub node cli.cjs <command>`.

## Pairing

**On your home network, no code is needed** (`CASA_PAIRING=home`, the default). A browser or a Tilez app at a private address (`192.168.x.x`, `10.x.x.x`, `172.16–31.x.x`; IPv6 `fd…`, `fe80…`) is paired the first time it opens the hub. A VPN that gives your phone a private address counts as home.

**From anywhere else, a pairing code is needed.** A code works once, for ten minutes. You get one:
- from a paired browser, on the admin page (`/admin`, under "Paired devices");
- or with `docker exec tilez-hub node cli.cjs pair`.

Set `CASA_PAIRING=code` to require a code everywhere.

**The Tilez app** pairs the same way: in its settings, add a Tilez Hub. Hubs on your network are listed; you can also type the address.

## From outside your home, or with HTTPS

Put the hub behind a **reverse proxy** that has a certificate browsers trust (Let's Encrypt), such as nginx Proxy Manager, Caddy or Traefik. The hub has no HTTPS of its own.

- **Forwarding:** forward `https://tilez.example.com` to `http://<the machine>:8080`. Turn WebSockets on, and send the usual `X-Forwarded-For` and `X-Forwarded-Proto` headers and the `Host` the browser asked for.
- **A proxy on another machine:** give its address, for example `CASA_PROXY_ADDRESSES=192.168.1.5`. A proxy on the same machine needs nothing.
- **The address:** set `CASA_PUBLIC_URL=https://tilez.example.com/`, so that the hub announces it.
- **Who can reach it:** consider an access list in the proxy that allows only your own network. Otherwise the hub is reachable from the internet. Only paired devices get in, but read [What a paired device can see](#what-a-paired-device-can-see) first.

## Settings

| Variable | Default | What |
|---|---|---|
| `CASA_PORT` | `8080` | The port the hub listens on (plain HTTP) |
| `CASA_PAIRING` | `home` | `home`: devices on your home network are paired without a code, others with one. `code`: a code for every device. `open`: no code for anyone who reaches the hub; use it only when a reverse proxy decides who gets through |
| `CASA_USERS` | `off` | `on`: devices sign in with a user's name and password instead of pairing. Make the first admin with `cli.cjs user add <name>` before turning it on |
| `CASA_HOME_NETWORK` | `on` | `on`: devices on your home network reach the hub directly at `http://<its address>:8080/`. `off`: only through the reverse proxy |
| `CASA_PUBLIC_URL` | | The address the apps use at your reverse proxy (`https://tilez.example.com/`). The hub announces it on your network, so the apps list it |
| `CASA_PROXY_ADDRESSES` | `127.0.0.0/8,::1,172.16.0.0/12` | The addresses or ranges your reverse proxy connects from. The default covers a proxy on the same machine, natively or in Docker |
| `CASA_NAME` | the machine's name | The name the apps show |
| `CASA_MDNS` | `on` | `off`: the hub does not announce itself on your network |
| `CASA_BRIDGE` | `on` | `off`: browsers cannot reach your network through the hub |
| `CASA_NET_ALLOW` | | Addresses or ranges the bridge may reach although they are refused by default, such as `127.0.0.1` for a service on the same machine |
| `CASA_NET_DENY` | | More addresses or ranges the bridge may not reach |
| `CASA_SECRETS` | `file` | `store`: dashboards' credentials are kept apart from the dashboard files, encrypted with a key of the hub's own (`CASA_SECRET_KEY_FILE`, outside the data volume; without it, the credentials are lost) |

By default the bridge reaches your network and the internet. It cannot reach the hub's own machine, link-local addresses, multicast or broadcast. Names your dashboard uses must resolve on the hub's machine: use a full name (`influx.lan`), not a single word (`influx`).

## Command line

Run `docker exec -it tilez-hub node cli.cjs <command>`, where the command is one of these:

| Command | What |
|---|---|
| `pair` | A new pairing code |
| `clients` | The paired browsers and apps, with their roles |
| `revoke <id>` | Unpairs a browser or app and deletes its data |
| `role <id> admin\|reader` | What a device may do: an admin can change dashboards and devices; a reader only shows dashboards |
| `users` | The users, with their roles |
| `user add <name> [--reader]` | A new user: an admin unless `--reader`. The password is asked twice |
| `user password <name>` | A new password; the user's devices are signed out |
| `user remove <name>` | Removes the user and unpairs their devices |

## What a paired device can see

A dashboard holds credentials: hub tokens and camera passwords. Every paired browser and app can read every dashboard the hub keeps, credentials included, and reach your network through the hub. Pair only the devices of people who may have them, and unpair a device you no longer use or have lost (on the admin page, or with `revoke`). Make a wall tablet a reader.

The data volume holds the pairing tokens (as hashes), the dashboards and their credentials. Keep backups of it as private as the dashboards themselves.
