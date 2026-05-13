# Prerequisites

The lab runs entirely in Docker, so you only need three things on your laptop:
Docker, a terminal, and a modern web browser.

## Required Software

| Tool | Minimum Version | Why |
|------|-----------------|-----|
| Docker Desktop (Mac/Windows) or Docker Engine (Linux) | 24.x | Runs MongoDB and mongo-express |
| A terminal (Terminal.app, iTerm2, Windows Terminal, GNOME Terminal) | -- | Driving `docker` and `mongosh` |
| A web browser (Chrome, Firefox, Safari, Edge) | Current | Accessing mongo-express at `http://localhost:8081` |

## Hardware Virtualization

Docker needs **hardware virtualization (Intel VT-x / AMD-V)** enabled at the
BIOS / UEFI level. On many corporate laptops this is **disabled by default**
and is the single most common reason Docker won't start on training day.

Confirm it before the session:

- **Windows**: open **Task Manager → Performance → CPU**. The bottom of the
  panel must say `Virtualization: Enabled`.
- **macOS**: Apple Silicon has virtualization always on. Intel Macs:
  `sysctl kern.hv_support` — must return `1`.
- **Linux**: `lscpu | grep Virtualization` — must show `VT-x` or `AMD-V`.

If `Disabled`, follow [Troubleshooting](04-troubleshooting.md) **before** the
training day. Resolving BIOS settings often needs IT support, so allow time.

## Verification

Open a terminal and run:

```bash
docker --version
docker compose version
```

You should see output similar to:

```text
Docker version 24.0.x, build ...
Docker Compose version v2.x.x
```

If either command is missing, install Docker Desktop from
<https://docs.docker.com/get-docker/> and restart your terminal.

## Required Ports

The lab binds to these local ports. Make sure nothing else is using them:

| Port | Used By |
|------|---------|
| 27017 | MongoDB |
| 8081 | mongo-express |

Check whether the ports are free:

```bash
lsof -i :27017
lsof -i :8081
```

No output means the port is free. If a process is using either port, stop it
before starting the lab.

## Disk Space

The MongoDB 7 image is roughly 800 MB. Allow at least 2 GB free disk space for
images, data volumes, and the docker overlay filesystem.

## Network

The first `docker compose up` will pull `mongo:7` and `mongo-express:1.0.2` from
Docker Hub. On a slow Wi-Fi this can take 5-10 minutes — do this **before** the
training day, not during the first break.

## Next Steps

- [Lab Setup](02-lab-setup.md) — bring up the stack and connect
- [Troubleshooting](04-troubleshooting.md) — if Docker won't start
