# MongoDB Database and Security — Lab Kit

[![License](https://img.shields.io/github/license/developers-hub-my/mongodb-lab?style=flat-square)](LICENSE)
[![MongoDB](https://img.shields.io/badge/MongoDB-7.0-47A248?style=flat-square&logo=mongodb&logoColor=white)](https://www.mongodb.com/)
[![Docker](https://img.shields.io/badge/Docker-required-2496ED?style=flat-square&logo=docker&logoColor=white)](https://docs.docker.com/get-docker/)

A self-contained Docker-based lab for the **MongoDB Database and Security** one-day course.
Participants use the kit to run MongoDB 7 locally, load sample data, and work through three
hands-on exercises covering CRUD, indexing, and security hardening.

## What's in This Kit

| File / Folder | Purpose |
|---------------|---------|
| `docker-compose.yml` | MongoDB 7 + mongo-express stack |
| `sample-data/` | JSON fixtures mounted read-only into the container at `/sample-data` |
| `sample-data/books.json` | Sample data for Module 2 CRUD exercise |
| `sample-data/products.json` | Sample data for Module 2 indexing exercise |
| `sample-data/users.json` | Sample data for Module 2 indexing and Module 3 user-management exercises |
| `docs/` | Full course documentation (lab guides and reference) |
| `todo.md` | Trainer prep and delivery checklist |

## Quick Start

```bash
# 1. Start the lab stack
docker compose up -d

# 2. Verify both containers are healthy
docker ps

# 3. Connect to MongoDB shell
docker exec -it mongo-lab mongosh -u admin -p 'ChangeMe123!'

# 4. Open mongo-express in a browser
open http://localhost:8081   # login: admin / ChangeMe123!
```

Stop the stack when done:

```bash
docker compose down            # keep data
docker compose down -v         # remove data volumes too
```

## Endpoints

| Service | URL | Credentials |
|---------|-----|-------------|
| MongoDB | `mongodb://localhost:27017` | `admin` / `ChangeMe123!` |
| mongo-express | `http://localhost:8081` | `admin` / `ChangeMe123!` |

> **Warning**: The bundled credentials are intentionally weak for lab use only. Never reuse
> them outside this kit. Module 3 covers how to harden them for real deployments.

## Documentation

Full course material lives in [`docs/`](docs/README.md):

- **[Getting Started](docs/01-getting-started/README.md)** — prerequisites, lab setup, loading sample data
- **[Exercises](docs/02-exercises/README.md)** — three hands-on lab guides (CRUD, querying & indexing, security)
- **[Reference](docs/03-reference/README.md)** — mongosh cheatsheet and security checklist

> **Note**: The trainer slide deck is delivered separately and is not stored in this repository.

## Course at a Glance

| Module | Topic | Duration |
|--------|-------|----------|
| 1 | MongoDB & NoSQL Fundamentals | 75 min |
| 2 | CRUD Operations & Querying | 120 min |
| 3 | Database Security | 135 min |

## Audience

Skim F officers (max 40 pax) with general IT background. Prior database experience helpful
but not required.

## License

Released under the [MIT License](LICENSE). Use, fork, adapt for your own
training — attribution appreciated.
