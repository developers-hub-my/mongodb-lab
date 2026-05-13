# mongosh Cheatsheet

The commands used during the session, grouped by purpose. Keep this nearby
when you're working through the exercises or building your own queries.

## Connecting

```bash
# From your laptop into the lab container
docker exec -it mongo-lab mongosh -u admin -p 'ChangeMe123!'

# Connect to a remote MongoDB
mongosh "mongodb://user:pass@host:27017/dbname?authSource=admin"
```

## Database and Collection Basics

| Command | Purpose |
|---------|---------|
| `show dbs` | List databases |
| `use library` | Switch to (or lazily create) the `library` database |
| `show collections` | List collections in the current database |
| `db.books.countDocuments()` | Count documents in a collection |
| `db.books.drop()` | Delete a collection |
| `db.dropDatabase()` | Delete the current database (requires `dbOwner` role) |

## Create

```javascript
db.books.insertOne({ title: "Clean Code", year: 2008 })

db.books.insertMany([
  { title: "Sapiens", year: 2011 },
  { title: "1984",    year: 1949 }
])
```

## Read

### Basics

```javascript
// Find everything
db.books.find()

// Find one document (returns the doc, not a cursor)
db.books.findOne({ title: "Clean Code" })

// Find with a filter
db.books.find({ author: "George Orwell" })
```

`find()` returns a **cursor** that you iterate to get results. `findOne()`
returns a **single document** (or `null` if no match) — handy when you know
there's only one or you want the first.

### Comparison Operators

```javascript
db.books.find({ year: { $gt: 2000 } })                           // published after 2000
db.books.find({ price: { $lte: 50 } })                           // RM 50 or less
db.books.find({ genre: { $in: ["Fiction", "History"] } })        // either genre
```

### Combining Conditions (Implicit AND)

Multiple fields in the same filter object are joined with **AND**
automatically — no `$and` needed in the common case.

```javascript
// Programming books published in 2010 or later
db.books.find({ genre: "Programming", year: { $gte: 2010 } })
```

Use `$or` explicitly when you want logical OR:

```javascript
db.books.find({ $or: [ { year: 1949 }, { year: 1965 } ] })
```

### Projection — Choose Which Fields to Return

```javascript
// Only the title and author; suppress _id
db.books.find(
  { genre: "Programming" },
  { title: 1, author: 1, _id: 0 }
)
```

The second argument controls fields: `1` includes, `0` excludes. The `_id`
field is included by default — set `_id: 0` to drop it.

### Sort, Skip, Limit

Chain these onto a cursor to shape the result set. Order of method calls
doesn't matter to MongoDB — but `sort` must come before `skip`/`limit`
mentally, because you almost always want to paginate **sorted** results.

```javascript
db.books.find({ genre: "Fiction" })
  .sort({ year: -1 })          // newest first (-1 desc, 1 asc)
  .skip(5)                     // skip the first 5
  .limit(10)                   // return the next 10
```

This is the standard "page 2 of 10 per page, newest first" pattern.

### Count

```javascript
db.books.countDocuments({ genre: "Programming" })
```

Use `countDocuments(filter)` rather than the deprecated `count()`. For an
unfiltered total, pass `{}` or omit the argument: `db.books.countDocuments()`.

## Query Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `$eq` | Equal | `{ year: { $eq: 2008 } }` |
| `$ne` | Not equal | `{ available: { $ne: false } }` |
| `$gt` / `$gte` | Greater than / or equal | `{ year: { $gte: 2010 } }` |
| `$lt` / `$lte` | Less than / or equal | `{ price: { $lt: 100 } }` |
| `$in` | In list | `{ genre: { $in: ["Programming", "Fiction"] } }` |
| `$nin` | Not in list | `{ year: { $nin: [2008, 2011] } }` |
| `$exists` | Field is set | `{ tags: { $exists: true } }` |
| `$and` / `$or` | Logical combine | `{ $or: [ { year: 1949 }, { year: 1965 } ] }` |
| `$regex` | Pattern match | `{ title: { $regex: "^The" } }` |

## MongoDB ↔ SQL

If you already think in SQL, use this section to translate. The mental
model maps cleanly for most read/write patterns — the genuine divergences
are JOINs and full-document replace, called out at the end.

### Worked Example — Pagination

A typical "newest Fiction books, page 2 of 10 per page" query:

```javascript
db.books.find({ genre: "Fiction" }).sort({ year: -1 }).skip(5).limit(10)
```

```sql
SELECT *
FROM books
WHERE genre = 'Fiction'
ORDER BY year DESC
LIMIT 10 OFFSET 5;
```

Piece by piece:

| MongoDB | SQL |
|---------|-----|
| `db.books` | `FROM books` |
| `.find({ genre: "Fiction" })` | `WHERE genre = 'Fiction'` (with implicit `SELECT *`) |
| `.sort({ year: -1 })` | `ORDER BY year DESC` |
| `.skip(5)` | `OFFSET 5` |
| `.limit(10)` | `LIMIT 10` |

### Translation Table — Common Patterns

