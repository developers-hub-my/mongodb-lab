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

### Plan D — Install MongoDB Natively (No Docker)

When the laptop is too locked down to install Docker at all — no BIOS
access, group-policy blocks, antivirus refuses — install **MongoDB Community
Server** directly. The exercises only depend on `mongosh`, so they all still
work; only the engine underneath changes.

Downloads:

| What | Where |
|------|-------|
| MongoDB Community Server | <https://www.mongodb.com/try/download/community> |
| mongosh (shell) | <https://www.mongodb.com/try/download/shell> |
| Database Tools (`mongoimport`, `mongodump`, `mongorestore`) | <https://www.mongodb.com/try/download/database-tools> |

> **Tip**: On Windows, the **MongoDB Community Server MSI** installer offers
> a "Complete" option that bundles `mongosh` and Compass — usually fastest.
> On macOS and Linux, the package manager paths below pull the same bits.

#### Windows (MSI Installer)

1. Download the MongoDB Community `.msi` from the link above.
2. Run it → choose **Complete** → tick **"Install MongoDB as a Service"**
   and **"Run service as Network Service user"**.
3. Tick **"Install MongoDB Compass"** if you want a GUI (optional —
   mongo-express isn't part of this path).
4. After install, MongoDB listens on `localhost:27017` automatically. From
   PowerShell or Command Prompt:

   ```powershell
   mongosh
   ```

   You should land at a `>` prompt with no authentication required.

#### macOS (Homebrew)

```bash
brew tap mongodb/brew
brew install mongodb-community@7.0
brew services start mongodb-community@7.0

# Verify
mongosh
```

#### Linux (Ubuntu / Debian)

```bash
# Import the public key
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

# Add the repo (Ubuntu 22.04 example — adjust codename for other versions)
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] \
  https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

sudo apt update
sudo apt install -y mongodb-org

sudo systemctl enable --now mongod
mongosh
```

#### Default Authentication (or Lack of It)

The most common confusion right after a native install: **what's the default
username and password?**

**Short answer**: there isn't one. Every native install path (Windows MSI,
Homebrew, Linux apt repo) ships with **authentication disabled by default**.
Anyone who can reach `localhost:27017` can read and write every database with
no credentials.

```bash
mongosh
# No -u / -p flags. You just land at a > prompt.
```

This is **different from this lab kit's Docker setup**, which sets credentials
explicitly in `docker-compose.yml`:

| Setup | Default credentials |
|-------|---------------------|
| Lab kit Docker stack (`docker-compose.yml`) | `admin` / `ChangeMe123!` — set via `MONGO_INITDB_ROOT_*` env vars |
| MSI installer | None — auth disabled |
| Homebrew (`mongodb-community@7.0`) | None — auth disabled |
| Linux apt repo (`mongodb-org`) | None — auth disabled |

If you try to use `admin` / `ChangeMe123!` against a native install, you'll
get `MongoServerError: Authentication failed` — those credentials simply
don't exist on this engine.

##### Optional: Enable Auth to Mirror the Docker Setup

If you want a native install that behaves like the Docker lab kit (so the
exercise commands match exactly), enable auth and create an admin user via
the **localhost exception**.

Edit `mongod.cfg` / `mongod.conf`:

| OS | Path |
|----|------|
| Windows | `C:\Program Files\MongoDB\Server\7.0\bin\mongod.cfg` |
| macOS (Homebrew, Apple Silicon) | `/opt/homebrew/etc/mongod.conf` |
| macOS (Homebrew, Intel) | `/usr/local/etc/mongod.conf` |
| Linux | `/etc/mongod.conf` |

Add or uncomment:

```yaml
security:
  authorization: enabled
```

Restart the service:

| OS | Command |
|----|---------|
| Windows (Admin PowerShell) | `Restart-Service MongoDB` |
| macOS | `brew services restart mongodb-community@7.0` |
| Linux | `sudo systemctl restart mongod` |

Then use the **localhost exception** to create the first admin user — this
exception lets one unauthenticated connection through, but only from the
same machine and only until the first user is created:

```javascript
// From the same machine running mongod
mongosh

use admin
db.createUser({
  user: "admin",
  pwd: "ChangeMe123!",
  roles: [{ role: "root", db: "admin" }]
})
exit
```

After that, every connection requires authentication:

```bash
mongosh -u admin -p 'ChangeMe123!' --authenticationDatabase admin
```

Reference:
<https://www.mongodb.com/docs/manual/core/localhost-exception/>

##### Training Note — This Is Module 3's Opening Demo

The unauthenticated default is **literally the scenario Module 3 warns about**
("ramai orang deploy MongoDB tanpa auth, jadi viral kat Shodan"). On a native
install you can use this as a live demo before Exercise 3:

```javascript
// Run from any terminal on the host — no password
mongosh
> show dbs                     // reads everything
> use library
> db.books.deleteMany({})      // destroys data, no audit trail
```

Then enable auth (steps above) before running Exercise 3 itself. The "before
and after" landing has more impact than reading Slide 30's bullet points.

#### Loading Sample Data (Native Install)

The `mongoimport` commands in the lab guides assume the sample files live
inside the Docker container at `/sample-data`. Native install changes the
path — point to wherever you cloned the lab kit instead:

