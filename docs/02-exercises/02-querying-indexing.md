# Exercise 2 — Querying and Indexing

**Lab Guide Part 3 (Module 2).** **Duration: 25 minutes.**

Shape query results with sort, projection, and limit. Then create indexes
and use `explain()` to confirm MongoDB is using them.

> **How to use this guide**: every task starts with the **goal** and a
> **hint**. Try writing the command yourself first. Click **Show solution**
> only after you've attempted it — that's where the learning happens.

## Prerequisites

- Lab stack running (or native MongoDB install — see
  [Plan D](../01-getting-started/04-troubleshooting.md#plan-d--install-mongodb-natively-no-docker))
- `mongosh` open and connected, with `use library`

## Tasks

### Task 1 — Load the Sample Data

**Goal**: import `products.json` and `users.json` into the `library` database.

**Hint**: use `mongoimport` with `--jsonArray` (because the sample files
are JSON arrays, not NDJSON).

<details>
<summary>**Show solution**</summary>

**Docker lab stack** — run from your host shell, not inside `mongosh`:

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

**Native install** — drop the `docker exec` prefix and point at the host
path:

```bash
cd /path/to/mongodb-lab

mongoimport --db library --collection products --jsonArray \
  --file sample-data/products.json

mongoimport --db library --collection users --jsonArray \
  --file sample-data/users.json
```

**Expected output**: `6 document(s) imported successfully` for `products`
and `8 document(s) imported successfully` for `users`.

**Verify in `mongosh`**:

```javascript
use library
db.products.countDocuments()    // 6
db.users.countDocuments()       // 8
```

</details>

### Task 2 — Find the 3 Most Expensive Products

**Goal**: list the top 3 products by price, most expensive first.

**Hint**: chain `.find()` with `.sort()` and `.limit()`. Sort direction
`-1` is descending.

<details>
<summary>**Show solution**</summary>

```javascript
db.products.find()
  .sort({ price: -1 })   // most expensive first
  .limit(3)
```

**Expected output** (in order):

| Rank | Product | Price (RM) |
|------|---------|------------|
| 1 | Laptop Pro 14 | 5499.00 |
| 2 | Smartphone X | 2899.00 |
| 3 | Standing Desk | 1899.00 |

**Bonus**: add projection to return only the name and price:

```javascript
db.products.find({}, { name: 1, price: 1, _id: 0 })
  .sort({ price: -1 })
  .limit(3)
```

</details>

### Task 3 — Find Engineering Users Who Joined Before 2020

**Goal**: filter the `users` collection by two fields at once.

**Hint**: multiple fields in the same filter object are joined with implicit
AND. The `joined` field is stored as an ISO date string (`"YYYY-MM-DD"`),
so string comparison works.

<details>
<summary>**Show solution**</summary>

```javascript
db.users.find({
  department: "Engineering",
  joined: { $lt: "2020-01-01" }
})
```

**Expected output**: 2 users —

| Username | Joined |
|----------|--------|
| `raj_kumar` | 2018-11-03 |
| `david_tan` | 2017-09-01 |

</details>

### Task 4 — Run `explain()` on the Query — Note `COLLSCAN`

**Goal**: confirm that without an index, MongoDB has to scan every document.

**Hint**: add `.explain("executionStats")` to the end of the cursor.
Look for `winningPlan.stage` in the output.

<details>
<summary>**Show solution**</summary>

```javascript
db.users.find({
  department: "Engineering",
  joined: { $lt: "2020-01-01" }
}).explain("executionStats")
```

In the output, find these lines:

```text
winningPlan: {
  stage: 'COLLSCAN',          // ← full collection scan
  ...
},
executionStats: {
  nReturned: 2,
  totalDocsExamined: 8,        // read all 8 docs to find 2
  totalKeysExamined: 0,
  executionTimeMillis: 0,
  ...
}
```

**What `COLLSCAN` means**: there's no index that helps this query, so
MongoDB reads every document and checks the filter in memory. Fine for
8 documents; a disaster at 8 million.

</details>

### Task 5 — Create a Compound Index and Re-Run `explain()`

**Goal**: add a single index that supports the Task 3 query, and confirm
MongoDB switches from `COLLSCAN` to `IXSCAN`.

**Hint**: create an index on the two fields you're filtering on, in the
same order as the filter. Then re-run the same `explain("executionStats")`.

<details>
<summary>**Show solution**</summary>

```javascript
// Create the compound index
db.users.createIndex({ department: 1, joined: 1 })

// Confirm it exists
db.users.getIndexes()

// Re-run the same explain()
db.users.find({
  department: "Engineering",
  joined: { $lt: "2020-01-01" }
}).explain("executionStats")
```

The `explain()` output now shows:

```text
winningPlan: {
  stage: 'IXSCAN',                              // ← index scan
  indexName: 'department_1_joined_1',
  ...
},
executionStats: {
  nReturned: 2,
  totalDocsExamined: 2,        // read exactly the 2 matching docs
  totalKeysExamined: 2,
  ...
}
```

**This is the aha moment**: same query, dramatically different work. At 8
documents you don't feel it. At 8 million you do — `COLLSCAN` would be
seconds while `IXSCAN` stays in milliseconds.

</details>

### Task 6 — Try a Duplicate Insert After Creating a Unique Index

**Goal**: create a unique index on `email`, then prove the constraint by
attempting to insert a user whose email already exists.

**Hint**: pass `{ unique: true }` as the second argument to `createIndex`.
Then call `insertOne` with an email from the sample data.

<details>
<summary>**Show solution**</summary>

```javascript
// Add the unique constraint
db.users.createIndex({ email: 1 }, { unique: true })

// Verify — should now show three indexes:
//   _id_, department_1_joined_1, email_1
db.users.getIndexes()

// Attempt to insert a duplicate email
db.users.insertOne({
  username: "ali_duplicate",
  email: "ali@example.com",      // ali_ahmad already has this email
  department: "Engineering"
})
```

**Expected output**:

```text
MongoServerError: E11000 duplicate key error collection: library.users
  index: email_1 dup key: { email: "ali@example.com" }
```

That `E11000` is MongoDB's duplicate-key error code — the database
equivalent of a SQL `UNIQUE` constraint violation. The new document is
**not** inserted; the existing one is untouched.

**Bonus**: confirm by counting documents — still 8, not 9:

```javascript
db.users.countDocuments()    // 8
```

</details>

## Reading `explain()` Output

| Field | Meaning |
|-------|---------|
| `winningPlan.stage = COLLSCAN` | Full collection scan — no index used |
| `winningPlan.stage = IXSCAN` | Index scan — MongoDB jumped to matches |
| `winningPlan.inputStage.indexName` | Which index is being used |
| `executionStats.totalDocsExamined` | How many docs MongoDB had to read |
| `executionStats.totalKeysExamined` | How many index entries it walked |
| `executionStats.executionTimeMillis` | Wall-clock time for the query |

Ideal: `totalDocsExamined` and `totalKeysExamined` close to `nReturned`.

## Compound Index Rule (Prefix Matters)

`{ department: 1, joined: 1 }` supports queries on:

- `{ department }` alone
- `{ department, joined }` together

It does **not** support a query on `{ joined }` alone — MongoDB can only
use the index from the leftmost field onwards. If you also need to query
by `joined` alone, create a separate index for it.

## Reset

To restart this exercise from scratch:

```javascript
// Drop the indexes you created
db.users.dropIndex("department_1_joined_1")
db.users.dropIndex("email_1")

// Drop and reimport users if you ran the duplicate-insert task
db.users.drop()
// then re-run the mongoimport from Task 1
```

The `_id_` index can't be dropped — it always exists.

## Trainer Notes

- 25 minutes — slightly more than Exercise 1 because reading `explain()`
  takes time.
- The aha moment is **Task 5**: identical query, `COLLSCAN` before,
  `IXSCAN` after. Demo on the projector with `.explain()` output side by
  side if possible.
- **Task 6** demonstrates a unique-constraint error in real time. Anchor
  the SQL analogy explicitly ("this is `UNIQUE (email)` in SQL").
- **Solutions are collapsed by default** — encourage participants to
  attempt each task before clicking *Show solution*. Walk the room and
  prompt anyone who's expanding all of them without trying first.
- **Early finishers**: ask them to drop the compound index and re-run
  Task 4 — confirm the `COLLSCAN` returns. Or: ask "why doesn't
  `{ joined: 1 }` alone work with this compound index?" and discuss
  prefix rule.

## Next Steps

- [Exercise 3 — Security Hardening](03-security-hardening.md)
- [mongosh Cheatsheet](../03-reference/01-mongosh-cheatsheet.md) — Indexes section
