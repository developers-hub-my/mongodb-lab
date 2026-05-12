# Exercise 3 — Security Hardening

**Lab Guide Part 4 (Module 3).** **Duration: 30 minutes.**

Create least-privilege users, verify access controls, take a `mongodump`
backup, and restore from it. This is the most cognitively demanding exercise
of the day — pair up if possible.

## Prerequisites

- Lab stack running
- `library` database has `books`, `products`, `users` collections
- `mongosh` open as `admin`

## Tasks

### Task 1 — Create a read/write app user

```javascript
use admin
db.createUser({
  user: "library_app",
  pwd: passwordPrompt(),     // type a strong password when asked
  roles: [{ role: "readWrite", db: "library" }]
})
```

Verify:

```javascript
db.getUsers()
```

### Task 2 — Create a read-only analytics user

```javascript
use admin
db.createUser({
  user: "analytics_ro",
  pwd: passwordPrompt(),
  roles: [{ role: "read", db: "library" }]
})
```

### Task 3 — Authenticate as `library_app` and prove the boundary

Exit the current shell, then reconnect as `library_app`:

```bash
docker exec -it mongo-lab mongosh \
  -u library_app -p \
  --authenticationDatabase admin
```

Verify you can read and write:

```javascript
use library
db.books.findOne()                            // OK
db.books.insertOne({ title: "Test", year: 2025 })  // OK
```

Now confirm you **cannot** drop the database:

```javascript
db.dropDatabase()
// MongoServerError: not authorized on library to execute command { dropDatabase: 1 }
```

That error is the point — least privilege working as intended.

### Task 4 — Authenticate as `analytics_ro` and prove the boundary

```bash
docker exec -it mongo-lab mongosh \
  -u analytics_ro -p \
  --authenticationDatabase admin
```

```javascript
use library
db.books.findOne()                            // OK
db.books.insertOne({ title: "Test", year: 2025 })
// MongoServerError: not authorized on library to execute command { insert: ... }
```

### Task 5 — Backup with `mongodump`

Reconnect as `admin` from the host shell, then:

```bash
docker exec mongo-lab mkdir -p /backups
docker exec mongo-lab mongodump \
  --username admin --password 'ChangeMe123!' --authenticationDatabase admin \
  --db library --out /backups
```

Expected output ends with `done dumping library.users (8 documents)` or similar
for each collection.

### Task 6 — Drop and restore

In `mongosh` as `admin`:

```javascript
use library
db.books.drop()
db.books.countDocuments()    // 0
```

Restore from the dump:

```bash
docker exec mongo-lab mongorestore \
  --username admin --password 'ChangeMe123!' --authenticationDatabase admin \
  --db library --drop /backups/library
```

Back in `mongosh`:

```javascript
db.books.countDocuments()    // 10 (or 11 if you ran the Task 3 insert)
```

### Bonus — Review `docker-compose.yml`

Open [`docker-compose.yml`](../../docker-compose.yml). List **three things you
would change before running this in production.**

Discussion seeds (don't read these before trying yourself):

- Hardcoded `ChangeMe123!` password — should come from a secret manager
- mongo-express uses HTTP basic auth over plain HTTP — needs TLS or removal
- MongoDB is exposed on `0.0.0.0:27017` — should bind to a private interface
- The `admin` root account is used by the app — should use `library_app` only
- No TLS between mongo-express and MongoDB
- No backup volume or scheduled `mongodump` job

## Cleanup

After the exercise:

```javascript
use admin
db.dropUser("library_app")
db.dropUser("analytics_ro")
```

## Trainer Notes

- 30 minutes — firmest time budget. Stop on time even if some pairs are mid-task.
- Pair participants where possible — security work benefits from a second pair of eyes.
- Bonus task is the discussion driver. Aim for 5 minutes of plenary discussion.
- Debrief prompts: "What surprised you?" and "What would you do differently at work?"

## Next Steps

- [Security Checklist](../03-reference/02-security-checklist.md) — take-home reference
