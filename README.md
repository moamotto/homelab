# Valheim Dedicated Server on a Windows NUC (Docker + Tailscale + playit.gg)

A write-up of how I built a headless Valheim dedicated server on an old Intel NUC, managed remotely, reachable by friends without port forwarding, with an optional server-side networking mod.

> **Heads up:** This documents what worked for my setup, including the mistakes I made along the way. Nothing here is official guidance from Docker, Tailscale, playit.gg, or the mod authors. Verify against current docs before relying on it.

## Contents

- [Goals](#goals)
- [Hardware and software](#hardware-and-software)
- [Architecture](#architecture)
- [Setup](#setup)
  1. [Base machine](#1-base-machine)
  2. [Remote access with Tailscale and RDP](#2-remote-access-with-tailscale-and-rdp)
  3. [Docker Desktop and the Valheim container](#3-docker-desktop-and-the-valheim-container)
  4. [Windows Firewall (the important part)](#4-windows-firewall-the-important-part)
  5. [Public access with playit.gg](#5-public-access-with-playitgg)
  6. [Optional: server-side networking mod](#6-optional-server-side-networking-mod)
- [Backups](#backups)
- [Troubleshooting and lessons learned](#troubleshooting-and-lessons-learned)
- [Security notes](#security-notes)
- [Repo layout](#repo-layout)

## Goals

- Run Valheim on spare hardware, headless (no monitor or keyboard after setup).
- Use Docker so the same machine can host other services later.
- Stay on Windows.
- No router port forwarding (my apartment network is restrictive).
- Manage the box from anywhere without exposing remote desktop to the internet.

## Hardware and software

| Item | Detail |
|---|---|
| Machine | Intel NUC (Skull Canyon generation) |
| CPU | Intel Core i7-6770HQ (quad-core, 2.6 GHz base) |
| RAM | 32 GB DDR4 |
| Storage | 1 TB SATA M.2 |
| OS | Windows 10 Pro |
| Containers | Docker Desktop (WSL2 backend) |
| Game server image | `lloesche/valheim-server` |
| Remote admin | Tailscale + Windows Remote Desktop |
| Public access | playit.gg UDP tunnel |

Windows 10 Pro matters: Windows Home cannot host incoming Remote Desktop sessions.

Note: Windows 10 reached end of mainstream support in October 2025. This NUC's CPU is not on Microsoft's official Windows 11 support list, so I stayed on Windows 10.

## Architecture

```
Friends' game clients
        |
   playit.gg relay (public address, UDP)
        |
   playit agent (runs natively on the NUC)
        |
   127.0.0.1:2456-2458/udp
        |
   Docker Desktop (WSL2) --> valheim-server container
                                  |
                  C:\Servers\valheim\config  (settings, BepInEx, backups)
                  C:\Servers\valheim\data    (server files)

Me (admin)
   |
Tailscale private network --> RDP to the NUC
(RDP is never exposed publicly; the firewall only allows it from Tailscale addresses)
```

## Setup

### 1. Base machine

1. Wire the NUC to the router with Ethernet if you can. Wi-Fi works, but wired avoids a lot of confusion.
2. Set Windows power settings so it never sleeps.
3. Reserve its IP in the router's DHCP settings (using the MAC address of the adapter you actually use).
   - If you switch between Wi-Fi and Ethernet, the MAC address changes and the reservation must be redone.
   - I once also set a static IP on the machine itself while a router reservation existed. That broke networking entirely. Use one method, not both.
4. Optional: configure Windows auto-login so Docker Desktop restarts after a reboot without anyone typing a password.
   - `netplwiz` may hide the needed checkbox if Windows Hello/PIN is enabled. Removing the PIN fixes it.
   - If `netplwiz` silently fails to save, the registry values can be created by hand under
     `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` (`AutoAdminLogon`, `DefaultUserName`, `DefaultPassword`).
   - **Tradeoff:** `DefaultPassword` is stored in plain text, and anyone with physical access gets a logged-in desktop.
   - Fast Startup can interfere with auto-login on anything short of a true cold boot.

### 2. Remote access with Tailscale and RDP

1. Create a Tailscale account and install the client on the NUC and your other devices.
2. In the Tailscale admin console:
   - Enable MagicDNS (so you can use a hostname instead of an IP).
   - Disable key expiry for the NUC so it doesn't silently drop off the network.
   - If a device authenticates but stays offline, check whether device approval is required.
3. On the NUC, enable Remote Desktop (Settings > System > Remote Desktop).
4. In `wf.msc` (Windows Defender Firewall with Advanced Security), edit every inbound "Remote Desktop" rule: on the **Scope** tab, set **Remote IP address** to `100.64.0.0/10` (Tailscale's address range). This restricts RDP to Tailscale devices only.
5. Test from a device on cellular data, not your home Wi-Fi.

Things that went wrong for me:

- Remote Desktop services were set to Manual and `SessionEnv` ("Remote Desktop Configuration") had stopped, so nothing was listening on port 3389 even though the settings toggle was on. Starting `SessionEnv` and rebooting fixed it.
- Check `netstat -ano | findstr :3389` to confirm something is actually listening.
- Windows blocks inbound ping by default. A failed ping does not prove a device is unreachable.

### 3. Docker Desktop and the Valheim container

1. Install WSL2 (`wsl --install`), then Docker Desktop. In Docker Desktop settings, enable "Start Docker Desktop when you log in."
2. Create folders, for example:
   - `C:\Servers\valheim\config`
   - `C:\Servers\valheim\data`
3. Copy `.env.example` to `.env` and fill in your own values (see [Security notes](#security-notes)).
4. Use the `docker-compose.yml` from this repo, then from `C:\Servers\valheim` run:

```powershell
docker compose up -d
docker logs valheim -f
```

On first run the container downloads the server files via SteamCMD, which takes several minutes. You do **not** need to install SteamCMD yourself.

Useful commands:

```powershell
docker ps                      # is it running?
docker port valheim            # are ports mapped?
docker compose stop            # stop
docker compose start           # start again
docker compose down && docker compose up -d   # recreate (needed after editing the compose file)
```

The file name must be exactly `docker-compose.yml`. Notepad likes to save `docker-compose.yml.txt`, which Docker will not find. Turn on "File name extensions" in File Explorer to check.

Valheim password rules: at least 5 characters, and it must not match the server or world name.

### 4. Windows Firewall (the important part)

This cost me the most time. A port rule alone was **not** enough.

1. Create an inbound rule allowing **UDP 2456-2458**, all profiles.
2. Also allow **`com.docker.backend.exe`** (Docker Desktop's backend process) through the firewall. With only the port rule in place, UDP traffic never reached the container, even though `docker port` showed the mappings were correct. Allowing that executable fixed it.

Symptoms when this is wrong: the game client fails instantly with no password prompt, and `docker logs` shows nothing at all when you try to connect.

Diagnostic approach that eventually worked: run a native (non-Docker) Valheim server on the same machine with the same firewall rule. When the native server was reachable and the container wasn't, the problem was clearly in the Docker/Windows path, not the router or the game.

Things that turned out **not** to be the cause for me (so check them last): network profile (Public vs Private), Windows excluded port ranges, NIC checksum offload, Docker Desktop version, WinNAT resets, Hyper-V vs WSL2 backend, third-party antivirus.

Also, a throwaway UDP test container using a flaky listener (BusyBox `nc` in a loop) gave me false negatives and sent me down the wrong path. Verify your test tool before trusting its result.

### 5. Public access with playit.gg

I used playit.gg so friends can connect without port forwarding or installing anything.

1. Create a playit.gg account.
2. Install the playit agent on the NUC (it runs natively on Windows, not in Docker).
3. Create a UDP tunnel pointing at local port `2456`.
4. Give friends the public address playit assigns, entered in Valheim under Join Game > Enter IP.
5. Test from outside your network (for example, a phone on cellular).

Notes:

- `SERVER_PUBLIC: "false"` only controls Steam's in-game server browser listing. It is not related to the tunnel.
- If the agent's Windows service fails to start right after a network change, check basic connectivity first. In my case the real cause was a conflicting static IP, not the agent.
- Don't commit your tunnel's public address or agent secret to a public repo.

### 6. Optional: server-side networking mod

Valheim has a reputation for network lag in busy areas. I tried the community mod **FiresGhettoNetworking** (published on Thunderstore by VerdantsAscent). Per its page, its effects reach all clients whether or not they have it installed, so friends don't have to mod their game. I have not independently benchmarked it, and it is a community project, so treat results as something to measure.

1. Back up your world files first.
2. Add this line under `environment:` in `docker-compose.yml` (already present in this repo's file):

   ```yaml
   BEPINEX: "true"
   ```

3. Recreate the container. It installs BepInEx and creates `config\bepinex\plugins`.
4. Download the mod from Thunderstore (use the latest version, and read its dependency notes), then copy its `.dll` file(s) into `config\bepinex\plugins`. The manifest, icon, and README in the zip are not needed.
5. Restart and check the log:

   ```powershell
   docker logs valheim 2>&1 | Select-String "plugins to load","FiresGhetto"
   ```

   You want to see something like `1 plugin to load` and a line loading the mod, rather than `0 plugins to load`.

Caveats:

- Do not combine it with other networking mods (for example Better Networking); they can conflict.
- Do not combine `BEPINEX` with `VALHEIM_PLUS`.
- Server mods can break after Valheim updates until the author patches them. If the server fails to start after an update, remove the DLL and restart to return to vanilla.
- With a very large world (mine is about 1.4 million ZDOs), expect some load time and load, regardless of the mod.

## Backups

- The container backs up worlds on a schedule (`BACKUPS_CRON`, `BACKUPS_MAX_AGE` in the compose file). Backups land in `config\backups`.
- Copy the `.db`/`.fwl` world files somewhere else before risky changes (mods, upgrades).
- Docker stats only show live usage. To log usage over time, a simple PowerShell loop appending `docker stats --no-stream` output to a CSV works.

## Troubleshooting and lessons learned

| Symptom | What it turned out to be |
|---|---|
| Tailscale shows "needs authentication" after browser says success | Check for pending device approval in the admin console; restart the client; check the clock |
| RDP fails, `netstat` shows nothing on 3389 | `SessionEnv` / `TermService` not running correctly; start services, reboot |
| Game client fails instantly, nothing in container logs | Firewall blocking `com.docker.backend.exe`, not just the UDP ports |
| Everything broke after switching Wi-Fi to Ethernet | A static IP set on the machine conflicting with the router's reservation |
| `docker compose` says no config file found | Wrong working directory, or the file was saved as `.yml.txt` |
| Mod isn't loading | DLL not in `config\bepinex\plugins`, container not recreated, or `BEPINEX` not set |
| Auto-login doesn't work | Registry values missing, Windows Hello PIN in the way, or Fast Startup |

## Security notes

- Never commit `.env`. It holds the server password. This repo ignores it; `.env.example` shows the format.
- Do not publish: your public IP, playit tunnel address or secrets, Tailscale IPs or auth keys, device serial numbers, your players' Steam IDs, or the contents of `config/` and `data/` (world files and logs can contain player identifiers).
- RDP is only reachable over Tailscale. Do not disable the Windows Firewall to "fix" connectivity; that also removes the RDP restriction.
- The Windows auto-login registry password is stored in plain text. Only use it on a machine you physically control.

## Repo layout

```
.
├── README.md             # this file
├── docker-compose.yml    # Valheim server container (reads values from .env)
├── .env.example          # template for your own .env (copy it, don't commit the real one)
└── .gitignore            # keeps secrets and world data out of git
```

## Credits

- [`lloesche/valheim-server-docker`](https://github.com/lloesche/valheim-server-docker) for the container image.
- Tailscale and playit.gg for the networking.
- The FiresGhettoNetworking author (VerdantsAscent) for the server-side mod.
