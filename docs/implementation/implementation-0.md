# Implementation 0: Stage 01, run the app locally

## Goal

Run the `goals` app on the server with a real Postgres database, and log in with Google from the laptop. This is the **local** half of stage 01. The AWS/Floci half comes later.

## Status (2026-09-28)

| # | Step | Status |
|---|------|--------|
| 1 | Google OAuth client created (redirect URI `http://localhost:8080/auth/google/callback`) | ✅ Done |
| 2 | `.env` filled in on the server (git-ignored) | ✅ Done and checked |
| 3 | Postgres running (`docker compose up -d`) | ✅ Running, Postgres 17.11 on `:5432` |
| 4 | Create the database tables (SQL from the README) | ⬜ Next |
| 5 | Build the app's Docker image | ⬜ |
| 6 | Run the app container on host port **8090** | ⬜ |
| 7 | SSH tunnel from the laptop, then log in at `http://localhost:8080` | ⬜ |
| 8 | Make yourself admin (`INSERT INTO administrators …`) | ⬜ |

## What we did

### `.env`

It holds four values: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URL` and `POSTGRES_URL`. The lines have no `export` prefix, which is the format Docker's `--env-file` expects.

If you ever load the file in a shell, run `set -a; source .env; set +a`, so the variables are passed on to programs.

### Postgres

```bash
docker compose up -d
```

**Issue:** the course's `docker-compose.yml` uses `image: postgres:alpine`, which now pulls **Postgres 18**. Postgres 18 refuses to start with the old mount at `/var/lib/postgresql/data`:

```
there appears to be PostgreSQL data in:
  /var/lib/postgresql/data (unused mount/volume)
...
The suggested container configuration for 18+ is to place a single mount
at /var/lib/postgresql ...
```

**Fix:** pin `postgres:17-alpine`. The course made the same fix later on its `main` branch, and 17 matches the RDS version used in stage 03.

```diff
-    image: postgres:alpine
+    image: postgres:17-alpine
```

```bash
docker compose down -v      # remove the broken container and its empty volume (this project only)
docker compose up -d
docker compose exec postgres pg_isready -U postgres
# /var/run/postgresql:5432 - accepting connections
```

## Plan for the remaining steps

### 4. Create the tables

Run the `CREATE TABLE` statements from the course README through `psql` in the Postgres container. The tables are `users`, `administrators`, `aspiration_updates`, `likes`, `followers` and `comments`.

```bash
docker compose exec -T postgres psql -U postgres -d postgres < schema.sql
```

### 5–6. Build and run the app in Docker (not `go run`)

The app hardcodes port `:8080`, and the server's 8080 belongs to another service, which we don't touch. So the app runs in a container, and we map it to a free port:

```bash
docker build -t goals:stage-01 .
docker run -d --name goals \
  --env-file .env \
  -e POSTGRES_URL=postgresql://postgres:password@host.docker.internal:5432/postgres?sslmode=disable \
  --add-host host.docker.internal:host-gateway \
  -p 127.0.0.1:8090:8080 \
  goals:stage-01
```

- `-p 127.0.0.1:8090:8080` publishes the app on the server's port 8090, only on the server itself (it's reached through the tunnel).
- Inside the container, `localhost` is the container itself, so `POSTGRES_URL` is overridden to point at the server (`host.docker.internal`).

### 7. Log in from the laptop

```bash
ssh -L 8080:localhost:8090 coz@coz
# open http://localhost:8080 → "Login with Google"
```

```
laptop browser → localhost:8080 ──tunnel──▶ server :8090 → container :8080 → app
```

### 8. Make yourself admin

```sql
INSERT INTO administrators (email, username)
SELECT email, username FROM users WHERE email = '<your gmail>';
```

## Next

Implementation 1: install the AWS CLI, goose and Terraform, then start Floci.
