# Exercise 1 — CRUD Practice

**Lab Guide Part 2 (Module 2).** **Duration: 20 minutes.**

Practice the four core operations — create, read, update, delete — against the
`books` collection.

## Prerequisites

- Lab stack running (see [Lab Setup](../01-getting-started/02-lab-setup.md))
- `books.json` imported (see [Loading Sample Data](../01-getting-started/03-loading-sample-data.md))
- `mongosh` open and connected as `admin`

If `books` is not yet loaded, import it now:

```bash
docker exec mongo-lab mongoimport \
  --username admin --password 'ChangeMe123!' --authenticationDatabase admin \
  --db library --collection books --jsonArray \
  --file /sample-data/books.json
```

## Tasks

### Task 1 — Switch to the database

```javascript
use library
db.books.countDocuments()   // should be 10
```

### Task 2 — Find all books published after 2010, newest first

```javascript
db.books.find({ year: { $gt: 2010 } }).sort({ year: -1 })
```

Expected: 4 books — *Atomic Habits* (2018), *Designing Data-Intensive Applications* (2017),
*Sapiens* (2011), *The Lean Startup* (2011), *Thinking, Fast and Slow* (2011).

### Task 3 — Count Programming books over RM 80

```javascript
db.books.countDocuments({ genre: "Programming", price: { $gt: 80 } })
```

Expected: 2 (*The Pragmatic Programmer*, *Clean Code*).

### Task 4 — Tag every pre-1970 book as 'classic'

```javascript
db.books.updateMany(
  { year: { $lt: 1970 } },
  { $push: { tags: "classic" } }
)
```

Expected: `modifiedCount: 2` (*1984*, *Dune*). Verify:

```javascript
db.books.find({ year: { $lt: 1970 } }, { title: 1, tags: 1, _id: 0 })
```

### Task 5 — 10% price cut on Self-Help books

```javascript
db.books.updateMany(
  { genre: "Self-Help" },
  { $mul: { price: 0.9 } }
)
```

Expected: `modifiedCount: 2`. Verify:

```javascript
db.books.find({ genre: "Self-Help" }, { title: 1, price: 1, _id: 0 })
```

### Task 6 — Delete one book and verify

Pick any book and delete it. Example:

```javascript
db.books.deleteOne({ title: "Dune" })
db.books.findOne({ title: "Dune" })   // null
db.books.countDocuments()              // 9
```

## Common Stumbles

| Symptom | Cause | Fix |
|---------|-------|-----|
| `db.books is not a function` | Forgot `use library` | Run `use library` first |
| `$push` replaces array instead of appending | Used `$set` instead of `$push` | Use `$push` to append, `$set` to replace |
| `modifiedCount: 0` on Task 4 | Year filter typo or already tagged | Check `db.books.find({ year: { $lt: 1970 } })` first |
| Money values look like `89.9` not `89.90` | JavaScript number display | Cosmetic only — the stored value is correct |

## Reset

To restart this exercise from scratch:

```javascript
use library
db.books.drop()
```

Then re-import `books.json` from the command line (see top of this guide).

## Trainer Notes

- 20 minutes is firm. Set a visible timer.
- Walk the room, aim to touch every participant at least once.
- After 15 minutes, pair finishers with strugglers.
- Common stumble: `$push` vs `$set` on arrays. Demo both side-by-side if needed.
- At 20 minutes, walk solutions on screen. Don't shame anyone who didn't finish.

## Next Steps

- [Exercise 2 — Querying and Indexing](02-querying-indexing.md)
