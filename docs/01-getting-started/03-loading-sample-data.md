# Loading Sample Data

Import `books.json`, `products.json`, and `users.json` into the `library` database.
Module 2 exercises assume all three collections exist.

## The Sample Files

| File | Collection | Documents | Used In |
|------|------------|-----------|---------|
| `books.json` | `books` | 10 | Exercise 1 (CRUD), Exercise 2 (querying) |
| `products.json` | `products` | 6 | Exercise 2 (sorting, projection) |
| `users.json` | `users` | 8 | Exercise 2 (indexing), Exercise 3 (security) |

The files live in `sample-data/` at the lab root and are mounted read-only into
the `mongo-lab` container at `/sample-data` (see `docker-compose.yml:15`).

## Import with mongoimport

`mongoimport` is bundled in the `mongo:7` image. Run it through `docker exec`:

```bash
docker exec mongo-lab mongoimport \
  --username admin --password 'ChangeMe123!' --authenticationDatabase admin \
  --db library --collection books --jsonArray \
  --file /sample-data/books.json

docker exec mongo-lab mongoimport \
  --username admin --password 'ChangeMe123!' --authenticationDatabase admin \
  --db library --collection products --jsonArray \
  --file /sample-data/products.json

docker exec mongo-lab mongoimport \
  --username admin --password 'ChangeMe123!' --authenticationDatabase admin \
  --db library --collection users --jsonArray \
  --file /sample-data/users.json
```

Each command prints `10 document(s) imported successfully` (or whatever the
count is for that file).

> **Note**: `--jsonArray` is required because the sample files are JSON arrays.
> Without it, `mongoimport` expects one document per line (NDJSON).

## Verify the Import

Open `mongosh`:

```bash
docker exec -it mongo-lab mongosh -u admin -p 'ChangeMe123!'
```

Then check each collection:

```javascript
use library

db.books.countDocuments()      // 10
db.products.countDocuments()   // 6
db.users.countDocuments()      // 8

db.books.findOne()
db.products.findOne()
db.users.findOne()
```

You should see one document from each collection echoed back.

## Reset the Data

If you've changed documents during an exercise and want to start fresh,
drop and re-import the affected collection:

```javascript
use library
db.books.drop()
```

Then re-run the matching `mongoimport` command from the shell.

## Alternative: Load via mongosh

If you prefer to stay inside `mongosh`, you can load the sample data with `load()`
after copying it into a JavaScript file — but `mongoimport` is the canonical
approach and is what the slides demonstrate.

## Next Steps

- [Exercises](../02-exercises/README.md) — work through the three hands-on lab guides
