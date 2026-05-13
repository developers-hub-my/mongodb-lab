# Exercise 3 — Security Hardening

**Lab Guide Part 4 (Module 3).** **Duration: 30 minutes.**

Create least-privilege users, verify access controls, take a `mongodump`
backup, and restore from it. This is the most cognitively demanding
exercise of the day — pair up if possible.

> **How to use this guide**: every task starts with a **goal** and a
> **hint**. Try the command yourself first. Click **Show solution** only
> after you've attempted it — the friction is the point.

## Prerequisites

- Lab stack running (or native MongoDB install with auth enabled — see
  [Default Authentication](../01-getting-started/04-troubleshooting.md#default-authentication-or-lack-of-it))
- `library` database has `books`, `products`, `users` collections from
  Exercises 1 and 2
- `mongosh` open as `admin`

## Tasks

### Task 1 — Create a Read/Write App User

**Goal**: create a user named `library_app` that can read and write the
`library` database — but nothing else.

**Hint**: in the `admin` database, call `db.createUser(...)` with a single
role: `readWrite` scoped to `library`. Use `passwordPrompt()` so you don't
type the password into shell history.

<details>
<summary>**Show solution**</summary>

```javascript
use admin

db.createUser({
  user: "library_app",
  pwd: passwordPrompt(),     // mongosh prompts you to type it interactively
  roles: [ { role: "readWrite", db: "library" } ]
})
```

**Verify**:

```javascript
db.getUsers()
// Should show library_app with the readWrite@library role
```

**Why `passwordPrompt()`**: typing the password as a literal string puts
it in your shell history and the `mongosh` command log. `passwordPrompt()`
reads it from stdin without echoing.

</details>

### Task 2 — Create a Read-Only Analytics User

**Goal**: create a user named `analytics_ro` that can read the `library`
database but cannot write to it.

**Hint**: same command as Task 1, but use the `read` role instead of
`readWrite`.

<details>
<summary>**Show solution**</summary>

```javascript
use admin

db.createUser({
  user: "analytics_ro",
  pwd: passwordPrompt(),
  roles: [ { role: "read", db: "library" } ]
})
```

**Verify**:

```javascript
db.getUsers()
// Two non-admin users now: library_app and analytics_ro
```

</details>

### Task 3 — Authenticate as `library_app` and Prove the Boundary

**Goal**: log in as `library_app`. Confirm you can read and write the
`library` database — but you **cannot** drop it.

**Hint**: exit your current `mongosh` (`.exit`), then reconnect with
`-u library_app --authenticationDatabase admin` flags. Try
`db.books.findOne()`, then `insertOne()`, then `dropDatabase()` and
observe which one is rejected.

<details>
<summary>**Show solution**</summary>

Exit the admin shell, reconnect as `library_app`:

```bash
# Docker lab stack
docker exec -it mongo-lab mongosh \
  -u library_app -p \
  --authenticationDatabase admin

# Native install
mongosh -u library_app -p --authenticationDatabase admin
```

Inside the new shell:

```javascript
use library

// Read — allowed
db.books.findOne()

// Write — allowed
db.books.insertOne({ title: "Test", year: 2025 })

// Drop the database — DENIED
db.dropDatabase()
```

**Expected error on `dropDatabase`**:

```text
MongoServerError: not authorized on library to execute command
  { dropDatabase: 1 }
```

That error is the point — least privilege working as intended. The
`readWrite` role lets the app modify documents but not destroy the
database.

</details>

### Task 4 — Authenticate as `analytics_ro` and Prove the Boundary

**Goal**: log in as `analytics_ro`. Confirm you can read, but you **cannot**
write.

**Hint**: same authentication pattern as Task 3, but with the `analytics_ro`
user. Try `findOne()` (works) then `insertOne()` (rejected).

<details>
<summary>**Show solution**</summary>

```bash
# Docker lab stack
docker exec -it mongo-lab mongosh \
  -u analytics_ro -p \
  --authenticationDatabase admin

# Native install
mongosh -u analytics_ro -p --authenticationDatabase admin
```

Inside the new shell:

```javascript
use library

// Read — allowed
db.books.findOne()

// Write — DENIED
db.books.insertOne({ title: "Test", year: 2025 })
```

**Expected error**:

```text
MongoServerError: not authorized on library to execute command
  { insert: "books", ... }
```

Read-only means read-only. The same boundary will protect production data
when an analyst's BI tool runs an accidental `UPDATE` against the wrong
cluster.

</details>

### Task 5 — Take a `mongodump` Backup

**Goal**: dump the `library` database to a `/backups` folder inside the
container (or a `backups/` folder on the host for native installs).

**Hint**: run `mongodump` from the host shell, not inside `mongosh`. It's
a separate command-line tool. The cleanest connection form is the
`--uri=` flag, which carries credentials and the host in one string.

<details>
<summary>**Show solution**</summary>

#### Option A — Dump a single database (what we'll restore from in Task 6)

```bash
# Docker lab stack
docker exec mongo-lab mkdir -p /backups

docker exec mongo-lab mongodump \
  --uri='mongodb://admin:ChangeMe123!@localhost:27017' \
  --db=library --out=/backups/library

# Native install
mkdir -p backups

mongodump \
  --uri='mongodb://admin:ChangeMe123!@localhost:27017' \
  --db=library --out=backups/library
```

#### Option B — Dump every database (date-stamped for daily snapshots)

```bash
# Docker lab stack
docker exec mongo-lab mongodump \
  --uri='mongodb://admin:ChangeMe123!@localhost:27017' \
  --out=/backups/$(date +%Y%m%d)

# Native install
mongodump \
  --uri='mongodb://admin:ChangeMe123!@localhost:27017' \
  --out=backups/$(date +%Y%m%d)
```

The `$(date +%Y%m%d)` expands on your **host** shell to today's date
(e.g. `20260513`), so each run lands in a fresh dated folder — the basic
pattern for "keep N days of backups, prune the rest."

#### Expected output

```text
done dumping library.books (10 documents)
done dumping library.products (6 documents)
done dumping library.users (8 documents)
```

#### Verify the dump exists

```bash
# Docker — for Option A
docker exec mongo-lab ls /backups/library/library

# Native — for Option A
ls backups/library/library
# books.bson  books.metadata.json  products.bson  ...
```

The dump is a folder **per database**, with `.bson` (binary documents)
and `.metadata.json` (indexes + options) files per collection. Note the
nested path — `--out=/backups/library` creates `/backups/library/library/`
because `mongodump` adds its own directory layer named after the database.

> **Tip — `--uri` vs split flags**: `--uri='mongodb://user:pass@host:port'`
> is the modern form; it also accepts `?authSource=admin&tls=true` query
> parameters. The older `--username admin -p` form is still valid and is
> safer in shared terminals (the `-p` flag without a value prompts
> interactively instead of putting the password in shell history).

</details>

### Task 6 — Drop the `books` Collection, Then Restore From Backup

**Goal**: simulate a data-loss incident by dropping `books`, then restore
it from the dump you just took.

**Hint**: drop from inside `mongosh`. Restore from the host shell using
`mongorestore`. The source path is the **inner** database folder created
by `mongodump` — `/backups/library/library`, not `/backups/library`.

<details>
<summary>**Show solution**</summary>

#### Step 1 — Simulate data loss

In `mongosh` as `admin`:

```javascript
use library

// Confirm the count, drop the collection, confirm it's gone
db.books.countDocuments()    // 10 (or 11 if Task 3 insert succeeded)
db.books.drop()
db.books.countDocuments()    // 0
```

#### Step 2 — Restore from the dump

From the host shell:

```bash
# Docker lab stack — restore the library DB from /backups/library/library
docker exec mongo-lab mongorestore \
  --uri='mongodb://admin:ChangeMe123!@localhost:27017' \
  --db=library /backups/library/library

# Native install
mongorestore \
  --uri='mongodb://admin:ChangeMe123!@localhost:27017' \
  --db=library backups/library/library
```

> **Why the doubled `library/library`**: `mongodump --out=/backups/library`
> created `/backups/library/`, then nested the database under it as
> `/backups/library/library/`. `mongorestore` expects to be pointed at
> the **inner** folder — the one that contains the `.bson` files.

#### Step 3 — Verify

Back in `mongosh`:

```javascript
db.books.countDocuments()    // 10 (back to the dumped count)
```

#### Variant — Restore and overwrite existing collections (`--drop`)

In step 1 we dropped `books` ourselves, so the restore just adds the
documents back. In real-life "restore over existing data" — for example,
after an accidental `updateMany` corrupted half a collection — you want
`mongorestore` to drop the target collections **before** restoring so
you don't end up with duplicates:

```bash
# Docker lab stack
docker exec mongo-lab mongorestore \
  --uri='mongodb://admin:ChangeMe123!@localhost:27017' \
  --drop --db=library /backups/library/library

# Native install
mongorestore \
  --uri='mongodb://admin:ChangeMe123!@localhost:27017' \
  --drop --db=library backups/library/library
```

Same source path, with `--drop` added. Try this variant against a populated
collection to confirm it cleans up before importing.

#### The Module 3 takeaway

A backup is only real if you've actually restored from it. You just did.
Schedule a restore test **once a month** at work — same way you schedule
fire drills. Otherwise you'll discover the broken backup on the day you
need it.

</details>

### Bonus — Review `docker-compose.yml`

**Goal**: open [`docker-compose.yml`](../../docker-compose.yml) and list
**three things you would change before running it in production.**

Try this **before** expanding the discussion seeds below.

<details>
<summary>**Show discussion seeds (try first!)**</summary>

There are at least six issues. Common spots people land on:

| # | Issue | Why it matters |
|---|-------|----------------|
| 1 | Hardcoded `ChangeMe123!` password in the file | Anyone with read access to the repo has prod credentials. Use a secret manager (Vault, AWS Secrets Manager, Docker Swarm secrets, Kubernetes Secrets). |
| 2 | mongo-express uses plain HTTP basic auth | Credentials travel in cleartext on the wire. Either remove mongo-express from production entirely or front it with TLS. |
| 3 | MongoDB exposed on `0.0.0.0:27017` | The DB is reachable from any interface, including any public IP attached to the host. Bind to `127.0.0.1` or a private VPC interface. |
| 4 | The app uses the `admin` root account | The compose file connects mongo-express as `admin`. Apps should use a `readWrite`-scoped account (the `library_app` user you created in Task 1). |
| 5 | No TLS between mongo-express and MongoDB | Even inside the docker network, eavesdroppers on the host can sniff. |
| 6 | No backup volume or scheduled `mongodump` job | Storage is on a local Docker volume only. Lose the host → lose the data. Mount a network volume and schedule dumps to off-host storage. |

The Module 3 [Security Checklist](../03-reference/02-security-checklist.md)
gives you a structured way to audit any deployment against these and other
items.

</details>

## Cleanup

After the exercise — drop the users so the next class starts fresh:

```javascript
use admin
db.dropUser("library_app")
db.dropUser("analytics_ro")
```

## Trainer Notes

- **30 minutes — firmest time budget.** Stop on time even if some pairs
  are mid-task. The plenary discussion of the Bonus task is the
  high-value moment.
- **Pair participants** where possible. Security work benefits from a
  second pair of eyes; the act of explaining what `readWrite` lets you
  do (and doesn't) makes it stick.
- **Bonus task is the discussion driver.** Aim for ~5 minutes of plenary
  discussion. Ask each pair for one issue; collect on the whiteboard
  before revealing the discussion seeds.
- **Debrief prompts**: "What surprised you?" and "What would you do
  differently at work?"
- **If solutions get expanded too fast**: gently remind participants
  that reading the answer is not the same as typing it. Walk the room.
- **Native install caveat**: if a participant is on Plan D and the
  default install has auth disabled, Task 1 has an extra prereq —
  enable `security.authorization` in `mongod.conf` and use the localhost
  exception to bootstrap. See
  [Default Authentication](../01-getting-started/04-troubleshooting.md#default-authentication-or-lack-of-it).

## Next Steps

- [Security Checklist](../03-reference/02-security-checklist.md) — take-home reference
- [mongosh Cheatsheet — User Management](../03-reference/01-mongosh-cheatsheet.md#user-management)
