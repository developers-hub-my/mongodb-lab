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

```javascript
// Find all
db.books.find()

// Filter
db.books.find({ year: { $gt: 2010 } })

// Project — return only certain fields
db.books.find({}, { title: 1, year: 1, _id: 0 })

// Sort, skip, limit
db.books.find().sort({ year: -1 }).skip(0).limit(5)

// Single document
db.books.findOne({ title: "Clean Code" })

// Count
db.books.countDocuments({ genre: "Programming" })
```

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
- [Slide Deck](../slide.md)