| SQL | MongoDB |
|-----|---------|
| `SELECT * FROM books` | `db.books.find()` |
| `SELECT title, year FROM books` | `db.books.find({}, { title: 1, year: 1, _id: 0 })` |
| `WHERE year = 2008` | `{ year: 2008 }` |
| `WHERE year > 2010` | `{ year: { $gt: 2010 } }` |
| `WHERE genre IN ('Fiction', 'History')` | `{ genre: { $in: ["Fiction", "History"] } }` |
| `WHERE title LIKE 'The%'` | `{ title: { $regex: "^The" } }` |
| `ORDER BY year DESC` | `.sort({ year: -1 })` |
| `LIMIT 10` | `.limit(10)` |
| `LIMIT 10 OFFSET 5` | `.skip(5).limit(10)` |
| `SELECT COUNT(*) FROM books` | `db.books.countDocuments()` |
| `SELECT DISTINCT genre FROM books` | `db.books.distinct("genre")` |
| `GROUP BY genre` | `db.books.aggregate([{ $group: { _id: "$genre", n: { $sum: 1 } } }])` |
| `INNER JOIN authors ON ...` | Embedded subdocument, **or** `$lookup` in aggregation |
| `INSERT INTO books (...) VALUES (...)` | `db.books.insertOne({ ... })` |
| `UPDATE books SET price = 90 WHERE title = 'X'` | `db.books.updateOne({ title: "X" }, { $set: { price: 90 } })` |
| `DELETE FROM books WHERE title = 'X'` | `db.books.deleteOne({ title: "X" })` |
| `CREATE INDEX idx_author ON books(author)` | `db.books.createIndex({ author: 1 })` |
| `CREATE UNIQUE INDEX idx_email ON users(email)` | `db.users.createIndex({ email: 1 }, { unique: true })` |

### Where the Model Genuinely Diverges

**JOINs aren't free in MongoDB.**

> There's no foreign-key join engine. You either **embed** the related data
> in the same document (preferred for 1-to-few relationships read together)
> or use **`$lookup`** in an aggregation pipeline (for many-to-many or
> rarely-joined data). Heavy `$lookup` usage usually means the schema should
> be reconsidered — that workload may belong in a relational database.

**Update without a `$` operator replaces the whole document.**

> `db.books.updateOne({...}, { newDoc })` **without** `$set`, `$inc`,
> `$push`, etc. replaces the entire document. There's no real SQL analogue
> — the closest would be a destructive full-row rewrite that drops every
> column you didn't list. Always use update operators unless you genuinely
> intend a wholesale replace.

## Update

```javascript
// Set or change a field
db.books.updateOne({ title: "Clean Code" }, { $set: { price: 90.00 } })

// Increment / multiply numeric fields
db.books.updateMany({ genre: "Self-Help" }, { $mul: { price: 0.9 } })
db.products.updateOne({ sku: "PH-002" }, { $inc: { stock: -1 } })

// Append to an array
db.books.updateMany({ year: { $lt: 1970 } }, { $push: { tags: "classic" } })

// Remove a field
db.books.updateOne({ title: "Clean Code" }, { $unset: { price: "" } })
```

> **Warning**: `updateOne({...}, { newDoc })` (without a `$` operator) **replaces**
> the entire document. Always use `$set`, `$inc`, `$push`, `$unset`, etc.

## Delete

```javascript
db.books.deleteOne({ title: "Dune" })
db.books.deleteMany({ available: false })
```

> **Tip**: Always run the same filter through `find()` first to confirm you're
> about to delete the right documents.

## Indexes

```javascript
// Single-field index
db.books.createIndex({ author: 1 })   // 1 = ascending, -1 = descending

// Compound index — order matters
db.users.createIndex({ department: 1, joined: 1 })

// Unique index
db.users.createIndex({ email: 1 }, { unique: true })

// Inspect
db.books.getIndexes()

// Drop
db.books.dropIndex("author_1")

// Confirm a query uses an index
db.books.find({ author: "Frank Herbert" }).explain("executionStats")
```

## Aggregation (Preview)

```javascript
db.books.aggregate([
  { $match: { available: true } },
  { $group: { _id: "$genre", count: { $sum: 1 }, avgPrice: { $avg: "$price" } } },
  { $sort: { count: -1 } }
])
```

## User Management

```javascript
use admin

db.createUser({
  user: "library_app",
  pwd: passwordPrompt(),
  roles: [{ role: "readWrite", db: "library" }]
})

db.getUsers()
db.dropUser("library_app")
db.changeUserPassword("library_app", passwordPrompt())
```

| Built-in Role | Scope |
|---------------|-------|
| `read` | Read a database |
| `readWrite` | Read and write a database |
| `dbAdmin` | DB-level admin (indexes, stats) — not user management |
| `userAdmin` | Manage users on a database |
| `dbOwner` | All admin operations on a database |
| `readAnyDatabase` | Read every database (cluster-wide) |
| `root` | Full cluster admin — do not use for apps |

## Backup and Restore (Run on the Host)

```bash
# Backup all databases
docker exec mongo-lab mongodump \
  --username admin --password 'ChangeMe123!' --authenticationDatabase admin \
  --out /backups

# Backup a single database
docker exec mongo-lab mongodump \
  --username admin --password 'ChangeMe123!' --authenticationDatabase admin \
  --db library --out /backups

# Restore a database, dropping existing collections first
docker exec mongo-lab mongorestore \
  --username admin --password 'ChangeMe123!' --authenticationDatabase admin \
  --db library --drop /backups/library
```

## Useful Shell Tricks

```javascript
// Pretty-print output
db.books.find().pretty()

// Save a query as a variable
const recent = db.books.find({ year: { $gt: 2010 } }).toArray()
recent.length

// Multi-line input — open editor
config.set("editor", "vim")
edit
```

## Next Steps

- [Security Checklist](02-security-checklist.md)
- [Exercises](../02-exercises/README.md)