```bash
cd /path/to/mongodb-lab

mongoimport --db library --collection books --jsonArray \
  --file sample-data/books.json

mongoimport --db library --collection products --jsonArray \
  --file sample-data/products.json

mongoimport --db library --collection users --jsonArray \
  --file sample-data/users.json
```

No `--username` / `--password` / `--authenticationDatabase` flags — the
default native install has authentication **off**, which is fine for the
exercises but is exactly what Module 3 tells you never to ship to production.

#### What's Different vs the Docker Path

| Aspect | Docker | Native Install |
|--------|--------|----------------|
| Engine | `mongo:7` container | OS-native MongoDB service |
| Default auth | Enabled (`admin` / `ChangeMe123!`) | **Disabled** — anyone on `localhost` can read/write |
| mongo-express UI | Yes, on `:8081` | Not included — install separately or use Compass |
| Connecting | `docker exec -it mongo-lab mongosh -u admin -p` | `mongosh` |
| Sample data path | `/sample-data/...` inside container | `sample-data/...` on host |
| Module 3 backup demo | `docker exec mongo-lab mongodump ...` | `mongodump --db library --out backups` from host |
| Cleanup after course | `docker compose down -v` | Uninstall MongoDB service via OS package manager |

#### Module 3 Adjustments for Native Installs

Module 3 Exercise 3 (Security Hardening) **still works**, but the workflow
is slightly different:

- The default native install has auth disabled. Step 1 of the exercise
  becomes "enable auth and create the first admin user" — see
  [Default Authentication (or Lack of It)](#default-authentication-or-lack-of-it)
  above for the exact `mongod.conf` change and the localhost-exception
  bootstrap.
- `mongodump` / `mongorestore` commands drop the `docker exec mongo-lab`
  prefix and run on the host directly.
- The Exercise 3 "bonus" reflection about `docker-compose.yml` doesn't
  apply — substitute a discussion of `mongod.conf` hardening instead.

#### "MongoNetworkError: ECONNREFUSED 127.0.0.1:27017"

This is the most common stumble on a native install. It means `mongosh`
runs fine, but **nothing is listening on port 27017** — the database engine
itself isn't running. Three root causes, in order of frequency:

##### Cause 1 — Only `mongosh` was installed, not MongoDB Server

The `mongosh` shell is shipped as a separate download from the database
engine. Installing only the shell gives you a working client with nothing
to connect to.

**Diagnose (Windows, PowerShell):**

```powershell
Get-Service -Name MongoDB
# If: "Cannot find any service with service name 'MongoDB'"
# -> only the shell is installed
```

**Diagnose (macOS):**

```bash
brew services list | grep mongodb
# If empty, the server formula isn't installed
```

**Diagnose (Linux):**

```bash
systemctl status mongod
# If: "Unit mongod.service could not be found"
```

**Fix**: install **MongoDB Community Server** from
<https://www.mongodb.com/try/download/community>. On the Windows MSI, tick
**"Install MongoDB as a Service"** during setup — without that tick, the
engine is installed but never started.

##### Cause 2 — Server installed but the service is stopped

The engine is installed, but the service hasn't started — either it crashed
or "Run service on completion" was unticked during install.

**Diagnose (Windows, PowerShell):**

```powershell
Get-Service MongoDB
# Status: Stopped
```

**Start it (PowerShell as Administrator):**

```powershell
Start-Service MongoDB
Get-Service MongoDB   # Status: Running
```

Or from elevated `cmd`: `net start MongoDB`.

**Start it (macOS):**

```bash
brew services start mongodb-community@7.0
brew services list   # mongodb-community@7.0 should be 'started'
```

**Start it (Linux):**

```bash
sudo systemctl enable --now mongod
systemctl status mongod   # active (running)
```

##### Cause 3 — Service starts then immediately crashes

The service registers, you try to start it, but it stops within seconds. The
log file tells you why.

**Read the log (Windows):**

```powershell
Get-Content "C:\Program Files\MongoDB\Server\7.0\log\mongod.log" -Tail 30
```

**Read the log (macOS):**

```bash
tail -n 30 /opt/homebrew/var/log/mongodb/mongo.log
# or /usr/local/var/log/mongodb/mongo.log on Intel Macs
```

**Read the log (Linux):**

```bash
sudo journalctl -u mongod -n 30 --no-pager
# or
sudo tail -n 30 /var/log/mongodb/mongod.log
```

Look for `ERROR` or `FATAL` lines. The common culprits:

| Log message | Cause | Fix |
|-------------|-------|-----|
| `NonExistentPath: Data directory ... not found` | `dbPath` directory missing | Create the folder shown in the error (Windows default: `C:\data\db`) or fix `dbPath` in `mongod.cfg` |
| `Permission denied` on `dbPath` | Service user can't write to the data folder | Re-run the MSI choosing "Network Service" user, or `chown` the folder to `mongod` on Linux |
| `Address already in use` / port `27017` in use | Another process owns the port | Windows: `netstat -ano \| findstr :27017` then stop that PID. Mac/Linux: `lsof -i :27017` |
| `Insufficient free space` | Disk is full or `wiredTiger` can't allocate | Free disk space, then restart the service |

##### Verify the Fix

Once the service is running:

```bash
mongosh
```

You should land at a `>` prompt with no error. Confirm the engine answers:

```javascript
db.runCommand({ ping: 1 })
// { ok: 1 }
```

That `ok: 1` is the green light — you can now load the sample data and
move on to the exercises.

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
