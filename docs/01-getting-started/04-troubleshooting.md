# Troubleshooting

Common setup blockers that surface before or during the lab, and how to resolve
them. The biggest single cause of "Docker won't start" on training day is
**hardware virtualization being disabled in BIOS/UEFI** — especially on
corporate-issued laptops where it's off by default. Walk this guide in order.

## Symptom

Docker Desktop fails to start, or the container engine refuses to launch with
errors like:

- `Docker Desktop requires a virtualization technology`
- `WSL 2 installation is incomplete`
- `Hyper-V is not available`
- `VT-x is disabled in the BIOS`

These are not MongoDB errors — they mean the hypervisor underneath Docker
cannot get hardware acceleration. Three layers can be wrong; check in order.

## Layer 1 — BIOS / UEFI (Most Common)

Virtualization (Intel VT-x / AMD-V / SVM) is shipped **disabled** on many OEM
laptops, especially corporate and government-issued models from HP EliteBook,
Dell Latitude, and Lenovo ThinkPad.

### Steps

1. Restart the laptop and enter BIOS/UEFI setup. The key varies — try **F2**,
   **F10**, **F12**, or **Del** while the vendor logo is showing.
2. Look for a setting named one of:
   - **Intel VT-x**
   - **Intel Virtualization Technology**
   - **AMD-V**
   - **SVM Mode**
   - **Virtualization** / **Virtualization Technology**
3. Set it to **Enabled**.
4. Save and exit (usually **F10**).

### Where to Find It By Vendor

| Vendor | Path |
|--------|------|
| Lenovo | Security → Virtualization |
| HP | System Configuration → Device Configurations → Virtualization Technology |
| Dell | Advanced → CPU Configuration → Intel Virtualization |
| ASUS | Advanced → CPU Configuration → SVM Mode |
| MSI | Advanced → CPU Configuration → SVM Mode |

### Confirm It Worked

In Windows, open **Task Manager → Performance → CPU**. Look at the bottom
right — `Virtualization: Enabled`. If it still says `Disabled` after a BIOS
toggle, the BIOS setting didn't save — re-enter BIOS and verify.

## Layer 2 — Windows Features (Windows Hosts Only)

If BIOS is correct but Docker still complains, Windows itself needs the right
features turned on. Docker Desktop on Windows uses either the **WSL 2**
backend (recommended) or **Hyper-V**.

### Quick Check

Open **PowerShell as Administrator**:

```powershell
systeminfo | findstr /C:"Hyper-V"
```

Reading the output:

| Line | Meaning |
|------|---------|
| `Virtualization Enabled In Firmware: Yes` | BIOS is correct (Layer 1 ✓) |
| `Virtualization Enabled In Firmware: No` | Go back to Layer 1 |
| `A hypervisor has been detected. Features required for Hyper-V will not be displayed.` | Hyper-V is already active — good for Docker |

### Enable WSL 2 (Recommended)

```powershell
wsl --install
```

That single command on Windows 10 21H2+ / Windows 11 enables both
`Microsoft-Windows-Subsystem-Linux` and `VirtualMachinePlatform` features
and installs a default Ubuntu distro. **Restart when prompted.**

Manual fallback if `wsl --install` is blocked by group policy:

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

After restart, update the kernel and set the default WSL version:

```powershell
wsl --update
wsl --set-default-version 2
```

Then in **Docker Desktop → Settings → General**, tick
**"Use the WSL 2 based engine"**.

## Layer 3 — Hypervisor Conflicts

If a participant has previously installed **VirtualBox**, **VMware Workstation**,
or enabled **Hyper-V** manually, the hypervisors can fight each other. Symptom:
Task Manager shows `Virtualization: Enabled`, BIOS is correct, but Docker
still won't start.

Toggle the Windows hypervisor flag from **PowerShell as Administrator**:

```powershell
# Disable Hyper-V (use VirtualBox-style direct virtualization)
bcdedit /set hypervisorlaunchtype off
# ...then restart

# Or enable Hyper-V (required by Docker Desktop's Hyper-V backend)
bcdedit /set hypervisorlaunchtype auto
# ...then restart
```

> **Tip**: With Docker Desktop's **WSL 2** backend, you do **not** need the
> Hyper-V Windows feature enabled — WSL 2 has its own lightweight hypervisor.
> This is why WSL 2 is the recommended backend for the lab.

## Pre-Training Mitigation

Add a virtualization check to the prerequisites email so participants resolve
this **before** the training day, not in the first 30 minutes of it. Suggested
wording:

> Sila pastikan virtualization (VT-x/AMD-V) **enabled** dalam BIOS sebelum
> hari training. Untuk check di Windows: buka **Task Manager → Performance
> → CPU** → tengok bahagian bawah, kena tertulis `Virtualization: Enabled`.
> Kalau `Disabled`, ikut [troubleshooting guide ini](04-troubleshooting.md)
> atau hubungi IT support anda.

## Backup Plans (Day-of)

When a participant arrives and can't resolve the virtualization issue in the
first 10 minutes, do not let them stall the whole session. Pick one:

### Plan A — Pair Programming

Pair the affected participant with someone whose Docker is already working.
Share-screen style: the stuck participant **types** the commands on the
working laptop. They still get the muscle memory; they just borrow the engine.

### Plan B — MongoDB Atlas Free Tier

Sign up for a free **M0** cluster at <https://www.mongodb.com/cloud/atlas>
(5 minutes). Use the connection string in `mongosh` directly:

```bash
mongosh "mongodb+srv://<user>:<pwd>@<cluster>.mongodb.net/library"
```

All exercise commands work unchanged — only the connection string differs.
When venue Wi-Fi is decent, this is often **smoother** than getting Docker
running on a contested laptop.

> **Caveat**: Atlas is shared infrastructure. Skip Exercise 3's `mongodump` /
> `mongorestore` tasks (M0 free tier restricts these), or run them on a
> peer's working Docker instance.

### Plan C — Vendor / Lab Workstation

If the venue provides workstations with Docker pre-installed (or you ship a
spare laptop), pre-load the lab kit on it. The stuck participant uses the
spare; you get back the half-hour you would have lost.

## Other Common Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Bind for 0.0.0.0:27017 failed: port is already allocated` | Another MongoDB / process is using port 27017 | `lsof -i :27017` (Mac/Linux) or `netstat -ano \| findstr :27017` (Windows), then stop that process |
| `mongo-express` keeps restarting | `mongo-lab` isn't healthy yet | Wait 10 seconds, refresh; `docker logs mongo-lab` to see why |
| `Authentication failed` in `mongosh` | Typo in password — `ChangeMe123` vs `ChangeMe123!` | Re-check; the `!` is part of the password |
| `pull access denied` on `docker compose up` | Docker Hub rate limit or offline | Sign in (`docker login`), or pre-stage images via USB |
| Slow `docker compose up` first run | Image pull on slow Wi-Fi | Always do the first pull **before** the session, not during it |

## Related Documentation

- [Prerequisites](01-prerequisites.md) — what to confirm before installing
- [Lab Setup](02-lab-setup.md) — the standard happy-path setup
- [Training Checklist](../../todo.md) — pre-training items the trainer owns
