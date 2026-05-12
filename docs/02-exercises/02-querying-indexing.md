# Exercise 2 — Querying and Indexing

**Lab Guide Part 3.** Referenced from Slide 26. **Duration: 25 minutes.**

Shape query results with sort, projection, and limit. Then create indexes and
use `explain()` to confirm MongoDB is using them.

## Prerequisites

- Lab stack running
- `products.json` and `users.json` imported
- `mongosh` open as `admin`, with `use library`

Import the two sample files if you haven't already:

```bash
docker exec mongo-lab mongoimport \
  --username admin --password 'ChangeMe123!' --authenticationDatabase admin \
  --db library --collection products --jsonArray \
  --file /sample-data/products.json

docker exec mongo-lab mongoimport \
  --username admin --password 'ChangeMe123!' --authenticationDatabase admin \
  --db library --collection users --jsonArray \
  --file /sample-data/users.json
```

## Tasks

### Task 1 — Three most expensive products

```javascript
db.products.find().sort({ price: -1 }).limit(3)
```

Expected (in order): *Laptop Pro 14* (5499), *Smartphone X* (2899), *Standing Desk* (1899).

### Task 2 — Engineering users who joined before 2020

```javascript
db.users.find({
  department: "Engineering",
  joined: { $lt: "2020-01-01" }
})
```

Expected: 2 users — *raj_kumar* (2018-11-03), *david_tan* (2017-09-01).

### Task 3 — Run `explain()` and look for COLLSCAN

```javascript
db.users.find({
  department: "Engineering",
  joined: { $lt: "2020-01-01" }
}).explain("executionStats")
```

In the output, look for `winningPlan.stage`. It will be `COLLSCAN` —
MongoDB is scanning every document because no index supports the query.

### Task 4 — Create a compound index and re-run explain

```javascript
db.users.createIndex({ department: 1, joined: 1 })

db.users.find({
  department: "Engineering",
  joined: { $lt: "2020-01-01" }
}).explain("executionStats")
```

`winningPlan.stage` is now `IXSCAN`. That's the **aha moment** — same query,
massively faster lookup pattern at scale.

### Task 5 — Add a unique index on email

```javascript
db.users.createIndex({ email: 1 }, { unique: true })
db.users.getIndexes()
```

Expected: three indexes — `_id_` (default), `department_1_joined_1`, `email_1`.

### Task 6 — Try to break the unique constraint

```javascript
db.users.insertOne({
  username: "ali_duplicate",
  email: "ali@example.com",      // already exists
  department: "Engineering"
})
```

Expected: `E11000 duplicate key error collection: library.users index: email_1`.
This is the database equivalent of a SQL `UNIQUE` constraint violation.

## Reading `explain()` Output

| Field | Meaning |
|-------|---------|
| `winningPlan.stage = COLLSCAN` | Full collection scan — no index used |
| `winningPlan.stage = IXSCAN` | Index scan — MongoDB jumped to matches |
| `winningPlan.inputStage.indexName` | Which index is being used |
| `executionStats.totalDocsExamined` | How many docs MongoDB had to read |
| `executionStats.totalKeysExamined` | How many index entries it walked |
| `executionStats.executionTimeMillis` | Wall-clock time for the query |

Ideal: `totalDocsExamined` and `totalKeysExamined` close to the number of
results returned.

## Compound Index Rule

`{ department: 1, joined: 1 }` supports queries on:

- `{ department }`
- `{ department, joined }`

It does **not** support a query on `{ joined }` alone — the prefix matters.

## Reset

```javascript
db.users.dropIndex("department_1_joined_1")
db.users.dropIndex("email_1")
```

## Trainer Notes

- 25 minutes — slightly more than Exercise 1 because reading `explain()` takes time.
- The aha moment is Task 4: identical query, COLLSCAN before, IXSCAN after.
- Task 6 demonstrates a unique-constraint error in real time. Anchor the SQL analogy.
- Early finishers: ask them to drop the compound index and re-run Task 4 — confirm
  the COLLSCAN returns.

## Next Steps

- [Exercise 3 — Security Hardening](03-security-hardening.md)
